date: 2019-03-10
abstract: Simulating cellular automaton in hardware using Xilinx Spartan 6 FPGA.

# Programming Xilinx Spartan 6

In the [last article](010-Learning-basics-of-Verilog) I was learning basics of Verilog and
as a result I created simulation of cellular automaton.

I have a board with Xilinx Spartan 6 FPGA to play with, so I wanted to program it
with this design.

## Clock goes too fast
The board has 50MHz external clock source for FPGA. But that doesn't mean FPGA
internal clock needs to be the same. Xilinx FPGAs have [dedicated modules for
managing clock resources](https://www.xilinx.com/support/documentation/user_guides/ug382.pdf).
By using [PLL](https://en.wikipedia.org/wiki/Phase-locked_loop) techniques it is possible
to generate higher or lower frequency and to introduce many phase shifted clock signals.

In this project I only wanted my clock to be really slow, so I could see cellular automaton working. Instead of using Xilinx built-in clock synthesis
I wrote really simple module to divide clock frequency. All it contains is
a counter. Adding one on each input clock cycle and giving last bit of a counter
as and output.

```Verilog
module clk_div
#(parameter BITS = 25)
(
    input  wire clk_in,
    output wire clk_out
);

reg [BITS:1] cnt;

initial
begin
	cnt = 0;
end

assign clk_out = cnt[BITS];

always @(posedge clk_in)
begin
	cnt[BITS:1] <= cnt[BITS:1] + 1'b1;
end

endmodule
```

## Reset signal

I also wanted to be able to reset automaton by push button.
This turned out to be harder than I thought. My first attempt wasn't working.
Generally, you can expect problems when reset signal is not released
synchronously with clock signal. To prevent this, combination of two
flip-flops called *reset bridge* is commonly used. More on resetting FPGA
in [this article](https://www.eetimes.com/document.asp?doc_id=1278998).

Reset bridge in Verilog:
```Verilog
reg rst_meta, rst_sync;
initial
begin
	rst_meta = 1;
	rst_sync = 1;
end

always @(posedge clk_slow or negedge reset)
begin
  if (!reset)
  begin
    rst_meta <= 1'b1;
    rst_sync <= 1'b1;
  end
  else
  begin
    rst_sync <= rst_meta;
    rst_meta <= 1'b0;
  end
end
```

## Constraints
Finally, when programming FPGA you need to tell which input/output ports
should be assigned with which physical pads. In Xilinx toolset it is done
with `.ucf` file, which can be very simple:

```
NET "clk" LOC = A10;
NET "reset" LOC = T8;

NET "u7_l[0]" LOC = E12;
...
NET "u7_l[26]" LOC = T12;

NET "led1" LOC = T9; // D1
NET "led2" LOC = R9; // D3
```

Assigning ports to pads is just one function of this file. It is also possible
to constraint location of [LUTs](https://www.allaboutcircuits.com/technical-articles/getting-started-with-fpgas-look-up-tables-and-flip-flops/), etc.
Another possibility is to constraint timing of signal paths.

## Conclusions

It was easy to get this simple project running. Next, I will try
something more complicated.

And here is a video showing cell automaton evolving with 9 LEDs:

<iframe width="560" height="315" src="https://www.youtube.com/embed/nB4hAfsUMjI" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
