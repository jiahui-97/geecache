# GeeCache
## 简介

一个分布式缓存系统

## 进度

- [x] LRU 缓存淘汰策略
- [x] 单机并发缓存
- [x] HTTP 服务端
- [x] 一致性哈希
- [x] 分布式节点
- [x] 防止缓存击穿
- [x] 使用 Protobuf 通信

## 总结
### 1. LRU 缓存淘汰策略
LRU 缓存即最近最少使用淘汰策略，使用双链表加哈希表来实现
需要注意的是，LRU 算法不抗扫描的问题，可以采用 LRU-k 来解决这个问题；另外，还有一种 arc 算法，存放 LRU，LFU 两种缓存队列，可以动态调整两个队列的大小

### 2. 单机并发缓存
上层可能接收到多个并发请求缓存，因此需要底层的 LRU 缓存实现并发安全，那么最简单的方式就是加锁。

从上到下结构为：
1. 实现 group，分 namespace 实现缓存结构，不同 namespace 不冲突，全局变量保存所有 group：var groups = make(map[string]*Group)
2. 每个 group，需要具体实现 get 方法，若 get 到就直接返回，没 get 到就自动插入缓存；
```
                            是
接收 key --> 检查是否被缓存 -----> 返回缓存值 ⑴
                |  否                         是
                |-----> 是否应当从远程节点获取 -----> 与远程节点交互 --> 返回缓存值 ⑵
                            |  否
                            |-----> 调用`回调函数`，获取值并添加到缓存 --> 返回缓存值 ⑶
```

```go
type Group struct {
	name      string    // namespace
	getter    Getter    // getter 方法，由业务方传入，获取 key 对应的 value，一般来讲是请求数据库，代表缓存没查到，需要到数据库里查
	mainCache cache     // LRU cache 具体实现
}
```

3. 实现返回 value 的统一类型 ByteView，存储类型为 []byte，可以兼容所有值类型
4. 下层实现真正调用的 mainCache

```go
type ByteView struct {
	b []byte
}
```

### 3. HTTP 服务端
提供 http 接口，接收 groupName 和 key 参数，调用实现的分布式缓存，并返回结果

### 4. 一致性哈希
在实现分布式节点前，需要先实现一致性哈希方法。在分布式缓存中，来了一个 key，需要快速准确找到其真实存储节点，同时要支持快速扩缩节点数量，解决方案就是一致性哈希方法。

算法步骤：
1. 将 key 映射到 2^32 空间中，数字首尾相连形成环
2. 顺时针寻找第一个比 key 映射的 hash 大的节点，访问该节点
- 节点的哈希值需要提前存储到环中，用一个升序数组来保存每个节点映射出的哈希值
- 服务器节点过少时，会引起数据倾斜问题，可能有大量 key 集中到了一个 server 中，因此一个服务器节点可以创建多个虚拟节点，相当于哈希环上加了分片

### 5. 分布式节点
在一致性哈希之上，实现分布式节点。

对于每个节点来说，需要实现一个 http 客户端，可以请求其它节点获取缓存值

若本地 get 不到，则 PickPeer，如果 peer 是其它节点，就与其它节点交互；如果 peer 是自己，就在自己缓存中添加上这一对 kv，并返回。

其中，PickPeer 需要调用一致性哈希方法

### 6. 防止缓存击穿
实现 singleflight，同一时间如果接收到多个相同的 key，只允许一个请求访问数据库

具体方式就是将当前正在请求的 key 保存到 map 中，每次加锁访问 map，若有 key，则等待结果，若无 key，则请求

### 7. 使用 Protobuf 通信
pb 通信，序列化成二进制，反序列化解析，比 json 性能高

参考：https://geektutu.com/post/geecache.html
