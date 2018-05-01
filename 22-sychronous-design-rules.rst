第22章：同步设计规则
=====================

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T22-syncrules/images/sync-corazon.png

简介
----

This chapter explains some synchronous design rules for the creation of complex digital circuits . Its application will avoid problems as the complexity of our circuits grows. We will apply them to improve the serial transmitter that was outlined in the previous chapter

同步系统
--------

Complex digital systems are synchronous : there is a clock signal (clk) that marks the times when the rest of the signals change. It is a "heart" clock , at the rhythm of whose pulses the bits are transported and modified. On the iCEstick board, our clock is 12Mhz

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T22-syncrules/images/sync-corazon.png

When designing synchronous digital circuits it is convenient to have a series of rules in the head , which will save us many problems, especially when more complex designs are made. Some of these rules have been violated in the examples presented so far. These are simple examples, which have allowed us to introduce concepts and test quickly in our FPGA. It's time to learn more powerful design rules , and apply them

But first let's learn a little more about the origin of the problems

数字电路中的问题
---------------------

罪恶之源：延迟
~~~~~~~~~~~~~~~~~~

The problems in the designs are caused by the delay in the signals , due to the propagation and the processing time of the logic gates. In the digital world we design the functionality assuming that the digital signals are perfect, that they change instantaneously and all the bits at the same time.

The reality is different. So for example, if we are using a NOT gate and we simulate it, we see the ideal case, in which exit B is denied from A at all times.

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T22-syncrules/images/retardo-not.png

In reality, output B is delayed with respect to the input (in addition to that the change from 0 to 1 is not instantaneous). And why is it a problem? Because looking at the timeline, we see that there is a moment of time where B is NOT met B is the denied of A. It happens that A = 1 and B = 1, which means that the BOOLEAN EQUATIONS of the digital logic are NOT FULFILLED in that interval of time (this must be in the head). In addition, these delays cause transient spurious pulses (glitches)

Spurious pulses (Glitches)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Each signal from a digital circuit has a different delay depending on the length of its cables and the doors it traverses. Thus, two input signals will arrive at the same door at different times. This causes the appearance of spurious pulses , which disappear in the permanent regime.

We will see its effect on a simple combinational circuit: an XOR gate with two inputs. The two inputs are at 0 (and therefore the output remains at 0). In an instant the input signals A and B change to 1. Ideally, what is obtained by the output is also 0, since 1 xor 1 is equal to 0.

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T22-syncrules/images/glitches-xor.png

However, in this example, due to the delays, the signal B has arrived a little later than the A. This causes that there is a moment in which A = 1 and B = 0, so the output is C = 1 Finally, when B = 1 the output is set to its stable value: C = 0.

The result is that a pulse has appeared in a signal that was supposed to always be at 0

Imagine what can happen if we have this circuit connected to the counter input of a counter 

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T22-syncrules/images/xor-contador.png

Due to the spurious pulse, the accountant will make a false account! It will count when it does NOT have to count! The designer, no matter how much he looks and remorses the digital design, will see that everything is correct. And ideally simulating everything will work fine ... but loading it on the FPGA: wow! It will count badly. These errors are very difficult to detect

To avoid these problems is why we use the synchronous design rules

同步设计规则
-------------

规则是：

规则1：一个独立时钟管理一切
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

ALL clock inputs must be connected directly to a single clock , common to all. This rule has been violated in almost all the examples of this tutorial until now, so that they were simple. From now on all the examples will respect this rule

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T22-syncrules/images/regla-2-unico-reloj.png

This rule forbids us to connect anything other than the system clock to the clock input . So that we can not connect any combinational circuit (for example, the xor gate of the explanation of the spurious pulses) nor sequentially (for example, the output of a counter to use it as a divider). In the following sections we will see what to add to the circuits so that they comply with this rule.

规则2：Sensitivity to the same flank: Everyone to one, fuenteovejuna!
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

All the elements that have a clock will be sensitive to the same flank. Well the rise or fall, is indifferent, but all sensitive to the same

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T22-syncrules/images/regla-1-mismo-flanco.png

