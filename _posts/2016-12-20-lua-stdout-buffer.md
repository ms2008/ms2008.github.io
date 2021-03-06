---
layout:     post
title:      Lua 的标准输出与缓存
subtitle:   ""
date:       2016-12-20
author:     ms2008
header-img: img/post-bg-rwd.jpg
catalog:    true
tags:
    - Lua
    - libc
    - 系统调用
---

最近我遇到了个奇怪的问题，我的一个 Lua 脚本需要通过 shell 的重定向将输出追加到一个日志文件中。但是那个 Lua 脚本的输出在日志文件里看来却不是实时的，输出的文本直到脚本结束时才能看到。

在 shell 下运行这个程序，是可以看到实时输出的：

```lua
-- buffer_test.lua
local socket = require "socket"

local const = 100

for i=1, const, 1 do
    print(i)
    socket.select(nil, nil, 1)
end
```

但是当通过重定向时，只有脚本结束后才能看到文件：

```sh
# lua buffer_test.lua >>log.txt 2>&1 &
# tail -f log.txt
```

看来当 Lua 的标准输出 stdout 连接的是终端时，采用了行缓存模式，而重定向到文件时则变成了完全缓存。翻遍了 Lua 的官方文档也没有找到这样的说明。但是在查看 stdio 的手册页时发现了下面的一段话：

> The stdio library is a part of the library libc and routines are auto-matically loaded as needed by the compilers cc(1) and pc(1).
> <br><br>
> At program startup, three text streams are predefined and need not be opened explicitly — standard input (for reading conventional input), — standard output (for writing conventional input), and standard error (for writing diagnostic output). These streams are abbreviated stdin,stdout and stderr. <span style="background-color: #FFFB00;">When opened, the standard error stream is not fully buffered; the standard input and output streams are fully buffered if and only if the streams do not to refer to an interactive device.</span>
> <br><br>
> Output streams that refer to terminal devices are always line buffered by default;

原来 stdio 都是由 libc 提供，而我在 `ldd lua` 时发现 Lua 确实也依赖于 libc。这下就可以解释了：Lua 在连接是终端的时候采用的是行缓存，而连接的是非活跃的设备时是采用的是完全缓存。

但是假如我们中途终止脚本，查看日志：

```sh
# nl log.txt
 1	lua: io.lua:9: interrupted!
 2	stack traceback:
 3		[C]: in function 'select'
 4		io.lua:9: in main chunk
 5		[C]: ?
 6	1
 7	2
 8	3
```

可以发现异常的日志难道不应该在最后面吗？其实上面的引用已经帮我们回答了：stderr 并不是完全缓存，当发生异常时，stderr 首先被写入日志，接着缓存区的 stdout 才会被刷入文件。

**回顾这个问题，给我们留下的经验是 Lua 的很多库的实现原理其实在 libc 与系统调用那儿，不要只把目光局限在 Lua 的文档上。其实在其他的语言中，我估计也差不多。我们在查找问题的时候，一定要跳出自己的固有思维，有时候自己非常有把握的知识恰恰是不准确的。**
