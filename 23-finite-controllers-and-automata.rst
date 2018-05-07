第23章：有限控制器和自动机
======================================

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T23-fsmtx/images/fsmtx-2.png

Introduction
-------------------

The controllers are part of the digital circuits that are responsible for generating all the signals (microorders) that govern the rest of the components . They are implemented by finite automata (also called state machines ). In this chapter we will design a controller for our serial transmitter .

Generic architecture
-----------------------------

The structure of the digital circuits can be broken down into two parts: the data path and the controller :

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T23-fsmtx/images/fsmtx-3.png

- 数据通路 : It is the part that processes the data . It includes all the functions units together with their connections to perform this processing: displacement registers, arithmetic-logic units, counters, etc.

- 控制器 : It is the part of government , which generates the necessary signals, called microorders , so that the data route behaves appropriately. Send microorders to start a counter, load a data in a register, perform such an arithmetic operation, etc. All this sequenced in time.

The controller makes decisions and generates microorders or others based on the partial results of the data route. For example, if a counter has reached a certain value, then such a signal should generally be made to do such a thing. All that the controller does.

The controllers are implemented by means of finite automata (also called state machines). It is necessary first to define the states of the circuit and the transitions between these states according to the conditions of the circuit. Then, according to these states, the microorders are generated. 

用Verilog写状态机
------------------------

We start from this diagram of 4 states , to explain how to describe it in Verilog:

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T23-fsmtx/images/fsmtx-4.png

Initially the circuit is in state 0 and the controller will generate the necessary microorders (not shown in the drawing). As long as the signal a is at 0, it will always remain in that state. As soon as it is set to 1, it will go to state 1 , where other microorders will be generated. In this state, depending on the value of signal b , either it will return to the initial state or advance to state 2 . State 2 has no conditions , so in the next clock cycle it goes to state 3 . When the signal c is 0, it returns to the initial state.

First we define the labels for the states by means of local parameters , and a record to store the state . Since we have 4 states, this record will be 2 bits::

 localparam STATUS0 = 0 ;
 localparam STATE1 = 1 ;
 localparam STATE2 = 2 ;
 localparam STATUS3 = 3 ;

 reg [ 1 : 0 ] state; 

The transitions between states are modeled by a process that depends on the clock and that upon receiving the reset is initialized in the initial state . Using a case we define the transitions for each state::

 always @ ( posedge clk)
   if (rst)
     // - Set the initial state when starting
     state <= STATUS0;
   else
     case (state)
       STATUS0:
         // - Conditional state change
         if (a) state <= STATUS1;
         else state <= STATUS0;

       STATE1:
         // - Condition conditiconal condition
         if (b) state <= STATUS2;
         else state <= STATE1;

       STATE2:
         // - Change status in the next cycle
         state <= STATE3;

       STATE3:
         // - Condition conditiconal condition
         if (c) state <= STATUS0;
         else state <= STATE3;

       default :
         // - You always have to put it, to avoid
         // - latches in the synthesis
         state <= STATUS0;

     endcase

In all states except STATE2 the transitions are conditional : depending on the state of the signals a, b and c, it jumps to one state or another. STATE2 only lasts 1 cycle . As soon as you enter it, you jump to STATE3 unconditionally , in the next clock cycle.

Now the generation part of the microorders is missing. We will assume that this circuit has 2 microorders: m1 and m2 . Depending on the state you are in, different values will be assigned. We implement it as a combinational process with a case::

 always @ *
   case (state)
     STATE0: begin
       m1 <= 0 ;
       m2 <= 0 ;
     end

     STATE1: begin
       m1 <= 1 ;
       m2 <= 0 ;
     end

     STATE2: begin
       m1 <= 0 ;
       m2 <= 0 ;
     end

     STATE3: begin
       m1 <= 1 ;
       m2 <= 1 ;
     end

     default : begin
       m1 <= 0 ;
       m2 <= 0 ;
     end
   endcase

