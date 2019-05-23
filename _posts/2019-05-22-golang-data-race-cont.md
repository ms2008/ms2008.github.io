---
layout:     post
title:      "谈谈 Golang 中的 Data Race（续）"
subtitle:   "编译器是可以优化的, 并且没有 memory barrier"
date:       2019-05-22
author:     "ms2008"
header-img: "img/post-bg-golang-blog-hero.jpg"
catalog:    true
tags:
    - Golang
    - Thread safety
    - Concurrent
    - Memory Barrier
---

我在上一篇文章中曾指出：**在 Go 的内存模型中，有 race 的 Go 程序的行为是未定义行为，理论上出现什么情况都是正常的**。并尝试通过一段有 data race 的代码来说明问题：

```go
package main

import (
    "fmt"
    "runtime"
    "time"
)

var i = 0

func main() {
    runtime.GOMAXPROCS(2)

    go func() {
        for {
            fmt.Println("i is", i)
            time.Sleep(time.Second)
        }
    }()

    for {
        i += 1
    }
}
```

当通过 `go run cmd.go` 执行时，大概率会得到下面这样的输出：

```
i is: 0
i is: 0
i is: 0
i is: 0
```

然而有些同学提到：之所以输出 0 是因为 `i += 1` 所在的 goroutine 没有新的栈帧创建，因此没有被调度器调度到。解释似乎也合理，但是事实却不是这样的。真实的原因是：**编译器把那段自增的 `for` 循环给全部优化掉了**。

要验证这一点，我们要先从编译器优化说起。传统的编译器通常分为三个部分，前端(frontEnd)，优化器(Optimizer)和后端(backEnd)。在编译过程中，前端主要负责词法和语法分析，将源代码转化为抽象语法树；优化器则是在前端的基础上，对得到的中间代码进行优化，使代码更加高效；后端则是将已经优化的中间代码转化为针对各自平台的机器代码。

go 的编译器也一样，在生成目标代码的时候会做很多优化，重要的有：

- 指令重排
- 逃逸分析
- 函数内联
- 死码消除

当我们通过:

```sh
go build cmd.go
go tool objdump -s main.main cmd
```

查看编译出的二进制可执行文件的汇编代码：

```asm
  cmd.go:11		0x4858c0		64488b0c25f8ffffff	MOVQ FS:0xfffffff8, CX
  cmd.go:11		0x4858c9		483b6110		CMPQ 0x10(CX), SP
  cmd.go:11		0x4858cd		7635			JBE 0x485904
  cmd.go:11		0x4858cf		4883ec18		SUBQ $0x18, SP
  cmd.go:11		0x4858d3		48896c2410		MOVQ BP, 0x10(SP)
  cmd.go:11		0x4858d8		488d6c2410		LEAQ 0x10(SP), BP
  cmd.go:12		0x4858dd		48c7042402000000	MOVQ $0x2, 0(SP)
  cmd.go:12		0x4858e5		e83605f8ff		CALL runtime.GOMAXPROCS(SB)
  cmd.go:14		0x4858ea		c7042400000000		MOVL $0x0, 0(SP)
  cmd.go:14		0x4858f1		488d05a8640300		LEAQ 0x364a8(IP), AX
  cmd.go:14		0x4858f8		4889442408		MOVQ AX, 0x8(SP)
  cmd.go:14		0x4858fd		e89eaafaff		CALL runtime.newproc(SB)
  cmd.go:22		0x485902		ebfe			JMP 0x485902
  cmd.go:11		0x485904		e8e79afcff		CALL runtime.morestack_noctxt(SB)
  cmd.go:11		0x485909		ebb5			JMP main.main(SB)
```

显然，下面这一段直接被优化没了：

```go
for {
    i += 1
}
```

why? 因为这段代码是有竞态的，没有任何同步机制。go 编译器认为这一段是 dead code，索性直接优化掉了。

而当我们通过 `go build -race cmd.go` 编译后：

```asm
  cmd.go:22		0x4d3430		488d05c9211100		LEAQ main.i(SB), AX
  cmd.go:22		0x4d3437		48890424		MOVQ AX, 0(SP)
  cmd.go:22		0x4d343b		e8d096faff		CALL runtime.raceread(SB)
  cmd.go:22		0x4d3440		488b05b9211100		MOVQ main.i(SB), AX
  cmd.go:22		0x4d3447		4889442410		MOVQ AX, 0x10(SP)
  cmd.go:22		0x4d344c		488d0dad211100		LEAQ main.i(SB), CX
  cmd.go:22		0x4d3453		48890c24		MOVQ CX, 0(SP)
  cmd.go:22		0x4d3457		e8f496faff		CALL runtime.racewrite(SB)
  cmd.go:22		0x4d345c		488b442410		MOVQ 0x10(SP), AX
  cmd.go:22		0x4d3461		48ffc0			INCQ AX
  cmd.go:22		0x4d3464		48890595211100		MOVQ AX, main.i(SB)
```

可以明显看到有 `INCQ` 指令了，这是因为 `-race` 选项打开了 data race detector 用来检查这个错误而关闭了相关的编译器优化：

```
==================
WARNING: DATA RACE
Read at 0x0000005e5600 by goroutine 6:
  main.main.func1()
      /root/gofourge/src/lab/cmd.go:16 +0x63

Previous write at 0x0000005e5600 by main goroutine:
  main.main()
      /root/gofourge/src/lab/cmd.go:22 +0x7b

Goroutine 6 (running) created at:
  main.main()
      /root/gofourge/src/lab/cmd.go:14 +0x4f
==================
i is: 4085
i is: 56001323
i is: 112465799
i is: 168640611
```

如此，运行结果就“看似正确”了。

最后再引用一句 golang-nuts 上的评论：

> Any race is a bug. When there is a race, the compiler is free to do whatever it wants.

### 参考资料

- [Go compiler - Loop transformations](https://groups.google.com/d/topic/golang-nuts/jVM0VhsPM04/discussion)
- [Would this race condition be considered a bug?](https://groups.google.com/d/topic/golang-nuts/HUfe1iGbo1w/discussion)