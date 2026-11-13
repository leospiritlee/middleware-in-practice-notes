# 十、四种交换机

项目里的 `MQConfig` 把四种都写了一遍，正好拿来当教材。

### Direct：精确匹配 routingKey

```java
@Bean
public Queue queue() {
    return new Queue(QUEUE, true);        // true = 持久化
}

// 发送（MQConfig 里没写交换机，用的是默认交换机）
amqpTemplate.convertAndSend(MQConfig.QUEUE, msg);
```

上面这个其实用的是**默认交换机**（名字是空字符串 `""`），它的规则是"routingKey 等于队列名就投递"。这是最简单的点对点。

真正的 Direct 交换机：

```java
@Bean
public DirectExchange directExchange() {
    return new DirectExchange("direct.exchange");
}

@Bean
public Binding directBinding() {
    return BindingBuilder.bind(queue()).to(directExchange()).with("order.create");
}
```

`routingKey = "order.create"` 的消息才会进这个队列。**精确字符串相等**，不模糊匹配。

### Topic：通配符匹配

项目里的写法：

```java
@Bean
public TopicExchange topicExchage() {
    return new TopicExchange(TOPIC_EXCHANGE);
}

@Bean
public Binding topicBinding1() {
    return BindingBuilder.bind(topicQueue1()).to(topicExchage()).with("topic.key1");
}

@Bean
public Binding topicBinding2() {
    return BindingBuilder.bind(topicQueue2()).to(topicExchage()).with("topic.#");
}
```

`topic.#` 里的 `#` 匹配**零个或多个**词，`*` 匹配**恰好一个**词（词之间用 `.` 分隔）。

发送端：

```java
amqpTemplate.convertAndSend(MQConfig.TOPIC_EXCHANGE, "topic.key1", msg + "1");
amqpTemplate.convertAndSend(MQConfig.TOPIC_EXCHANGE, "topic.key2", msg + "2");
```

结果：

| routingKey | queue1 (`topic.key1`) | queue2 (`topic.#`) |
|---|---|---|
| `topic.key1` | 收到 | 收到 |
| `topic.key2` | 不收到 | 收到 |
| `topic.key1.sub` | 不收到 | 收到 |
| `topic` | 不收到 | **收到**（`#` 可匹配零个词） |

最后一行是 `#` 的特性，容易让人意外。

### Fanout：广播，忽略 routingKey

```java
@Bean
public FanoutExchange fanoutExchage() {
    return new FanoutExchange(FANOUT_EXCHANGE);
}

@Bean
public Binding FanoutBinding1() {
    return BindingBuilder.bind(topicQueue1()).to(fanoutExchage());
}
```

发到这个交换机的消息会进**所有**绑定的队列，routingKey 完全被忽略。

典型用途：配置变更通知、缓存失效广播、日志分发。

### Headers：按消息头匹配

```java
@Bean
public Binding headerBinding() {
    Map<String, Object> map = new HashMap<>();
    map.put("header1", "value1");
    map.put("header2", "value2");
    return BindingBuilder.bind(headerQueue1()).to(headersExchage()).whereAll(map).match();
}
```

匹配依据是消息的 headers 而不是 routingKey。`whereAll` 要求全部匹配，`whereAny` 任一匹配。

**实务中几乎不用。** 因为性能比 routingKey 差（要遍历 headers），而且不如 Topic 直观。它的存在主要是 AMQP 协议完整性。

### 四种交换机对比

| 类型 | 匹配依据 | 性能 | 典型场景 |
|---|---|---|---|
| Direct | routingKey 精确相等 | 最高 | 点对点、按业务类型分发 |
| Topic | routingKey 通配符 | 高 | 多维度订阅（`order.*.paid`） |
| Fanout | 不匹配，全广播 | 最高 | 广播通知、缓存失效 |
| Headers | 消息头键值 | 低 | 几乎不用 |

::: warn 项目里的配置有个坑
`FanoutBinding1` 和 `FanoutBinding2` 绑的是 `topicQueue1` / `topicQueue2`——这两个队列**同时**绑在 Topic 交换机和 Fanout 交换机上。

后果：往 Topic 交换机发 `topic.key1`，queue1 收到；往 Fanout 交换机发任何消息，queue1 和 queue2 都收到。**如果两个交换机的消费者逻辑不同，会互相干扰。**

而且 `MQConfig` 把测试用的四种交换机全塞进了一个 `@Configuration`，生产代码里应该只保留实际用到的那一种。
:::
