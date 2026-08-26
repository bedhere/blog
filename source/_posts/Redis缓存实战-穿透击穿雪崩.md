---
title: Redis 缓存实战 — 穿透、击穿、雪崩问题与解决方案
date: 2026-05-21 10:00:00
updated: 2026-05-21 10:00:00
description: 深入剖析 Redis 缓存三大经典问题——缓存穿透、缓存击穿、缓存雪崩的成因与解决方案，涵盖布隆过滤器、互斥锁、逻辑过期等实战编码思路
keywords: [Redis, 缓存穿透, 缓存击穿, 缓存雪崩, 布隆过滤器, 互斥锁, 逻辑过期]
tags:
  - Redis
  - 缓存
  - 后端
categories:
  - [技术, 开发经验]
cover: /img/food1.png
top_img:
---

## 前言

在上一篇文章中，我们聊到了 Redis 的基础数据结构与常用命令。但在实际项目里，Redis 更多是作为**缓存层**挡在数据库前面，扛住高并发流量。然而，"缓存好用，坑也不少"——缓存穿透、缓存击穿、缓存雪崩，这三个问题如果没有处理好，轻则接口响应变慢，重则数据库被打垮导致整个服务雪崩。

本文将从**问题成因 → 解决思路 → 编码落地**三个层面，逐一拆解这三大经典缓存问题。

![Redis缓存问题全景](/img/food1.png)

---

## 一、先理解缓存的基本工作模式

在讲问题之前，先回顾一下缓存的正常读写流程：

```
客户端请求 → 查 Redis 缓存
                ↓
          缓存命中？──是──→ 直接返回数据
                ↓否
          查 MySQL 数据库
                ↓
          将结果写入 Redis（设置过期时间）
                ↓
          返回数据给客户端
```

这个流程看起来完美，但问题就出在**缓存未命中**的这几种特殊情况上。

---

## 二、缓存更新策略与一致性

在深入三个问题之前，有必要先理清缓存的更新策略，因为**策略选错了，本身就是问题的根源**。

### 2.1 三种主流更新策略

| 策略 | 做法 | 优点 | 缺点 |
|------|------|------|------|
| **先删缓存，再更新数据库** | 写操作时先删除 Redis，再写 MySQL | 简单直接 | 并发时可能读到旧数据写回缓存 |
| **先更新数据库，再删缓存** | 写操作时先写 MySQL，再删除 Redis | 数据一致性更好 | 删除失败会导致脏数据 |
| **双写（更新数据库 + 更新缓存）** | 同时更新两边的数据 | 缓存始终有数据 | 并发写容易产生不一致 |

### 2.2 推荐方案：延迟双删

```
① 删除缓存
② 更新数据库
③ 休眠一小段时间（比如 200ms）
④ 再次删除缓存
```

这样即使有并发读在步骤②期间把旧数据写回了缓存，步骤④也会把它清理掉。虽然不能做到绝对的强一致性，但对于大多数业务场景来说，**最终一致性已经足够了**。

> **重要认知**：缓存和数据库的一致性是一个 trade-off。CAP 理论告诉我们，在分布式系统中，一致性和可用性往往不可兼得。使用缓存，本质上就是用"一定程度的最终一致性"换取"极大的性能提升"。

---

## 三、缓存穿透（Cache Penetration）

### 3.1 什么是缓存穿透？

**缓存穿透**是指查询一个**根本不存在的数据**——这个数据在缓存中没有，在数据库中也没有。

```
用户查询 id = -1 的数据
    ↓
查 Redis → 没有（因为根本不存在）
    ↓
查 MySQL → 也没有
    ↓
返回 null，但下次请求还是会查一遍
```

由于这个数据不存在，缓存永远不会被写入，每次请求都会直达数据库。如果有恶意攻击者不断用不存在的 id 发起请求，数据库就会被打垮。

### 3.2 解决方案

#### 方案一：缓存空对象

最直接的思路：既然数据不存在，那就**把"不存在"这个结果也缓存起来**。

```java
public Goods queryById(Long id) {
    // 1. 查缓存
    String key = "goods:" + id;
    String json = redisTemplate.opsForValue().get(key);
    
    // 2. 缓存命中
    if (json != null) {
        // 判断是否是空对象标记
        if ("".equals(json)) {
            return null;  // 命中空对象，直接返回 null
        }
        return JSON.parseObject(json, Goods.class);
    }
    
    // 3. 查数据库
    Goods goods = goodsMapper.selectById(id);
    
    // 4. 写入缓存
    if (goods != null) {
        redisTemplate.opsForValue().set(key, JSON.toJSONString(goods), 
                                       30, TimeUnit.MINUTES);
    } else {
        // 缓存空对象，设置较短的过期时间，防止占用空间
        redisTemplate.opsForValue().set(key, "", 2, TimeUnit.MINUTES);
    }
    
    return goods;
}
```

