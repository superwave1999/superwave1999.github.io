---
title: "Building a New Home Server for all my Data"
date: 2025-06-14
tags:
  - self-hosting
  - zfs
  - hardware
  - linux
  - home-server
  - data-sovereignty
categories: [documentation]
media_subpath: /assets/posts/2025-06-14-setup-tour
---

# Why?

Hello world! I'm on a mission to regain sovereignty of my data. Thing is, I'm getting old, and I long for the days of simpler, non-spyware, non-adware tech where most of your data was stored locally on hard drive or CDs.

Over the course of my life, I've seen the progressive decay and misuse of tech, along with a population that over-prioritises commodity over function, sustainability and ownership.

We live in a society where people complain about rising Netflix prices, faulty hardware tied to online subscriptions (that also rises in price) and expensive "mid" media which can be deleted in the blink of an eye after the purchase.

All of this is either ignorance, laziness, lack of knowledge and/or valuing the iPhone for it's appearance instead of the possibilities the chinese-sweatshop-made device offers.

See: enshittification, Switch 2, standing-up seats on planes.

I'm here to bring solutions to some of these issues, so without further ado...

# State of the union

Years ago, in 2017, I began my journey to degoogle my life. I bought an MSI Cubi 3 Silent barebone with an i3-7100 with a Samsung 970 EVO. As the years went on, and my now-wife started actively using it too, storage ran out.

At the end of 2021 I decided to invest in storage, so I got 4x8TB Western Digital drives. The mini-PC obviously doesn't allow much expansion, so the issue of connectivity was solved by a QNAP TR-004 drive enclosure, which I left in JBOD mode and let RAID be software-controlled by ZFS.

I was fully aware that having the two pillars of the setup connected via a single wiggly cable was finicky, and that these pillars each had their own power supply was even more finicky.

Surprisingly enough, this setup lasted me all those years *until now*.

The only two minor issues I've had with the system are:
- A random unexplained offlining of a drive in the summer of '24. I checked the drive was physically OK, and the ZFS pool was rebuilt successfully.
- Suddenly, the mini-PC stopped fully powering-off when I pressed the button, or sent a poweroff command. I didn't upgrade the system, BIOS, firmware or changed any setting.

The big issue faces is the offlining of drives 2-3 (of 1-4). One kept the kernel device ID in ZFS, and the other only retained the ZFS ID. I checked the drives were OK by plugging them in to another PC and running tests. I also ruled-out the power supply. Although it had a very slight "ticking" sound, this didn't cause the issue. The USB cable was also ruled out. As for power delivery, both of these machines get power via an Eaton Ellpise ECO 650 UPS. The only outliers are the circuitry of the TR-004, or something wrong with the mini-PC. I plugged the TR-004 into another port and began the restoration of my data (more on that in a future post).

After a week, drive 3 disappeared again.

Given the age of the main system and the distrust of the TR-004, I convinced the wife to approve the budget of an upgrade.

# The new hardware

I have a plan and I have a budget (of 300-350€). What could be so complicated?

Avilability of the components.

Nonetheless, I managed to source all the components from a combination of Amazon, PCComponentes and CaseKing. Parts below.

## Case

I'm putting this section first since having a system with a small footprint and access to many drives is a high priority.

Narrowing it down to ITX-only form-factor with support for multiple drives, I found only two viable contenders:

### Fractal Design Node 304

This is the cheaper case of the two, and perfect if you're building a NAS / Gaming / Rendering hybrid machine due to it's full-length GPU support. The main cons for NAS usage is the fact that the drives are inside the case, and every odd drive is flipped 180º sideways, also introducing twists in the SATA power and data cables.

__94.48€__

### Jonsbo N2

It's clear to me that this was designed as a small-footprint storage-focused case from the start. It measures 22x22x22 and fits 5 drives with front access. The main limitations are SFX power supplies only, and no full-size GPU support.

But for my needs, it's perfect and is worth the price of __137€__.

