date: 2020-03-15
abstract: Brainfuck machine is a FPGA softcore CPU created to run brainfuck programs effectively.
image: 017.png

# Brainfuck machine

Long time ago I started this blog by describing my experiments with
[programming microcontroller in Forth](001-Play-with-forth-and-STM32).
I like this language, because it's very simple, yet it gives much of flexibility.

And sometimes it's really practical. James Bowman decided to create special
softcore CPU with instruction set that matches closely basic Forth words.
He named it J1.
He used this CPU for high speed transmission of digital camera data
trough gigabit Ethernet.
I was impressed to see [how simple J1 CPU is](https://github.com/jamesbowman/j1/blob/master/verilog/j1.v).
You can read more about it [here](https://excamera.com/sphinx/fpga-j1.html).

Everybody who had something to do with computer science knows what
[Turing Machine](https://en.wikipedia.org/wiki/Turing_machine) is.
But not everybody knows [brainfuck language](https://en.wikipedia.org/wiki/Brainfuck)
or [P''](https://en.wikipedia.org/wiki/P%E2%80%B2%E2%80%B2),
which are inspired by Turing Machine.
Nevertheless, brainfuck is perhaps the most famous
[esoteric programming language](https://en.wikipedia.org/wiki/Esoteric_programming_language)
and there are [lots of programs for it](http://rosettacode.org/wiki/Category:Brainf***).

Recently, I came across Forth implementation of brainfuck compiler.
I was struck by [simplicity of it](http://rosettacode.org/wiki/Execute_Brain****/Forth):
```forth
1024 constant size
: init  ( -- p *p ) here size erase  here 0 ;
: right ( p *p -- p+1 *p ) over c!  1+  dup c@ ;
: left  ( p *p -- p-1 *p ) over c!  1-  dup c@ ;		\ range check?

: compile-bf-char ( c -- )
  case
  [char] [ of postpone begin
              postpone dup
              postpone while  endof
  [char] ] of postpone repeat endof
  [char] + of postpone 1+     endof
  [char] - of postpone 1-     endof
  [char] > of postpone right  endof
  [char] < of postpone left   endof
  [char] , of postpone drop
              postpone key    endof
  [char] . of postpone dup
              postpone emit   endof
    \ ignore all other characters
  endcase ;

: compile-bf-string ( addr len -- )
  postpone init
  bounds do i c@ compile-bf-char loop
  postpone swap
  postpone c!
  postpone ;
;
```

It almost looks like brainfuck is a subset of Forth language!

That's when I decided I will write my first CPU softcore for FPGA
and it will be dedicated for executing brainfuck programs.
I will start from J1, remove what's unnecessary and implement
only instructions needed for brainfuck.

## Instruction set

When we look at sample brainfuck program...
```
++++++++[>++++[>++>+++>+++>+<<<<-]>+>+>->>+[<]<-]>>.>---.+++++++..+++.>>.<-.<.+++.------.--------.>>+.>++.
```
...not knowing the language (please forget it for a moment if you do know it),
we can see that:

1. it looks like fish bones in ascii art, or maybe skeleton of a snake;
2. there are repetitions of the same symbol, like `++++++` or `<<<<`;
3. square braces are balanced and nested correctly.

Impression number 1 is not important, but other two are easy to understand
once we know the language.

* there are 8 instructions, each denoted by a single symbol, one of `-+<>[].,`
* `[` opens the loop and `]` closes the loop
* `<` decrements by 1 and `>` increments by one data pointer
* `-` decrements by 1 and `+` increments by one data under the pointer
* `,` and `.` are used for input and output operations

Not surprisingly:

2. Incrementation/decrementation by a constant other than 1 is useful.
3. Loops need to be nested properly.

Because of that, I decided to support incrementation/decrementation by
a constant in my CPU.

Instructions are generally 1 byte long (there is long jump, which is two
  bytes long). We have program counter (PC) and data address (A).
We also have stack (S) to store program address of beginning of a loop.

| Instruction | What it does?                                 | Used by |
| ----------- | -------------------------                     | ------- |
| `00XXXXXX`  | A += X (signed); PC++                         | `<>`    |
| `01XXXXXX`  | (*A) += X (signed); PC++                      | `-+`    |
| `10000000`  | if (*A) { PC = top(S) } else { pop(S); PC++ } | `]`     |
| `100XXXXX` (X &ne; 0)| if (*A) { push(S,PC+1); PC++ } else { PC += X } | `[` |
| `101XXXXX`&nbsp;`XXXXXXXX`| if (*A) { push(S,PC+2); PC+=2 } else { PC += X } | `[` |
| `11000000` | store input to (*A)     | `,` |
| `11100000` | send (*A) to the output | `.` |

## Compilation

Translating brainfuck to bytecode is straightforward.
There is only one difficulty: sometimes we need to
use long jump and that makes jump addresses calculation
more complicated.

Anyone interested can find compiler written in Python
[here](https://github.com/tocisz/brainfuck_machine/blob/master/compile_simulate/comp-bf.py).

## The CPU

This is [Harvard architecture](https://en.wikipedia.org/wiki/Harvard_architecture) CPU. It has separate instruction memory and data memory. It also has I/O lines and uses
(very simple) ALU.

I implemented it in Verilog and used [Verilator](https://www.veripool.org/projects/verilator/wiki/Intro)
to test it on compiled brainfuck programs.

Let's go trough [Verilog code](https://github.com/tocisz/brainfuck_machine/blob/master/verilog/bf1.v).
No worries, it's really simple!

### Ports
As anyone could expect, we have clock and reset signals.
```Verilog
input wire clk,
input wire resetq,
```

We need to interface data memory.
We have memory address, data input, data output
and memory write signal for that.
```Verilog
output wire [`DADDR_WIDTH-1:0] mem_addr,
output reg  mem_wr,
output reg  [`DATA_WIDTH-1:0] mem_dout,
input  wire [`DATA_WIDTH-1:0] mem_din,
```

For I/O instructions (`.,`) we have:
```Verilog
output reg  io_wr,
input  wire [`DATA_WIDTH-1:0] io_din,
output wire [`DATA_WIDTH-1:0] io_dout,
```

There is interface to program memory too:
```Verilog
output wire [`CADDR_WIDTH-1:0] code_addr,
input  wire [7:0] insn,
```

### Clock synchronization
Most of the CPU definition is combinatorial logic.
There is only one block which synchronizes with positive
edge of clock signal.
```Verilog
always @(negedge resetq or posedge clk)
begin
  if (!resetq) begin
    { pc, rsp, maddr, lj, lj_offset } <= 0;
  end else begin
    { pc, rsp, maddr, lj, lj_offset }
    <= { pcN, rspN, maddrN, ljN, lj_offsetN };
  end
end
```
As you can see, we have a bunch of registers
to store CPU state (`pc, rsp, maddr, lj, lj_offset`)
and for each of them we have variable that can
be used to set value in the next clock cycle
(`pcN, rspN, maddrN, ljN, lj_offsetN`).

### Stack for code addresses
I use stack module taken from J1 CPU:
```Verilog
reg [`DEPTH-1:0] rsp, rspN;
reg rstkW = 0;                 // R stack write
wire [`CADDR_WIDTH-1:0] rstkD;   // R stack write value
wire [`CADDR_WIDTH-1:0] rst0;
stack #(.DEPTH(`DEPTH),.WIDTH(`CADDR_WIDTH)) rstack (
  .clk(clk),
  .ra(rsp),
  .rd(rst0),
  .we(rstkW),
  .wa(rspN),
  .wd(rstkD)
);
```

### ALU

ALU is very simple. The only operation is signed (complement of two) addition.
```Verilog
reg signed [`DADDR_WIDTH-1:0] alu_a;
reg signed [`CADDR_WIDTH-1:0] alu_b;
reg [`DADDR_WIDTH-1:0] alu_c;

always @(alu_a, alu_b)
begin
   alu_c = alu_a + ($signed({alu_b,2'b0}) >>> 2);
end
```

### Prepare data for ALU

ALU is used by `<>-=[` instructions. Depending on instruction
different values of `alu_a` and `alu_b` are set.

```Verilog
always @(maddr, insn, mem_din, lj, lj_offset, pc)
begin
  alu_a  = 15'bX; // let synthesis decide what takes least resources
  alu_b  = 13'bX;
  casez ({lj,insn[7:6]})
    3'b0_00: begin alu_a = maddr;           alu_b = $signed({insn[5:0],7'b0}) >>> 7; end // < >
    3'b0_01: begin alu_a = {7'b0,mem_din};  alu_b = $signed({insn[5:0],7'b0}) >>> 7; end // - +
    3'b0_10: begin alu_a = {2'b0,pc};       alu_b = $signed({insn[5:0],7'b0}) >>> 7; end // [
    3'b1_??: begin alu_a = {2'b0,pc};       alu_b = {lj_offset,insn}; end // long jump
    3'b0_11: ; // ALU not used
  endcase
end
```

### Instruction decode

This is the most important part. Here we
have ALU output calculated and we can use it
to change CPU state.

For clarity, calculation of program
counter (`pcN`) is in a separate block.
In this block we only decide whether we enter the loop
(`[`) or leave it (`]`).

```Verilog
assign io_dout = mem_din; // nothing else can go as IO output
always @(pc, maddr, insn, alu_c, io_din, lj)
begin
  // defaults
  mem_wr = 0;
  io_wr  = 0;
  ljN = 0;
  maddrN = maddr;
  mem_dout = io_din;
  do_jump_or_ret = 0;
  do_jump = 0;

  casez ({lj,insn[7:5]})
    4'b0_00?: begin   maddrN = alu_c; end // < or >
    4'b0_01?: begin mem_dout = alu_c[7:0]; mem_wr = 1; end // - or +
    4'b0_100: begin do_jump_or_ret = 1; do_jump = |insn[4:0]; end // [ or ]
    4'b1_???: begin do_jump_or_ret = 1; do_jump = 1; end // do long jump
    4'b0_101: begin     ljN = 1; end // begin long jump
    4'b0_110: begin  mem_wr = 1; end // ,
    4'b0_111: begin   io_wr = 1; end // .
  endcase
end
```

### Calculating PC

PC is usually set to PC+1. Except when `do_jump_or_ret`
is high.
```Verilog
assign rstkD = pcN; // if we put anything on stack, it's pcN
assign lj_offsetN = insn[4:0]; // remember offset from previous instruction
always @ (do_jump_or_ret, do_jump, pc, mem_din, rsp, rst0, alu_c)
begin
  // default: go to the next instruction
  pcN   = pc + 1'b1;
  rspN  = rsp;
  rstkW = 0;

  if (do_jump_or_ret)
  begin
    if (do_jump)
    begin // [
      if (mem_din != 0) begin
        rspN = rsp + 1'b1; // into the loop
        rstkW = 1;
      end else begin
        pcN = alu_c[12:0]; // skip the loop
      end
    end
    else
    begin // ]
      if (mem_din != 0) pcN = rst0; // loop again
      else rspN = rsp - 1'b1; // leave the loop
    end
  end
end
```

## Conclusions

Of course I'm not the first one. There are other brainfuck
CPUs. There is even [a paper about implementing 256-core
brainfuck CPU on FPGA](http://people.csail.mit.edu/wjun/papers/sigtbd16.pdf) from MIT.
But it's my first CPU and I think it's quite elegant.

I plan to run it on my FPGA and communicate with it trough UART.
