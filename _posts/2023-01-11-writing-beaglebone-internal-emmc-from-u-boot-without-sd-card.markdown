---
layout: post
title: Writing BeagleBone internal eMMC from U-Boot, without SD card
date: '2023-01-11 19:10:54'
---

BeagleBone has the possibility to load the bootloader using the serial console and xmodem. To do that lunch picocom with additional --send-cmd definition:

    $ picocom -b 115200 /dev/ttyUSB0 --send-cmd "sx -vv"

If there is no available bootloader on the eMMC and the SD card is not available the device will continuously print the character 'C'. This means, it's ready to receive data. Press CTRL + A and then S. When prompted for the filename send the u-boot-spl.bin file. After that send u-boot.img. The process looks similar to:

    CCCCCCCCCCCCCC
    *** file: /home/lukasz/workspace/custom-oe-distro/build/tmp-glibc/deploy/images/beaglebone/u-boot-spl.bin-beaglebone-2021.01+gitAUTOINC+78a217ca9e-r33
    $ sx -vv /home/lukasz/workspace/custom-oe-distro/build/tmp-glibc/deploy/images/beaglebone/u-boot-spl.bin-beaglebone-2021.01+gitAUTOINC+78a217ca9e-r33
    Sending /home/lukasz/workspace/custom-oe-distro/build/tmp-glibc/deploy/images/beaglebone/u-boot-spl.bin-beaglebone-2021.01+gitAUTOINC+78a217ca9e-r33, 851 blocks: Give your local XMODEM receive command now.
    Xmodem sectors/kbytes sent: 0/ 0kRetry 0: NAK on sector
    Retry 0: NAK on sector
    Retry 0: NAK on sector
    Retry 0: NAK on sector
    Bytes Sent: 109056 BPS:7951                            
    
    Transfer complete
    
    ***exit status: 0***
    
    U-Boot SPL 2021.01-g78a217ca9e (Nov 02 2022 - 20:54:51 +0000)
    Trying to boot from UART
    CC
    *** file: /home/lukasz/workspace/custom-oe-distro/build/tmp-glibc/deploy/images/beaglebone/u-boot-beaglebone-2021.01+gitAUTOINC+78a217ca9e-r33.img
    $ sx -vv /home/lukasz/workspace/custom-oe-distro/build/tmp-glibc/deploy/images/beaglebone/u-boot-beaglebone-2021.01+gitAUTOINC+78a217ca9e-r33.img
    Sending /home/lukasz/workspace/custom-oe-distro/build/tmp-glibc/deploy/images/beaglebone/u-boot-beaglebone-2021.01+gitAUTOINC+78a217ca9e-r33.img, 7008 blocks: Give your local XMODEM receive command now.
    Xmodem sectors/kbytes sent: 0/ 0kRetry 0: NAK on sector
    Bytes Sent: 897152 BPS:8800                            
    
    Transfer complete
    
    ***exit status: 0***
    xyzModem - CRC mode, 7010(SOH)/0(STX)/0(CAN) packets, 4 retries
    Loaded 897140 bytes
    
    
    U-Boot 2021.01-g78a217ca9e (Nov 02 2022 - 20:54:51 +0000)
    
    CPU : AM335X-GP rev 2.1
    Model: TI AM335x BeagleBone Black
    DRAM: 512 MiB
    WDT: Started with servicing (60s timeout)
    NAND: 0 MiB
    MMC: OMAP SD/MMC: 0, OMAP SD/MMC: 1
    Loading Environment from FAT... <ethaddr> not set. Validating first E-fuse MAC
    Net: eth2: ethernet@4a100000, eth3: usb_ether
    Hit any key to stop autoboot: 0 

