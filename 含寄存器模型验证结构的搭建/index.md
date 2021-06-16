# 含寄存器模型验证结构搭建

寄存器模型中存储了DUT寄存器中的地址和数据信息，方便寄存器的读写，简化参考模型，便于覆盖率收集等特点。本文以MCDF的控制寄存器为例搭建含有寄存器模型的验证结构。

<!--more-->

### 1. 控制寄存器功能描述
#### 接口
![reg_if](/images/mcdf/reg_if.PNG)

<font size=3>psel:APB总线选择信号</font>

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

#### 寄存器
<font size=3>last_st, cur_st：用于表示APB总线状态</font>

<font size=3></font>

<font size=3>ctrl_men：MCDF控制寄存器</font>

<font size=3>ro_men：reg_if模块中只读寄存器</font>

### 2.验证结构

![rgm_stru](/images/mcdf/v1_str.png)

<font size=3>控制寄存器模块可通过APB总线进行读取和配置，可将控制寄存器分为两部分进行验证，一部分是模拟外部总线访问设计模块，一部分是模拟设计内部信号观察控制寄存器的反应。APB_agent用于访问控制寄存器模块的APB总线，reg_agent用于模拟设计内部信号。</font>

<font size=3>APB_agent中APB_seq产生随机的读写命令，通过driver发送至DUT。monitor用于监测总线变化，监测数据通过predictor、adapter最终传入寄存器模型。寄存器模型在APB总线每次被读写时更新。当monitor监测到总线数据传输完成时，同时触发寄存器模型更新和scoreboard（先缓存至analysis fifo）.</font>

<font size=3>scoreboard在触发后，读取寄存器模型中更新的值，作为参考设计的值，通过后门访问，直接获取DUT中的真实值，对比后输出结果。</font>