Using the case is the direct way, but you have to define the value of all the microorders for each state . Through direct assignments the code can be reduced substantially. In the previous example, we see that the microorder m2 is only activated in STATUS3 and in the rest it is deactivated. For m1 we see that it is activated in the STATE1 and STATE3 states. This we can describe it like this::

 assign m1 = (state == STATE1 || state == STATE3)?  1 : 0 ;
 assign m2 = (state == STATE3)?  1 : 0 ;

这样的代码更加紧凑。

改进的串行发送器
---------------------

We will apply this architecture to the serial transmitter, dividing it in its data path and the controller

fsmtx.v: 连续传送
~~~~~~~~~~~~~~~~~~~~~~~

We will make a transmitter that sends the character A when the start signal is set to 1, which we will control through the dtr signal (as in the examples in the previous chapter). However, this signal does not directly control the transmission, but only initiates it . It will be the controller who controls this transmission. In this way, there will never be a character left to transmit as it did in the previous chapter.

The scheme is the following:

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T23-fsmtx/images/fsmtx-1.png

数据通路
`````````

In the data path there is the shift register that makes the serialization of the data, the baud generator , the initializer, a 4-bit counter to track the sent bits and the flip-flops to record signals and comply with the standards of synchronous design

The counter keeps track of how many bits have been sent and will help the controller know when the transmission has finished. The "load" signal serves to reset it , and is the same as the one used to load the shift register 

控制器
```````

The state diagram of the controller is as follows:

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T23-fsmtx/images/fsmtx-2.png

The transmitter can be in 3 states :

- IDLE : Rest state. Nothing is being transmitted. Wait for the start signal to activate to start transmitting the "A" character

- START : Start of transmission. The data is loaded in the shift register. The clock is started for the timing of the bits at the configured speed. The bit counter is set to zero. It is a transient state that lasts 1 clock cycle

- TRANS : Transmission status. It remains in this state during character transmission. As soon as the bit counter reaches the value 11 (10 bits already transmitted), it goes to the initial state of rest

To control the data path, you need 2 microorders , load and baudgen . The first is used to load the data in the shift register and reset the bit counter. The second is the enabling of the bit timer (baud generator)

