# 十三、死信队列与延迟队列

### 死信（DLX）是什么

消息变成"死信"的三种情况：

1. 被 `basicNack` / `basicReject` 且 `requeue=false`
2. 消息 TTL 过期
3. 队列达到最大长度，溢出

死信不会凭空消失，如果队列配了 `x-dead-letter-exchange`，消息会被转发到指定的交换机。

### 配置死信队列

```java
// 死信交换机 + 死信队列
@Bean
public DirectExchange dlxExchange() {
    return new DirectExchange("dlx.exchange");
}

@Bean
public Queue dlxQueue() {
    return QueueBuilder.durable("dlx.queue").build();
}

@Bean
public Binding dlxBinding() {
    return BindingBuilder.bind(dlxQueue()).to(dlxExchange()).with("dlx");
}

// 业务队列，绑定死信
@Bean
public Queue businessQueue() {
    return QueueBuilder.durable("business.queue")
            .withArgument("x-dead-letter-exchange", "dlx.exchange")
            .withArgument("x-dead-letter-routing-key", "dlx")
            .build();
}
```

之后消费失败的、过期的消息都会进 `dlx.queue`。可以单独写一个消费者处理死信：告警、人工介入、或者落库等补偿。

### 延迟队列：订单超时关闭

需求：下单后 30 分钟未支付，自动关闭订单。

**方案一：TTL + DLX（推荐）。**

利用"消息在队列里过期后变成死信"这个特性：

```java
// 延迟队列：没有消费者，只用来"存放"消息直到过期
@Bean
public Queue delayQueue() {
    return QueueBuilder.durable("order.delay.queue")
            .withArgument("x-dead-letter-exchange", "order.exchange")
            .withArgument("x-dead-letter-routing-key", "order.close")
            .withArgument("x-message-ttl", 30 * 60 * 1000)     // 30 分钟
            .build();
}

// 真正处理关单的队列
@Bean
public Queue closeQueue() {
    return QueueBuilder.durable("order.close.queue").build();
}

@Bean
public Binding closeBinding() {
    return BindingBuilder.bind(closeQueue())
            .to(new DirectExchange("order.exchange")).with("order.close");
}
```

流程：

```
下单 → 发消息到 order.delay.queue（无消费者）
     ↓ 30 分钟后消息过期
     ↓ 变成死信，转到 order.exchange
     ↓ routingKey = order.close
     → 进 order.close.queue
     → 消费者收到，检查订单是否已支付
       已支付 → 忽略
       未支付 → 关闭订单，回滚库存
```

**关键点**：消费时要**再查一次订单状态**，因为用户可能在第 29 分钟付款了，但消息已经排在队列里。

::: warn 方案一的致命缺陷：队列级 TTL 不可变
`x-message-ttl` 是**队列级别**的。如果不同订单需要不同的超时时间（普通订单 30 分钟，秒杀订单 5 分钟），一个队列搞不定。

而且 RabbitMQ 只检查**队首**消息是否过期。如果队首是 30 分钟的消息，后面有个 5 分钟的，那个 5 分钟的也要等 30 分钟才能被处理——**消息不会按 TTL 排序**。

解法：每种超时时长建一个队列；或者用下面的方案二。
:::

**方案二：延迟插件（rabbitmq-delayed-message-exchange）。**

装插件后可以直接声明延迟交换机：

```java
@Bean
public CustomExchange delayExchange() {
    Map<String, Object> args = new HashMap<>();
    args.put("x-delayed-type", "direct");
    return new CustomExchange("delay.exchange", "x-delayed-message", true, false, args);
}
```

发送时指定延迟毫秒：

```java
rabbitTemplate.convertAndSend("delay.exchange", "order.close", message, msg -> {
    msg.getMessageProperties().setDelay(30 * 60 * 1000);
    return msg;
});
```

优点：**每条消息可以有自己的延迟时间**，不受队列 TTL 限制。
缺点：需要装插件（生产环境要评估），且延迟消息存在 Erlang 的 Mnesia 里，量大会影响性能。

**方案三：Redis ZSet 轮询。**

前面第二章提过。适合延迟精度要求不高（秒级）、量不大的场景。

### 三种方案对比

| 方案 | 精度 | 灵活性 | 运维成本 | 适用 |
|---|---|---|---|---|
| TTL + DLX | 秒级 | 差（队列级 TTL） | 低 | 统一超时时长 |
| 延迟插件 | 毫秒级 | 好（消息级） | 中（要装插件） | 多种超时时长 |
| Redis ZSet | 秒级 | 好 | 低 | 量小、可接受轮询延迟 |
