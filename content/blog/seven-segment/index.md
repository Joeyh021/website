---
title: "Rust, FPGAs, and Seven Segment Displays"
date: 2022-06-14T23:06:57+01:00
description: "My experience building and programming a simple I/O device for a RISC-V core on an FPGA."
summary: "Getting my feet wet with FPGA design"
draft: true
---

The title of my third year project is "Programmable I/O for Flexible Interfaces in Embedded RISC-V Systems", which basically just means making something very similar to the [Programmable I/O blocks on the Raspberry Pi Pico](https://hackspace.raspberrypi.com/articles/what-is-programmable-i-o-on-raspberry-pi-pico) (which I think are insanely cool), and then integrating them with a RISC-V core to create a simple microcontroller that can interface with basically anything at fairly low power. This means a lot of work with FPGAs and Verilog/HDL which are two things I don't have much experience with.

I did an introductory digital systems design course as part of my degree last year, and learned a bit about Verilog and FPGA design & architecture. The coursework was making a video game (a very common application for FPGAs), and our shitty asteroids clone was mostly thrown together in one very long week of late nights spent fighting with Xilinx Vivado, the FPGA IDE and toolchain, and also perhaps the worst piece of software ever (if you know, you know, and by god do you know).

So, instead of revising for my exams, I spent a few days trying to get to grips better with Xilinx's design tools and FPGA work in general by designing a hardware interface for a seven segment display, integrating it with an existing RISC-V core, and then writing a software driver for it in Rust.

![](fpga sevenseg pic)

## The Setup

My staring point was [this GitHub repo](https://github.com/eugene-tarassov/vivado-risc-v)

talk about rocket chip

show pic of block design and how good it is

started with linux which was very cool

moved to embedded

## RIIR

minimal no-std with volatile-register

did not work - resorted to using eugene's link script

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
