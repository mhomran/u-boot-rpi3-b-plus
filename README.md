# About


In this repository, I will demonstrate how to boot a 64-bit kernel using u-boot and busybox. I struggled to find a complete guide to make me able to do that. I did some reverse engineering on a working system to understand what I'm supposed to do to boot my custom kernel. So I will try to show that in detail with some screen shots. My purpose is to make things easy for everyone else to build their own kernel without hassle.



# Raspberry pi boot process


<img src="imgs/boot-process.png" width="200">


First, the ROM bootloader shipped with the vendor starts. Then, the second bootloader `bootcode.bin` starts. After that, `start.elf` starts and makes a Flattened Device Tree (FDT) using the device tree binary .dtb in the boot folder and applies overlays that're written in `config.txt`.


It's important to mention how to apply a device tree overlay when it comes to raspberry pi. `u-boot` uses a library called `libfdt` to deal with device trees and apply overlays to them. Unfortunately, raspberry pi device trees don't follow the standard that this library uses. So, whenever I tryied to apply an overlay, it always fails with `FDT_ERR_NOTFOUND`.


To get over this problem, we can <b>reuse</b> the same `FDT` offered by `start.elf` and make u-boot pypasses it to the kernel. This will allow us to enable whatever overlays we want by writing their names in `config.txt`. The downside of this method is that we always need to provide the device tree in the boot directory. It can't be downloaded from an