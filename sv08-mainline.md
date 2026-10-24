---
title: Mainline Klipper on a Sovol SV08 in 2026
description: Notes on converting a Sovol SV08 printer to run Mainline Klipper
published: true
date: 2026-10-24T11:29:33.821Z
tags: 
editor: markdown
dateCreated: 2026-10-23T16:22:06.717Z
---

# Introduction

After buying a [Sovol SV08 3D printer](https://www.sovol3d.com/products/sovol-sv08-3d-printer), one of the first things I wanted to do was install mainline Klipper.  The canonical instructions for this process are at [https://github.com/Rappetor/Sovol-SV08-Mainline](https://github.com/Rappetor/Sovol-SV08-Mainline).  However, time, hardware revisions, and bugs resulted in several changes to my installation, which I wanted to document for my own reference, as well as anyone else who may find it helpful.

This guide is in two phases:

1. **[Phase 1](#phase-1-basic-installation-stock-hardware)**: mainline Klipper on the printer's stock hardware (stock probe, stock toolhead).
2. **[Phase 2](#phase-2-btt-eddy-with-tap)**: adding a BTT Eddy (USB) as the Z endstop, bed-mesh scanner, and "tap" Z-zeroing probe.

Throughout, places where I deviate from the Rappetor repo are called out like this:

> **Deviation from Rappetor:** a description of what's different and why.
{.is-warning}

My working configuration is backed up (with [Klipper-Backup](https://github.com/Staubgeborener/Klipper-Backup)) to a private Git repo; the first commit of that repo is essentially the end of the base install in Phase 1.  This was done in October 2026 against mainline Klipper as of that date.  Klipper changed a number of eddy-probe options in 2026, which is the main reason the Rappetor Eddy config needs changes (see Phase 2).

# Before you begin

Ideally, back up the entire `~/printer_data/config` directory on your printer.  At a minimum, back up your `printer.cfg` file.  You can do this through the Mainsail web interface.

You'll need:

- An eMMC module and a USB reader/adapter for it (I recommend a fresh module, so the stock one stays untouched as a fallback).
- An **ST-Link V2** (a cheap clone is fine) and the [STM32CubeProgrammer](https://www.st.com/en/development-tools/stm32cubeprog.html) software.  Rappetor's guide only uses this once, to install Katapult; because Katapult doesn't work on my board (see [below](#firmware-flash-klipper-directly-with-the-st-link)), you'll use it for every firmware update to the mainboard and toolhead.
- Optionally, an HDMI display.  I use a BTT HDMI5 with KlipperScreen in place of the stock screen.

# Base OS image

The Rappetor repo calls for use of the BTT CB1 v2.3.4 image.  This is based on Debian 11 (Bullseye), which is now EOL.  An image of Armbian is available specifically for the SV08, which is based on Debian 13 (Trixie), the current release.  That can be downloaded from [https://armbian.com/boards/sovol-sv08](https://armbian.com/boards/sovol-sv08).

> **Deviation from Rappetor:** Armbian (Debian 13) instead of the BTT CB1 v2.3.4 image (Debian 11).  This means you don't need Rappetor's custom `.dtb` file, the `BoardEnv.txt`/`system.cfg` edits, or the archived-Debian `sources.list` workaround.  The Armbian image ships its own SV08 device tree (`sun50i-h616-sovol-sv08.dtb`).
{.is-warning}

Once you write that image to your eMMC module, you'll need to make a few edits to the `armbianEnv.txt` file.  For my system, using a BTT HDMI5 display, I added the following to the beginning of that file:

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

# Phase 1: Basic installation (stock hardware)

## Flash and first boot

Install the eMMC module in your printer, plug in a keyboard and display, connect to your (wired) network, and power up.[^1]

When you boot the system, run through the first-boot process: create a root password, set up a non-root user (I used `biqu`), set your locale, time zone, etc.  Run `apt update && apt upgrade` to install system updates, `apt install git mc`, then reboot.

At this point, you can unplug the keyboard and display, and do the remainder of the installation via SSH.

## KIAUH

SSH to the printer as the user you created at first boot.  Then run `cd ~ && git clone https://github.com/dw-0/kiauh.git`.  This will install KIAUH, the Klipper Installation and Upgrade Helper, a script that you'll use to install much of the remaining software.  Also clone the `Sovol-SV08-Mainline` repo: `cd ~ && git clone https://github.com/Rappetor/Sovol-SV08-Mainline`.  You'll use this later to grab some configuration files.

## Klipper, Moonraker, etc.

Start KIAUH by running `./kiauh/kiauh.sh`.  Use it to install, in order, Klipper, Moonraker, Mainsail, Crowsnest, and (if you want it) KlipperScreen.  Using the Advanced menu option, install the additional dependencies for the Input Shaper.  Then reboot.

## Moonraker-timelapse

Install `moonraker-timelapse` by running:

```bash
cd ~/
git clone https://github.com/mainsail-crew/moonraker-timelapse.git
cd ~/moonraker-timelapse
make install
```

Then edit `~/printer_data/config/moonraker.conf` and add the following:

```ini
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

## Software configuration

In this step, you'll need to copy several files from the `Sovol-SV08-Mainline` repo into the Klipper config; I found it easiest to use Midnight Commander to do this.  Run `mc` to start it.  You can use Tab to switch between the panes, Ctrl-T to tag files to copy, and F5 to copy them.  From `~/Sovol-SV08-Mainline/files-used/sovol-addons/`, copy `probe_pressure.py` and `z_offset_calibration.py` into `~/klipper/klippy/extras/`.  Then, from `~/Sovol-SV08-Mainline/files-used/config/`, copy everything (except `README.md`) into `~/printer_data/config`.

Don't restart Klipper yet; the MCU firmware needs to be replaced first, and the config needs the changes described [below](#configuration-changes-from-the-rappetor-files).

## Firmware: flash Klipper directly with the ST-Link

Katapult didn't work on my printer, which uses a rev 1.3 mainboard (silkscreened `H616_JC_6Z_V1.3`).  This is a reported issue on Rappetor's repo ([#260](https://github.com/Rappetor/Sovol-SV08-Mainline/issues/260)): after Katapult is flashed, the MCU fails USB enumeration (`error -62`) every time, while restoring the stock firmware fixes it immediately.  So rather than flashing Katapult, I built the Klipper firmware, and flashed it to both the mainboard and the toolhead using the ST-Link.

> **Deviation from Rappetor:** Skip Rappetor's Step 7 (Katapult) and the `update_klipper_mcus_sv08.sh` script in Step 8 entirely.  Klipper is built with **no bootloader** and written straight to flash with the ST-Link.  Future updates to the firmware on those boards, if any, will again require using the ST-Link adapter; I won't be able to do them over USB as I could if Katapult worked.
{.is-warning}

### Back up the stock firmware

Do this first, exactly as described in Rappetor's **Step 6**, for both the mainboard and the toolhead.  With the printer **powered off**, wire the ST-Link to the board's SWD header (`3V3`→3.3V, `IO`→SWDIO, `CK`→SWCLK, `G`→GND; Rappetor's README has photos of where the headers are), connect, then *Read → Read all* and *Read → Save As…*.  Check that the backups are the right size (at least 128 KB for the toolhead, 512 KB for the mainboard).  These backups are your path back to stock.

### Build the firmware

The mainboard and the toolhead both use an STM32F103, but they need separate builds because the startup GPIO pins differ.  I keep a separate Kconfig file for each MCU so I don't have to redo `menuconfig` every time I update:

```bash
cd ~/klipper
make menuconfig KCONFIG_CONFIG=config.mainboard
```

Set:

```
[*] Enable extra low-level configuration options
    Micro-controller Architecture (STMicroelectronics STM32)
    Processor model (STM32F103)
[ ] Only 10KiB of RAM (for rare stm32f103x6 variant)
[ ] Disable SWD at startup (for GigaDevice stm32f103 clones)
    Bootloader offset (No bootloader)
    Clock Reference (8 MHz crystal)
    Communication interface (USB (on PA11/PA12))
(PA1,PA3) GPIO pins to set at micro-controller startup
```

> **Deviation from Rappetor:** Rappetor's menuconfig screenshots use the **8KiB bootloader** offset, which is for Katapult.  Without Katapult, it must be **No bootloader**, or the firmware won't boot.  Also make sure *Disable SWD at startup* stays **unchecked**; you need SWD to keep working, since the ST-Link is now the only way to update these boards.
{.is-warning}

`PA1,PA3` turns on the mainboard fan and the light at power-up (before Klipper connects); without it the printer starts up dark and silent.  Then build, and save a copy of the result:

```bash
make clean KCONFIG_CONFIG=config.mainboard
make KCONFIG_CONFIG=config.mainboard
cp out/klipper.bin ~/klipper-mainboard.bin
```

Repeat for the toolhead, with the same settings except the GPIO pins line, which is `PA6` (the hotend fan):

```bash
make menuconfig KCONFIG_CONFIG=config.toolhead
make clean KCONFIG_CONFIG=config.toolhead
make KCONFIG_CONFIG=config.toolhead
cp out/klipper.bin ~/klipper-toolhead.bin
```

Copy `~/klipper-mainboard.bin` and `~/klipper-toolhead.bin` to the computer that has STM32CubeProgrammer (`scp`, or SFTP).

### Flash the firmware

For each board, with the printer **powered off**:

1. Connect the ST-Link to the board, then to the computer, and click **Connect** in STM32CubeProgrammer.
2. On the *Erasing & Programming* tab, do a manual **Full chip erase**.
3. On the *Memory & File editing* tab, *Open file* the matching `.bin`, confirm the start address is `0x08000000`, and click **Download**.
4. Disconnect and remove the ST-Link.

Be careful not to swap the two files.

### Find the MCU serials

Power the printer on and run `ls /dev/serial/by-id/`.  You should see two entries beginning with `usb-Klipper_stm32f103xe_`.  To tell which is which, first, refer to your backup copy of `printer.cfg`.  If you can't find that, or if the paths there look like `/dev/ttyACM0`, unplug the USB cable at the toolhead and list them again: the one that disappears is the toolhead (`[mcu extra_mcu]`); the one that remains is the mainboard (`[mcu]`).  Plug the toolhead back in.  Put the full paths (not `ttyACM0`/`ttyACM1`) into `printer.cfg`:

```ini
[mcu]
serial: /dev/serial/by-id/usb-Klipper_stm32f103xe_<mainboard-id>-if00
restart_method: command

[mcu extra_mcu]
serial: /dev/serial/by-id/usb-Klipper_stm32f103xe_<toolhead-id>-if00
restart_method: command
```

> If the `[mcu]` and `[mcu extra_mcu]` serials are swapped, Klipper drives the wrong pins on each board, and **the heater can turn on at full power**.  Double-check before restarting Klipper.
{.is-danger}

## Configuration changes from the Rappetor files

Apart from the serials, these are the changes I made to Rappetor's `files-used/config` files for the stock-hardware configuration.

### printer.cfg

| Change | Why |
|---|---|
| Comment out `[include options/lcd/*.cfg]` | I replaced the stock screen with a BTT HDMI5 running KlipperScreen.  Leave this include in place if you keep the stock screen. |
| Comment out `[controller_fan MCU_fan]` and uncomment the `[temperature_fan MCU_fan]` block below it | The mainboard fan is controlled by host (CB1) temperature, with a 50 °C target, instead of running flat out whenever the motors or heaters are on.  Per the warning in Rappetor's comments, this doesn't work with a Noctua 24V 4010 fan; use `control: watermark` or keep the `controller_fan` in that case. |
| Uncomment `[adxl345]` and `[resonance_tester]`, and add `axes_map` | Rappetor leaves these commented out until the toolhead has new firmware, which it now has.  The toolhead's accelerometer isn't mounted in the default orientation, so it needs `axes_map: x, -z, y`.  See below. |
| `[input_shaper]` `damping_ratio_x`/`_y`: `0.001` → `0.1` | 0.001 is unrealistically low; 0.1 is Klipper's default and a sane value for this machine. |

The resulting accelerometer sections:

```ini
[adxl345]
cs_pin: extra_mcu:PB12
axes_map: x, -z, y

[resonance_tester]
accel_chip: adxl345
probe_points:
    177.5, 182, 20
```

The `axes_map` is what was correct for my printer.  To check yours, run `AXES_MAP_CALIBRATION` (or `ACCELEROMETER_QUERY` while the toolhead is at rest: the axis reading about 9800 mm/s² should come out as Z).

### sovol-macros.cfg

Rappetor's macros have a few bugs that I fixed while using the printer.  None are specific to the Eddy.

1. **Filament sensor checks never work.**  `START_PRINT`, `END_PRINT` and `RESUME` test `printer['filament_switch_sensor filament_sensor'].enable`, but the status field is `enabled`.  Change every `.enable` to `.enabled`.  In `RESUME` (the `state == 'normal'` branch), the test is also inverted (`.enable != True`); it should match the others:

   ```
   {% if printer['filament_switch_sensor filament_sensor'].enabled == True and
         printer['filament_switch_sensor filament_sensor'].filament_detected != True
   %}
   ```

2. **`PAUSE` only parks if both X and Y differ from the park position.**  Change the `and` to `or`:

   ```
   {% if printer.gcode_move.position.x != x_park or
           printer.gcode_move.position.y != y_park
   %}
   ```

3. **`RESUME` always un-retracts the pause-retract distance, whether or not `PAUSE` actually retracted** (for example, after a runout unload, or if the nozzle was cold).  That leaves a blob or a gap where the print resumes.  I track the retraction that still needs to be undone in a variable on `RESUME`:
   - In `[gcode_macro RESUME]`, add `variable_unretract: 0` under `variable_state`.  In the `state == 'normal'` branch, replace the unconditional `G91` / `G1 E{e_restract} F300` / `G90` with:
     ```
     {% if unretract > 0 %}
         G91
         G1 E{unretract} F300
         G90
     {% endif %}
     SET_GCODE_VARIABLE MACRO=RESUME VARIABLE=unretract VALUE=0
     ```
     Also add `SET_GCODE_VARIABLE MACRO=RESUME VARIABLE=unretract VALUE=0` just before `RESUME_BASE` in the `filament_change` branch.  The `{% set e_restract = ... %}` line in `RESUME` is no longer needed.
   - In `PAUSE`, add `SET_GCODE_VARIABLE MACRO=RESUME VARIABLE=unretract VALUE=0` right after `PAUSE_BASE`, and `SET_GCODE_VARIABLE MACRO=RESUME VARIABLE=unretract VALUE={e_restract}` right after the `G1 E-{e_restract} F300` / `G90` lines (the only place it really retracts).
   - In `LOAD_FILAMENT`, after the `G1 E-2 F300` / `G90` lines, add:
     ```
     {% if printer.print_stats.state == "paused" %}
         SET_GCODE_VARIABLE MACRO=RESUME VARIABLE=unretract VALUE=2
     {% endif %}
     ```
     so that loading filament mid-print un-retracts its own 2 mm on resume.

### options/probe/stock.cfg

The `[homing_override]` checks `'X' in rawparams` (and so on).  `rawparams` keeps the case it was sent in, so a lowercase `G28 x` matches no axis and homes nothing.  Change the three tests to `'X' in rawparams|upper`, `'Y' in rawparams|upper`, and `'Z' in rawparams|upper`.  (I found this after moving to the Eddy, and fixed it in `eddy.cfg`, but `stock.cfg` has the same code.)

### crowsnest.conf

`device: /dev/video0` can change between boots.  I use the camera's stable path instead, which you can find with `ls /dev/v4l/by-id/`:

```ini
device: /dev/v4l/by-id/usb-USB_Camera_USB_Camera_<id>-video-index0
```

## Calibration

Restart Klipper (`FIRMWARE_RESTART`) and check that both MCUs connect.  Then:

1. **PID tune** the hotend and bed at your usual temperatures, e.g. `PID_CALIBRATE HEATER=extruder TARGET=240` and `PID_CALIBRATE HEATER=heater_bed TARGET=65`, then `SAVE_CONFIG`.  `SAVE_CONFIG` comments out Rappetor's PID values in the `[extruder]` and `[heater_bed]` sections automatically and writes the new values to the auto-generated block at the bottom.
2. **Input shaper**: `SHAPER_CALIBRATE`, then `SAVE_CONFIG`.  Again, the hard-coded `shaper_type_*`/`shaper_freq_*` lines are commented out automatically.  For reference, mine came out to MZV at 54.6 Hz on X and MZV at 38.6 Hz on Y.
3. Set up the **slicer start G-code** and (optionally) heat-soak time as in Rappetor's Step 5, items 4 and 5.
4. Add the webcam in Mainsail (*Settings → Webcams*).

This is the end of Phase 1: a working SV08 on mainline Klipper with the stock probe.

# Phase 2: BTT Eddy with TAP

In this phase the BTT Eddy (I have the Eddy Duo, used in **USB** mode) replaces the stock probe for Z homing, quad gantry leveling, and bed meshing.  It also zeroes Z with a nozzle **tap** before each print.  Tap zeroing measures the actual nozzle-to-bed contact, so it is not affected by the probe's thermal drift, and I don't need to set the Z offset per print.

Tap support for eddy probes is in mainline Klipper; you don't need eddy-ng or BTT's Klipper fork.  But several eddy options were renamed or changed in 2026, so Rappetor's `options/probe/eddy.cfg` (which is based on BTT's older sample config) **needs the changes below to work on current Klipper**.

## Hardware

I mounted the Eddy with [this mount](https://www.printables.com/model/968900-sovol-sv08-btt-eddy-mount), which puts it under the stock shroud where the old probe was.  Rappetor's `eddy.cfg` offsets (`x_offset: -16`, `y_offset: 11.5`) are correct for that mount; if you mount it some other way, measure your own.  The Eddy connects to the host by USB.

## Eddy firmware

The Eddy uses an RP2040 and gets normal Klipper firmware, flashed over USB (no ST-Link needed).

```bash
cd ~/klipper
make menuconfig KCONFIG_CONFIG=config.eddy
```

```
[*] Enable extra low-level configuration options
    Micro-controller Architecture (Raspberry Pi RP2040/RP235x)
    Processor model (rp2040)
    Bootloader offset (No bootloader)
    Flash chip (W25Q080 with CLKDIV 2)
    Communication interface (USBSERIAL)
```

Then hold the Eddy's **BOOT** button while plugging it in (it shows up in `lsusb` as `2e8a:0003`), and:

```bash
make clean KCONFIG_CONFIG=config.eddy
make flash KCONFIG_CONFIG=config.eddy FLASH_DEVICE=2e8a:0003
ls /dev/serial/by-id/
```

You should now see a `usb-Klipper_rp2040_<id>-if00` entry.

## Configuration changes

### printer.cfg

1. Switch the probe include:

   ```ini
   # [include options/probe/stock.cfg] # Stock probe, default
   [include options/probe/eddy.cfg] # BTT Eddy probe
   ```

2. Comment out (or remove) the `[probe]` section (the one marked `# PLEASE REMOVE THIS WHEN USING EDDY`).  If there's a `[probe]` section in the `SAVE_CONFIG` block at the bottom, remove it too.

Nothing else in `printer.cfg` needs to change.  `[stepper_z]` already has `position_min: -5`, which tap probing needs (the nozzle has to be able to move below Z=0 to press on the bed), and `max_z_accel: 500` is above the 300 mm/s² the Klipper docs recommend for tap.  `[quad_gantry_level]` can stay at `horizontal_move_z: 10`.

### options/probe/eddy.cfg

1. Set `serial:` under `[mcu eddy]` to your `usb-Klipper_rp2040_...` path.

2. Rename `z_offset` to `descend_z` in `[probe_eddy_current btt_eddy]`.

   > **Deviation from Rappetor:** Klipper renamed this option in March 2026 (`z_offset` is deprecated), and it isn't a nozzle Z offset anyway: it's the height at which a probing move stops.
   {.is-warning}

   ```ini
   descend_z: 2.5
   ```

3. Add `zero_reference_position` to `[bed_mesh]`, at the same nozzle XY where `TAP_Z` (below) taps, so the mesh is exactly 0 at the point where Z was zeroed:

   ```ini
   zero_reference_position: 177.5,177.5
   ```

4. Fix the `[homing_override]` case bug (`'X' in rawparams|upper` and so on), as for [stock.cfg](#optionsprobestockcfg).  Also replace `SET_Z_FROM_PROBE` in its Z section with `_EDDY_SET_Z` (defined below).

5. Replace the entire macro block that follows (`[delayed_gcode RESTORE_PROBE_OFFSET]`, `SET_Z_FROM_PROBE`, `Z_OFFSET_APPLY_PROBE` and the `SET_GCODE_OFFSET` override, i.e. everything before `PROBE_EDDY_CURRENT_CALIBRATE_AUTO`) with:

   ```ini
   # Z babystepping uses Klipper's own SET_GCODE_OFFSET (no override). To keep a babystep
   # for future tap-zeroed prints, run Z_OFFSET_APPLY_PROBE (or Mainsail's Save button) and
   # then SAVE_CONFIG; Klipper stores it as tap_z_offset. See Z_OFFSET_APPLY_PROBE below.

   # Sets Z from the last PROBE: last_probe_position.z is the bed contact height in toolhead
   # coordinates, so shift the kinematic Z to make that contact point Z=0. Used after the eddy
   # homing move and after a tap. Must stay a separate macro so it is evaluated after PROBE finishes.
   [gcode_macro _EDDY_SET_Z]
   gcode:
       SET_KINEMATIC_POSITION Z={printer.toolhead.position.z - printer.probe.last_probe_position.z} SET_HOMED=Z

   # Taps the nozzle on the bed centre and zeroes Z at the contact point.
   # Requires homed axes and a clean nozzle at 150C or cooler (protects the PEI).
   [gcode_macro TAP_Z]
   gcode:
       {% if printer.toolhead.homed_axes != "xyz" %}
         {action_raise_error("TAP_Z: home all axes first")}
       {% endif %}
       {% if printer.extruder.target > 150 or printer.extruder.temperature > 155 %}
         {action_raise_error("TAP_Z: nozzle must be 150C or cooler before tapping")}
       {% endif %}
       BED_MESH_CLEAR
       SET_GCODE_OFFSET Z=0
       G90
       G1 Z5 F1000
       G1 X177.5 Y177.5 F6000
       M400
       PROBE METHOD=tap
       _EDDY_SET_Z
       G1 Z5 F1000
       M400

   # Mainsail's Z-offset Save button (and Sovol's _CALIBRATION_ZOFFSET) send a bare
   # Z_OFFSET_APPLY_PROBE, which on an eddy probe would rewrite the height calibration.
   # Default it to METHOD=tap so babysteps go to tap_z_offset instead. To deliberately
   # shift the eddy height calibration, use Z_OFFSET_APPLY_PROBE METHOD=calibration.
   [gcode_macro Z_OFFSET_APPLY_PROBE]
   rename_existing: _Z_OFFSET_APPLY_PROBE_EDDY
   gcode:
       {% if params.METHOD is defined %}
         _Z_OFFSET_APPLY_PROBE_EDDY {rawparams}
       {% else %}
         _Z_OFFSET_APPLY_PROBE_EDDY METHOD=tap {rawparams}
       {% endif %}
   ```

   > **Deviation from Rappetor:** Rappetor's `eddy.cfg` uses BTT's "beta Z-offset" macros, which override `SET_GCODE_OFFSET` and keep the offset in `saved_variables.cfg` as `nvm_offset`.  On current Klipper they don't work: `SET_Z_FROM_PROBE` reads `printer.probe.last_z_result` (deprecated January 2026; the replacement, `last_probe_position`, already has the probe offsets applied) and `z_offset` (renamed `descend_z`).  Klipper now stores the Z offset natively as `tap_z_offset`, so the overrides aren't needed at all.  The new `_EDDY_SET_Z` sets Z by shifting the *kinematic* position instead of applying a G-code offset, so babystepping with the normal `SET_GCODE_OFFSET` works.
   {.is-warning}

   I named the Z-setting macro `_EDDY_SET_Z` instead of Rappetor's `SET_Z_FROM_PROBE`: the leading underscore hides it in Mainsail, and the new name won't collide with other macro packages that define a `SET_Z_FROM_PROBE` of their own.

   The `Z_OFFSET_APPLY_PROBE` wrapper is important.  A bare `Z_OFFSET_APPLY_PROBE` on an eddy probe adjusts the **height calibration** instead of the tap offset, and Mainsail's "Save" button next to the Z-offset sends exactly that.

Leave `PROBE_EDDY_CURRENT_CALIBRATE_AUTO` and `PROBE_EDDY_MOVE_TO_SCREW` as they are.

### sovol-macros.cfg

1. **Disable the `PROBE_CALIBRATE` override.**  `probe_eddy_current` doesn't provide `PROBE_CALIBRATE`, so the `rename_existing: PROBE_CALIBRATE_BASE` macro makes Klipper fail at startup.  Comment out the whole `[gcode_macro PROBE_CALIBRATE]` block (restore it if you ever go back to the stock probe).

2. **Use the Eddy for meshing.**  In both `START_PRINT` and the `BED_MESH_CALIBRATE` override, change

   ```
   BED_MESH_CALIBRATE_BASE ADAPTIVE=1 #METHOD=rapid_scan
   ```

   to

   ```
   BED_MESH_CALIBRATE_BASE ADAPTIVE=1 METHOD=rapid_scan
   ```

   > Rappetor (and BTT) suggest `rapid_scan`, and it's what I use.  The Klipper docs suggest `METHOD=scan` for normal printing, as slightly more accurate for a bit more time; it's worth trying if your first layers aren't consistent.
   {.is-info}

3. **Tap before meshing.**  In `START_PRINT`, just after the `{% endif %}` that closes the QGL block and before the `M117 Bed Mesh` line, add:

   ```
   # Zero Z with an eddy tap. CLEAN_NOZZLE wipes the nozzle, re-homes Z and leaves it at 130C,
   # cool enough to tap without marking the PEI. The mesh below is referenced to the tap point.
   M117 Clean nozzle
   CLEAN_NOZZLE
   M117 Tap Z
   TAP_Z
   ```

   Sovol's stock `CLEAN_NOZZLE` heats to 200 °C, wipes, and then sets the hotend to 130 °C and re-homes Z, so the nozzle is clean and cool enough to pass `TAP_Z`'s 150 °C check.  `START_PRINT` then heats to printing temperature after the mesh.  Leave `Z_OFFSET_CALIBRATION` (the stock pressure-sensor offset routine) commented out; it isn't used with the Eddy.

## Calibration

Run these in order.  Each step that ends in `SAVE_CONFIG` restarts Klipper.

1. **Drive current.**  Z can't be homed yet (the Eddy is the Z endstop and isn't calibrated), so: `G28 X Y`, move to the center with `G1 X177.5 Y177.5 F6000`, and get the nozzle to roughly 20 mm above the bed.  If it's lower than that, use `SET_KINEMATIC_POSITION Z=0` followed by `G1 Z20 F600` to raise it.  Then:

   ```
   LDC_CALIBRATE_DRIVE_CURRENT CHIP=btt_eddy
   SAVE_CONFIG
   ```

   Mine came out to `reg_drive_current = 15`.

2. **Height calibration.**  Run `PROBE_EDDY_CURRENT_CALIBRATE_AUTO CHIP=btt_eddy`.  The macro homes X/Y, centers the toolhead, and (if Z isn't homed) tells Klipper the toolhead is near maximum Z, so you can bring the nozzle down to the bed.  Do a paper test with `TESTZ` (or Mainsail's manual-probe dialog), then `ACCEPT`.  The Eddy then moves over that spot and maps frequency to height, which takes a couple of minutes.  `SAVE_CONFIG`.  Because `[temperature_probe btt_eddy]` is defined, this also records `calibration_temp`.  `G28` should work from here on.

3. **Thermal drift calibration.**  Start with the bed, nozzle and Eddy cold, and a clean nozzle.  `G28`, heat the nozzle to about 160 °C (`M109 S160`) so it doesn't expand during the calibration, then:

   ```
   TEMPERATURE_PROBE_CALIBRATE PROBE=btt_eddy TARGET=<max coil temp>
   ```

   It moves to `calibration_position` (over the center-right bed screw, `319,176,5`) and asks for a paper test.  `ACCEPT`, then turn the bed on at its maximum (100–105 °C).  Each time the coil temperature rises by `STEP` (2 °C by default), it asks for another paper test.  Set `TARGET` to the highest temperature the coil actually reaches (watch `btt_eddy` in Mainsail); on an open-frame SV08 that's fairly low.  If it levels off before reaching `TARGET`, run `TEMPERATURE_PROBE_COMPLETE`.  Then `SAVE_CONFIG`.  This takes a while.  Tap zeroing doesn't depend on it, but QGL and the mesh do.

4. **Tap threshold.**  Make sure the nozzle is clean and at 150 °C or cooler, and the bed is clean.  `G28`, then `G1 X177.5 Y177.5 Z5`, and be ready to hit the emergency stop (`M112`) if the nozzle doesn't stop when it touches the bed:

   ```
   PROBE_EDDY_CURRENT_TAP_CALIBRATE TAP=guess
   PROBE_EDDY_CURRENT_TAP_CALIBRATE TAP=refine
   PROBE_EDDY_CURRENT_TAP_CALIBRATE TAP=verify
   SAVE_CONFIG
   ```

   Before each command, move back to Z 3–10 mm over the center.  `verify` taps five times; if it passes, `SAVE_CONFIG` stores `tap_threshold`.  Mine was about 3776, but yours will differ.  If a step fails, the troubleshooting section of the [Klipper eddy probe docs](https://www.klipper3d.org/Eddy_Probe.html#tap-calibration) explains how to adjust it by hand with `PROBE METHOD=tap TAP_THRESHOLD=<value>`.

   > Klipper changed how `tap_threshold` is interpreted on 2026-05-01.  Values from older guides or from before that date need to be recalibrated.
   {.is-info}

5. **Test.**  `G28`, `QUAD_GANTRY_LEVEL`, `G28 Z`, then `TAP_Z`.  The nozzle should come down at the bed center, touch, and lift back to Z5.  Then `BED_MESH_CALIBRATE` to check the mesh.

6. **First-layer Z offset.**  Start a print, and babystep the Z offset in Mainsail until the first layer looks right.  Then click **Save** next to the Z-offset (because of the wrapper above, this runs `Z_OFFSET_APPLY_PROBE METHOD=tap`) and `SAVE_CONFIG` when the print is done.  The offset is stored as `tap_z_offset` and applied to every later `TAP_Z`.  Mine is `-0.040`.

After these steps, every print does QGL, re-homes Z, cleans the nozzle, taps to zero Z, takes an adaptive mesh referenced to the tap point, and then heats to printing temperature.

# Summary of deviations from Rappetor

| Area | Rappetor | This build |
|---|---|---|
| OS image | BTT CB1 v2.3.4 (Debian 11) + custom `.dtb`, `BoardEnv.txt` | Armbian for SV08 (Debian 13), `armbianEnv.txt` |
| Bootloader | Katapult on both MCUs | None; Klipper flashed directly with the ST-Link ([#260](https://github.com/Rappetor/Sovol-SV08-Mainline/issues/260)) |
| MCU firmware | 8 KiB bootloader offset, built/flashed by `update_klipper_mcus_sv08.sh` | No bootloader, built per MCU with `KCONFIG_CONFIG=`, flashed with STM32CubeProgrammer |
| Display | Stock LCD | BTT HDMI5 + KlipperScreen (`options/lcd` include commented out) |
| MCU fan | `controller_fan` | `temperature_fan` on host temperature |
| Accelerometer | Commented out | Enabled with `axes_map: x, -z, y`; `damping_ratio` 0.1 |
| Macros | `.enable` typo, PAUSE park `and`, unconditional un-retract on RESUME | `.enabled`, `or`, tracked un-retract |
| Homing override | Case-sensitive axis match | `rawparams\|upper` |
| Webcam | `/dev/video0` | `/dev/v4l/by-id/...` |
| Eddy: options | `z_offset`, `last_z_result` | `descend_z`, `last_probe_position` |
| Eddy: Z offset | BTT beta macros (`nvm_offset`) | Native `tap_z_offset` with `Z_OFFSET_APPLY_PROBE` defaulting to `METHOD=tap` |
| Eddy: Z zero | Eddy homing only | Tap before each mesh, with `zero_reference_position` |
| Eddy: macros | `PROBE_CALIBRATE` override left in place | Commented out (breaks startup with the Eddy) |

[^1]: If you need to connect your printer to WiFi, the process is a little complicated.  Once you've run through the first-boot process, you'll need to configure it by creating a YAML file.  To do this, at the shell prompt, run `nano /etc/netplan/30-wifis-dhcp.yaml`.  For its contents, enter this:
    ```yaml
    network:
      version: 2
      renderer: networkd
      wifis:
        wlan0:
          dhcp4: true
          dhcp6: true
          access-points:
            "Your-SSID":
              password: "your-password"
    ```
    These indents use spaces, and the spacing is critical.  And, of course, substitute your own WiFi SSID and password.  Once finished, Ctrl-X to exit, Y to save.  Then `chmod 600 /etc/netplan/*.yaml` to make the file readable by root only, then `netplan try`.  The last command will validate the file syntax and attempt to connect to your WiFi.
