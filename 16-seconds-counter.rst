第16章：秒计数器
========================

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T16-countsec/images/countsec-1.png

简介
----

为了练习分频器，我们将会实现 **一个每秒自增** 的计数器。计数由iCEstick的LED显示。


divisor.v: 硬件描述
------------------------------

我们会用上一章的分频器，带一些改进::

 // - divider.v
 // - Include constants defined for this component
 `include " divider.vh "
    
 // - Input: clk_in.  Original signal
 // - Output: clk_out.  Frequency signal 1 / M of the original
 module divider ( input wire clk_in, output wire clk_out);
    
 // - Default value of the divider
 // - We set it to 1 Hz (Constants defined in divider.vh)
 parameter M = `F_1Hz;
    
 // - Number of bits to store the divider
 // - They are calculated with the function of verilog $ clog2, which returns the
 // - number of bits needed to represent the number M
 // - It is a local parameter, which can not be modified when instantiating
 localparam N = $ clog2 (M);
    
 // - Register to implement the module meter M
 reg [N - 1 : 0 ] divcounter = 0 ;
    
 // - Module counter M
 always @ ( posedge clk_in)
   divcounter <= (divcounter == M - 1 )?  0 : divcounter + 1 ;
    
 // - Remove the most significant bit by clk_out
 assign clk_out = divcounter [N - 1 ];
    
 endmodule

第一个改进是包含了 **divisor.vh** 文件，这里 **定义** 了 **几个常数** 用于帮助产生特定频率的信号。

::

  `include " divider.vh "

相比于在分频器用一个除数，通过定义把它们关联到相应的频率更清楚。例如，要获得一个1Hz信号，可以用常数 F_1Hz 而不是数值 12_000_000::

  parameter M = `F_1Hz;

另一个改进是在描述模块M时用操作符 ``?:`` 而非 ``if-else``. 这样代码更紧凑，也更像C，那样很酷:-)

::

 // - Module counter M
 always @ ( posedge clk_in)
   divcounter <= (divcounter == M - 1 )?  0 : divcounter + 1 ;


divisor.vh: 常数
-----------------------------

**扩展名 .vh** 在verilog中用于定义常数和宏。定义常数和在C中类似，使用关键字 **`define**.这个文件定义了下列常数::

  // - Megahertz MHz
 `define F_4MHz 3
 `define F_1MHz 12
    
 // - Hertz (Hz)
 `define F_2Hz 6_000_000
 `define F_1Hz 12_000_000

在后续章节会有更多，因为我们用更多的频率和例子。

countsec.v: 秒计数器
-------------------------------

我们要实现一个4比特秒计数器，由LED取出数值。

硬件描述
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

这个设计包含一个4比特计数器，其时钟输入结合了一个 **分频器** 产生一个1Hz信号。

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T16-countsec/images/countsec-2.png

Verilog代码如下::

 // - countsec.v
 // - Include the constants of the divisor module
 `include " divider.vh "
    
 // - Parameteros:
 // - clk: Input clock of the iCEstick board
 // - data: Value of the seconds counter, to be drawn by the LEDs of the iCEstick
 module countsec ( input wire clk, output wire [ 3 : 0 ] data);
    
 // - Divider parameter.  Set it to 1Hz
 // - It is defined as a parameter to be able to modify it from the testbench
 // - to make tests
 parameter M = `F_1Hz;
    
 // - 1Hz clock signal.  Divider output
 wire clk_1HZ;
    
 // - Second counter
 reg [ 3 : 0 ] counter = 0 ;
    
 // - Instance the divider
 divider # (M)
   DIV (
     .clk_in (clk),
     .clk_out (clk_1HZ)
   );
    
 // - Increase the counter on each rising edge of the 1Hz signal
 always @ ( posedge clk_1HZ)
   counter <= counter + 1 ;
    
 // - Take the meter data to the LEDs
 assign data = counter;
    
 endmodule

综合
~~~~

要综合它，我们连接iCEstick的12MHz信号到时钟输入，连接输出到LED.

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T16-countsec/images/countsec-1.png

执行命令::

 $ make sint

使用资源有：

========   ======
  资源       占用
========   ======
  IOPs      4/96
  PLBs      12/160
  BRAMs     0/16
========   ======

要加载到FPGA，我们执行::

  $ iceprog countsec.bin


模拟
------

test bench和此前章节一样。获得1Hz频率不模拟，因为它花费大量时间和空间。我们用1个10分频器来验证计数器在工作::

 // - countsec_tb.v
 module countsec_tb ();
    
 // - Register to generate the clock signal
 reg clk = 0 ;
 wire [ 3 : 0 ] data;
    
 // - Instance the component and set the value of the divider
 // - A low value is set to simulate (otherwise it would take a long time)
 countsec # ( 10 )
   dut (
     .clk (clk),
     .data (data)
   );
    
 // - Clock generator.  Period 2 units
 always 
   # 1 clk <= ~ clk;
    
 // - Process at the beginning
 initial begin
    
   // - File to store the results
   $ dumpfile ( "countsec_tb.vcd" );
   $ dumpvars ( 0 , countsec_tb);
    
   # 100 $ display ( "END of the simulation" );
   $ finish ;
 end
 endmodule

要模拟我们执行命令::

 $ make sim

在模拟器中我们会看到计数器怎么“计数”：

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T16-countsec/images/T16-countsec-sim-1.png


提议的练习
------------

* 习题1：修改计数器的频率为2Hz
* 习题2：修改频率，从而计数器计数1/10秒，而不是秒

