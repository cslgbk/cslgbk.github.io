---
title: "含寄存器模型验证结构搭建"
date: 2021-04-21T16:00:48+08:00
draft: true
tags: ["UVM", "MCDF", "RGM"]
categories: ["UVM", "MCDF"]
---
寄存器模型中存储了DUT寄存器中的地址和数据信息，方便寄存器的读写，简化参考模型，便于覆盖率收集等特点。


<!--more-->
# 1.验证结构

![rgm_stru](/images/mcdf/v_str.png)


# 2. reg_if功能描述
## 接口
![reg_if](/images/mcdf/reg_if.PNG)

<font size=3>psel:APB总线选择信号（类似片选）</font>

<font size=3>paddr:APB总线地址总线</font>

<font size=3>pwr:APB总线读写信号</font>

<font size=3>pwdata:APB总线写入数据总线</font>

<font size=3>pwdata:APB总线读取数据总线</font>

<font size=3>pen:APB总线使能信号</font>

<font size=3>pready:读写完成标志位</font>

<font size=3>pslverr:总线错误标准位</font>

<font size=3>error_clr:清除slave_node错误标志位</font>

<font size=3>slv_en:slave_node使能控制位</font>

<font size=3>slvx_id:通道x ID号</font>

<font size=3>slvx_len:通道x数据包长度</font>

<font size=3>slvx_free_slot:通道x可用fifo数量</font>

<font size=3>slvx_parity_err:通道x奇偶校验标志位</font>

## 寄存器
<font size=3>last_st, cur_st：用于表示APB总线状态</font>

<font size=3></font>

<font size=3>ctrl_men：MCDF控制寄存器</font>

<font size=3>ro_men：reg_if模块中显示通道中FIFO的余量</font>


