第14章：带同步复位的N比特寄存器
================================

简介
----

寄存器是构造数字电路的常用元素。虽然它在Verilog中的描述很简单，我们会创建一个在我们的设计中可以实例化的 **通用的N比特寄存器**。这可以让我们 **容易地放置很多记录**。此外，我们会添加一个 **同步复位输入**，它可以用一个初始值加载这个记录（默认是0但是我们可以在初始化记录时制定另一个值）。作为一个例子，我们会做一个 **有2个寄存器的序列发生器**，LED获取存储在它们之中的数据。

register.v: 带同步复位的N比特寄存器
-------------------------------------------------------

寄存器的方案如下图：

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T14-regreset/images/regreset-2.png

它有一个 **N比特的输入din**，和相应的 **dout输出**。数据在时钟信号的 **上升沿** 获取。它有一个 **同步复位输入**：如果收到了0信号并且时钟沿到来，寄存器就加载初始值(INI)，它在实例化时定义。

我们在之前的章节中已经知道这种寄存器初始化的方式，但现在已经集成到寄存器本身。它在Verilog的描述如下::

 // - register.v
 module register (rst, clk, din, dout);

 // - Parameters:
 parameter N = 4 ;  // - Number of bits in the registry
 parameter INI = 0 ;  // - Initial value

 // - Declaration of the ports
 input wire rst;
 input wire clk;
 input wire [N - 1 : 0 ] din;
 output reg [N - 1 : 0 ] dout;

 // - Registration
 always @ ( posedge (clk))
   if (rst == 0 )
     dout <= INI;  // - Initialization
   else
     dout <= din;  // - Normal operation

 endmodule

这和之前的实现等价，在这里我们用两个过程：一个用于多选器，一个用于记录。这里在同一过程中使用两个部件。这是 **实现带初始化的寄存器的一般方式**。

regreset.v: 带2个4比特寄存器的序列发生器
-----------------------------------------------

作为一个测试例子，我们要实现一个 **2状态序列发生器**，使用 **2个寄存器**。每个开始时存放要在LED每个状态上显示的值。 **寄存器是链起来的**，从而一个的输出连到另一个的输入。这样，每次一个时钟上升沿到来，寄存器 **交换它们的值**。它们中的一个的输出连到LED.

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T14-regreset/images/regreset-3.png

寄存器0的输出口是输出到外界（数据口）的输出，但同时还发到寄存器1的输入，其输出连到寄存器0的输入。

在寄存器的复位输入端， **一个初始化器** 用于在第一个时钟上升沿完成 **初始化的加载**。复位线在图中用红色画出。在其余的周期它们像标准的寄存器一样工作，加载通过它们的din输入过来的数据。

时钟（绿线）经过一个前置分频器然后进入寄存器和初始化器。

Verilog代码如下::

 // - regreset.v
 module regreset ( input wire clk, output wire [ 3 : 0 ] data);

 // - Sequencer parameters:
 parameter NP = 23 ;  // - Bits of the prescaler
 parameter INI0 = 4'b1001 ;  // - Initial value for record 0
 parameter INI1 = 4'b0111 ;  // - Initial value for record 1

 // - Clock at the exit of the presacaler
 wire clk_pres;

 // - Output of the regitros
 wire [ 3 : 0 ] dout0;
 wire [ 3 : 0 ] dout1;

 // - Reset initialization signal
 reg rst = 0 ;

 // - Initializer
 always @ ( posedge (clk_pres))
   rst <= 1 ;

 // - Registration 0
 register # (.INI (INI0))
   REG0 (
     .clk (clk_pres),
     .rst (rst),
     .din (dout1),
     .dout (dout0)
   );

 // - Registration 1
 register # (.INI (INI1))
   REG1 (
     .clk (clk_pres),
     .rst (rst),
     .din (dout0),
     .dout (dout1)
   );

 // - Take the output of record 0 for that of the component
 assign data = dout0;

 // - Prescaler
 prescaler # ( .N (NP))
   PRES (
     .clk_in (clk),
     .clk_out (clk_pres)
   );

 endmodule


在FPGA里综合
-----------------

要在FPGA里综合，我们连接数据输出到LED，以及iCEstick板的时钟输入。

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T14-regreset/images/regreset-1.png

我们用以下命令综合::

  $ make sint

使用的资源有:

========   ======
  资源       占用
========   ======
  IOPs      5/96
  PLBs      11/160
  BRAMs     0/16
========   ======

要加载到FPGA，我们执行::

  $ iceprog regreset.bin


模拟
----

test bench比较基础，它实例化了regreset部件，1比特用于前置分频器（从而模拟用时较少）。它有一个时钟信号过程，和一个模拟初始化的过程。

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T14-regreset/images/regreset-4.png

verilog代码::

 // - regreset_tb.v
 module regreset_tb ();
    
 // - Register to generate the clock signal
 reg clk = 0 ;
    
 // - Component output data
 wire [ 3 : 0 ] data;
    
 // - Instance the component, with 1-bit prescaler (for simulation)
 regreset # (. NP ( 1 ))
   dut (
    .clk (clk),
    .data (data)
 );
    
 // - Clock generator.  Period 2 units
 always # 1 clk = ~ clk;
    
 // - Process at the beginning
 initial begin 
    
   // - File to store the results
   $ dumpfile ( "regreset_tb.vcd" );
   $ dumpvars ( 0 , regreset_tb);
    
   # 30 $ display ( "END of the simulation" );
   $ finish ;
 end
    
 endmodule

用以下命令模拟::

  $ make sim

gtkwave的结果：

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T14-regreset/images/T14-regreset-sim-1.png

你可以看到序列1001, 0111, 1001, 0111 ... 的交替方式。

提议的练习
------------

* 习题1：修改设计，添加2个寄存器，使得序列发生器有4个状态
