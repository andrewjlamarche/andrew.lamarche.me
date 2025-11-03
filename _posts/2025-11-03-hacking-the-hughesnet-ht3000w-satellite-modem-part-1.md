---
title: Hacking the HughesNet HT3000W Satellite Modem | Part 1
author: Andrew LaMarche
date: 2025-11-03
categories: [linux, satcom, hacking]
tags: [linux, satcom, hacking]
---

> ##### DISCLAIMER
>
> DISCLAIMER: This post reflects myself and only myself in its entirety. It is in no way related to or sponsored by my employer or anyone else. This is a followup to [Hacking the HughesNet HT2000W Satellite Modem](https://andrew.lamarche.me/posts/hacking-the-hughesnet-ht2000w-satellite-modem/). As of November 03, 2025, I have still not heard from Hughes' security team.
{: .block-warning }

The HT3000W was released to support Hughes' Jupiter 3 satellite, which provides their 6th generation of service up to 100mbit/s download to customers. This device serves the exact same functionality as the HT2000W and has the same dual-PCB design - a standalone modem PCB and a standalone router PCB connected via an unfamiliar connector. The router is an 802.11ax (Wi-Fi 6) AX1800 class, meaning both 2.4GHz and 5GHz each have 2 spatial streams, and is based on Qualcomm's IPQ6018 CPU. There are plenty of other hardware and software differences between gen 6 and gen 5 devices that we will dig into.

# The Modem

Here's a look at the modem's PCB. Immediately, we notice there is a 3-pin unpopulated header above the CPU and a third chip next to the two RAM chips.

![ht3000w-top](/assets/images/ht3000w/ht3000w-top.jpg)

On the back side, there is no chip that looks like a flash.

![ht3000w-bottom](/assets/images/ht3000w/ht3000w-bottom.jpg)

This must mean the third chip on the top is a flash chip, but it doesn't look big enough to be an eMMC. It is clearly marked a [Spansion MS04G200BHI00](https://rocelec.widen.net/view/pdf/gbix3cm5uq/FASLS03878-1.pdf?t.download=true&u=5oefqw), which is a 4gbit (512MB) NAND flash in a BGA63 package... yikes. There are also 16 test pads above the chip but we will investigate those later. Let's probe the suspected UART header. Using our multimeter in continuity mode, we can touch the black probe to any grounded area, such as an RF shield, and the red probe to each of the pads. The pad opposite the square is ground, indicating the other two pads are likely TX/RX. However, the they stay at 0v when the modem is powered on, indicating UART is disabled somewhere. As any reasonable hardware hacker would do, I decided to take the destructive route.

## Dumping the NAND and Dealing with ECC/OOB Data

Some flux, solder, isopropyl alcohol, q-tips/makeup pads, a hot air station, a soldering iron, some soldering wick and a pair of tweezers are _"all"_ that's needed to lift and clean the NAND.

![ht3000w-nand-top](/assets/images/ht3000w/ht3000w-nand-top.jpg)

![ht3000w-nand-bottom](/assets/images/ht3000w/ht3000w-nand-bottom.jpg)

While we have the NAND off, let's see if the test pads connect back to anything useful. The datasheet gives us a pinout:

![ht3000w-nand-pinout-datasheet](/assets/images/ht3000w/ht3000w-nand-pinout-datasheet.jpg)

And we can use a multimeter in continuity mode to trace the test pads to the NAND pads:

![ht3000w-nand-pinout](/assets/images/ht3000w/ht3000w-nand-pinout.jpg)

The [XGecu T48 Programmer](https://www.aliexpress.us/item/3256804233005308.html?gatewayAdapt=glo2usa4itemAdapt) and a [T48-specific BGA64 Adapter](https://www.aliexpress.us/item/3256804481685981.html?gatewayAdapt=glo2usa4itemAdapt) are a great-priced duo that can dump this chip. [Xgpro](http://www.xgecu.com/download.html), while utterly sus, supports this chip as well. If you want to use open source software, [Minipro](https://gitlab.com/DavidGriffith/minipro) may work. Let's have another look at the [datasheet](https://rocelec.widen.net/view/pdf/gbix3cm5uq/FASLS03878-1.pdf?t.download=true&u=5oefqw) again. In case you don't know, NAND chips are prone to errors, and thus Error Correcting Code (ECC) or Out Of Bounds (OOB) data is needed to correct any errors. Page 3 of the datasheet shows two configurations for the 4Gbit chip, x8 and x16. The x8 (8 data lines) has a 2048-byte page size with 128-byte OOB/ECC data, while the x16 (16 data lines) has a 1024-byte page size with 64-byte OOB/ECC data. A good way to figure out how our chip is configured is to look for a long string of text. 

![ht3000w-oob-identify](/assets/images/ht3000w/ht3000w-oob-identify.jpg)

Here, there are 2176 bytes of data highlighted. We can pretty safely assume that this is an x8 chip since the first 1024 + 64 bytes of data appear normal; no random OOB/ECC data. Moving on, you may notice the last 128 bytes of data appear to be OOB (the big chunk starting with `0xFF`...). However, after 32 `0xF` bytes, there is normal text again... 24 bytes of it, followed by another 72 bytes of OOB data. But if the first 2048 bytes are valid data and another 24 bytes that are valid data, then there will be more data than this chip has actual capacity. A closer look at the first 2048 bytes shows 8 bytes of OOB data after the first 512-byte chunk: `67CFFDCB 2DD10C00`. Following along, for each 2176-byte chunk (2048 + 128), here's what we discover:

- 512 valid bytes
- 8 OOB bytes
- 512 valid bytes
- 8 OOB bytes
- 512 valid bytes
- 8 OOB bytes
- 488 valid bytes
- 32 OOB bytes
- 24 valid bytes
- 8 OOB bytes (488 + 24)
- 64 OOB bytes

A quick python script can remove this OOB data for us. It assumes `ht3000w-oob.bin` exists in the same directory and will write out `ht3000w.bin` with the OOB data removed.

```python
#!/usr/bin/env python3
def remove_oob(input_file, output_file):
    try:
        with open(input_file, 'rb') as infile, open(output_file, 'wb') as outfile:
            while True:
                chunk = infile.read(512)
                if not chunk:
                    break
                outfile.write(chunk)

                infile.seek(8, 1)
                chunk = infile.read(512)
                outfile.write(chunk)
                infile.seek(8, 1)
                chunk = infile.read(512)
                outfile.write(chunk)
                infile.seek(8, 1)
                chunk = infile.read(488)
                outfile.write(chunk)
                infile.seek(32, 1)
                chunk = infile.read(24)
                outfile.write(chunk)
                infile.seek(72, 1)

    except FileNotFoundError:
        print(f"Error: File '{input_file}' not found.")
    except Exception as e:
        print(f"An error occurred: {e}")

remove_oob('ht3000w-oob.bin', 'ht3000w.bin')
```

## Splitting the NAND

Splitting the NAND is useful for separting baremetal binaries (bootloaders, etc.) from filesystems. Let's try to split ours. Running `strings` on the new binary shows that the device runs U-Boot 2019. Opening the binary in a hex editor shows the `mtdparts` cmdline arg passed into the kernel. There are two `mtdparts` lines, though only one has partitions that add up to the full 512MB: `mtdparts=mtdparts=flash:1M(sbl),2M(u-boot),1M(factory),60M(fallback),448M(fl0)`. Therefore, there are 5 partitions:

- mtd0 (1MB):   sbl (second stage bootloader)
- mtd1 (2MB):   u-boot (final stage bootloader)
- mtd2 (1MB):   factory (factory configuration data)
- mtd3 (60MB):  fallback/fallback (initramfs + rootfs)
- mtd4 (448MB): fl0 (filesystem 0)

Therefore, we can come up with the following partition map:

```
sbl:		0x00000000 - 0x00100000 (1MB)
uboot:		0x00100000 - 0x00300000 (2MB)
factory:	0x00300000 - 0x00400000 (1MB)
fallback:	0x00400000 - 0x04000000 (60MB)
fl0:		0x04000000 - 0x20000000 (448MB)
```

Now, we can use `dd` to carve out each one (very, very slow):

```
dd if=ht3000w.bin bs=$((0x400)) count=$((0x400)) of=ht3000w-sbl.bin
dd if=ht3000w.bin bs=$((0x400)) skip=$((0x400)) count=$((0x800)) of=ht3000w-uboot.bin
dd if=ht3000w.bin bs=$((0x400)) skip=$((0xc00)) count=$((0x400)) of=ht3000w-factory.bin
dd if=ht3000w.bin bs=$((0x400)) skip=$((0x1000)) count=$((0xf000)) of=ht3000w-fallback.bin
dd if=ht3000w.bin bs=$((0x400)) skip=$((0x10000)) of=ht3000w-fl0.bin
```

## Examining the fallback and fl0 Partitions

The `fallback` and `fl0` partitions are UBI images:

```
ht3000w-fallback.bin:	UBI image, version 1
ht3000w-fl0.bin:		UBI image, version 1
```

There are 2 ways to extract the files in these partitions: [ubireader](https://github.com/onekey-sec/ubi_reader) and [nandsim](https://manpages.ubuntu.com/manpages/bionic/man4/nandsim.4freebsd.html).

### Extracting with ubireader

Let's use ubireader to exact the `fallback` and `fl0` partitions. `fallback` looks like a kernel and an initramfs.

```
ubireader_extract_files ht3000w-fallback.bin
Extracting files to: ubifs-root/1800224575/fallback
```

```
ls -laR ubifs-root/1800224575/fallback
ubifs-root/1800224575/fallback:
total 16
drwxrwxr-x 3 alamarche alamarche 4096 Jan 16 02:58 .
drwxrwxr-x 4 alamarche alamarche 4096 Jan 16 02:59 ..
drwxrwxr-x 2 alamarche alamarche 4096 Jan  1  1970 boot
-rw-rw-r-- 1 alamarche alamarche 1834 Jan  1  1970 uEnv.txt

ubifs-root/1800224575/fallback/boot:
total 36804
drwxrwxr-x 2 alamarche alamarche     4096 Jan  1  1970 .
drwxrwxr-x 3 alamarche alamarche     4096 Jan 16 02:58 ..
-rw-rw-r-- 1 alamarche alamarche  4421828 Jan  1  1970 Image.gz
-rw-rw-r-- 1 alamarche alamarche      256 Jan  1  1970 Image.gz.sig
-rw-rw-r-- 1 alamarche alamarche 13080456 Jan  1  1970 initramfs_data.cpio.xz
-rw-rw-r-- 1 alamarche alamarche      256 Jan  1  1970 initramfs_data.cpio.xz.sig
-rw-rw-r-- 1 alamarche alamarche     7566 Jan  1  1970 j3_3050.dtb
-rw-rw-r-- 1 alamarche alamarche     7593 Jan  1  1970 j3_cons.dtb
lrwxrwxrwx 1 alamarche alamarche       10 Jan 16 02:58 j3.dtb -> j3_uni.dtb
-rw-rw-r-- 1 alamarche alamarche     7674 Jan  1  1970 j3_dvt.dtb
-rw-rw-r-- 1 alamarche alamarche     7597 Jan  1  1970 j3_ent.dtb
-rw-rw-r-- 1 alamarche alamarche     7498 Jan  1  1970 j3_eval.dtb
-rw-rw-r-- 1 alamarche alamarche     7630 Jan  1  1970 j3_moca.dtb
-rw-rw-r-- 1 alamarche alamarche     7802 Jan  1  1970 j3_uni_0.dtb
-rw-rw-r-- 1 alamarche alamarche     7798 Jan  1  1970 j3_uni_1.dtb
-rw-rw-r-- 1 alamarche alamarche     7798 Jan  1  1970 j3_uni_2.dtb
-rw-rw-r-- 1 alamarche alamarche     7798 Jan  1  1970 j3_uni.dtb
-rw-rw-r-- 1 alamarche alamarche     7470 Jan  1  1970 j3_wifi.dtb
-rw-rw-r-- 1 alamarche alamarche 20070910 Jan  1  1970 main.bin
```

`fl0` is a filesystem with things related to the VSAT operations.

```
ubireader_extract_files ht3000w-fl0.bin 
Extracting files to: ubifs-root/1800224575/fl0
```

```
ls ubifs-root/1800224575/fl0
adhoc.json  config        key_burn.log  newODU.dat      rptResetCnt    startup_file                    stats      wifi_fallback
aeroLogs    ddnsc_cfg     light         oduinfo.txt     sbc.cfg        statecode_monitor_previous.txt  stats.cfg  wifi_mode.dat
apps        etc           loginfo.dat   patsbc.cfg      sbc_state.cfg  state_info.dat                  sys.log    wifi_ub_upd.sh
bin         evtlogs       logrt.conf    pid_update.log  sbc_stats.dat  state_info_er3.dat              sys.pipe   x.log
boot        htr_mode.dat  logs          ranging.dat     sdl            state_info_er6.dat              var
cimcfg.a    install.dat   main.dat      reset.csv       ssh            state_info.log                  wifi.dat
```

### Extracting with nandsim

Perhaps a more accurate title would be "Simulating with nandsim", since we're not really "extracting" anything. Instead, we are creating a simulated NAND device, writing the data each partition and mounting it. Let's create an mtd device with 5 partitions.

```
sudo modprobe nandsim id_bytes=0x01,0xac,0x90,0x15,0x56 parts=0x8,0x10,0x8,0x1e0
``` 

The `id_bytes` are derived from page 31 of the datasheet:
- byte 1: manufacturer code
- byte 2: device identifier
- byte 3: internal chip number
- byte 4: page size, block size, spare size, serial access time and origanization
- byte 5: ECC and multiplane data

The `parts` option specifies the size of each partition. This chip's eraseblock size is 128kb (also found in the datasheet), and since we know the size of each partition from the `mtdparts` string in U-Boot, we can specify the partition size by dividing the partition size (1024kb / 128kb) = `0x08`.

Next, we can write the extracted partition data to each partition:

```
sudo nandwrite /dev/mtd0 ht3000w-sbl.bin
sudo nandwrite /dev/mtd1 ht3000w-uboot.bin
sudo nandwrite /dev/mtd2 ht3000w-factory.bin
sudo nandwrite /dev/mtd3 ht3000w-fallback.bin
sudo nandwrite /dev/mtd4 ht3000w-fl0.bin
```

Then, we can mount each UBI partition:

```
sudo mkdir /mnt/fallback /mnt/fl0
sudo modprobe ubi mtd=/dev/mtd3,2048 mtd=/dev/mtd4,2048
sudo mount -t ubifs /dev/ubi0_0 /mnt/fallback
sudo mount -t ubifs /dev/ubi1_1 /mnt/fl0
```

The contents of each can now be viewed in `/mnt/fallback` and `/mnt/fl0`. Inside `/mnt/fallback/boot`, there are several device trees. The markings on the RAM chip suggest each is 256MB for a total of 512MB. Let's start decompiling these device trees to see which one likely matches:

```
dtc -I dtb -O dts j3_uni_0.dtb > j3_uni_0.dts
``` 

`model = "Hughes Jupiter 3 Unified Device Tree - 512MB DDR";`. Looks like a hit! Heres the full device tree:

<details markdown="1">
<summary>HughesNet Jupiter 3 Unified Device Tree</summary>

```
/dts-v1/;

/ {
	compatible = "hughes,j3_uni";
	interrupt-parent = <0x01>;
	#address-cells = <0x02>;
	#size-cells = <0x02>;
	model = "Hughes Jupiter 3 Unified Device Tree - 512MB DDR";

	cpus {
		#address-cells = <0x01>;
		#size-cells = <0x00>;

		cpu@0 {
			device_type = "cpu";
			compatible = "arm,cortex-a53";
			next-level-cache = <0x02>;
			enable-method = "spin-table";
			cpu-release-addr = <0x00 0xa000000>;
			reg = <0x00>;
			linux,phandle = <0x03>;
			phandle = <0x03>;
		};

		cpu@1 {
			device_type = "cpu";
			compatible = "arm,cortex-a53";
			next-level-cache = <0x02>;
			enable-method = "spin-table";
			cpu-release-addr = <0x00 0xa000008>;
			reg = <0x01>;
			linux,phandle = <0x04>;
			phandle = <0x04>;
		};

		cpu@2 {
			device_type = "cpu";
			compatible = "arm,cortex-a53";
			next-level-cache = <0x02>;
			enable-method = "spin-table";
			cpu-release-addr = <0x00 0xa000010>;
			reg = <0x02>;
			linux,phandle = <0x05>;
			phandle = <0x05>;
		};

		cpu@3 {
			device_type = "cpu";
			compatible = "arm,cortex-a53";
			next-level-cache = <0x02>;
			enable-method = "spin-table";
			cpu-release-addr = <0x00 0xa000018>;
			reg = <0x03>;
			linux,phandle = <0x06>;
			phandle = <0x06>;
		};
	};

	l2-cache@0 {
		compatible = "cache";
		cache-level = <0x02>;
		linux,phandle = <0x02>;
		phandle = <0x02>;
	};

	timer {
		compatible = "arm,armv8-timer";
		interrupts = <0x01 0x0d 0x08 0x01 0x0e 0x08 0x01 0x0b 0x08 0x01 0x0a 0x08>;
	};

	pmu {
		compatible = "arm,cortex-a53-pmu\0arm,armv8-pmuv3";
		interrupts = <0x00 0x3c 0x04 0x00 0x3d 0x04 0x00 0x3e 0x04 0x00 0x3f 0x04>;
		interrupt-affinity = <0x03 0x04 0x05 0x06>;
	};

	interrupt-controller@0e000000 {
		compatible = "arm,gic-400";
		#interrupt-cells = <0x03>;
		interrupt-controller;
		reg = <0x00 0xe001000 0x00 0x1000 0x00 0xe002000 0x00 0x2000>;
		clocks = <0x07>;
		linux,phandle = <0x01>;
		phandle = <0x01>;
	};

	pcs@05b80000 {
		compatible = "hughes,pcs";
		reg = <0x00 0x5a40000 0x00 0x40000 0x00 0x5b80000 0x00 0x1000>;
		interrupts = <0x00 0x18 0x04>;
	};

	wdt@05a80000 {
		compatible = "hughes,wdt";
		reg = <0x00 0x5a80000 0x00 0x10>;
		interrupts = <0x00 0x34 0x04>;
	};

	clocktree {

		osc40mhz {
			compatible = "fixed-clock";
			#clock-cells = <0x00>;
			clock-frequency = <0x2625a00>;
			clock-accuracy = <0x64>;
			linux,phandle = <0x08>;
			phandle = <0x08>;
		};

		pll1ghz {
			compatible = "fixed-factor-clock";
			clocks = <0x08>;
			#clock-cells = <0x00>;
			clock-mult = <0x19>;
			clock-div = <0x01>;
			linux,phandle = <0x09>;
			phandle = <0x09>;
		};

		pllbusprogdiv {
			compatible = "hns,clk_by_pcs_mode\0fixed-factor-clock";
			clocks = <0x09>;
			#clock-cells = <0x00>;
			clock-div = <0x02>;
			clock-mult = <0x01>;
			linux,phandle = <0x07>;
			phandle = <0x07>;
		};

		divprogdiv2 {
			compatible = "fixed-clock";
			#clock-cells = <0x00>;
			clock-frequency = <0xee6b280>;
			clock-accuracy = <0x64>;
			linux,phandle = <0x0a>;
			phandle = <0x0a>;
		};

		pllbusdiv10 {
			compatible = "fixed-factor-clock";
			clocks = <0x09>;
			#clock-cells = <0x00>;
			clock-div = <0x0a>;
			clock-mult = <0x01>;
		};

		pllbusdiv100 {
			compatible = "fixed-factor-clock";
			clocks = <0x09>;
			#clock-cells = <0x00>;
			clock-div = <0x64>;
			clock-mult = <0x01>;
		};

		pllbusdiv250 {
			compatible = "fixed-factor-clock";
			clocks = <0x09>;
			#clock-cells = <0x00>;
			clock-div = <0xfa>;
			clock-mult = <0x01>;
		};

		pllbusdiv4 {
			compatible = "fixed-factor-clock";
			clocks = <0x09>;
			#clock-cells = <0x00>;
			clock-div = <0x04>;
			clock-mult = <0x01>;
		};
	};

	soc@c000000 {
		compatible = "simple-bus";
		#address-cells = <0x02>;
		#size-cells = <0x02>;
		ranges;

		uart@c000000 {
			compatible = "snps,dw-apb-uart";
			reg = <0x00 0xc000000 0x00 0x1000>;
			reg-shift = <0x02>;
			clock-frequency = <0x1dcd6500>;
			interrupt-parent = <0x01>;
			interrupts = <0x00 0x13 0x04>;
		};

		uart@C001000 {
			compatible = "snps,dw-apb-uart";
			reg = <0x00 0xc001000 0x00 0x1000>;
			reg-shift = <0x02>;
			clock-frequency = <0x1dcd6500>;
			interrupt-parent = <0x01>;
			interrupts = <0x00 0x12 0x04>;
		};

		uart@C00A000 {
			compatible = "snps,dw-apb-uart";
			reg = <0x00 0xc00a000 0x00 0x1000>;
			reg-shift = <0x02>;
			clock-frequency = <0x1dcd6500>;
			interrupt-parent = <0x01>;
			interrupts = <0x00 0x11 0x04>;
			status = "disabled";
		};

		uart@14000000 {
			compatible = "snps,dw-apb-uart";
			reg = <0x00 0x14000000 0x00 0x1000>;
			reg-shift = <0x02>;
			clock-frequency = <0x1dcd6500>;
			interrupt-parent = <0x01>;
			interrupts = <0x00 0x19 0x04>;
			status = "disabled";
		};

		uart@1C000000 {
			compatible = "snps,dw-apb-uart";
			reg = <0x00 0x1c000000 0x00 0x1000>;
			reg-shift = <0x02>;
			clock-frequency = <0x1dcd6500>;
			interrupt-parent = <0x01>;
			interrupts = <0x00 0x1a 0x04>;
			status = "disabled";
		};

		uart@4000000 {
			compatible = "snps,dw-apb-uart";
			reg = <0x00 0x4000000 0x00 0x1000>;
			reg-shift = <0x02>;
			clock-frequency = <0x1dcd6500>;
			interrupt-parent = <0x01>;
			interrupts = <0x00 0x17 0x04>;
			status = "disabled";
		};

		gpios@0x0C009000 {
			compatible = "hughes,gpios";
			reg = <0x00 0xc009000 0x00 0x100 0x00 0x1c00a000 0x00 0x100 0x00 0x4009000 0x00 0x100>;
		};

		pwm@0x0C007000 {
			compatible = "hughes,pwm";
			reg = <0x00 0xc007000 0x00 0x20 0x00 0x1c008000 0x00 0x20 0x00 0x14008000 0x00 0x20>;
		};

		timer@0C005000 {
			compatible = "hughes,gpt";
			reg = <0x00 0xc005000 0x00 0x1000>;
			interrupts = <0x00 0x0d 0x04>;
			clock-frequency = <0xf4240>;
			clock-type = <0x00>;
		};

		timer@0C006000 {
			compatible = "hughes,gpt";
			reg = <0x00 0xc006000 0x00 0x1000>;
			interrupts = <0x00 0x0c 0x04>;
			clock-frequency = <0xf4240>;
			clock-type = <0x01>;
		};

		glue@f000000 {
			compatible = "hughes,swp-glue";
			reg = <0x00 0xf000000 0x00 0x600>;
		};

		eFuse@C800000 {
			compatible = "hughes,swp-eFuse";
			reg = <0x00 0xc800000 0x00 0x100200>;
		};

		usb2phy {
			compatible = "usb-nop-xceiv";
			linux,phandle = <0x0b>;
			phandle = <0x0b>;
		};

		usb3phy {
			compatible = "usb-nop-xceiv";
			linux,phandle = <0x0c>;
			phandle = <0x0c>;
		};

		usb@f400000 {
			compatible = "hns,dwc3";
			clocks = <0x0a>;
			clock-names = "core";
			#address-cells = <0x02>;
			#size-cells = <0x02>;
			speed-config = "usb2_int_clk";
			ranges;

			dwc3 {
				compatible = "snps,dwc3";
				reg = <0x00 0xf400000 0x00 0x10000>;
				interrupts = <0x00 0x30 0x01>;
				dr_mode = "host";
				usb-phy = <0x0b 0x0c>;
				phy_type = "utmi_wide";
			};
		};

		ipc@5200000 {
			compatible = "hughes,ipc";
			interrupts = <0x00 0x23 0x04 0x00 0x25 0x04>;
			dev = "dpp\0upp";
		};

		dlk@25100000 {
			compatible = "hughes,dlk";
			interrupts = <0x00 0x19 0x04>;
		};

		pcie@0x8030000000 {
			status = "okay";
			compatible = "snps,dw-pcie-ep";
			device_type = "pci";
			reg = <0x80 0x30000000 0x00 0x100000 0x80 0x30100000 0x00 0x100000 0x80 0x40000000 0x00 0x10000000>;
			reg-names = "dbi\0dbi2\0addr_space";
			num-ib-windows = <0x10>;
			num-ob-windows = <0x10>;
			num-lanes = <0x01>;
			interrupts = <0x00 0x2f 0x04>;
		};

		europa_nand@20000000 {
			compatible = "hughes,europa-nand";
			reg = <0x00 0x20000000 0x00 0x9000>;
			nand-ecc-mode = "hw";
			nand-ecc-algo = "bch";
			nand-ecc-step-size = <0x200>;
			nand-ecc-strength = <0x04>;

			partitions {
				compatible = "fixed-partitions";
				#address-cells = <0x01>;
				#size-cells = <0x01>;

				sbl@0 {
					label = "SBL";
					reg = <0x00 0x100000>;
				};

				u-boot@100000 {
					label = "u-boot";
					reg = <0x100000 0x200000>;
				};

				factory@300000 {
					label = "factory";
					reg = <0x300000 0x100000>;
				};

				fallback@400000 {
					label = "fallback";
					reg = <0x400000 0x3c00000>;
				};

				fl0@4000000 {
					label = "fl0";
					reg = <0x4000000 0x1c000000>;
				};
			};
		};
	};

	chosen {
	};

	memory {
		device_type = "memory";
		reg = <0x00 0x40000000 0x00 0x20000000>;
	};

	reserved-memory {
		#address-cells = <0x02>;
		#size-cells = <0x02>;
		ranges;

		swbuffer_elastic {
			no-map;
			reg = <0x00 0x5b300000 0x00 0x3780000>;
		};

		swbuffer_fixed {
			no-map;
			reg = <0x00 0x5ea80000 0x00 0xd80000>;
		};

		fwprivate {
			no-map;
			reg = <0x00 0x5f800000 0x00 0x800000>;
		};
	};

	ethernet@e400000 {
		#address-cells = <0x01>;
		#size-cells = <0x01>;
		compatible = "hns,dwmac-5.10a";
		reg = <0x00 0xe400000 0x00 0x2000>;
		interrupt-parent = <0x01>;
		interrupts = <0x00 0x2c 0x04>;
		interrupt-names = "macirq";
		mac-id = <0x01>;
		mac-address = [02 00 de ad be e0];
		phy-mode = "rgmii-id";
		max-speed = <0x3e8>;

		mdio_eth0@0x0e400000 {
			compatible = "hns,mdio";
			reg = <0xe400000 0x2000>;
			#address-cells = <0x01>;
			#size-cells = <0x00>;

			phy00@0 {
				reg = <0x00>;
			};

			phy01@1 {
				reg = <0x01>;
			};

			phy02@2 {
				reg = <0x02>;
			};

			phy03@3 {
				reg = <0x03>;
			};

			phy05@5 {
				reg = <0x05>;
			};
		};
	};

	ethernet@e800000 {
		#address-cells = <0x01>;
		#size-cells = <0x01>;
		status = "disabled";
		compatible = "hns,dwmac-5.10a";
		reg = <0x00 0xe800000 0x00 0x2000>;
		interrupt-parent = <0x01>;
		interrupts = <0x00 0x2d 0x04>;
		interrupt-names = "macirq";
		mac-id = <0x02>;
		mac-address = [02 00 de ad be e1];
		phy-mode = "rgmii-id";
		max-speed = <0x3e8>;

		mdio_eth1@0x0e400000 {
			compatible = "hns,mdio";
			reg = <0xe400000 0x2000>;
			#address-cells = <0x01>;
			#size-cells = <0x00>;

			phy10@0 {
				reg = <0x00>;
			};

			phy13@3 {
				reg = <0x03>;
			};
		};
	};
};
```
</details>

## Europa

The device tree tells us that the modem has 4 cortex-a53 cores, 6 UARTs, a GPIO and eFuses amongst other things. Oh, and the device's codename is "Europa". Europa is Jupiter's 4th largest moon, and is the 6th closest to the planet. It is also one of the most likely places in our solar system to contain life.

## Part 2: Coming Soon<sup>TM</sup>

Part 2 will come along eventually, once I figure out how to reconstruct the OOB data in order to modify flash contents on the NAND. I realise this may have some implications due to secure boot, but I may have some tricks to try.