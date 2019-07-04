---
layout:     post
title:      "线上一次大量 CLOSE_WAIT 复盘"
subtitle:   "redigo 的超时机制"
date:       2019-07-04
author:     "ms2008"
header-img: "img/post-bg-golang-blog-hero.jpg"
catalog:    true
tags:
    - Golang
    - TCP
typora-root-url: ..
---

最近，我在压测线上的一个长连接服务时，发现服务端出现大量的 CLOSE_WAIT 状态长时间不会释放，并且伴随着 goroutine 暴增，这里做个复盘，介绍下排查思路。

说起 CLOSE_WAIT，就不得不再复习一遍 TCP 的状态变迁：

![](/img/in-post/TCP-State.png)

出现 CLOSE_WAIT 本质上是因为**服务端收到客户端的 FIN 后，仅仅回复了 ACK（由系统的 TCP 协议栈自动发出），并没有发 4 次断开的第二轮 FIN（由应用主动调用 `Close()` 或 `Shutdown()` 发出）**。

而且观察到服务端的 CLOSE_WAIT 状态 RECV 缓冲区基本都有数据：

![](/img/in-post/server-close_wait.png)

说明服务端还没有调用 `recv` 读取，并且在**客户端关闭连接后，CLOSE_WAIT 依然不会消失，只能说明服务端 HANG 在了某处，没有调用 `close`**。

接下里就需要打印出服务的调用栈信息，看下服务端究竟在干什么：

```
goroutine profile: total 1380684
816132 @ 0x430ecf 0x40766a 0x407640 0x40732b 0x7f1bfc 0x7f0ed7 0x801842 0x9349a8 0x45e091
#	0x7f1bfb	github.com/gomodule/redigo/redis.(*Pool).get+0x8ab	foo/vendor/github.com/gomodule/redigo/redis/pool.go:278
#	0x7f0ed6	github.com/gomodule/redigo/redis.(*Pool).Get+0x36	foo/vendor/github.com/gomodule/redigo/redis/pool.go:179
#	0x801841	foo/tool.POSOnlineUpdate+0x41		foo/tool/redis.go:549
#	0x9349a7	foo/engine.(*Client).reader.func3+0x77	foo/engine/client.go:417

189361 @ 0x430ecf 0x440708 0x924ef4 0x45e091
#	0x924ef3	foo/engine.(*Client).writer+0x103	foo/engine/client.go:777

159055 @ 0x430ecf 0x40766a 0x407640 0x40732b 0x7f1bfc 0x7f0ed7 0x8036cd 0x920ced 0x925778 0x6bd294 0x6bf196 0x6c0568 0x6bc251 0x45e091
#	0x7f1bfb	github.com/gomodule/redigo/redis.(*Pool).get+0x8ab	foo/vendor/github.com/gomodule/redigo/redis/pool.go:278
#	0x7f0ed6	github.com/gomodule/redigo/redis.(*Pool).Get+0x36	foo/vendor/github.com/gomodule/redigo/redis/pool.go:179
#	0x8036cc	foo/tool.UpdateRegisteTTL+0x4c		foo/tool/redis.go:614
#	0x920cec	foo/engine.(*Client).reader+0xadc	foo/engine/client.go:410
#	0x925777	foo/engine.ServeWs+0x357			foo/engine/client.go:841
#	0x6bd293	net/http.HandlerFunc.ServeHTTP+0x43			/usr/local/go/src/net/http/server.go:1995
#	0x6bf195	net/http.(*ServeMux).ServeHTTP+0x1d5			/usr/local/go/src/net/http/server.go:2375
#	0x6c0567	net/http.serverHandler.ServeHTTP+0xa7			/usr/local/go/src/net/http/server.go:2774
#	0x6bc250	net/http.(*conn).serve+0x850				/usr/local/go/src/net/http/server.go:1878
```

可以看到事故发生时，出现了将近 140W 的 goroutine，有将近 100W 都 block 在了 redis 连接的获取上。顺手确认 redis 的连接情况：`ss -tn dport = :6379 | sed 1d | wc -l` 发现 redis 连接池已经占满。理论上也不应该啊，我们的 redis 都是有超时保护的，理论上不应该出现一直 block 的情况。我们的连接池是这么初始化的：

```go
connTimeout := redis.DialConnectTimeout(time.Duration(10) * time.Second)
readTimeout := redis.DialReadTimeout(time.Duration(10) * time.Second)
writeTimeout := redis.DialWriteTimeout(time.Duration(10) * time.Second)

redisPool = &redis.Pool{
	MaxIdle:     conf.MaxIdle,
	MaxActive:   conf.MaxActive,
	Wait:        true,
	IdleTimeout: 240 * time.Second,
	Dial: func() (redis.Conn, error) {
		c, err := redis.Dial("tcp", conf.Addr, connTimeout, readTimeout, writeTimeout)
		if err != nil {
			return nil, err
		}
		return c, err
	},
	TestOnBorrow: func(c redis.Conn, t time.Time) error {
		if time.Since(t) < time.Minute {
			return nil
		}
		_, err := c.Do("PING")
		return err
	},
}
```

> p.s. redigo 初始化连接池的时候如果没有传入 timeout，那么在执行命令时将永远不会超时！！！

业务获取连接时，基本都是这样：

```go
func getFoo() {
	c := redisPool.Get()
	defer c.Close()

	// XXX
}
```

