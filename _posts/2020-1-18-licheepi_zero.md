---
layout: post
title: Hacking the LicheePi Zero: Crash Course
---

# Hacking the LicheePi Zero: Crash Course

The LicheePi Zero is a lovely, tiny single-board computer, running on the ubiquitous and low-cost Allwinner V3S platform. Unfortunately, most of the (already sparse) documentation is in Chinese, and many of the necessary files are hosted on Chinese sites that are difficult to use or access from the States. Thus, I wanted to share a guide for my English-speaking friends,serving as a concise tutorial for compiling the Linux kernel, a bootloader, and creating a root filesystem for the board.

## Background

One of the main sources of reference for working with Allwinner SoC-based platforms will be [linux-sunxi](https://www.sunxi.org), an open source community that develops software for these low-cost single board computers. They have a great guide for compiling the various components for these boards, and also host a wide variety of repositories containing code specialized for these platforms. We'll be using their configuration for u-boot, and some of the shell snippets I'll share are borrowed from their guides.

The bootloader we'll be using is the popular Das U-Boot, a basic bootloader designed to run on pretty much anything. Because the V3S SoC is well supported, we'll be able to use the mainline, upstream U-Boot repository, with no need for any specialized Lichee or Sunxi forks.

For the kernel, we'll be using the LicheePi fork of the Linux kernel. Theoretically, upstream Linux would work just fine, but I can't find a configuration file for the V3S in the upstream repository. Lichee hosts a fork that includes a configuration file that I've found to work well, so we'll use that. If you figure out the configuration file for the most recent upstream kernel, please let me know, and I can update this! Ideally, we ought to use the upstream versions of both U-Boot and the kernel, but we'll settle for just using upstram U-Boot for now, and using Lichee's Linux fork.

You should be using a relatively up-to-date version of Linux on your workstation; it's possible to do some of this on Windows, but certain tasks like compiling U-Boot and the kernel are much more difficult on Windows. Personally, I try and use WSL for most tasks on my Windows workstation, but even WSL won't be up to the task today. Either run a VM or create a Linux partition. You won't need much disk space for this, the minimum disk space allocation should be fine.

## Hardware

The single-board computer we'll be setting up is, of course, the LicheePi Zero. The V3S powering the board was originally intended as a dashcam SoC, but has since found its way into many different applications, including this SBC for hobbyists.
