第21章：波特和传送
======================

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T21-baud-tx/images/baudtx-1.png

简介
====

我们从发送器开始设计我们的UART. 在这一章，我们关注比特传送。作为例子，我们会做几个电路的例子来以115200波特的速度发送字符到PC.

发送比特
========

比特一个接一个地通过Tx引脚发送到PC. The data is not sent in isolation, but is embedded in a plot . The serial transmission standard defines different frames. We will use the typical one, known as 8N1 (8 bits of data, None of parity and 1 bit of stop) that has the following format :

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T21-baud-tx/images/serial-frame-format-2.png

The frame begins with a bit at 0 , which is called a start bit . Then there are the 8 bits of the data to be transmitted , but starting with bit 0 (the transmission is made starting with the bit of least weight, 0, up to the highest, 7). The frame ends with a bit to 1 , called a stop bit .

Thus, to transmit a data, the line (tx) will take the following values. Initially it will be at rest (tx = 1). The start bit is transmitted first, so tx = 0. Then the bit with the lowest weight: tx = bit0, then the next bit, tx = bit1, and the next tx = bit2 until the one with the highest weight tx = bit7 . Finally, the stop bit is sent, setting tx = 1. tx Remains at 1 until the next frame is sent.

As an example we will see how to transmit the ASCII character "K" . Its value in hexadecimal is 0x4B and in binary: 01001011

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T21-baud-tx/images/k-car.png

The transmission line over time will have this form:

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T21-baud-tx/images/serial-frame-format.png

All bits have the same duration , which we will call bit period (Tb)

传输速度：波特
================

The transmission speed is measured in baud . Since we are using a binary transmission, in which there are only two values ​​(0 and 1), a baud equals one bit per second (bps)

So that different circuits can communicate with each other, the speeds are normalized . They can have the following values: 115200, 56700, 38400, 19200, 9600, 4800, 2400, 1200, 600 and 300 baud. We will set it to the maximum: 115200 baud

To transmit at a rate of X baud , we need to generate a square signal whose frequency is equal to X. Each rising edge of this signal indicates when to send the next bit:

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T21-baud-tx/images/serial-frame-3.png

用于传输的时钟信号产生器
=========================

The first thing we need to transmit data is to generate the clock signal with the appropriate frequency. We already know how to do this: we will use a frequency divider .

When we work with the iCEstick board, the divisor to get a B baud rate is given by the equation: M = 12000000 / B

Therefore, to transmit to 115200 baud we need a divisor of: M = 12000000/115200 = 104.16 -> M = 104

The values ​​of the dividers to transmit at the standard speeds are calculated with the previous formula and are in the file baudgen.vh ::

 // - File baudgen.vh
 `define B115200 104
 `define B57600 208
 `define B38400 313

 `define B19200 625
 `define B9600 1250
 `define B4800 2500
 `define B2400 5000
 `define B1200 10000
 `define B600 20000
 `define B300 40000

To generate the clock signal to transmit at a speed (for example 115200 baud) is as simple as instantiating the divisor that we already know using the previous constants::

  divider # (`B115200)
   BAUD0 (
     .clk_in (clk),
     .clk_out (clk_baud)
   );

例1：发送字母'K'
=================

This first example sends the "K" character from the FPGA to the computer every time the dtr signal goes from 0 to 1, at the speed of 115200 baud

功能
~~~~

To carry out a transmission of the data we will use a displacement register , with parallel load . The serial output is connected directly to the transmission line tx , through a multiplexer. When the load signal is at 0, the register is loaded with a value of 10 bits: the data "K", followed by the bits 01 (start bit and idle bit).

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T21-baud-tx/images/baudtx-1.png

When the load is at 1 the displacement is made to the right and on the left a 1. is inserted. In this way, when transmitting the complete frame, the least significant bit will be a 1, so that the line tx remains at 1 (in repose)

The output multiplexer puts the transmission line at rest (a 1) when the record is being loaded. It is used so that "junk" characters are not sent on the tx line at boot time. The displacement register initially has the value 0 in all its bits, so tx would be set to 0 and receiver would interpret it as being a start bit, receiving a "garbage" character that did not want to be sent.

To do the tests, load is connected to the dtr signal , so we can control it manually from the pc . When you set it to 1, the transmission begins. When the K character has been sent, the shift register will have all its bits to 1 and what goes out through tx will always be a 1. That is, there will be no transmission. When putting load to 0, it is loaded with the new value, and when it passes to 1 the new data will be sent.

