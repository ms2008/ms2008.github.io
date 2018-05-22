---
layout:     post
title:      Kong 插件非官方 FAQ
subtitle:   Unofficial FAQ
date:       2018-05-22
author:     ms2008
header-img: img/post-bg-kongtribuitor-shirt.png
catalog:    true
tags:
    - Kong
---

经过了前面对 Kong 插件机制的分析，这里来整理一下**非官方** FAQ 以加深理解，以下 FAQ 针对于 **Kong 0.12.3** 版本。

#### 1. 插件怎么用？

插件可以应用在 API 上；也可以应用在 Consumer 上；同样还能应用在指定 API 的指定 Consumer 上；当然也少不了 GLOBAL 用法。总之，Kong 插件可以有四种启用方式：

- api
- consumer
- api & consumer
- global

#### 2. 一个 API 或者是 Consumer 可以添加同一个插件多次吗？

不可以。但是插件不同应用方式是可以添加相同的插件的，比如：API 可以添加 rate-limit 插件；Consumer 同样可以添加 rate-limit，只不过最后只有一个会生效。

#### 3. 插件的执行顺序？

插件的执行顺序由插件自身的优先级唯一确定。即，Kong 一旦启动，其插件的执行顺序就已经确定，和启用插件的方式无关，并不会在运行中动态改变。Kong 默认自带插件的优先级如下（越大越优先）：

PLUGIN                    | PRIORITY
:-------------------------|:------------
bot-detection             | 2500
cors                      | 2000
jwt                       | 1005
oauth2                    | 1004
key-auth                  | 1003
ldap-auth                 | 1002
basic-auth                | 1001
hmac-auth                 | 1000
ip-restriction            | 990
request-size-limiting     | 951
acl                       | 950
rate-limiting             | 901
response-ratelimiting     | 900
request-transformer       | 801
response-transformer      | 800
aws-lambda                | 750
http-log                  | 12
statsd                    | 11
datadog                   | 10
file-log                  | 9
udp-log                   | 8
tcp-log                   | 7
loggly                    | 6
runscope                  | 5
syslog                    | 4
galileo                   | 3
request-termination       | 2
correlation-id            | 1

#### 4. 插件生效的优先级？

根据[插件怎么用？](#1-插件怎么用)，这里提到的四种应用方式，如果插件使用冲突的话。其生效策略优先级是：

`api & consumer` > `consumer` > `api` > `global`

#### 5. 插件的执行阶段？

插件的执行阶段贯穿于请求生命周期中的不同阶段，不过目前大多数的插件均运行在 access 以及之后的阶段。

#### 6. 怎么写自己的插件？

==To be continued...==