**要点**：空对象的过期时间要设得短一些（比如 2 分钟），否则大量不存在的 key 会撑爆 Redis 内存。

#### 方案二：布隆过滤器（Bloom Filter）

缓存空对象虽然简单，但如果攻击者不断换不同的不存在的 id，还是会产生大量空缓存。这时候需要**在缓存前面再加一道防线**。

**布隆过滤器**的原理很巧妙：

```
布隆过滤器是一个 bit 数组 + 多个哈希函数
添加元素时：计算多个哈希值，将对应 bit 位置为 1
查询元素时：计算哈希值，如果所有 bit 位都是 1 → 可能存在
                                 如果有一个是 0 → 一定不存在
```

```java
// Guava 实现布隆过滤器
BloomFilter<String> filter = BloomFilter.create(
    Funnels.stringFunnel(Charset.defaultCharset()),
    1000000,    // 预期元素数量
    0.01        // 误判率 1%
);

// 初始化：把所有数据库中的 id 加入布隆过滤器
List<Long> allIds = goodsMapper.selectAllIds();
for (Long id : allIds) {
    filter.put("goods:" + id);
}

// 查询时先过布隆过滤器
public Goods queryByIdWithBloom(Long id) {
    String key = "goods:" + id;
    
    // 布隆过滤器判断：如果说不存在，那一定不存在
    if (!filter.mightContain(key)) {
        return null;  // 直接返回，不再查数据库
    }
    
    // 剩下的逻辑同上...
}
```

```
请求进来
    ↓
布隆过滤器：这个 key 可能存在吗？
    ↓否 → 直接返回 null（拦截掉大部分无效请求）
    ↓是（可能存在）
查 Redis 缓存
    ↓
...
```

**布隆过滤器的特点**：
- ✅ 空间效率极高（一个 bit 存一个元素）
- ✅ 查询速度极快（O(k)，k 为哈希函数个数）
- ⚠️ 有误判率（会误判不存在的为存在，但不会漏判）
- ⚠️ 不能删除元素（删除会影响到其他元素）

---

## 四、缓存击穿（Cache Breakdown）

### 4.1 什么是缓存击穿？

**缓存击穿**是指一个**热点 key** 在过期的一瞬间，大量并发请求同时打到数据库。

```
热点 key "goods:100" 过期了
    ↓
1000 个并发请求同时发现缓存没命中
    ↓
1000 个请求全部打到 MySQL
    ↓
数据库压力瞬间飙升，可能直接宕机
```

和缓存穿透的区别：
- **穿透**：查的是不存在的数据，一直穿透
- **击穿**：查的是存在的数据，只是刚好 key 过期了，并发穿透

### 4.2 解决方案

#### 方案一：互斥锁（Mutex Lock）

核心思路：**只让一个线程去查数据库并重建缓存，其他线程等待**。

```java
public Goods queryByIdWithMutex(Long id) {
    String key = "goods:" + id;
    String lockKey = "lock:goods:" + id;
    
    // 1. 查缓存
    String json = redisTemplate.opsForValue().get(key);
    if (json != null) {
        return JSON.parseObject(json, Goods.class);
    }
    
    // 2. 缓存未命中 → 尝试获取锁
    try {
        // 用 setnx 实现互斥锁，设置 10 秒过期防止死锁
        Boolean locked = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, "1", 10, TimeUnit.SECONDS);
        
        if (Boolean.TRUE.equals(locked)) {
            // 3. 获取锁成功 → 查数据库，重建缓存
            Goods goods = goodsMapper.selectById(id);
            if (goods != null) {
                redisTemplate.opsForValue().set(key, JSON.toJSONString(goods),
                                               30, TimeUnit.MINUTES);
            }
            return goods;
        } else {
            // 4. 没拿到锁 → 休眠一会儿，重新查缓存
            Thread.sleep(50);
            return queryByIdWithMutex(id);  // 递归重试
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        return null;
    } finally {
        // 5. 释放锁
        redisTemplate.delete(lockKey);
    }
}
```

**流程图**：

```
1000 个并发请求发现缓存过期
    ↓
1000 个请求同时竞争互斥锁
    ↓
只有 1 个线程抢到锁 → 查 MySQL，重建缓存
    ↓
其余 999 个线程休眠重试 → 缓存已重建 → 直接命中缓存
```

