---
title: "从一次 K8s Controller 线上 Fatal 聊起：Go Map 的并发陷阱与实战方案"
date: 2026-04-18
draft: false
tags: ["Go", "Map", "Kubernetes", "并发", "中文"]
summary: "生产环境的 K8s controller 突然 fatal error: concurrent map read and map write 崩溃退出，不是 panic，recover 也救不了。从这个事故出发，搞清楚 map 的并发问题、遍历随机性、delete 不缩容等实战中真正会踩的坑。"
weight: 2
ShowToc: true
TocOpen: true
ShowReadingTime: true
---

## 一个半夜的 Fatal 崩溃

你维护的 Kubernetes controller 稳定运行了几个月，突然半夜开始频繁崩溃。日志里只有一行：

```
fatal error: concurrent map read and map write

goroutine 847 [running]:
runtime.throw({0x1a2b3c, 0x23})
    /usr/local/go/src/runtime/panic.go:1077 +0x48
runtime.mapaccess1_faststr(...)
    /usr/local/go/src/runtime/map_faststr.go:21 +0x42c
main.(*Controller).getCache(...)
    /app/controller.go:156 +0x84
```

你马上加了 `recover`，重新部署。结果——**还是崩**。

因为这不是 `panic`，这是 `fatal error`。**Go runtime 检测到并发 map 操作后，直接调用 `throw` 终止进程，`recover` 拦不住。**

这是 Go 设计者的**故意选择**：并发操作 map 会破坏内部数据结构，与其让程序在损坏的数据上继续运行产生不可预测的结果，不如直接崩溃，让你修 Bug。

问题代码长这样：

```go
type Controller struct {
    cache map[string]*Resource  // 缓存最近处理过的资源
}

func (c *Controller) Reconcile(ctx context.Context, req Request) error {
    // 读缓存
    if res, ok := c.cache[req.Name]; ok {
        return c.processFromCache(res)
    }

    // 从 API server 获取
    res, err := c.client.Get(ctx, req.Name)
    if err != nil {
        return err
    }

    // 写缓存
    c.cache[req.Name] = res

    return c.process(res)
}
```

Controller-runtime 默认用多个 worker goroutine 并发调用 `Reconcile`。多个 goroutine 同时读写 `c.cache` 这个 map，触发了 fatal error。

---

## 为什么 Map 不是并发安全的？

Go 团队的设计哲学：**不为所有人买单**。

大多数 map 的使用场景是**单 goroutine 访问**。如果内置 mutex：

- 每次读写都要加锁/解锁，即使只有一个 goroutine
- 性能损失 10-30%（mutex 本身有 atomic 操作开销）
- 无法根据场景选择最优的并发策略（mutex、RWMutex、sync.Map、分片 map）

所以 Go 选择了：**默认不加锁，检测到并发直接崩溃，让开发者自己选择并发策略。**

Runtime 的检测方式很简单——在 map 的内部结构中维护一个 `flags` 标志位。每次写操作开始时设置 `hashWriting` 标志，结束时清除。如果读或写操作发现这个标志位已经被设置，说明有另一个 goroutine 正在写，直接 `throw`（不是 `panic`，`recover` 拦不住）。

---

## 并发方案选择

```go
// 方案一：sync.Mutex（通用方案）
type SafeCache struct {
    mu    sync.RWMutex
    items map[string]*Resource
}

func (c *SafeCache) Get(key string) (*Resource, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    v, ok := c.items[key]
    return v, ok
}

// 方案二：sync.Map（读多写少场景）
var cache sync.Map
cache.Store("key", value)
v, ok := cache.Load("key")

// 方案三：分片 map（高并发场景）
type ShardedMap struct {
    shards [256]struct {
        mu    sync.RWMutex
        items map[string]*Resource
    }
}
```

| 方案 | 适用场景 | 特点 |
|---|---|---|
| `sync.Mutex` / `RWMutex` | 通用 | 简单可靠，读多用 RWMutex |
| `sync.Map` | 读多写少、key 稳定 | 读接近无锁，写较慢 |
| 分片 map | 高并发读写 | 减少锁竞争，实现稍复杂 |

回到我们的 controller bug，最简单的修复是用 `sync.RWMutex`：