To transfer the image, the builtin Ethernet or USB Ethernet gadget, if supported, can be used. With the gadget it might happen that larger data transfers are interrupted. It might also be that the image won't fit into available RAM. In order to mitigate those problems, the following python script will split the image and print the correct mmc write commands:

    #!/usr/bin/env python3
    
    
    import math
    import mmap
    import os
    import sys
    import tempfile
    
    
    MAX_FILE_SIZE = 100 * 1024 * 1024
    BLOCK_SIZE = 512
    
    
    def print_usage():
        print("split.py <path_to_image_file>")
        sys.exit(1)
    
    
    def get_size(input_file):
        return os.stat(input_file).st_size
    
    
    def write_chunk(input_file, output_file, start, length):
        filesize = get_size(input_file)
        with open(input_file, "r+b") as in_f:
            m = mmap.mmap(in_f.fileno(), filesize)
            with open(output_file, "w+b") as out_f:
                end = start + length if start + length < filesize else filesize
                out_f.write(m[start:end])
    
        return end - start
    
    
    if __name__ == " __main__":
        if len(sys.argv) != 2:
            print_usage()
    
        input_file = sys.argv[1]
    
        if not os.path.exists(input_file):
            print_usage()
    
        if not os.path.isfile(input_file):
            print_usage()
    
        parts = math.ceil(get_size(input_file) / MAX_FILE_SIZE)
    
        t = tempfile.mkdtemp()
    
        for i in range(0, parts):
            output_file = f"{t}/{os.path.basename(input_file)}.{i:03}"
            sz = write_chunk(input_file, output_file, i * MAX_FILE_SIZE, MAX_FILE_SIZE)
    
            block_start = hex(i * math.floor(MAX_FILE_SIZE / BLOCK_SIZE))
            block_count = hex(math.floor(sz / BLOCK_SIZE))
    
            print(
                f"tftp ${{loadaddr}} {os.path.basename(output_file)}; mmc write ${{fileaddr}} {block_start} {block_count}"
            )
    
        print(t)

    split.py core-image-minimal-beaglebone.wic
    tftp ${loadaddr} core-image-minimal-beaglebone.wic.000; mmc write ${fileaddr} 0x0 0x32000
    tftp ${loadaddr} core-image-minimal-beaglebone.wic.001; mmc write ${fileaddr} 0x32000 0x32000
    tftp ${loadaddr} core-image-minimal-beaglebone.wic.002; mmc write ${fileaddr} 0x64000 0x32000
    tftp ${loadaddr} core-image-minimal-beaglebone.wic.003; mmc write ${fileaddr} 0x96000 0x32000
    tftp ${loadaddr} core-image-minimal-beaglebone.wic.004; mmc write ${fileaddr} 0xc8000 0x146c
    /tmp/tmpufzvd4ah

U-Boot needs to know what IP address it should assign itself, and what is the IP address of the TFTP server. Those are set using environmental variables. Additionally, it is needed to choose the correct mmc device, which is 1 in case of the builtin eMMC.

Now, what is left to be done is to use the above tftp and mmc commands:

    ==> setenv serverip 192.168.20.20
    ==> setenv ipaddr 192.168.20.1
    ==> mmc dev 1
    
    ...
    
    
    => tftp ${loadaddr} core-image-minimal-beaglebone.wic.004; mmc write ${fileaddr} 0xc8000 0x146c
    using musb-hdrc, OUT ep1out IN ep1in STATUS ep2in
    MAC de:ad:be:ef:00:01
    HOST MAC de:ad:be:ef:00:00
    RNDIS ready
    musb-hdrc: peripheral reset irq lost!
    high speed config #2: 2 mA, Ethernet Gadget, using RNDIS
    USB RNDIS network up!
    Using usb_ether device
    TFTP from server 192.168.20.20; our IP address is 192.168.20.1
    Filename 'core-image-minimal-beaglebone.wic.004'.
    Load address: 0x82000000
    Loading: #################################################################
    	 #################################################################
    	 #####################################################
    	 249 KiB/s
    done
    Bytes transferred = 2676736 (28d800 hex)
    
    MMC write: dev # 1, block # 819200, count 5228 ... 5228 blocks written: OK

On the server side, [tftpy](https://tftpy.sourceforge.net/) can be used to serve binaries:

    $ ip addr add 192.168.20.20/24 dev usb0
    $ tftpy_server.py -p 69 -r /tmp/tmpr_erwzlk
    [2023-01-10 21:02:58,437] Server requested on ip 0.0.0.0, port 69
    [2023-01-10 21:02:58,437] Starting receive loop...
    [2023-01-10 21:03:00,789] Setting tidport to 3872
    [2023-01-10 21:03:00,790] Dropping unsupported option 'timeout'
    [2023-01-10 21:03:00,790] requested file is in the server root - good
    [2023-01-10 21:03:00,790] Opening file /tmp/tmpr_erwzlk/core-image-minimal-beaglebone.wic.004 for reading
    [2023-01-10 21:03:00,790] Currently handling these sessions:
    [2023-01-10 21:03:00,790] 192.168.20.1:3872 <tftpy.TftpStates.TftpStateExpectACK object at 0x7ff8dc5116c0>
    [2023-01-10 21:03:01,258] Reached EOF on file core-image-minimal-beaglebone.wic.004
    [2023-01-10 21:03:01,258] Received ACK to final DAT, we're done.
    [2023-01-10 21:03:01,258] Successful transfer.
    [2023-01-10 21:03:01,258] 
    [2023-01-10 21:03:01,259] Session 192.168.20.1:3872 complete
    [2023-01-10 21:03:01,259] Transferred 2676736 bytes in 0.47 seconds
    [2023-01-10 21:03:01,259] Average rate: 44560.57 kbps
    [2023-01-10 21:03:01,259] 0.00 bytes in resent data
    [2023-01-10 21:03:01,259] 0 duplicate packets

After completing the transfer of all parts and resetting the board, the image should boot successfully:

    tequila 2023.01 beaglebone ttyS0
    
    beaglebone login: root
    root@beaglebone:~# uname -a
    Linux beaglebone 6.1.0 #1 SMP Sun Dec 11 22:15:18 UTC 2022 armv7l GNU/Linux

