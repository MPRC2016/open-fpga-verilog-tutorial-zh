第20章：异步串行通信
==========================

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T20-serialcomm-1/images/serialcomm-2.png

简介
----

**异步串行通信** 是一种在两个数字系统之间交换数据的方式。在以下的章节，我们会学习设计我们自己的 **异步串行通信单元** (UART),它让我们可以 **使FPGA和PC及其他微控制器板之间通信** （如Arduino）。

在这章 **我们从最简单的开始：** 通过拉一条FPGA中的线连接接收信道（Rx）和传输信道（Tx）转发所有收到的数据到PC端。我们确保 **物理层在控制之下** 。


异步串行通信
-------------

这种通信方式有个优点，每个方向 **只需要一条线** ：从传输方到接收方。传送方的引脚，数字数据的来源，称为 **Tx**. 接收引脚叫 **Rx**. 比特一个接一个地按顺序发送。

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T20-serialcomm-1/images/serialcomm-1.png

通信可以 **同时发生在两个方向** （全双工）。在这种情况下有 **两条线** ，每个电路都同时成为传输方（Tx）和接收方（Rx），如图所示。

**tx** 和 **rx** 引脚是 **串口** 的一部分。它在PC和不同电路的连接中用得很多。例如，夹在程序到 **Arduino板** 。


iCEstick板的串行通信
------------------------

iCEstick板有一个 **USB-串口转换器** （FTDI芯片）， **Tx** 和 **Rx** 信号通过它到达FPGA **引脚8和9**. 这样，我们在 **PC的串口** 和 **FPGA** 之间有 **直接的连接** ，我们可以实现我们用于数据传输的UART.

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T20-serialcomm-1/images/serialcomm-3.png

**串口** 除了用于数据传输的Tx和Rx外， **还结合了其他控制信号** 。两个常用的是 **RTS** 和 **DTR** ，它们从PC传到FPGA. 它们是通用的2个比特。例如，在Arduino兼容的板子上，PC用其中一个信号来 *重置* 和开始加载程序。

在这个教程，要和iCEstick工作，我们关心的是下图总结的东西：

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T20-serialcomm-1/images/serialcomm-4.png

在我们看来， **我们的PC有一个原生的串口** ，它们的信号 Tx,Rx,DTR,RTS 直接连接FPGA的引脚。


PC上的串行通信
-----------------

To access the serial port of the PC we have to install a communications terminal . With it we can send data to the FPGA, act on the DTR and RTS signals and receive data. The one we will use in this tutorial is the gtkterm , although we can use any other, such as the serial monitor of the Arudino environment.

In ubuntu it is installed very easily with::

  $ sudo apt-get install gtkterm 

To have access permissions to the serial port , we have to put our user inside the dialout group , executing the command::

  $ sudo adduser $USER dialout 

It is necessary to leave the session and re-enter to have effect. If we install the arduino environment, the permissions will be created automatically without using any command

We connect the iCEstick board to the USB and execute the gtkterm::

  $ gtkterm 

If this is the first time we have booted it, we may get an error message. No problem. First we have to configure the serial port by clicking on the configuration/port option and a configuration window will open. We establish as serial port the /dev/ttyUSB1, at the speed of 115200.

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T20-serialcomm-1/images/gtkterm-screenshot-1.png

Note : In ubuntu 15.04, when clicking the iCEstick, 2 serial ports appear. The one that allows us to access the FPGA as a serial port is /dev/ttyUSB1

We give OK. To remember this configuration we click on the configuration/save configuration option and we give it to the ok.

The terminal looks like this:

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T20-serialcomm-1/images/gtkterm-screenshot-2.png


实验1：检查tx,rx,dtr和rts
-----------------------------

我们检查所有信号正确工作。为此，我们制作一个电路，它简单地做一个 **信号间的连线** ： **RTS和DTR连接到LED的引脚** 。这样让我们可以观看它的状态，并在PC端改变它。 **Rx** 信号 **物理上连接到Tx** ，这样所有收到的字符转发回PC端（FPGA中没有字符处理，它就是一条线缆）。

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T20-serialcomm-1/images/serialcomm-5.png

Verilog的描述很直接::

 // - echowire1.v
 module echowire1 ( input wire dtr,
                  input wire rts,
                  rx input ,
                  output wire tx,
                  output wire D1,
                  output wire D2);

 // - Remove signal dtr and rts by the leds
 assign D1 = dtr;
 assign D2 = rts;

 // - Connect rx to tx so that a "wired" echo is made
 assign tx = rx;

 endmodule

我们用以下命令综合::

  $ make sint

占用的资源：

========   ======
  资源       占用
========   ======
  IOPs      4/96
  PLBs      0/160
  BRAMs     0/16
========   ======

要加载到FPGA，我们执行::

  $ iceprog echowire1.bin

