---
title: The Ubiquity Toolkit
author: Andrew LaMarche
date: 2024-01-03
categories: [linux, hacking]
tags: [linux, hacking]
---

As always, use the resources provided here responsibly. I am in no way, shape or form responsible for what you do with this.

Having hacked/rooted many embedded devices, I've found a gaping hole in my tool belt; it's near impossible to find and "hacker" tools that can run directly on these devices. Most of these run on outdated kernels, with glibs or uclibc from 2013 or earlier and had firmware created using some proprietary toolchain/SDK that will only run on Ubuntu 2012 (hint Arris/Commscope/Vantiva). It was a huge pain and when I needed to launch an nmap scan directly from a satellite modem, I couldn't! That's why I created the [Ubiquity Toolkit](https://github.com/andrewjlamarhe/ubiquity-toolkit).

Source: [https://github.com/andrewjlamarche/ubiquity-toolkit](https://github.com/andrewjlamarche/ubiquity-toolkit)

The toolkit consists of many super useful tools that have been fully statically linked. I put emphasis on fully. To ship a binary and have it run on any linux system, we must include the application's compiled code and also any supporting libraries all together. Otherwise, it may try to load a shared library from the running system which may not exist, or may not be compatible. This is achieved by using [musl-libc](https://musl.libc.org) over the standard [glibc](https://www.gnu.org/software/libc/).

# Why musl-libc over glibc?
For a more in-depth answer, check out this [Stack Overflow](https://stackoverflow.com/questions/57476533/why-is-statically-linking-glibc-discouraged/57478728#57478728) link. Otherwise, the tl;dr answer is glibc makes extensive use of `dlopen()`, which loads a shared library. This defeats the purpose of statically linking, since we want EVERYTHING to be included in the binary. musl does not have these same limitations.

Rich Felker, the author and maintainer of musl-libc, also provides an awesome and easy to use repository to build musl cross compilers called [musl-cross-make](https://github.com/richfelker/musl-cross-make). For example, if I wanted to build an arm big endian cross compiler, I can just run:

```bash
git clone https://github.com/richfelker/musl-cross-make
cd musl-cross-make
make -j $(nproc) TARGET=armeb-linux-musleabi install
```

Just add `output/bin` to your path and you can call the compiler directly.

# MIPS? ARM big endian? Who uses these any more?
Many communications devices use big endian and [network byte order](https://www.ibm.com/docs/fi/zos/2.4.0?topic=hosts-network-byte-order) is also big endian. Network byte order is the widely agreed upon byte order for network traffic to be in. By having the communications device also be big endian, there is no need to waste CPU cycles switching data from big to little endian.

As an example, MaxLinear, maker of the Puma 7 DOCSIS chips, who previously purchased Intel's Puma 6 intellectual property, who previously purchased this formerly Puma 5 IP from Texas Instruments, uses a dual-cpu design, where the networking processor is an armv6 big endian core. Thus, the `armeb-linux-musleabi` toolchain creates binaries that can run directly on cable modems.

# So what exactly makes up the Ubiquity Toolkit?
In its repository form, the Ubiquity Toolkit is just a collection of Makefiles that take care of downloading a tools and support libraries, configuring them, applying necessary patches, and then building them, while linking to any necessary libraries. The output is a fully statically linked and stripped down binary (for size) that should run on any linux system.

Pre-compiled versions are available here: [https://cantcrack.me/binaries/](https://cantcrack.me/binaries/) but don't expect them to be updated very frequently as stuff breaks with each version and it's a massive pain to figure out what each time.

Here's a quick look at the Busybox directory:

```
busybox-arm32:   ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), statically linked, stripped
busybox-arm32eb: ELF 32-bit MSB executable, ARM, EABI5 version 1 (SYSV), statically linked, stripped
busybox-arm64:   ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), statically linked, stripped
busybox-mips32:  ELF 32-bit MSB executable, MIPS, MIPS-I version 1 (SYSV), statically linked, stripped
busybox-mips64:  ELF 64-bit MSB executable, MIPS, MIPS-III version 1 (SYSV), statically linked, stripped
busybox-x64:     ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, stripped
busybox-x86:     ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), statically linked, stripped
```

