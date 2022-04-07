---
title: "A Foray Into Embedded Rust"
description: "Programming an Arduino Nano in Rust"
summary: "Who knew how exciting blinking LEDs could be?"
date: 2022-04-06
draft: true
---

**Rust on embedded devices is great.**

It makes perfect sense, because embedded software often needs to be bulletproof, and Rust can do that while still providing the performance and low-level that you need from a systems programming language. C and C++ have been really the only option for embedded for far too long now, and it's great that there is finally an alternative to segfaults and memory CVEs for embedded programmers (and everyone else). I don't have half as much experience with embedded systems as I'd like to, but I've played around with Arduino and Raspberry Pi stuff in the past. It's one of those things where I always want to learn/do more, but anything substantial requires hardware and time, two things that I don't have much of.

I'd bought an Arduino Nano 33 IoT, with grand intentions of Yet Another Project which never came to fruition properly. I only really got as far as flashing some Rust code onto it, and using it to blink LEDs and read a few button presses. This was still cool though, because I learned a lot about the whole embedded ecosystem in Rust along the way.

So, when [UWCS](https://uwcs.co.uk) were looking for people to give talks, I thought I might as well take the opportunity do a little Rust evangelism, and also talk a bit about bare metal programming. The whole idea was to introduce Rust, how it works in the embedded domain, and then do the classic blinky example. This blog post is an accompaniment to that talk I gave (6 months ago now (I've been busy, okay)). It's on Youtube, and personally, I think it's worth the watch.

{{< youtube id="-6nDuX_jMBw" title="My talk on embedded Rust" >}}

## Embedded Rust

[Embedded is a big target for the Rust project](https://www.rust-lang.org/what/embedded), and there is an excellent community working hard to make Rust run on as many things as possible. Microarchitecture crates and peripheral access crates (PACs) provide access to the low-level peripherals in a microcontroller, and then Hardware Abstraction Layers (HALs) provide a, device-agnostic, type-safe, idiomatic, Rusty API over it all. The [Embedded HAL crate](https://github.com/rust-embedded/embedded-hal) provides a set of traits that device-specific HALs then implement. The idea is then that people write drivers generic over the traits, and any device that implements the HAL traits can work with the driver.

For example, there's a [trait to represent a digital output pin](https://docs.rs/embedded-hal/latest/embedded_hal/digital/v2/trait.OutputPin.html) which a new microcontroller could implement for it's own output pins. Any pre-existing device drivers that use digital output pins would then work for this microcontroller. Traits, Abstraction, Polymorphism. Great stuff.

![Rust Evangelism Strike Force](./rust!.webp "Do you have a moment to talk about our Lord and Saviour, Ferris?")

My Arduino is an [ATSAMD31G](https://www.microchip.com/en-us/product/ATsamd21g18), a neat little 32-bit ARM Cortex-M0+ device with 256kB of flash memory, and 32kB of SRAM, all operating at a blazing-fast 48 MHz.

The [`atsamd-rs`](https://github.com/atsamd-rs/atsamd) repo provides me with a [PAC](https://docs.rs/atsamd21g/latest/atsamd21g/), [HAL](https://github.com/atsamd-rs/atsamd/tree/master/hal), and [specific board support crate](https://github.com/atsamd-rs/atsamd/tree/master/boards/arduino_nano33iot) for my board with MCU pins mapped to physical board pins, which makes it easy to get started (especially because the blinky example in the repo is also the exact thing I used as a demo).

![My Arduino Nano](./arduino-pic.jpg)

## Bare Metal

Writing Rust for microcontrollers means I'm running in a bare-metal environment: no operating system. My code is the only thing running on that CPU, which means theres a few things to consider:

- I can't link to Rust's standard library, because much of the code there assumes the presence of an operating system, a luxury we do not have
- I need a linker script to tell the linker how to lay out my object code in the binary, and where in memory to place it
- I need a panic handler to tell Rust what to do in case of a panic. Without the standard library, we have to define this behaviour explicitly
- The [`cortex_m_rt`](https://github.com/rust-embedded/cortex-m/tree/master/cortex-m-rt) crate provides minimal startup code and a runtime for ARM Cortex-based devices, which I'll link in. It takes care of few micro-architecture specific things for me, such as declaring the entry point of the program and populating the device's vector table

I find taking taking care of all these details interesting. Its the kind of code I enjoy writing that appeals to the engineer in me, as it teaches a lot about how computers _actually_ work (something something "real programming").

## Blinky

[One Github repo later](https://github.com/Joeyh021/arduino-blinky), and the LED I wired up (electronics engineer btw) to pin 10 is blinking. Riveting stuff.

![Blinking LED wired up to the Arduino](./blinky.gif "I love it when my LED turns off and then on again every 200 milliseconds.")

You can watch the video if you want a full walkthrough of the code and how it all works (except for the bit at the end where it doesn't work because I missed a line from my linker script and forgot to declare one of my variables mutable (the old curse of live demos)), but I want to point out some bits I found interesting

Linkers and linkage

panic halts

cross compile

peripherals

- Linker scripts are Cool and Interesting
- `cargo-objcopy` lets me compile, and then copy from an ELF to a `.bin` in one command, which is neat
- I `use panic_halt as _;`, which does nothing except tell cargo to link the crate into my binary so the panic handler symbol within it is there for Rust to use if/when it needs it.
- Peripherals modelled as singletons is neat, because it means code has to explicitly take ownership of peripherals when it needs it. This means a bit of boilerplate just to get a delay device, but we need to
- `rustc` flags are set in `.cargo/config.toml`, where we tell rustc that no, do not compile this code for windows, compile it for the target triple `thumbv6m-none-eabi`: my Arduino
  - Other linker argument fuckery goes on here too
