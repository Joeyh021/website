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

## The Setup

My staring point was [this GitHub repo](https://github.com/eugene-tarassov/vivado-risc-v), which was basically perfect because it provides a RISC-V SoC with UART, Ethernet, an SD card controller, and scripts to build bootloaders and the Linux kernel. It was literally as easy running `make` as per the instructions, flashing the kernel onto an SD card, and then connecting over the serial port. I had Debian up and running on the Nexys A7 board the School of Engineering had kindly let me borrow with surprisingly little difficulty. The Vivado block design of the SoC is shown below, it's just a core hooked up to the onboard DDR2 memory and a few I/O components using AXI busses.

![](block design)

The CPU core used is [Rocket Chip](https://github.com/chipsalliance/rocket-chip), an open-source RISC-V SoC generator built and maintained by UC Berkeley Architecture Research. Rocket Chip can generate a RTL RISC-V implementation that has virtual memory, a coherent multi-level cache hierarchy, IEEE-compliant floating-point units, and all the other bits and bobs you need for a useful CPU. All the HDL is entirely parametrised, so can be customised to generate whatever kind of core takes your fancy. It's written in Chisel, a HDL embedded in Scala, which is what enables this. Scala for HDL sounds cool (read: better than Verilog, which is not much competition), and I really want to try it out at some point.

Linux running on an FPGA was cool, but I didn't do much with it besides brag about it on IRC

![]()

Instead, I went back and compiled [the bare metal example](https://github.com/eugene-tarassov/vivado-risc-v/tree/master/bare-metal) because Linux was super slow, and writing bare metal drivers is much simpler than Linux drivers (part 2, perhaps?). The example is just sending hello world over the serial port by writing to the UART registers, but the key bit is the linker script and startup code, which will come in handy later. The more observant among you will notice one problem with this code though...

## RIIR

It's written in C, so obviously it needed the Rewrite It In Rust treatment. My idea was to create a minimal `no-std` executable, using [`riscv-rt`](), and [`volatile-register`]() for volatile reads/writes to the UART registers. Of course, it couldn't be that simple, and I ended up spending the best part of a day grappling with rustc/lld to generate me an executable that would run, all to no avail. I probably tried every command line flag and cargo configuration parameter in existence, but could not for the life of me get anything to work using the `riscv-rt` crate. My hacky solution in the end was to just compile the rust crate to a static library exposing only `main`, and then invoke the gcc cross-compiler to link using the linker script and startup assembly code from the example. This worked without any trouble, and I had Rust code running on an FPGA!

![](hello from rust.jpg)

The serial interface code is shown below. The hardware has 4 registers: Rx, Tx, and control and status registers. I can't find much documentation on how to use the control/status registers besides the example code, but the idea is simple enough: put some bytes in the Tx register, and it gets sent to the console on the other end.

## Display controller

copied from labs

## Putting it all together

axi and playing with vivado

tested with debugger,
didnt work for ages because active low

## More Rust

new software

## Conclusion

look at scala
much more complex i/o to be done
want to look at axi better

thank you euegene you king
