---
layout:     post
title:      "Uber Cadence 学习"
subtitle:   "相反 Netflix Conductor 的 JSON DSL 简直就是噩梦"
date:       2021-06-06
author:     "ms2008"
header-img: "img/post-bg-service-mesh.jpg"
catalog:    true
tags:
    - Golang
    - Cadence
typora-root-url: ..
---

## Cadence

"Cadence is a distributed, scalable, durable, and highly available orchestration engine to execute asynchronous long-running business logic in a scalable and resilient way. "

"Fault-Oblivious Stateful Code Platform."

简单来讲就是一个工作流引擎，是个好东西。可惜文档晦涩难懂，不使用业内通用模式和架构，自己创造一套，这大概也是 Uber 的一个特色吧🐶

> 相反 Netflix Conductor 的 JSON DSL 简直就是噩梦。

演进历史：

```
AWS Simple Workflow -> Uber Cadence -> Temporal
    	            -> AWS Step Function
```

### 🧩 Architecture

![](/img/in-post/cadence-overview.png)

- Frontend service

  相当于网关，使用自己实现的[TChannel][5]协议和其他角色通信

- History service

  负责 workflow 状态变迁

- Matching service

  托管 tasklist 进行 task scheduling, dispatching

- Worker service
  - Decision Worker

    又叫 Workflow Worker，负责执行 workflow func 以生成 activity task

  - Activity Worker

    执行 activity

### 🚥 Workflow

什么是工作流？

我们都知道，任何微小的工作，都可以拆分成多个步骤，这些步骤顺序相连，依次进行，最终输出成果，有些步骤可能存在多个分支，并且最终输出多个成果。**这些步骤依次执行，并且向后传递阶段性信息的流，就是工作流**。

工作流是个很宽泛的概念，审批系统算，容器编排、CI 的 pipeline 也都可以算。不同的工作流系统设计上有它的侧重点，所以可复杂可简单。但本质上（不是很精确的解释哈），都是在解决「流程的定义」和「流程的执行」这两件事。

1. 流程定义就是说设计一种数据结构，来表达业务流程，通常来说最后会落地成一张**有向图**（图结构）。实际系统中，由于流程可能会非常复杂，或者说需要可视化的与业务方人员沟通，这时就涉及到了流程的建模。

2. 流程执行就是核心功能了，简单的说就是读进流程定义，创建流程的实例（用来持久化流程相关的用户数据和状态），根据流程和实例的状态来执行流程。

常见的工作流引擎的自动化理论主要有：

- 有限状态机（FSM）
  - 简单、最常见
  - 可以有环
  - 描述的是单个对象的状态，也就是说（一个工作流实例内）仅能够追踪一个任务
- 有向无环图（DAG）
  - [AirFlow](https://airflow.apache.org) 、[Conductor](https://netflix.github.io/conductor) 采用的工作流理论
  - 不能有环
  - 工作流实例在一个时刻能够处于多个状态，可以追踪多个任务
- PetriNet
  - 主要用于面向 BPM 的工作流引擎
  - 可以有环
  - 工作流实例在一个时刻能够处于多个状态，可以追踪多个任务

可见 PetriNet 同时拥有 FSM 和 DAG 的特点，能够很好的支持最复杂的工作流的应用场景。Amazon workflow engine 就是基于此设计。Cadence 作者认为这个模型过于复杂，自创了一套：[State machines are wonderful tools][1]。

关于「流程的定义」业内通用的模式是通过 DSL 来描述，之后再写代码实现 worker 来完成 「流程的执行」；而 Cadence 不一样，它是通过代码来描述「流程的定义」，同样通过 worker 来执行流程。Cadence 作者有他自己的[看法][2]，总结起来就是大佬认为 DSL 的表现能力有限，不如代码来的直接。况且即使定义了 DSL，最后还是要自己实现 worker，何必多此一举。

###  ⏳ State

Cadence Workflow 内部状态获取有两种方式：

- History Protocol

  `eventId` 核心概念

- Query Handler

  业务自己埋点

异步状态也有两种方式：

- Signal

- Asynchronous Activity

### 👨‍💻 Demo Time

按照官方的 [samples][3] 来一遍，基本就会玩了。当然还有更复杂的用例：[uber eats][4]

### 参考资料

- [Cadence — The only workflow orchestrator you will ever need](https://blog.usejournal.com/cadence-the-only-workflow-orchestrator-you-will-ever-need-ea8f74ed5563)
- [Using Cadence workflows to spin up Kubernetes](https://banzaicloud.com/blog/introduction-to-cadence)
- [Building your first Cadence Workflow](https://medium.com/stashaway-engineering/building-your-first-cadence-workflow-e61a0b29785)

[1]: https://news.ycombinator.com/item?id=25614487
[2]: https://news.ycombinator.com/item?id=19734067
[3]: https://github.com/uber-common/cadence-samples
[4]: https://github.com/ms2008/cadence-codelab
[5]: https://github.com/uber/tchannel-go
