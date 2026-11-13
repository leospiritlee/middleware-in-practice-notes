# 四、客户端与连接池

### Jedis / Lettuce / Redisson 怎么选

| 客户端 | 线程模型 | 连接复用 | 特点 | 适用 |
|---|---|---|---|---|
| **Jedis** | 同步阻塞，一个连接一个线程 | 靠连接池 | 简单直接，API 贴近 Redis 命令 | 传统项目、连接数可控 |
| **Lettuce** | 基于 Netty，异步非阻塞 | 单连接多路复用 | 线程安全，Spring Boot 2.x 默认 | 高并发、响应式 |
| **Redisson** | 基于 Netty | 单连接多路复用 | 提供分布式锁、布隆过滤器等高级对象 | 需要分布式协调 |

项目用的是 **Jedis**。原因是它简单，而且连接池模型对"每个请求借还一次"的场景够用。

::: warn Jedis 不是线程安全的
`Jedis` 实例**不能跨线程共享**。多线程共用一个 `Jedis` 会收到 `Unexpected end of stream` 或读到别人的响应。

正确姿势：从 `JedisPool` 借，用完还。项目里的 `RedisService` 每个方法都是这个模式：

```java
Jedis jedis = null;
try {
    jedis = jedisPool.getResource();
    // ... 用 jedis
} finally {
    returnToPool(jedis);
}
```
:::

### 连接池配置怎么算

项目的 `RedisPoolFactory`：

```java
@Bean
public JedisPool JedisPoolFactory() {
    JedisPoolConfig poolConfig = new JedisPoolConfig();
    poolConfig.setMaxIdle(redisConfig.getPoolMaxIdle());       // 200
    poolConfig.setMaxTotal(redisConfig.getPoolMaxTotal());     // 1000
    poolConfig.setMaxWaitMillis(redisConfig.getPoolMaxWait() * 1000);  // 3000ms
    return new JedisPool(poolConfig, redisConfig.getHost(), redisConfig.getPort(),
            redisConfig.getTimeout() * 1000, redisConfig.getPassword(), 0);
}
```

三个参数的含义：

| 参数 | 含义 | 配小了会怎样 | 配大了会怎样 |
|---|---|---|---|
| `maxTotal` | 池里最多多少个连接 | 高并发时借不到，阻塞到超时 | 占满 Redis 的 `maxclients`，别的应用连不上 |
| `maxIdle` | 最多保留多少空闲连接 | 频繁创建销毁连接，TCP 握手开销 | 空闲连接白占 Redis 资源 |
| `maxWaitMillis` | 借不到时等多久 | 直接抛异常 | 线程长时间阻塞，拖垮整个请求 |

**经验公式**：

```
maxTotal ≈ 单机 QPS × 平均耗时(秒) × 冗余系数(1.5~2)
```

比如单机 5000 QPS，每次 Redis 操作平均 1ms，那么同时在用的连接约 5 个，`maxTotal` 设 10~20 就够。**绝大多数项目把 `maxTotal` 设成几百上千，其实完全没必要**，反而会打爆 Redis。

::: warn maxIdle 不能大于 maxTotal
`JedisPoolConfig` 继承自 `GenericObjectPoolConfig`，如果 `maxIdle > maxTotal`，连接池会抛 `IllegalStateException`。有些版本只是警告，但行为不确定。

另外：**`maxIdle` 设成和 `maxTotal` 一样是最常见的配置**，因为 Jedis 归还连接时如果空闲数超过 `maxIdle` 会直接销毁连接，等于每次高并发后都要重建一批连接。
:::

### 密码传 null 会怎样

```java
new JedisPool(poolConfig, host, port, timeout, password, 0);
```

这个构造器的 `password` 参数传 `null` 是合法的（表示无密码）。但如果 Redis 配了密码而你传了 `null`，第一次执行命令才会报 `NOAUTH Authentication required`，**连接池初始化时不报错**。

这就是第六章说的 GBK 编码坑为什么难查——配置文件里那行 `redis.password=` 被吞掉，`RedisConfig.password` 是 `null`，连接池照建不误，直到真正执行命令才炸，堆栈还指向业务代码。

### 连接泄漏的排查

`finally` 里 `returnToPool` 看似万无一失，但有两个漏洞：

**漏洞一：`getResource()` 本身抛异常。** 此时 `jedis` 是 `null`，`returnToPool(null)` 要判空，否则 NPE 掩盖真实异常。

**漏洞二：`returnToPool` 里 `jedis.close()` 抛异常。** 这个连接就永久泄漏了。

排查方法：`INFO clients` 看 `connected_clients`，或者用 `CLIENT LIST` 看哪些连接的 `idle` 时间异常长。

```bash
redis-cli info clients
# connected_clients:1023   ← 接近 maxclients 就是泄漏了
```

::: tip 更稳的写法：try-with-resources
```java
try (Jedis jedis = jedisPool.getResource()) {
    return jedis.get(key);
}
```
`Jedis` 实现了 `Closeable`，`close()` 等价于归还连接。这样连 `finally` 都不用写。
:::
