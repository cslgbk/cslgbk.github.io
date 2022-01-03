---
title: "Synchronous design - RTL Design Style Guide"
date: 2021-11-07T18:37:19+08:00
draft: false
tags: ["RTL"]
categories: ["RTL Design Style"]
---

### 同步设计
<!--more-->
[1] 设计时尽可能使用时钟的一个沿

[2] 不使用基本逻辑单元（与门、或门、非门等）搭建锁存器和触发器

[3] 组合逻辑中不得使用反馈信号（避免combination loop)

![""](/images/RTL_DESIGN_STYLE/1-2.png)

{{<admonition note "">}}
* 随着电路的规模越来越大，数字电路的速度通过DC、PT、BG等工具进行分析，因此尽量使用时钟的上升沿或者下降沿作为触发器的时钟，以降低电路的复杂性。
* 虽然触发器可以通过基本的门搭建，但是工具往往会将门电路搭建的触发器作为combination loop报错

{{</admonition>}}