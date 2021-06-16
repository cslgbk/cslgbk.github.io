---
title: "自动化生成agent代码"
date: 2021-05-16T14:38:00+08:00
draft: false
tags: ["UVM", "验证自动化", "python"]
categories: ["UVM", "python", "验证自动化"]
---

### 1.总体思路
<font size=3/>在搭建验证框架时经常要输入很多重复性的代码，有两种快速获得验证的方法，一种是复制原有的框架代码，在代码上面修改，一种是利用脚本语言快速生成代码。本文通过python脚本，快速生成agent的代码，提高工作效率。</font>
<!--more-->
### 2.python脚本
<font size=3/>利用open、write函数创建SV文件，写入UVM框架代码。</font>

<font size=3/>code_gen.py:</font>

```
#coding=utf-8

import sys

name = input("input agentname:\n")

f = open(name+"_pkg.sv", 'w+', encoding="utf-8")

f.write("`include \"uvm_macros.svh\"\n\n")

f.write("package "+name+"_pkg;\n\n")
f.write("import uvm_pkg::*;\n\n")
name_trans = name + "_trans"
f.write("class "+name_trans+" extends uvm_sequence_item;\n")
f.write("\tfunction new(string name=\""+name_trans+"\");\n")
f.write("\t\tsuper.new(name);\n")
f.write("\tendfunction\n\n")
f.write("\t//输入变量\n")
f.write("\trand ;\n")
f.write("\trand ;\n")

f.write("\t`uvm_object_utils_begin("+name_trans+")\n")
f.write("\t\t`uvm_field_\n")
f.write("\t\t`uvm_field_\n")
f.write("\t`uvm_object_utils_end\n")

f.write("\nendclass\n")

name_seq = name+"_seq"
f.write("\nclass "+name_seq+" extends uvm_sequence;\n")
f.write("\t`uvm_object_utils("+name_seq+")\n\n")
f.write("\tfunction new(string name=\""+name_seq+"\");\n")
f.write("\t\tsuper.new(name);\n")
f.write("\tendfunction\n\n")
f.write("\tvirtual task body();\n")
f.write("\t\t"+name_trans+" tr;\n")
f.write("\t\trepeat(10) begin\n")
f.write("\t\t\t`uvm_do(tr)\n")
f.write("\t\tend\n")
f.write("\t\tendtask\n")
f.write("endclass\n\n")
name_sqr = name+"_sqr"
f.write("class "+name_sqr+" extends uvm_sequencer#("+name_trans+");\n")
f.write("\t`uvm_component_utils("+name_sqr+")\n\n")
f.write("\tfunction new(string name=\""+name_sqr+"\", uvm_component parent=null);\n")
f.write("\t\tsuper.new(name, parent);\n")
f.write("\tendfunction\n")
f.write("endclass\n\n")
name_drv = name+"_drv"
f.write("class "+name_drv+" extends uvm_driver#("+name_trans+");\n")
f.write("\t`uvm_component_utils("+name_drv+");\n\n")
f.write("\tfunction new(string name=\""+name_drv+"\", uvm_component parent=null);\n")
f.write("\t\tsuper.new(name,parent);\n")
f.write("\tendfunction\n\n")
f.write("\t function void set_interface();\n")
f.write("\t\t//设置接口\n\n")
f.write("\tendfunction\n\n")
f.write("\t"+name_trans+" tmp;\n")
f.write("\ttask main_phase(uvm_phase phase);\n")
f.write("\t\tforever begin\n")
f.write("\t\t\tseq_item_port.get_next_item(tmp);\n")
f.write("\t\t\t//驱动\n\n")
f.write("\t\t\tseq_item_port.item_done();\n")
f.write("\t\tend\n")
f.write("\tendtask\n\n")
f.write("endclass\n\n")
name_mon = name + "_mom"
f.write("class "+name_mon+" extends uvm_monitor#("+name_trans+");\n")
f.write("\t`uvm_component_utils("+name_mon+")\n\n")
f.write("\tuvm_analysis_port#("+name_trans+") ap;\n\n")
f.write("\tfunction new(string name=\""+name_mon+"\", uvm_component parent=null);\n")
f.write("\t\tsuper.new(name, parent);\n")
f.write("\t\tap = new(\"ap_mon\",this);\n")
f.write("\tendfunction\n\n")
f.write("\ttask main_phase(uvm_phase phase);\n")
f.write("\t\t//驱动\n\n\n")
f.write("\tendtask\n")
f.write("endclass\n\n")
name_agnt = name+"_agent"
f.write("class "+name_agnt+" extends uvm_agent;\n")
f.write("\t`uvm_component_utils("+name_agnt+")\n\n")
f.write("\tfunction new(string name=\""+name_agnt+"\", uvm_component parent=null);\n")
f.write("\tsuper.new(name, parent);\n")
f.write("\tendfunction\n\n")
f.write("\t"+name_drv+" drv;\n")
f.write("\t"+name_mon+" mon;\n")
f.write("\t"+name_sqr+" sqr;\n")
f.write("\tfunction void build_phase(uvm_phase phase);\n")
f.write("\t\tdrv = "+name_drv+"::type_id::create(\""+name_drv+"\", this);\n")
f.write("\t\tmon = "+name_mon+"::type_id::create(\""+name_mon+"\", this);\n")
f.write("\t\tsqr = "+name_sqr+"::type_id::create(\""+name_sqr+"\", this);\n")
f.write("\tendfunction\n\n")
f.write("\tfunction void connect_phase(uvm_phase phase);\n")
f.write("\t\tsuper.connect_phase(phase);\n")
f.write("\t\tdrv.seq_item_port.connect(sqr.seq_item_export);\n")
f.write("\tendfunction\n\n")
f.write("endclass\n")
f.write("\nendpackage\n")

