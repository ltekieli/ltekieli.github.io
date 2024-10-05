---
layout: post
title: Buildroot with Raspberry Pi - U-Boot
date: '2016-04-10 14:54:42'
tags:
- buildroot
- raspberry-pi
- linux
- u-boot
---

Raspberry Pi has a fairly complicated boot process with two bootloaders. The first one resides in built-in ROM and is responsible for starting the GPU. The GPU executes bootcode.bin, the second bootloader, which in the end runs the kernel. Although, there is a possibility to have the root file system booted from network with the stock firmware (actually the kernel allows that), lets look at an interesting alternative. I will use [U-Boot](http://www.denx.de/wiki/U-Boot), and show how to step by step migrate to a more customizable bootloader.

Check [how to start with Buildroot and Raspberry Pi first](/buildroot-with-raspberry-pi-what-where-and-how/).

Use Buildroot to compile U-Boot:

    $ make nconfig

Go to **Bootloaders** and select **U-Boot**. **U-Boot board name** shows up, and needs to be set to **rpi**. After exiting config, run:

    $ make all

After compiling, Buildroot puts **u-boot.bin** in **output/images**. Now the SD card needs to be mounted and the U-Boot binary should be copied to boot partition. Also, in **config.txt** , which is also on the boot partition, change:

    kernel=zImage

to:

    kernel=u-boot.bin

Powering up the RPi with serial console attached gives this result:

    U-Boot 2016.01 (Apr 09 2016 - 14:55:52 +0200)
    
    DRAM: 412 MiB
    RPI Model B rev2 (0xe)
    MMC: bcm2835_sdhci: 0
    reading uboot.env
    
    **Unable to read "uboot.env" from mmc0:1**
    Using default environment
    
    In: serial
    Out: lcd
    Err: lcd
    Net: Net Initialization Skipped
    No ethernet found.
    Hit any key to stop autoboot: 0 
    U-Boot>

U-Boot waits 3 seconds for the user to interrupt the autoboot. After that it will either search for a start script or try to setup network interfaces and boot from network. Interrupting the autoboot will give access to command line interface. In order to boot Linux, the following needs to be done:

    mmc dev 0
    fatload mmc 0:1 ${kernel_addr_r} zImage
    fatload mmc 0:1 ${fdt_addr_r} bcm2708-rpi-b.dtb
    setenv bootargs console=ttyAMA0,115200 root=/dev/mmcblk0p2 rootwait
    bootz ${kernel_addr_r} - ${fdt_addr_r}

What is going on here is that first a switch to proper memory card is made. Then, the zImage (kernel) and device tree blob are loaded from flash to memory at addresses represented by **kernel\_\_\_addr\_\_\_r** and **fdt\_\_\_addr\_\_\_r** variables. Next, kernel boot arguments are set. **bootz** command starts the kernel. It actually needs three arguments, but for now the second argument is omitted with a hyphen, because it's not mandatory. After that, Linux is up and running.

Of course, typing those commands every time is pointless, so a startup script can be created. The above commands need to be put in a script named boot.scr and compiled into U-Boot image with:

    $ mkimage -A arm -O linux -T script -C none -n boot.scr -d boot.scr boot.scr.uimg

**Note!** mkimage is a part of u-boot-tools available in Ubuntu

boot.scr.uimg should be copied to boot partition on SD card. U-Boot searches for a script with this name, and executes the compiled commands.

###### initramfs

Now that U-Boot is ready, lets force the kernel to use an in-memory root file system. Again, this can be achieved by changing Buildroot configuration:

    $ make nconfig

Go to **Filesystem images** , check **cpio the root filesystem** and choose **gzip** as **Compression method**. Also **Create U-Boot image of the root filesystem** needs to be checked. Run **make all** and copy the **rootfs.cpio.uboot** from **output/images** to boot partition.

Now, boot.scr needs to be altered:

    mmc dev 0
    fatload mmc 0:1 ${kernel_addr_r} zImage
    fatload mmc 0:1 ${fdt_addr_r} bcm2708-rpi-b.dtb
    fatload mmc 0:1 ${ramdisk_addr_r} rootfs.cpio.uboot
    setenv bootargs console=ttyAMA0,115200
    bootz ${kernel_addr_r} ${ramdisk_addr_r} ${fdt_addr_r}

Linux will not use the rootfs from mmcblk0p2, instead it will initialize the root file system from the rootfs.cpio.uboot image provided. Note that any changes made in runtime will be lost on reboot, since ramfs does not update the image.

Using initramfs is especially handy, when you care about the lifetime of the memory card. Now all reads and writes are done in-memory, and flash is used only at boot time.

Now the second partition can be erased:

    $ sudo fdisk /dev/mmcblk0
    d - delete partition
    2 - second partition
    n - create partition
    p - primary
    2 - partition number
    first sector: default
    last sector: default
    w - write

Create file system:

    $ sudo mkfs.ext4 /dev/mmcblk0p2

Data partition can be mounted on RPi:

    # mkdir /mnt/sdcard
    # mount /dev/mmcblk0p2 /mnt/sdcard

###### tftp boot

To simplify the process of loading new kernels and rootfs', instead of copying them to the SD card, and moving it back and forth, U-Boot allows to setup full network boot using tftp.

To achive this the machine serving the images needs to have a dhcp and tftp server running. Fortunately, **dnsmasq** provides both. It is available in every Linux distribution, so installing it is easy. Configuration can be found in **/etc/dnsmasq.conf** and it should be as follows:

    interface=eth0
    dhcp-range=eth0,192.168.0.50,192.168.0.150,12h
    enable-tftp
    tftp-root=/var/ftpd

You can choose the interface and IP ranges served by dhcp. Also a path to a directory, where images will be stored, needs to be specified.

After copying zImage, rootfs.cpio and bcm2708-rpi-b.dtb to **/var/ftpd** , use U-Boot to fetch those images:

    usb start  
    dhcp ${kernel_addr_r} zImage  
    tftp ${fdt_addr_r} bcm2708-rpi-b.dtb  
    tftp ${ramdisk_addr_r} rootfs.cpio.uboot  
    setenv bootargs console=ttyAMA0,115200  
    bootz ${kernel_addr_r} ${ramdisk_addr_r} ${fdt_addr_r}

USB needs to be started first, as it turns out that RPi has a network card which is a USB device. zImage is downloaded after dhcp client acquires an IP address. Device tree blob and rootfs are downloaded with tftp, as U-Boots already has the IP.

We can even prepare multiple scripts, one for downloading and storing the images on SD card and, second one, the default, to always boot what can be found on the card

boot.scr:

    mmc dev 0  
    fatload mmc 0:1 ${kernel_addr_r} zImage  
    fatload mmc 0:1 ${fdt_addr_r} bcm2708-rpi-b.dtb  
    fatload mmc 0:1 ${ramdisk_addr_r} rootfs.cpio.uboot  
    setenv bootargs console=ttyAMA0,115200  
    bootz ${kernel_addr_r} ${ramdisk_addr_r} ${fdt_addr_r}  

update.scr:

    usb start  
    dhcp ${kernel_addr_r} zImage
    fatwrite mmc 0:1 ${fileaddr} zImage ${filesize}
    tftp ${fdt_addr_r} bcm2708-rpi-b.dtb
    fatwrite mmc 0:1 ${fileaddr} bcm2708-rpi-b.dtb ${filesize}
    tftp ${ramdisk_addr_r} rootfs.cpio.uboot
    fatwrite mmc 0:1 ${fileaddr} rootfs.cpio.uboot ${filesize}
    setenv bootargs console=ttyAMA0,115200  
    bootz ${kernel_addr_r} ${ramdisk_addr_r} ${fdt_addr_r}

    $ mkimage -A arm -O linux -T script -C none -n boot.scr -d boot.scr boot.scr.uimg
    $ mkimage -A arm -O linux -T script -C none -n update.scr -d update.scr update.scr.uimg

Scripts themselves can also be downloaded via tftp:

    usb start
    dhcp ${scriptaddr} update.scr.uimg
    source ${scriptaddr}

**source** command runs the script.

To sum up, U-Boot gives a lot of options to boot the RPi. Besides the ones I showed, one can also perform U-Boot update or write sophisticated scripts to decide how to perform the boot process, which is especially handy during development.

<!--kg-card-end: markdown-->