fsmtx.v: 硬件描述
``````````````````
The complete description of the improved transmitter, in Verilog, is::

 // - File fsmtx.v
 `default_nettype none

 `include " baudgen.vh "

 // --- Module that sends a character when load is 1
 // --- The output tx IS REGISTERED
 module fsmtx ( input wire clk, // - System clock (12MHz in ICEstick)
               input wire start, // - Activate 1 to transmit
               output reg tx // - Serial data output (to the PC)
              );

 // - Parameter: transmission speed
 // - Evidence of the worst case: at 300 baud
 parameter BAUD = `B300;

 // - Character to send
 parameter CAR = "A" ;

 // - 10-bit register to store the frame to send:
 // - 1 bit start + 8 bits data + 1 bit stop
 reg [ 9 : 0 ] shifter;

 // - Registered start signal
 reg start_r;

 // - Watch for transmission
 wire clk_baud;

 // - Reset
 reg rstn = 0 ;

 // - Bitcounter
 reg [ 3 : 0 ] bitc;

 // --------- Microorders
 wire load;  // - Load of the shift register.  Set to 0 of
               // - bit counter
 wire baud_en;  // - Enable the baud generator for transmission

 // -------------------------------------
 // - DATA ROUTE
 // -------------------------------------

 // - Register the start entry
 // - (to comply with the synchronous design rules)
 always @ ( posedge clk)
   start_r <= start;

 // - Movement register, with parallel load
 // - When load_r is 0, the frame is loaded
 // - When load_r is 1 and the baud clock is 1 it moves towards
 // - the right, sending the next bit
 // - '1's are introduced on the left
 always @ ( posedge clk)
   // - Reset
   if (rstn == 0 )
     shifter <= 10'b11_1111_1111 ;

   // - Load mode
   else if (load == 1 )
     shifter <= {CAR, 2'b01 };

   // - Scroll mode
   else if (load == 0 && clk_baud == 1 )
     shifter <= { 1'b1 , shifter [ 9 : 1 ]};

 always @ ( posedge clk)
   if (load == 1 )
     bitc <= 0 ;
   else if (load == 0 && clk_baud == 1 )
     bitc <= bitc + 1 ;

 // - Remove by tx the least significant bit of the displacement records
 // - When we are in load mode (load_r == 0), a 1 is always output for
 // - that the line is always in a resting state.  In this way in the
 // - start tx is at rest, although the value of the shift register
 // - be unknown
 // - IS A REGISTERED OUTPUT, since tx connects to a synchronous bus
 // - and you have to avoid spurious pulses (glitches)
 always @ ( posedge clk)
   tx <= shifter [ 0 ];

 // - Splitter to get the transmission clock
 baudgen # (BAUD)
   BAUD0 (
     .clk (clk),
     .clk_ena (baud_en),
     .clk_out (clk_baud)
   );

 // ------------------------------
 // - CONTROLLER
 // ------------------------------

 // - States of finite controller automata
 localparam IDLE = 0 ;
 localparam START = 1 ;
 localparam TRANS = 2 ;

 // - States of the controller automaton
 reg [ 1 : 0 ] state;

 // - Transitions between states
 always @ ( posedge clk)

   // - Reset of the automata.  To the initial state
   if (rstn == 0 )
     state <= IDLE;

   else
     // - Transitions to the following states
     case (state)

       // - State of rest.  It goes out when the signal
       // - of start is set to 1
       IDLE:
         if (start_r == 1 )
           state <= START;
         else
           state <= IDLE;

       // - State of beginning.  Prepare to start
       // - to transmit.  Duration: 1 clock cycle
       START:
         state <= TRANS;

       // - Transmitting.  It is in this state until
       // - all pending bits have been transmitted
       TRANS:
         if (bitc == 11 )
           state <= IDLE;
         else
           state <= TRANS;

       // - By default.  NOT USED.  Position for
       // - cover all cases and that no latches are generated
       default :
         state <= IDLE;

     endcase

 // - Generation of micro-orders
 assign load = (state == START)?  1 : 0 ;
 assign baud_en = (state == IDLE)?  0 : 1 ;


 // - Initializer
 always @ ( posedge clk)
   rstn <= 1 ;

 endmodule

模拟
````

The test bed is this::

 // - File: fsmtx.v
 `include " baudgen.vh "

 module fsmtx_tb ();

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

 // - Simulation of the start signal
 reg start = 0 ;

 // - Instance the component
 fsmtx # (. BAUD (BAUD))
   dut (
     .clk (clk),
     .start (start),
     .tx (tx)
   );

 // - Clock generator.  Period 2 units
 always
   # 1 clk <= ~ clk;


 // - Process at the beginning
 initial begin

   // - File to store the results
   $ dumpfile ( "fsmtx_tb.vcd" );
   $ dumpvars ( 0 , fsmtx_tb);

   # 1 start <= 0 ;

   // - Send first character
   #FRAME_WAIT start <= 1 ;
   # (BITRATE * 2 ) start <= 0 ;

   // - Second shipment (2 more characters)
   # (FRAME_WAIT * 2 ) start <= 1 ;
   # (FRAME * 1 ) start <= 0 ;

   # (FRAME_WAIT * 4 ) $ display ( "END of the simulation" );
   $ finish ;
 end

 endmodule

用以下命令模拟::

 $ make sim

gtkwave的结果是：

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T23-fsmtx/images/fsmtx-sim-1.png

The top line is the start line . We observe how this signal is set to 0 before the first character ends, but it is still sent. When finished, it is reset to 1 to send the next one. This time it is left up longer, so that 2 more characters are sent (30 bits are sent in total, so there are 30 pulses of the clk_baud signal).

In the lower line you can see the bit counter and the states through which the controller passes

