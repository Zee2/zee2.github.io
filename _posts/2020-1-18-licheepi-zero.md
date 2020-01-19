---
layout: post
title: "Hacking the LicheePi Zero: Crash Course"
---

The LicheePi Zero is a lovely, tiny, single-board computer, running on the ubiquitous and low-cost Allwinner V3S platform. Extremely cheap single-board computers have exploded in popularity in recent years, even beyond the (in)famous Raspberry Pi. Other, smaller manufacturers have popped up, designing simple SBCs around inexpensive ARM SoCs. For hobbyists and hackers that want a more hands-on and challenging experience than you'd find with a Raspberry Pi, these cheap SoCs are a great hobby project and weekend adventure.

To add to the already significant challenge, most of the (sparse) documentation is in Chinese, and many of the necessary files are hosted on Chinese sites that are difficult to use or access from the States. While some challenges are technical in nature and can provide some value to the intrepid hobbyist, crappy documentation and unresponsive Chinese file-sharing sites are *not* the kind of issues I'd like to let stand. Thus, I wanted to share a guide for my English-speaking friends, serving as a concise tutorial for compiling the Linux kernel, a bootloader, and creating a root filesystem for the board.

## Background

One of the main sources of reference for working with Allwinner SoC-based platforms will be [linux-sunxi](https://www.sunxi.org), an open source community that develops software for these low-cost single board computers. They have a great guide for compiling the various components for these boards, and also host a wide variety of repositories containing code specialized for these platforms. We'll be using their configuration for u-boot, and some of the shell snippets I'll share are borrowed from their guides.

The bootloader we'll be using is the popular Das U-Boot, a basic bootloader designed to run on pretty much anything. Because the V3S SoC is well supported, we'll be able to use the mainline, upstream U-Boot repository, with no need for any specialized Lichee or Sunxi forks.

For the kernel, we'll be using the LicheePi fork of the Linux kernel. Theoretically, upstream Linux would work just fine, but I can't find a configuration file for the V3S in the upstream repository. Lichee hosts a fork that includes a configuration file that I've found to work well, so we'll use that. If you figure out the configuration file for the most recent upstream kernel, please let me know, and I can update this! Ideally, we ought to use the upstream versions of both U-Boot and the kernel, but we'll settle for just using upstram U-Boot for now, and using Lichee's Linux fork.

You should be using a relatively up-to-date version of Linux on your workstation; it's possible to do some of this on Windows, but certain tasks like compiling U-Boot and the kernel are much more difficult on Windows. Personally, I try and use WSL for most tasks on my Windows workstation, but even WSL won't be up to the task today. Either run a VM or create a Linux partition. You won't need much disk space for this, the minimum disk space allocation should be fine.

## Hardware

The LicheePi Zero is Lichee's midrange SBC, powered by the Allwinner V3S SoC. The V3S was originally intended for dashcams, but has found a second life as a cheap SoC for hobbyist SBCs! The unit I have here was rougly $12 USD [on Aliexpress](https://www.aliexpress.com/item/4000153270987.html?spm=a2g0s.9042311.0.0.3c2a4c4d3JSZv9), with international shipping included. To run Linux at 1.2 GHz, with integrated DDR2, a 3D graphics accelerator, floating point support, 40-pin LCD output, and more, all in the size of a thumb drive, for only $12? Sounds like a deal to me!

![alt](/images/licheepi_zero.jpg)

The V3S specs are below, as taken from the [sunxi wiki](https://linux-sunxi.org/V3s):

- CPU: Cortex-A7 1.2GHz (ARM v7-A) Processor which have both VFPv3 and NEON co-processors:
- FPU: Vector Floating Point Unit (standard ARM VFPv4 FPU Floating Point Unit)
- SIMD: NEON (ARM's extended general-purpose SIMD vector processing extension engine)
- Integrated 64MB DRAM

As a dashcam SoC, it has robust support for H.264 codecs at 1080p, which is quite remarkable for a sub-$5 chip. 

You'll need a few other bits of hardware, too: most importantly, you'll need some way to talk to the UART on the board. The best way is to use an FTDI breakout board, variants of which are sold *literally everywhere* and can be found for just a few dollars. We'll use this as our `/dev/ttyUSB0` to talk to our board, both while we are working with the bootloader and when we're talking to our Linux shell.

Additionally, we'll be using an MicroSD card (sometimes referred to as a TF card, by some documentation) to flash the bootloader, kernel, and rootfs. The system can support a flash chip, but that's outside the scope of this tutorial, for now. Some people say that larger-capacity MicroSD cards can cause issues; I'm using an old 8GB Kingston card for this tutorial, which hasn't given me any issues. If you're using one with capacity greater than, say, 32GB, and you're having issues; maybe try a smaller one.

## Toolchain

We need to install the compiler toolchain; one pitfall I ran into was that the V3S has hardware floating point support, unlike some other Allwinner chips (F1C100S, for instance). Therefore, we have to be careful to use the `gcc-arm-linux-gnueabihf` toolchain instead of the `gcc-arm-linux-gnueabi` toolchain. Install the toolchain like so:

```
$ sudo apt install gcc-arm-linux-gnueabihf
```

Make sure that your system is new enough that it downloads at least version >=6.0. U-Boot will not compile without a sufficiently new toolchain. I had to upgrade my ElementaryOS/Ubuntu installation to 18.04 for the included PPAs to have >=6.0.

## U-Boot

First step is to compile our bootloader. We'll be using the mainline, upstream U-Boot distribution, as the V3S is well-supported and requires no extra patches or special support. Clone the U-Boot repository with

```
$ git clone https://github.com/u-boot/u-boot.git
$ cd u-boot
```

You may need the swing and python-dev libraries. Install them before proceeding with the U-Boot compilation process.

```
$ sudo apt install swig python-dev
```

In order to compile U-Boot for our particular setup, we'll use the `configs/LicheePi_Zero_defconfig` that Lichee provides as part of the mainline U-Boot repository we just cloned.

```
$ make CROSS_COMPILE=arm-linux-gnueabihf- LicheePi_Zero_defconfig
```

If that works, compile the bootloader:

```
$ make CROSS_COMPILE=arm-linux-gnueabihf-
```

If the system complains about not being able to find your toolchain, ensure you included the trailing hypen, as it will use the CROSS_COMPILE environment variable as a prefix to find the rest of the toolchain utilities (gcc, as, ld, etc).

If all goes well, the system should generate a file called `u-boot-sunxi-with-spl.bin`. This the bootloader binary, and we'll copy this onto our SD card once we have the rest of the components ready.

## Kernel

Next, we'll compile the kernel. For this, we'll need the Lichee fork of the Linux repository; they've kindly created a kernel configuration that works well on the board. It would be a long and arduous process to figure out the correct configuration on our own, so we'll use this configuration for our kernel installation. As the Linux repo is very large with a deep Git history, we'll do a shallow clone of depth=1 and only clone the particular branch we need:

```
$ git clone --single-branch --branch="zero-5.2.y" --depth=1 https://github.com/Lichee-Pi/linux.git
$ cd linux
```

This is the mainline Linux kernel, as of version 5.2, with a few changes; as I mentioned above, they added the configuration file for the LicheePi Zero, as well as the device tree (.dts) file and a useful touchscreen device driver. (We won't use that in this tutorial, but it's nice to have anyway.)