Rule 3: Before entering a combinational circuit, go through registration, please
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

All the inputs of the combinational circuits must be connected to sequential circuit outputs , synchronized with the system clock, or to other combinational circuits that comply with this rule . That is, any input of a combinational circuit has to be captured before by a record. Entries that meet this rule are called synchronized entries

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T22-syncrules/images/regla-3-entradas-sincronizadas.png

Rule 4: Before entering a sequential circuit, go through registration please
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

All the inputs of the sequential circuits must come from the outputs of other sequential or combinational circuits that comply with rule 3 . That is, even to enter the sequential circuits, it is necessary that the signals are synchronized .

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T22-syncrules/images/regla-4-entradas-sec-sincronizadas.png

Combining the rules 3 and 4, we conclude that any input to our synchronous circuit has to be registered .

Rule 5: Outputs of a combinational: Only to inputs of another combinational, synchronous inputs or outputs of the synchronous circuit
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This rule tells us where we can connect the output of a combinational circuit . Well at the entrance of another combinacional , to the synchronous input of a sequential or as a direct output of our circuit . It is strictly forbidden to connect them to the inputs of the combinational itself (no direct feedback) or to the input of the clock signal

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T22-syncrules/images/regla-5-salidas-combinacionales.png

改善串口发送器
------------------------

We will redesign the examples of the serial transmitter to comply with the rules

例1：txtest.v
~~~~~~~~~~~~~~~~~~~~~~

We start from the baudtx example, where the character "K" is sent with the rising edge of the DTR signal

Applying the synchronous design rules
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The scheme of the original circuit is the following:

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T22-syncrules/images/baudtx-1-errors.png

We observe several violations of the synchronous design rules:

1. There is not a single global clock (rule 1 is not respected). The clock signal does not enter directly in the shift register. The signal from the divider can NOT enter directly through the clock input but there must be a new synchronized input. For the divider to work properly, the output pulse width must be 1 period of length.

2. The load input is NOT registered (rule 3 is violated)

3. The TX output comes from a combinational circuit and can be removed directly without violating any rule. However, being connected to an asynchronous bus , any spurious pulse sent will be interpreted as a data and will cause an error in communications . Therefore it is safer to record this output. 

Baudgen.v: Modification of the divisor
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We will design a specific divider to use in the transmitter . Since the output of the divider can NOT be entered directly by the clock input of the transmitter's shift register (prohibited by the synchronous design rules), it will be connected to a new synchronous input. However, it will be necessary for the pulse to be only 1 period of length

As an improvement, we will put an enable signal of the divisor ( clk_ena ) so that when clk_ena = 1 the pulses will be produced and otherwise the output (clk_out) will remain at 0

The scheme of the circuit is the following:

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T22-syncrules/images/baudgen-diagram.png

and the description in verilog::

 // - File baudgen.v
 `include " baudgen.vh "

 // - TICKETS:
 // - -clk: System clock signal (12 MHZ on the iceStick)
 // - -clk_ena: Enabling.
 // - 1. normal operation.  Emitting pulses
 // - 0: Initialized and stopped.  No pulses are emitted
 //
 // - DEPARTURES:
 // - - clk_out.  Output signal that marks the time between bits
 // - Width of 1 period of clk.  UNREGISTERED DEPARTURE
 module baudgen ( input wire clk,
                input wire clk_ena, 
                output wire clk_out);

 // - Default value of the baud rate
 parameter M = `B115200;

 // - Number of bits to store the baud divider
 localparam N = $ clog2 (M);

 // - Register to implement the module meter M
 reg [N - 1 : 0 ] divcounter = 0 ;

 // - Module counter M
 always @ ( posedge clk)

   if (clk_ena)
     // - Normal operation
     divcounter <= (divcounter == M - 1 )?  0 : divcounter + 1 ;
   else
     // - Counter "frozen" to the maximum value
     divcounter <= M - 1 ;

 // - Take out a pulse width 1 clock cycle if the generator
 // - is enabled (clk_ena == 1)
 // - otherwise 0 is taken out
 assign clk_out = (divcounter == 0 )?  clk_ena: 0 ;

 endmodule 

