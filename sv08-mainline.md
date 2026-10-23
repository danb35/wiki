---
title: Mainline Klipper on a Sovol SV08 in 2026
description: Notes on converting a Sovol SV08 printer to run Mainline Klipper
published: true
date: 2026-10-23T16:22:06.717Z
tags: 
editor: markdown
dateCreated: 2026-10-23T16:22:06.717Z
---

# Introduction
After buying a [Sovol SV08 3D printer](https://www.sovol3d.com/products/sovol-sv08-3d-printer), one of the first things I wanted to do was install mainline Klipper.  The canonical instructions for this process are at https://github.com/Rappetor/Sovol-SV08-Mainline.  However, time, hardware revisions, and bugs resulted in several changes to my installation, which I wanted to document for my own reference, as well as anyone else who may find it helpful.
# Base OS image
The Rappetor repo calls for use of the use of the BTT CB1 v2.3.4 image.  This is based on Debian 11 (Bullseye), which is now EOL.  An image of Armbian is available specifically for the SV08, which is based on Debian 13 (Trixie), the current release.  That can be downloaded from https://armbian.com/boards/sovol-sv08.

Once you write that image to your eMMC module (and I recommend purchasing a fresh module for this purpose), you'll need to make a few edits to the `armbianEnv.txt` file.  For my system, using a BTT HDMI5 display, I added the following to the beginning of that file:
```
verbosity=1
bootlogo=false
console=both
disp_mode=800x480p60
overlay_prefix=sun50i-h616
overlays=uart3-pi
fdtfile=sun50i-h616-sovol-sv08.dtb
overlays=ws2812
overlays=spidev1_1
```
The remaining lines (beginning with `rootdev=`) remain unchanged.
# Installation process
Install the eMMC module in your printer, plug in a keyboard and display, and connect to your network.  When you boot the system, run through the first-boot process: create a root password, set up a non-root user (I used `biqu`), set your locale, time zone, etc.  Run `apt update && apt upgrade` to install system updates, `apt install git mc`, then reboot.

At this point, you can unplug the keyboard and display, and do the remainder of the installation via SSH.

SSH to the printer, download KIAUH, and use it to install Klipper, Moonraker, and the rest of the components specified in Rappetor's repo.  Current (late 2026) KIAUH is entirely capable of installing the input shaper dependencies, so there's no reason to manually do so.

Also clone the `Sovol-SV08-Mainline` repo: `cd ~ && git clone https://github.com/Rappetor/Sovol-SV08-Mainline`.

Katapult didn't work on my printer, which uses a rev 1.3 mainboard.  This is a reported issue on Rappetor's repo ([#260](https://github.com/Rappetor/Sovol-SV08-Mainline/issues/260)).  So rather than flashing Katapult, I built the Klipper firmware, and flashed it to both the mainboard and the toolhead using the ST-Link.  Future updates to the firmware on those boards, if any, will again require using the ST-Link adapter; I won't be able to do them over USB as I could if Katapult worked.
