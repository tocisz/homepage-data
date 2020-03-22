date: 2020-03-22
abstract: In this article I will compare two implementations of a stack on Xilinx Spartan 6 FPGA.
image: 018.png

# How to stack? (on an FPGA)

## Using LUTs as memory (distributed RAM)

In [j1 processor](https://github.com/jamesbowman/j1)
stack is implemented as a RAM page:
```Verilog
module stack
  #(
    parameter DEPTH=4,
    parameter WIDTH=16
  )
  (input wire clk,
  input wire [DEPTH-1:0] ra,
  output wire [WIDTH-1:0] rd,
  input wire we,
  input wire [DEPTH-1:0] wa,
  input wire [WIDTH-1:0] wd);

  reg [WIDTH-1:0] store[0:(2**DEPTH)-1];

  always @(posedge clk)
    if (we) begin
      store[wa] <= wd;
    end

  assign rd = store[ra];
endmodule
```

Xilinx ISE when seeing an array can infer to use RAM block:
```
Synthesizing Unit <stack>.
    Related source file is "/home/ise/VBoxShare/bf1/stack.v".
        DEPTH = 4
        WIDTH = 13
    Found 16x13-bit dual-port RAM <Mram_store> for signal <store>.
    Summary:
	inferred   1 RAM(s).
Unit <stack> synthesized.
```

In "advanced synthesis" step it decides to use LUTs to implement it:
```
Synthesizing (advanced) Unit <stack>.
INFO:Xst:3218 - HDL ADVISOR - The RAM <Mram_store> will be implemented on LUTs either because you have described an asynchronous read or because of currently unsupported block RAM features. If you have described an asynchronous read, making it synchronous would allow you to take advantage of available block RAM resources, for optimized device usage and improved timings. Please refer to your documentation for coding guidelines.
    -----------------------------------------------------------------------
    | ram_type           | Distributed                         |          |
    -----------------------------------------------------------------------
    | Port A                                                              |
    |     aspect ratio   | 16-word x 13-bit                    |          |
    |     clkA           | connected to signal <clk>           | rise     |
    |     weA            | connected to signal <we>            | high     |
    |     addrA          | connected to signal <wa>            |          |
    |     diA            | connected to signal <wd>            |          |
    -----------------------------------------------------------------------
    | Port B                                                              |
    |     aspect ratio   | 16-word x 13-bit                    |          |
    |     addrB          | connected to signal <ra>            |          |
    |     doB            | connected to internal node          |          |
    -----------------------------------------------------------------------
Unit <stack> synthesized (advanced).
```

How much LUTs it uses? In "Map" report, we can see:
```
Number of Slice LUTs:                        120 out of   9,112    1%
  Number used as logic:                      109 out of   9,112    1%
    Number using O6 output only:              93
    Number using O5 output only:              11
    Number using O5 and O6:                    5
    Number used as ROM:                        0
  Number used as Memory:                      10 out of   2,176    1%
    Number used as Dual Port RAM:             10
      Number using O6 output only:             2
      Number using O5 output only:             0
      Number using O5 and O6:                  8
```

Looking at [Spartan-6 FPGA Configurable Logic Block](ug384-Spartan-6 FPGA Configurable Logic Block.pdf) documentation:

* 4 LUTs can be used to create "Simple Dual-Port 32 x 6-bit RAM"
* 2 LUTs can be used to create "Dual-Port 64 x 1-bit RAM"

We can see that using twice "Simple Dual-Port 32 x 6-bit RAM" and once
"Dual-Port 64 x 1-bit RAM" we use 10 LUTs and get desired stack capacity.

In fact we can make stack twice as deep (32 elements) with no additional
resources. 10 LUTs is really not much, and I think it's better to use them
than to use [dedicated memory block](ug383-Spartan-6 FPGA Block RAM Resources.pdf) for the stack.

## Using shift register

In [j1a](https://github.com/jamesbowman/swapforth/tree/master/j1a) processor
stack is implemented as a bidirectional shift register.

```Verilog
module stack2
#(
  parameter DEPTH=16,
  parameter WIDTH=16
)
(
  input wire clk,
  input wire we,
  input wire [1:0] delta,
  output wire [WIDTH-1:0] rd,
  input  wire [WIDTH-1:0] wd
);
  localparam BITS = (WIDTH * DEPTH) - 1;

  wire move = delta[0];

  reg [WIDTH-1:0] head;
  reg [BITS:0] tail;
  wire [WIDTH-1:0] headN;
  wire [BITS:0] tailN;

  assign headN = we ? wd : tail[WIDTH-1:0];
  assign tailN = delta[1] ? {{WIDTH{1'b0}}, tail[BITS:WIDTH]} : {tail[BITS-WIDTH:0], head};

  always @(posedge clk) begin
    if (we | move)
      head <= headN;
    if (move)
      tail <= tailN;
  end

  assign rd = head;
endmodule
```

Spartan-6 LUTs can be used for efficient implementation of shift registers,
but not bidirectional ones. Flip-flops need to be used to implement it.
In one Spartan-6 slice there are 4 LUTs and 8 flip-flops. Each flip-flop
can store one bit, so we need at least 26 slices for the stack (versus 5
slices in previous implementation).

Version of brainfuck CPU that uses this stack [can be found here](https://github.com/tocisz/brainfuck_machine/tree/stack2).

Let's look at Xilinx ISE synthesis output:
```
Synthesizing Unit <stack2>.
    Related source file is "/home/ise/VBoxShare/bf1/stack2.v".
        DEPTH = 16
        WIDTH = 13
    Found 208-bit register for signal <tail>.
    Found 13-bit register for signal <head>.
    Summary:
	inferred 221 D-type flip-flop(s).
	inferred   1 Multiplexer(s).
Unit <stack2> synthesized.
```

In "Map" output we can see that there are indeed many flip-flops used
and no LUTs used as memory:
```
Slice Logic Utilization:
  Number of Slice Registers:                   259 out of  18,224    1%
    Number used as Flip Flops:                 259
    Number used as Latches:                      0
    Number used as Latch-thrus:                  0
    Number used as AND/OR logics:                0
  Number of Slice LUTs:                        341 out of   9,112    3%
    Number used as logic:                      340 out of   9,112    3%
      Number using O6 output only:             325
      Number using O5 output only:              11
      Number using O5 and O6:                    4
      Number used as ROM:                        0
    Number used as Memory:                       0 out of   2,176    0%
```

Finally, number of slices is 94 (40 in distributed RAM implementation):
```
Slice Logic Distribution:
  Number of occupied Slices:                    94 out of   2,278    4%
```
So number of slices used increased by 54 &mdash; twice as much as I have expected.

Is there any upside? Maximal clock speed is estimated to be 212 MHz,
opposed to 176 MHz in previous implementation. This implementation
is slightly faster.

Why it was used by j1a processor? This processor is targeted for
[Lattice iCEstick](http://www.latticesemi.com/icestick), not for
Xilinx Spartan. I suspect that it is better suited to the architecture
of this FPGA.
