# 六、原子性与 Lua

### 为什么单条命令原子，多条不原子

Redis 是单线程执行命令的，所以**单条命令天然原子**。但两条命令之间可能插入别的客户端的命令：

```
客户端 A: GET stock     → 100
客户端 B: GET stock     → 100
客户端 A: SET stock 99
客户端 B: SET stock 99     ← 应该是 98，超卖了
```

这叫 check-then-act 竞态。解法有两种：用单条原子命令，或者用 Lua 脚本把多条命令打包。

### 优先用单条命令

能用一条命令解决的，绝不用两条。常见替代：

| 想做的事 | 错误写法 | 正确写法 |
|---|---|---|
| 库存减一 | `GET` 判断 >0 再 `SET` | `DECR`，判断返回值 |
| 加锁 | `SETNX` + `EXPIRE` | `SET key val NX PX 30000` |
| 存在才删 | `EXISTS` + `DEL` | `DEL`，判断返回值 |
| 计数并设过期 | `INCR` + `EXPIRE` | Lua（这个确实没单命令） |
| 集合弹出 | `SMEMBERS` + `SREM` | `SPOP` |

### Lua 脚本：把多条打包

Redis 执行 Lua 脚本时是**整体原子**的——脚本执行期间不会插入其他命令。

**场景一：库存扣减 + 记录用户（防重复购买）。**

```lua
-- KEYS[1] = 库存 key, KEYS[2] = 已购用户 set
-- ARGV[1] = userId
if redis.call('SISMEMBER', KEYS[2], ARGV[1]) == 1 then
    return -2                                  -- 已经买过了
end
local stock = tonumber(redis.call('GET', KEYS[1]))
if stock == nil or stock <= 0 then
    return -1                                  -- 库存不足
end
redis.call('DECR', KEYS[1])
redis.call('SADD', KEYS[2], ARGV[1])
return stock - 1                               -- 返回剩余库存
```

返回值的语义自己定：`-2` 重复购买，`-1` 售罄，`>=0` 成功。一次网络往返完成三个判断，且没有竞态。

**场景二：限流（令牌桶简化版）。**

```lua
-- KEYS[1] = 限流 key, ARGV[1] = 上限, ARGV[2] = 窗口秒数
local current = redis.call('INCR', KEYS[1])
if current == 1 then
    redis.call('EXPIRE', KEYS[1], ARGV[2])
end
if current > tonumber(ARGV[1]) then
    return 0
end
return 1
```

**场景三：释放锁（必须校验持有者）。**

```lua
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
else
    return 0
end
```

这个脚本是分布式锁的**必备**，第七章详讲。

### 在 Jedis 里执行 Lua

```java
public Long seckillLua(String stockKey, String userSetKey, String userId) {
    String script =
        "if redis.call('SISMEMBER', KEYS[2], ARGV[1]) == 1 then return -2 end " +
        "local stock = tonumber(redis.call('GET', KEYS[1])) " +
        "if stock == nil or stock <= 0 then return -1 end " +
        "redis.call('DECR', KEYS[1]) " +
        "redis.call('SADD', KEYS[2], ARGV[1]) " +
        "return stock - 1";

    try (Jedis jedis = jedisPool.getResource()) {
        Object r = jedis.eval(script,
                Arrays.asList(stockKey, userSetKey),   // KEYS
                Arrays.asList(userId));                // ARGV
        return (Long) r;
    }
}
```

::: warn eval 每次都要传脚本全文
`jedis.eval()` 每次都把脚本字符串发过去，Redis 每次都要解析。高频调用时应该用 `SCRIPT LOAD` 先加载，拿到 sha1，之后用 `EVALSHA`：

```java
String sha = jedis.scriptLoad(script);     // 启动时加载一次
jedis.evalsha(sha, keys, args);            // 之后都用 sha
```

如果 Redis 重启（脚本缓存丢失），`EVALSHA` 会报 `NOSCRIPT`，要捕获这个异常回退到 `eval`。
:::

### Lua 的三个限制

1. **脚本必须快。** Redis 单线程，一个跑了 100ms 的脚本会阻塞所有其他命令。超过 `lua-time-limit`（默认 5 秒）会开始拒绝其他命令。
2. **不能用随机函数。** `math.random` 在 Redis 4.0 之后需要显式初始化种子，而且主从复制时可能导致数据不一致（旧版本）。
3. **不能跨 key 保证原子性（集群模式）。** Redis Cluster 里，Lua 脚本涉及的所有 key 必须在**同一个 slot**。解决方法是给 key 加 hash tag：`{goods:123}:stock` 和 `{goods:123}:users` 会落在同一个 slot。
