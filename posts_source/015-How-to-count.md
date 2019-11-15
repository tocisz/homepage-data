date: 2019-11-14
abstract: How to count modulo? Should I use modulo operator or "if" statement? Does it matter at all?
image: 015-LFSR.jpg

# How to count?

After a while since [using Verilator](010-Learning-basics-of-Verilog) I wanted
to see what advice will give it's lint on [my latest project](014-Connecting-FPGA-to-a-computer).

It didn't find anything really bad. There was WIDTH mismatch somewhere,
but also warning related to the use of modulo `%` operator.

Internet says that modulo operator should be avoided in Verilog.
[Xilinx forum](https://forums.xilinx.com/t5/Synthesis/Modulus-synthesizable-or-non-synthesizable/m-p/747498#M20683)
says it won't synthesize unless it's \\(\\mod 2^N\\).

In my design of [binary numeric screen mode](013-Binary-numeric-screen-mode)
I use modulo 9 and it works! Is it very bad?

## Test problem

Should I change my design? Should I really avoid using modulo in synthesizable
Verilog?

To answer this I decided to compare resources usage of different solutions to
the following simple problem:

> Input: clock signal
>
> Output: Counter that goes from 0 to 99 cyclically.

Harder version:

> Input: clock signal
>
> Output 1: Counter that goes from 0 to 99 cyclically.
>
> Output 2: As above, but modulo 9.

## Solutions

### Simple problem solution using modulo

```Verilog
module c100_mod (
    input clk,
    output reg [6:0] c100,
    output [3:0] c9
);
assign c9 = 4'd0;

initial c100 = 7'd0;

always @(posedge clk)
	c100 <= (c100 + 1'd1) % 100;

endmodule
```

### Simple problem solution using if

```Verilog
module c100_if (
    input clk,
    output reg [6:0] c100,
    output [3:0] c9
);
assign c9 = 4'd0;

initial c100 = 7'd0;

always @(posedge clk)
	if (c100 == 7'd99)
		c100 <= 7'd0;
	else
		c100 <= c100 + 1'd1;

endmodule
```

Is there a difference in FPGA resources usage?

| Utilization                                         | c100_mod | c100_if  |
| --------------------------------------------------- | --------:| --------:|
| Number of Slice Registers                                         | 7 | 7 |
| * Number used as Flip Flops                                       | 7 | 7 |
| * Number used as Latches                                          | 0 | 0 |
| * Number used as Latch-thrus                                      | 0 | 0 |
| * Number used as AND/OR logics                                    | 0 | 0 |
| Number of Slice LUTs                                              | 9 | 9 |
| * Number used as logic                                            | 9 | 9 |
| * * Number using O6 output only                                   | 8 | 7 |
| * * Number using O5 output only                                   | 0 | 0 |
| * * Number using O5 and O6                                        | 1 | 2 |
| * * Number used as ROM                                            | 0 | 0 |
| * Number used as Memory                                           | 0 | 0 |
| Number of occupied Slices                                         | 3 | 5 |
| Number of MUXCYs used                                             | 0 | 0 |
| Number of LUT Flip Flop pairs used                                | 9 | 9 |
| * Number with an unused Flip Flop                                 | 3 | 2 |
| * Number with an unused LUT                                       | 0 | 0 |
| * Number of fully used LUT-FF pairs                               | 6 | 7 |
| * Number of unique control sets                                   | 1 | 2 |
| * Number of slice register sites lost to control set restrictions | 1 | 9 |

In both designs we have 7 slices used as flip-flops and 9 LUTs used as logic.
Surprisingly, FFs and LUTs can be packed better in the design using modulo operator,
giving difference of 3 slices used vs 5 slices used.

## Harder problem solutions with modulo

In these solutions instead of:

```Verilog
assign c9 = 4'd0;
```

We have:
```Verilog
assign c9 = c100 % 9;
```

Verilator lint doesn't like this. Is says that `c100 % 9` has different
width than `c9`. We can fix this warning by writing:

```Verilog
wire [6:0] c100mod9;
assign c100mod9 = c100 % 9;
assign c9 = c100mod9[3:0];
```

or

```Verilog
reg [6:0] c100mod9;
always @*
begin
	c100mod9 = c100 % 9;
	c9 = c100mod9[3:0];
end
```

All three versions are equivalent and use the same resources.

Solution combining this code with **c100_mod** will be referenced as
**c100_mod_mod9**, while solution combining this code with
**c100_if** will be referenced as **c100_if_mod9**. Let's compare them:

| Utilization                               | c100_mod_mod9 | c100_if_mod9  |
| --------------------------------------------------- | --------:| --------:|
| Number of Slice Registers                                         | 7  | 7  |
| * Number used as Flip Flops                                       | 7  | 7  |
| * Number used as Latches                                          | 0  | 0  |
| * Number used as Latch-thrus                                      | 0  | 0  |
| * Number used as AND/OR logics                                    | 0  | 0  |
| Number of Slice LUTs                                              | 15 | 16 |
| * Number used as logic                                            | 15 | 16 |
| * * Number using O6 output only                                   | 11 | 11 |
| * * Number using O5 output only                                   | 0  | 0  |
| * * Number using O5 and O6                                        | 4  | 5  |
| * * Number used as ROM                                            | 0  | 0  |
| * Number used as Memory                                           | 0  | 0  |
| Number of occupied Slices                                         | 5  | 5  |
| Number of MUXCYs used                                             | 0  | 0  |
| Number of LUT Flip Flop pairs used                                | 15 | 16 |
| * Number with an unused Flip Flop                                 | 9  | 11 |
| * Number with an unused LUT                                       | 0  | 0  |
| * Number of fully used LUT-FF pairs                               | 6  | 5  |
| * Number of unique control sets                                   | 1  | 1  |
| * Number of slice register sites lost to control set restrictions | 1  | 1  |

Both versions occupy 5 slices, but version using modulo requires one LUT less.

## Harder problem solution with if

Finally, I implemented counter modulo 9 with its own registers.
This version is called **c100_if_if**.

```Verilog
initial
begin
	c100 = 7'd0;
	c9   = 4'd0;
end

always @(posedge clk)
	if (c100 == 7'd99)
	begin
		c100 <= 7'd0;
		c9   <= 4'd0;
	end
	else
	begin
		c100 <= c100 + 1'd1;
		if (c9 == 4'd8)
			c9 <= 0'd0;
		else
			c9 <= c9 + 1'd1;
	end
```

I didn't expect too much of it, but here are results:

| Utilization                               | c100_mod_mod9 | c100_if_mod9  | c100_if_if |
| --------------------------------------------------- | --------:| --------:| ----------:|
| Number of Slice Registers                                         | 7  | 7  | 11 |
| * Number used as Flip Flops                                       | 7  | 7  | 11 |
| * Number used as Latches                                          | 0  | 0  | 0 |
| * Number used as Latch-thrus                                      | 0  | 0  | 0 |
| * Number used as AND/OR logics                                    | 0  | 0  | 0 |
| Number of Slice LUTs                                              | 15 | 16 | 14 |
| * Number used as logic                                            | 15 | 16 | 14 |
| * * Number using O6 output only                                   | 11 | 11 | 13 |
| * * Number using O5 output only                                   | 0  | 0  | 0 |
| * * Number using O5 and O6                                        | 4  | 5  | 1 |
| * * Number used as ROM                                            | 0  | 0  | 0 |
| * Number used as Memory                                           | 0  | 0  | 0 |
| Number of occupied Slices                                         | 5  | 5  | 4 |
| Number of MUXCYs used                                             | 0  | 0  | 0 |
| Number of LUT Flip Flop pairs used                                | 15 | 16 | 14 |
| * Number with an unused Flip Flop                                 | 9  | 11 | 4 |
| * Number with an unused LUT                                       | 0  | 0  | 0 |
| * Number of fully used LUT-FF pairs                               | 6  | 5  | 10 |
| * Number of unique control sets                                   | 1  | 1  | 1 |
| * Number of slice register sites lost to control set restrictions | 1  | 1  | 5 |

Surprisingly, this version uses less LUTs than **c100_mod_mod9**.
What's more interesting, it occupies less Slices than version **c100_if**!
(which is basically the same, but without `c9` related logic)

Better Slice usage can be partially explained by higher number of
*fully used LUT-FF pairs*. Previous designs are more combinatorial,
while this design uses additional registers. Slice contains both
LUTs (that are used for combinatorial logic) and registers (that are
used as flip-flops or latches). Design can be better packed into
slices when number of registers used is close to the number of LUTs used.

## Summary

Modulo (by constant) operator is not as bad as Internet says (at least
when using Xilinx FPGAs). There is no simple rule to say which design is
better. Balancing combinatorial logic and registers usage may be important.

## Other thoughts

Often when I think about some state machine I don't care how states are
represented. I usually use numbered states \\(0 ... N\\) in that case.
But is it optimal?

Incrementing state from \\(n\\) to \\(n+1\\) represented as binary numbers
means dealing with [carry propagation delay](https://en.wikipedia.org/wiki/Propagation_delay).

Are there more efficient ways to change states? I'm not the first one to ask
this question. There is some [research about it](https://www.researchgate.net/publication/335441783_Cyclic_Sequence_Generators_as_Program_Counters_for_High-Speed_FPGA-based_Processors).

More on this topic:

* What is [LFSR](https://en.wikipedia.org/wiki/Linear-feedback_shift_register)?
* [Number of states in LFSR](https://crypto.stackexchange.com/questions/5683/number-of-states-in-a-lfsr)
* How to [create LFSR with arbitrary number of states](https://crypto.stackexchange.com/questions/11395/feasibility-of-using-a-base-26-lfsr-for-cryptography-by-hand)?

But what if it's not just a cyclic counter, but a state machine that has inputs?
Than I think it really gets interesting.
