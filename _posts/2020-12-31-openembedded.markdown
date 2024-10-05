---
layout: post
title: Creating a custom OpenEmbedded distro
date: '2020-12-31 08:30:16'
---

All the code that you can read below, can be found under: [https://github.com/ltekieli/custom-oe-distro](https://github.com/ltekieli/custom-oe-distro)

#### OpenEmbedded, Yocto, bitbake confusion.

I have to admit that Yocto and OpenEmbedded were always quite confusing in terms of what is what, especially looking at the history of both and the integration decisions that were made for Yocto. In order to understand both let's take a look at the repositories of Yocto and OpenEmbedded-Core:

- [http://git.yoctoproject.org/clean/cgit.cgi/poky/tree/](http://git.yoctoproject.org/clean/cgit.cgi/poky/tree/)
- [https://git.openembedded.org/openembedded-core/tree/](https://git.openembedded.org/openembedded-core/tree/)

We can see much similarities here. At first sight we can assume that one is build on top of another. Yocto source contains additional directories, most importantly: bitbake, meta-poky and meta-yocto-bsp. What do we need to build an image using yocto?

1. Clone the poky repository
2. Source the oe-init-build-env file
3. Invoke bitbake to build an image, optionally with the MACHINE environment variable

    $ git clone git://git.yoctoproject.org/poky && cd poky
    $ . oe-init-build-env
    $ MACHINE=beaglebone-yocto bitbake core-image-minimal

How does this work with plain OpenEmbedded-Core?

1. Clone the OpenEmbedded-Core repository
2. Obtain bitbake by cloning the repository or making sure it's available in your PATH
3. Source the oe-init-build-env file
4. Invoke bitbake to build an image, for OE-Core (only qemu images are supported by default).

    $ git clone git://git.openembedded.org/openembedded-core && cd openembedded-core
    $ git clone git://git.openembedded.org/bitbake
    $ . oe-init-build-env
    $ bitbake core-image-minimal

The build configurations reported by bitbake differ:

    Build Configuration:
    BB_VERSION = "1.49.0"
    BUILD_SYS = "x86_64-linux"
    NATIVELSBSTRING = "ubuntu-20.04"
    TARGET_SYS = "arm-poky-linux-gnueabi"
    MACHINE = "beaglebone-yocto"
    DISTRO = "poky"
    DISTRO_VERSION = "3.2+snapshot-10955631b09cd0fbf45c018dfc3a4ed687b2fa06"
    TUNE_FEATURES = "arm vfp cortexa8 neon callconvention-hard"
    TARGET_FPU = "hard"
    meta                 
    meta-poky            
    meta-yocto-bsp = "master:10955631b09cd0fbf45c018dfc3a4ed687b2fa06"

    Build Configuration:
    BB_VERSION = "1.49.0"
    BUILD_SYS = "x86_64-linux"
    NATIVELSBSTRING = "ubuntu-20.04"
    TARGET_SYS = "x86_64-oe-linux"
    MACHINE = "qemux86-64"
    DISTRO = "nodistro"
    DISTRO_VERSION = "nodistro.0"
    TUNE_FEATURES = "m64 core2"
    TARGET_FPU = ""
    meta = "master:c2d9612279fce9cbcb738913b2042949f692c4a5"

For Yocto/Poky we can see that the DISTRO variable is set to "poky", whereas for OE-Core it's "nodistro". Also for Yocto/Poky there are by default two additional layers added: meta-poky and meta-yocto-bsp.

The machines are different because we explicitly asked bitbake to build for beaglebone-yocto machine.

So now we can see that Yocto/Poky combines multiple elements:

1. OE-Core
2. bitbake
3. Additional layers: meta-yocto-bsp for reference board support, and meta-poky for reference distribution support

The integration approach made by the Yocto/Poky is to keep all of those components in a single repository that is maintained as a reference integration.

But we can build our own integration approach on top of OE-Core without Yocto/Poky.

#### Checkout and versioning

First thing to consider is: how to checkout the software in our local workspace and how to maintain versioning. We can approach it as Yocto/Poky did it: copy over particular repositories into one repository. We can also use a tool like repo or git submodules to have simpler control over versioning. However, I prefer to use the kas tool: [https://kas.readthedocs.io/en/1.0/](https://kas.readthedocs.io/en/1.0/), which can help maintaining dependencies, versions and build flavors in a consistent way.

To start working with kas, we need a yaml configuration file:

    $ cat kas-x86-64-project.yml 
    header:
      version: 8
    machine: qemux86-64
    distro: nodistro
    target: core-image-minimal
    repos:
      oe-core:
        url: "git://git.openembedded.org/openembedded-core"
        refspec: 514b595bda487ff74ae16539d716628a1d0be8af
        layers:
          meta:
      bitbake:
        url: "git://git.openembedded.org/bitbake"
        refspec: 71aaac9efa69abbf6c27d174e0862644cbf674ef
        layers:
          conf: disabled
    bblayers_conf_header:
      standard: |
        POKY_BBLAYERS_CONF_VERSION = "7"
        BBPATH = "${TOPDIR}"
        BBFILES ?= ""
    local_conf_header:
      standard: |
        CONF_VERSION = "1"
        SDKMACHINE = "x86_64"
        USER_CLASSES = "buildstats image-mklibs image-prelink"
        PATCHRESOLVE = "noop"
        DL_DIR = "/opt/oe/dl"
        SSTATE_DIR = "/opt/oe/sstate"
      debug-tweaks: |
        EXTRA_IMAGE_FEATURES ?= "debug-tweaks"
      diskmon: |
        BB_DISKMON_DIRS = "\
            STOPTASKS,${TMPDIR},1G,100K \
            STOPTASKS,${DL_DIR},1G,100K \
            STOPTASKS,${SSTATE_DIR},1G,100K \
            STOPTASKS,/tmp,100M,100K \
            ABORT,${TMPDIR},100M,1K \
            ABORT,${DL_DIR},100M,1K \
            ABORT,${SSTATE_DIR},100M,1K \
            ABORT,/tmp,10M,1K"

The bblayers\_conf\_header and local\_conf\_header contain standard entries you would find in a out-of-the-box bblayers.conf and local.conf created by oe-init-build-env by default, but with kas we need to be explicit what we want to use. The interesting parts are:

1. header - for kas tool compatibility checks
2. distro - will be set in local.conf under DISTRO variable
3. machine - the MACHINE we want to use
4. target - default target for bitbake
5. repos - contains all the repositories we want the kas tool to checkout, their versions, as well the meta layers to include in bblayers.conf

With this simple yaml config file we can now issue:

    $ kas build kas-x86-64-project.yml 
    2020-12-28 17:36:06 - INFO - kas 2.3.3 started
    2020-12-28 17:36:06 - INFO - /home/tekieli/workspace/oe-plain$ git rev-parse --show-toplevel
    2020-12-28 17:36:06 - INFO - /home/tekieli/workspace/oe-plain$ hg root
    2020-12-28 17:36:06 - INFO - /home/tekieli/workspace/oe-plain$ git clone -q git://git.openembedded.org/openembedded-core /home/tekieli/workspace/oe-plain/oe-core
    2020-12-28 17:36:06 - INFO - /home/tekieli/workspace/oe-plain$ git clone -q git://git.openembedded.org/bitbake /home/tekieli/workspace/oe-plain/bitbake
    2020-12-28 17:36:09 - INFO - Repository bitbake cloned
    2020-12-28 17:36:19 - INFO - Repository oe-core cloned
    2020-12-28 17:36:19 - INFO - /home/tekieli/workspace/oe-plain/oe-core$ git status -s
    2020-12-28 17:36:19 - INFO - /home/tekieli/workspace/oe-plain/oe-core$ git rev-parse --verify -q origin/514b595bda487ff74ae16539d716628a1d0be8af
    2020-12-28 17:36:19 - INFO - /home/tekieli/workspace/oe-plain/oe-core$ git checkout -q 514b595bda487ff74ae16539d716628a1d0be8af
    2020-12-28 17:36:19 - INFO - /home/tekieli/workspace/oe-plain/bitbake$ git status -s
    2020-12-28 17:36:19 - INFO - /home/tekieli/workspace/oe-plain/bitbake$ git rev-parse --verify -q origin/71aaac9efa69abbf6c27d174e0862644cbf674ef
    2020-12-28 17:36:19 - INFO - /home/tekieli/workspace/oe-plain/bitbake$ git checkout -q 71aaac9efa69abbf6c27d174e0862644cbf674ef
    2020-12-28 17:36:19 - INFO - /home/tekieli/workspace/oe-plain/oe-core$ /tmp/tmpqhd_gk12/get_bb_env /home/tekieli/workspace/oe-plain/build
    2020-12-28 17:36:19 - INFO - /home/tekieli/workspace/oe-plain/build$ /home/tekieli/workspace/oe-plain/bitbake/bin/bitbake -k -c build core-image-minimal
    Loading cache: 100% | | ETA: --:--:--
    Loaded 0 entries from dependency cache.
    Parsing recipes: 100% |########################################################################################################################################################| Time: 0:00:09
    Parsing of 804 .bb files complete (0 cached, 804 parsed). 1411 targets, 74 skipped, 0 masked, 0 errors.
    NOTE: Resolving any missing task queue dependencies
    
    Build Configuration:
    BB_VERSION = "1.49.0"
    BUILD_SYS = "x86_64-linux"
    NATIVELSBSTRING = "ubuntu-20.04"
    TARGET_SYS = "x86_64-oe-linux"
    MACHINE = "qemux86-64"
    DISTRO = "nodistro"
    DISTRO_VERSION = "nodistro.0"
    TUNE_FEATURES = "m64 core2"
    TARGET_FPU = ""
    meta = "HEAD:514b595bda487ff74ae16539d716628a1d0be8af"

kas will checkout the repositories and initiate the build.

Now if we want to add support for more boards, we can create new yaml configurations, add needed layers and change the machine:

    $ cat kas-bbb-project.yml 
    header:
      version: 8
    machine: beaglebone
    
    ...
    
      meta-arm:
        url: "git://git.yoctoproject.org/meta-arm"
        refspec: 9ed331bbd73c2448b7cd0c370e50fe23e5c027f4
        layers:
          meta-arm:
          meta-arm-toolchain:
      meta-openembedded:
        url: "git://git.openembedded.org/meta-openembedded"
        refspec: f431022415f3eaeb1f4563f4c7cdb7e33461e805
        layers:
          meta-oe:
      meta-ti:
        url: "git://git.yoctoproject.org/meta-ti"
        refspec: 46d0622963125ece57338cd46aa9b3710c9e0681
      
    ...
    
    
    $ cat kas-rpi0w-project.yml 
    header:
      version: 8
    machine: raspberrypi0-wifi
    
    ...
    
      meta-openembedded:
        url: "git://git.openembedded.org/meta-openembedded"
        refspec: f431022415f3eaeb1f4563f4c7cdb7e33461e805
        layers:
          meta-oe:
      meta-raspberrypi:
        url: "git@github.com:agherzan/meta-raspberrypi.git"
        refspec: 3e2a8534a6f00d66860a9fab4c59b3117ebda43a
        
    ...

#### Adding a custom layer

Since creating distros and custom integrations require applying specific configurations, it is good to keep them in a separate layer. For this we can use the kas' shell command to run any command in the OE env:

    $ kas shell kas-x86-64-project.yml -c "bitbake-layers create-layer ../meta-custom"

Which will create a new layer in the current directory (or one directory above the build dir).

After that it only needs to be included in the yaml configurations:

        meta-custom:
          path: "../meta-custom"

#### Specifying generic distro policies

In the new meta-custom layer, we need to add a file under conf/distro that will describe our new distribution:

    $ cat meta-custom/conf/distro/tequila.conf 
    DISTRO_NAME="tequila"
    DISTRO_VERSION="2020.12"
    
    SDK_VERSION="tequila.2020.12"
    
    SDKIMAGE_FEATURES += "package-management"
    
    INIT_MANAGER="systemd"
    
    DISTRO_FEATURES_DEFAULT_remove = " alsa bluetooth nfs 3g nfc x11"
    
    DISTRO_FEATURES_BACKFILL_CONSIDERED += "gobject-introspection-data pulseaudio"
    
    EXTRA_IMAGE_FEATURES += "\
        package-management \
        ssh-server-openssh \
    "
    
    PACKAGE_CLASSES += "package_ipk"
    
    CORE_IMAGE_EXTRA_INSTALL += "\
        avahi-daemon \
        e2fsprogs-resize2fs \
    "
    
    PACKAGE_FEED_URIS = "\
        http://192.168.20.1:8080/ipk \
    "

We define the name and version of the distribution as well as the SDK version. As a distro policy we define systemd to be the init system. We enable additional features, like package management for both SDK and the image, as well as ssh server. Our package management format will be IPK and by default the devices can find the package feed on the specified URI. We also remove some unnecessary DISTRO\_FEATURES that are by default enabled in OE-Core.

With this config file we just need to set the distro name to "tequila" in the yaml configuration files and rebuild:

    Build Configuration:
    BB_VERSION = "1.49.0"
    BUILD_SYS = "x86_64-linux"
    NATIVELSBSTRING = "ubuntu-20.04"
    TARGET_SYS = "x86_64-oe-linux"
    MACHINE = "qemux86-64"
    DISTRO = "tequila"
    DISTRO_VERSION = "2020.12"
    TUNE_FEATURES = "m64 core2"
    TARGET_FPU = ""
    meta-custom = "<unknown>:<unknown>"
    meta-oe = "HEAD:f431022415f3eaeb1f4563f4c7cdb7e33461e805"
    meta = "HEAD:514b595bda487ff74ae16539d716628a1d0be8af"

#### Specifying machine related policies

Since we are supporting both Beaglebone and RPI0 Wifi boards, we could use the usb gadget networking to communicate with the devices. Since those are device specific options we could add them in in the yaml files to be included in local.conf files. But on the other hand those are in some way related to our new distribution, we can then include them in a conditional way in the tequila.conf distro configuration file:

    require conf/machine/default.conf
    require conf/machine/${MACHINE}-extra.conf

In the default.conf file we will specify configurations not related to a particular board, and in the ${MACHINE}-extra.conf everything that is specific to a board:

    $ cat meta-custom/conf/machine/default.conf 
    MACHINE_FEATURES_remove =" apm usbhost screen alsa rtc"
    
    MACHINE_FEATURES_BACKFILL_CONSIDERED += "qemu-usermode"
    
    MACHINE_ESSENTIAL_EXTRA_RRECOMMENDS += "kernel-modules"
    
    
    $ cat meta-custom/conf/machine/beaglebone-extra.conf 
    PREFERRED_PROVIDER_virtual/kernel = "linux-ti-mainline"
    
    CORE_IMAGE_EXTRA_INSTALL += "gadget-init-network"
    
    
    $ cat meta-custom/conf/machine/raspberrypi0-wifi-extra.conf 
    PREFERRED_PROVIDER_virtual/kernel = "linux-raspberrypi-dev"
    
    DISABLE_SPLASH = "1"
    ENABLE_UART = "1"
    ENABLE_DWC2_PERIPHERAL = "1"
    RPI_USE_U_BOOT = "1"
    
    
    $ cat meta-custom/conf/machine/qemux86-64-extra.conf 
    PREFERRED_PROVIDER_virtual/kernel = "linux-yocto-dev"

We specified to have all the kernel-modules installed in the images as well as chosen the kernel to be the most up to date one. Additionally we installed packages needed for usb gadget networking and enabled all needed config options.

#### Modifying recipes with appends

For our new distro policies there are some additional configuration changes required in some of the packages.

For systemd we need to enable networkd and configure the network:

    $ tree meta-custom/recipes-core/
    meta-custom/recipes-core/
    ├── systemd
    │   └── systemd_%.bbappend
    └── systemd-conf
        ├── files
        │   ├── g_ether.conf
        │   └── usb0.network
        └── systemd-conf_%.bbappend
        
        
    $ cat meta-custom/recipes-core/systemd/systemd_%.bbappend 
    PACKAGECONFIG_append = " networkd resolved"
    
    
    $ cat meta-custom/recipes-core/systemd-conf/systemd-conf_%.bbappend 
    FILESEXTRAPATHS_prepend := "${THISDIR}/files:"
    
    SRC_URI += " \
        file://g_ether.conf \
        file://usb0.network \
    "
    
    FILES_${PN} += " \
        ${sysconfdir}/modules-load.d/g_ether.conf \
        ${sysconfdir}/systemd/network/usb0.network \
    "
    
    do_install_append() {
        install -d ${D}${sysconfdir}/modules-load.d
        install -d ${D}${sysconfdir}/systemd/network
        install -m 0644 ${WORKDIR}/usb0.network ${D}${sysconfdir}/systemd/network
        install -m 0644 ${WORKDIR}/g_ether.conf ${D}${sysconfdir}/modules-load.d
    }
    
    
    $ cat meta-custom/recipes-core/systemd-conf/files/g_ether.conf 
    g_ether
    
    
    $ cat meta-custom/recipes-core/systemd-conf/files/usb0.network 
    [Match]
    Name=usb0
    
    [Network]
    DHCP=yes

We told systemd to automatically load the g\_ether module and also specified how to configure the usb0 network.

For the TI kernel we decided to use mainline version 5.10:

    $ tree meta-custom/recipes-kernel/
    meta-custom/recipes-kernel/
    └── linux
        └── linux-ti-mainline_git.bbappend
    
    
    $ cat meta-custom/recipes-kernel/linux/linux-ti-mainline_git.bbappend 
    # 5.10 Mainline version
    SRCREV = "2c85ebc57b3e1817b6ce1a6b703928e113a90442"
    PV = "5.10+git${SRCPV}"

And for the gadget-init from meta-ti we need to add missing dependencies needed for runtime operation:

    $ tree meta-custom/recipes-ti/
    meta-custom/recipes-ti/
    └── beagleboard
        └── gadget-init.bbappend
    
    
    $ cat meta-custom/recipes-ti/beagleboard/gadget-init.bbappend 
    SRC_URI_remove = " file://udhcpd.rules"
    
    RDEPENDS_${PN}-network += " \
        bc \
        devmem2 \
    "

After adding the custom kernel and meta-ti appends we need to mask them in the configuration files for x86-64 and rpi0w, as those recipes do not exist in any layers for their build:

    bblayers_conf_header:
      standard: |
        POKY_BBLAYERS_CONF_VERSION = "7"
        BBPATH = "${TOPDIR}"
        BBFILES ?= ""
        BBMASK += "meta-custom/recipes-ti"
        BBMASK += "meta-custom/recipes-kernel/linux/linux-ti-mainline_git.bbappend"

#### Testing the images

If you are using systemd networkd, you can configure it to automatically create a bridge network for all your usb network devices:

    /etc/systemd/network
    ├── br0.netdev
    ├── br0.network
    └── enp0s.network
    
    
    $ cat /etc/systemd/network/enp0s.network 
    [Match]
    Driver=cdc_*
    
    [Network]
    Bridge=br0
    
    
    $ cat /etc/systemd/network/br0.netdev 
    [NetDev]
    Name=br0
    Kind=bridge
    
    
    $ cat /etc/systemd/network/br0.network 
    [Match]
    Name=br0
    
    [Network]
    Address=192.168.20.1/24
    DHCPServer=true
    IPMasquerade=true
    
    [DHCPServer]
    PoolOffset=100
    PoolSize=20
    EmitDNS=yes
    DNS=192.168.1.198

The images for the devices are created in the deploy directory:

    $ ls build/tmp-glibc/deploy/images/beaglebone/core-image-minimal-beaglebone.wic.xz
    $ ls build/tmp-glibc/deploy/images/raspberrypi0-wifi/core-image-minimal-raspberrypi0-wifi.wic.bz2

And after extracting they can be directly copied to the sd cards.

After succesful boot you should be able to ping and ssh to both Beaglebone and RPi0:

    $ ssh root@beaglebone
    root@beaglebone:~# uname -a
    Linux beaglebone 5.10.0-g2c85ebc57b #1 PREEMPT Tue Dec 22 14:08:36 UTC 2020 armv7l GNU/Linux
    
    $ ssh root@raspberrypi0-wifi
    root@raspberrypi0-wifi:~# uname -a
    Linux raspberrypi0-wifi 5.10.1 #1 Tue Dec 22 14:51:00 UTC 2020 armv6l GNU/Linux

#### Generating package feeds

Since we already have support for package management with opkg on the system we can now build all the packages supported by our configurations and expose them via simple http server:

    $ kas shell kas-bbb-project.yml -c "bitbake -k world"
    
    $ cd build/tmp-glibc/deploy
    $ python3 -m http.server 8080

If the device have access to this server, package installation should already work:

    $ ssh root@raspberrypi0-wifi
    root@raspberrypi0-wifi:~# opkg update
    Downloading http://192.168.20.1:8080/ipk/all/Packages.gz.
    Updated source 'uri-all-0'.
    Downloading http://192.168.20.1:8080/ipk/arm1176jzfshf-vfp/Packages.gz.
    Updated source 'uri-arm1176jzfshf-vfp-0'.
    Downloading http://192.168.20.1:8080/ipk/raspberrypi0_wifi/Packages.gz.
    Updated source 'uri-raspberrypi0_wifi-0'.
    root@raspberrypi0-wifi:~# opkg find tree
    tree - 1.8.0-r0
    root@raspberrypi0-wifi:~# opkg install tree
    Installing tree (1.8.0) on root
    Downloading http://192.168.20.1:8080/ipk/arm1176jzfshf-vfp/tree_1.8.0-r0_arm1176jzfshf-vfp.ipk.
    Configuring tree.
    root@raspberrypi0-wifi:~# tree .
    .
    `-- log.log

#### Generating SDK

SDK can be obtained in the regular way using:

    $ kas shell kas-rpi0w-project.yml -c "bitbake core-image-minimal -c populate_sdk"

Which will create a SDK that has a sysroot that conforms to the one located on the image. Alternatively one can build meta-toolchain instead, which contains only the compilers and necessary runtime.

In order to enable package management for the SDK as well, we need to reconfigure the opkg inside the SDK:

    $ tree meta-custom/recipes-devtools/
    meta-custom/recipes-devtools/
    └── opkg
        ├── files
        │   └── environment.d-opkg.sh
        ├── opkg-arch-config_%.bbappend
        └── opkg_%.bbappend
        
        
    $ cat meta-custom/recipes-devtools/opkg/opkg_%.bbappend 
    FILESEXTRAPATHS_prepend := "${THISDIR}/files:"
    
    SRC_URI_append_class-nativesdk = " \
        file://environment.d-opkg.sh \
    "
    
    FILES_${PN}_append_class-nativesdk = " ${SDKPATHNATIVE}/environment-setup.d/opkg.sh"
    
    do_install_append_class-nativesdk() {
        mkdir -p ${D}${SDKPATHNATIVE}/environment-setup.d
        install -m 644 ${WORKDIR}/environment.d-opkg.sh ${D}${SDKPATHNATIVE}/environment-setup.d/opkg.sh
    }
    
    
    $ cat meta-custom/recipes-devtools/opkg/opkg-arch-config_%.bbappend 
    do_compile_append() {
    	mkdir -p ${S}/${sysconfdir}/opkg/
    
    	feedconf=${S}/${sysconfdir}/opkg/sdk-feed.conf
    
    	rm -f $feedconf
    	feeds="${PACKAGE_FEED_URIS}"
    	ipkgarchs="${PACKAGE_ARCHS}"
    	for feed in $feeds; do
    		for arch in $ipkgarchs; do 
    			echo "src/gz $arch $feed/$arch" >> $feedconf
    		done
    	done
    }
    
    
    $ cat meta-custom/recipes-devtools/opkg/files/environment.d-opkg.sh 
    export OPKG_CONF_DIR="/etc/opkg"
    export OFFLINE_ROOT="$SDKTARGETSYSROOT"
    
    if ! grep -q "^option cache_dir" "$SDKTARGETSYSROOT/etc/opkg/opkg.conf"; then
        echo "option cache_dir /var/cache/opkg" >> "$SDKTARGETSYSROOT/etc/opkg/opkg.conf"
    fi
    
    if ! grep -q "^option info_dir" "$SDKTARGETSYSROOT/etc/opkg/opkg.conf"; then
        echo "option info_dir /var/lib/opkg/info" >> "$SDKTARGETSYSROOT/etc/opkg/opkg.conf"
    fi
    
    if ! grep -q "^option status_file" "$SDKTARGETSYSROOT/etc/opkg/opkg.conf"; then
        echo "option status_file /var/lib/opkg/status" >> "$SDKTARGETSYSROOT/etc/opkg/opkg.conf"
    fi
    
    if ! grep -q "^option ignore_uid" "$SDKTARGETSYSROOT/etc/opkg/opkg.conf"; then
        echo "option ignore_uid true" >> "$SDKTARGETSYSROOT/etc/opkg/opkg.conf"
    fi

This will add the necessary package feeds as well as configure the opkg in native sdk to be able to install packages to the target sysroot inside the SDK:

    $ ./oecore-x86_64-armv7at2hf-neon-toolchain-tequila.2020.12.sh -y -d sdk_bbb
    tequila SDK installer version tequila.2020.12
    =============================================
    You are about to install the SDK to "/home/tekieli/workspace/bbb_temp/sdk_bbb". Proceed [Y/n]? Y
    Extracting SDK.........................................................done
    Setting it up...done
    SDK has been successfully set up and is ready to be used.
    Each time you wish to use the SDK in a new shell session, you need to source the environment setup script e.g.
     $ . /home/tekieli/workspace/bbb_temp/sdk_bbb/environment-setup-armv7at2hf-neon-oe-linux-gnueabi
     
    $ source /home/tekieli/workspace/bbb_temp/sdk_bbb/environment-setup-armv7at2hf-neon-oe-linux-gnueabi
    
    $ opkg update
    Downloading http://192.168.20.1:8080/ipk/all/Packages.gz.
    Updated source 'all'.
    Downloading http://192.168.20.1:8080/ipk/armv7ahf-neon/Packages.gz.
    Updated source 'armv7ahf-neon'.
    Downloading http://192.168.20.1:8080/ipk/armv7at2hf-neon/Packages.gz.
    Updated source 'armv7at2hf-neon'.
    Downloading http://192.168.20.1:8080/ipk/beaglebone/Packages.gz.
    Updated source 'beaglebone'.
    
    $ opkg find *spdlog*
    libspdlog-dbg - 1.8.1-r0
    libspdlog-dev - 1.8.1-r0
    libspdlog-src - 1.8.1-r0
    libspdlog1 - 1.8.1-r0
    
    $ opkg install libspdlog-dev
    Installing libfmt7 (7.1.3) on root
    Downloading http://192.168.20.1:8080/ipk/armv7at2hf-neon/libfmt7_7.1.3-r0_armv7at2hf-neon.ipk.
    Installing libspdlog1 (1.8.1) on root
    Downloading http://192.168.20.1:8080/ipk/armv7at2hf-neon/libspdlog1_1.8.1-r0_armv7at2hf-neon.ipk.
    Installing libfmt-dev (7.1.3) on root
    Downloading http://192.168.20.1:8080/ipk/armv7at2hf-neon/libfmt-dev_7.1.3-r0_armv7at2hf-neon.ipk.
    Installing libspdlog-dev (1.8.1) on root
    Downloading http://192.168.20.1:8080/ipk/armv7at2hf-neon/libspdlog-dev_1.8.1-r0_armv7at2hf-neon.ipk.

