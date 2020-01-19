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

```bash
$ sudo apt install gcc-arm-linux-gnueabihf
```

Make sure that your system is new enough that it downloads at least version >=6.0. U-Boot will not compile without a sufficiently new toolchain. I had to upgrade my ElementaryOS/Ubuntu installation to 18.04 for the included PPAs to have >=6.0.

## U-Boot

First step is to compile our bootloader. We'll be using the mainline, upstream U-Boot distribution, as the V3S is well-supported and requires no extra patches or special support. Clone the U-Boot repository with

```bash
$ git clone https://github.com/u-boot/u-boot.git
$ cd u-boot
```

You may need the swing and python-dev libraries. Install them before proceeding with the U-Boot compilation process.

```bash
$ sudo apt install swig python-dev
```

In order to compile U-Boot for our particular setup, we'll use the `configs/LicheePi_Zero_defconfig` that Lichee provides as part of the mainline U-Boot repository we just cloned.

```bash
$ make CROSS_COMPILE=arm-linux-gnueabihf- LicheePi_Zero_defconfig
```

If that works, compile the bootloader:

```bash
$ make CROSS_COMPILE=arm-linux-gnueabihf-
```

If the system complains about not being able to find your toolchain, ensure you included the trailing hypen, as it will use the `CROSS_COMPILE` environment variable as a prefix to find the rest of the toolchain utilities (`gcc`, `as`, `ld`, etc).

If all goes well, the system should generate a file called `u-boot-sunxi-with-spl.bin`. This the bootloader binary, and we'll copy this onto our SD card once we have the rest of the components ready.

## Kernel

Next, we'll compile the kernel. For this, we'll need the Lichee fork of the Linux repository; they've kindly created a kernel configuration that works well on the board. It would be a long and arduous process to figure out the correct configuration on our own, so we'll use this configuration for our kernel installation. As the Linux repo is very large with a deep Git history, we'll do a shallow clone of depth=1 and only clone the particular branch we need:

```bash
$ git clone --single-branch --branch="zero-5.2.y" --depth=1 https://github.com/Lichee-Pi/linux.git
$ cd linux
```

This is the mainline Linux kernel, as of version 5.2, with a few changes; as I mentioned above, they added the configuration file for the LicheePi Zero, as well as the device tree (.dts) file and a useful touchscreen device driver. (We won't use that in this tutorial, but it's nice to have anyway.)

Make the configuration by running the following command. This will parse the kernel configuration and generate a .config that the main Make process will use to compile our kernel.

```bash
$ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- licheepi_zero_defconfig
```

If this succeeds, you're ready to compile your kernel. Make note of how many threads/cores you'd like to assign to the job, and use the `-j` option to split the workload across them. For example, my system has 8 threads, so I'll use `-j8`.

```bash
$ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j8 all
```

THis will also compile the device tree. In a nutshell, the device's hardware setup is described in the `arch/arm/boot/dts/sun8i-v3s-licheepi-zero.dts` file. The compilation process (with `make all`) will compile the `.dts` into a compiled, "binary" `.dtb` file that the bootloader/system can read. We'll be copying this `.dtb` file over to our boot SD card, along with the zImage.

## Boot script

To automate U-Boot's boot process, we'll create a small file that serves as an auto-running script. It contains a few U-Boot commands that will run when U-Boot initializes. Create a file called boot.cmd, containing the following U-Boot commands:

```
setenv bootargs console=ttyS0,115200 root=/dev/mmcblk0p2 rootwait panic=10
load mmc 0:1 0x43000000 ${fdtfile}
load mmc 0:1 0x42000000 zImage
bootz 0x42000000 - 0x43000000
```

Because the Sunxi wiki doesn't explain these whatsoever, I'll do the honors and break down each command and what purpose it serves.

```
setenv bootargs console=ttyS0,115200 root=/dev/mmcblk0p2 rootwait panic=10
```

This line sets the boot arguments, essentially a few short arguments we pass to the booting kernel to initialize a few options. To ensure the boot process outputs to the correct UART, we set the boot argument `console=ttyS0,115200`. This will ensure the critical boot debugging information is passed to UART0 at 115200 baud. This is usually what it would default to anyway, but it's nice to be sure. `root=/dev/mmcblk0p2` sets the root of the filesystem to the second partition of the SD card. This will become more clear later, but the second partition is the ext4 partition onto which we will load the actual rootfs (OS filesystem). `rootwait` ensures the boot process will stall and wait for the root storage medium to respond. This is important if the storage is being a bit slow. `panic=10` tells the boot process to reboot/retry booting after 10 seconds if the kernel panics during boot.

```
load mmc 0:1 0x43000000 ${fdtfile}
load mmc 0:1 0x42000000 zImage
```

These commands load the `.dtb` file and zImage into memory from the SD card. The `0:1` identifier indicates we're loading from mmc0, and we're loading from the first (boot) partition. The `.dtb` file will be loaded into 0x43000000, and the kernel image will be loaded into 0x42000000. These are *relatively* arbitrary locations; they're recommended for most of the Allwinner SoCs, but some use different load addresses. (For instance, the trimmed-down F1C100s SoC needs a high memory address like 0x80000000 and 0x80c00000 to load correctly.)

