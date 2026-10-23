---
title: Mainline Klipper on a Sovol SV08 in 2026
description: Notes on converting a Sovol SV08 printer to run Mainline Klipper
published: true
date: 2026-10-23T16:40:50.428Z
tags: 
editor: markdown
dateCreated: 2026-10-23T16:22:06.717Z
---

# Introduction
After buying a [Sovol SV08 3D printer](https://www.sovol3d.com/products/sovol-sv08-3d-printer), one of the first things I wanted to do was install mainline Klipper.  The canonical instructions for this process are at https://github.com/Rappetor/Sovol-SV08-Mainline.  However, time, hardware revisions, and bugs resulted in several changes to my installation, which I wanted to document for my own reference, as well as anyone else who may find it helpful.
# Before you begin
Ideally, back up your entire `/config` directory.  At a minimum, back up your `printer.cfg` file.  You can do this through the Mainsail web interface.

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
## Flash and first boot
Install the eMMC module in your printer, plug in a keyboard and display, and connect to your network.  When you boot the system, run through the first-boot process: create a root password, set up a non-root user (I used `biqu`), set your locale, time zone, etc.  Run `apt update && apt upgrade` to install system updates, `apt install git mc`, then reboot.

At this point, you can unplug the keyboard and display, and do the remainder of the installation via SSH.
## KIAUH
SSH to the printer as the user you created at first boot.  Then run `cd ~ && git clone https://github.com/dw-0/kiauh.git`.  This will install KIAUH, the Klipper Installation and Upgrade Helper, a script that you'll use to install much of the remaining software.  Also clone the `Sovol-SV08-Mainline` repo: `cd ~ && git clone https://github.com/Rappetor/Sovol-SV08-Mainline`.  You'll use this later to grab some configuration files.
## Klipper, Moonraker, etc.
Start KIAUH by running `./kiauh/kiauh.sh`.  Use it to install, in order, Klipper, Moonraker, Mainsail, Crowsnest, and (if you want it) Klipperscreen.  Using the Advanced menu option, install the additional dependencies for the Input Shaper.  Then reboot.
## Moonraker-timelapse
Install `moonraker-timelapse` by running:
```
cd ~/
git clone https://github.com/mainsail-crew/moonraker-timelapse.git
cd ~/moonraker-timelapse
make install
```
Then edit `~/printer_data/config/moonraker.conf` and add the following:
```
[update_manager timelapse]
type: git_repo
primary_branch: main
path: ~/moonraker-timelapse
origin: https://github.com/mainsail-crew/moonraker-timelapse.git
managed_services: klipper moonraker

[timelapse]
##   Following basic configuration is default to most images and don't need
##   to be changed in most scenarios. Only uncomment and change it if your
##   Image differ from standart installations. In most common scenarios 
##   a User only need [timelapse] in their configuration.
output_path: ~/timelapse/                ##   Directory where the generated video will be saved
frame_path: /tmp/timelapse/              ##   Directory where the temporary frames are saved
ffmpeg_binary_path: /usr/bin/ffmpeg      ##   Directory where ffmpeg is installed
```
# Software configuration
In this step, you'll need to copy several files from the `Sovol-SV08-Mainline` repo into the Klipper config; I found it easiest to use Midnight Commander to do this.  Run `mc` to start it.  You can use Tab to switch between the panes, Ctrl-T to tag files to copy, and F5 to copy them.  From `~/Sovol-SV08-Mainline/files-used/sovol-addons/`, copy `probe_pressure.py` and `z_offset_calibration.py` into `~/klipper/klippy/extras/`.  Then, from `~/Sovol-SV08-Mainline/files-used/config/`, copy everything (except `README.md`) into `~/printer_data/config`.



Katapult didn't work on my printer, which uses a rev 1.3 mainboard.  This is a reported issue on Rappetor's repo ([#260](https://github.com/Rappetor/Sovol-SV08-Mainline/issues/260)).  So rather than flashing Katapult, I built the Klipper firmware, and flashed it to both the mainboard and the toolhead using the ST-Link.  Future updates to the firmware on those boards, if any, will again require using the ST-Link adapter; I won't be able to do them over USB as I could if Katapult worked.