```go
type Controller struct {
    mu    sync.RWMutex
    cache map[string]*Resource
}

func (c *Controller) Reconcile(ctx context.Context, req Request) error {
    c.mu.RLock()
    res, ok := c.cache[req.Name]
    c.mu.RUnlock()
    if ok {
        return c.processFromCache(res)
    }

    res, err := c.client.Get(ctx, req.Name)
    if err != nil {
        return err
    }

    c.mu.Lock()
    c.cache[req.Name] = res
    c.mu.Unlock()

    return c.process(res)
}
```

---

## Map 遍历为什么是随机顺序？

```go
m := map[string]int{"a": 1, "b": 2, "c": 3}
for k, v := range m {
    fmt.Println(k, v)  // 每次运行顺序可能不同
}
```

这是 Go 的**故意设计**——map 遍历的起始位置是随机选择的。

为什么？因为 **map 的内部元素顺序本身就不稳定**——扩容会改变元素的存储位置。如果不随机化，开发者可能在某个版本测试发现遍历顺序碰巧稳定，就依赖了这个顺序。等 Go 版本升级、map 内部实现调整，代码就莫名其妙地坏了。

随机化是在**教育层面防御**——让你在开发阶段就发现顺序不稳定，而不是在生产环境。

---

## Delete 不会缩容

```go
m := make(map[string]int)
// 插入 100 万个 key
for i := 0; i < 1000000; i++ {
    m[fmt.Sprintf("key-%d", i)] = i
}
// 删除 99 万个 key
for i := 0; i < 990000; i++ {
    delete(m, fmt.Sprintf("key-%d", i))
}
// 此时 map 只有 1 万个 key，但内存占用和 100 万时差不多！
```

`delete` 只是标记元素为空，不会释放底层内存，也不会收缩桶数量。如果你的场景中 map 会经历"暴增 → 大量删除 → 只剩少量"的生命周期，要么定期创建新 map 替换旧的，要么考虑用其他数据结构。

---

## 扩容机制

Map 元素增多时会触发扩容，但**不是一次性搬迁**，而是**渐进式**——每次读写 map 时，顺带搬迁一两个旧桶，把搬迁成本分摊到后续操作中，避免一次性大停顿。

这和 Redis 的渐进式 rehash 是同一个思路。知道这个概念即可，不需要记住具体的内部数据结构。

---

## 其他实战要点

### Key 必须是 comparable 类型

map 的 key 必须支持 `==` 比较。以下类型不能做 key：

- `slice`
- `map`
- `func`

常用 key 类型：`string`、`int`、`struct`（所有字段都 comparable）。

### Key 类型对性能有影响

`int` 做 key 比 `string` 快——`int` 的哈希计算几乎是一条指令，而 `string` 需要遍历每个字节。如果你的 key 可以用 `int` 表示（比如 ID），优先用 `int`。

---

## 关键结论

- **写 struct 带 map 字段时，先把 mutex 写上去**——"现在只有一个 goroutine 用"不是理由，代码会演化，后来的人不会记得这个假设
- **并发保护默认选 `sync.RWMutex`，不要默认选 `sync.Map`**——sync.Map 用错场景反而更慢，RWMutex 永远不会是错误选择
- **需要稳定顺序输出时，先把 key 排序再遍历**——直接 range map 的顺序每次都变，测试能过不代表生产能过
- **map 经历过暴涨后，即使删光也要用新 map 替换旧的**——否则那块内存永远不会还给 runtime
- **能用 int 做 key 就别用 string**——省掉逐字节哈希的开销，在热路径上差距明显

## 总结

| 知识点 | 核心要点 |
|---|---|
| 并发安全 | 不安全，fatal 不是 panic，recover 救不了 |
| 并发方案 | RWMutex（通用）、sync.Map（读多写少）、分片 map（高并发） |
| 遍历顺序 | 故意随机化，不可依赖 |
| delete | 不缩容，大量删除后内存不释放 |
| 扩容 | 渐进式搬迁，分摊成本 |
| key 类型 | 必须 comparable，int 比 string 快 |

---

*这是「Go 底层原理实战」系列的第二篇。下一篇我们从一个 interface nil 判断的经典坑出发，聊 interface 的底层原理。*
