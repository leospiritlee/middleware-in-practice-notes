# 七、分布式锁

### 为什么需要分布式锁

单机用 `synchronized` 或 `ReentrantLock` 就够了。但部署两台机器时，两个 JVM 各有各的锁，互相看不见，等于没锁。

分布式锁的本质是：**找一个所有节点都能访问的、支持原子操作的地方存一个标记。**

### 最小可用实现

```java
public boolean tryLock(String key, String value, int expireSeconds) {
    try (Jedis jedis = jedisPool.getResource()) {
        String r = jedis.set(key, value, SetParams.setParams().nx().px(expireSeconds * 1000L));
        return "OK".equals(r);
    }
}

public boolean unlock(String key, String value) {
    String script =
        "if redis.call('GET', KEYS[1]) == ARGV[1] then " +
        "  return redis.call('DEL', KEYS[1]) " +
        "else return 0 end";
    try (Jedis jedis = jedisPool.getResource()) {
        return ((Long) jedis.eval(script, List.of(key), List.of(value))) == 1L;
    }
}
```

四个关键点：

| 点 | 为什么 |
|---|---|
| `NX` | 只有 key 不存在时才设置，保证互斥 |
| `PX 30000` | 设过期时间，防止持锁者崩溃后死锁 |
| `value` 唯一（UUID） | 解锁时校验，防止误删别人的锁 |
| Lua 校验后删除 | "判断是我的" 和 "删除" 必须原子 |

### 四个必须知道的坑

**坑一：忘了设过期时间。**

`SETNX` 成功之后如果进程崩溃，锁永远不释放。整个业务卡死。

所以**永远用 `SET key val NX PX`**，不要用 `SETNX` + 单独的 `EXPIRE`（这两条之间也可能崩溃）。

**坑二：解锁不校验持有者。**

```java
// 错误写法
public void unlock(String key) {
    jedis.del(key);       // 谁都能删
}
```

时序问题：

```
A 拿到锁，TTL 30s
A 的业务执行了 35s（超时了，锁自动释放）
B 拿到锁
A 执行完，调用 del(key)  ← 把 B 的锁删了
C 拿到锁  ← B 和 C 同时持锁
```

所以必须校验 value，且用 Lua 保证原子。

**坑三：业务执行时间超过锁的 TTL。**

上面的时序问题根源就在这。解法有两种：

- **看门狗（watchdog）**：起一个后台线程，每 10 秒检查一次"锁还在不在、业务跑完没"，没跑完就续期。Redisson 内置了这个机制。
- **估算 TTL**：把 TTL 设成业务 P99 耗时的 3~5 倍，接受偶尔超时。

::: warn 项目里的续期是坏的
`RedisService` 里有个 `set()` 方法，用的是 `setIfAbsent`（即 NX 语义）。如果代码想用它来实现"续期"（把 TTL 重置），会因为 key 已存在而直接返回 false，**续期永远失败**。

正确续期要用 `EXPIRE key 30` 或 `SET key val XX PX 30000`（XX = 只在存在时设置）。这个缺陷在第八章细讲。
:::

**坑四：主从切换丢锁。**

Redis 主从是异步复制的。A 在主节点加了锁，主节点还没把这条命令同步到从节点就宕机了，从节点升主——锁没了。

这是**架构层面**的问题，单靠 Redis 解决不了。要绝对可靠只能上 **RedLock**（向 N/2+1 个独立 Redis 实例加锁）或者 **ZooKeeper / etcd**（基于共识算法）。

RedLock 本身争议很大（Martin Kleppmann 和 antirez 打过笔仗），核心分歧是"依赖时钟的算法能否保证安全"。**实务建议：对绝大多数业务，单节点 Redis 锁 + 合理 TTL 就够了；涉及钱的场景用数据库行锁或 ZooKeeper。**

### 一个完整的锁工具类

```java
public class RedisLock {
    private final JedisPool pool;
    private final String key;
    private final String token = UUID.randomUUID().toString();
    private final int ttlMs;

    public RedisLock(JedisPool pool, String key, int ttlMs) {
        this.pool = pool; this.key = key; this.ttlMs = ttlMs;
    }

    public boolean tryLock(long waitMs) throws InterruptedException {
        long deadline = System.currentTimeMillis() + waitMs;
        while (System.currentTimeMillis() < deadline) {
            try (Jedis jedis = pool.getResource()) {
                if ("OK".equals(jedis.set(key, token,
                        SetParams.setParams().nx().px(ttlMs)))) {
                    return true;
                }
            }
            Thread.sleep(50);       // 简单退避，生产建议加随机抖动
        }
        return false;
    }

    public boolean unlock() {
        String lua = "if redis.call('GET',KEYS[1])==ARGV[1] then " +
                     "return redis.call('DEL',KEYS[1]) else return 0 end";
        try (Jedis jedis = pool.getResource()) {
            return ((Long) jedis.eval(lua, List.of(key), List.of(token))) == 1L;
        }
    }
}
```

用法：

```java
RedisLock lock = new RedisLock(pool, "lock:order:" + orderId, 30_000);
if (lock.tryLock(3000)) {
    try {
        doBusiness();
    } finally {
        lock.unlock();
    }
} else {
    throw new BizException("操作太频繁，请稍后重试");
}
```
