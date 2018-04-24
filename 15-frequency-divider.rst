第15章：分频器
========================

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T15-divisor/images/divM-sintesis.png

简介
----

用 **前置分频器** 我们得到低频的时钟信号。然而，我们不能得到想要的准确频率。例如，我们想实现一个每秒自增的计数器，我们怎么做？我们需要 **分频器**。

有了这些分频器，我们可以 **产生准确频率的信号**。它们对通信（UART、SPI、I2C），屏幕刷新，PWM信号的产生，计时器，音调的产生等应用很重要。

在这章我们会实现一个 **通用分频器**，并且我们会用它 **以1Hz的频率让一个LID闪烁** （每秒一次）。


分频器
--------

对一个信号进行2分频很简单：我们放一个1比特前置分频器。一般地，要进行2的幂次（2,4,8,16,...,2^N）分频我们需要一个 **N比特前置分频器**。对其他频率，我们需要分频器。

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T15-divisor/images/divisor-1.png

这个组建有一个输入信号（clk_in），周期为T_in.输出为另一个信号（clk_out），频率是输入信号除以M，或者从周期看，输出信号的周期是输入信号的M倍。


"Hello world": 3分频器
-----------------------------

我们从最小可做的分频器开始，一个 **3分频器"hello world"**. 我们要获得一个1/3频率的信号。在有12MHz时钟的iCEstick上测试，我们会得到一个12/3=4MHz的信号。

输入输出信号如下：

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T15-divisor/images/divisor-2.png

我们可以看到clk_out的周期是T_in的3倍，尽管 **活动周期不同**。clk_in在高低电平有相同的时间，而clk_out有2/3的周期在低电平，1/3在高电平。对于时序问题，**活动周期没有不同**。重要的是频率。

一个 **M分频器** 用一个 **模M计数器** 实现，并 **输出最高比特**。

要实现这个hello world分频器，我们要实现一个如下图所示的 **模3计数器**

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T15-divisor/images/divisor-3.png

模3计数器
~~~~~~~~~~~~

一般地说，一个 **模M计数器** 在M个脉冲后后初始化。它开始是0计数，然后是1,2,3,4,...到M-1，在下一个脉冲它回到0重新开始。

模3计数器重复计数：0,1,2,0,1,2,0,1,2...

实现它的硬件方案如下所示：

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T15-divisor/images/divisor-4.png

一个 **2比特寄存器** 存储当前计数，由data输出。通过 **比较器** 检查值是否为2，如果是，选择输入1被激活，从而计数器初始化为0,否则，选择另一个 **输出值加1** 的输入，记录增加。

这个 **模3计数器** 可以用 **Verilog** 简单地描述::

 reg [ 1 : 0 ] data = 0 ;
 
 always @ ( posedge clk)
   if (data == 2 )
     data <= 0 ;
   else
     data <= data + 1 ;

**注**：我们总是用 ``@(posedge clk)`` 而不是 ``@(posedge (clk))``.它们是等价的。

div3.v: 3分频器的硬件描述
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

我们已经有实现3分频器的所有元素。完整代码如下::

 // - div3.v
 module div3 ( input wire clk_in, output wire clk_out);
    
 reg [ 1 : 0 ] divcounter = 0 ;
    
 // - Module 3 counter
 always @ ( posedge clk_in)
   if (divcounter == 2 )
     divcounter <= 0 ;
   else
     divcounter <= divcounter + 1 ;
    
 // - Remove the most significant bit by clk_out
 assign clk_out = divcounter [ 1 ];
    
 endmodule

div3.v: 模拟
~~~~~~~~~~~~~~~~~~

要模拟，我们创建一个 **很简单的test bench**,它简单地实例化这个部件，产生一个时钟信号并初始化模拟。

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T15-divisor/images/div3_tb.png

Verilog代码::

 // - div3_tb.v
 module div3_tb ();
    
 // - Register to generate the clock signal
 reg clk = 0 ;
 wire clk_3;
    
 // - Instance the divider
 div3
   dut (
     .clk_in (clk),
     .clk_out (clk_3)
   );
    
 // - Clock generator.  Period 2 units
 always # 1 clk = ~ clk;
    
 // - Process at the beginning
 initial begin
    
   // - File to store the results
   $ dumpfile ( "div3_tb.vcd" );
   $ dumpvars ( 0 , div3_tb);
    
   # 30 $ display ( "END of the simulation" );
   $ finish ;
 end
 endmodule

我们执行命令::

 $ make sim

模拟结果为：

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T15-divisor/images/div3_sim.png

我们看到输出信号 clk_out 的周期是输入信号的3倍。

div3: 综合和测试
~~~~~~~~~~~~~~~~~~~~~~~~

要综合，我么连接iCEstick的12MHz时钟到clk_in并将clk_out连到一个LED.

由于输出频率是4MHz，我们不会看到LED闪烁，我们只是看到它亮。我们要连接一个示波器来验证信号是4MHz.

