---
date: '2024-05-11T16:30:06+08:00'
draft: false
title: 'go map和sync.Map'
tags: ['go','go map']
archives: ['2024-05-11']
categories: ["Golang"]
---

## **map**

Go 的 map 是一个 高性能哈希表，支持自动扩容、哈希冲突处理和渐进迁移，但 不是线程安全的，并且遍历无序、不能用不可比较类型做 key。

map 源码（runtime/map.go）

### 基本用法

```go
// 声明但未初始化，值是 nil
var m1 map[string]int

// 初始化一个空 map
m2 := make(map[string]int)

// 使用字面量初始化
m3 := map[string]string{
    "name": "Go",
    "type": "language",
}

// 操作
m := make(map[string]int)
m["a"] = 1         // 添加
v := m["a"]        // 读取
delete(m, "a")     // 删除
_, ok := m["a"]    // 判断 key 是否存在

// 注意
var m map[string]int
m["a"] = 1 // panic: assignment to entry in nil map
```

### 结构

Go 的 map 是由一组 **buckets**（桶）构成的：

- 每个 `bucket` 存储多个键值对（最多 8 个）
- 当 key 哈希后找到某个 bucket，在 bucket 内线性查找(根据哈希值某几位定位数组索引)
- 冲突（哈希冲突）时会链式处理或溢出 bucket

```go
type hmap struct {
    count     int           // 元素个数
    buckets   unsafe.Pointer // 指向 bucket 数组
    B         uint8         // buckets 的数量是 2^B
    oldbuckets unsafe.Pointer // 扩容时旧的 buckets
    ...
}
type bmap struct {
    tophash [8]uint8       // 存储哈希值高位，快速判断
    keys    [8]keytype     // key 数组
    values  [8]valuetype   // value 数组
    overflow *bmap         // 溢出指针（链表结构）
}
```

### 扩容 动态扩容

触发条件:
1. load factor 装载因子超过6.5(元素太多→ 扩容)
2. 删除太多元素(为了避免内存浪费→shrink)

渐进式扩容过程:
**在后续的读写操作中渐进完成的**
**避免高并发访问时一次性扩容的性能抖动**
1. 新分配一个更大的bucket数组
2. 每次访问map,迁移一点旧数据到新bucket,一次一个旧bucket
3. 最终旧buckets被完全清空,完成迁移
4. 系统监控定期扫描map,迁移一部分bucket
    - 如果长时间没人访问 map，也能渐进迁移完毕

### 查找定位

```go
// bucket定位
bucketIndex := hash(key) & (2^B - 1)
b := buckets[bucketIndex]

// 桶内查找
tophash := uint8(hash >> (keyHashBits - 8))  // 提取哈希高位

for b != nil {
    for i := 0; i < 8; i++ {
        if b.tophash[i] != tophash {
            continue // 快速跳过不匹配的槽
        }
        if b.keys[i] == key {
            return b.values[i] // 命中！
        }
    }
    b = b.overflow // 进入下一个溢出桶
}
```

### 使用技巧
| 技巧                                  | 说明                   |
| ------------------------------------- | ---------------------- |
| make(map[K]V, n)                      | 提前指定容量，避免扩容 |
| 遍历大 map 建议用 slice 存 key 再排序 | 保证有序输出           |
| 避免使用 interface{} 做 key           | 会降低性能             |
| map 的值最好是结构体指针              | 减少复制、提升性能     |

interface{} 做 key 的缺点:
- 类型信息比较 → 增加开销
- 无法在编译期发现错误 → 增加风险
- 频繁断言、转换 → 增加代码复杂度

## **sync.Map**

`sync.Map` 是 Go 为高并发场景设计的读多写少优化版 map，采用了读写分离 + lazy promotion 的策略，在读多写少、key 重复的缓存型场景下性能极高。

### 设计理念

空间换时间
- 普通map在读写期间频繁加锁解锁,锁的竞争十分激烈,也就意味着高并发情况下读写性能低下
- sync.Map通过冗余的数据结构read和dirty来实现读写分离,空间换时间
- 空间复杂度是2倍普通map,read和dirty各自维护key,底层使用entry指针,对read的entry进行的修改也能被dirty看到

读写分离
- 通过read和dirty来实现读写分离
- 在读多写少的场景下,大部分读操作在read中完成,无锁,性能非常高
- 写操作在dirty中完成,受mu保护,避免了读写冲突的情况

延迟删除
- 删除操作不会立即生效,而是标记为删除
- 在后续的dirty->read的迁移过程中,会将标记为删除的key从read中移除
- 一次性批量清理,避免频繁的单个删除操作

双检查机制
- 在涉及dirty的操作时,会进行双检查
- 加锁前在read中检查,接着在dirty中检查,确保不会在dirty操作时read的信息被修改

lazy promotion
- 当misses超过阈值时,会将dirty提升为read
- 这样可以避免频繁的dirty->read迁移
- 通过misses来控制迁移时机,控制dirty的大小
    及时清理,及时将最新数据提升到read并发读

### 基本用法

```go
var m sync.Map

// 存储
m.Store(key, value)

// 获取
value, _ := m.Load(key)

// 删除
m.Delete(key)
```

### 结构

```go
type Map struct {
    mu Mutex // 写锁
    read atomic.Value // 存仅读数据，原子操作，并发读安全，实际存储readOnly类型的数据
    dirty map[interface{}]*entry // 写操作的临时 map
    misses int // 用于决定何时把 dirty 提升为 read
}

// readOnly 存储map中仅读数据的结构体
type readOnly struct {
	m       map[interface{}]*entry // 其底层依然是个map
	amended bool                   // 标志位，标识m.dirty中存储的数据是否和m.read中的不一样，flase 相同，true不相同
}
```

| 字段   | 说明                                                            |
| ------ | --------------------------------------------------------------- |
| read   | 只读部分，多个 goroutine 可无锁并发访问                         |
| dirty  | 写时更新的 map，修改都在这，受 mu 保护                          |
| misses | 如果读 map 查不到 key，累加；超过阈值后，会把 dirty 升级为 read |
| entry  | 是对值的封装，支持 deleted 状态处理                             |

读取流程:
1. 从 read 中读取
2. 如果 read 中没有,加锁，从 dirty 中读取, misses++
3. 如果 dirty 中没有，返回 nil

写入流程:
1. 加锁
2. 写入 dirty
3. 解锁

更新流程:
1. read 中存在有效的对应key,原子更新
2. read 中不存在key,加锁,写入 dirty
3. 解锁

删除流程:
1. read 中存在有效的对应key,标记删除
2. read 中不存在key,加锁,从 dirty 中删除
3. 解锁

提升 dirty -> read 流程:
1. misses达到阈值
2. 加锁
3. 遍历 dirty,清理将删除状态的kv
4. 将dirty提升为read,dirty置空
5. 维护其他信息,misses,amended等
6. 解锁

### 不适用场景
- kv数量较少, 性能不如普通map
- 大量写操作(新增,read无效key更新)（性能不如锁）
- 内存占用敏感的场景, 想控制 map 大小 / 回收机制
- 频繁 `Range()` 操作, 会复制kv而不是直接返回原始数据, 大量的临时内存分配
- key 分布高度随机（miss 很多，dirty 不断 rebuild）

### 其他参考

[鸟窝 Go 1.9 sync.Map揭秘](https://colobu.com/2017/07/11/dive-into-sync-Map/)

[咖啡色的羊驼 由浅入深聊聊Golang的sync.Map](https://juejin.cn/post/6844903895227957262#heading-10)
