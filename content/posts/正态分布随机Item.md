---
title: "正态分布随机化item"
date: 2021-04-12T10:33:21+08:00
draft: false
tags: ["UVM", "DPI", "C", "python"]
categories: ["UVM", "python"]

---
### 1.总体思路
<font size=3/>通过正态分布generator产生数据传递item，最终通过driver将数据打印至日志文件，利用python脚本绘制分布图，检验产生数据是否正确</font>
<!--more-->
<font size=3/>产生正态分布数据有两种方法，第一种是在利用SV中的权重分布模拟正态函数分布，第二种是利用DPI接口实现正态函数。考虑到实现难易程度，选择后者。总体框架如下图所示。</font>
<br/>
![str](/images/else/normal_str.png)
<br/>

### 2.在C中实现正态分布函数
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

### 3.UVM搭建
测试平台由代码：
```
`include "uvm_macros.svh"

import "DPI-C" function bit [31:0] get_normal();  //导入C函数
import uvm_pkg::*;
	
class my_trans extends uvm_sequence_item;        //定义my_trans
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

class my_sequence extends uvm_sequence;         //定义my_sequence
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

class my_sequencer extends uvm_sequencer#(.REQ(my_trans),.RSP(my_trans));     //定义my_sequencer
	`uvm_component_utils(my_sequencer)
	function new(string name = "my_sequencer", uvm_component parent);
		super.new(name,parent);
	endfunction 
	
endclass

class my_driver extends uvm_driver#(.REQ(my_trans),.RSP(my_trans));   //定义my_driver
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

class my_env extends uvm_env;           //定义my_env
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
<font size=3/>testbench:</font>

```
module tb;
initial begin
	run_test("test0");
end
endmodule 
```

### 4. 利用python脚本提取数据，查看结果

<font size=3/>VCS最终打印数据：</font>

```
--------------------------------
Name       Type      Size  Value
--------------------------------
dat_trans  my_trans  -     @572 
  dat      integral  32    'd43 
--------------------------------
```

<font size=3/>利用python脚本提取dat的数值：</font>
```
import matplotlib
import matplotlib.pyplot as plt

f = open("sim.log")

line = f.readlines()
dat=[]
for tmp in line:
    s  = tmp.split()
    for i in range(0,len(s)-1):
        if s[i] == " ":
            s.pop(i)
    if len(s) > 3:
        if s[0]=="dat":
            if s[1] == "integral":
                dat.append(int(s[3][2:])) # 提取数据


plt.hist(dat,100,(0,99))
plt.show()
```

### 5. makefile文件编写

<font size=3/>vcs仿真器makefile：</font>

```
VCS = vcs -f tb.list +vc ../tb/gen.c\
		-full64 -debug_all -sverilog -timescale="1ns/1ps" -l log.txt \
		+vpi \
		+define+UVM_OBJECT_MUST_HAVE_CONSTRUCTOR \
		$(UVM_HOME)/src/dpi/uvm_dpi.cc -CFLAGS -DVCS

SIM = ./simv -l sim.log +UVM_TIMEOUT=100000000

DISP = python3 analyze_dat.py

gui:
	$(SIM) -gui 

comp:
	$(VCS)
	
sim:
	$(SIM)
	
disp:
	$(DISP)

run:
	$(VCS)
	$(SIM) 
	$(DISP)

clean:
	cp makefile ../makefile
	cp tb.list ../tb.list
	cp analyze_dat.py ../analyze_dat.py
	rm -rf *
	mv ../makefile makefile
	mv ../tb.list tb.list
	mv ../analyze_dat.py analyze_dat.py
```

### 6. 运行结果

<font size=3/>make运行</font>

```
make run
```

<font size=3/>最终结果：</font>

![result](/images/else/normal_dat.png)