我们用以下命令综合::

 $ make sint

使用资源有：

========   ======
  资源       占用
========   ======
  IOPs      2/96
  PLBs      1/160
  BRAMs     0/16
========   ======

要加载到FPGA，我们执行::

  $ iceprog div3.bin


M分频器
-------------

要获得更多的输出频率，我们要用一个 **通用分频器**，它可以对输入频率除M. 它和3分频器类似，有考虑如下因素：

- 用一个 **模M计数器**
- 它有一个 **N比特寄存器**，在这里N是可以存储数M的长度
- clk_out信号是记录的最高位：**位N-1**

divM.v: 硬件描述
~~~~~~~~~~~~~~~~~~~~~~~~~~

verilog代码如下::

 // - divM.v
 module divM ( input wire clk_in, output wire clk_out);
    
 // - Default value of the divider
 // - As in the iCEstick the clock is 12MHz, we put a value of 12M
 // - to obtain an output frequency of 1Hz
 parameter M = 12_000_000;
    
 // - Number of bits to store the divider
 // - They are calculated with the function of verilog $ clog2, which returns the
 // - number of bits needed to represent the number M
 // - It is a local parameter, which can not be modified when instantiating
 localparam N = $ clog2 (M);
    
 // - Register to implement the module meter M
 reg [N - 1 : 0 ] divcounter = 0 ;
    
 // - Module counter M
 always @ ( posedge clk_in)
   if (divcounter == M - 1 ) 
     divcounter <= 0 ;
   else 
     divcounter <= divcounter + 1 ;
    
 // - Remove the most significant bit by clk_out
 assign clk_out = divcounter [N - 1 ];
    
 endmodule

模M计数器和模3分频器一样，但使用了 **常数M**.对计数器的最高位和clk_out的连接，用了 **常数N**.

**M是分频器的参数**，从其他组件实例化时可以设置。默认它设为12000000(12M)以获得1Hz输出频率。

f_out = end / M = 12Mhz / 12M = 1Hz

为了改善数值的可读性，并不会发生疑惑，verilog中的数值的 **数码** 可以 **用下划线(_)分开**。这样，12M可以写成 12_000_000

**常数N是本地的**，且不能在divM模块外修改。这是通过用 **localparam** 声明来实现的。

**N定义了需要用来存放M值的比特位数**，例如M为3,我们需要2位，M为200,我们需要8位。这个计算用 **Verilog函数 $ clog2(M)** 完成。数学上通过对M取以2为底的对数，再上取整(ceiling操作)获取。

同样的计算可以用Python按以下方式完成：

 import math as m
    
 M = 12000000
 N = int (m.ceil (m.log (M, 2 )))
 print (N)

返回结果 **N=24**.也就是说，**产生一个1Hz信号，我们需要一个24比特计数器**。

divM.v: 综合
~~~~~~~~~~~~~~~~~~

和3分频器的方式一样：12MHz时钟输入连接clk_in，而1Hz输出信号通过LED输出。

我们执行命令综合::

 $ make sint2

(2指代本章的工程2)

使用资源有：

========   ======
  资源       占用
========   ======
  IOPs      3/96
  PLBs      8/160
  BRAMs     0/16
========   ======

要加载到FPGA，我们执行::

  $ iceprog divM.bin

LED会开始以1Hz闪烁：每秒1次

divM.v: 模拟
~~~~~~~~~~~~~~~~~~

test bench和3分频器类似，但现在在实例化通用分频器时，我们设置它的参数M.为了让模拟不花太多时间，我们用 **小的M值**。我们用 **M=5**，对信号做5分频::

 // - divM.v
 module divM_tb ();
 
 // - Register to generate the clock signal
 reg clk = 0 ;
 wire clk_out;
 
 // - Instance the component and set the value of the divider
 divM # ( 5 )
   dut (
     .clk_in (clk),
     .clk_out (clk_out)
   );
 
 // - Clock generator.  Period 2 units
 always # 1 clk = ~ clk;
 
 // - Process at the beginning
 initial begin
 
   // - File to store the results
   $ dumpfile ( "divM_tb.vcd" );
   $ dumpvars ( 0 , divM_tb);
 
   # 30 $ display ( "END of the simulation" );
   $ finish ;
 end
 endmodule

要进行模拟，我们执行::

 $ make sim2

gtkwave的输出是：

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T15-divisor/images/divM_sim_M5.png

我们要检查分频器另一个值：M = 7. 在test bench中你要修改为::

 divM #(7)

现在gtkwave的输出是：

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T15-divisor/images/divM_sim_M7.png

输出信号有等于7周期输入信号的时钟周期。


提议的练习
------------

* 习题1：修改设计分频器值，从而LED以0.5Hz的频率闪烁
* 习题2：思考如何让LED在1Hz闪烁，但有50%的占空因数，也就是说，LED有0.5秒亮，另外0.5秒灭