baudtx.v: 硬件描述
~~~~~~~~~~~~~~~~~~~~

上述电路Verilog硬件描述如下::

 // - File: baudtx.v
 `default_nettype none

 `include " baudgen.vh "

 // --- Module that sends a character when load is 1
 module baudtx ( input wire clk, // - System clock (12MHz in ICEstick)
               input wire load, // - Load / shift signal
               output wire tx // - Serial data output (to the PC)
              );

 // - Parameter: transmission speed
 parameter BAUD = `B115200;

 // - 10-bit register to store the frame to send:
 // - 1 bit start + 8 bits data + 1 bit stop
 reg [ 9 : 0 ] shifter;

 // - Watch for transmission
 wire clk_baud;

 // - Movement register, with parallel load
 // - When DTR is 0, the frame is loaded
 // - When DTR is 1, it moves to the right, and
 // - enter '1's on the left
 always @ ( posedge clk_baud)
   if (load == 0 )
     shifter <= { "K" , 2'b01 };
   else
     shifter <= { 1'b1 , shifter [ 9 : 1 ]};

 // - Remove by tx the least significant bit of the displacement records
 // - When we are in load mode (dtr == 0), a 1 is always output for
 // - that the line is always in a resting state.  In this way in the
 // - start tx is at rest, although the value of the shift register
 // - be unknown
 assign tx = (load)?  shifter [ 0 ]: 1 ;

 // - Splitter to get the transmission clock
 divider # (BAUD)
   BAUD0 (
     .clk_in (clk),
     .clk_out (clk_baud)
   );

 endmodule

第一行代码是新的::

 `default_nettype none

By default in Verilog, if labels do not declare they are assumed to be wires (wire type). This might seem very useful but it is a source of problems in debugging. When the design is complex and there are many cables, it can happen that one of them is written wrong. The compiler, instead of giving an error, will assume that it is a new cable. This behavior can be changed with the previous instruction. When defining the cable type to none , every time an undeclared identifier is detected, an error message will be output

In the component, the divisor is instantiated to generate the clock signal to transmit at 115200 baud. This signal is used as a clock for the shift register 

模拟
~~~~

The test bank instantiates the baudtx component, a process is established to generate the relog and another one that generates 3 pulses in dtr so that 3 characteres are sent

要模拟，我们执行::

 $ make sim

结果是：

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T21-baud-tx/images/baudtx-1-sim.png

The first signal is the clock for the transmission of the bits at 115200 baud. When dtr is reset the record is loaded. When you set to 1 you start sending the data in series. In the screenshot you can see 2 pulses in dtr and how after them you start to send the data in series

综合和测试
~~~~~~~~~~

要综合，我们执行::

 $ make sint

占用的资源：

========   ======
  资源       占用
========   ======
  IOPs      6/96
  PLBs      8/160
  BRAMs     0/16
========   ======

我们执行以下命令加载到FPGA::

  $ iceprog baudtx.bin

To test it, we start the gtkterm and configure it so that the port is / dev / ttyUSB1 at the speed of 115200 baud . With the F7 key we change the state of the DTR signal . The first time we press a "K" will appear on the screen. The next time DTR changes state, nothing happens. If we press again another "K" will appear. Every two presses of F7 we will obtain a "K". If we leave F7 pressed, multiple "K" s will appear

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T21-baud-tx/images/baudtx-1-gtkterm.png


例2：连续发送
==============

With this example we check if the transmitter works correctly at the maximum speed , transmitting one character immediately after the other . Every time the DTR signal is set to 1, the character K is transmitted constantly

功能
~~~~

The continuous transmission schedule for character K is shown in this figure:

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T21-baud-tx/images/serial-frame-4.png

Dotted vertical lines show the beginning and end of sent characters. However, if we do not have those visual references, it is impossible to distinguish where the plot begins and ends . The same happens to the serial receiver of the PC. If we make the FPGA send characters constantly and start the gtkterm program, the reading will start at any point in the frame. If this happens, it will be interpreted as a start bit other than the correct one , receiving a burst of incorrect characters. In the drawing, another point has been marked in red, called A, which could be read as a start bit. If this happens, both receiver and transmitter will be desynchronized until the line reaches the idle state.

It is one of the reasons why the DTR signal is used to start the data transmission in example. This way we guarantee that there will always be synchronization.

To achieve the transmission, just make a small change in the previous example: Now instead of entering "1" s on the left of the shift register, we will insert the sent bits , connecting the ser_out with ser_in , so that we build a ring in which all the bits of the frame are constantly being sent

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T21-baud-tx/images/baudtx2-1.png

