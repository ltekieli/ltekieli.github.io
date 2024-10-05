---
layout: post
title: Using a custom GCC toolchain for embedded Linux development
date: '2020-08-05 17:18:01'
---

So [previously](/using-crosstool-ng-to-generate-a-gcc-toolchain/) I shortly introduced how to build a custom toolchain, now it's time to put it to use.

A bare toolchain is rarely sufficient enough to build useful applications. We usually depend on external libraries, whether it's boost, protobuf or any other piece of code.

This time, let's use [buildroot](https://buildroot.org/) to generate a complete root filesystem, which we can then use to boot our device, as well as a sysroot which we will utilize during cross compilation.

    $ git clone git://git.busybox.net/buildroot
    cd buildroot
    git checkout 2020.05.1
    make raspberrypi3_64_defconfig
    make menuconfig

In the main menu navigate to "Toolchain":

![7](/content/images/2020/08/7.png)

Change the toolchain type to "External Toolchain" and also mark "Custom toolchain". Point the path to the previously built toolchain and set the prefix. Select correct gcc version as well as kernel headers. Don't forget to mark that this toolchain supports C++:  
 ![8](/content/images/2020/08/8.png)

Since the toolchain was built against a recent kernel, it's worth using also a newer one in this build:  
 ![9](/content/images/2020/08/9.png)  
96a3278497cc6540d32dc4a7dba9100da6df16f0 points to the current head of rpi-5.5.y branch.

In order to add a custom library to the system, let's choose [spdlog](https://github.com/gabime/spdlog), which you can find in "Target packages -\> Libraries -\> Logging":  
 ![10](/content/images/2020/08/10.png)

Now build:

    make

As a result we will get the sdcard.img which is ready to be copied to the sd card and will boot the device:

    $ tree -L 1 output/images/
    output/images/
    ├── bcm2710-rpi-3-b.dtb
    ├── bcm2710-rpi-3-b-plus.dtb
    ├── bcm2837-rpi-3-b.dtb
    ├── boot.vfat
    ├── Image
    ├── rootfs.ext2
    ├── rootfs.ext4 -> rootfs.ext2
    ├── rpi-firmware
    └── sdcard.img

And also the sysroot for cross compiling:

    $ tree -L 1 output/staging
    output/staging
    ├── bin
    ├── dev
    ├── etc
    ├── lib
    ├── lib64 -> lib
    ├── media
    ├── mnt
    ├── opt
    ├── proc
    ├── root
    ├── run
    ├── sbin
    ├── sys
    ├── tmp
    └── usr

Now it's time to build an app that uses spdlog:

    $ cp $WORKSPACE/buildroot/output/build/spdlog*/example/example.cpp .
    $ $WORKSPACE/toolchains/aarch64-rpi3-linux-gnu/bin/aarch64-rpi3-linux-gnu-g++ \ 
        --sysroot=$WORKSPACE/buildroot_rpi3_64/output/staging \
        -DSPDLOG_FMT_EXTERNAL \
        -lfmt -lpthread \
        -o example example.cpp

After copying the image to sdcard and the example application to the root filesystem, we check that it runs as expected:

    # uname -a
    Linux buildroot 5.5.19-v8 #1 SMP PREEMPT Mon Aug 3 21:33:07 CEST 2020 aarch64 GNU/Linux
    
    # ./example 
    [1970-01-01 00:00:20.621] [info] Welcome to spdlog version 1.5.0 !
    [1970-01-01 00:00:20.621] [warning] Easy padding in numbers like 00000012
    [1970-01-01 00:00:20.621] [critical] Support for int: 42; hex: 2a; oct: 52; bin: 101010
    [1970-01-01 00:00:20.622] [info] Support for floats 1.23
    ...

So now we have a toolchain, a sysroot and a bootable image. Later I will explain how to put this all together and what are the options for an integration tool.

<!--kg-card-end: markdown-->