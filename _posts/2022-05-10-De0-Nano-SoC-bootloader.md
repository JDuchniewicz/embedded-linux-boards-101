---
layout: post
title: "De0-Nano-SoC Intro and bootloader setup"
date: 2022-05-10
categories: De0-Nano-SoC
---

# De0-Nano-SoC Intro and bootloader setup

This series of posts will guide you step by step in order to setup your custom embedded linux setup on De0-Nano-SoC development board. This board is equipped with a FPGA  and we will also setup a custom driver and device tree entry.

This tutorial assumes that you have already installed respective Quartus version (I am still using 19.1 as it is good enough for my needs and I found out that future versions have removed support for this board which is a big shame...). Once you installed it you will need to add a license file that can be obtained through Intel's website.

In this guide I assume that you are somewhat familiar with the world of embedded software, and if it is your first time, then I highly recommend reading RocketBoards [intro to the topic](https://rocketboards.org/foswiki/Documentation/EmbeddedLinuxBeginnerSGuide) (it covers somewhat similar scenario although is at the time of writing slightly outdated). It also contains very informative illustrations in case you feel lost in the boot process or how the device-tree works.

## Obtaining the GHRD
First of all, we need to obtain GHRD  that will contain all the necessary pin mappings and configs that we will use to create a full custom linux distribution. This is best done from the rocketboards.org [website](https://rocketboards.org/foswiki/Documentation/EmbeddedLinuxBeginnerSGuide) or alternatively on Altera's website. Please note that this is fairly old file but don't be discouraged - we will make it work in no time!

After unpacking the file, you can peruse various catalogues and get familiar with its contents. What is of most relevance to us is: `ghrd.v` and `soc_system.dts`. The first one is Verilog description of the various peripherals and pins and how they connect to each other - basically a top file. The second one is a DTS  which stands for a Device Tree Struct - a mapping of various peripherals in the system and their respective settings. These files are later used for synthesis and subsequent compilations of the bootloader and Linux kernel.

In this guide I will assume we want to create a custom component for our FPGA and provide it to the Linux kernel via the device tree. //Backlink to the relevant page

## File structure
We will follow an arbitrary structure for our GHRD directory, which looks like this:
```
 atlas_linux_ghrd
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

## Toolchain

I assume that you already know how to use a toolchain and what it is. In short, it is a set of binaries/headers/libraries that are used for cross-compilation of our bootloader, kernel and applications. It is probably best to use a reasonably fresh toolchain as they are constantly optimized and tweaked so why give all of that away. In this case we will be using ARM's toolchain (previously Linaro):
* [arm-none-linux-gnueabihf-](https://developer.arm.com/-/media/Files/downloads/gnu-a/10.3-2021.07/binrel/gcc-arm-10.3-2021.07-aarch64-arm-none-linux-gnueabihf.tar.xz)

After you obtain it remember to unpack it somewhere convenient and add it to your PATH or simply remember to prefix it to every compilation command.

## Preloader generation

Since this system is quite complex, we need a preloader to set up HPS IO pins, clocking and initialize our SDRAM.

Launch embedded_command_shell in the root directory:
```
  <path-to-quartus-tools>/embedded/embedded_command_shell.sh
```

Then launch the bsp-editor:
```
  bsp-editor &
```

Select **File->New HPS BSP** and click on the button next to the **Preloader settings directory** field. Put `atlas_linux_ghrd/hps_isw_handoff/soc_system_hps_0`. This will fill all the necessary fields and they are good enough for us.

After clicking **OK** you can peruse different options and customize the preloader to your needs. We are interested in setting:
* BOOT_FROM_SDMMC
* FAT_SUPPORT
* FAT_BOOT_PARTITION = 1
* FAT_PAYLOAD_NAME = "u-boot.img"

After enabling these options, press **Generate** and wait until it completes to finally **Exit**.

The files you just generated will appear under `software/generated` paired with a Makefile and settings.bsp files. We can now run `make` in the `software` directory in result creating `u-boot-socfpga` and filling it with some preloader specific files.

## UBoot configuration and compilation

Even though, we have the `u-boot-socfpga` directory, we need to obtain the official sources for the UBoot from Altera's GitHub:

```
  git clone https://github.com/altera-opensource/u-boot-socfpga.git
  cd u-boot-socfpga
```

It is recommended to use release branches that are thoroughly tested and verified instead of the HEAD. Hence, checkout the release branch of your liking. For me it was:
```
  git checkout -t ACDS19.3_REL_GSRD_PR
```

----
The following steps were adopted from [another guide that is dedicated solely to bootloader generation for Cyclone V](https://rocketboards.org/foswiki/Documentation/BuildingBootloaderCycloneVAndArria10). Also if you plan to be using a newer Quartus (>19.3) then please follow the flow in the link (however I am not sure if you will be able to run the rest of steps due to dropped support for this board in future releases of Quartus).
---

Then, clean the directory of any lingering thrash:
```
  make mrproper
```

You can take a while to look at the configuration file for Cyclone V at `include/configs/socfpga_cyclone5.h`, however most of the options there won't be crucial to you unless you are designing a custom board with this SoC.

Use the configuration and compile the bootloader:
```
  make CC=<path-to-your-GCC-toolchain> CROSS_COMPILE=arm-none-linux-gnueabihf- socfpga_cyclone5_config
  make CC=<path-to-your-GCC-toolchain> CROSS_COMPILE=arm-none-linux-gnueabihf- -j48
```

After successful compilation, you will obtain `u-boot-with-spl.sfp` that contains both the Preloader and UBoot binaries. However, we still need more steps to have a bootable Linux environment.

Therefore we proceed to:

## Boot Script creation

This is probably one of least documented and most frustrating parts of setting up this board. Various options grow outdated by the day so hopefully with this guide we will keep it up to date :smile:.

The script briefly looks like this:

```
  echo -- Programming FPGA --
  setenv fpgadata 0x2000000;
  fatload mmc 0:1 $fpgadata soc_system.rbf;
  fpga load 0 $fpgadata $filesize;
  bridge enable;

  echo -- Setting Env Variables --
  setenv fdtimage soc_system.dtb;
  setenv fdtaddr 0x00001000;
  setenv bootimage zImage;
  setenv mmcloadcmd fatload;
  setenv mmcroot /dev/mmcblk0p2;
  setenv mmcload 'mmc rescan;${mmcloadcmd} mmc 0:${mmcloadpart} ${loadaddr} ${bootimage};${mmcloadcmd} mmc 0:${mmcloadpart} ${fdtaddr} ${fdtimage};';
  setenv mmcboot 'setenv bootargs console=ttyS0,115200 root=${mmcroot} rw rootwait earlyprintk; bootz ${loadaddr} - ${fdtaddr}';

  run mmcload;
  run mmcboot;
```

Most of the parameters can be checked either in the [UBoot documentation](http://www.denx.de/wiki/view/DULG/UBootCommandLineInterface) or via `help commandname` in the UBoot shell. However, I will explain some of the most important here:

* `setenv key value` - sets the environment variable `key` to a `value`
* `fatload` - instructs the UBoot to read the data to a given address (in our case it is `soc_system.rbf` located at partition 0:1 of mmc to be loaded to address at `fpgadata` env variable)
* `fpga load` - loads the code to the FPGA
* `bride enable` - enable the FPGA to HPS bridges

The rest of the commands are responsible for setting the Device Tree Blob to be loaded at `fdtaddr`, setting the kernel bootImage, setting the root partition, setting the kernel bootargs and finally running the commands defined by us to start up the Linux.

If the bootloader stops booting the kernel, then make sure that nothing new is necessary to add to the boot script.

## Compiling the Boot Script

Compile it to a binary:
```
  mkimage -A arm -O linux -T script -C none -a 0 -e 0 -n "Boot Script DE0-Nano" -d boot.script u-boot.scr
```

## Putting it on the SD card

Now we can put our bootloader binary and the boot script to the SD card directory:

```
  cp u-boot-with-spl.sfp u-boot.scr ../sd_card
```

Next we will go through creating our custom FPGA component and putting it under the Device Tree - we will need it when we boot to our custom kernel.
