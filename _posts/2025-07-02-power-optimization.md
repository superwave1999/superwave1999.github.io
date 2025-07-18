---
title: "Tuning Linux Power Consumption with ASPM, C-States, and the ASM1064 SATA Controller"
date: 2025-07-02
tags: [linux, power-saving, zfs, pcie, aspm, c-states, asm1064, bios]
categories: [documentation]
media_subpath: /assets/posts/2025-07-02-power-optimization
---

# The goal and my approach

Given this is a system that will be powered on quite some time, I want it to sip the minimum required power at any given moment, but without it being too aggressve as to offline my ZFS drives or anything dramatic.

Now, I want to highlight the sources ([1](https://winraid.level1techs.com/t/latest-firmware-for-asm1064-1166-sata-controllers/98543/45), [2](https://z8.re/blog/aspm.html), [3](https://forums.truenas.com/t/power-efficient-truenas-with-asm1166-sata-controller/21498/7)) of my newly-acquired knowledge around tuning ASPM and CPU C-States.

Additionally, this is a good moment to check if my ASM1064 expansion has a good firmware that supports all interesting PCI-e features. Older firmwares may not offer full support.

If you've read posts about firmware updates and flashing the ASM1064, spoiler, it seems like I don't have to in my case as we will confirm below.

## ASM1064 checks

Let's find our SATA controllerby running sudo lspci -vv and scrolling down to where it says 02:00.0 SATA controller [0106]: ASMedia Technology Inc. ASM1064 Serial ATA Controller [1b21:1064] (rev 02).

In the output we can see the following:

MSI is enabled.

```text
MSI: Enable+ Count=1/1 Maskable- 64bit+
```

Relaxed Ordering, Extended Tags, No Snoop enabled.

```text
DevCap: MaxPayload 256 bytes ...
DevCtl: CorrErr+ NonFatalErr+ FatalErr+ UnsupReq+
        RlxdOrd+ ExtTag+ ...
```

But somehow, I don't see the following telltale sign of ASPM being enabled:

```text
LnkCtl: ASPM L1 Enabled
```

On the kernel level, everything is at the defaults:

```bash
cat /sys/module/pcie_aspm/parameters/policy
```

```text
[default] performance powersave powersupersave
```

I'm confident that the card will have no issues with ASPM once enabled.

## BIOS

Since I suspect something is off, let's delve into our BIOS and see what we can tune.

The only important change that I had to make was enabling native ASPM, but here is a breakdown of my current settings (most came set by default):

![BIOS 1](/bios-1.jpg)
![BIOS 2](/bios-2.jpg)

Note the C-State set to Auto, if I'm not happy with the result, I'll probably change this later.

Additionally, I set the case fan to "Silent" since it's the HDD cooling fan and even in a hot Spanish summer (32ºC indoors) the HDDs sit at 41-43ºC, the CPU temp maxes out at 58ºC full-load with a passive heatsink.

## PCI re-check and powertop

As expected, my ASM1064 now shows ASPM support confirming that it's not a dud.

```text
LnkCtl: ASPM L1 Enabled
```

With the very-important SATA controller out of the way, let's check the ASPM state of my other PCI devices (abbreviated):

```bash
sudo lspci -vvv | grep "ASPM .*abled"
```

Great! Everything has ASPM enabled.

```text
                LnkCtl: ASPM L0s L1 Enabled; RCB 64 bytes, Disabled- CommClk-
                LnkCtl: ASPM L1 Enabled; RCB 64 bytes, Disabled- CommClk+
                LnkCtl: ASPM L1 Enabled; RCB 64 bytes, Disabled- CommClk+
                LnkCtl: ASPM L1 Enabled; RCB 64 bytes, Disabled- CommClk+
                LnkCtl: ASPM L1 Enabled; RCB 64 bytes, Disabled- CommClk+
                LnkCtl: ASPM L1 Enabled; RCB 64 bytes, Disabled- CommClk+
                LnkCtl: ASPM L1 Enabled; RCB 64 bytes, Disabled- CommClk+
```

Now that the system is prepared for power savings, let's check what powertop says (apt install powertop):

```text
           Pkg(HW)  |            Core(HW) |            CPU(OS) 0
                    |                     | C0 active  16.7%
                    |                     | POLL        0.0%    0.0 ms
                    |                     | C1E         4.3%    0.2 ms
C2 (pc2)    8.5%    |                     |
C3 (pc3)   45.4%    | C3 (cc3)    0.0%    |
C6 (pc6)    0.0%    | C6 (cc6)   76.8%    | C6          2.5%    0.8 ms
C7 (pc7)    0.0%    | C7 (cc7)   74.8%    |
C8 (pc8)    0.0%    |                     | C8          3.9%    1.8 ms
C9 (pc9)    0.0%    |                     |
C10 (pc10)  0.0%    |                     |
                    |                     | C10        73.1%    6.8 ms
```

We can see that the CPU Package never drops below C3, this can be improved!

## Applying improvements

So we will now apply further changes to our system, with the following in mind:

- ZFS is sensitive to drive response times and is eager to offline disks.
- These changes will increase latency if the device was asleep. Usually less than 1 second observed with my UPS.
- Powertop --auto-tune on boot isn't always the best option.
- As always for your case YMMV!

The basic approach:

- Wake-on-LAN/USB/PCI OFF!
- USB autosuspend ON
- All PCI devices will have power optimizations (ASPM) enabled.

So, I wrote a script that runs on boot and a bunch of utilities. (TODO: Link to script)

The result is that we are reaching deeper C-states on everything but the package (PKG), the only remaining tune is in the BIOS, as suspected.

## Final BIOS tweaks

So, after applying the BIOS change (explicitly allowing down to C10) and setting up my script to run on boot, sadly, there isn't much of a safe improvement to my C-States.

I YOLOd' it and used powertop --auto-tune, and although it was initially successful, ZFS started having issues a few minute later, so I'll ignore these suggestions since storage integrity is crucial.

Final powertop idle stats:

```text
           Pkg(HW)  |            Core(HW) |            CPU(OS) 0
                    |                     | C0 active   0.5%
                    |                     | POLL        0.0%    0.0 ms
                    |                     | C1E         0.1%    0.3 ms
C2 (pc2)    2.0%    |                     |
C3 (pc3)   93.5%    | C3 (cc3)    0.0%    |
C6 (pc6)    0.0%    | C6 (cc6)   99.2%    | C6          1.7%    0.9 ms
C7 (pc7)    0.0%    | C7 (cc7)   97.5%    |
C8 (pc8)    0.0%    |                     | C8          0.0%    1.3 ms
C9 (pc9)    0.0%    |                     |
C10 (pc10)  0.0%    |                     |
                    |                     | C10        97.6%   56.1 ms
```

PCIE power modes:

```text
0000:00:00.0         auto       00:00.0 Host bridge: Intel Corporation Device 461c
0000:00:02.0         auto       00:02.0 VGA compatible controller: Intel Corporation Alder Lake-N [UHD Graphics]
0000:00:0a.0         auto       00:0a.0 Signal processing controller: Intel Corporation Platform Monitoring Technology (rev 01)
0000:00:14.0         auto       00:14.0 USB controller: Intel Corporation Alder Lake-N PCH USB 3.2 xHCI Host Controller
0000:00:14.2         auto       00:14.2 RAM memory: Intel Corporation Alder Lake-N PCH Shared SRAM
0000:00:15.0         auto       00:15.0 Serial bus controller: Intel Corporation Device 54e8
0000:00:16.0         auto       00:16.0 Communication controller: Intel Corporation Alder Lake-N PCH HECI Controller
0000:00:17.0         auto       00:17.0 SATA controller: Intel Corporation Device 54d3
0000:00:1c.0         auto       00:1c.0 PCI bridge: Intel Corporation Device 54b8
0000:00:1c.3         auto       00:1c.3 PCI bridge: Intel Corporation Device 54bb
0000:00:1c.6         auto       00:1c.6 PCI bridge: Intel Corporation Device 54be
0000:00:1d.0         auto       00:1d.0 PCI bridge: Intel Corporation Device 54b0
0000:00:1f.0         auto       00:1f.0 ISA bridge: Intel Corporation Alder Lake-N PCH eSPI Controller
0000:00:1f.3         auto       00:1f.3 Audio device: Intel Corporation Alder Lake-N PCH High Definition Audio Controller
0000:00:1f.4         auto       00:1f.4 SMBus: Intel Corporation Device 54a3
0000:00:1f.5         auto       00:1f.5 Serial bus controller: Intel Corporation Device 54a4
0000:02:00.0         auto       02:00.0 SATA controller: ASMedia Technology Inc. ASM1064 Serial ATA Controller (rev 02)
0000:03:00.0         auto       03:00.0 Ethernet controller: Realtek Semiconductor Co., Ltd. RTL8111/8168/8211/8411 PCI Express Gigabit Ethernet Controller (rev 15)
0000:04:00.0         auto       04:00.0 Non-Volatile memory controller: Samsung Electronics Co Ltd NVMe SSD Controller SM981/PM981/PM983
```

Wall power consumption (CPU transcoding, 100% usage): __52W__

Better than my initial calculations, even if we can't drop below C3. During early-boot with no power management active and all HDDs spinning up, the consumption can go up to __105W__ momentarily.
