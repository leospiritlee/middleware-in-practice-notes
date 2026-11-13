# 九、RabbitMQ：AMQP 模型

### 和"简单队列"的根本区别

很多人第一次接触 MQ 是 Redis 的 `LPUSH` / `BRPOP`。那种模型是：生产者推到一个列表，消费者从列表取。**生产者和队列直接绑定。**

RabbitMQ 多了一层 **Exchange（交换机）**：

```
生产者 → Exchange → (Binding + RoutingKey) → Queue → 消费者
```

这一层带来的能力是：**生产者不需要知道有几个队列、队列叫什么。** 它只管把消息发给交换机，附带一个 routing key，由交换机和绑定规则决定消息去哪。

代价是概念变多。五个核心概念：

| 概念 | 作用 | 类比 |
|---|---|---|
| **Connection** | 一条 TCP 连接 | 一根网线 |
| **Channel** | 连接上的逻辑通道 | 网线里的多路复用通道 |
| **Exchange** | 接收消息并按规则路由 | 邮局分拣中心 |
| **Queue** | 存消息 | 收件人的信箱 |
| **Binding** | Exchange 和 Queue 的关系 + 路由规则 | 分拣规则表 |

### 为什么要有 Channel

一个应用只需要**一条** TCP 连接，但可以开**多个** Channel。所有收发都在 Channel 上做。

原因：TCP 连接建立成本高（三次握手），而 RabbitMQ 的协议是单连接多通道复用。**Connection 是重量级的，Channel 是轻量级的。**

::: warn Channel 不是线程安全的
`Channel` 实例**不能在多线程间共享**。多个线程同时在一个 Channel 上 `basicPublish` 会导致协议帧交错，RabbitMQ 会直接关闭连接，报 `COMMAND_INVALID` 或 `UNEXPECTED_FRAME`。

Spring AMQP 的 `RabbitTemplate` 内部用 `ThreadLocal` 保证每个线程一个 Channel，所以业务代码不用操心。但如果你自己用原生 Java Client，必须每个线程一个 Channel。
:::

### 消息的生命周期

```
1. producer 发消息到 exchange（带 routingKey）
2. exchange 根据类型和 binding 找到匹配的 queue
3. 消息进入 queue（可能持久化到磁盘）
4. consumer 订阅 queue，broker 推送消息
5. consumer 处理完，发 ack
6. broker 收到 ack，删除消息
```

第 5 步是关键。**没有 ack，消息不会删。** 如果 consumer 崩溃没发 ack，消息会重新入队（redeliver），可能被别的 consumer 拿到。

### 一个最小的 Spring AMQP 配置

```properties
spring.rabbitmq.host=127.0.0.1
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
spring.rabbitmq.virtual-host=/
# 生产端确认
spring.rabbitmq.publisher-confirm-type=correlated
spring.rabbitmq.publisher-returns=true
# 消费端手动 ack
spring.rabbitmq.listener.simple.acknowledge-mode=manual
spring.rabbitmq.listener.simple.prefetch=10
```

`prefetch=10` 的意思是"这个 consumer 最多同时持有 10 条未 ack 的消息"。配大了会导致消息分配不均（一个 consumer 囤一堆，另一个闲着），配小了吞吐上不去。经验值：**消费者处理越快，prefetch 越大**，一般 10~50。
