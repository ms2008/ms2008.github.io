---
layout:     post
title:      "关于 etcd 的一些谣言"
subtitle:   "纸上谈兵"
date:       2019-12-04
author:     "ms2008"
header-img: "img/post-bg-golang-blog-hero.jpg"
catalog:    true
tags:
    - Golang
    - etcd
---

### 1. 分区脑裂

这是一个被广为流传的误解，众所周知 etcd 使用 Raft 协议来解决数据一致性问题。一个 Raft Group 只能有一个 Leader 存在，如果一旦发生网络分区，Leader 只会在多数派一边被选举出来，而少数派则全部处于 Follower 或 Candidate 状态，所以一个长期运行的集群是不存在脑裂问题的。etcd 官方文档也明确了这一点：

> The majority side becomes the available cluster and the minority side is unavailable; there is no “split-brain” in etcd.

但是有一种特殊情况，假如旧的 Leader 和集群其他节点出现了网络分区，其他节点选出了新的 Leader，但是旧 Leader 并没有感知到新的 Leader，那么此时集群可能会出现一个短暂的「**双 Leader**」状态。这种情况并不能称之为脑裂，原因有二：

1. 这不是一个长期运行状态，维持时间不会超过一个投票周期
2. etcd 也有 `ReadIndex`、`Lease read` 机制来解决这种状态下的一致性语义问题

### 2. 节点数不能是偶数

这也是错误的，集群是需要一半以上节点才能正常工作的。例如创建 4 个节点，半数以上正常节点数是 3。也就是最多只允许 1 台机器 down 掉。而 3 台节点，半数以上正常节点数是 2，也是最多允许 1 台机器 down 掉。4 个节点，多了 1 台机器的成本，但是健壮性和 3 个节点的集群一样，基于成本的考虑也是不推荐的。

另外，etcd 集群是一个 Raft Group，没有 shared。所以它的极限有两部分，一是单机的容量限制，内存和磁盘；二是网络开销，每次 Raft 操作需要所有节点参与，节点越多性能越低。所以扩展很多 etcd 节点是没有意义的，一般是 3、5、7，再多感觉也没意义了。

### 3. 分区后，少数派依然可以工作

这个描述并不准确，分区后少数派全部处于 Follower 或 Candidate 状态，所以肯定无法执行写操作（CP 系统，无法保证可用性）。那么读请求呢？答案是：**默认无法读取！**

etcd 默认使用 `ReadIndex` 的机制来实现线性一致性读。<u>其原理就是 etcd 在处理读请求时，那么它必须先要向 Leader 去查询 commit index，此时少数派联系不上 Leader，所以必然会失败。但是这时候是支持非一致性读请求的</u>。不过需要开启一个选项：

- SDK 访问需要配置 `WithSerializable` 选项（默认并不开启）
- `etcdctl` 访问需要配置 `--consistency=s` 选项

### 4. 每个 revision 由 `{mainID, subID}` 唯一标识

这个描述不算错，但要看站在哪个位置。在数据库领域，面对高并发环境下数据冲突的问题，业界常用的解决方案有两种：

1. 悲观锁，常见的实现有读写锁（Read/Write Locks）、两阶段锁（Two-Phase Locking）等
2. 乐观锁，常见的实现有逻辑时钟（Logical Clock）、MVCC（Multi-version Cocurrent Control）等

etcd 选择的也是 MVCC 来实现，revision 即对应到 MVCC 中的版本：

```go
// A revision indicates modification of the key-value space.
// The set of changes that share same main revision changes the key-value space atomically.
type revision struct {
	// main is the main revision of a set of changes that happen atomically.
	main int64

	// sub is the sub revision of a change in a set of changes that happen
	// atomically. Each change has different increasing sub revision in that
	// set.
	sub int64
}
```

etcd 中的每一次 key-value 的操作都会有一个相应的 revision。这里的 `main` 属性对应事务 ID，全局递增不重复，它在 etcd 中被当做一个逻辑时钟来使用。`sub` 代表一次事务中不同的修改操作（如 `put` 和 `delete`）编号，从 `0` 开始依次递增。所以在一次事务中，每一个修改操作所绑定的 revision 依次为 `{txID, 0}`, `{txID, 1}`, `{txID, 2}` ...

所以站在客户端的角度来看，一个事务中的所有写操作，共享同一个 revision（main ID）；站在 etcd 服务端的角度来看，一个事务中的所有写操作，其实是不同的 `{mainID, subID}` 对。

### 5. 新 Leader 会无条件提交旧 Leader 日志

这个问题比较复杂，先上结论：**新 Leader 不能直接 commit 前任留下的日志，否则会导致 Raft 算法错误**。

其实这里有 4 种情况：

1. Leader 复制给少数节点，然后宕机
2. Leader 复制给多数节点，然后宕机
3. Leader 复制给多数节点，本地提交成功，返回客户端成功，然后宕机
4. Leader 复制给多数节点，本地提交成功，返回客户端成功，广播 commit 给了多数节点，然后宕机

场景 1-2 压根没有给客户端承诺，所以是新 Leader 不会立即 commit 前任 Leader 的日志；场景 3-4 承诺了客户端，无论如何日志是不允许丢的，所以新 Leader 一定会 commit 日志。

### 6. 不适合存储大规模数据

这个其实还是要看数据规模，先看下 etcd 官方的定位：

> Distributed reliable key-value store for the most critical data of a distributed system

其定位是一个分布式存储，并不仅仅是存一些元数据而已，实际上早期 etcd 官方推荐的存储空间的确是 2GB。这是因为 etcd 存储层可以看成由两部分组成，一层在内存中的基于 btree 的索引层，一层基于 boltdb 的磁盘存储层。boltdb 的内部使用 B+ tree 作为存储数据的数据结构，叶子节点存放具体的真实存储键值。当数据写入时，其内部 B+ tree 结构会频繁发生调整(如再平衡，分裂合并树的节点)，这个是其性能的瓶颈所在。

好在阿里已经优化了这部分的操作，并且合并到了 etcd 官方，他们给出来的测试结论是即使存储规模到达 100GB 的数据量，读写延迟也无显著增长，和存储 2GB 一样流畅。

### 参考文献

- [CAP一致性协议及应用解析][1]
- [etcd-raft的线性一致读方法一：ReadIndex][2]
- [Etcd 架构与实现解析][3]
- [MVCC 在 etcd 中的实现][4]
- [etcd使用经验总结][5]
- [raft本质理解][6]
- [etcd 在超大规模数据场景下的性能优化][7]

[1]: https://tech.youzan.com/cap-coherence-protocol-and-application-analysis/
[2]: https://zhuanlan.zhihu.com/p/31050303
[3]: http://jolestar.com/etcd-architecture/
[4]: https://blog.betacat.io/post/mvcc-implementation-in-etcd/
[5]: https://alexstocks.github.io/html/etcd.html
[6]: https://yuerblog.cc/2018/07/28/understand-raft/
[7]: https://segmentfault.com/a/1190000019185217
