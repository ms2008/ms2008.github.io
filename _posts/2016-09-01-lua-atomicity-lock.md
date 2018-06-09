---
layout:     post
title:      OpenResty 中的 Atomicity & Lock
subtitle:   OR 中的原子性和锁
date:       2016-09-01
author:     ms2008
header-img: img/post-bg-re-vs-ng2.jpg
catalog:    true
tags:
    - OpenResty
    - Atomicity
    - Lock
---

先来引用官方的描述：

> Atomicity is only guaranteed on the method call level. That is, "get" is atomic, "set" is atomic, but the calling sequence of "get" and "set" is not. If you want to lock a sequence of calls, you have to emulate a high-level lock yourself as discussed here:
><br><br>
> You can use ngx.shared.DICT objects' "add" and "delete" methods to emulate a custom lock, as in:
><br><br>
>```lua
> local key = "updating"
>
> -- the exptime, 60 sec, is to prevent dead-locking
> local ok, err = mystore:add(key, 1, 60)
> if not ok then
>     if err == "exists" then
>         -- some other nginx worker is already updating it, so we give up
>     else
>         -- some other error happens, handle it here
>     end
> else
>     -- go update the data in mystore, you'd better use pcall here to prevent crashing in the middle
>     mystore:delete(key)
> end
>```

`ngx.shared.DICT` 的操作都是 “原子的”，只有一个操作序列才会不能保证原子性(默认使用 nginx 自己的 spin_lock )。

> It uses the lock mechanism provided by the nginx core to ensure atomicity and consistency. Basically it uses spinlocks to try first, failing that, falling back to semaphores.

lua-resty-lock 是使用 `ngx.shared.DICT.add` 来实现的全局锁，其与传统的进程间互斥锁的模型是有区别的:

-  DICT 借助 add 类的操作，单个操作本身隐含了 init 和 lock 两个语义
- Mutex 在 init 成功之后，再对已经分配到的锁进行 trylock/lock，试图改变它的状态，以占有它

问题的关键就在于：

<u>DICT 的抢锁，在 init 阶段也是可能失败的，这时应该返回分配失败</u>

DICT 和 lrucache 的区别：

- DICT 使用的是共享内存，每次操作都是全局锁。如果高并发环境，不同 worker 之间容易引起竞争。所以单个 shared.dict 的体积不能过大。
- lrucache 是 worker 内使用的，由于 nginx 是单进程方式存在，所以永远不会触发锁。并且没有 shared.dict 的体积限制。

lurcache，效率更高，但是不同 worker 之间不共享，同一缓存数据可能被冗余存储。

最后需要注意的是，对于 DICT 来说，`flush_all` 并不会实际释放共享内存的空间，它只是把所有的元素标记为过期而已。`flush_expired` 会实际释放过期了的元素。`delete` 也会释放内存，和 `set(key, nil)` 操作等价。
