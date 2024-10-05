---
layout: post
title: Using crosstool-ng to generate a gcc toolchain
date: '2020-07-22 16:19:33'
---

[crosstool-ng](https://crosstool-ng.github.io/) is a handy tool for obtaining custom gcc toolchains which are tailored to your needs. This is especially useful when you are up to building a custom system for your embedded device, like a raspberry pi or a beagleboard.

### Building crosstool-ng itself

crosstool-ng can be obtained in binary form, but I encourage to build it from source, as you can get latest updates whenever you want. The procedure is easy, but it's worth thinking about supporting different versions in parallel. My setup looks like this:

    # put those to ~/.bashrc
    export WORKSPACE="$HOME/workspace"
    # from commit 5659366bf62b5555bf914b5f55e8a01c92d6c6f1
    export CROSSTOOL_NG_VERSION="5659366b"

    $ source ~/.bashrc

    # prepare workspace
    $ mkdir -p "$WORKSPACE"
    
    $ cd "$WORKSPACE"
    $ git@github.com:crosstool-ng/crosstool-ng.git
    
    $ cd crosstool-ng
    $ git checkout "$CROSSTOOL_NG_VERSION"
    $ ./bootstrap
    $ ./configure --prefix="$WORKSPACE"/crosstool-ng-"$CROSSTOOL_NG_VERSION"-bin
    $ make -j $(nproc)
    $ make install

    # put this to ~/.bashrc
    export PATH="$WORKSPACE/crosstool-ng-$CROSSTOOL_NG_VERSION-bin/bin:$PATH"

    $ source ~/.bashrc

Now crosstool-ng is installed in a custom directory and available to use anywhere  
after configuring the PATH enviroment variable.

You can now run:

    ct-ng help

in order to check the installation.

### Building a 64bit toolchain for raspberry pi 3b+

Toolchains can be build in any directory, however, we can help crosstool-ng decide where to store artifacts and where to store built toolchains:

    # directory for storing toolchains
    $ mkdir -p $WORKSPACE/toolchains
    # directory for storing download artifacts
    $ mkdir -p $WORKSPACE/src

    # put this to .bashrc
    # tells crosstool-ng the place to store toolchains after building
    export CT_PREFIX="$WORKSPACE/toolchains"

    $ source ~/.bashrc

Now let's build the actual toolchain:

    # workspace for building RPI3 toolchain
    $ mkdir -p $WORKSPACE/toolchain_rpi3_64
    $ cd $WORKSPACE/toolchain_rpi3_64
    
    # check available toolchains
    $ ct-ng list-samples
    
    # initialize config
    $ ct-ng aarch64-rpi3-linux-gnu
    
    # run menuconfig in order to adjust configuration
    $ ct-ng menuconfig
    
    # In menuconfig:
    # Navigate to "Paths and misc options"
    # Change "${HOME}/src" to "${WORKSPACE}/src" for "Local tarballs directory"
    # Navigate to "Operating system"
    # Change "Version of linux" to 5.5.5
    # Navigate to "Toolchain options"
    # Check "Build Static Toolchain"
    
    # start building
    $ ct-ng build
    

It's essential to choose the correct version of the Linux kernel. Rule of thumb is to use version which is older or equal to the one that will be used on the target system.  
Building a static toolchain is optional, but it gives you the possibility to move the toolchain between different machines.

After the build, toolchain is available at:

    $ ls -a1 $WORKSPACE/toolchains/
    .
    ..
    aarch64-rpi3-linux-gnu

### Verifying the toolchain

main.cpp:

    #include <iostream>
    
    int main() {
        std::cout << "Hello RPI3B+64" << std::endl;
        return 0;
    }

    $ "$WORKSPACE"/toolchains/aarch64-rpi3-linux-gnu/bin/aarch64-rpi3-linux-gnu-g++ main.cpp
    
    # check if it is correctly cross-compiled:
    $ file a.out
    
    # run with qemu-aarch64
    $ qemu-aarch64 -L "$WORKSPACE"/seld/workspace/toolchains/aarch64-rpi3-linux-gnu/aarch64-rpi3-linux-gnu/sysroot/ a.out

### Summary

The toolchain we've just build contains only the bare minimum that is needed for cross compiling applications: compiler, libc, c++ runtime. To have something usable we also need to be able to provide additional libraries. This can be achieved by using root filesystem generators like [buildroot](https://buildroot.org/), which I will describe later.

<!--kg-card-end: markdown-->