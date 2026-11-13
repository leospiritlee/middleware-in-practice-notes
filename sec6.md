# 五、缓存三大问题：穿透、击穿、雪崩

这三个词经常被混用，但成因和治法完全不同。

### 一张表分清

| 问题 | 一句话 | 请求特征 | 后果 |
|---|---|---|---|
| **穿透** | 查一个**根本不存在**的数据 | 大量不同的 key，全部未命中 | 每次都打到数据库 |
| **击穿** | 查一个**存在但刚好过期**的热点 key | 大量**相同**的 key，同时未命中 | 数据库瞬间被打爆 |
| **雪崩** | **大批 key 同时过期** | 大量不同 key，同一时刻全失效 | 数据库整体被压垮 |

### 穿透：布隆过滤器与空值缓存

场景：攻击者用 `goodsId=-1`、`goodsId=999999999` 这种不存在的 ID 疯狂请求。缓存里没有，数据库里也没有，每次都白查一趟。

**治法一：缓存空值。**

```java
GoodsVo goods = redisService.get(GoodsKey.getGoodsDetail, "" + id, GoodsVo.class);
if (goods == null) {
    goods = goodsService.getGoodsVoById(id);
    if (goods == null) {
        // 缓存一个空对象，TTL 短一点（比如 60 秒）
        redisService.set(GoodsKey.getGoodsDetail, "" + id, new GoodsVo());
        return null;
    }
    redisService.set(GoodsKey.getGoodsDetail, "" + id, goods);
}
```

代价是**每次都要存一个空对象**，如果攻击者用随机 ID，会把 Redis 撑爆。所以 TTL 要短（60 秒级），并且最好配合限流。

**治法二：布隆过滤器。**

在缓存前面加一层位数组，把所有存在的 ID 预先写进去。查询时先问布隆："这个 ID 可能存在吗？"

- 说不存在 → **一定不存在**，直接返回
- 说存在 → **可能存在**，继续查缓存

布隆的代价是**假阳性**（说不存在的可能实际存在，但概率可控），好处是**绝不假阴性**，而且内存占用极小（1 亿个 ID 约 114 MB，误判率 1%）。

Redis 4.0 起有 `BF.ADD` / `BF.EXISTS` 命令（RedisBloom 模块）。项目里没用，这是个可以改进的点。

### 击穿：互斥锁与逻辑过期

场景：商品 123 是爆款，缓存 60 秒过期。第 60.001 秒，一万个请求同时发现缓存没了，同时去查数据库。

**治法一：互斥锁（简单可靠）。**

```java
GoodsVo goods = redisService.get(key, id, GoodsVo.class);
if (goods != null) return goods;

// 只让一个线程去重建缓存
String lockKey = "lock:goods:" + id;
if (redisService.setIfAbsent(lockKey, "1", 10)) {     // 抢到锁
    try {
        goods = goodsService.getGoodsVoById(id);       // 查库
        redisService.set(key, id, goods);              // 回填
    } finally {
        redisService.del(lockKey);
    }
} else {
    // 没抢到，等一会儿再读缓存
    Thread.sleep(50);
    return redisService.get(key, id, GoodsVo.class);
}
```

代价是**其他线程要等**，吞吐下降。但对爆款商品来说，这个代价可以接受——总比打穿数据库强。

**治法二：逻辑过期（不设 TTL）。**

key 永不过期，但 value 里带一个 `expireAt` 字段。读到之后判断：

- 没过期 → 直接返回
- 过期了 → **异步**起一个线程重建缓存，当前请求仍然返回旧值

好处是**永远不阻塞**，坏处是**会返回短暂脏数据**。适合对一致性要求不高的场景（比如商品详情页的浏览量）。

### 雪崩：过期时间加随机

场景：凌晨 0 点做了一次全量预热，所有商品缓存 TTL 都是 3600 秒。凌晨 1 点，全部同时过期，数据库被瞬间打爆。

**治法一：TTL 加随机扰动。**

```java
int base = 3600;
int ttl = base + ThreadLocalRandom.current().nextInt(300);   // 3600 ~ 3900
redisService.setex(key, ttl, value);
```

把"同一时刻过期"打散成"5 分钟内陆续过期"，峰值被摊平。

**治法二：多级缓存。**

本地 Caffeine + Redis。本地缓存扛住绝大部分请求，Redis 只是第二层。本地缓存过期时间更短（比如 5 秒），即使 Redis 全挂，本地还能顶一会儿。

**治法三：熔断降级。**

Redis 挂了不要紧，关键是别让请求全部涌向数据库。用 Sentinel / Resilience4j 做熔断，Redis 不可用时直接返回兜底数据（比如"系统繁忙，请稍后再试"）。

::: warn 项目里的雪崩隐患
`GoodsKey.getGoodsList` 和 `getGoodsDetail` 都是固定的 60 秒 TTL，**没有随机扰动**。如果预热时批量写入，60 秒后会同时过期。

秒杀场景下这个风险被"库存预减"挡掉了一部分（库存 key 永不过期），但商品详情页仍然有雪崩可能。
:::
