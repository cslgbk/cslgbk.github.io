---
title: "使用正态分布随机Item"
date: 2021-04-12T10:33:21+08:00
draft: false
tags: ["UVM", "DPI", "C", "randomize"]
categories: ["UVM"]

---
# 1.总体思路
<font size=3/>第一种是在利用SV中的权重分布模拟正态函数分布，第二种是利用DPI接口实现正态函数。考虑到实现难易程度，选择后者。总体框架如图1.1所示。</font>
<br/>
<!--more-->
# 2.在C中实现正态分布函数
<font size=3/>实现正态分布函数有两种方法，一种是采用泰勒展开式，计算出最终值，一种是通过中心-极限定理得出正态分布。正态函数的泰勒展开式 -_-|| ... 一般采用中心-极限定理比较方便。
中心-极限定理的大致解释，每次从这些总体中随机抽取 n 个抽样，一共抽 m 次。 然后把这 m 组抽样分别求出平均值。 这些平均值的分布接近正态分布。</font>


```C代码
    #include<svdpi.h>           //dpi头文件，仿真器自带
    #include<stdlib.h>
    unsigned int get_normal()
    {
        unsigned int dat=0;     //产生数据
        unsigned char i;        
        unsigned int seed;      //随机种子
        seed = rand();
        srand(seed);
        for (i=0;i<10;i++)
        {
            dat += rand()%99;   //0~99随机数求和
        }
        return dat/i;           //返回平均值
    }

```
# 3.UVM搭建
测试平台由代码：
```
`include "uvm_macros.svh"

import "DPI-C" function bit [31:0] get_normal();    //导入C函数
import uvm_pkg::*;
	
class my_trans extends uvm_sequence_item;           //定义my_trans
	bit [31:0] dat;
	
	`uvm_object_utils_begin(my_trans)
		`uvm_field_int(dat,UVM_ALL_ON)
	`uvm_object_utils_end
	
	function new(string name =" ");
		super.new(name);
		dat = get_normal();
	endfunction 
	
	virtual function void do_print(uvm_printer printer);
		// TODO Auto-generated function stub
		//super.do_print(printer);
		//printer.print_field("dat", dat, $bits(dat), UVM_DEC);
	endfunction : do_print
endclass

class my_sequence extends uvm_sequence;             //定义my_sequence
	`uvm_object_utils(my_sequence)
	
	function new(string name = "my_sequence");
		super.new(name);	
	endfunction 
	
	task body();
		my_trans dat_trans;
		repeat(100000) begin
			#1
			`uvm_do(dat_trans)
		end
	endtask	
endclass

class my_sequencer extends uvm_sequencer#(.REQ(my_trans),.RSP(my_trans));           //定义my_sequencer
	`uvm_component_utils(my_sequencer)
	function new(string name = "my_sequencer", uvm_component parent);
		super.new(name,parent);
	endfunction 
	
endclass

class my_driver extends uvm_driver#(.REQ(my_trans),.RSP(my_trans));     //定义my_driver
	`uvm_component_utils(my_driver)

	function new(string name, uvm_component parent);
		super.new(name,parent);	
	endfunction 
	
	virtual  task main_phase(uvm_phase phase);
		REQ req;
		uvm_default_printer.knobs.default_radix=UVM_DEC;
		forever begin
			seq_item_port.get_next_item(req);
			req.print();
			seq_item_port.item_done();
		end
	endtask
endclass

class my_env extends uvm_env;                           //定义my_env
	`uvm_component_utils(my_env)
	
	my_sequencer sqr;
	my_driver drv;
	
	function new(string name, uvm_component parent);
		super.new(name, parent);
	endfunction 

	virtual function  void build_phase(uvm_phase phase);
		super.build_phase(phase);
		drv = my_driver::type_id::create("drv", this);
		sqr = my_sequencer::type_id::create("sqr", this);
	endfunction

	virtual function void connect_phase(uvm_phase phase);
		super.connect_phase(phase);
		drv.seq_item_port.connect(sqr.seq_item_export);
	endfunction
endclass 
	
class test0 extends uvm_test;
	my_env env;
	`uvm_component_utils(test0)
	function new(string name, uvm_component parent = null);
		super.new(name,parent);
	endfunction 

	virtual function  void build_phase(uvm_phase phase);
		super.build_phase(phase);
		env = my_env::type_id::create("env", this);
	endfunction : build_phase
	
		virtual task main_phase(uvm_phase phase);
		my_sequence dat_seq;
		super.main_phase(phase);
		phase.raise_objection(this);
			dat_seq = my_sequence::type_id::create("dat_seq");
			dat_seq.start(env.sqr);
		phase.drop_objection(this);
		endtask : main_phase
	
endclass

```
testbench:
```
module tb;
initial begin
	run_test("test0");
end
endmodule 
```