要证明它，我们启动 **gtkterm**. 按 **F7** 修改 **DTR** 的状态，从而改变了 **LED**. 而 **F8** 则是改变 **RTS** 信号。

任何我们按的键会发到FPGA并 **回显** 。终端在屏幕上显示所有收到的东西。 **结果是我们会在屏幕上看到我们写的所有东西。**

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T20-serialcomm-1/images/gtkterm-screenshot-3.png


模拟
~~~~

我们做一个基础的模拟来检查所有东西是有序工作的。test bench如下::

 // - File echowire1_tb.v
 module echowire1_tb ();

 // - Declaration of the cables
 reg dtr = 0 ;
 reg rts = 0 ;
 reg rx = 0 ;
 wire tx, led1, led2;

 // - Instance the component
 echowire1
   dut (
     .dtr (dtr),
     .rts (rts),
     .D1 (led1),
     .D2 (led2),
     .tx (tx),
     .rx (rx)
   );

 // - Generate changes in dtr.  They must be reflected on the D1 cable
 always
   # 2 dtr <= ~ dtr;

 // - Generate changes in rts.  They should be reflected on the D2 cable
 always
   # 3 rts = ~ rts;

 // - Generate changes in rs.  They are reflected in TX
 always
   # 1 rx <= ~ rx;

 // - Process at the beginning
 initial begin

   // - File to store the results
   $ dumpfile ( "echowire1_tb.vcd" );
   $ dumpvars ( 0 , echowire1_tb);

   # 200 $ display ( "END of the simulation" );
   $ finish ;
 end

 endmodule

要模拟我们执行::

 $ make sim

结果是：

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T20-serialcomm-1/images/echowire1-sim.png

必须有相同波形的信号已经按颜色分组。可以验证所有东西都符合预期地工作。


实验2：用外部线缆连接tx和rx
-------------------------------

我们修改之前的例子，从而 **Tx和Rx之间的通过一条外部线缆连接** 。这样，如果我们去掉了线缆，通信会中断。我们一放上线缆，回显会回来。

这个部件的方案是：

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T20-serialcomm-1/images/serialcomm-6.png

它的Verilog描述::

 // - echowire2.v
 module echowire2 ( input wire dtr,
                  input wire rts,
                  rx input ,
                  input wire tx2,
                  output wire tx,
                  output wire rx2,
                  output wire D1,
                  output wire D2);

 // - Remove signal dtr and rts by the leds
 assign D1 = dtr;
 assign D2 = rts;

 // - Take tx and rx signals abroad
 assign rx2 = rx;
 assign tx = tx2;

 endmodule 

我们用以下命令综合::

  $ make sint2

占用的资源：

========   ======
  资源       占用
========   ======
  IOPs      5/96
  PLBs      0/160
  BRAMs     0/16
========   ======

要加载到FPGA，我们执行::

  $ iceprog echowire2.bin

测试和之前的例子一样，但是我们现在放置一条连接引脚44和45的外部线缆：

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T20-serialcomm-1/images/echowire2-icestick.png

线缆连接时，回显正常。如果我们去掉了线缆，通信会中断，正如你可以在下面的Youtube视频看到的一样。

https://www.youtube.com/watch?v=fDhtMBKYPBg

模拟
~~~~

test bench的代码和前面的一样，不过增加了一条“外部线缆”::

 module echowire2_tb ();

 // - Declaration of the cables
 reg dtr = 0 ;
 reg rts = 0 ;
 reg rx = 0 ;
 wire tx, led1, led2;
 wire tx2, rx2;
 wire extwire;

 // - Instance the component
 echowire2
   dut (
     .dtr (dtr),
     .rts (rts),
     .D1 (led1),
     .D2 (led2),
     .tx2 (tx2),
     .rx2 (rx2),
     .tx (tx),
     .rx (rx)
   );

 // - Generate changes in dtr.  They must be reflected on the D1 cable
 always
   # 2 dtr <= ~ dtr;

 // - Generate changes in rts.  They should be reflected on the D2 cable
 always
   # 3 rts = ~ rts;

 // - Generate changes in rs.  They are reflected in TX
 always
   # 1 rx <= ~ rx;

 // - Connect the external cable
 assign tx2 = rx2;

 // - Process at the beginning
 initial begin

   // - File to store the results
   $ dumpfile ( "echowire2_tb.vcd" );
   $ dumpvars ( 0 , echowire2_tb);

   # 200 $ display ( "END of the simulation" );
   $ finish ;
 end

 endmodule

要模拟我们执行::

 $ make sim2

结果是：

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T20-serialcomm-1/images/echowire2-sim.png


提议的练习
-----------

* **高级** ：用Python写一个程序，用 **pyserial** 库在DTR产生一个时钟信号。这对于从PC控制电路的速度很有用。

总结
----

我们有了可以开始实现我们的串行通信单元(UART)的所有东西。
