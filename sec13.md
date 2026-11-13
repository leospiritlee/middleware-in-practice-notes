# 十二、幂等与重复消费

### 为什么会重复

MQ 的投递语义是 **at-least-once（至少一次）**，不是 exactly-once。重复的来源有三个：

1. consumer 处理完，`basicAck` 的网络包丢了 → broker 以为没收到 → 重投
2. consumer 处理超时，broker 认为失败 → 重投
3. 生产者 confirm 没收到 → 重发

**任何 MQ 都会重复。** 想不重复，只能靠消费端幂等。

### 幂等的四种实现

**方案一：数据库唯一索引（最可靠）。**

```sql
CREATE TABLE seckill_order (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    goods_id BIGINT NOT NULL,
    ...
    UNIQUE KEY uk_user_goods (user_id, goods_id)
);
```

重复插入时抛 `DuplicateKeyException`，捕获后当成"已处理"返回成功。

```java
try {
    orderMapper.insert(order);
} catch (DuplicateKeyException e) {
    log.info("重复消息，已忽略: user={}, goods={}", userId, goodsId);
    return;   // 当成成功，ack 掉
}
```

优点：**绝对可靠**，由数据库保证。
缺点：需要业务表有天然的唯一键。

**方案二：Redis 去重（快，但不绝对）。**

```java
String dedupKey = "mq:dedup:" + messageId;
if (!redisService.setIfAbsent(dedupKey, "1", 3600)) {
    return;      // 已处理过
}
doBusiness();
```

问题：**setIfAbsent 成功但业务失败**时，key 已经写进去了，重试会被误判为"已处理"。

改进：先执行业务，成功后再写 key。但这样又有"业务成功、写 key 失败"的窗口。

**方案三：状态机（适合有状态的业务）。**

订单状态：`待支付 → 已支付 → 已发货`。

消费"支付成功"消息时，先查当前状态：

```java
Order order = orderMapper.selectById(orderId);
if (order.getStatus() != OrderStatus.WAIT_PAY) {
    return;      // 已经是已支付或更后面，忽略
}
// 用乐观锁更新
int rows = orderMapper.updateStatus(orderId, OrderStatus.WAIT_PAY, OrderStatus.PAID);
if (rows == 0) {
    return;      // 被别人抢先了
}
```

`updateStatus` 的 SQL 带 `WHERE status = 待支付`，天然幂等。

**方案四：全局消息 ID + 去重表。**

给每条消息分配一个全局唯一 ID（UUID 或雪花 ID），消费前查去重表：

```sql
CREATE TABLE mq_consume_log (
    msg_id VARCHAR(64) PRIMARY KEY,
    consumer_group VARCHAR(64),
    consume_time DATETIME
);
```

插入成功才继续处理，插入失败说明重复。**必须和业务在同一个事务里**，否则还是会漏。

### 四种方案对比

| 方案 | 可靠性 | 性能 | 适用 |
|---|---|---|---|
| 数据库唯一索引 | 最高 | 中 | 有天然唯一键的业务（订单、支付） |
| Redis 去重 | 中 | 高 | 允许极少数重复的场景（通知、日志） |
| 状态机 | 高 | 中 | 有明确状态流转的业务 |
| 去重表 | 高 | 低（多一次写） | 没有唯一键的通用场景 |

::: tip 项目里用的是哪种
`MQReceiver.receive()` 里先查一次 `orderService.getOrderByUserIdGoodsId()`，查到就 return。这是**方案三的简化版**（查状态而非状态机更新），配合数据库的唯一索引（如果建了）能保证幂等。

但它是"先查再插"，中间有窗口期。真正的幂等要靠唯一索引兜底——查不到但插入时冲突，捕获异常即可。
:::
