---
layout:     post
title:      pairs 的遍历顺序
subtitle:   其实本质上还是按照顺序遍历的
date:       2018-02-01
author:     ms2008
header-img: img/post-bg-2015.jpg
catalog:    true
tags:
    - Lua
    - JSHash
    - Algorithm
    - Open Addressing
---

在 Lua 中，我们经常使用 `pairs` 来遍历一个 "hashmap"，但是你有没有想过，`pairs` 遍历的顺序到底依据的是啥？

为了搞清楚这个问题，这里先来做个测试：

> **注：以下测试结果均基于 Lua 5.1.4**

## case 1

```lua
local st = {
    lguafYWz = "1 ",
    BNwGryzZ = "2 ",
    FuKaDkdd = "3 ",
    ZeyjLHcL = "4 ",
    eqkaoYqk = "5 ",
    FrGOtITQ = "6 ",
    FuRZNbsI = "7 ",
    IsqhGXoV = "8 ",
    buYArQCZ = "9 ",
    PZylgsNp = "10",
}

for k,v in pairs(st) do
    print(k)
end
for k,v in pairs(st) do
    print(k)
end
```

在一个 Lua 实例中执行上面的用例，发现两次循环得到的结果是一样的。因此可以得出结论：<u>`pairs` 在遍历的过程中，并不是随机的</u>。

> 在 golang 中，map 每次遍历顺序都是不一致的。是因为 go 的底层实现并不保证这一点。因此，go 索性对 key 次序做随机化，以提醒大家不要依赖 `range` 遍历返回的 key 次序。

## case 2

```lua
local st = {
    lguafYWz = "1 ",
    BNwGryzZ = "2 ",
    FuKaDkdd = "3 ",
    ZeyjLHcL = "4 ",
    eqkaoYqk = "5 ",
    FrGOtITQ = "6 ",
    FuRZNbsI = "7 ",
    IsqhGXoV = "8 ",
    buYArQCZ = "9 ",
    PZylgsNp = "10",
}

for k,v in pairs(st) do
    print(k)
end
```

接着在另外一个 Lua 实例中执行上面的用例，可以看到输出和 case 1 的结果相同。这可以说明：<u>`pairs` 的遍历，也不会和 VM 有关联</u>。

> Lua 5.3 中，为了解决 String Hash Dos 问题，使用了一个随机种子用于字符串哈希值的计算，这个种子是放在全局表中的，会导致相同的字符串在不同的 Lua VM 中，总是有不同的 hash，因此遍历的顺序也是不同的。

***

根据上面的两个测试，我们知道在 Lua 中 `pairs` 的遍历不是随机的。**其实 `pairs` 内部依赖于 `next` 来做表遍历，`next` 则是根据 key 的 hash 值来按顺序遍历的**。

> 严格来说，遍历的顺序是在 hash 之后对 tablesize 取模后的大小。

根据前面的文章，这里可以轻松的算出 key 的遍历顺序：

```lua
local bit = require "bit"

local lshift = bit.lshift
local rshift = bit.rshift
local bxor = bit.bxor

local byte = string.byte
local sub = string.sub
local len = string.len

local function JSHash(str)
    local l = len(str)
    local h = l
    local step = rshift(l, 5) + 1

    for i=l,step,-step do
        h = bxor(h, (lshift(h, 5) + rshift(h, 2) + byte(sub(str, i, i))))
    end

    return h
end

local st = {
    lguafYWz = "1 ",
    BNwGryzZ = "2 ",
    FuKaDkdd = "3 ",
    ZeyjLHcL = "4 ",
    eqkaoYqk = "5 ",
    FrGOtITQ = "6 ",
    FuRZNbsI = "7 ",
    IsqhGXoV = "8 ",
    buYArQCZ = "9 ",
    PZylgsNp = "10",
}
for k,v in pairs(st) do
    print(k, JSHash(k)%16)
end
```

> 注意，这里的 tablesize 表示的是幂次，而不是实际大小。它是不小于 hash 区域数量的 2<sup>n</sup> 形式的整数。

按照大小排列之后：

```
FrGOtITQ    3
FuKaDkdd    4
lguafYWz    4
ZeyjLHcL    6
eqkaoYqk    9
BNwGryzZ    11
IsqhGXoV    13
FuRZNbsI    14
PZylgsNp    15
buYArQCZ    15
```

原生 Lua 5.1.4 中 `pairs` 的结果是：

```
FrGOtITQ
lguafYWz
ZeyjLHcL
eqkaoYqk
PZylgsNp  -- 这种是 hash 冲突导致的，会重新定址。
BNwGryzZ
FuKaDkdd  -- 同上
IsqhGXoV
FuRZNbsI
buYArQCZ
```

可以看到遍历的顺序基本是按照 key 的 hash 值排列的的。<u>不一致的两个是由于 hash 冲突而引起的重新定址（table 用的是闭散列算法）</u>。

值得一提的是：**lua string hash 用的开散列算法；而 table hash 使用的是闭散列算法。**

最后，感谢知乎用户 @PaintDream 关于 `hashpow2` 的指教。
