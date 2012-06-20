---
layout: post
title: "Hacking Linksys EA3500 firmware for SSH access"
date: 2012-06-19 17:37
comments: true
categories: [linksys, firmware, hack]
---

So I recently upgraded to a [Linksys EA3500](http://homesupport.cisco.com/en-us/support/routers/EA3500) router, deciding to retire my ol' trusty but very hackable [Linksys WRT 160NL](http://homesupport.cisco.com/en-us/wireless/lbc/WRT160NL) for 2 very important, to me, reasons:

- I live in an apartment building and can generally see 20+ wireless networks, and performance often suffers due to the fact that there are only 3 channels in the 2.4GHz band that don't overlap.  With 5Ghz, there are several more available channels, and nobody's using them yet!  But I need the dual band support so that I can still use my 2.4GHz only [iPhone 4S](http://www.apple.com/iphone/).  All of my other devices support the 5GHz bands, no problem.
- The EA3500 has gigabit ethernet ports, as does my [ReadyNAS Duo](http://www.netgear.com/home/products/storage/prosumer/), where I have all my movies and music stored.  One day I noticed that I was transferring data to my [AppleTV 2](http://www.apple.com/appletv/) quicker over 802.11 than over a network cable, and decided I really needed gigabit so I don't spend all day copying movies to my NAS.

I've been using [OpenWRT](http://openwrt.org) for quite a while, and am pretty spoiled with respect to the configurability of my routers, so I started out seeing what I could do to my new EA3500.  It turned out, not much.  None of OpenWRT, [DD-WRT](http://www.dd-wrt.com), or [Tomato](http://www.polarcloud.com/tomato) currently support it.

### Getting and Analyzing the Firmware

A quick bit of googling led me to [Hacking Linksys E4200v2 firmware](http://bramp.net/blog/2012/01/hacking-linksys-e4200v2-firmware/), which is a very similar router, and plenty of discouraging posts about how it's running a Marvell SoC which isn't very community friendly and won't be supported by the open source firmware community anytime soon.  Bummer.  The [Firmware Modificaiton Kit](http://code.google.com/p/firmware-mod-kit/) used for a lot of older routers also doesn't support the EA3500, at least out of the box.

Starting from Brampton's blog post, I was able to obtain the actual firmware image from Linksys's servers using the same techniques, giving me [FW_EA3500_1.0.30.126544.SSA](http://download.linksys.com/updates/to0037158864.pdx/FW_EA3500_1.0.30.126544.SSA).  Analyzing this file, I've determined that it contains:

1. A 64 byte legacy uImage bootloader header (offset 0)
1. A kernel image (offset 0x40)
1. A JFFS2 root filesystem image (offset 0x290000)
1. A cryptographic signature (the last ~400 bytes of the file)

Further analysis has led me to believe that the cryptographic signature is only verified for updated firmware images that are automatically downloaded by the device; those that are uploaded manually via the web interface will not be verified, and it should be possible to modify the root filesystem however you want, and proceed to flash it.  That said, until I have a working serial console, I'm hesitant to actually try it!

I made a quickie script which splits out the SSA firmware file:

{% gist 2957075 %}

The resulting root image can be mounted and examined using the same techniques Brampton used.

### Disabling WPS

My first goal was to disable [WPS](http://en.wikipedia.org/wiki/Wi-Fi_Protected_Setup) as I simply don't trust it after some [recent security issues](http://www.smallnetbuilder.com/wireless/wireless-features/31664-waiting-for-the-wps-fix).  A quick grep of the root filesystem led me to believe that maybe I could just set `wl0_wps_state=disabled` and `wl1_wps_state=disabled` in nvram, so I set out to make those modifications.  Examination of the firmware led to the discovery of a simple structure for the nvram backups, which are DES encrypted with a hardcoded key before being downloaded by the client browser.  I wrote a script to decrypt and re-encrypt these backups, which allows for changing arbitrary nvram values with the stock firmware.

{% gist 2931407 %}

Unfortunately, setting the `wl*_wps_state=disabled` variables didn't work; it turns out whether to disable wps is a decision that is reevaluated based on several factors, but lucky for me, all you actually have to do is disable SSID broadcasting on your wireless networks and WPS will be disabled as well.  I confirmed this with `iw wlan0 scan` in a [Backtrack VM](http://www.backtrack-linux.org/), and seeing that, indeed, WPS was no longer enabled.  No hacking required ... just not exactly obvious!

### SSH Access

I noticed while poking around the root filesystem that the EA3500 has special treatment for any USB mass storage device that is plugged in with a `packages` directory off the root.  The contents of this directory are automatically linked to `/opt`, and anything in the `/opt/etc/registration.d` directory is executed, init script style, to extend the functionality of the stock firmware.

I should note that this could be a security hole, in that, at first glance, anybody with FTP access to the router could create files in these special directories that will later (after a reboot) be executed by the root user.  But for my purposes, I was happy to take advantage.

I cross compiled the [dropbear ssh server](https://matt.ucc.asn.au/dropbear/dropbear.html) for ARM, using the same toolchain - [CodeBench Lite Edition](http://www.mentor.com/embedded-software/sourcery-tools/sourcery-codebench/editions/lite-edition/) - as was used by Cisco, and placed it in the appropriate directory structure on a USB flash drive.  Upon plugging in the flash drive, the SSH server was launched and I was able to login to the box as the root user (with the admin password for my router.)

```
└┌(%:~)┌- ssh root@gateway 
root@gateway's password: 
~ # id
uid=0(root) gid=0 groups=0
~ # uname -a
Linux gateway 2.6.35.8 #1 Fri Dec 23 17:46:50 PST 2011 armv5tel GNU/Linux
~ # cat /proc/cpuinfo 
Processor   : Feroceon 88FR131 rev 1 (v5l)
BogoMIPS    : 799.53
Features    : swp half thumb fastmult edsp 
CPU implementer : 0x56
CPU architecture: 5TE
CPU variant : 0x2
CPU part    : 0x131
CPU revision    : 1

Hardware    : Feroceon-KW
Revision    : 0000
Serial      : 0000000000000000
~ # cat /proc/version 
Linux version 2.6.35.8 (root@ubuntu) (gcc version 4.2.0 20070413 (prerelease) (CodeSourcery Sourcery G++ Lite 2007q1-21)) #1 Fri Dec 23 17:46:50 PST 2011
```

I'm making available an [archive](files/linksys-ea3500-dropbear.zip) of the dropbear binaries and corresponding init script, for anyone else who wishes to get ssh access to their device.  Permissions don't matter much.  I just extracted it in the root of a normal FAT32 filesystem, but it looks like HFS or even ext2 should work.  I also imagine the same archive will work with an E4200, but havn't tested it.

### What's Left

At this point, the only other thing I really want to accomplish is Tomato style QoS rules (I love the "any TCP connection with more than x kB of downloaded data shall be lower priority" rule), but with shell access it shouldn't be a problem, and I didn't even need to risk flashing a custom firmware!

It seems that, to run OpenWRT or something similar, the wireless driver is the big blocker, but it's available as a binary kernel module (`ap8x.o`) in the root filesystem.  The open source [mwl8k](http://linuxwireless.org/en/users/Drivers/mwl8k) driver, however, says that it currently supports the 88W8366 chipset, which [appears to be the EA3500 chipset](http://www.techinfodepot.info/index.php/Linksys_EA3500), so maybe support isn't as far off as one would think.
