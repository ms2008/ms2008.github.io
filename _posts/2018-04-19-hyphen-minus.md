---
layout:     post
title:      Source Code Pro 字体其实并不完美
subtitle:   Courier New 不愧为终端之王
date:       2018-04-19
author:     ms2008
header-img: img/home-bg-o.jpg
catalog:    true
tags:
    - Unicode
    - Source Code Pro
    - IDN Homograph Attack
    - 同形异义字攻击
typora-root-url: ..
---

事情的起因是这样的，前两天我在服务器上看到一个莫名其妙的文件夹 `‐p`，所以决定删了它，于是顺手就敲了 `rm -rf -- -p`。

然而执行结果却令人意外：`rm` 提示目录并不存在。

> `rm` 删除包含特殊字符的文件时，需要 `--` 参数

**显然这里显示出来的 `‐p` 其实并不是 ASCII 中的 `-p`**。之后我又手动的拷贝了 `ll` 输出的 `-p`，这一次成功删除了这个目录。

![](/img/in-post/hyphen/source_code_pro.png)

为了彻底弄清楚这个字符，我把它拷贝到了 UE，试图用 16 进制来揭开它的真面目：

![](/img/in-post/hyphen/HYPHEN_HEX.png)

可以看到拷贝得到的 `‐` UTF-8 (*因为终端就是 UTF-8编码*) 编码为 3 个字节，而我们习惯上的 `-` 只会占 1 个字节。查看其 Unicode 码点为 `U+2010`：

![](/img/in-post/hyphen/HYPHEN_unicode_point.png)

其实这个字符叫 **Hyphen (U+2010)**，而我们经常使用的减号叫 **Hyphen-minus (U+002D)**。它们都分布在 Unicode 的 BMP 平面上，只是所属的 block 不同而已。「Hyphen」属于 General Punctuation Block 而「Hyphen-minus」属于 Basic Latin Block。**实际上人们都只使用继承自 ASCII 的 Hyphen-minus (U+002D)**。

<u>关于两者的区别就是它们的宽度其实上是不同的，但是这要取决于字体的表现</u>。就拿上面的 Source Code Pro 字体来讲，其在终端上的表现就是不尽如人意的。它把「Hyphen」渲染的几乎和「Hyphen-minus」一样了，肉眼根本无法分辨出来。而 Courier New 字体就不会这样：

![](/img/in-post/hyphen/courier_new.png)

也许这也是 Courier New 长期称霸终端的原因之一。

至于系统是怎样创建 `‐p` 这样目录的，这个锅应该甩给 Typora 了。我们这边的部署文档都是使用 markdown 写的，之后用 Typora 导出 PDF 给运维。问题是老版本的 Typora 会将 **Hyphen-minus (U+002D)** 替换为 **Hyphen (U+2010)**。运维部署的时候拷贝了 PDF 中的命令，这样命令就被成功的 “转义” 了，于是系统就出现了那些奇怪的目录。好在目前最新版的 Typora 已经修复了这个问题，建议各位尽早升级。


## 背景知识

其实上面讨论的问题，很类似一种攻击即：同形异义字攻击。这种欺骗攻击就是网址看起来是合法的，但实际上不是，因为其中的一个字符或者多个字符已经被 Unicode 字符代替了。

许多 Unicode 字符，代表的是国际化的域名中的希腊、斯拉夫、亚美尼亚字母，看起来跟拉丁字母一样，但是计算机却会把他们处理成完全不一样的地址。

比如说，斯拉夫字母 “а”（U+0430）和拉丁字母 “a”（U+0061）会被浏览器处理成不同的字符，但是在地址栏当中都显示为 “a”。

![](/img/in-post/hyphen/latin_a.png)

去年国内的 @Xudong Zheng 就展示了这种攻击：[Phishing with Unicode Domains][1]，而且最后还得到了 Google 的 2000 美元奖励。另外，github 上也有个项目 [EvilURL][2]，就是专门生成这种 URL 的。

为了防止这种钓鱼攻击，许多浏览器使用 “**Punycode**” 编码来表示 URL 中的 Unicode 字符。Punycode 是浏览器使用的特殊编码，目的是将 Unicode 字符转义成字符数目有限的 ASCII 码字符集（A-Z，0-9），由国际化域名（IDN）系统支持。

比如说，中文域名「短.co」用 Punycode 来表示就是「xn--s7y.co」。此处的 xn 前缀是一个 “ASCII兼容编码” 前缀，意味着浏览器将采用 Punycode 编码来代表 Unicode 字符。这里就不再介绍其细枝末节了。

***

## 参考文献

- [Hyphen](https://en.wikipedia.org/wiki/Hyphen)
- [Dash](https://en.wikipedia.org/wiki/Dash)
- [Plane “Basic Multilingual Plane”](https://www.compart.com/en/unicode/plane/U+0000)
- [英文写作中标点连字号（hyphen）与连接号（dash）的输入](http://blog.sciencenet.cn/blog-301516-669888.html)
- [一种几乎无法被检测到的Punycode钓鱼攻击](http://www.freebuf.com/news/132240.html)

[1]: https://www.xudongz.com/blog/2017/idn-phishing/
[2]: https://github.com/UndeadSec/EvilURL
