---
title: STM32 Ethernet Bootloader - I barely know her.
tags:
  - low-level
  - embedded
  - stm32
  - lwip
---
## Inspiration
During my time at my previous internship (shout-out Rocket Lab) I had the opportunity to develop (and debug...) software spanning the full stack. I built reusable libraries to control star trackers and reaction wheel from a POSIX machine, looked at a lot of programs written in C (ISO and non-ISO) running on embedded platforms, and even integrated my control library into a portable web application. But what really fascinated me the most was the code written that explicitly manipulated data directly in registers and directly drove peripherals. 

The programs that fascinated me the most were written to do things like change the contents of the persistent flash, or that utilised specialised hardware on the CPU to parallelize instructions performed on entire arrays of data at fractions of the clock cycles of serial operations. One of those programs being the bootloader! 

Bootloaders come in various shapes with different feature sets, but all of them have something in common: they load a program onto your device and direct your device to begin executing the new program.

You might be familiar with boot loaders already. The device you are reading this on certainly has a bootloader. A bootloader such as the one
on your device may load your operating system (android, IOS, windows, linux, etc) from persistant memory like a hard drive, or a USB stick, or even off your ethernet peripheral. 

But a bootloader does not just load operating systems, it can load any program! It can even load itself if you wanted it to. 

As you might of noticed, there are many ways to load a program using a bootloader, in fact the bootloader is what makes it so easy. It provides a standard interface using whatever peripherals you set up for communication with your device to load and run a new program. 

But how do you get a bootloader on your device in the first place?