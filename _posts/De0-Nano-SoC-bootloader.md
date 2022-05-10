---
layout: post
title: "De0-Nano-SoC Intro and bootloader setup"
date: 2022-05-10
categories: De0-Nano-SoC
---

# De0-Nano-SoC Intro and bootloader setup

This series of posts will guide you step by step in order to setup your custom embedded linux setup on De0-Nano-SoC development board. This board is equipped with a FPGA  and we will also setup a custom driver and device tree entry.

This tutorial assumes that you have already installed respective Quartus version (I am still using 19.1 as it is good enough for my needs and I found out that future versions have removed support for this board which is a big shame...). Once you installed it you will need to add a license file that can be obtained through Intel's website.

## Obtaining the GHRD
First of all, we need to obtain GHRD  that will contain all the necessary pin mappings and configs that we will use to create a full custom linux distribution. This is best done from the rocketboards.org [website](https://rocketboards.org/foswiki/Documentation/EmbeddedLinuxBeginnerSGuide) or alternatively on Altera's website. Please note that this is fairly old file but don't be discouraged - we will make it work in no time!

After unpacking the file, you can peruse various catalogues and get familiar with its contents. What is of most relevance to us is: `ghrd.v` and `soc_system.dts`. The first one is Verilog description of the various peripherals and pins and how they connect to each other - basically a top file. The second one is a DTS  which stands for a Device Tree Struct - a mapping of various peripherals in the system and their respective settings. These files are later used for synthesis and subsequent compilations of the bootloader and Linux kernel.

In this guide I will assume we want to create a custom component for our FPGA and provide it to the Linux kernel via the device tree. //Backlink to the relevant page

## File structure
We will follow an arbitrary structure for our GHRD directory, which looks like this:
```
   .
   ├── db // compilation stuff
   ├── greybox_tmp // as well
   ├── hardware
   ├── hps_isw_handoff // handoff files for compilation
   ├── incremental_db // again compilation
   ├── ip // custom IP components we use in this project
   ├── sd_card // our SD-card image and contents strc
   ├── soc_system // the body of our system, connects HPS with FPGA
   └── software

   9 directories

```

Most interesting to us is: `hardware`, `software` and `sd_card`. You need to create them on your own like this:
```
   mkdir -p hardware software sd_card
```

We will be putting UBoot, Linux Kernel, rootfs, and other _soft_ components into the `software` catalogue.

## UBoot configuration and compilation

