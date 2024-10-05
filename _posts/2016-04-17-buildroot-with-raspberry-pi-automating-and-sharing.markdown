---
layout: post
title: Buildroot with Raspberry Pi - Automating and sharing
date: '2016-04-17 08:54:05'
tags:
- buildroot
- raspberry-pi
- linux
- u-boot
- git
- automation
---

Automation is an important part of every software project. If you need to do something repeatedly, chances are, that it can be easily put in a script and done for you. It is especially important in build systems, that is why I would like to automate all the steps, that are needed for building a custom Linux for RPi using Buildroot.

First, check out previous posts:

- [Buildroot with Raspberry Pi - What, where and how to start](/buildroot-with-raspberry-pi-what-where-and-how/)
- [Buildroot with Raspberry Pi - U-Boot](/buildroot-with-raspberry-pi-u-boot/)

First step is to choose a version control system. Since Buildroot uses git, I will stick to that. A fork of Buildroot's repository needs to be made:

    $ git clone https://git.buildroot.net/buildroot
    $ cd buildroot
    $ git push git@bitbucket.org:ltekieli/buildroot_rpi.git origin/master
    

Basically, I copied the existing repo to another, hosted on bitbucket. This allows me to make all the changes, and, when needed, merge any new code that is added to the original repository. From now on, I can clone the repository directly from:

    $ git clone git@bitbucket.org:ltekieli/buildroot_rpi.git

Most of the configuration changes made to Buildroot up until now were made using **make nconfig**. The configuration can be saved using:

    $ make savedefconfig

The outcome of this command is a new configuration file which can be found in **configs/raspberrypi\_defconfig**. Just copy this configuration to the newly setup repo, and from now on, Buildroot will use this config by default.

Now, u-boot command scripts needs to be added. Copy **boot.scr** and **update.scr** into **board/raspberrypi/**

In order to convert them to u-boot images the **board/raspberrypi/post-image.sh** needs to be changed to:

    #!/bin/sh
    
    BOARD_DIR="$(dirname $0)"
    BOARD_NAME="$(basename ${BOARD_DIR})"
    GENIMAGE_CFG="${BOARD_DIR}/genimage-${BOARD_NAME}.cfg"
    GENIMAGE_TMP="${BUILD_DIR}/genimage.tmp"
    MKIMAGE=$HOST_DIR/usr/bin/mkimage
    BOOT_CMD=$BOARD_DIR/boot.scr
    BOOT_CMD_H=$BINARIES_DIR/boot.scr.uimg
    UPDATE_CMD=$BOARD_DIR/update.scr
    UPDATE_CMD_H=$BINARIES_DIR/update.scr.uimg
    
    # U-Boot script
    if [-e $MKIMAGE -a -e $BOOT_CMD];
    then
        $MKIMAGE -A arm -O linux -T script -C none -d $BOOT_CMD $BOOT_CMD_H
    fi
    
    # U-Boot script
    if [-e $MKIMAGE -a -e $UPDATE_CMD];
    then
        $MKIMAGE -A arm -O linux -T script -C none -d $UPDATE_CMD $UPDATE_CMD_H
    fi
    
    # Mark the kernel as DT-enabled
    mkdir -p "${BINARIES_DIR}/kernel-marked"
    ${HOST_DIR}/usr/bin/mkknlimg "${BINARIES_DIR}/zImage" \
        "${BINARIES_DIR}/kernel-marked/zImage"
    
    rm -rf "${GENIMAGE_TMP}"
    
    genimage \
        --rootpath "${TARGET_DIR}" \
        --tmppath "${GENIMAGE_TMP}" \
        --inputpath "${BINARIES_DIR}" \
        --outputpath "${BINARIES_DIR}" \
        --config "${GENIMAGE_CFG}"
    
    exit $?

**post-image.sh** is a script which is automatically run by Buildroot after compilation and rootfs preparation. In case of Raspberry Pi, this scripts generates the **sdcard.img** using genimage tool, based on **board/raspberrypi/genimage-raspberrypi.cfg**

The **genimage-raspberrypi.cfg** needs to be changed to add u-boot, scripts and initial root file system:

    image boot.vfat {
      vfat {
        files = {
          "bcm2708-rpi-b.dtb",
          "bcm2708-rpi-b-plus.dtb",
          "bcm2708-rpi-cm.dtb",
          "rpi-firmware/bootcode.bin",
          "rpi-firmware/cmdline.txt",
          "rpi-firmware/config.txt",
          "rpi-firmware/fixup.dat",
          "rpi-firmware/start.elf",
          "kernel-marked/zImage",
          "boot.scr.uimg",
          "update.scr.uimg",
          "u-boot.bin",
          "rootfs.cpio.uboot"
        }
      }
      size = 32M
    }
    
    image sdcard.img {
      hdimage {
      }
    
      partition boot {
        partition-type = 0xC
        bootable = "true"
        image = "boot.vfat"
      }
    }

Lastly, we need to change **cmdline.txt** and **config.txt** , so that u-boot.bin is executed at startup. Both files are part of rpi-firmware, which can be found in **package/rpi-firmware/**

After all, the changes are:

    $ git status
    On branch master
    Your branch is up-to-date with 'origin/master'.
    Changes to be committed:
      (use "git reset HEAD <file>..." to unstage)
    
    	new file: board/raspberrypi/boot.scr
    	modified: board/raspberrypi/genimage-raspberrypi.cfg
    	modified: board/raspberrypi/post-image.sh
    	new file: board/raspberrypi/update.scr
    	modified: configs/raspberrypi_defconfig
    	modified: package/rpi-firmware/cmdline.txt
    	modified: package/rpi-firmware/config.txt
    

After pushing those changes, the whole build system, with its initial configuration, can be shared with other developers.

<!--kg-card-end: markdown-->