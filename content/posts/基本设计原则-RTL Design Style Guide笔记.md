---
title: "RTL基本设计原则-RTL Design Style Guide笔记"
date: 2021-10-04T21:48:04+08:00
draft: false
tags: ["RTL"]
categories: ["RTL Design Style"]
---

### RTL基本设计原则
&bull; RTL命名惯例

&bull; 同步设计

&bull; 复位

&bull; 时钟

&bull; 异步电路处理

&bull; 层次化设计


### 1.RTL命名惯例

#### 1.1 基本命名惯例

[1] 文件名应与模块名同名。

[2] 命名只能使用字符数组和下划线，并且命名以字符开头。

[3] SystemVerilog 和 VHDL中的关键字不允许在命名时使用。

[4] "VDD","VSS","VCC","GND","VREF"等命名要杜绝使用。

[5] 不要通过大小写区分不同的命名.

[6] 命名不能以"_"结尾，应杜绝在命名中连续使用下划线.

[7] 可以加一些特定的符号以便于区分逻辑的极性，如"rst_n"等.

[8] 模式例化时命名应当和模块名有关联，同一模块多次例化时可以用"模块名_数量"的方式命名.

[9] 在设计顶层中，模块和端口的命名不应超过16个字符，并且不能通过大小写对模块和端口进行区分.

[10] 不要使用和库中相同的名称例化模块.

{{<admonition note "">}}
* 当工程比较复杂时，可以将树状层次结构的模块放在同一文件中.
* 处理语法关键字，应当尽量避免EDIF,SDF中的关键字，如：
<br>
    <font color=red>ABSOLUTE, cell, celltype, edif, DELAY, HOLD, IOPATH, NET, VIEW, SETUP</font>
{{</admonition>}}
<br>

### 1.2 层次化命名惯例

[1] 模块名和例化名应当控制在2-32个字符之间。

[2] 在每个模块名前最好加上层次化标识符。

[3] 上一级的层次化标识符应当应用到所有子模块中。

[4] 模块的输出端口应当采用 "<hierarchy identification character>_<xxxx>"形式

[5] 端口命名应当不同于内部信号的命名

[6] 输出端口命名应该和连接的信号名字相同，输入端口命名应当和上一级线网相同。

!["hierarchy_naming"](/images/RTL_DESIGN_STYLE/naming.png)

### 1.3 信号名应当有意义


### 1.4 include file、 parameter、 `define 命名规则


### 1.5 时钟命名


