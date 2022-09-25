# About


In this repository, I will demonstrate how to boot a 64-bit kernel using u-boot and busybox. I struggled to find a complete guide to make me able to do that. I did some reverse engineering on a working system to understand what I'm supposed to do to boot my custom kernel. So I will try to show that in detail with some screen shots. My purpose is to make things easy for everyone else to build their own kernel without hassle. 

I use a USB drive to boot my custom kernel and not an SD card. A USB drive is much easier for you to remove and plug back in again. Luckily, rpi3b+ can boot from a USB drive without any necessary configuration. All you just need to do is to unplug the SD card if it exists.



# Raspberry pi boot process


<img src="imgs/boot-process.png" width="200">


First, the ROM bootloader shipped with the vendor starts. Then, the second bootloader `bootcode.bin` starts. After that, `start.elf` starts and makes a Flattened Device Tree (FDT) using the device tree binary .dtb in the boot folder and applies overlays that're written in `config.txt`.


It's important to mention how to apply a device tree overlay when it comes to raspberry pi. `u-boot` uses a library called `libfdt` to deal with device trees and apply overlays to them. Unfortunately, raspberry pi device trees don't follow the standard that this library uses. So, whenever I tryied to apply an overlay, it always fails with `FDT_ERR_NOTFOUND`.


To get over this problem, we can <b>reuse</b> the same `FDT` offered by `start.elf` and make u-boot pypasses it to the kernel. This will allow us to enable whatever overlays we want by writing their names in `config.txt`. The downside of this method is that we always need to provide the device tree in the boot directory. It can't be downloaded from an


# 1- Prepare your USB drive

Your USB drive should have two partitions. One should be in `FAT` format. Usually, it's called `boot`. This filesystem will hold the kernel files, the bootloaders, device trees, and overlays. Another one can be in ext4 format. This one holds the linux file system. It's usually called `rootfs`. The file system size should be large enough to hold the content in it.

You can partition your SD card using either the Ubuntu Disks utility or `fdisk` command. There are plenty of sources out there to show you how to partition your drive.

<img src="imgs/partitions.png">
This is my USB drive partitions.

# 2- Vendor-specific bootloaders and files

There are vendor-specific files that should exist in the `boot` filesystem. These files are:

1. `boocode.bin`: this is the bootloader, which is loaded by the SoC on boot, does some very basic setup, and then loads the `start.elf` file.
1. `start.elf`: its job is to combine overlays with an appropriate base device tree, and then to pass a fully resolved Device Tree to the kernel.
1. `config.txt`: Contains many configuration parameters for setting up the Raspberry Pi.
1. `fixup.dat`: This is a linker file.

Those files can be found in the firmware repository of raspberry pi <a href="https://github.com/raspberrypi/firmware/tree/master/boot">here</a>.

<img src="imgs/vendor-specific.png">