**互斥锁的优缺点**：
- ✅ 保证了只有一个线程查数据库
- ✅ 实现简单，容易理解
- ⚠️ 其余线程要等待，有一定延迟
- ⚠️ 可能出现死锁（需要设置锁的过期时间）

#### 方案二：逻辑过期（Logical Expiration）

互斥锁的缺点是：拿到锁的线程要同步查数据库，其他线程需要等待。如果数据库查询本身比较慢，用户体验会受影响。

**逻辑过期**的思路则是：**不设真正的过期时间，而是异步更新缓存**。

```java
// 定义带逻辑过期时间的数据包装类
@Data
public class RedisData {
    private LocalDateTime expireTime;  // 逻辑过期时间
    private Object data;               // 实际数据
}

// 查询逻辑
public Goods queryByIdWithLogicalExpire(Long id) {
    String key = "goods:" + id;
    String lockKey = "lock:goods:" + id;
    
    // 1. 查缓存
    String json = redisTemplate.opsForValue().get(key);
    
    // 2. 缓存未命中 → 直接返回 null（热点 key 应该被提前预热）
    if (json == null) {
        return null;
    }
    
    // 3. 反序列化
    RedisData redisData = JSON.parseObject(json, RedisData.class);
    Goods goods = JSON.parseObject((String) redisData.getData(), Goods.class);
    LocalDateTime expireTime = redisData.getExpireTime();
    
    // 4. 判断是否逻辑过期
    if (expireTime.isAfter(LocalDateTime.now())) {
        // 未过期 → 直接返回
        return goods;
    }
    
    // 5. 已过期 → 尝试获取锁，异步更新
    Boolean locked = redisTemplate.opsForValue()
        .setIfAbsent(lockKey, "1", 10, TimeUnit.SECONDS);
    
    if (Boolean.TRUE.equals(locked)) {
        // 开启独立线程异步更新缓存
        threadPoolExecutor.submit(() -> {
            try {
                // 查数据库
                Goods newGoods = goodsMapper.selectById(id);
                // 重建缓存，设置新的逻辑过期时间（比如 30 分钟后）
                RedisData newRedisData = new RedisData();
                newRedisData.setData(JSON.toJSONString(newGoods));
                newRedisData.setExpireTime(LocalDateTime.now().plusMinutes(30));
                redisTemplate.opsForValue().set(key, JSON.toJSONString(newRedisData));
            } finally {
                // 释放锁
                redisTemplate.delete(lockKey);
            }
        });
    }
    
    // 6. 不管拿没拿到锁，都先返回旧数据（保证可用性）
    return goods;
}
```

**逻辑过期方案流程图**：

```
请求发现逻辑过期
    ↓
直接返回旧数据（用户无感知）
    ↓
尝试获取互斥锁
    ↓
拿到锁 → 开启新线程异步查 MySQL 重建缓存
没拿到 → 说明已经有线程在重建了，不用管
```

**逻辑过期的优缺点**：
- ✅ 用户不会阻塞等待，体验更好
- ✅ 旧数据仍然可用，保证高可用
- ⚠️ 实现稍复杂，需要线程池
- ⚠️ 返回的是旧数据（短暂的最终一致性）
- ⚠️ 需要提前预热热点 key

### 4.3 两种方案对比

| 维度 | 互斥锁 | 逻辑过期 |
|------|--------|----------|
| 一致性 | 强一致 | 最终一致（短暂旧数据） |
| 可用性 | 需等待 | 无需等待 |
| 实现复杂度 | 简单 | 中等 |
| 适用场景 | 一致性要求高 | 可用性要求高 |
| 额外开销 | 线程阻塞等待 | 需要线程池异步更新 |

> **选型建议**：如果数据是商品详情、文章内容等对实时性要求不高的场景，推荐逻辑过期；如果是库存扣减、秒杀等对一致性要求极高的场景，推荐互斥锁。

---

## 五、缓存雪崩（Cache Avalanche）

### 5.1 什么是缓存雪崩？

**缓存雪崩**是指**大量 key 同时过期**，或者 **Redis 服务宕机**，导致所有请求瞬间压到数据库。

```
场景一：同一时刻大量 key 过期
凌晨 0 点，所有今日推荐商品的缓存同时过期
    ↓
下一个请求到来，发现全部 miss
    ↓
所有请求全部打到数据库
    ↓
数据库扛不住，崩了

场景二：Redis 宕机
Redis 服务挂了
    ↓
所有请求直接打到数据库
    ↓
数据库也被打崩
    ↓
服务不可用
```

