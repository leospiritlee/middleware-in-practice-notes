# 三、键设计：前缀模式

这一章讲项目里一个很值得学的设计——`KeyPrefix` 体系。

### 问题：key 散落各处

不用前缀的话，代码里到处是这种字符串：

```java
jedis.set("user:" + id, json);
jedis.set("goods:" + id, json);
jedis.set("seckill:stock:" + id, String.valueOf(stock));
jedis.decr("seckill:stock:" + id);
```

问题很明显：拼错了不报错、改前缀要全文搜索、TTL 分散在各处、没法统一管理。

### 项目的做法：三层抽象

**第一层：接口**

```java
public interface KeyPrefix {
    int expireSeconds();          // 过期时间，0 表示永不过期
    String getPrefix();           // 前缀字符串
}
```

**第二层：抽象基类**

```java
public abstract class BasePrefix implements KeyPrefix {
    private int expireSeconds;
    private String prefix;

    public BasePrefix(String prefix) {          // 默认永不过期
        this(0, prefix);
    }

    public BasePrefix(int expireSeconds, String prefix) {
        this.expireSeconds = expireSeconds;
        this.prefix = prefix;
    }

    @Override public int expireSeconds() { return expireSeconds; }

    @Override public String getPrefix() {
        String className = getClass().getSimpleName();
        return className + ":" + prefix + ":";
    }
}
```

注意 `getPrefix()` 里用了 `getClass().getSimpleName()`——**类名自动成为前缀的一部分**。`GoodsKey` 类生成的 key 一定以 `GoodsKey:` 开头，不可能和 `UserKey` 撞。

**第三层：具体 Key 类**

```java
public class GoodsKey extends BasePrefix {
    public GoodsKey(int expireSeconds, String prefix) { super(expireSeconds, prefix); }

    public static GoodsKey getGoodsList   = new GoodsKey(60, "gl");
    public static GoodsKey getGoodsDetail = new GoodsKey(60, "gd");
    public static GoodsKey getSeckillGoodsStock = new GoodsKey(0, "gs");
}
```

最终生成的 key 形如：

```
GoodsKey:gl:      → 商品列表缓存，60 秒过期
GoodsKey:gd:123   → 商品 123 的详情，60 秒过期
GoodsKey:gs:123   → 商品 123 的秒杀库存，永不过期
```

::: tip 这个设计好在哪
**TTL 和 key 绑在一起。** 你看到 `GoodsKey.getGoodsList` 就知道它 60 秒过期，不用去翻调用处。改 TTL 只改一行。

**前缀自动带类名。** 不需要人为约定命名规范，类名就是命名空间。

**类型安全。** 调用 `redisService.get(GoodsKey.getGoodsDetail, "" + id, Goods.class)`，key 是对象不是字符串，拼错编译不过。
:::

### 但这里有个真实缺陷

`getPrefix()` 用的是 `getClass().getSimpleName()`。如果两个不同包的类**同名**（比如 `com.a.UserKey` 和 `com.b.UserKey`），前缀会完全一样，key 冲突。

更隐蔽的是：这个前缀在**序列化/反序列化、跨服务共享**时是隐式契约。A 服务升级改了类名，B 服务就读不到了。

生产环境的做法一般是把前缀写成**显式常量**，或者加应用名前缀：

```java
public String getPrefix() {
    return appName + ":" + className + ":" + prefix + ":";
}
```
