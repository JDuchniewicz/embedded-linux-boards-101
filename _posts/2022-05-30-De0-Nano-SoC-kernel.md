---
layout: post
title: "De0-Nano-SoC kernel preparation, compilation and kernel modules"
date: 2022-05-30
categories: De0-Nano-SoC
---

# De0-Nano-SoC kernel preparation, compilation and kernel modules

This is a second post in series pertaining to De0-Nano-SoC configuration for having a custom embedded Linux setup with FPGA peripheral. In this part we will go deep into how to compile your own kernel and what steps are crucial for having it up and running.

## Kernel preparation

We will be using Altera's fork of the Linux kernel which you can obtain from under [this link](https://github.com/altera-opensource/linux-socfpga). At the time of writing I am using the newest kernel version which is: 5.15 and in order to do so you first need to clone the repository and then checkout this branch (or a relevant tag).

```
git clone https://github.com/altera-opensource/linux-socfpga
git checkout socfpga-5.15
```

Now, we need to start preparing our kernel - you may want to either export the necessary environment variables:
```
export ARCH=arm
export CROSS_COMPILE=arm-none-linux-gnueabihf-
```
for bash/zsh shells (if you have any other shell you will probably know how to do it :smile:) or specify them each time you run a `make` command.

```
make socfpga_defconfig
make menuconfig
```
After your run menuconfig you will be presented with an ncurses menu where you can choose various configuration variables some of which are necessary for our kernel to work with the FPGA. These are:
* General Setup -> Local version append -> N
* Enable LKM -> Y
* System Type -> Altera SOCFPGA Family -> Y
* Device Drivers -> FPGA Configuration Framework -> Manager + Bridges
* File Systems -> Ext4
* Kernel Hacking -> Kernel Debugging -> Y

Choose the rest of the options suiting your needs (don't worry, you will find yourself getting familiar with them soon enough once you need some functionality that is missing from your bare installation, such as networking or audio).

## Kernel compilation
With the kernel configured, we can finally compile it:
```
make -j24 zImage
```

After the compilation is successful, the resultant zImage can be seen in `arch/arm/boot`.

## Creating and compiling the kernel modules

Once you compiled your kernel, you are almost good to go and start writing and compiling your own drivers. (In case you are fresh to this topic, be sure to peruse [RocketBoards guide](https://rocketboards.org/foswiki/Documentation/EmbeddedLinuxBeginnerSGuide))

You can find an example of a simple driver in the [repository](https://github.com/JDuchniewicz/de0-nano-soc-starter/tree/master/software/module); compile it and you will be presented about messages that symbols are unknown or that a linker script is missing. Since kernel 5.10, when building kernel modules out-of-tree you need to run two more steps that will fulfill these requirements. Run these two commands in the kernel directory:
```
make modules_prepare
make modules
```

Now you will be able to successfully compile the module, good job!

## Loading the kernel and modules on our board

After we have created both kernel and our custom module(s) we usually would like to check whether or not our work was in vain. Probably, putting our kernel first and then the modules is the proper order of setting things up, but that is up to you.

Remember our [directory layout](De0-Nano-SoC-bootloader.md#File structure) from the previous post? We will again be putting the content onto the `sd_card` and into the `sdfs` partition.

In the `arch/arm/boot` directory of the linux-socfpga repository run:
```
cp zImage ../../../../sd_card/sdfs
```

The kernel module will be copied once we have the rootfs up and running. Also, we are still missing a key component of our system - the Device Tree Structure (or its binary version called a "blob"). They are covered in the next posts in these series.
