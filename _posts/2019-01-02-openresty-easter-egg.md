---
layout:     post
title:      "OpenResty Con 2017 中的一个彩蛋"
subtitle:   "welcome to the future"
date:       2019-01-02
author:     "ms2008"
header-img: "img/post-bg-binary.jpg"
catalog:    true
tags:
    - OpenResty
    - Lua
typora-root-url: ..
---

上周的[「Ant Design」][1]圣诞节彩蛋事件确实炸开了锅，我相信加彩蛋的初衷是好的，只是这次玩过了火。最后搞得比较重，作者不得不出来发布道歉。其实在开源软件中，加彩蛋是一种乐趣，并不为奇。那为什么这个事件会成为一个反例？我觉得[@依云][2]的看法就很好：

> **在程序库中加入未预期的行为，是十分不负责任的表现。**
><br/><br/>
> 库应当提供机制而非策略，并且具有良好定义的行为。软件中彩蛋这种东西由来已久，为什么这次影响这么大呢？其根本原因不在于它是开源软件，也不在于使用广泛，而是在于——它是库。库能不能提供彩蛋呢？是可以的，只要它是以 opt-in 的形式提供的，并且有文档明确其行为，使用方需要显式启用就没有任何问题。库的作者不需要知道[圣诞节][3]还可能在1月，也不需要知道代码是运行在哪个国家，他的职责应当是提供清晰明确的行为，而不是某天给你耍个花招。只有最终面向用户的应用才知道什么样的彩蛋对于它的用户是合适的，所以决定权在于应用。

其实呢，在 OpenResty Con 2017 中也有一个彩蛋，就是在大会发的宣传卡片上，有这么一段二进制内容：

![](/img/in-post/openresty-brand.jpg)

如果按字节切开的话，应该有 21 个字节：

```
01110111 01100101 01101100
01100011 01101111 01101101
01100101 00100000 01110100
01101111 00100000 01110100
01101000 01100101 00100000
01100110 01110101 01110100
01110101 01110010 01100101
```

当初看到时，我就觉得不应该是随意输入的 `01` 序列，打算回去之后解析下。不过最后忘了这事儿，但是这张卡片我并没有丢弃，而是拿来当书签用了😂。前几天翻书时，偶然又看到了这个卡片，索性就看看到底有没有什么秘密：

```lua
do
    local binStr = "011101110110010101101100011000110110111101101101011001010010000001110100011011110010000001110100011010000110010100100000011001100111010101110100011101010111001001100101"

    local function bin2bytes(s)
        local byteArray = {}

        for i=1,#s,8 do
           table.insert(byteArray, tonumber(s:sub(i,i+7), 2))
        end

        return byteArray
    end

    local function bytes2char(t)
        for i=1,#t do
           t[i] = string.char(t[i])
        end

        return table.concat(t, "")
    end

    print(bytes2char(bin2bytes(binStr)))
end
```

编码完运行一看：`welcome to the future`

哈哈，果然不出我所料嘛~~

[1]: https://github.com/ant-design/ant-design/issues/13848
[2]: https://blog.lilydjwg.me/2018/12/25/on-the-ant-design-easter-egg-incident.213909.html
[3]: https://zh.wikipedia.org/wiki/%E5%9C%A3%E8%AF%9E%E8%8A%82