baudtx2.v: 硬件描述
~~~~~~~~~~~~~~~~~~~~~~~

新的Verilog描述如下::

 `default_nettype none

 `include " baudgen.vh "

 // --- Module that sends a character when load is 1
 module baudtx2 ( input wire clk, // - System clock (12MHz in ICEstick)
                input wire load, // - Load / shift signal
                output wire tx // - Serial data output (to the PC)
               );

 // - Parameter: transmission speed
 parameter BAUD = `B115200;

 // - 10-bit register to store the frame to send:
 // - 1 bit start + 8 bits data + 1 bit stop
 reg [ 9 : 0 ] shifter;

 // - Watch for transmission
 wire clk_baud;

 // - Movement register, with parallel load
 // - When DTR is 0, the frame is loaded
 // - When DTR is 1, it moves to the right, and
 // - enter '1's on the left
 always @ ( posedge clk_baud)
   if (load == 0 )
     shifter <= { "K" , 2'b01 };
   else
     shifter <= {shifter [ 0 ], shifter [ 9 : 1 ]};

 // - Remove by tx the least significant bit of the displacement records
 // - When we are in load mode (dtr == 0), a 1 is always output for
 // - that the line is always in a resting state.  In this way in the
 // - start tx is at rest, although the value of the shift register
 // - be unknown
 assign tx = (load)?  shifter [ 0 ]: 1 ;

 // - Splitter to get the transmission clock
 divider # (BAUD)
   BAUD0 (
     .clk_in (clk),
     .clk_out (clk_baud)
   );

 endmodule

和之前例子不同的地方是这行::

 shifter <= {shifter [ 0 ], shifter [ 9 : 1 ]};

Now on the left you insert shifter [0] instead of a bit to 1 

模拟
~~~~

The test bench is very similar to the previous example. The timing of the dtr signal has been changed::

 `include " baudgen.vh "

 module baudtx2_tb ();

 // - Register to generate the clock signal
 reg clk = 0 ;

 // - Transmission line
 wire tx;

 // - Simulation of the dtr signal
 reg dtr = 0 ;

 // - Instance the component to work at 115200 baud
 baudtx2 # (`B115200)
   dut (
     .clk (clk),
     .load (dtr),
     .tx (tx)
   );

 // - Clock generator.  Period 2 units
 always
   # 1 clk <= ~ clk;


 // - Process at the beginning
 initial begin

   // - File to store the results
   $ dumpfile ( "baudtx2_tb.vcd" );
   $ dumpvars ( 0 , baudtx2_tb);

   // - First shipment: upload and send
   # 10 dtr <= 0 ;
   # 300 dtr <= 1 ;

   // - Second shipment
   # 10000 dtr <= 0 ;
   # 2000 dtr <= 1 ;

   // - Third shipment
   # 10000 dtr <= 0 ;
   # 2000 dtr <= 1 ;

   # 5000 $ display ( "END of the simulation" );
   $ finish ;
 end

 endmodule

要进行模拟，我们执行::

 $ make sim2

模拟结果是：

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T21-baud-tx/images/baudtx-2-sim.png

When dtr is set to 1, continuous transmission is observed. When it is set to 0 it stops transmitting and the line remains at rest. Again it is set to 1 and another transmission burst begins

综合和测试
~~~~~~~~~~~

要综合，我们执行::

 $ make sint2

占用的资源：

========   ======
  资源       占用
========   ======
  IOPs      6/96
  PLBs      8/160
  BRAMs     0/16
========   ======

我们执行以下命令加载到FPGA::

 $ iceprog baudtx2.bin

Open the gtkterm and press F7 to change the DTR. They will begin to appear Ks filling quickly the complete screen of the terminal:

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T21-baud-tx/images/baudtx-2-gtkterm.png

Pressing F7 again the burst stops

例3：周期传输
=============

在这个例子中，我们每250ms发送字母K，不用DTR.

功能
~~~~

This circuit has no external input for loading the register , but a divider has been added to generate a 250ms signal and to be periodically loaded. In this way, every 250ms, the character K is sent:

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T21-baud-tx/images/baudtx3-1.png

The load signal is not connected directly from the splitter, but is passed through a register with clock input clk_baud . It has been added so that the load signal is synchronized with that of clk_baud , and that its relative distances are always the same. If it is not set, situations may occur where the rise edges are too close and incorrect data is captured