The operation of the new divisor can be seen in this schedule . In the next cycle to be activated clk_enable , a pulse of 1 period of length appears, which is repeated every M clock cycles. When clk_ena is 0, no pulses are generated

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T22-syncrules/images/baudgen-chronogram.png

txtest.v: 发送器的修改
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The new scheme is shown below. In addition to complying with the rules of the synchronous design , an improvement has been added: since the flank is received until it starts transmitting, only 1 clock cycle elapses . Before, we had to wait for a 1 in the baud signal, so this time varied. To achieve this, baudgen is controlled through its new enabling entry: clk_ena . When there is no information transmission, the baud signal is disabled.

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T22-syncrules/images/txtest-diagram.png

The new displacement register , in verilog is the following::

 always @ ( posedge clk)
   // - Load mode
   if (load_r == 0 )
     shifter <= { "K" , 2'b01 };

   // - Scroll mode
   else if (load_r == 1 && clk_baud == 1 )
     shifter <= { 1'b1 , shifter [ 9 : 1 ]};

Now it depends on the global clock signal , to comply with rule 1. It is also on the rising edge , because the rest of the circuits we have done so and by rule 2 they all have to be sensitive to the same flank

The load signal used is load_r , which is the registered load signal::

 always @ ( posedge clk)
   load_r <= load;

The register shifts the bits to the right when we are not in load mode (load_r == 1) and the signal clk_baud is at 1 . For this reason, the clk_baud signal can only be at 1 during 1 clock cycle . If the pulse width were greater, a displacement would be made with each global clock edge, the timing being lost.

The output TX output is registered , to ensure that it is completely clean on the asynchronous bus and that spurious pulses are not sent::

 always @ ( posedge clk)
   tx <= (load_r)?  shifter [ 0 ]: 1 ; 

The complete code is shown below::

 // - File: txtest.v
 `default_nettype none

 `include " baudgen.vh "

 // --- Module that sends a character when load is 1
 // --- The output tx IS REGISTERED
 module txtest ( input wire clk, // - System clock (12MHz in ICEstick)
               input wire load, // - Load / shift signal
               output reg tx // - Serial data output (to the PC)
              );

 // - Parameter: transmission speed
 // - Evidence of the worst case: at 300 baud
 parameter BAUD = `B300;

 // - 10-bit register to store the frame to send:
 // - 1 bit start + 8 bits data + 1 bit stop
 reg [ 9 : 0 ] shifter;

 // - Registered load signal
 reg load_r;

 // - Watch for transmission
 wire clk_baud;

 // - Register the load entry
 // - (to comply with the synchronous design rules)
 always @ ( posedge clk)
   load_r <= load;

 // - Movement register, with parallel load
 // - When load_r is 0, the frame is loaded
 // - When load_r is 1 and the baud clock is 1 it moves towards
 // - the right, sending the next bit
 // - '1's are introduced on the left
 always @ ( posedge clk)
   // - Load mode
   if (load_r == 0 )
     shifter <= { "K" , 2'b01 };

   // - Scroll mode
   else if (load_r == 1 && clk_baud == 1 )
     shifter <= { 1'b1 , shifter [ 9 : 1 ]};

 // - Remove by tx the least significant bit of the displacement records
 // - When we are in load mode (load_r == 0), a 1 is always output for
 // - that the line is always in a resting state.  In this way in the
 // - start tx is at rest, although the value of the shift register
 // - be unknown
 // - IS A REGISTERED OUTPUT, since tx connects to a synchronous bus
 // - and you have to avoid spurious pulses (glitches)
 always @ ( posedge clk)
   tx <= (load_r)?  shifter [ 0 ]: 1 ;

 // - Splitter to get the transmission clock
 baudgen # (BAUD)
   BAUD0 (
     .clk (clk),
     .clk_ena (load_r),
     .clk_out (clk_baud)
   );

 endmodule

模拟
~~~~

The test bench has been improved to simulate at any speed . By default the simulation is at 115200 baud so that it runs quickly. If we simulate 300 baud, the times are longer so you have to use a longer simulation time and it will take more time to complete.

The test bench generates 2 pulses in the dtr signal . The circuit must send the K character on the serial line. The code is::

 // - File txtest.v
 `include " baudgen.vh "

 module txtest_tb ();

 // - Baud with which to perform the simulation
 // - At 300 baud, the simulation takes longer to complete because the
 // - times are longer.  At 115200 baud the simulation is much
 // - faster
 localparam BAUD = `B115200;

 // - Clock tics for sending data at that speed
 // - Multiply by 2 because the clock period is 2 units
 localparam BITRATE = (BAUD << 1 );

 // - Ticks needed to send a complete serial frame, plus an additional bit
 localparam FRAME = (BITRATE * 11 );

 // - Time between two sent bits
 localparam FRAME_WAIT = (BITRATE * 4 );

 // - Register to generate the clock signal
 reg clk = 0 ;

 // - Transmission line
 wire tx;

 // - Simulation of the dtr signal
 reg dtr = 0 ;

 // - Instance the component
 txtest # (. BAUD (BAUD))
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
   $ dumpfile ( "txtest_tb.vcd" );
   $ dumpvars ( 0 , txtest_tb);

   # 1 dtr <= 0 ;

   // - Send first character
   #FRAME_WAIT dtr <= 1 ;
   #FRAME dtr <= 0 ;

   // - Second shipment
   #FRAME_WAIT dtr <= 1 ;
   #FRAME dtr <= 0 ;

   #FRAME_WAIT $ display ( "END of the simulation" );
   $ finish ;
 end

 endmodule

我们用以下命令模拟::

 $ make sim

gtkwave中的结果是：

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T22-syncrules/images/txtest-1-sim.png

We observe that when a rising edge arrives in dtr , the circuit sends the character. We also see that the clock signal of the bits is only working at the moment of transmitting the bits. As soon as dtr is set to 0 the signal stops.

综合和测试
~~~~~~~~~~~~~~

我们用以下命令综合::

 $ make sint

所用的资源有：

========   ======
  资源       占用
========   ======
  IOPs      5/96
  PLBs      17/160
  BRAMs     0/16
========   ======

我们执行以下命令加载到FPGA::

 $ iceprog txtest.bin

By default the tests are at the speed of 300 baud , which is the slowest. We open the gtkterm. Every time we send a pulse to the DTR signal, the character K is received：

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T22-syncrules/images/txtest-1-gtkterm.png

If we leave the F7 key pressed, Ks will be continuously received. You can see how an erroneous character is received from time to time . This will depend on the PC in which we test it. The problem is that if for some reason the DTR signal is deactivated while a character is being sent , what will be received is "garbage". This will be something that we will have to solve in the next chapter by introducing a controller , implemented with finite automata.

Moving to the hexadecimal view mode (View / Hexadecimal) you can see exactly what the garbage character received is : 0xCB , when the 0x4B should be received. Analyzing the bits it is observed that the change is in the bit of greater weight , which changes from 0 to 1, converting 4B into CB. Just the most important bit is the last one that is sent

txtest2.v: 连续传输的例子
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This example is similar to the previous one, but the serial output of the shift register is fed back by the input , so that when the signal dtr is at 1, a continuous transmission of the character K is performed . This allows to verify that it works correctly at its maximum speed

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T22-syncrules/images/txtest2-diagram.png

The code is the same as that of txtest.v, but slightly changing the shift register::

 always @ ( posedge clk)
   // - Load mode
   if (load_r == 0 )
     shifter <= { "K" , 2'b01 };

   // - Scroll mode
   else if (load_r == 1 && clk_baud == 1 )
     shifter <= {shifter [ 0 ], shifter [ 9 : 1 ]};

模拟
~~~~

The test bench is similar to the one in the previous example, but the dtr signals are kept at 1 more time, in order to see the continuous transmission::

 `include " baudgen.vh "

 module txtest2_tb ();

 // - Baud with which to perform the simulation
 // - At 300 baud, the simulation takes longer to complete because the
 // - times are longer.  At 115200 baud the simulation is much
 // - faster
 localparam BAUD = `B115200;

 // - Clock tics for sending data at that speed
 // - Multiply by 2 because the clock period is 2 units
 localparam BITRATE = (BAUD << 1 );

 // - Ticks needed to send a complete serial frame, plus an additional bit
 localparam FRAME = (BITRATE * 11 );

 // - Time between two sent bits
 localparam FRAME_WAIT = (BITRATE * 4 );

 // - Register to generate the clock signal
 reg clk = 0 ;

 // - Transmission line
 wire tx;

 // - Simulation of the dtr signal
 reg dtr = 0 ;

 // - Instance the component
 txtest2 # (. BAUD (BAUD))
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
   $ dumpfile ( "txtest2_tb.vcd" );
   $ dumpvars ( 0 , txtest2_tb);

   # 1 dtr <= 0 ;

   // - Send first character
   #FRAME_WAIT dtr <= 1 ;
   # (FRAME * 3 ) dtr <= 0 ;

   // - Second shipment
   #FRAME_WAIT dtr <= 1 ;
   # (FRAME * 3 ) dtr <= 0 ;

   #FRAME_WAIT $ display ( "END of the simulation" );
   $ finish ;
 end

 endmodule

用以下命令模拟::

 $ make sim2

gtkwave的结果是：

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T22-syncrules/images/txtest-2-sim.png

It is observed how they send 3 characters "K" in each burst of activation of the dtr 

综合和测试
~~~~~~~~~~~~~

我们用以下命令综合::

 $ make sint2

所用资源有：

========   ======
  资源       占用
========   ======
  IOPs      5/96
  PLBs      13/160
  BRAMs     0/16
========   ======

我们用以下命令将其加载至FPGA::

 $ iceprog txtest2.bin

The test has been done at 300 baud. When pressing F7 to cut the burst of transmissions, garbage characters are obtained, because the transmission is halfway and aborted

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T22-syncrules/images/txtest-2-gtkterm.png

txtest3.v: 定时传输的例子
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In this example , the character K is transmitted periodically , every 250ms . For this we have added the classic divisor (the one we have been using so far) governed by the global clock. Its output is passed through a register to synchronize the signal and is used as a load signal

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T22-syncrules/images/txtest3-diagram.png

We note that there are 5 elements that have clock input , and ALL of them are connected to the system clock (Rule 1 of synchronous design)

Verilog的描述为::

 // - File txtest3.v
 `default_nettype none

 `include " baudgen.vh "
 `include " divider.vh "

 // --- Module that sends a character periodically
 // --- The output tx IS REGISTERED
 module txtest3 ( input wire clk, // - System clock (12MHz in ICEstick)
                output reg tx // - Serial data output (to the PC)
               );

 // - Parameter: transmission speed
 // - Evidence of the worst case: at 300 baud
 parameter BAUD = `B300;

 // - Parameter: Period of the character
 parameter DELAY = `T_250ms;

 // - 10-bit register to store the frame to send:
 // - 1 bit start + 8 bits data + 1 bit stop
 reg [ 9 : 0 ] shifter;

 // - Registered load signal
 reg load_r;

 // - Watch for transmission
 wire clk_baud;

 // - Periodic signal
 wire load;

 // - Movement register, with parallel load
 // - When load_r is 0, the frame is loaded
 // - When load_r is 1 and the baud clock is 1 it moves towards
 // - the right, sending the next bit
 // - '1's are introduced on the left
 always @ ( posedge clk)
   // - Load mode
   if (load_r == 0 )
     shifter <= { "K" , 2'b01 };

   // - Scroll mode
   else if (load_r == 1 && clk_baud == 1 )
     shifter <= { 1'b1 , shifter [ 9 : 1 ]};

 // - Remove by tx the least significant bit of the displacement records
 // - When we are in load mode (load_r == 0), a 1 is always output for
 // - that the line is always in a resting state.  In this way in the
 // - start tx is at rest, although the value of the shift register
 // - be unknown
 // - IS A REGISTERED OUTPUT, since tx connects to a synchronous bus
 // - and you have to avoid spurious pulses (glitches)
 always @ ( posedge clk)
   tx <= (load_r)?  shifter [ 0 ]: 1 ;

 // - Splitter to get the transmission clock
 baudgen # (BAUD)
   BAUD0 (
     .clk (clk),
     .clk_ena (load_r),
     .clk_out (clk_baud)
   );

 // - Register the load entry
 // - (to comply with the synchronous design rules)
 always @ ( posedge clk)
   load_r <= load;

 // - Divider to generate periodic signal
 divider # (DELAY)
   DIV0 (
     .clk_in (clk),
     .clk_out (load)
   );

 endmodule

模拟
~~~~

The test bench is simpler because there is no need to generate impulses to be introduced by the dtr. Just wait a while to see the signal that comes out by tx::

 `include " baudgen.vh "
 `include " divider.vh "

 module txtest3_tb ();

 // - Baud with which to perform the simulation
 // - At 300 baud, the simulation takes longer to complete because the
 // - times are longer.  At 115200 baud the simulation is much
 // - faster
 localparam BAUD = `B115200;

 // - Clock tics for sending data at that speed
 // - Multiply by 2 because the clock period is 2 units
 localparam BITRATE = (BAUD << 1 );

 // - Ticks needed to send a complete serial frame, plus an additional bit
 localparam FRAME = (BITRATE * 11 );

 // - Register to generate the clock signal
 reg clk = 0 ;

 // - Transmission line
 wire tx;

 // - Instance the component
 txtest3 # (. BAUD (BAUD), .DELAY ( 4000 ))
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
   $ dumpfile ( "txtest3_tb.vcd" );
   $ dumpvars ( 0 , txtest3_tb);

   # (FRAME * 10 ) $ display ( "END of the simulation" );

   $ finish ;
 end

 endmodule

用以下命令模拟::

 $ make sim3

gtkwave的结果是：

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T22-syncrules/images/txtest3-sim.png

You can see that the pulse trains sent by tx are periodic

综合和测试
~~~~~~~~~~~~~~

We make the synthesis with the following command::

 $ make sint3

所用资源有：

========   ======
  资源       占用
========   ======
  IOPs      5/96
  PLBs      25/160
  BRAMs     0/16
========   ======

我们用以下命令将其加载至FPGA::

 $ iceprog txtest3.bin

We open the gtkterm to test it, at the speed of 300 baud. We see how we receive the character K periodically. Now you should not receive any "junk" character since no one is cut in half. However, if the repetition time is reduced (for example to 100ms) then the characters will go wrong (at 300 baud)

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T22-syncrules/images/txtest3-gtkterm.png

发送器缺少的东西
------------------------------

We have not finished the transmitter yet. We already have a part ready: your data path . We lack the controller, which is the one that emits the different signals (microorders) to govern this route. For them we will have to learn to make state machines .

提议的练习
------------------

* 用不同的波特率测试这些例子来检查它们的操作

致谢
----

* `Heart <https://commons.wikimedia.org/wiki/File:Coraz%C3%B3n.svg#/media/File:Coraz%C3%B3n.svg>`__ Licensed under CC BY-SA 3.0 via Wikimedia Commons
* `XOR door <https://commons.wikimedia.org/wiki/File:Puerta_XOR.svg#/media/File:Puerta_XOR.svg>`__ by Dnu72 - Own work. Licensed under CC BY-SA 3.0 via Wikimedia Commons
* `Unico Anello <https://commons.wikimedia.org/wiki/File:Unico_Anello.png#/media/File:Unico_Anello.png>`__ by Xander - own work, (not derivative from the movies). Licensed under Public Domain via Commons
