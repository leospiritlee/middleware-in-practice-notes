# 十一、可靠投递

"消息丢了"是 MQ 最常见的事故。消息可能在三个环节丢：

```
生产者 →(1)→ Broker →(2)→ 磁盘 →(3)→ 消费者
```

三个环节都要各自保证。

### 环节一：生产者 → Broker

**问题**：`convertAndSend` 返回 void，消息发出去到底到没到 broker，生产者不知道。

**解法一：publisher confirm（确认）。**

```properties
spring.rabbitmq.publisher-confirm-type=correlated
```

```java
rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
    if (!ack) {
        log.error("消息未到达 broker: {}, 原因: {}",
                correlationData != null ? correlationData.getId() : "null", cause);
        // 重发 or 落库补偿
    }
});
```

`ack=true` 表示 broker 收到了。注意：**收到 ≠ 存到磁盘**，只表示进了 exchange。

**解法二：publisher return（退回）。**

confirm 只能告诉你"broker 收到了"，但如果 exchange 找不到匹配的 queue，消息会被**静默丢弃**（`mandatory=false` 时）。

```properties
spring.rabbitmq.publisher-returns=true
spring.rabbitmq.template.mandatory=true
```

```java
rabbitTemplate.setReturnsCallback(returned -> {
    log.error("消息无法路由: exchange={}, routingKey={}, replyText={}",
            returned.getExchange(), returned.getRoutingKey(), returned.getReplyText());
});
```

`mandatory=true` 是必须的——否则 broker 直接丢掉，连回调都不触发。

**解法三：本地消息表（最终一致性）。**

前两个都是"实时反馈"，失败后还是要重发。更稳的做法是**先落库再发**：

```
1. 业务事务里：写业务表 + 写消息表（status=待发送），同一个本地事务
2. 事务提交后：发 MQ
3. 发送成功：更新消息表 status=已发送
4. 定时任务扫描 status=待发送 且超过 N 秒的，重发
```

这个方案不依赖 MQ 的可靠性，代价是要多一张表和一次扫描。

### 环节二：Broker 内存 → 磁盘

**问题**：broker 收到消息后先放内存，异步刷盘。如果这时 broker 崩溃，内存里的消息丢了。

**解法：队列和消息都持久化。**

```java
// 队列持久化
new Queue(QUEUE, true);       // 第二个参数 durable

// 消息持久化
Message message = MessageBuilder.withBody(bytes)
        .setDeliveryMode(MessageDeliveryMode.PERSISTENT)
        .build();
```

::: warn 三个条件缺一不可
1. 队列 `durable=true`
2. 消息 `deliveryMode=PERSISTENT`
3. 交换机 `durable=true`

少任何一个，重启后消息就没了。

项目里 `new Queue(QUEUE, true)` 是对的，但 `SECKILL_QUEUE` 用的是 `new Queue(SECKILL_QUEUE)`——**默认构造器也是 durable=true**，所以没问题。不过消息持久化要看 `MessageConverter` 的配置，`Jackson2JsonMessageConverter` 默认是 `PERSISTENT`，但显式设一下更安全。
:::

### 环节三：Broker → 消费者

**问题**：consumer 收到消息，处理到一半崩溃。消息算不算消费成功？

**默认行为（auto ack）**：broker 把消息推给 consumer 就立刻删除。consumer 崩溃 → **消息永久丢失**。

**正确做法（manual ack）**：

```properties
spring.rabbitmq.listener.simple.acknowledge-mode=manual
```

```java
@RabbitListener(queues = MQConfig.SECKILL_QUEUE)
public void receive(String message, Channel channel,
                    @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException {
    try {
        doBusiness(message);
        channel.basicAck(tag, false);              // 处理成功，确认
    } catch (BusinessException e) {
        // 业务异常，不重试，直接丢弃（或进死信）
        channel.basicNack(tag, false, false);
    } catch (Exception e) {
        // 系统异常，重新入队
        channel.basicNack(tag, false, true);
    }
}
```

`basicNack(tag, multiple, requeue)` 的第三个参数很关键：

- `requeue=true` —— 消息重新入队，可能被再次消费。**如果代码有 bug 导致必然失败，会无限循环**。
- `requeue=false` —— 消息丢弃（或进死信队列，如果配了）。

::: warn 无限重试是 MQ 事故的头号原因
一个反序列化 bug 导致每条消息都抛异常，`requeue=true`，消息被重新投递，再抛异常，再投递……CPU 打满，队列堆积，整个消费链路瘫痪。

**防御手段**：
1. 业务异常和系统异常分开处理，业务异常不 requeue。
2. 用 `x-death` header 统计重试次数，超过 3 次进死信队列。
3. 消费端加重试上限（Spring 的 `RetryTemplate`，配 `maxAttempts`）。
:::

### 可靠投递的三道关汇总

| 环节 | 手段 | 保证 |
|---|---|---|
| 生产 → Broker | publisher confirm | 消息到达 exchange |
| 生产 → Broker | publisher return + mandatory | 消息路由到 queue |
| 生产 → Broker | 本地消息表 | 最终一定发出 |
| Broker 内存 → 磁盘 | 队列/交换机/消息都持久化 | 重启不丢 |
| Broker → 消费 | 手动 ack | 处理完才删 |
| Broker → 消费 | 死信队列 | 失败有兜底 |