baudtx3.v: 硬件描述
~~~~~~~~~~~~~~~~~~~~~

这个电路的Verilog描述为::

 // - baudtx3.v
 `default_nettype none

 `include " baudgen.vh "
 `include " divider.vh "

 // --- Module that sends a character when load is 1
 module baudtx3 ( input wire clk, // - System clock (12MHz in ICEstick)
                output wire tx // - Serial data output (to the PC)
               );

 // - Parameter: transmission speed
 parameter BAUD = `B115200;
 parameter DELAY = `T_250ms;

 // - 10-bit register to store the frame to send:
 // - 1 bit start + 8 bits data + 1 bit stop
 reg [ 9 : 0 ] shifter;

 // - Watch for transmission
 wire clk_baud;

 // - Periodic load signal
 wire load;

 // - Load signal synchronized with clk_baud
 reg load2;

 // - Movement register, with parallel load
 // - When DTR is 0, the frame is loaded
 // - When DTR is 1, it moves to the right, and
 // - enter '1's on the left
 always @ ( posedge clk_baud)
   if (load2 == 0 )
     shifter <= { "K" , 2'b01 };
   else
     shifter <= { 1'b1 , shifter [ 9 : 1 ]};

 // - Remove by tx the least significant bit of the displacement records
 // - When we are in load mode (dtr == 0), a 1 is always output for
 // - that the line is always in a resting state.  In this way in the
 // - start tx is at rest, although the value of the shift register
 // - be unknown
 assign tx = (load2)?  shifter [ 0 ]: 1 ;

 // - Synchronize the load signal with clk_baud
 // - If it is not done, 1 of each X characters sent will have some bit
 // - corrupt (in the tests with the icEstick they went wrong 1 out of every 13 approx.
 always @ ( posedge clk_baud)
   load2 <= load;

 // - Splitter to get the transmission clock
 divider # (BAUD)
   BAUD0 (
     .clk_in (clk),
     .clk_out (clk_baud)
   );

 // - Divider to generate the charging signal periodically
 divider # (DELAY)
   DIV0 (
     .clk_in (clk),
     .clk_out (load)
   );

 endmodule

记录加载信号是load2而不是load，它和clk_baud同步。

模拟
~~~~

The test bench is like the one in the other examples, except that when the baudtx3 component is instantiated, the DELAY parameter is added. In the simulation, another one is placed shorter than 250ms, so that the eternal simulation is not done ::

 // - File baudtx3_tb.v
 `include " baudgen.vh "

 module baudtx3_tb ();

 // - Register to generate the clock signal
 reg clk = 0 ;

 // - Transmission line
 wire tx;


 // - Instance the component to work at 115200 baud
 baudtx3 # (. BAUD (`B115200), .DELAY ( 4000 ))
   dut (
     .clk (clk),
     .tx (tx)
   );

 // - Clock generator.  Period 2 units
 always
   # 1 clk <= ~ clk;


 // - Process at the beginning
 initial begin

   // - File to store the results
   $ dumpfile ( "baudtx3_tb.vcd" );
   $ dumpvars ( 0 , baudtx3_tb);

   # 40000 $ display ( "END of the simulation" );
   $ finish ;
 end

 endmodule

我们用以下命令模拟::

 $ make sim3

结果是：

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T21-baud-tx/images/baudtx3-sim.png

It is observed how a new data is sent with each load pulse. You can also observe how the load and load2 signals are slightly different. The distance between them is varying because load is asynchronous , but load2 is already synchronized.

综合和测试
~~~~~~~~~~~

要综合，我们执行::

 $ make sint3

占用的资源：

========   ======
  资源       占用
========   ======
  IOPs      6/96
  PLBs      19/160
  BRAMs     0/16
========   ======

我们执行以下命令加载到FPGA::

 $ iceprog baudtx3.bin

gtkterm启动的时候，字母K会在屏幕上显示，每秒4个。

如果load信号不同步，你会得到这样的结果：

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T21-baud-tx/images/baudtx-3-gtkterm.png

可以看到有一个字符周期性地没有正确接收。

当所有都同步好了，结果是：

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T21-baud-tx/images/baudtx-3-gtkterm-ok.png


练习
====

* 测试最后一个例子，但修改传送速度为19600和9600波特

致谢
====

* 虽然这个发送器是从头开始写的，我是受到了 James Bowman 的 `swapforth <https://github.com/jamesbowman/swapforth>`__ 项目的UART的启发。非常感谢！