## Motherboard / CPU considerations

Nothing flashy needed. Anything modern that performs on par or better than an i3-7100 (while using less power) will get a pass from me. Thankfully the Intel N100 exists. This CPU is very efficient, rated at 6W.

Browsing online for motherboards, I noticed that this CPU comes soldered onto two available motherboards:
- ASRock N100DC‑ITX (160€)
- **Asus PRIME N100I‑D** (113€, spoiler: my pick)

### Power
Coming from a system that has barrel-jacks and external laptop-charger-style bricks, I'm looking for something more standard. The ASRock uses a barrel-jack, and the Asus a standard 24-pin ATX connector.

1 point for the Asus.

### Expandability & I/O
The ASRock has more I/O out-of-the-box, and offers a single DDR4 DIMM RAM slot. Sadly, the Asus uses a laptop-style SODIMM. Additionally the ASRock has two SATA connectors on-board, but the Asus offers one.

Honestly, the RAM slot isn't a big issue, and paying notably more for an additional on-board SATA when I'm going to use a dedicated PCI-E to SATA expansion card anyway (for all the drives) makes no sense.

Both have M.2 SSD support in PCI-E mode only.

The rest of the connectivity features are very similar.

### Performance and thermals

Both of these boards have their CPU passively-cooled which could become a problem in warmer climates like mine.

Given my use-case of not having the cpu pinned at 100% for hours-on-end (but maybe an occasional HEVC re-encode) and that I don't need much more performance than what I've already got, the Asus is a better fit __for me__.

The ASRock runs the CPU at a higher TDP, producing more heat that I don't want.

### Price & recap

As stated, I have a budget and the Asus board fits right into the budget. Paying 50€ more for the disadvantage of the barrel-jack, the very minor advantage of an additional SATA and 20-30% more performance with the downside of heat doesn't seem the best of deals.

## RAM

I'm going to re-use the not-so-old **16GB DDR4 2133MHz SODIMM Corsair Value Select** RAM from my old system. No, It isn't ECC RAM. The possibility of bit-flips isn't zero, but it's very minor ahd I haven't seen issues with this yet. I'll take my chances.

## OS drive

I have a **Samsung 970 EVO** (M.2 PCI-E) lying around in good condition and treated well, so I'm going to promote it to be my OS drive.

## Storage drives

I'm bringing over my existing four **8TB WD Red Plus** drives from my old system. They should have a couple of years left in them.

I'm considering adding a fifth drive for more redundancy, but that isn't in the scope of this article.

## Expansion cards

The expansion slot on my board is a PCI-E 3.0 x1.

To power these 4 drives, I searched online for the opinions of the community. There's a split between multiple camps:
- HBA's are still good and reliable.
- HBA's are reliable, but you're buying older second-hand tech that may be unsupported by the manufacturer.
- Use a PCI-E to SATA card of a dubious manuracturer (this guy did successfully for now).

AFAIK, most HBA's are designed for x8 or x4 slots, not x1 slots. So given the possibilities and budget, I'm going for the last option.

On the positive side, it's still better than running the drives through USB.

My drives max out at around 200MB/s sequential. PCI-E 3.0 x1 maxes out at around 970 MB/s. Multiply. I'm well under the limit imposed by PCI-E.

So, I've settled for an **ASM1064-based expansion card** ([this one](https://amzn.eu/d/3y4jf53)) for all my drives.

About the potential fifth drive, I'll plug it into the board. For those sysadmins reading this and cringing at using two different controllers for the same RAID, I must remind you that I'm not Google, and I'm doing the best I can with the given budget and resources ;)

## Power supply

Note: The ASRock doesn't include the power supply.

Given the above parts, these are the full-load over-estimated consumption calculations. I'll measure exact numbers from the wall when it's built (update soon).

