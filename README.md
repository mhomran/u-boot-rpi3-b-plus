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
1. `start.elf`: its job is to combine overlays with an appropriate base device tree, and then to pass a fully resolved Device Tree to u-boot.
1. `config.txt`: Contains many configuration parameters for setting up the Raspberry Pi.
1. `fixup.dat`: This is a linker file.

Those files can be found in the firmware repository of raspberry pi <a href="https://github.com/raspberrypi/firmware/tree/master/boot">here</a>.

<img src="imgs/vendor-specific.png">

# 3- Build a Device Tree Binary (DTB)

u-boot expects a flattened device tree (FDT) from the start.elf bootloader. The start.elf bootloader needs a device tree binary.dtb file to turn it into an FDT and pass it to u-boot. At this moment, we don't have one. So, we need to compile a device tree source first.


1. We need to clone the Linux kernel. We have two options. The first option is to clone the Linux repository and start configuring the kernel without any default configuration. This is a lot of work. The second option, which I chose, is to clone the Linux kernel maintained by the Raspberry Pi because it provides a default configuration to start from.

    The kernel repository can be found <a href="https://github.com/raspberrypi/linux">here</a>. You can just run 

    `git clone https://github.com/raspberrypi/linux --depth=1`

    `--depth=1` makes sure that we don't get the whole linux repository history which is so big and not useful for our case. We just need the latest tag.

1. We need to have the cross compiler `aarch64-linux-gnu-`
you can install it on ubuntu using this command:

    `sudo apt-get install gcc-aarch64-linux-gnu`

1. Configure the linux kernel using this command:

    `make bcmrpi3_defconfig ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-`

1. Compile the DTBs using this command:

    `make -j12 dtbs ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-`

1. Move the compiled Device Tree Binary file to the boot filesystem. You can find it under 

    `arch/arm64/boot/dts/broadcom/bcm2710-rpi-3-b-plus.dtb`

<b>Note</b>: I found that this device tree is the most compatible one with the overlays. I tried others like bcm2837.

<img src="imgs/dtb.png">


# 4- Configure the raspberry pi through `config.txt`

Your `config.txt` should have these lines:

<img src="imgs/config-txt.png">

- `enable_uart=1` is basically enabling the the uart to be used by `u-boot.bin`

- `kernel=u-boot.bin` gives the kernel name to be loaded which is `u-boot.bin` in our case.

- `arm_64bit=1` forces the kernel loading system to assume a 64-bit kernel, starts the processors up in 64-bit mode.

- `core_freq=250` Frequency of the GPU processor core in MHz.

- `device_tree` specifiy the name of the `.dtb` file to be loaded.

# 5- `u-boot.bin` build

1. clone `u-boot` repository

    `git clone https://github.com/u-boot/u-boot --depth=1`

1. set the rpi3b+ default configuration

    `make rpi_3_b_plus_defconfig ARCH=arm CROSS_COMPILE=aarch64-linux-gnu-`

1. Make further configuration

    `make menuconfig ARCH=arm CROSS_COMPILE=aarch64-linux-gnu-`

    - Save fdt address, passed by the previous bootloader `start.elf`, to env vat `prevbl_fdt_addr` you can find it here:
        
        `Boot options->Save fdt address`

    - Change the Shell prompt to my username, you can find it here:

        `Command Line Interafce->Shell Command`
    
    - Increase the autoboot delay to 20 seconds so that we can have enough time to interrupt the autoboot, you can find it here:

        `Boot options->Autoboot options->delay`
    


1. Build `u-boot`

    - There's an issue I found when I tried to build `u-boot` which is multiple definitions of the function `save_boot_params`. I could get over this by deleting the implementation in board/raspberrypi/rpi/lowlevel_init.S.

    - Now you can run:

        `make -j12 ARCH=arm CROSS_COMPILE=aarch64-linux-gnu-`

    - Move `u-boot-bin` to the boot filesystem

    <img src="imgs/u-boot.png">

    - Plug the usb drive into the raspberry pi. At this point, the u-boot should boot successfully.

<img src="imgs/u-boot-op.png">
u-boot is bootloaded !

# 6- Building the kernel

In step 3, we just built the device tree binaries. There are two more steps to build our custom kernel:
    
1. Configure the kernel with this command:

    `make menuconfig ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-`


1. build the kernel with this command:

    `make -j12 Image ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-`

1. Move the Image to the boot filesystem. You can find it under:
   
    `arch/arm64/boot/Image`

Note: for 64bit kernel, there's no option for making a compressed image e.g. zImage

Note: you can setup a trivial ftp server and grap the image from the host machine. (to be discussed later)

<img src="imgs/kernel-image.png">