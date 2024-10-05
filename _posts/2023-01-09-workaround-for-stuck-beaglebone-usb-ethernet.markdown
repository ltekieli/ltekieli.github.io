---
layout: post
title: Workaround for stuck BeagleBone USB Ethernet gadget
date: '2023-01-09 19:35:20'
---

The BaegleBone board seems to have some problem with USB Ethernet gadget not working correctly. It happens quite frequently that the the USB Ethernet device is not able to receive any packets, although transmission is still possible:

    root@beaglebone:~# ifconfig usb0
    usb0 Link encap:Ethernet HWaddr 22:04:F8:8E:6D:8D  
              inet addr:192.168.20.1 Bcast:192.168.20.255 Mask:255.255.255.0
              inet6 addr: fe80::2004:f8ff:fe8e:6d8d/64 Scope:Link
              UP BROADCAST RUNNING MULTICAST MTU:1500 Metric:1
              RX packets:1 errors:0 dropped:0 overruns:0 frame:0
              TX packets:31 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:1000 
              RX bytes:311 (311.0 B) TX bytes:4290 (4.1 KiB)

Pinging PC from the board over serial:

    root@beaglebone:~# ping 192.168.20.24
    PING 192.168.20.24 (192.168.20.24): 56 data bytes

The packets are reaching the PC connected to the board, but the reply never reaches the board back:

    tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
    listening on enx960e95b84bdd, link-type EN10MB (Ethernet), snapshot length 262144 bytes
    20:26:57.139227 ARP, Request who-has yoga7 tell 192.168.20.1, length 28
    20:26:58.209063 ARP, Request who-has yoga7 tell 192.168.20.1, length 28
    20:26:58.241307 ARP, Reply yoga7 is-at 96:0e:95:b8:4b:dd (oui Unknown), length 28
    20:26:58.241311 ARP, Reply yoga7 is-at 96:0e:95:b8:4b:dd (oui Unknown), length 28
    20:26:59.248556 ARP, Request who-has yoga7 tell 192.168.20.1, length 28
    20:26:59.248579 ARP, Reply yoga7 is-at 96:0e:95:b8:4b:dd (oui Unknown), length 28

In order to work around this problem, all that needs to be done is reloading the g\_ether kernel module:

    root@beaglebone:~# modprobe -r g_ether
    root@beaglebone:~# modprobe g_ether
    [218.703634] using random self ethernet address
    [218.708296] using random host ethernet address
    [218.713667] usb0: HOST MAC 56:4c:ac:00:11:1d
    [218.718078] usb0: MAC 8e:c7:ec:87:4c:ac
    [218.721994] using random self ethernet address
    [218.726461] using random host ethernet address
    [218.731073] g_ether gadget.0: Ethernet Gadget, version: Memorial Day 2008
    [218.737935] g_ether gadget.0: g_ether ready

After this command the network is working again:

    root@beaglebone:~# ping 192.168.20.22
    PING 192.168.20.22 (192.168.20.22): 56 data bytes
    64 bytes from 192.168.20.22: seq=0 ttl=64 time=0.870 ms
    64 bytes from 192.168.20.22: seq=1 ttl=64 time=0.847 ms
    64 bytes from 192.168.20.22: seq=2 ttl=64 time=0.746 ms

This fix can be wrapped in a systemd service:

    [Unit]
    Description=Restarts the gadget ethernet
    After=systemd-udevd.service
    
    [Service]
    Type=oneshot
    ExecStart=/sbin/modprobe -r g_ether
    ExecStart=/bin/sleep 1
    ExecStart=/sbin/modprobe g_ether
    RemainAfterExit=true
    StandardOutput=journal
    
    [Install]
    WantedBy=multi-user.target

The deployment with OpenEmbedded can be found here:

- [https://github.com/ltekieli/custom-oe-distro/tree/c588be88c227c2b6733113b28a7b3cd60bc79486/meta-custom/recipes-core/systemd-bbb-gether-fix](https://github.com/ltekieli/custom-oe-distro/tree/c588be88c227c2b6733113b28a7b3cd60bc79486/meta-custom/recipes-core/systemd-bbb-gether-fix)
