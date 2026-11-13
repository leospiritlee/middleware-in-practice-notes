# 一、Redis：不是缓存，是前置挡板

### 一个常见的误解

很多人把 Redis 当成"更快的数据库"，用法就是"查库之前先查 Redis"。这个理解在低并发下没问题，但在秒杀场景会失效。

为什么？因为**缓存只减少回源次数，不减少并发数**。十万个请求同时来，缓存全部命中，那十万次 `GET` 照样打在 Redis 上。如果后面还跟着一次 `DECR` 和一次数据库写，数据库还是会被打穿。

真正的做法是把 Redis 当成**挡板**：请求进来先过一道闸，闸门后面的世界（数据库）根本不知道前面来过十万人。

### 秒杀的四道闸

看项目里 `SeckillServiceImpl` 的实际结构，从上到下是四道：

| 层 | 手段 | 挡掉什么 | 成本 |
|---|---|---|---|
| 1 | 页面静态化 + CDN | 静态资源请求 | 几乎为零 |
| 2 | 页面缓存（Redis 存渲染好的 HTML） | 商品详情页查询 | 一次 GET |
| 3 | 内存标记 + 库存预减（`DECR`） | 超卖、无效请求 | 一次 DECR |
| 4 | 消息队列异步下单 | 数据库写压力 | 一次入队 |

只有第 3 层通过，才会进第 4 层。**十万人里，能走到队列的可能只有一千个。**

### 内存标记：比 Redis 还快的一道

项目里有个细节值得单独说：`SeckillServiceImpl` 用了一个 `ConcurrentHashMap<Long, Boolean>` 做本地标记，商品卖完就把 `goodsId -> true` 写进去，后续请求直接返回"已售罄"，**连 Redis 都不查**。

```java
private Map<Long, Boolean> localOverMap = new ConcurrentHashMap<>();

// 秒杀入口
boolean over = localOverMap.get(goodsId);
if (over) {
    return Result.error(CodeMsg.SECKILL_OVER);
}
```

这一层的意义在于：卖完之后的长尾请求（可能有几十万）全部在 JVM 内存里被拦掉，Redis 的 QPS 压力骤降。

::: warn 本地标记的代价
`ConcurrentHashMap` 是**单机**的。集群部署时，A 机器卖完了，B 机器还以为有货，请求照样打到 Redis。所以本地标记只能作为"第一道粗筛"，**真正的准确性必须由 Redis 的原子操作保证**。两层都要有，缺一不可。
:::

### 库存预减：为什么必须用 DECR

```java
long stock = redisService.decr(GoodsKey.getSeckillGoodsStock, "" + goodsId);
if (stock < 0) {
    localOverMap.put(goodsId, true);
    return Result.error(CodeMsg.SECKILL_OVER);
}
```

`DECR` 是原子操作，返回值就是减完之后的值。十万人同时 `DECR`，Redis 单线程串行执行，第 1001 个人拿到 -1，立刻知道卖完了。

如果换成"先 GET 再判断再 SET"，就是经典的 check-then-act 竞态：两个人同时读到 `stock=1`，都判断"有货"，都去减，结果卖出两件。

::: warn 但 DECR 有个致命前提
**key 必须存在。** Redis 的 `DECR` 对不存在的 key 会先当成 0，然后减 1，返回 **-1**。

如果库存 key 因为过期被删了，所有请求都会拿到负数，整个活动瞬间变成"已售罄"。这个坑在项目里真实存在，第八章细讲。
:::
