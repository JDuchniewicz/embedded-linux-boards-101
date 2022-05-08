---
layout: post
title: "De0-Nano-SoC Intro and bootloader setup"
date: 2022-05-10
categories: De0-Nano-SoC
---

# De0-Nano-SoC Intro and bootloader setup

This series of posts will guide you step by step in order to setup your custom embedded linux setup on De0-Nano-SoC development board. This board is equipped with a FPGA  and we will also setup a custom driver and device tree entry.

This tutorial assumes that you have already installed respective Quartus version (I am still using 19.1 as it is good enough for my needs and I found out that future versions have removed support for this board which is a big shame...). Once you installed it you will need to add a license file that can be obtained through Intel's website.

First of all, we need to obtain GHRD  that will contain all the necessary pin mappings and configs that we will use to create a full custom linux distribution. This is best done from the rocketboards.org [website](https://rocketboards.org/foswiki/Documentation/EmbeddedLinuxBeginnerSGuide) or alternatively on Altera's website. Please note that this is fairly old file but don't be discouraged - we will make it work in no time!

After unpacking the file, you can peruse various catalogues and get familiar with its contents. What is of most relevance to us is: `ghrd.v` and `soc_system.dts`. The first one is Verilog description of the various peripherals and pins and how they connect to each other - basically a top file. The second one is a DTS  which stands for a Device Tree Struct - a mapping of various peripherals in the system and their respective settings. These files are later used for synthesis and subsequent compilations of the bootloader and Linux kernel.