为什么超时保护机制没有生效？只能从 redigo 源码里一探究竟：

```go
func (p *Pool) lazyInit() {
	// Fast path.
	if atomic.LoadUint32(&p.chInitialized) == 1 {
		return
	}
	// Slow path.
	p.mu.Lock()
	if p.chInitialized == 0 {
		p.ch = make(chan struct{}, p.MaxActive)
		if p.closed {
			close(p.ch)
		} else {
			for i := 0; i < p.MaxActive; i++ {
				p.ch <- struct{}{}
			}
		}
		atomic.StoreUint32(&p.chInitialized, 1)
	}
	p.mu.Unlock()
}

// get prunes stale connections and returns a connection from the idle list or
// creates a new connection.
func (p *Pool) get(ctx interface {
	Done() <-chan struct{}
	Err() error
}) (*poolConn, error) {

	// Handle limit for p.Wait == true.
	if p.Wait && p.MaxActive > 0 {
		p.lazyInit()
		if ctx == nil {
			<-p.ch
		} else {
			select {
			case <-p.ch:
			case <-ctx.Done():
				return nil, ctx.Err()
			}
		}
	}
```

原来 redigo 是通过 `p.lazyInit()` 初始化一个 channel 来限制最大连接数的。发生 block 时，几乎全都是阻塞在了 `<-p.ch` 上，还没有走到执行 redis 命令的时刻，也就不会有什么保护机制了。**本质就是 `redisPool.Get()` 获取连接时如果没有执行的到 `dial()`，那么不会有任何超时保护机制**。

问题已经似乎找到了，但是仔细想想也不对。<u>即使一时半会儿拿不到连接，只要时间足够长，其他调用释放掉连接，之后也应该是可以获取到连接的</u>。但是当时的情况是，服务一直 HANG 在那里不动，只能说明当时 redis 连接根本就没有释放。继续分析当时的栈信息，终于找到一处关键的地方：

```
156 @ 0x430ecf 0x40766a 0x407640 0x40732b 0x7f1bfc 0x7f0ed7 0x7ff6cd 0x7fe389 0x921921 0x923ea0 0x45e091
#	0x7f1bfb	github.com/gomodule/redigo/redis.(*Pool).get+0x8ab		foo/vendor/github.com/gomodule/redigo/redis/pool.go:278
#	0x7f0ed6	github.com/gomodule/redigo/redis.(*Pool).Get+0x36		foo/vendor/github.com/gomodule/redigo/redis/pool.go:179
#	0x7ff6cc	foo/tool.MessageShopDel+0x4c			foo/tool/redis.go:424
#	0x7fe388	foo/tool.MessageListShopGet+0x2c8		foo/tool/redis.go:372
#	0x921920	foo/engine.(*Client).RegisterMsgSend+0x100	foo/engine/client.go:473
#	0x923e9f	foo/engine.(*Client).UnAckMessageLoop+0x6f	foo/engine/client.go:656

144 @ 0x430ecf 0x40766a 0x407640 0x40732b 0x7f1bfc 0x7f0ed7 0x7fea39 0x7fe347 0x921921 0x923ea0 0x45e091
#	0x7f1bfb	github.com/gomodule/redigo/redis.(*Pool).get+0x8ab		foo/vendor/github.com/gomodule/redigo/redis/pool.go:278
#	0x7f0ed6	github.com/gomodule/redigo/redis.(*Pool).Get+0x36		foo/vendor/github.com/gomodule/redigo/redis/pool.go:179
#	0x7fea38	foo/tool.MessageShopGet+0x58			foo/tool/redis.go:379
#	0x7fe346	foo/tool.MessageListShopGet+0x286		foo/tool/redis.go:356
#	0x921920	foo/engine.(*Client).RegisterMsgSend+0x100	foo/engine/client.go:473
#	0x923e9f	foo/engine.(*Client).UnAckMessageLoop+0x6f	foo/engine/client.go:656
```

正是 `UnAckMessageLoop` 这个调用占满了所有的 redis 连接并且没有释放。怎么判断出来的呢？我们的 redis 连接池最大配置为 300，而这里的 goroutine 恰好也是 300 个在获取连接（156+144）。继续看下里面是怎么调用的：

```go
func getFoo() {
	c := redisPool.Get()
	defer c.Close()

	// XXX
}

func getBar() {
	c := redisPool.Get()
	defer c.Close()

	getFoo()
}
```

恍然大悟，原来业务里有个嵌套的调用，这可不就是一个 deadlock 嘛。所以当时的状况应该是，并发到 300 个 `getBar()` 的时候，其里面的 `getFoo()` 会因为 MaxActive 的限制永远拿不到连接，间接导致 `getBar()` 也无法释放其连接，导致后续的 `close` 无法被正常调用，产生大量的 `CLOSE_WAIT` 状态同时伴随大量的 goroutine 堆积。

综上，我们在使用 redis 的时候一定要**小心处理这些嵌套调用，以免留下这种 deadlock 隐患**。其实 redigo 也提供了一个更安全的获取连接的接口：`GetContext()`，通过显式传入一个 `context` 来控制 `Get()` 的超时：

```go
func (p *Pool) GetContext(ctx context.Context) (Conn, error) {
	pc, err := p.get(ctx)
	if err != nil {
		return errorConn{err}, err
	}
	return &activeConn{p: p, pc: pc}, nil
}
```

需要注意的是，要想使用这个接口需要 go1.7+。