| Part                | Power (Heavy Load) |
|---------------------|--------------------|
| N100 + Board + RAM  | 18 to 28W          |
| WD80EFBX HDD        | 8.8W               |
| WD80EFBX HDD        | 8.8W               |
| WD80EFBX HDD        | 8.8W               |
| WD80EFBX HDD        | 8.8W               |
| Samsung 970 Evo SSD | 5W                 |
| ASM1064 PCI-E       | 2W                 |
| **TOTAL**           | **70.2W**          |


The estimates are a combination of:
- Manufacturer specs.
- Independent review data (AnandTech, Tom's Hardware).
- Real‑user measured results from Reddit or Level1Techs.

Given that I wont exceed 100W, and probably sit around 20-30W most of the time, I'm in the search for a low-power power supply from a reputable brand that has the required SFX form factor for my case.

Enter the **be quiet! SFX Power 3 300W** at __63.99€__.


## Conclusion

New rig, close enough to the upgrade budget (**360€**) and with a smaller footprint and power consumption than the old version. Gone are the days of split power between the system and the drives and USB cables to link them both up.

# Software stack

## Operating System

Given the winds of change, I also re-evaluated my operating system priorities. My previous build was running Ubuntu + Docker, and it seems like it's going to stay this way.

1. Linux.
2. Distro with fixed-releases with decent support.
3. Distro with a sizeable community.
4. Distro with some corporate backing.
5. Distro with advanced filesystem support. I'll be using either ZFS or BTRFS, not both.
6. Docker support.
7. If the distro has a Web-UI, it should be fully featured and support my use-case of advances permission management and containers with conencted volumes.

Based on the above, Arch and Fedora are immediately excluded. Ubuntu, Debian and Alma Linux remain.

### The Canonical affaire

To clarify point #4, for a distro to survive it needs corporate backing. I could release __Imanol Linux__ tomorrow with 10 years of support and disappear. In pure community-led distros infighting may occur (although large community distros suvive) unless someone calls the shots using the modern instrument of money to sway interests.

Given this, I'm ruling out Debian. Sorry.

But, if Ubuntu breaks things by moving too fast with their moves to Rusty coreutils, or sticks more ads in their system, I may consider Debian again.

### Alms for the poor

Why not try Alma? It ticks all the boxes.

Yes, it has more years of support, but has less community guides and Docker doesn't officially support it.

## Filesystem - BTRFS vs ZFS

Without further ado, here is a reviewed table generated by ChatGPT:

| Feature              | ZFS                                | BTRFS                                            |
| -------------------- | ---------------------------------- | ------------------------------------------------ |
| **Stability**        | Very stable, enterprise‑ready      | Stable for most use, RAID5/6 still experimental  |
| **Licensing**        | CDDL (out‑of‑tree)                 | GPL (in‑kernel)                                  |
| **RAM Usage**        | High (8GB+ recommended)            | Low‑to‑moderate                                  |
| **Best Use Case**    | Large pools, heavy I/O             | General use, desktop/laptop                      |
| **RAID Features**    | RAID‑Z, hot spares, scrubbing      | RAID0/1/10 okay, RAID5/6 risky                   |
| **Snapshot/Clone**   | Excellent, highly efficient        | Good, supported natively                         |
| **Self‑Healing**     | Yes (end‑to‑end checksums)         | Limited                                          |
| **Flexibility**      | Very robust, enterprise‑focused    | More flexible for common Linux setups            |
| **Ease of Recovery** | Complex                            | Simpler tooling available                        |
| **Best When**        | You value long‑term data integrity | You want seamless kernel support and general use |

I picked ZFS mainly due to it's maturity, stability, integrity checks, being supported by Canonical.

I also want to use ZFS for my boot drive, even if this means having to install alternative bootloaders. Grubless ZFS Ubuntu install guide coming soon.

## Summary

Below is a diagram of my containers and the connections between them. No planned change here either.

![Diagram](/software-stack.png)

# Final Conclusion

Happy with the result. A semi-professional setup at a good price. I'll be posting my journey using this new system to this page.

TODO: Add pic