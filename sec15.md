# 十四、秒杀项目里的 MQ 用法复盘

### 用法本身是对的

项目的主链路是这样的：

```java
// 1. SeckillController：Redis 预减成功后才发消息
long stock = redisService.decr(GoodsKey.getSeckillGoodsStock, "" + goodsId);
if (stock < 0) {
    return Result.error(CodeMsg.SECKILL_OVER);
}
SeckillMessage sm = new SeckillMessage();
sm.setUser(user);
sm.setGoodsId(goodsId);
mqSender.sendSeckillMessage(sm);
return Result.success(0);      // 0 表示"排队中"

// 2. MQReceiver：异步真正下单
@RabbitListener(queues = MQConfig.SECKILL_QUEUE)
public void receive(String message) {
    SeckillMessage sm = redisService.stringToBean(message, SeckillMessage.class);
    User user = sm.getUser();
    long goodsId = sm.getGoodsId();

    GoodsVo goods = goodsService.getGoodsVoById(goodsId);
    int stock = goods.getStockCount();
    if (stock <= 0) return;

    SeckillOrder order = orderService.getOrderByUserIdGoodsId(user.getId(), goodsId);
    if (order != null) return;

    seckillService.seckill(user, goods);      // 事务：减库存 + 下订单 + 写秒杀订单
}
```

**这个设计思路是对的**：Redis 挡住 99%，MQ 把剩下 1% 削峰，数据库平稳落库。

### 但有五个问题

**问题一：消息体塞了整个 User 对象。**

```java
sm.setUser(user);     // User 里有 id、name、password、salt、registerDate、lastLoginDate...
```

后果：

- 消息体积变大（User 全字段序列化可能有几百字节，而实际只需要 `id`）
- **敏感信息进了队列**。如果 User 里带 `password`（哪怕是加盐哈希），会出现在 RabbitMQ 的管理界面、日志、以及死信队列里。
- User 对象结构一变，队列里积压的旧消息反序列化会失败。

**修法**：只传 `userId`，消费端按需查库。

```java
public class SeckillMessage {
    private Long userId;        // 只传 ID
    private Long goodsId;
}
```

**问题二：消费端没有幂等保证（只有查重）。**

```java
SeckillOrder order = orderService.getOrderByUserIdGoodsId(user.getId(), goodsId);
if (order != null) return;
```

"先查再插"中间有窗口：两条重复消息同时到达，两个消费者都查到 null，都去插，结果插了两条。

**修法**：`seckill_order` 表加唯一索引 `UNIQUE(user_id, goods_id)`，捕获 `DuplicateKeyException`。

**问题三：`goods` 可能为 null，直接 NPE。**

```java
GoodsVo goods = goodsService.getGoodsVoById(goodsId);
int stock = goods.getStockCount();     // ← goods 为 null 时 NPE
```

商品被下架或删除后，队列里积压的消息会全部 NPE。而 NPE 抛出去后，默认行为是**重新入队**（Spring AMQP 默认 `requeue=true`），于是无限循环。

**修法**：判空 + 业务异常不 requeue。

**问题四：没有手动 ack，异常会无限重试。**

```java
@RabbitListener(queues = MQConfig.SECKILL_QUEUE)
public void receive(String message) { ... }
```

方法签名里没有 `Channel`，说明用的是 **auto ack** 模式（Spring AMQP 的 `AcknowledgeMode.AUTO` 实际上是"方法正常返回就 ack，抛异常就 nack + requeue"）。

配合问题三的 NPE，就是**无限重投 + 无限 NPE**。

**修法**：

```properties
spring.rabbitmq.listener.simple.acknowledge-mode=manual
spring.rabbitmq.listener.simple.retry.enabled=true
spring.rabbitmq.listener.simple.retry.max-attempts=3
spring.rabbitmq.listener.simple.retry.initial-interval=1000
```

或者用 `default-requeue-rejected=false`，让异常消息直接丢弃（配合死信队列收集）。

**问题五：没有死信队列，失败消息无处可去。**

`MQConfig` 里的队列都没配 `x-dead-letter-exchange`。消费失败的消息要么无限重投，要么被丢弃，**没有任何可追溯的记录**。

线上出现"用户说抢到了但没订单"这类问题时，无从查起。

**修法**：所有业务队列都配死信，死信队列单独消费，落库 + 告警。

### 复盘表

| 编号 | 问题 | 后果 | 修法 |
|---|---|---|---|
| 1 | 消息体塞整个 User | 体积大、泄露敏感信息、结构耦合 | 只传 userId |
| 2 | 消费端只有查重没有幂等 | 重复下单 | 唯一索引兜底 |
| 3 | goods 为 null 直接 NPE | 无限重投 | 判空 + 业务异常处理 |
| 4 | auto ack + 异常 requeue | 死循环打满 CPU | 手动 ack + 重试上限 |
| 5 | 无死信队列 | 失败消息不可追溯 | 配 DLX + 落库告警 |

::: tip 这五条的共同点
**都是"异常路径没设计"。** 正常流程写得挺好，但没人问"如果这一步失败了呢"。

MQ 的所有坑几乎都在异常路径上：消息丢了、消息重复了、消费失败了、失败了还在重试。设计的时候把这几条画出来，比看十篇文章有用。
:::
