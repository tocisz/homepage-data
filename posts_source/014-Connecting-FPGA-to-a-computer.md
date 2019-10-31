date: 2019-10-31
abstract: Connecting binary numeric LCD controller to a computer with UART.
image: 001-1.jpg

# Connecting FPGA to a computer with UART

In [the last blog entry](013-Binary-numeric-screen-mode) I mapped memory block
to VGA screen. But it was boring to see
how memory is being filled with ones by very simple process.
So last couple of days I was working on connecting my FPGA to computer with
UART.

I use cheap Chinese USB-to-UART converter. The same that was used
[in my first blog port](001-Play-with-forth-and-STM32).

This time I decided I don't want to implement UART communication by myself.
I found a [project on opencores.org](https://opencores.org/projects/uart2bus) that was ready
to use. All I had to do was to connect UART controller to a memory block.

## Direct connection

In the future I plan to use UART connection not only to read and write memory,
but also to control other modules (processor, memory, etc.)
on FPGA chip. This can be done by assigning address space to these
modules and handling read/write operations in a module specific manner.

As for now I am only connecting 1kB block of RAM, but I will interface trough
a routing module anyway. Let's see input and output signals of this module:

```Verilog
module uart_sram_bridge
(
  input  wire clk50_dup,

  input  wire [15:0] uart_address,  // address bus to register file
  output wire  [7:0] uart_read_data,  // write data to register file
  input  wire  [7:0] uart_write_data,  // write data to register file
  input  wire        uart_write,    // write control to register file
  input  wire        uart_read,    // read control to register file
  input  wire        uart_req,    // bus access request signal
  output wire        uart_gnt,    // bus access grant signal

  output reg  [9:0] sram_address,
  input  wire [7:0] sram_read_data,
  output reg  [7:0] sram_write_data,
  output wire       sram_write_enable
);
```

We have a clock signal. Not used now, but may be useful later.

We have a section of signals that are connected to UART interface. UART
module handles incoming read/write commands and asserts `uart_read`,
`uart_write` signals, passing address and data on `uart_read_data`
and `uart_write_data`. UART module will set `uart_read` or `uart_write`
only when `uart_gnt` is high.

Than we a have section of signals that go to/from RAM.

My first attempt of connecting UART to RAM is very simple:

```Verilog
assign sram_address      = uart_address[9:0];
assign sram_write_data   = uart_write_data;
assign uart_read_data    = sram_read_data;
assign sram_write_enable = uart_write;
assign uart_gnt = 1;
```

1. Always grant access to the bus.
2. Connect address and data lines directly.
3. Assert sram_write_enable only when UART says it wants to write (otherwise it
   RAM is in read mode).
4. Ensure that RAM gets clock signal that is phase shifted.

## Adding delay (and latches)

After connecting signals as it is described in the previous section,
I found that it's not working.

The usual problem is that RAM memory expects address and data signals
to be stable some time before positive clock signal edge.
This can be done by using memory clock signal that is delayed by 180&deg;.
But it didn't help. I could plainly see (that part was working)
that I can't write to the memory. I don't know why.

I suspected that some signals are delayed or there is not enough time
for the memory, etc.
Since UART is much slower than internal clock, I decided to hold
signals for configured number of clock cycles.

This can be done with a simple latch. When read or write request comes
from UART I latch data and address signals for a couple of cycles. I also
set `uart_gnt` signal low to block requests when memory is busy.

Delay is a parameter of a module:

```Verilog
module uart_sram_bridge
#(
  parameter LATCH_DELAY = 1,
  parameter LATCH_DELAY_W = 1
)
...
```

We start counting from 0 up to LATCH_DELAY when write request is granted.

```Verilog
wire write_req_granted;
assign write_req_granted = uart_write & uart_gnt;

// count from 0 to LATCH_DELAY starting on write_req_granted signal
reg [LATCH_DELAY_W-1:0] wr_latch_delay;
initial wr_latch_delay = LATCH_DELAY;
always @(posedge clk50_dup)
begin
  if (write_req_granted)
    wr_latch_delay <= 0;
  else if (wr_latch_delay != LATCH_DELAY)
    wr_latch_delay <= wr_latch_delay + 1'b1;
end
```

We assert `sram_write_enable` for LATCH_DELAY cycles and set `uart_gnt` low
when write is in progress.

```Verilog
assign sram_write_enable = wr_latch_delay < LATCH_DELAY; // set sram_write_enable for LATCH_DELAY clock cycles
assign uart_gnt = !sram_write_enable; // block bus while we write
```

Order of assign commands doesn't matter in Verilog. Code above could be rewritten
as:

```Verilog
assign write_req_granted = uart_write & !(wr_latch_delay < LATCH_DELAY);
```

This ensures that after `wr_latch_delay <= 0` is executed at positive edge
of clock signal `write_req_granted` will be low, so counting will continue
until `LATCH_DELAY` is reached.

Finally, latches:

```Verilog
always @*
begin
  if (uart_req) // read or write - anyway - address is necessary
  begin
    sram_address <= uart_address[9:0];
  end
  if (write_req_granted)
  begin // latch data and address on write_req_granted signal
	 sram_write_data <= uart_write_data;
  end
end
```

As you can see:

1. No clock signal is involved.
2. There are no `else` parts of `if` statements that do assignment. So once
   condition is false, value will stay as it was last set (it is latched).

This version works perfectly fine. Xilinx FPGA synthesis software doesn't like it.
It warns about it:

> Xst:737 - Found 1-bit latch for signal &lt;sram_address&lt;9&gt;&gt;. Latches may be generated from incomplete case or if statements. We do not recommend the use of latches in FPGA/CPLD designs, as they may lead to timing problems.

I don't know how much of a truth is in this warning. I've checked that
version with flip-flop also works.

Flip-flop is basically a pair
of latches, so it seems that using flip-flops is more expensive.
But that's not the case. Spartan FPGA is optimized for usage of flip-flops,
so when I use a latch, one half of a flip-flop is wasted and there
is no gain is resources usage.

What to do to convert latches to flip-flops? It's enough to say that transitions
should happen only on a clock edge:

```Verilog
always @(posedge clk50_dup) // also works with @* (latches instead of flip-flops)
...
```

## Summary

The plan was to connect UART in an hour. It took much more time. But I've learnt
something about latches vs flip-flops. And I hope this article is more interesting
because of that too.

As usually, you can find latest version of my project
[on GitHub](https://github.com/tocisz/verilog-vesa-ca/tree/vesa-vled).