Finally, the `bootz` command boots the kernel, with three arguments specified; first, the kernel location, secondly, an option argument for the initramfs that we're not using, and thirdly, the location of the `.dtb` file.

Once you've created the `boot.cmd` file with these commands, we'll format this file into a `.scr` binary script file that U-Boot can use. Run the command:

```bash
$ mkimage -C none -A arm -T script -d boot.cmd boot.scr
```

The `boot.scr` file is what we'll actually copy onto the SD card; `boot.cmd` will not be used.

## Rootfs

The "rootfs" is the filesystem that is actually used by the operating system; it contains pretty much everything the operating system needs to provide the user (or root, in this case) with all the creature comforts that one expects while using a Linux system. It will contain binaries of common applications, like text editors, system utilities, as well as system files and programs.

There are several ways to create a rootfs. I've only had success with two methods, but that's mostly due to my inexperience, rather than an actual technical limitation. The best method I've found is to use the excellent Buildroot utility to create a BusyBox-based rootfs that contains trimmed down versions of common Linux utilities. This is best suited for extremely resource-constrained systems; it won't give you more advanced functionality like a C compiler, package management, or other features. However, the build process is painless, self contained, and relatively idiot-proof, lending itself well to a beginner's tutorial.

Download the [latest stable release of Buildroot](https://buildroot.org/download.html), extract, and install the application. 

*WIP: Add Buildroot screenshots.*

We'll select a few options for our Buildroot configuration. Run the menu-based configurator.

```bash
$ make menuconfig
```

*WIP: Add Buildroot configurations, once I get back to my Linux machine!*

Once you've configured the Buildroot system with your favorite BusyBox utilities, build the filesystem:

```bash
$ make
```

Yes, it's really that simple. This will take a good while (especially on slower systems), so, make a cup of tea, relax, and come back later. A few interesting notes while you're waiting:

- Buildroot will use its own, internal, freshly downloaded compiler toolchain (if selected).
- Buildroot will resolve most, if not all, of its own internal dependencies at compile time.
- This helps builds be more reliable, less prone to weird package inconsistencies, and just be generally more convenient and idiot-proof!

Once the lengthy build process is complete, you'll have a `rootfs.tar` file in your `output` folder. This is what we'll un-archive into our SD card.

## SD Card

We need to prep the card for use as a boot medium. Most of this is directly from the wonderful Sunxi [bootable SD card guide](https://linux-sunxi.org/Bootable_SD_card), but I've edited it down to the steps that we need to perform for our project.

First, we need to mount and clean the SD card. Safe to say, anything on the card previously will be wiped, so please don't use a card that is precious to you! If you're used to working with removable storage on Linux, this will be old news, but here's the boilerplate commands anyway.

The commands are slightly different depending on whether you're using a USB-based external card reader, or an internally mounted MMC reader. I'm using a USB card reader, so when I run `sudo fdisk -l` to view the connected storage devices, I see my SD card as `/dev/sdb`. If you're using a USB reader too, you'll see your card as `/dev/sdX`, where X is some letter, depending on your configuration and if other SD devices are connected.

**Make sure you identify the correct device. If you accidentally choose the incorrect device, you can permanently destroy data on another device, and irreversibly lose your information.**

Once you've identified your device, export the name as as shell variable for easy use later.

```bash
$ export card=/dev/sdX
```

If the card is connected as a raw MMC device, it will probably appear as `/dev/mmcblk0`. Thus, export the variable as such:

```bash
$ export card=/dev/mmcblk0
```

Wipe the card's partition table with the following command:

```bash
$ sudo dd if=/dev/zero of=${card} bs=1M count=1
```

If you're not familiar with `dd`, the `bs` option indicates we'll be writing a block of size 1M, and `count=1` indicates we'll be writing a single 1M block. `if` is the input, which will be /dev/zero as we're zeroing out the partition table, and `of` is the output, which is our SD card. (Basic stuff, but still important.)

Next, we write the bootloader binary onto the device. Locate the `u-boot-sunxi-with-spl.bin` file we created earlier.


```bash
$ sudo dd if=/path/to/your/binfile/u-boot-sunxi-with-spl.bin of=${card} bs=1024 seek=8
```

Again, we're using `dd` similarly as before, but this time our input is our `.bin` file, and we're seeking 8 1024-byte blocks into the card, because the bootloader needs to start at 8K into the memory region of the card.

Next, we need to create a few partitions; notably, we'll make a small boot partition that will hold our kernel zImage, the device tree binary, and the boot script.

- zImage is the kernel image we compiled earlier. It's a compressed form of the binary, which is decompressed at boot.
- The device tree binary is that compiled `.dtb` file we generated earlier. It's placed alongside the kernel image in the boot partition.
- The boot script, as described above, will be run by U-Boot at startup.