f.close()
```

### 3.生成结果
```
python3 ./code_gen.py
```
<font size=3/>生成文件test_pkg:</font>
```
`include "uvm_macros.svh"

package test_pkg;

import uvm_pkg::*;

class test_trans extends uvm_sequence_item;
	function new(string name="test_trans");
		super.new(name);
	endfunction

	//输入变量
	rand ;
	rand ;
	`uvm_object_utils_begin(test_trans)
		`uvm_field_
		`uvm_field_
	`uvm_object_utils_end

endclass

class test_seq extends uvm_sequence;
	`uvm_object_utils(test_seq)

	function new(string name="test_seq");
		super.new(name);
	endfunction

	virtual task body();
		test_trans tr;
		repeat(10) begin
			`uvm_do(tr)
		end
		endtask
endclass

class test_sqr extends uvm_sequencer#(test_trans);
	`uvm_component_utils(test_sqr)

	function new(string name="test_sqr", uvm_component parent=null);
		super.new(name, parent);
	endfunction
endclass

class test_drv extends uvm_driver#(test_trans);
	`uvm_component_utils(test_drv);

	function new(string name="test_drv", uvm_component parent=null);
		super.new(name,parent);
	endfunction

	 function void set_interface();
		//设置接口

	endfunction

	test_trans tmp;
	task main_phase(uvm_phase phase);
		forever begin
			seq_item_port.get_next_item(tmp);
			//驱动

			seq_item_port.item_done();
		end
	endtask

endclass

class test_mom extends uvm_monitor#(test_trans);
	`uvm_component_utils(test_mom)

	uvm_analysis_port#(test_trans) ap;

	function new(string name="test_mom", uvm_component parent=null);
		super.new(name, parent);
		ap = new("ap_mon",this);
	endfunction

	task main_phase(uvm_phase phase);
		//驱动


	endtask
endclass

class test_agent extends uvm_agent;
	`uvm_component_utils(test_agent)

	function new(string name="test_agent", uvm_component parent=null);
	super.new(name, parent);
	endfunction

	test_drv drv;
	test_mom mon;
	test_sqr sqr;
	function void build_phase(uvm_phase phase);
		drv = test_drv::type_id::create("test_drv", this);
		mon = test_mom::type_id::create("test_mom", this);
		sqr = test_sqr::type_id::create("test_sqr", this);
	endfunction

	function void connect_phase(uvm_phase phase);
		super.connect_phase(phase);
		drv.seq_item_port.connect(sqr.seq_item_export);
	endfunction

endclass

endpackage

```
