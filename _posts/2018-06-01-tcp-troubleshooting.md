---
layout:     post
title:      TCP 常见故障排查
subtitle:   TCP Troubleshooting
date:       2018-06-01
author:     ms2008
header-img: img/post-bg-unix-linux.jpg
catalog:    true
mathjax:    true
tags:
    - TCP
typora-root-url: ..
---

TCP 协议相当复杂，并充斥着各种细节。然而 TCP 协议又是如此重要的一个协议，引领风骚三十年，可以说是互联网的奇迹。这些细节正是 TCP 协议成功的原因，并值得我们深入了解。


#### 1. 丢包，错包

对于 `ifconfig` 这个命令，我想大家并不陌生，我们常常用它来查看本机的 IP 地址。但是还有些细节往往容易被忽略，那就是网卡的错包和丢包情况：

![](/img/in-post/tcp-troubleshooting/ifconfig.png)

当然你还可以通过 `ethtool -S ens32 | grep errors` 来获取更多的细节统计。

发生错包的原因有很多，但是一般都是由于网线或者网卡等硬件故障造成。如果你的服务器在换了机房或者网络发生了变更之后，延迟明显增加。这个时候你就要怀疑是不是网卡丢包或者是错包引起的了。另外，还可以通过下面这两个命令开查看每秒的错包和重传情况：

```sh
# 丢包率、错包率
sar -n EDEV 1

# 重传
sar -n ETCP 1
```


#### 2. 队列溢出

我原先专门写过一篇文章介绍 TCP 的两个队列：[捋一捋 backlog 的作用][1]，这里就简单说下，关于细节可以参考前面的链接。

TCP 里有两个队列分别是 SYN Queue 和 Accept Queue，Accept Queue 就是三次握手成功后等待应用 `accept()` 连接的队列。如果瞬间有大量的请求进入或者是应用 `accept()` 取出连接的速度太慢，那么这个队列将会溢出，此时系统会根据 `tcp_abort_on_overflow` 这个内核参数决定是直接丢弃数据包还是发送 RST 给客户端。反映出来的现象就是前端应用无法建立连接。

可以使用 `ss` 来查看当前这个队列的大小：

![](/img/in-post/tcp-troubleshooting/backlog.png)

如上图，当前面 `Recv-Q` 的值接近于 `Send-Q` 时，就说明当前已完成队列快溢出了。

需要注意的是这个队列的大小不能设置的过大，否则会导致前端应用超时，具体细节可以参考[这里][2]。


#### 3. 滑动窗口很小

为了提升服务器的吞吐能力，我们一般都会优化系统的 TCP 缓冲区大小，比如：

```
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65535 16777216
```

内核正是通过这两个参数，进而动态控制滑动窗口（rwnd）的大小。

> rwnd 原先最大的值为 65525(64K)，不过现在内核基本都支持 `tcp_window_scaling`，所以这个大小也就提高到了 1G 字节。其初始值为 20 个 MSS 大小，即 29200 字节。
><br/><br/>
> 与 rwnd 对应的是 cwnd(拥塞窗口)， 在旧一点的内核中，其初始值是 3 个 MSS 大小。后来采取了 Google 的建议提高到了 14600 字节，即 10 个 MSS 大小。

不过有时候你通过 `ss` 观察到缓冲区明明没有满，但是通过抓包后却发现窗口很小：

*下面两张截屏其实并不对应，我这里只是为了说明这个问题。另外 `ss` 输出的 `Recv-Q` 和 `Send-Q` 在不同状态下的含义不同，具体可以参考[这里][3]。*

![](/img/in-post/tcp-troubleshooting/socket_buffer.png)

![](/img/in-post/tcp-troubleshooting/wireshark_win.png)

**其实这里观察到的窗口大小并不是实际大小，实际大小应该是 $calculated\ window\ size$**。如果你打开包的细节可以看到这么一个参数：

![](/img/in-post/tcp-troubleshooting/win_factor.png)

根据前面提到的 `tcp_window_scaling` 特性，正是利用这个值来计算实际窗口的大小，计算公式为：

$$
calculated\ window\ size = (window\ size) \times (scaling\ factor)
$$

这个 $(scaling\ factor)$ 的取值范围是 $2^{0-14}$，并且只通过握手包（SYN）包携带。<u>这里显示的 -1 就是因为没有抓到握手包，所以不知道这个因子是多少</u>。而实际上窗口的大小很有可能大于 $(window\ size)$。

值得一提的是，wireshark 有个功能可以填充这个值，这个在没有抓到握手包的情况下非常有用。

![](/img/in-post/tcp-troubleshooting/fill_factor.png)

> 其实 cwnd 设置为 10 对于高速网络来说还是有些偏低，行业内各大厂商都调整过 cwnd 值，普遍取值在 10-20 之间。


#### 4. 单个数据包大于 MTU

在使用 tcpdump 抓包时，可能会经常看到一些大包，就像下面这样：

![](/img/in-post/tcp-troubleshooting/large_packet.png)

这些包的长度都达到了 8K 大小，为什么没有分片呢？

原因就在于系统开启了 TSO(TCP Segment Offload) 特性。由网卡代替 CPU 实现 packet 的分段和合并，节省系统资源，让系统处理更多的连接。而 tcpdump 工作在网卡和协议栈之间，抓取的是网卡上层的包，所以我们可能会观察到大小超过 MTU 的包：

![](/img/in-post/tcp-troubleshooting/tcpdump_layer.png)

**如果是在交换机端抓取的包肯定都是小于 MTU 的**。

同样系统还有个特性叫 LRO(Large Receive Offload)。会将接收到的数据合并成较大的数据包，然后发送至 TCP/IP 协议栈。所以在接收端也是可以看到大小超过 MTU 的包。

可以使用 `ethtool` 来查看系统的这两个特性是否开启：

![](/img/in-post/tcp-troubleshooting/nic_tso.png)


***

### 参考文献

- [How Can the Packet Size Be Greater than the MTU?](http://packetbomb.com/how-can-the-packet-size-be-greater-than-the-mtu/)
- [Window size scaling factor: -1 (unknown)](https://osqa-ask.wireshark.org/questions/10071/window-size-scaling-factor-1-unknown)

[1]: https://ms2008.github.io/2016/09/06/socket-backlog/
[2]: http://www.cnxct.com/something-about-phpfpm-s-backlog/
[3]: https://stackoverflow.com/questions/36466744/use-of-recv-q-and-send-q
