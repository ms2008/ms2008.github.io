---
layout:     post
title:      tail call 到底有啥用？
subtitle:   来体验下 proper tail call 强大的威力
date:       2016-09-09
author:     ms2008
header-img: img/post-bg-universe.jpg
catalog:    true
tags:
    - Lua
    - Recursive
    - tail call
---

在聊今天这个话题之前，我们需要知道什么叫 tail call。先来看下，lua 程序设计是怎么定义的：

> 尾调用是一种类似在函数结尾的 goto 调用，当函数最后一个动作是调用另外一个函数时，我们称这种调用尾调用。例如：
><br><br>
>function f(x)<br>
>&nbsp;&nbsp;&nbsp;&nbsp;return g(x)<br>
>end<br>
><br>
> g 的调用是尾调用。
><br><br>
> 例子中 f 调用 g 后不会再做任何事情，这种情况下当被调用函数 g 结束时程序不需要返回到调用者 f；所以尾调用之后程序不需要在栈中保留关于调用者的任何信息。一些编译器比如 Lua 解释器利用这种特性在处理尾调用时不使用额外的栈，我们称这种语言支持正确的尾调用。
><br><br>
> 由于尾调用不需要使用栈空间，那么尾调用递归的层次可以无限制的。

是不是有些迷糊？那我继续来解释下「[详情可以参考廖雪峰的blog](https://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/00137473836826348026db722d9435483fa38c137b7e685000)」：

我们知道在函数内部，可以调用其他函数。如果一个函数在内部调用自身本身，这个函数就是递归函数。递归函数的优点是定义简单，逻辑清晰。**但是使用递归函数有个要命的缺点就是需要注意防止栈溢出！**

在计算机中，函数调用是通过栈（stack）这种数据结构实现的，每当进入一个函数调用，栈就会加一层栈帧，每当函数返回，栈就会减一层栈帧。由于栈的大小不是无限的，所以，递归调用的次数过多，会导致栈溢出。可以试试这个求阶乘的用例：

```lua
do
    -- recursive
    local function fact(n)
        if n == 1 then
            return 1
        else
            return n * fact(n-1)
        end
    end

    print(fact(5))
    print(fact(10000))
    print(fact(1000000))
end

-- output:
-- 120
-- inf
-- stdin:7: stack overflow
-- stack traceback:
--     stdin:7: in function 'fact'
--     stdin:7: in function 'fact'
--     stdin:7: in function 'fact'
--     stdin:7: in function 'fact'
--     stdin:7: in function 'fact'
--     stdin:7: in function 'fact'
--     stdin:7: in function 'fact'
--     stdin:7: in function 'fact'
--     stdin:7: in function 'fact'
--     stdin:7: in function 'fact'
--     ...
--     stdin:7: in function 'fact'
--     stdin:7: in function 'fact'
--     stdin:7: in function 'fact'
--     stdin:7: in function 'fact'
--     stdin:7: in function 'fact'
--     stdin:7: in function 'fact'
--     stdin:7: in function 'fact'
--     stdin:7: in function 'fact'
--     stdin:13: in main chunk
--     [C]: ?
```

解决递归调用栈溢出的方法是通过尾递归优化。尾递归是指，在函数返回的时候，调用自身本身，并且，`return` 语句不能包含表达式。这样，编译器或者解释器就可以把尾递归做优化，使递归本身无论调用多少次，都只占用一个栈帧，不会出现栈溢出的情况。

上面的 `fact(n)` 函数由于 `return n * fact(n - 1)` 引入了乘法表达式，所以就不是尾递归了。要改成尾递归方式，需要多一点代码，主要是要把每一步的乘积传入到递归函数中：

```lua
do
    -- proper tail recursion
    local function fact(n)
        return fact_iter(n, 1)
    end

    function fact_iter(n, now)
        if n <= 1 then
            return now
        else
            return fact_iter(n-1, now*n)
        end
    end

    print(fact(5))
    print(fact(10000))
    print(fact(1000000))
end

-- output:
-- 120
-- inf
-- inf
```

可以看到，`return fact_iter(n-1, now*n)` 仅返回递归函数本身，`n-1` 和 `now*n` 在函数调用前就会被计算，不影响函数调用。

**任何递归函数都存在栈溢出的问题**。尾递归调用时，如果做了优化，栈不会增长，因此，无论多少次调用也不会导致栈溢出。这也正是 tail call 的威力所在。

遗憾的是，大多数编程语言没有针对尾递归做优化，当然 lua 除外 :-)
