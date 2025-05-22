---
title: DOCSIS - Hacking Puma 6 and Puma 7
author: Andrew LaMarche
date: 2024-01-21
categories: [linux, docsis, hacking]
tags: [linux, docsis, hacking]
---

As always, use the resources provided here responsibly. I am in no way, shape or form responsible for what you do with this.

**DOCSIS** is the **Data Over Cable Service Interface Specification**, which is the technology that runs on cable modems in North America. **EuroDOCSIS** is largely the same, with some small tweaks to make it operate in European markets. This post shares some ideas and concepts that are common for many cable devices using the Puma chips. It will also focus more on the RDK-B implementations.

# Software

There are a variety of different OSes that Puma modems run. Some are proprietary Linux flavors, e.g. the SB6190 [https://sourceforge.net/projects/sb6190.arris/files/](https://sourceforge.net/projects/sb6190.arris/files/), others run a more unified OS called Reference Design Kit for Broadband. In my own opinion, RDK-B is an over-complex and extremely bloated OS that tries to do too much in overly complex ways.

An example is the [Utopia](https://wiki.rdkcentral.com/display/RDK/Utopia) software package. It tries to manage bridging, LAN services, DHCP, SSH, firewall, logging and more. On top of this, the firewall is managed behind the scenes by a C application [(firewall.c)](https://code.rdkcentral.com/r/plugins/gitiles/rdkb/components/opensource/ccsp/Utopia/+/refs/heads/rdk-next/source/firewall/firewall.c) that more or less just applies iptables rules. I think everyone would consider this an anti-abstraction that makes it significantly more difficult to understand develop with. As a hacker though, it presents much more attack surface! Not to mention, the utopia "bundle" should really be broken out into init scripts/service files and their associating config scripts. Further, there are many CLI tools: dmcli, syscfg, sys event, etc. that make operating the OS a disaster. On top of all this, a database is used to store some parts of the config, rather than placing them all somewhere actually organized and useful (see [OpenWrt's UCI](https://openwrt.org/docs/guide-user/base-system/uci) where everything goes into `/etc/config`).

Overall, the unification goal of RDK-B seems to do the opposite, at least all of the modems I've looked at the run RDK-B are twisted and obscured so far that they barely resemble it any more. Enough bashing though, let's continue!

# Chip Architecture

The Puma 6 and Puma 7 chips are nearly identical from the user's perspective; the only difference being that Puma 6 only supports DOCSIS 3.0 while Puma 7 supports DOCSIS 3.1. The [upcoming Puma 8 chip](https://www.maxlinear.com/news/press-releases/2023/maxlinear-announces-availability-of-puma™-8,-its-docsis®-4-0-cable-modem-and-gateway-platfor) will support DOCSIS 4.0, but its design is currently unknown as it is not available to the public yet.

Both Puma 6 and 7 use a dual core design, where one core is an **armv6eb** (big endian) core and the other is an Intel **i686** 32-bit core. Neither of which are super powerful, but they do their job okay-ish.

# Basic Chip Security

In proper implementations, the CPU will be "fused", meaning a public key is stored inside the CPU's eFuses. These are tiny OTP (One-Time Programmable) memory fuses, and they serve the purpose of being tamper resistant; once they are written to, they cannot be changed. At a really high level, the idea is that each firmware component (bootloader, kernel, etc.) are signed with the corresponding private key by the device manufacturer. A cryptographic check is performed on these signed images using the public key stored in the eFuses. This way, (assuming no bootROM exploits exists or they didn't leave UART open), it can be pretty safely assumed that the code running on the device is the original code intended to be run by the manufacturer. For a more in-depth overview, check out Mediatek?' Wiki on it. This is great, however when manufacturers want to be the first to push a new product to the market, sometimes these security features are "forgotten".

# DOCSIS Core: armv6eb

The DOCSIS core is, by today's standards, ancient. Nobody uses big endian ARM cores (granted the justification is that this matches network byte order), and armv6 itself is probably considered antique by now. That being said, its main goal is to run and control any DOCSIS protocol or signal processing code.

```
# uname -a
Linux puma 4.9.215 #1 PREEMPT Tue Apr 7 14:02:36 UTC 2020 armv6b GNU/Linux
```

# Router-Gateway (RG) Core: i686

The router-gateway code is also ancient by today's standards. Nobody uses 32-bit Intel cores any more. That being said, its main goal is only really useful in gateway devices that is, cable modems advertised with built in wireless routers. On standalone cable modems, this core is largely unused. However, gateway devices often use it to serve as the customer's wireless router, firewall, etc. In this configuration, this core is what gets a public IP address from the ISP and then shares it to the customer via Network Address Translation (NAT).

# Stitching Them Together

There are a few different communication mechanisms that take place between these cores, but most notably is the creation of the `adp0` and `ndp0` interfaces that connect the two. Commonly, vlan 555 bridges the two CPUs.

armv6eb core: **192.168.254.253**, network device processor (ndp0)
```
3: ndp0.555@ndp0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 2000 qdisc noqueue state UP group default qlen 1000
    link/ether 00:50:f1:00:00:00 brd ff:ff:ff:ff:ff:ff
    inet 192.168.254.253/24 brd 192.168.254.255 scope global ndp0.555
       valid_lft forever preferred_lft forever
```

i686 core: **192.168.254.254**, appication device processor (adp0)
```
13: adp0.555@adp0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 2000 qdisc noqueue state UP group default qlen 1000
    link/ether 00:50:f1:80:00:00 brd ff:ff:ff:ff:ff:ff
    inet 192.168.254.254/24 brd 192.168.254.255 scope global adp0.555
       valid_lft forever preferred_lft forever
```

# I want to run {}

Toolchains do exist to build software to run directly on these modems, but they are almost antique. Most need Ubuntu 12 to run. A different solution is to use musl-libc and build statically linked applications for one-off tools. For this, I created the Ubiquity Toolkit ([blog post](/blog/the-ubiquity-toolkit), [GitHub](https://github.com/andrewjlamarche/ubiquity-toolkit)). While statically linking can be a bit more painful as all the dependent libraries need to be built as well and linked to by hand, I've found it much more rewarding to figure it out once for one arch, then just switch the cross compiler to build it for another. I'd recommend you read the blog post and check out my GitHub to see how you might build another software package to run on these devices! Pull requests are always welcome.

# Keys, Please!

In many cases, SSH is enabled between the two via a dropbear private key, often located in `/root/.ssh/id_rsa` on each core's rootfs. While this works when your ssh binary is from dropbear, it won't work when you're using OpenSSH. However, dropbear includes a handy binary that allows easy converting between the two. To go from dropbear to OpenSSH and put the new key at `/tmp/ida_rsa`:

```
dropbearconvert dropbear openssh /home/root/.ssh/id_rsa /tmp/id_rsa
```

If SSH is not open on the LAN, you can probably open it by inserting an iptables rule on the RG side:

```
iptables -I INPUT -p tcp --dport 22 -j ACCEPT
```

Then use ssh on your local machine to remote in:

```
ssh -oPubkeyAcceptedKeyTypes=+ssh-rsa -oHostKeyAlgorithms=+ssh-rsa -I id_rsa root@<ip>
```

Otherwise, to pivot between cores, just use the included ssh binary and pass in the key and the RG or arm core IP.