### 3.APB总线
<font size=3>AMBA总线介绍可参考[此文档](http://res.diandianme.com/sdr/resource/6ajyt.pdf)</font>

<font size=3>AMBA总线包含AHB（高级高性能总线）、ASB（高级系统总线）、APB（高级外设总线）。APB总线状态机和时序如下：</font>

![APB_Status](/images/mcdf/APB_Status.png)
![APB_Write](/images/mcdf/APB_Write.png)
![APB_Read](/images/mcdf/APB_Read.png)

### 4.验证环境搭建
<font size=3>AMBA总线介绍可参考[此文档](http://res.diandianme.com/sdr/resource/6ajyt.pdf)</font>

<font size=3>APB总线APB_pkg.sv：</font>

```
`include "uvm_macros.svh"

`define APB_DAT 32
`define APB_ADDR 8

package APB_pkg;

import uvm_pkg::*;

class APB_item extends uvm_sequence_item;
	rand bit[`APB_ADDR-1:0] paddr;
	rand bit[`APB_DAT-1:0]  pwdata;
	rand bit[`APB_DAT-1:0]  prdata;
	rand bit pwr;
	
	`uvm_object_utils_begin(APB_item)
		`uvm_field_int(paddr, UVM_ALL_ON)
		`uvm_field_int(pwdata, UVM_ALL_ON)
		`uvm_field_int(prdata, UVM_ALL_ON)
		`uvm_field_int(pwr, UVM_ALL_ON)
	`uvm_object_utils_end
	
    /*
	constraint addr_con	{
		paddr dist {8'h00:/8,	
					8'h04:/8,	
					8'h08:/8,	
					8'h0C:/8,
					8'h80:/8,
					8'h84:/8,
					8'h88:/8,
					8'h8C:/8,
					8'h90:/8,
					8'h94:/8,
					8'h98:/8,
					8'h9C:/8};
		}
        */
	
	function new(string name = " ");
		super.new(name);   
	endfunction : new
	
endclass

class APB_seq extends uvm_sequence;
	`uvm_object_utils(APB_seq)
	
	function new(string name=" ");
		super.new(name);
	endfunction
	
	virtual task body();
		APB_item tr;
		#200
		forever begin
			`uvm_do(tr)
		end
	endtask

endclass

class APB_sqr extends uvm_sequencer#(APB_item);
	`uvm_component_utils(APB_sqr)
	
	function new(string name="APB_sqr", uvm_component parent=null);
		super.new(name,parent);
	endfunction

endclass
	
class APB_driver extends uvm_driver#(APB_item);
	
	`uvm_component_utils(APB_driver)
	
	function new(string name = "APB_driver", uvm_component parent);
		super.new(name, parent);
	endfunction
	
	virtual reg_interface APB_vif;
	
	function void set_interface(virtual reg_interface reg_vif);
		APB_vif = reg_vif;
	endfunction
	
	task main_phase(uvm_phase phase);
		//uvm_sequence_item tmp;
		APB_item t;
		APB_idle();
		//@(APB_vif.rstn);
		forever begin
			seq_item_port.get_next_item(t);
			
			if (t.pwr == 0) begin
				APB_read(t);
			end 
			if (t.pwr == 1) begin
				APB_write(t);
			end
			seq_item_port.item_done();
		end
	endtask

	task APB_idle();
		APB_vif.apb_driver.psel < = 0;
		APB_vif.apb_driver.pen < = 0;
	endtask
	
	task APB_write(APB_item tmp);
		//@(APB_vif.pslverr|APB_vif.pready);
		//while(APB_vif.apb_driver.pslverr);
		@APB_vif.apb_driver;
		if( APB_vif.apb_driver.pready ) return;
		fork
		//@(!(APB_vif.pslverr|APB_vif.pready));
		//@(!APB_vif.pslverr);
		//@(!APB_vif.pready);
		APB_vif.apb_driver.psel < = 1;       //markdonwn问题， 实际为非阻塞赋值“<=”
		APB_vif.apb_driver.pen < = 0;
		APB_vif.apb_driver.paddr < = tmp.paddr;
		APB_vif.apb_driver.pwdata < = tmp.pwdata;
		APB_vif.apb_driver.pwr < = tmp.pwr;
		join
		
		@APB_vif.apb_driver 
		APB_vif.apb_driver.pen < = 1;
		@APB_vif.apb_driver fork
		APB_vif.apb_driver.pen < = 0;
		APB_vif.apb_driver.psel < = 0;
		join
	endtask
	
	task APB_read(APB_item tmp);
		//while(APB_vif.apb_driver.pslverr);
		
		@APB_vif.apb_driver;
		if (APB_vif.apb_driver.pready) return;
		fork
		//@(!(APB_vif.pslverr|APB_vif.pready));
		APB_vif.apb_driver.psel < = 1;
		APB_vif.apb_driver.pen < = 0;
		APB_vif.apb_driver.paddr < = tmp.paddr;
		APB_vif.apb_driver.pwr < = tmp.pwr;
		join
		@APB_vif.apb_driver 
		APB_vif.apb_driver.pen < = 1;
		@APB_vif.apb_driver fork
		tmp.prdata = APB_vif.apb_driver.prdata;
		APB_vif.apb_driver.pen < = 0;
		APB_vif.apb_driver.psel < = 0;
		join
	endtask
endclass
	
	
class APB_monitor extends uvm_monitor;
	`uvm_component_utils(APB_monitor)
	
	uvm_analysis_port#(APB_item) ap;
	
	APB_item tmp;
	int s;
	
	virtual reg_interface reg_intf;
	
	function void set_interface(virtual reg_interface vif);
		reg_intf = vif;
	endfunction
	
	function new(string name="APB_monitor", uvm_component parent);
		super.new(name, parent);
		ap = new("mon_ap",this);
		
	endfunction
	
	task main_phase(uvm_phase phase);
		
		forever begin
			@reg_intf.apb_sample;
			// `uvm_info("DEBUG",$sformatf("psel:%h pen:%h", reg_intf.apb_sample.psel, reg_intf.apb_sample.pen), UVM_LOW);
			if (reg_intf.apb_sample.pslverr) continue;
			if (s == 0) begin
				if (reg_intf.apb_sample.pready) continue;
			end
			
			if (reg_intf.apb_sample.psel) begin
				s = 1;
			
				if (reg_intf.apb_sample.pen) begin
					if (s != 2) begin
						s = 2;
						tmp = new("APB_monitor");
						tmp.pwr = reg_intf.apb_sample.pwr;
						tmp.paddr = reg_intf.apb_sample.paddr;
						tmp.prdata =reg_intf.apb_sample.prdata;
						tmp.pwdata = reg_intf.apb_sample.pwdata;
						ap.write(tmp);
					end
				end
			end else begin
				s = 0;
			end
			
		end
	endtask
endclass


class APB_agent extends uvm_agent;
	`uvm_component_utils(APB_agent)
	function new(string name, uvm_component parent);
		super.new(name,parent);
	endfunction
	
	APB_monitor monitor;
	APB_driver driver;
	APB_sqr sqr;
	//reg_monitor reg_mon;
	
	function void build_phase(uvm_phase phase);
		super.build_phase(phase);
		monitor = APB_monitor::type_id::create("APB_monitor",this);
		driver = APB_driver::type_id::create("APB_driver",this);
		sqr = APB_sqr::type_id::create("APB_sqr",this);
//		reg_mon = reg_monitor::type_id::create("reg_mon",this);
	endfunction
	
	function void connect_phase(uvm_phase phase);
		super.connect_phase(phase);
		driver.seq_item_port.connect(sqr.seq_item_export);
	endfunction
	
	function void set_interface(virtual reg_interface reg_vif);
		driver.set_interface(reg_vif);
		monitor.set_interface(reg_vif);
//		reg_mon.set_interface(reg_vif);
	endfunction
	
endclass


class rm2reg_adapter extends uvm_reg_adapter;
	`uvm_object_utils(rm2reg_adapter)
	function new(string name="rm2reg_adapter");
		super.new(name);
	endfunction
	
	function uvm_sequence_item reg2bus(const ref uvm_reg_bus_op rw);
		APB_item tmp = APB_item::type_id::create("reg2bus_item");
		tmp.pwr = (rw.kind == UVM_WRITE) ? 1:0;
		tmp.paddr = rw.addr;
		tmp.pwdata = rw.data;
		return tmp;
	endfunction
	
	function void bus2reg(uvm_sequence_item bus_item, ref uvm_reg_bus_op rw);
		APB_item tmp;
		if (!$cast(tmp, bus_item)) begin
			`uvm_fatal("ADAPTER_ERROR_BUS2REG","cast error!")
			return;
		end
		
		rw.kind = (tmp.pwr == 1) ? UVM_WRITE : UVM_READ;
		rw.addr = tmp.paddr;
		rw.data = (tmp.pwr == 1) ? tmp.pwdata:tmp.prdata;
		rw.status = UVM_IS_OK;
		//$display("rw");
		//$display(rw);
	endfunction
	
endclass

endpackage


```

<font size=3>reg_if_pkg.sv:</font>

```
`include "uvm_macros.svh"

import uvm_pkg::*;

class reg_status_tran extends uvm_sequence_item;
	
	rand bit slv0_parity_err;
	rand bit slv1_parity_err;
	rand bit slv2_parity_err;
	rand bit slv3_parity_err;

	rand bit[5:0] slv0_free_slot;
	rand bit[5:0] slv1_free_slot;
	rand bit[5:0] slv2_free_slot;
	rand bit[5:0] slv3_free_slot;
	
	function new(string name="reg_status_tran");
		super.new(name);
	endfunction
	
	`uvm_object_utils_begin(reg_status_tran)
		
		`uvm_field_int(slv0_parity_err, UVM_ALL_ON)
		`uvm_field_int(slv1_parity_err, UVM_ALL_ON)
		`uvm_field_int(slv2_parity_err, UVM_ALL_ON)
		`uvm_field_int(slv3_parity_err, UVM_ALL_ON)
		
		`uvm_field_int(slv0_free_slot, UVM_ALL_ON)
		`uvm_field_int(slv1_free_slot, UVM_ALL_ON)
		`uvm_field_int(slv2_free_slot, UVM_ALL_ON)
		`uvm_field_int(slv3_free_slot, UVM_ALL_ON)
		
	`uvm_object_utils_end
	
endclass

class reg_status_seq extends uvm_sequence#(reg_status_tran);
	
	`uvm_object_utils(reg_status_seq)
	
	function new(string name="reg_status_seq");
		super.new(name);
	endfunction
	
	reg_status_tran tr;
	virtual task body();
		`uvm_do_with(tr, {	slv0_free_slot == 10;
							slv0_parity_err == 0;
							slv1_free_slot == 10;
							slv1_parity_err == 0;
							slv2_free_slot == 10;
							slv2_parity_err == 0;
							slv3_free_slot == 10;
							slv3_parity_err == 0;})
		# 2000;
	endtask
	
endclass

class reg_sqr extends uvm_sequencer#(reg_status_tran);
	
	`uvm_component_utils(reg_sqr)
	
	function new(string name="reg_sqr", uvm_component parent=null);
		super.new(name,parent);
	endfunction
	
endclass

class reg_drv extends uvm_driver#(reg_status_tran);
	
	`uvm_component_utils(reg_drv)
	
	function new(string name="drv", uvm_component parent = null);
		super.new(name, parent);
	endfunction
	
	virtual reg_interface reg_vif;
	
	function set_interface(virtual reg_interface reg_vif);
		this.reg_vif = reg_vif;
	endfunction 
	
	task main_phase(uvm_phase phase);
		reg_status_tran tr;
		
		forever begin
			seq_item_port.get_next_item(tr);
			driver_status(tr);
			seq_item_port.item_done();
		end
	endtask
		
	task driver_status(reg_status_tran tr);
		@reg_vif.reg_driver;
		reg_vif.reg_driver.slv0_parity_err < = tr.slv0_parity_err;   
		reg_vif.reg_driver.slv1_parity_err < = tr.slv1_parity_err;
		reg_vif.reg_driver.slv2_parity_err < = tr.slv2_parity_err;
		reg_vif.reg_driver.slv3_parity_err < = tr.slv3_parity_err;
		
		reg_vif.reg_driver.slv0_free_slot < = tr.slv0_free_slot;
		reg_vif.reg_driver.slv1_free_slot < = tr.slv1_free_slot;
		reg_vif.reg_driver.slv2_free_slot < = tr.slv2_free_slot;
		reg_vif.reg_driver.slv3_free_slot < = tr.slv3_free_slot;
	endtask
	
endclass

class reg_if_agent extends uvm_agent;
	
	`uvm_component_utils(reg_if_agent)
	
	function new(string name="reg_if_agent", uvm_component parent = null);
		super.new(name, parent);
	endfunction 
	
	reg_sqr sqr;
	reg_drv drv;
	
	
	function void build_phase(uvm_phase phase);
		super.build_phase(phase);
		
		sqr = reg_sqr::type_id::create("reg_sqr", this);
		drv = reg_drv::type_id::create("reg_drv", this);
		
	endfunction
	
	function void connect_phase(uvm_phase phase);
		super.connect_phase(phase);
		drv.seq_item_port.connect(sqr.seq_item_export);
	endfunction 
	
	function void set_interface(virtual reg_interface reg_vif);
		drv.set_interface(reg_vif);
	endfunction 
	
endclass

```

<font size=3>采用时钟块，避免信号之间竞争 reg_interface.sv:</font>

```
interface reg_interface(input clk, input rstn);
	
logic  [7:0]       paddr;
logic              pwr;
logic              pen;
logic              psel;
logic  [31:0]      pwdata;
logic  [31:0]      prdata;	
	
logic             pready;
logic             pslverr; 

clocking apb_sample @(posedge clk);
	default input #1 output #1;
	input paddr;
	input pwr;
	input pen;
	input psel;
	input pwdata;
	input prdata;
	input pready;
	input pslverr;
endclocking

clocking apb_driver @(posedge clk);
	default input #1 output #1;
	output psel;
	output pen;
	output pwr;
	output paddr;
	output pwdata;
	input pready;
	input pslverr;
	input prdata;
endclocking

logic [3:0]       slv_en;  
logic [3:0]       err_clr;
logic [7:0]       slv0_id;
logic [7:0]       slv1_id;
logic [7:0]       slv2_id;
logic [7:0]       slv3_id;

logic [7:0]       slv0_len;
logic [7:0]       slv1_len;
logic [7:0]       slv2_len;
logic [7:0]       slv3_len;

logic              slv0_parity_err;
logic              slv1_parity_err;
logic              slv2_parity_err;
logic              slv3_parity_err;

logic  [5:0]       slv0_free_slot;
logic  [5:0]       slv1_free_slot;
logic  [5:0]       slv2_free_slot;
logic  [5:0]       slv3_free_slot;

clocking reg_listen @(posedge clk);
	default input #1 output  #1;
	
	input pslverr;
	input pready;
	
	input slv_en;
	input err_clr;
	input slv0_id;
	input slv1_id;
	input slv2_id;
	input slv3_id;
	
	input slv0_len;
	input slv1_len;
	input slv2_len;
	input slv3_len;
	
	input slv0_parity_err;
	input slv1_parity_err;
	input slv2_parity_err;
	input slv3_parity_err;
	
	input slv0_free_slot;
	input slv1_free_slot;
	input slv2_free_slot;
	input slv3_free_slot;
	
endclocking

clocking reg_driver @(posedge clk);
	default input #1 output #1;
	
	output slv0_parity_err;
	output slv1_parity_err;
	output slv2_parity_err;
	output slv3_parity_err;
	
	output slv0_free_slot;
	output slv1_free_slot;
	output slv2_free_slot;
	output slv3_free_slot;
	
endclocking

endinterface

```

<font size=3>寄存器模型reg_rm.sv:</font>

```
`include "uvm_macros.svh"

import uvm_pkg::*;

class slv_en extends uvm_reg;
	`uvm_object_utils(slv_en)
	
	function new(string name = "slv_en");
		super.new(name, 32, UVM_NO_COVERAGE);
	endfunction
	
	uvm_reg_field reserved;
	
	rand uvm_reg_field slv3_en;
	rand uvm_reg_field slv2_en;
	rand uvm_reg_field slv1_en;
	rand uvm_reg_field slv0_en;
	
	function build();
		reserved = uvm_reg_field::type_id::create("reserved");
		slv3_en = uvm_reg_field::type_id::create("slv3_en");
		slv2_en = uvm_reg_field::type_id::create("slv2_en");
		slv1_en = uvm_reg_field::type_id::create("slv1_en");
		slv0_en = uvm_reg_field::type_id::create("slv0_en");
		reserved.configure(this, 28, 4, "RO", 0, 28'h0, 1, 0, 0);
		slv3_en.configure(this, 1, 3, "RW", 0, 0, 1, 1, 0);
		slv2_en.configure(this, 1, 2, "RW", 0, 0, 1, 1, 0);
		slv1_en.configure(this, 1, 1, "RW", 0, 0, 1, 1, 0);
		slv0_en.configure(this, 1, 0, "RW", 0, 0, 1, 1, 0);
	endfunction
endclass


class parity_err_clr extends uvm_reg;
	`uvm_object_utils(parity_err_clr)
	rand uvm_reg_field reserved;
	rand uvm_reg_field parity_err_clr_3;
	rand uvm_reg_field parity_err_clr_2;
	rand uvm_reg_field parity_err_clr_1;
	rand uvm_reg_field parity_err_clr_0;
	
	function new(name="parity_err_clr");
		super.new(name, 32, UVM_NO_COVERAGE);
	endfunction
	
	virtual function build();
		reserved = uvm_reg_field::type_id::create("reserved");
		parity_err_clr_3 = uvm_reg_field::type_id::create("parity_err_3");
		parity_err_clr_2 = uvm_reg_field::type_id::create("parity_err_2");
		parity_err_clr_1 = uvm_reg_field::type_id::create("parity_err_1");
		parity_err_clr_0 = uvm_reg_field::type_id::create("parity_err_0");
		
		reserved.configure(this, 28, 4, "RO", 0, 28'h0, 1, 0, 0);
		parity_err_clr_3.configure(this, 1, 3, "RW", 0, 0, 1, 1, 0);
		parity_err_clr_2.configure(this, 1, 2, "RW", 0, 0, 1, 1, 0);
		parity_err_clr_1.configure(this, 1, 1, "RW", 0, 0, 1, 1, 0);
		parity_err_clr_0.configure(this, 1, 0, "RW", 0, 0, 1, 1, 0);
	endfunction
endclass

class slv_id extends uvm_reg;
	`uvm_object_utils(slv_id)
	
	function new(string name="slv_id");
		super.new(name, 32, UVM_NO_COVERAGE);
	endfunction
	
	rand uvm_reg_field slv3_id;
	rand uvm_reg_field slv2_id;
	rand uvm_reg_field slv1_id;
	rand uvm_reg_field slv0_id;
	
	function build();
		slv3_id = uvm_reg_field::type_id::create("slv3_id");
		slv2_id = uvm_reg_field::type_id::create("slv2_id");
		slv1_id = uvm_reg_field::type_id::create("slv1_id");
		slv0_id = uvm_reg_field::type_id::create("slv0_id");
		
		slv3_id.configure(this, 8, 24, "RW", 0, 8'h0, 1, 1, 0);
		slv2_id.configure(this, 8, 16, "RW", 0, 8'h0, 1, 1, 0);
		slv1_id.configure(this, 8, 8,  "RW", 0, 8'h0, 1, 1, 0);
		slv0_id.configure(this, 8, 0,  "RW", 0, 8'h0, 1, 1, 0);
	endfunction
	
endclass

class slv_len extends uvm_reg;
	`uvm_object_utils(slv_len)
	
	function new(string name="slv_len");
		super.new(name, 32, UVM_NO_COVERAGE);
	endfunction
	
	rand uvm_reg_field slv3_len;
	rand uvm_reg_field slv2_len;
	rand uvm_reg_field slv1_len;
	rand uvm_reg_field slv0_len;
	
	function build();
		
		slv3_len = uvm_reg_field::type_id::create("slv3_len");
		
		slv2_len = uvm_reg_field::type_id::create("slv2_len");
		
		slv1_len = uvm_reg_field::type_id::create("slv1_len");
		
		slv0_len = uvm_reg_field::type_id::create("slv0_len");
		
		slv3_len.configure(this, 8, 24, "RW", 0, 8'h0, 1, 1, 0);
		slv2_len.configure(this, 8, 16, "RW", 0, 8'h0, 1, 1, 0);
		slv1_len.configure(this, 8, 8,  "RW", 0, 8'h0, 1, 1, 0);
		slv0_len.configure(this, 8, 0,  "RW", 0, 8'h0, 1, 1, 0);
	endfunction
	
endclass

class slv_free_slot extends uvm_reg;
	`uvm_object_utils(slv_free_slot)
	function new(string name="slv_free_slot");
		super.new(name, 32, UVM_NO_COVERAGE);
	endfunction

	rand uvm_reg_field reserved;
	rand uvm_reg_field free_slot;
	
	function build();
		reserved = uvm_reg_field::type_id::create("reserved");
		free_slot = uvm_reg_field::type_id::create("free_slot");
		
		reserved.configure(this, 26, 6, "RO", 0, 26'h0, 1, 0, 0);
		free_slot.configure(this, 6, 0, "RW", 0, 6'h0, 1, 1, 0);
	endfunction
	
endclass

class slv_parity_err extends uvm_reg;
	`uvm_object_utils(slv_parity_err)
	
	function new(string name="slv_parity_err");
		super.new(name, 32, UVM_NO_COVERAGE);
	endfunction
	
	rand uvm_reg_field reserved;
	rand uvm_reg_field parity_err;
	
	function build();
		reserved = uvm_reg_field::type_id::create("reserved");
		parity_err = uvm_reg_field::type_id::create("parity_err");
		
		reserved.configure(this, 31, 1, "RO", 0, 31'h0, 1, 0, 0);
		parity_err.configure(this, 1, 0, "RW", 0, 0, 1, 1, 0);
	endfunction
	
endclass

class reg_rgm extends uvm_reg_block;
	`uvm_object_utils(reg_rgm)
	
	function new(string name = "reg_rgm");
		super.new(name, UVM_NO_COVERAGE);
	endfunction
	
	rand slv_en slv_en_reg;
	rand parity_err_clr parity_err_clr_reg;
	rand slv_id	slv_id_reg;
	rand slv_len slv_len_reg;
	
	rand slv_free_slot slv0_free_slot_reg;
	rand slv_free_slot slv1_free_slot_reg;
	rand slv_free_slot slv2_free_slot_reg;
	rand slv_free_slot slv3_free_slot_reg;
	
	rand slv_parity_err slv0_parity_err_reg;
	rand slv_parity_err slv1_parity_err_reg;
	rand slv_parity_err slv2_parity_err_reg;
	rand slv_parity_err slv3_parity_err_reg;
	
	uvm_reg_map map;
	
	function build();
		
		slv_en_reg = slv_en::type_id::create("slv_en_reg");
		slv_en_reg.configure(this);
		slv_en_reg.build();
		
		parity_err_clr_reg = parity_err_clr::type_id::create("parity_err_clr_reg");
		parity_err_clr_reg.configure(this);
		parity_err_clr_reg.build();
		
		slv_id_reg=slv_id::type_id::create("slv_id_reg");
		slv_id_reg.configure(this);
		slv_id_reg.build();
		
		slv_len_reg = slv_len::type_id::create("slv_len_reg");
		slv_len_reg.configure(this);
		slv_len_reg.build();
		
		slv0_free_slot_reg = slv_free_slot::type_id::create("slv0_free_slot_reg");
		slv0_free_slot_reg.configure(this);
		slv0_free_slot_reg.build();
		
		slv1_free_slot_reg = slv_free_slot::type_id::create("slv1_free_slot_reg");
		slv1_free_slot_reg.configure(this);
		slv1_free_slot_reg.build();

		slv2_free_slot_reg = slv_free_slot::type_id::create("slv2_free_slot_reg");
		slv2_free_slot_reg.configure(this);
		slv2_free_slot_reg.build();
		
		slv3_free_slot_reg = slv_free_slot::type_id::create("slv3_free_slot_reg");
		slv3_free_slot_reg.configure(this);
		slv3_free_slot_reg.build();
		
		slv0_parity_err_reg = slv_parity_err::type_id::create("slv0_parity_err_reg");
		slv0_parity_err_reg.configure(this);
		slv0_parity_err_reg.build();
		
		slv1_parity_err_reg = slv_parity_err::type_id::create("slv1_parity_err_reg");
		slv1_parity_err_reg.configure(this);
		slv1_parity_err_reg.build();
		
		slv2_parity_err_reg = slv_parity_err::type_id::create("slv2_parity_err_reg");
		slv2_parity_err_reg.configure(this);
		slv2_parity_err_reg.build();
		
		slv3_parity_err_reg = slv_parity_err::type_id::create("slv3_parity_err_reg");
		slv3_parity_err_reg.configure(this);
		slv3_parity_err_reg.build();
		
		map = create_map("reg_map", 'h0, 4, UVM_LITTLE_ENDIAN);
		
		add_hdl_path("reg_if_tb.reg_tb");
		
		map.add_reg(slv_en_reg, 8'h00, "RW");
		slv_en_reg.add_hdl_path_slice("ctrl_mem[0]", 0 ,32);
		
		map.add_reg(parity_err_clr_reg, 8'h04, "RW");
		parity_err_clr_reg.add_hdl_path_slice("ctrl_mem[1]", 0, 32);
		
		map.add_reg(slv_id_reg, 8'h08, "RW");
		slv_id_reg.add_hdl_path_slice("ctrl_mem[2]", 0, 32);
		
		map.add_reg(slv_len_reg, 8'h0C, "RW");
		slv_len_reg.add_hdl_path_slice("ctrl_mem[3]", 0, 32);
		
		map.add_reg(slv0_free_slot_reg, 8'h80, "RO");
		slv0_free_slot_reg.add_hdl_path_slice("ro_mem[0]", 0, 32);
		
		map.add_reg(slv1_free_slot_reg, 8'h84, "RO");
		slv1_free_slot_reg.add_hdl_path_slice("ro_mem[1]", 0, 32);
		
		map.add_reg(slv2_free_slot_reg, 8'h88, "RO");
		slv2_free_slot_reg.add_hdl_path_slice("ro_mem[2]", 0, 32);
		
		map.add_reg(slv3_free_slot_reg, 8'h8C, "RO");
		slv3_free_slot_reg.add_hdl_path_slice("ro_mem[3]", 0, 32);
		
		map.add_reg(slv0_parity_err_reg, 8'h90, "RO");
		slv0_parity_err_reg.add_hdl_path_slice("ro_mem[4]", 0, 32);
		
		map.add_reg(slv1_parity_err_reg, 8'h94, "RO");
		slv1_parity_err_reg.add_hdl_path_slice("ro_mem[5]", 0, 32);
		
		map.add_reg(slv2_parity_err_reg, 8'h98, "RO");
		slv2_parity_err_reg.add_hdl_path_slice("ro_mem[6]", 0, 32);
		
		map.add_reg(slv3_parity_err_reg, 8'h9C, "RO");
		slv3_parity_err_reg.add_hdl_path_slice("ro_mem[7]", 0, 32);
		
		lock_model();
	endfunction
	
endclass
```

<font size=3>环境连接关系reg_test.sv:</font>

```
`include "uvm_macros.svh"
import uvm_pkg::*;
import APB_pkg::*;


class APB_scoreboard extends uvm_scoreboard;
	`uvm_component_utils(APB_scoreboard)
	
	APB_item tmp;
	reg_rgm rm;
	uvm_reg reg_tmp;
	
	int dat_exp;
	int actual;
	uvm_status_e s;
	
	uvm_blocking_get_port#(APB_item) scb_p;
	
	function new(string name="apb_scb", uvm_component parent = null);
		super.new(name, parent);
		scb_p = new("scb", this);
	endfunction
	
	virtual task main_phase(uvm_phase phase);
		
		super.main_phase(phase);
		forever begin
			scb_p.get(tmp);
			reg_tmp = rm.map.get_reg_by_offset(tmp.paddr);
				
			if (reg_tmp == null) begin
				`uvm_info("BUS ERROR!", $sformatf("\taddr:%0h cannot find", tmp.paddr), UVM_LOW)
					
			end else begin
				dat_exp = reg_tmp.get_mirrored_value();
				reg_tmp.peek(s, actual);
				
				if (actual == dat_exp) begin
					`uvm_info("SCB_PASS",$sformatf("\tpwr:%h addr:%h actual:%h, exp: %h", tmp.pwr, tmp.paddr, actual, dat_exp), UVM_LOW)
				end else begin
					
					`uvm_info("SCB_ERR",$sformatf("\tpwr:%h addr:%h actual:%h, exp: %h", tmp.pwr, tmp.paddr, actual, dat_exp), UVM_LOW)
				end
			end
		end
	endtask
	
endclass

class reg_if_coverage extends uvm_subscriber#(APB_item);
	
	`uvm_component_utils(reg_if_coverage)
	
	reg_rgm rm;
	
	covergroup reg_cg;
		
		coverpoint rm.slv_en_reg.slv0_en.value {bins len[]={0, 1};}	
		coverpoint rm.slv_en_reg.slv1_en.value {bins len[]={0, 1};}
		coverpoint rm.slv_en_reg.slv2_en.value {bins len[]={0, 1};}
		coverpoint rm.slv_en_reg.slv3_en.value {bins len[]={0, 1};}
		
		coverpoint rm.parity_err_clr_reg.parity_err_clr_0.value {bins len[]={0, 1};}
		coverpoint rm.parity_err_clr_reg.parity_err_clr_1.value {bins len[]={0, 1};}
		coverpoint rm.parity_err_clr_reg.parity_err_clr_2.value {bins len[]={0, 1};}
		coverpoint rm.parity_err_clr_reg.parity_err_clr_3.value {bins len[]={0, 1};}
		
		coverpoint rm.slv_id_reg.slv0_id.value;
		coverpoint rm.slv_id_reg.slv1_id.value;
		coverpoint rm.slv_id_reg.slv2_id.value;
		coverpoint rm.slv_id_reg.slv3_id.value;
		
		coverpoint rm.slv_len_reg.slv0_len.value;
		coverpoint rm.slv_len_reg.slv1_len.value;
		coverpoint rm.slv_len_reg.slv2_len.value;
		coverpoint rm.slv_len_reg.slv3_len.value;
		
		coverpoint rm.slv0_free_slot_reg.free_slot.value;
		coverpoint rm.slv1_free_slot_reg.free_slot.value;
		coverpoint rm.slv2_free_slot_reg.free_slot.value;
		coverpoint rm.slv3_free_slot_reg.free_slot.value;
		
	endgroup
	
	function new(string name="reg_if_coverage", uvm_component parent=null);
		super.new(name,parent);
		reg_cg = new();
	endfunction
	
	function void build_phase(uvm_phase phase);
		
	endfunction 
	
	function void write(T t);
		reg_cg.sample();
	endfunction

	function void set_interface(reg_rgm rgm);
		this.rm = rgm;
	endfunction
	
endclass

class APB_env extends uvm_env;
	
	APB_agent apb_agent;
	rm2reg_adapter reg_adapter;
	reg_rgm reg_rm;
	APB_scoreboard scb;
	reg_if_agent reg_agent;
	reg_if_coverage reg_cg;
	
	virtual reg_interface reg_vif;
	
	uvm_reg_predictor #(APB_item) reg_predictor;
	uvm_tlm_analysis_fifo#(APB_item) scb_fifo;
	
	`uvm_component_utils(APB_env)
	
	function new(string name="APB_env", uvm_component parent);
		super.new(name,parent);
	endfunction
	
	function void build_phase(uvm_phase phase);
		
		if(!uvm_config_db#(virtual reg_interface)::get(this, "", "reg_vif", reg_vif)) begin
			`uvm_fatal("CONFIG_DB_ERROR", "reg_vif")
		end
		
		apb_agent = APB_agent::type_id::create("apb_agent",this);
		
		reg_adapter = rm2reg_adapter::type_id::create("reg_adapter",this);
		uvm_config_db#(reg_rgm)::get(this,"","reg_rm",reg_rm);
		reg_predictor = uvm_reg_predictor#(APB_item)::type_id::create("reg_predictor",this);
		reg_predictor.map = reg_rm.default_map;
		reg_predictor.adapter = reg_adapter;
		
		scb = APB_scoreboard::type_id::create("APB_scb", this);
		scb_fifo = new("scb_fifo");
		
		reg_agent = reg_if_agent::type_id::create("reg_agent", this);
		
		reg_cg = reg_if_coverage::type_id::create("reg_cg", this);
		reg_cg.set_interface(reg_rm);
		
		
	endfunction
	
	function void connect_phase(uvm_phase phase);
		
		apb_agent.set_interface(reg_vif);
		reg_agent.set_interface(reg_vif);
		
		scb.rm = reg_rm;
		reg_rm.default_map.set_sequencer(apb_agent.sqr, reg_adapter);
		apb_agent.monitor.ap.connect(reg_predictor.bus_in);
		
		apb_agent.monitor.ap.connect(scb_fifo.analysis_export);
		scb.scb_p.connect(scb_fifo.blocking_get_export);
		
		reg_agent.drv.seq_item_port.connect(reg_agent.sqr.seq_item_export);
		
		apb_agent.monitor.ap.connect(reg_cg.analysis_export);
		
	endfunction
endclass

class base_test extends uvm_test;
	
	`uvm_component_utils(base_test)
	
	reg_rgm reg_rm;
	APB_env env;
	uvm_status_e s;
	bit[31:0] dat;
	
	reg_status_seq reg_status;
	
	function new(string name="base_test", uvm_component parent=null);
		super.new(name,parent);
	endfunction
	
	function void build_phase(uvm_phase phase);
		reg_rm = reg_rgm::type_id::create("reg_rm",this);
		reg_rm.build();
		reg_rm.default_map.set_auto_predict(0);
		reg_rm.lock_model();
		reg_rm.reset();
		
		uvm_config_db#(reg_rgm)::set(this,"env","reg_rm",reg_rm);
		env = APB_env::type_id::create("env",this);
		
	endfunction
	
endclass

```

<font size=3>测试用例case.sv:</font>
```
`include "uvm_macros.svh"

import uvm_pkg::*;
import APB_pkg::*;

class case0_seq extends APB_seq;
		
	`uvm_object_utils(case0_seq)
	
	function new(string name ="case0_seq");
		super.new(name);
	endfunction
	
	APB_item t;
	
	
	
	task body();
		#100;
		repeat(100000) begin
			//`uvm_info("CASE0","Start...",UVM_LOW)
			`uvm_do(t)
		end
	endtask
	
endclass

class case1_seq extends APB_seq;
		
	`uvm_object_utils(case1_seq)
	
	function new(string name ="case1_seq");
		super.new(name);
	endfunction
	
	APB_item t;
	
	task body();
		#100;
		repeat(100000) begin
			//`uvm_info("CASE0","Start...",UVM_LOW)
			`uvm_do_with(t, {paddr==8'h8c;})
		end
	endtask
	
endclass


class case0_test extends base_test;
	
	`uvm_component_utils(case0_test)
	
	function new(string name="case0_test", uvm_component parent=null);
		super.new(name, parent);
	endfunction
	
	function void build_phase(uvm_phase phase);
		super.build_phase(phase);
		`uvm_info("CASE0","Building",UVM_LOW)
	endfunction
	
	case0_seq c;
	reg_status_seq r;
	
	task main_phase(uvm_phase phase);
		phase.raise_objection(this);
		
		c = case0_seq::type_id::create("case0");
		r = reg_status_seq::type_id::create("status");
		fork
		#20 c.start(env.apb_agent.sqr);
		r.start(env.reg_agent.sqr);
		join
		
		phase.drop_objection(this);
	endtask
	
endclass
```

<font size=3>测试tb.sv:</font>

```
`include "uvm_macros.svh"

import uvm_pkg::*;

module reg_if_tb();

reg clk;
reg rstn;

reg_interface reg_intf(clk,rstn);

reg_if reg_tb(	.clk_i(reg_intf.clk),
				.rst_n_i(reg_intf.rstn),
				.paddr_i(reg_intf.paddr),
				.pwr_i(reg_intf.pwr),
				.pen_i(reg_intf.pen),
				.psel_i(reg_intf.psel),
				.pwdata_i(reg_intf.pwdata),
				.prdata_o(reg_intf.prdata),
				.pready_o(reg_intf.pready),
				.pslverr_o(reg_intf.pslverr),
				.slv_en_o(reg_intf.slv_en),
				.err_clr_o(reg_intf.err_clr),
				.slv0_id_o(reg_intf.slv0_id),
				.slv1_id_o(reg_intf.slv1_id),
				.slv2_id_o(reg_intf.slv2_id),
				.slv3_id_o(reg_intf.slv3_id),
				.slv0_len_o(reg_intf.slv0_len),
				.slv1_len_o(reg_intf.slv1_len),
				.slv2_len_o(reg_intf.slv2_len),
				.slv3_len_o(reg_intf.slv3_len),
				.slv0_parity_err_i(reg_intf.slv0_parity_err),
				.slv1_parity_err_i(reg_intf.slv1_parity_err),
				.slv2_parity_err_i(reg_intf.slv2_parity_err),
				.slv3_parity_err_i(reg_intf.slv3_parity_err),
				.slv0_free_slot_i(reg_intf.slv0_free_slot),
				.slv1_free_slot_i(reg_intf.slv1_free_slot),
				.slv2_free_slot_i(reg_intf.slv2_free_slot),
				.slv3_free_slot_i(reg_intf.slv3_free_slot));	
	
initial begin
	forever begin
		#10 clk = 0;
		#10 clk = 1;
	end
end

initial begin
	rstn = 0;
	#50 rstn = 1;
end

initial begin
	uvm_config_db#(virtual reg_interface )::set(null, "uvm_test_top.env", "reg_vif", reg_intf);
end

initial begin
	run_test("case0_test");
end

endmodule
```
<font size=3>makefile:</font>

```
VCS = vcs -full64 -f tb.list -sverilog -debug_all\
	  -timescale="1ns/1ps" -l comp.md \
	  +define+UVM_OBJECT_MUST_HAVE_CONSTRUCTOR\
	  $(UVM_HOME)/src/dpi/uvm_dpi.cc -CFLAGS -DVCS\
	  -cm line+cond+fsm+tgl \
	  -LDFLAGS -rdynamic -P $(VERDI_HOME)/share/PLI/VCS/LINUX64/novas.tab \
	  $(VERDI_HOME)/share/PLI/VCS/LINUX64/pli.a
	  

SIMV = ./simv -l sim.md +UVM_OBJECTION_TRACE -ucli -i ../scripts/dump_fsdb.tcl +fsdb+autoflush -cm line+cond+fsm+tgl

RUN_VERDI = verdi -ssf apb.fsdb -l verdi.md -nologo -f tb.list

RESULT = dve -full64 -covdir simv.vdb

comp:
	$(VCS)

gui:
	$(RUN_VERDI)

clean:
	mv ./makefile ../makefile
	mv ./tb.list ../tb.list
	rm -rf *
	mv ../makefile ./makefile
	mv ../tb.list ./tb.list

run:
	$(VCS)
	$(SIMV)

result:
	$(RESULT)

```

### 5.运行
<font size=3>运行</font>

```
make run
```

### 6.查看覆盖率
```
make result
```
![code_coverage](/images/mcdf/code_coverage.png)
![fun_coverage](/images/mcdf/fun_coverage.png)

