# 二、五种数据结构与选型

Redis 有九种类型（5 基础 + 4 扩展），但日常真正需要判断的只有五种。

### 一张选型表

| 结构 | 底层 | 适合存什么 | 秒杀项目里的用途 |
|---|---|---|---|
| String | SDS 动态字符串 | 单个值、计数、序列化对象 | 库存数、用户对象、页面 HTML |
| Hash | 哈希表 + 压缩列表 | 一个对象的多个字段，需要单独改某字段 | 购物车、部分用户信息 |
| List | 双向链表 / 快速列表 | 有序、两端进出、可重复 | 简单队列、最新 N 条 |
| Set | 哈希表 / 整数集合 | 去重、交并差 | 用户标签、抽奖池 |
| ZSet | 跳表 + 哈希表 | 需要按分数排序、排行榜 | 延时队列、热搜榜 |

### String 的三个高频用法

**用法一：计数器。** `INCR` / `DECR` / `INCRBY`，原子。库存、点赞数、限流计数都用它。

**用法二：分布式锁。** `SET key value NX PX 30000`，一条命令完成"不存在才设置 + 设过期"。第七章详讲。

**用法三：序列化对象。** 把 Java 对象转成 JSON 存进去。项目里的做法：

```java
public <T> boolean set(KeyPrefix prefix, String key, T value) {
    Jedis jedis = null;
    try {
        jedis = jedisPool.getResource();
        String str = beanToString(value);
        if (str == null || str.length() <= 0) return false;
        String realKey = prefix.getPrefix() + key;
        int seconds = prefix.expireSeconds();
        if (seconds <= 0) {
            jedis.set(realKey, str);
        } else {
            jedis.setex(realKey, seconds, str);
        }
        return true;
    } finally {
        returnToPool(jedis);
    }
}
```

注意 `prefix.expireSeconds() <= 0` 时用 `set`（永不过期），否则用 `setex`（带 TTL）。这个设计把"要不要过期"下放给了 Key 类，调用方不用每次传。

### Hash 什么时候比 String 好

存一个用户对象，两种写法：

```
# 写法 A：整个对象序列化成一个 String
user:1001  ->  {"id":1001,"name":"leo","level":3,"lastLogin":...}

# 写法 B：每个字段一个 field
user:1001  ->  {id:1001, name:"leo", level:3, lastLogin:...}
```

| 场景 | 选哪个 | 原因 |
|---|---|---|
| 总是整存整取 | String | 一次序列化，网络往返少 |
| 只改一个字段（比如加积分） | Hash | `HINCRBY user:1001 level 1`，不用读出来再写回去 |
| 对象很大（>10KB） | Hash | 避免每次改一个字段都要传输整个对象 |
| 需要给整个对象设 TTL | String | Hash 的 TTL 只能设在 key 上，不能设在 field 上 |

项目里 `User` 对象用的是 String，因为登录后整存整取，改字段的需求很少。

### ZSet 的隐藏用法：延时队列

ZSet 的 score 可以是**时间戳**。存进去之后，用一个定时任务轮询：

```
ZADD delay:order 1730000000 order:123
ZRANGEBYSCORE delay:order 0 1730000000 LIMIT 0 10
```

取出 score 小于当前时间的成员，就是"该执行的任务"。订单超时未支付自动关闭，用这个方案比 RabbitMQ 的延迟插件更轻。

::: tip 但生产上更推荐 RabbitMQ 的延迟队列
ZSet 方案要自己写轮询，要处理"取出来但执行失败"的重试，要保证多实例不重复取。RabbitMQ 的死信 + TTL 方案（第十三章）把这些都做进中间件了。
:::
