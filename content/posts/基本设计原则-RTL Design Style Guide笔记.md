---
title: "RTL基本设计原则-RTL Design Style Guide笔记"
date: 2021-10-04T21:48:04+08:00
draft: false
tags: ["RTL"]
categories: ["RTL Design Style"]
---

### RTL基本设计原则
* RTL命名惯例

* 同步设计

* 复位

* 时钟

* 异步电路处理

* 层次化设计

<!--more-->

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

### 1.3 信号命名

[1] 命名应区别内部信号和端口

[2] 信号命名应便于理解

[3] 命名应控制在2-40个字符之间，最好是24个字符以下

![""](/images/RTL_DESIGN_STYLE/1-3.png)

### 1.4 include file、 parameter、 `define 命名规则

[1] 对于被include的文件，应以 ".h, .inc, .ht, .vh, .tsk"等作为后缀

[2] parameter命名应有单独的命名规范（如"P_"作为前缀）

[3] 不同模块中的parameter命名应当不同

[4] 仅仅在一个模块中使用`define

[5] parameter中应当包含层次结构的信息

[6] 固定的数值不能连接到输出端口，不推荐将固定数值连接到输入端口。（一般情况下，固定数值连接到端口不会产生问题。在综合优化的时候会导致冗余。logic equivelancy checks过程中可能会出现问题。）

[7] parameter 赋值时应用"'h, 'o, 'b, 'd"标明进制

[8] parameter 赋值时应标明数据位宽

![""](/images/RTL_DESIGN_STYLE/1-5.png)

### 1.5 有时钟的系统命名

[1] 如果端口是寄存器输出，建议命名中标记时钟

[2] 使用"CLK, CK"给时钟命名， "RST_X, RESET_X"给复位信号命名, "EN"给使能信号命名

[3] 命名中应体现时钟

[4] 命名中应体现寄存器

![""](/images/RTL_DESIGN_STYLE/1-6.png)

{{<admonition note "">}}
* 命名时应将信号分为：时钟信号、复位信号、使能信号等。
* 对于时钟、复位、使能信号应当使用特殊的标识，如"CLK, RST, EN"等
* 为了区分寄存器和组合逻辑，可以在寄存器名字后面添加"_R"或"_r"

{{</admonition>}}
