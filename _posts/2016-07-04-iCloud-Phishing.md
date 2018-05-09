---
layout:     post
title:      挖掘 iCloud 钓鱼站汇总
subtitle:   凡事留一线，日后好相见
date:       2016-07-04
author:     ms2008
header-img: img/post-bg-unix-linux.jpg
catalog:    true
tags:
    - Phishing
    - Security
    - Pwned
typora-root-url: ..
---

首贴发布于 [V站](https://www.v2ex.com/t/264398)，前几日又在朋友圈儿看到被钓鱼，遂做下整理，望各位继续补充

- [www.my-appleid-iphone.com](#)
- [www.appleoue.com](#)
- [apple-ios9icloud.com](#)
- [apple.nunu520.com](#)
- [web.osepfy.loan](#)
- [locatemyidevice.net/ios.login/](#)

仿真度非常高，而且还会更新，可见已经形成了完整的产业链

![](/img/in-post/phishing/pwned-1.jpg)
![](/img/in-post/phishing/pwned-2.jpg)
![](/img/in-post/phishing/pwned-3.jpg)

测试了下 SQL 注入 和 XSS 成功拿下，基本都是一套程序，所以很多钓鱼站已经沦陷（我很后悔当时只顾拉源码了，并没有植入后门，哎，好可惜）

![](/img/in-post/phishing/pwned-4.jpg)
![](/img/in-post/phishing/pwned-17.jpg)
![](/img/in-post/phishing/pwned-5.jpg)
![](/img/in-post/phishing/pwned-6.jpg)
![](/img/in-post/phishing/pwned-7.jpg)
![](/img/in-post/phishing/pwned-8.jpg)
![](/img/in-post/phishing/pwned-9.jpg)

后来又发现几个钓鱼站，做的很完美，都是 ASP 封包程序，可以过 360，而且还是线上更新的那种，看来作者也是投入了很多精力。

![](/img/in-post/phishing/pwned-10.png)
![](/img/in-post/phishing/pwned-11.png)

QQ 密保都钓到了:

![](/img/in-post/phishing/pwned-14.png)
![](/img/in-post/phishing/pwned-15.png)
![](/img/in-post/phishing/pwned-16.png)

还可以更改模板：

![](/img/in-post/phishing/pwned-12.png)
![](/img/in-post/phishing/pwned-13.png)

根据爆破的几个邮箱来看，估计 10000+ ID 被钓鱼。如果按照某宝上的价格 300/ID 算，300 W RMB 市场，而且这还只是「冰山一角」
