Netgear EX2700 OpenWrt flash procedure
======================================

First, using the web interface, update your EX2700 to the latest firmware
(version 1.0.1.8 as of 2016-01-17).

Then, you'll need to setup a TFTP server on your computer. Instructions on
how to do this can be found elsewhere. For the rest of this howto, we'll
assume that your computer uses `192.168.1.2`, while your EX2700 uses
`192.168.1.1`.

The device runs a modified version of OpenWrt (KAMIKAZE, bleeding edge,
r18571) by the way, but we want vanilla, obviously!

## Enabling telnet

By default, telnet is disabled on this device, but there are numerous
"telnetenable" utilities, many of which don't work with this device. I've
used [NetgearTelnetEnable](https://github.com/insanid/NetgearTelnetEnable)
(binaries for Linux and Windows available 
[here](https://github.com/insanid/NetgearTelnetEnable/tree/master/binaries)).

`001122AABBCC` is the router's MAC address. `admin` and `password` are should
be set to the username and password of the router's web interface.

````
$ ./telnetenable 192.168.1.1 001122AABBCC admin password
$ telnet 192.168.1.1
Trying 192.168.1.1...
Connected to 192.168.1.1.
Escape character is '^]'.
 === IMPORTANT ============================
  Use 'passwd' to set your login password
  this will disable telnet and enable SSH
 ------------------------------------------


BusyBox v1.4.2 (2015-06-05 10:24:33 CST) Built-in shell (ash)
Enter 'help' for a list of built-in commands.

  _______                     ________        __
 |       |.-----.-----.-----.|  |  |  |.----.|  |_
 |   -   ||  _  |  -__|     ||  |  |  ||   _||   _|
 |_______||   __|_____|__|__||________||__|  |____|
          |__| W I R E L E S S   F R E E D O M
 KAMIKAZE (bleeding edge, r18571) ------------------
  * 10 oz Vodka       Shake well with ice and strain
  * 10 oz Triple sec  mixture into 10 shot glasses.
  * 10 oz lime juice  Salute!
 ---------------------------------------------------
root@EX2700:/# 
````

## Download neccessary files

The `wget` utility on this device is broken, so we'll have to use `tftp`.

````
root@EX2700:/# cd /tmp
root@EX2700:/tmp# tftp -g -r openwrt-ramips-mt7620-ex2700-squashfs-sysupgrade.bin 192.168.2.1
root@EX2700:/tmp# tftp -g -r ex2700.tar.gz 192.168.2.1
root@EX2700:/tmp#  # tar -xzf ex2700.tar.gz
root@EX2700:/tmp# cd ex2700
````

## Unlocking the bootloader

**!! WARNING !! Modifying the bootloader environment is dangerous, and has
the potential of bricking<sup>[1](#fn1)</sup> your device! Proceed with extreme caution!**

First, print the current value of `bootcmd`. If the output is not exactly
as displayed, do **NOT** proceed!

````
root@EX2700:/tmp/ex2700# ./fw_printenv bootcmd
bootcmd=nmrp; nor_two_part_fw_integrity_check 0xbc040000; bootm 0xbc040000
````

The problematic part here is `nor_two_part_fw_integrity_check 0xbc040000`,
which performs additional integrity checks, which our vanilla OpenWrt will
not pass<sup>[2](#fn2)</sup>. The "u-boot-env" partition is normally locked, 
meaning you cannot modify any of these values:

````
root@EX2700:/tmp/ex2700# ./fw_setenv bootcmd "nmrp; bootm 0xbc040000"
Can't open /dev/mtd1: Permission denied
Error: can't write fw_env to flash
````

To overcome this, we'll use a kernel module which will unlock all locked mtd
partitions ([source](mtd-unlocker.c)).

````
root@EX2700:/tmp/ex2700# insmod mtd-unlocker.ko
root@EX2700:/tmp/ex2700# ./fw_setenv bootcmd "nmrp; bootm 0xbc040000"
````

Now verify that the value is correct.

````
root@EX2700:/tmp/ex2700# ./fw_printenv bootcmd 
bootcmd=nmrp; bootm 0xbc040000
````

## Flashing OpenWrt

Flashing OpenWrt is the easy part:

````
root@EX2700:/tmp/ex2700# cd /tmp
root@EX2700:/tmp/# mtd write openwrt-ramips-mt7620-ex2700-squashfs-sysupgrade.bin firmware
Unlocking firmware ...
Writing from openwrt-ramips-mt7620-ex2700-squashfs-sysupgrade.bin to firmware ... [w]
````

If everything is ok, we can reboot. Keep your fingers crossed!

````
root@EX2700:/tmp/# reboot
````
------------------------------------------------------------
<a name="fn1">1</a>: In most cases, you'll be able to unbrick your device using
a serial console.

<a name="fn2">2</a>: It is entirely possible to create an image that can be flashed
via the router's stock web interface, and which passes the bootloader's integrity
checks, but the kernel in that case is constrained to 982976 bytes (960 kilobytes, 
minus 64 bytes), because the bootloader (u-boot) expects a uImage header in the last
64 bytes of the stock "kernel" mtd partition.
