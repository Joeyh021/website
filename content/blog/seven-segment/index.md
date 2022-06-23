---
title: "Rust, FPGAs, and Seven Segment Displays"
date: 2022-06-14T23:06:57+01:00
description: "My experience building and programming a simple I/O device for a RISC-V core on an FPGA."
summary: "Getting my feet wet with FPGA design"
draft: true
---

The title of my third year project is "Programmable I/O for Flexible Interfaces in Embedded RISC-V Systems", which basically just means making something very similar to the [Programmable I/O blocks on the Raspberry Pi Pico](https://hackspace.raspberrypi.com/articles/what-is-programmable-i-o-on-raspberry-pi-pico) (which I think are insanely cool), and then integrating them with a RISC-V core to create a simple microcontroller that can interface with basically anything at fairly low power. This means a lot of work with FPGAs and Verilog/HDL which are two things I don't have much experience with.

I did do an introductory digital systems design course as part of my degree last year, and learned a bit about Verilog and FPGA design & architecture. The coursework was making a video game (a very common application for FPGAs), and our shitty asteroids clone was mostly thrown together in one very long week of late nights spent fighting with Xilinx Vivado, the FPGA IDE and toolchain, and also perhaps the worst piece of software ever (if you know, you know, and by god do you know). However, going from making an arcade game to a microcontroller is a big leap that's going to require a lot of learning.

So instead of revising for my exams, I spent a few days trying to get to grips better with Xilinx's design tools and FPGA work in general by designing a hardware interface for a seven segment display, integrating it with an existing RISC-V core, and then writing a software driver for it in Rust.

![](fpga sevenseg gif)

## Getting Going

My staring point was [this GitHub repo](https://github.com/eugene-tarassov/vivado-risc-v), which was basically perfect because it provides a RISC-V SoC with UART, Ethernet, an SD card controller, and scripts to build bootloaders and the Linux kernel. It was literally as easy running `make` as per the instructions, flashing the kernel onto an SD card, and then connecting over the serial port. I had Debian up and running on the Nexys A7 board the School of Engineering had kindly let me borrow with surprisingly little difficulty. The Vivado block design of the SoC is shown below, it's just a core hooked up to the onboard DDR2 memory and a few I/O components using AXI busses.

![](block design)

The CPU core used is [Rocket Chip](https://github.com/chipsalliance/rocket-chip), an open-source RISC-V SoC generator built and maintained by UC Berkeley Architecture Research. Rocket Chip can generate a RTL RISC-V implementation that has virtual memory, a coherent multi-level cache hierarchy, IEEE-compliant floating-point units, and all the other bits and bobs you need for a useful CPU. All the HDL is entirely parametrised, so can be customised to generate whatever kind of core takes your fancy. It's written in Chisel, a HDL embedded in Scala, which is what enables this. Scala for HDL sounds cool (read: better than Verilog, which is not much competition), and I really want to try it out at some point.

Linux running on an FPGA was cool, but I didn't do much with it besides brag about it on IRC

![]()

Instead, I went back and compiled [the bare metal example](https://github.com/eugene-tarassov/vivado-risc-v/tree/master/bare-metal) because Linux was super slow, and writing bare metal drivers is much simpler than Linux drivers (part 2, perhaps?). The example is just sending hello world over the serial port by writing to the UART registers, but the key bit is the linker script and startup code, which will come in handy later. The more observant among you will notice one problem with this code though...

## RIIR

It's written in C, so obviously it needed the Rewrite It In Rust treatment. My idea was to create a minimal `no-std` executable using the [`riscv-rt`](), which is meant to provide a startup and runtime for embedded RISC-V targets. Of course, it couldn't be that simple, and I ended up spending the best part of a day grappling with rustc/lld to generate me an executable that would run, all to no avail. I probably tried every command line flag and cargo configuration parameter in existence, but could not for the life of me get anything to work using the `riscv-rt` crate. My hacky solution in the end was to just compile the rust crate to a static library exposing only `main`, and then invoke the gcc cross-compiler to link using the linker script and startup assembly code from the example. This worked without any trouble, and I had Rust code running on an FPGA!

![](hello from rust.jpg)

The serial interface code is shown below. The hardware has 4 registers: Rx, Tx, and control and status registers. I can't find much documentation on how to use the control/status registers besides the example code, but the idea is simple enough: put some bytes in the Tx register, and it gets sent to the console on the other end.

```rust
register code

```

I'm using [`volatile_register`](), which provides read-only, write-only and read-write cells with volalile access for modelling CPU registers. Rust really shines here, as I can encode what functions are read-only and what requires a write in the type system, and through the use of immutable vs mutable references.

## The Display Controller

So I had the bare minimum working, it was time to extend it a little with something actually exciting. The 8-digit 7 segement display on my board has a hardware interface that looks like the following:

![image of hardware]() the not at all confusing display

Each digit has it's own common anode, and all the same segments on each digit share a pin. This means it's fairly easy to write a controller which controls all the digits in unison: tie all the anodes high, and then work out what patterns of segment signals correspond to each digit. In Verilog, that's a simple behavioural case statement:

```Verilog
digit patterns
```

Controlling each digit individually is a bit harder, as the segments are all connected together. The way to do it is to strobe the anodes, having only one of them active at a time and displaying the segment pattern that we want, but strobe it fast enough that our eyes can't tell that only one digit is on at a time. The onboard clock is 100MHz, so we use an N-BIT counter to slow that down to a much more tame HF(UHF) HERTZ and select which digit is active.

```Verilog
strobing
```

Rather counterintuitively, the anodes and digits are both active low. I didn't figure this out until I'd spent an entire weekend debugging everything instead of just reading the docs. upside down smiley.

Writing a simple verilog module to act as a controller was the easy part, integrating it with the existing SoC was the challenge. All the other I/O blocks are connected over an AXI4 bus, which is fairly simple to extend. AXI4 (what does this stand for) is a bus specification developed by AMD as part of the (whatever AMBA stand for) spec. It includes three different kinds of bus: [KINDS OF BUS AND ALSO THE 5 CHANNELS].


I went with AXI Lite, as all I was connecting was 8 registers. Vivado includes a "wizard' to generate a template for an AXI IP block, so it created a new project for me with a top level module defining an AXI interface, and very kindly pointed out where to insert my own port and module instantiations. I instantiated my module and connected the Bus registers to the display controller, and then added two new output ports to the block, `an[7:0]` for the 8 anodes and `seg[6:0]` for the 7 segments.

![IP block image]() my very own vivado IP core!

## Putting it all together

Adding my new IP to the design was relatively painless, just a few steps in the block diagram GUI to drop it in and connect the wires up. I had to also extend the I/O address space to memory-map my registers, and add a new constraints file to connect the output from the design to the peripherals. Constraints are <WHAT ARE CONSTRAINTS>, and in this case they're used to connect the right wires to the right FPGA output pins.

To verify this worked I used Xilinx System Debugger (`xsdb`) to connect to the FPGA, which provides a neat little command line interface where you can poke and prod at running targets on an FPGA or Xilinx SoC. I manually wrote into the address I'd mapped the registers to to verify that everything worked, which it did (after I remembered that the anodes were active low and fixed that).

more debug methods such as jtag, bscan , ILA, want to play with it more because epic

## More Rust

Once I knew the hardware worked I had to write a driver to get it going in software. I basically replicated the UART setup, but with more registers, and a few functions to make the interface a little more convenient.

## Conclusion

look at scala
much more complex i/o to be done
want to look at axi better
this involved a lot of googling bad xilinx docs and playing around, it did not all come together anywhere near as cleanly as it may seem.

thank you euegene you king