### 5.2 解决方案

#### 方案一：过期时间加随机值

不让 key 在同一时刻过期，给每个 key 的过期时间加上一个随机偏移量。

```java
// 在设置过期时间时，加上随机值
int baseExpire = 30;  // 基础过期时间 30 分钟
int randomOffset = new Random().nextInt(10);  // 随机偏移 0~10 分钟
int actualExpire = baseExpire + randomOffset;

redisTemplate.opsForValue().set(key, value, actualExpire, TimeUnit.MINUTES);
```

这样即使同一批写入的缓存，过期时间也会分散在 30~40 分钟内，避免了集中过期。

#### 方案二：Redis 高可用集群

单机 Redis 宕机是雪崩的最严重原因。需要通过**高可用架构**来保障：

```
               ┌──────────────┐
               │   客户端      │
               └──────┬───────┘
                      │
               ┌──────▼───────┐
               │  Sentinel    │  ← 哨兵监控 + 自动主从切换
               └──────┬───────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
   ┌────▼────┐  ┌────▼────┐  ┌────▼────┐
   │ Master  │  │ Slave 1 │  │ Slave 2 │
   │ (读写)  │  │ (只读)  │  │ (只读)  │
   └─────────┘  └─────────┘  └─────────┘
        │
   Master 挂了 → Sentinel 自动选举 Slave 晋升
```

或者使用 **Redis Cluster** 分片集群，将数据分散在多个节点上。

#### 方案三：多级缓存 + 服务降级

在 Redis 之前再加一层本地缓存（如 Caffeine），即使 Redis 挂了，本地缓存还能扛一阵：

```
请求 → 本地缓存(Caffeine) → Redis → MySQL
        命中则直接返回      命中则返回   最后一道防线
```

```java
// 多级缓存示例
public Goods queryWithMultiLevelCache(Long id) {
    String key = "goods:" + id;
    
    // 1. 查本地缓存（Caffeine）
    Goods goods = caffeineCache.getIfPresent(key);
    if (goods != null) {
        return goods;
    }
    
    // 2. 查 Redis
    try {
        String json = redisTemplate.opsForValue().get(key);
        if (json != null) {
            goods = JSON.parseObject(json, Goods.class);
            caffeineCache.put(key, goods);  // 写入本地缓存
            return goods;
        }
    } catch (Exception e) {
        // Redis 挂了 → 降级：直接查数据库或返回兜底数据
        log.error("Redis 不可用，执行降级策略", e);
        return getFallbackGoods(id);
    }
    
    // 3. 查数据库
    goods = goodsMapper.selectById(id);
    if (goods != null) {
        // 尝试写 Redis（如果 Redis 恢复了）
        try {
            redisTemplate.opsForValue().set(key, JSON.toJSONString(goods),
                                           30 + new Random().nextInt(10),
                                           TimeUnit.MINUTES);
        } catch (Exception ignored) {}
        caffeineCache.put(key, goods);
    }
    
    return goods;
}

// 降级兜底数据
private Goods getFallbackGoods(Long id) {
    Goods goods = new Goods();
    goods.setId(id);
    goods.setName("商品加载中...");
    goods.setDescription("系统繁忙，请稍后再试");
    return goods;
}
```

---

## 六、三大问题总结对比

| 维度 | 缓存穿透 | 缓存击穿 | 缓存雪崩 |
|------|---------|---------|---------|
| **问题根源** | 查询不存在的数据 | 热点 key 过期 | 大量 key 同时过期 / Redis 宕机 |
| **现象** | 每次请求都穿透到 DB | 瞬间大量并发打到 DB | 所有请求同时打到 DB |
| **缓存状态** | 缓存中没有，DB 中也没有 | 缓存过期，DB 中有 | 大量缓存同时过期 |
| **核心方案** | 布隆过滤器 / 缓存空对象 | 互斥锁 / 逻辑过期 | 随机过期时间 / 高可用集群 |
| **难度** | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |

### 问题关系图

```
                    缓存问题全景
                        │
        ┌───────────────┼───────────────┐
        │               │               │
    缓存穿透         缓存击穿         缓存雪崩
   (不存在的数据)   (热点key过期)   (大量key同时过期)
        │               │               │
   布隆过滤器        互斥锁          随机过期时间
   缓存空对象        逻辑过期        Redis高可用
                                  多级缓存+降级
```

---

## 七、实战中的综合建议

