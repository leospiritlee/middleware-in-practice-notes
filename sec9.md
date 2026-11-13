# 八、秒杀项目里的四个 Redis 缺陷

这一章是我读 `sec-kill-product-master` 源码时找出来的真实问题。每一条都能在线上炸。

### 缺陷一：页面缓存 key 不含 userId，导致跨用户信息泄漏

**现象**：A 用户登录后打开商品详情页，页面右上角显示"A 已登录"。B 用户（未登录）打开同一商品，看到的是"A 已登录"。

**原因**：`GoodsController` 的详情页把**渲染好的 HTML** 存进 Redis，key 是 `GoodsKey:gd:{goodsId}`。但这个 HTML 里包含了当前登录用户的信息（用户名、"我的订单"入口）。

```java
// GoodsController.toDetail 的大致逻辑
String html = redisService.get(GoodsKey.getGoodsDetail, "" + goodsId, String.class);
if (!StringUtils.isEmpty(html)) {
    return html;                    // ← 直接返回，不区分是谁
}
// 缓存未命中，渲染页面（含当前 user 信息）后写回
redisService.set(GoodsKey.getGoodsDetail, "" + goodsId, html);
```

**为什么严重**：这不只是"显示错了"，而是**越权**。如果页面里有"我的订单号""收货地址"这类信息，就是数据泄漏。

**修法**：页面缓存只能缓存**与用户无关**的部分。要么把用户信息改成前端异步拉取，要么把 userId 拼进 key（但那样缓存命中率极低，不如不缓存整页）。

### 缺陷二：库存 key 设了 TTL，过期后整个活动被判售罄

**现象**：秒杀活动开始 30 分钟后，所有请求都返回"已售罄"，但实际库存还剩几百件。

**原因**：库存预热时给 key 设了过期时间。

```java
// 预热逻辑（大意）
redisService.set(GoodsKey.getSeckillGoodsStock, "" + goodsId, stock);
// 而 GoodsKey.getSeckillGoodsStock 的定义是：
public static GoodsKey getSeckillGoodsStock = new GoodsKey(1800, "gs");   // ← 1800 秒
```

1800 秒后 key 过期。此时 `DECR` 对不存在的 key 会**先当 0 再减 1**，返回 `-1`。

```java
long stock = redisService.decr(GoodsKey.getSeckillGoodsStock, "" + goodsId);
if (stock < 0) {
    localOverMap.put(goodsId, true);       // 标记售罄
    return Result.error(CodeMsg.SECKILL_OVER);
}
```

`-1 < 0` 成立 → 标记售罄 → 后续所有请求直接被本地标记挡掉。**活动提前结束，且不会自愈**（因为 `localOverMap` 一旦写入就不再清除）。

**修法**：

1. 库存 key **必须永不过期**（`new GoodsKey(0, "gs")`）。
2. 更稳妥：预热时用 `SET` 而不是 `SETEX`，并且加一个定时任务巡检，发现 key 不存在就重新预热。
3. 防御性写法：`DECR` 之后如果返回负数，先检查 key 是否存在（`EXISTS`），不存在说明是"key 丢了"而不是"卖完了"，应该触发重新预热而不是标记售罄。

::: warn 这条最值得记
**任何参与原子运算的 key，都不能设 TTL。** 计数器、库存、限流窗口，全都一样。TTL 一到，`DECR` 从 0 开始减，`INCR` 从 0 开始加，业务逻辑全部错位。
:::

### 缺陷三：用 setIfAbsent 实现"续期"，续期永远失败

**现象**：会话/锁的过期时间不会延长，用户操作到一半突然掉线。

**原因**：`RedisService.set()` 内部用的是 `setIfAbsent`：

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
            jedis.setex(realKey, seconds, str);      // ← 这里其实是对的
        }
        return true;
    } finally {
        returnToPool(jedis);
    }
}

public <T> boolean setIfAbsent(KeyPrefix prefix, String key, T value) {
    // 用的是 jedis.setnx + expire，语义是"不存在才设"
}
```

问题出在**调用方**：想"续期"的代码调了 `setIfAbsent`，key 已经存在，`setnx` 返回 0，方法返回 `false`，**TTL 一点没变**。

**修法**：续期必须用 `EXPIRE`（只改 TTL 不动值）或 `SET ... XX PX`（存在才设，会覆盖值）。

```java
public boolean renew(KeyPrefix prefix, String key, int seconds) {
    try (Jedis jedis = jedisPool.getResource()) {
        return jedis.expire(prefix.getPrefix() + key, seconds) == 1L;
    }
}
```

### 缺陷四：反序列化返回 null 后直接拆箱，NPE

**现象**：偶发 `NullPointerException`，堆栈指向一行看起来不可能为 null 的代码。

**原因**：`stringToBean` 在 JSON 解析失败或值为空时返回 `null`，调用方直接赋值给 `int` / `long` 字段，自动拆箱抛 NPE。

```java
// 危险写法
int stock = redisService.get(GoodsKey.getSeckillGoodsStock, "" + goodsId, Integer.class);
// 缓存里没这个 key → get 返回 null → 拆箱 → NPE
```

**修法**：一律用包装类型接，然后判空。

```java
Integer stock = redisService.get(GoodsKey.getSeckillGoodsStock, "" + goodsId, Integer.class);
if (stock == null) {
    // 缓存缺失，回源或重新预热
}
```

::: tip 这一类问题的通用规律
**缓存读取的返回值，永远是"可能没有"。** 任何 `redisService.get(...)` 的结果都必须当成可空处理，不能假设它一定存在。写代码时把这一点当成硬约束，能消掉一大半线上 NPE。
:::

### 顺带一提：重复的错误码

`CodeMsg` 里有两个不同的错误用了同一个 code `500100`。前端拿到这个 code 无法区分是"库存不足"还是"重复下单"，只能显示一个含糊的提示。

这类问题不影响功能，但会让线上排障变难——日志里看到 `500100`，你不知道是哪条业务分支。

### 缺陷汇总表

| 编号 | 缺陷 | 严重度 | 触发条件 | 修法 |
|---|---|---|---|---|
| 1 | 页面缓存 key 不含 userId | 高（越权） | 多用户访问同一商品 | 缓存与用户无关的部分 |
| 2 | 库存 key 设了 TTL | 高（活动中断） | 活动超过 TTL | TTL 设为 0 + 巡检 |
| 3 | setIfAbsent 当续期用 | 中 | 需要延长会话/锁 | 改用 `EXPIRE` |
| 4 | 反序列化 null 拆箱 | 中（NPE） | 缓存缺失 | 用包装类型 + 判空 |
| 5 | 错误码重复 | 低 | 排障时 | 重新分配 code |