综合和测试
`````````````

我们用以下命令综合::

 $ make sint

所用的资源有：

========   ======
  资源       占用
========   ======
  IOPs      6/96
  PLBs      21/160
  BRAMs     0/16
========   ======

我们执行以下命令加载到FPGA::

 $ iceprog fsmtx.bin

From the gtkterm , every time we give the F7 to modify the DTR signal, it will start to send the A characters (default to the 300 baud rate). If now we press F7 or press it randomly, we will see that no garbage characters appear , because the controller guarantees that the character is always sent

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T23-fsmtx/images/fsmtx-gtkterm-1.png

fsmtx2.v: 定时传输
~~~~~~~~~~~~~~~~~~~~~

This circuit periodically transmits the character "A" every 100ms . The circuit is similar to the previous example but the start signal is taken from a 100ms splitter instead of the external DTR signal

To only send 1 character at a time, the divider is modified to generate a pulse of 1 cycle width .

The scheme of the circuit is: 

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T23-fsmtx/images/fsmtx2-1.png

Verilog描述::

 // - File: fsmtx2.v
 `default_nettype none

 `include " baudgen.vh "
 `include " divider.vh "

 // --- Module that sends a character when load is 1
 // --- The output tx IS REGISTERED
 module fsmtx2 ( input wire clk, // - System clock (12MHz in ICEstick)
               output reg tx // - Serial data output (to the PC)
              );

 // - Parameter: transmission speed
 parameter BAUD = `B115200;

 // - Character to send
 parameter CAR = "A" ;

 // - Shipping time
 parameter DELAY = `T_100ms;

 // - 10-bit register to store the frame to send:
 // - 1 bit start + 8 bits data + 1 bit stop
 reg [ 9 : 0 ] shifter;

 wire start;

 // - Registered start signal
 reg start_r;

 // - Watch for transmission
 wire clk_baud;

 // - Reset
 reg rstn = 0 ;

 // - Bitcounter
 reg [ 3 : 0 ] bitc;

 // --------- Microorders
 wire load;  // - Load of the shift register.  Set to 0 of
               // - bit counter
 wire baud_en;  // - Enable the baud generator for transmission

 // -------------------------------------
 // - DATA ROUTE
 // -------------------------------------

 // - Register the start entry
 // - (to comply with the synchronous design rules)
 always @ ( posedge clk)
   start_r <= start;

 // - Movement register, with parallel load
 // - When load_r is 0, the frame is loaded
 // - When load_r is 1 and the baud clock is 1 it moves towards
 // - the right, sending the next bit
 // - '1's are introduced on the left
 always @ ( posedge clk)
   // - Reset
   if (rstn == 0 )
     shifter <= 10'b11_1111_1111 ;

   // - Load mode
   else if (load == 1 )
     shifter <= {CAR, 2'b01 };

   // - Scroll mode
   else if (load == 0 && clk_baud == 1 )
     shifter <= { 1'b1 , shifter [ 9 : 1 ]};

 always @ ( posedge clk)
   if (load == 1 )
     bitc <= 0 ;
   else if (load == 0 && clk_baud == 1 )
     bitc <= bitc + 1 ;

 // - Remove by tx the least significant bit of the displacement records
 // - When we are in load mode (load_r == 0), a 1 is always output for
 // - that the line is always in a resting state.  In this way in the
 // - start tx is at rest, although the value of the shift register
 // - be unknown
 // - IS A REGISTERED OUTPUT, since tx connects to a synchronous bus
 // - and you have to avoid spurious pulses (glitches)
 always @ ( posedge clk)
   tx <= shifter [ 0 ];

 // - Splitter to get the transmission clock
 baudgen # (BAUD)
   BAUD0 (
     .clk (clk),
     .clk_ena (baud_en),
     .clk_out (clk_baud)
   );

 // ---------------------------
 // - Timer
 // ---------------------------
 dividerp1 # (. M (DELAY))
   DIV0 (.clk (clk),
          .clk_out (start)
        );

 // ------------------------------
 // - CONTROLLER
 // ------------------------------

 // - States of finite controller automata
 localparam IDLE = 0 ;
 localparam START = 1 ;
 localparam TRANS = 2 ;

 // - States of the controller automaton
 reg [ 1 : 0 ] state;

 // - Transitions between states
 always @ ( posedge clk)

   // - Reset of the automata.  To the initial state
   if (rstn == 0 )
     state <= IDLE;

   else
     // - Transitions to the following states
     case (state)

       // - State of rest.  It goes out when the signal
       // - of start is set to 1
       IDLE:
         if (start_r == 1 )
           state <= START;
         else
           state <= IDLE;

       // - State of beginning.  Prepare to start
       // - to transmit.  Duration: 1 clock cycle
       START:
         state <= TRANS;

       // - Transmitting.  It is in this state until
       // - all pending bits have been transmitted
       TRANS:
         if (bitc == 11 )
           state <= IDLE;
         else
           state <= TRANS;

       // - By default.  NOT USED.  Position for
       // - cover all cases and that no latches are generated
       default :
         state <= IDLE;

     endcase

 // - Generation of micro-orders
 assign load = (state == START)?  1 : 0 ;
 assign baud_en = (state == IDLE)?  0 : 1 ;

 // - Initializer
 always @ ( posedge clk)
   rstn <= 1 ;

 endmodule

dividerp1.v: 1-cycle width pulse splitter 
`````````````````````````````````````````````

The divisor that we have used in other examples is modified so that the width is 1 clock cycle:

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T23-fsmtx/images/fsmtx2-2.png

Verilog描述::

 // - File: dividerp1.v
 `include " divider.vh "

 // - TICKETS:
 // - -clk: System clock signal (12 MHZ on the iceStick)
 //
 // - DEPARTURES:
 // - - clk_out.  Output signal to achieve the baud rate indicated
 // - Width of 1 period of clk.  UNREGISTERED DEPARTURE
 module dividerp1 ( input wire clk,
                  output wire clk_out);

 // - Default value of the baud rate
 parameter M = `T_100ms;

 // - Number of bits to store the baud divider
 localparam N = $ clog2 (M);

 // - Register to implement the module meter M
 reg [N - 1 : 0 ] divcounter = 0 ;

 // - Module counter M
 always @ ( posedge clk)
     divcounter <= (divcounter == M - 1 )?  0 : divcounter + 1 ;

 // - Take out a pulse width 1 clock cycle if the generator
 assign clk_out = (divcounter == 0 )?  1 : 0 ;

 endmodule

模拟
````

The test bench only generates the clock pulses to check the operation. The 100ms are not simulated, so that it goes faster. The start signal is changed to 8Khz

The verilog code is::

 // - File fsmtx2_tb.v
 `include " baudgen.vh "

 module fsmtx2_tb ();

 // - Baud with which to perform the simulation
 // - At 300 baud, the simulation takes longer to complete because the
 // - times are longer.  At 115200 baud the simulation is much
 // - faster
 localparam BAUD = `B115200;

 // - Time between characters
 localparam DELAY = `F_8KHz;

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

 // - Instance the component
 fsmtx2 # (. BAUD (BAUD), .DELAY (DELAY))
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
   $ dumpfile ( "fsmtx2_tb.vcd" );
   $ dumpvars ( 0 , fsmtx2_tb);

   # (FRAME_WAIT * 10 ) $ display ( "END of the simulation" );
   $ finish ;
 end

 endmodule

我们用以下命令模拟::

 $ make sim2

gtkwave的结果是：

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T23-fsmtx/images/fsmtx2-sim.png

The upper signal corresponds to tx and the one below to clk_baud , which marks the time of sending the bits. In the simulation you see how 2 complete characters are sent and the beginning of the third one.

The periodic signal start is the lower one. You can see the three pulses and how a character is sent for each one of them

综合和测试
```````````

We make the synthesis with the following command::

 $ make sint2

所用的资源有：

========   ======
  资源       占用
========   ======
  IOPs      6/96
  PLBs      21/160
  BRAMs     0/16
========   ======

我们执行以下命令加载到FPGA::

 $ iceprog fsmtx2.bin

When opening the gtkterm to the 115200 baud we will see how the A characters appear periodically：

.. image:: https://raw.githubusercontent.com/Obijuan/open-fpga-verilog-tutorial/master/tutorial/ICESTICK/T23-fsmtx/images/fsmtx2-gtkterm.png

提议的练习
------------

- Test the samples at different bauds and changing the delay of sending characters in the second 

- Modify the controller so that a ready output signal is generated, set to 1 when the transmitter is ready to send the next character and remains at 0 when busy