在实际项目中，这三种问题往往不会孤立出现。以下是我在实践中总结的几条综合建议：

### 7.1 缓存设计四原则

1. **永远设置过期时间**：不要设永不过期的 key，否则内存会不断增长
2. **过期时间加随机偏移**：把 `+ random(0, 600)` 当作习惯
3. **热点数据提前预热**：上线前或凌晨定时把热点数据加载到缓存
4. **对 DB 做限流保护**：即使缓存全部失效，DB 层也要有最后的兜底限流

### 7.2 监控与告警

```java
// 缓存命中率统计
public class CacheMonitor {
    private AtomicLong hitCount = new AtomicLong(0);
    private AtomicLong missCount = new AtomicLong(0);
    
    public void recordHit() { hitCount.incrementAndGet(); }
    public void recordMiss() { missCount.incrementAndGet(); }
    
    public double getHitRate() {
        long total = hitCount.get() + missCount.get();
        return total == 0 ? 0 : (double) hitCount.get() / total;
    }
}
```

关注这几个指标：
- **缓存命中率**：低于 90% 就要排查
- **数据库 QPS**：突然飙升说明缓存出了问题
- **Redis 内存使用率**：过高可能导致 OOM

### 7.3 一个"比较完整"的缓存查询模板

把以上所有方案整合在一起，一个生产级别的缓存查询方法大概是这样的：

```java
public Goods queryById(Long id) {
    String key = "goods:" + id;
    String lockKey = "lock:goods:" + id;
    
    // ① 布隆过滤器 —— 拦截不存在的数据（防穿透）
    if (!bloomFilter.mightContain(key)) {
        return null;
    }
    
    // ② 查缓存
    String json = redisTemplate.opsForValue().get(key);
    if (json != null && !"".equals(json)) {
        return JSON.parseObject(json, Goods.class);
    }
    if ("".equals(json)) {
        return null;  // 空对象标记
    }
    
    // ③ 缓存未命中 → 互斥锁（防击穿）
    Boolean locked = redisTemplate.opsForValue()
        .setIfAbsent(lockKey, "1", 10, TimeUnit.SECONDS);
    
    if (Boolean.TRUE.equals(locked)) {
        try {
            // ④ 双重检查（Double Check）
            json = redisTemplate.opsForValue().get(key);
            if (json != null) {
                return "".equals(json) ? null : JSON.parseObject(json, Goods.class);
            }
            
            // ⑤ 查数据库
            Goods goods = goodsMapper.selectById(id);
            
            // ⑥ 写入缓存（过期时间 + 随机偏移，防雪崩）
            int expire = 30 + new Random().nextInt(10);
            if (goods != null) {
                redisTemplate.opsForValue().set(key, JSON.toJSONString(goods),
                                               expire, TimeUnit.MINUTES);
            } else {
                // 缓存空对象，短过期时间（防穿透）
                redisTemplate.opsForValue().set(key, "", 2, TimeUnit.MINUTES);
            }
            return goods;
        } finally {
            redisTemplate.delete(lockKey);
        }
    } else {
        // ⑦ 没拿到锁 → 休眠重试
        try { Thread.sleep(50); } catch (InterruptedException ignored) {}
        return queryById(id);
    }
}
```

> 这个模板整合了**布隆过滤器（防穿透）+ 缓存空对象（防穿透）+ 互斥锁（防击穿）+ 随机过期（防雪崩）**，可以作为大多数业务场景的起手式。

---

## 八、总结

Redis 缓存的三大问题，本质上都是在处理"缓存未命中时，如何保护数据库"这件事：

- **缓存穿透**：数据根本不存在 → 用布隆过滤器在源头拦截，用空对象兜底
- **缓存击穿**：热点数据刚好过期 → 用互斥锁控制并发，用逻辑过期提升可用性
- **缓存雪崩**：大量数据同时过期或 Redis 宕机 → 用随机过期分散压力，用高可用集群保证服务不挂

这三个问题的学习路径我用了大约 28 个小时，从理论到编码，从互斥锁到逻辑过期，一路踩了不少坑。希望这篇文章能帮你少走一些弯路。

**记住一句口诀**：
> 穿透用布隆，击穿用锁控，雪崩加随机，高可用保底。

---

> **学习参考**：Redis 缓存三大问题实战  
> **核心知识点**：布隆过滤器 | 互斥锁 | 逻辑过期 | 缓存空对象 | 随机过期 | Redis 高可用  
> **适用场景**：一切使用 Redis 做缓存的高并发业务场景
