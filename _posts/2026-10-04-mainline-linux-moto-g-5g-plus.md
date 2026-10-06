---
layout: post
title: Mainline Linux and Arch Linux ARM on the Moto G 5G Plus
date: 2026-10-04
description: The problems I hit bringing a mainline kernel and Arch Linux ARM up on the Motorola Moto G 5G Plus (SM7250, "nairo"), and how each one was solved.
tags: linux arm qualcomm kernel
categories: projects
toc:
  sidebar: left
---

The Motorola Moto G 5G Plus (XT2075-3, codename **nairo**) is built around the
Qualcomm SM7250 (Snapdragon 765G). This post collects the problems I ran into
while getting a mainline kernel (linux-next, 7.3-rc5) and a native Arch Linux ARM
userspace running on it, and how each was solved. It is not a copy-paste guide,
but it follows the order in which things had to happen.

Current state: the phone boots Arch Linux ARM from the SD card, with USB
networking and SSH, UFS storage, the SD card, touch, the backlight, battery
monitoring and a working Adreno GPU: Plasma Mobile composites with OpenGL on
it. There is no display driver yet (frames are copied into the bootloader
framebuffer), and no Wi-Fi, modem or audio.

## Starting point

- SM7250 is not in mainline. A base series (CPUs, GCC, RPMh, serial) has been
  posted to linux-arm-msm but isn't merged, and it has no pinctrl driver. I
  applied it on top of linux-next.
- The bootloader is unlocked, but `fastboot boot` is not supported. Every test
  has to be flashed. I kept LineageOS on slot A and flashed test kernels to
  `boot_b`, switching with `fastboot set_active b`. If a kernel doesn't boot,
  vol-down into fastboot and `set_active a` recovers the phone.
- With no serial console, early logs come from **pstore/ramoops**: the kernel
  log survives a warm reset and can be read from LineageOS in
  `/sys/fs/pstore/console-ramoops-0`. `PSCI SYSTEM_OFF` at a chosen point was
  the most reliable "did we get this far?" probe; "spin forever" probes were
  confused by watchdog resets.
- The downstream 4.19 kernel source and, more importantly, the **live device
  tree** pulled from the running LineageOS (`/sys/firmware/fdt`) were the
  hardware reference for addresses, GPIOs and regulators.

## Getting the bootloader to start a mainline kernel

Motorola's ABL turned out to be the hardest part of the first boot. Reading its
disassembly explained the failures:

1. **No EFI stub header.** ABL refused to start an `Image` with the MZ/PE
   header, so the kernel is built with `CONFIG_EFI` disabled.
2. **The dtbo partition can't be removed.** ABL applies an overlay from `dtbo`
   and _aborts the boot_ if that overlay fails to apply. The LineageOS overlays
   reference dozens of labels that a mainline DT doesn't have, and every
   blank, empty or zero-entry dtbo I built was rejected.
3. **The model string must match.** ABL skips any DTB whose `model` isn't
   `Qualcomm Technologies, Inc. Lito` (case-insensitive).

The fix was to keep the original LineageOS dtbo and give it something harmless
to apply to. A script reads the stock dtbo, collects every label its overlays
reference, and generates an **overlay sink**: a disabled node full of empty
child nodes with fixed phandles, exported in `__symbols__` under those label
names. The overlays apply cleanly, and all they modify is dead nodes.

One side effect showed up much later: the overlay also **replaces the root
`compatible`** with `"qcom,lito-nairo", "qcom,lito-moto", "qcom,lito"`. Kernel
code that matches on the machine compatible never sees `qcom,sm7250` (see the
GPU section).

## First boot hangs: TrustZone-owned GPIOs

With that, the kernel started: 8 CPUs, the bootloader's framebuffer as
`simple-framebuffer`, fbcon. It then hung in late initcalls. Bisecting with
`initcall_debug` led to the TLMM pinctrl driver. GPIOs 59–62 belong to the
fingerprint sensor's SPI bus, which TrustZone owns, and reading their
registers from Linux hangs the bus. `gpio-reserved-ranges = <59 4>` in the
board DT fixed it, and boot completed.

## USB networking and SSH

The phone is USB 2.0 only. Two things were needed for the dwc3 gadget:

- `qcom,select-utmi-as-pipe-clk`, because there's no USB3 PHY pipe clock.
  Without it, binding the gadget failed with `failed to enable ep0out`.
- The **apps SMMU** node with the USB stream ID. Without it, the controller's
  first DMA reset the whole phone within about 15 seconds (and wiped pstore,
  which made it hard to find).

The initramfs then brings up an NCM gadget, a DHCP server and dropbear, so I
could SSH in over the cable.

## Storage: UFS and the SD card

UFS (a Samsung part with 6 LUNs) needed:

- an SM7250-specific QMP UFS PHY configuration, with tables converted from the
  downstream driver;
- the UFS reset line, which is exposed as a TLMM GPIO;
- a **32-bit DMA mask**. With 64-bit DMA, descriptors above 4 GiB were silently
  never processed: no interrupt, no SMMU fault. Downstream gets away with
  64-bit because it never resets the SMMU; the proper fix is UFS's SMMU stream
  ID, which I haven't found yet.

The link currently only negotiates HS-G1 instead of G3. The SD card works at
UHS-I SDR104, also with 64-bit DMA masked off for the same reason.

The userdata partition uses Android metadata encryption and is unreadable from
mainline, so Arch Linux ARM lives on the **SD card**: an exFAT partition for
media and an ext4 partition with the Arch Linux ARM tarball extracted.

## Booting Arch Linux ARM

The boot image carries a small initramfs. Its init:

1. brings up the USB gadget, so there is always a way in;
2. waits for the SD card and mounts the root partition given by
   `nairo.root=` on the command line;
3. runs a preparation script that installs the SSH key and a
   systemd-networkd config for the USB interface, and enables sshd and a
   serial getty on the USB ACM port;
4. moves `/dev`, `/proc`, `/sys` and `/run` and calls `switch_root` into
   systemd.

If anything fails, it stays in the initramfs shell, still reachable over USB.

## Touch

The touchscreen is a Novatek NT36672C in a TDDI panel, on SPI, with no flash:
the firmware is downloaded at every power-on. Mainline only has an I²C Novatek
driver. A community SPI driver for a sibling chip (NT36523) worked after
several fixes:

- Motorola's firmware file layout (the image size comes from an end flag, and
  there's a different header);
- the hardware-CRC download order;
- real chip detection from the trim ID instead of a hardcoded memory map;
- clean failure when the download fails, instead of a crash.

## A rule learned the hard way: load new drivers by hand

An early version of the touch driver crashed during probe. Because it loaded at
boot, it locked me out: sshd reset connections and logins looped. Worse, even
after I blacklisted it, running `depmod -a` let udev autoload it again.

Since then, every new driver is built as a module into a **private module
tree** (`/usr/local/lib/nairo-modules`) that depmod never indexes for the
running kernel, so nothing loads it automatically. I load it over SSH, with
`journalctl -kf` open in a second session:

```sh
modprobe -d /usr/local/lib/nairo-modules <module>
```

Only drivers that have proven stable get moved to boot. So far that's UFS, the
backlight and the GPU clock controller (built in), touch (loaded from the
initramfs) and the GPU driver, which a small systemd unit loads from the private
tree before the display manager starts.

## Battery: charger and fuel gauge

- **Charger:** the PM7250B's SMB5 charger is covered by an upstream series
  adding SMB5 support to `qcom_smbx`. I applied it and added the charger node.
  The DT only gives the capacity, so the hardware keeps the charge limits the
  bootloader programmed.
- **Fuel gauge:** the PM7250B "QG" gauge has no mainline driver, so I wrote a
  small read-only one. It reads battery voltage and current every 10 seconds
  and estimates the state of charge from an open-circuit-voltage table.
  - The table comes from the downstream battery profile, converted to the
    mainline `ocv-capacity-table` format by a script.
  - The first version read the averaged FIFO registers, which only update when
    the FIFO completes. Readings stayed frozen for minutes. Switching to the
    latest ADC sample and burst-averaged current registers, which downstream
    uses for its "now" values, fixed it.

A test ran using `screen` and unplugging the USB showed +145 mA while charging and −396 mA on battery, so the sign
convention is right. There is currently no other way to test this as the WiFi stack still has to be implemented. Accuracy against LineageOS's gauge is still to be checked.
`for i in 1 2 3 4 5 6 7 8 9 10; do grep -E "VOLTAGE_NOW|CURRENT_NOW" /sys/class/power_supply/qcom-qg/uevent; sleep 10; done`

## Backlight

The panel backlight is the PM8150L WLED, which the mainline `qcom-wled` driver
supports. I used the downstream settings, but limited the current to 20 mA per
string because early PM8150L revisions can't do the default 25 mA. Loading and
unloading the module showed two real driver bugs:

- the driver never called `platform_set_drvdata()`, so removing it
  dereferenced a NULL pointer;
- when the bootloader had already left the WLED on, the driver didn't record
  it, and the first brightness change enabled the over-voltage interrupt a
  second time (`Unbalanced enable for IRQ`).

With both fixed, brightness works over its full 0–4095 range.

## GPU: Adreno 620

The GPU is an Adreno 620 (chip ID `0x06020000`). Mainline's `msm` driver has
the closely related A621 but not the A620. Getting it to probe took several
pieces:

- **Clock controller.** The SM7250 GPU clock controller is the SM8250 one with
  a different PLL configuration, so it became a variant of the existing
  driver.
- **Catalog entry.** A new A620 entry, modelled on the A650 and using the
  downstream firmware names (`a650_sqe.fw`, `a650_gmu.bin`, a620 zap shader).
- **Power-controller setup.** Mainline assumes boot firmware programs the GPU
  power controller (PDC) on this whole GPU family. Downstream programs it from
  the kernel on the A620, so the driver now does that, taking the XO resource
  address from the command DB.
- **Bus bandwidth table.** The addresses are read from the command DB by name.
  Mainline's hardcoded A650 values would have been wrong on this SoC.
- **Probe order.** The GPU's IOMMU gives up on its clock controller after the
  10-second deferred-probe timeout and never retries, and its driver doesn't
  allow manual binding. Building the clock controller in fixed it.
- **The compatible rewrite.** The last failure was `Couldn't find UBWC config
data for this platform`. The memory-compression settings are looked up by
  root compatible, and the dtbo overlay had replaced `qcom,sm7250` with
  `qcom,lito`. A local table entry for `qcom,lito` fixed it.

That got `msm` to initialise, but the GPU firmware only starts on first use,
and the first real client brought out four more problems:

- **The zap shader region.** TrustZone verified the zap shader's signature but
  refused to place it: the reserved region from the downstream device tree
  starts at `0x8bc15400`, which isn't page-aligned. Downstream never actually
  uses that region (it loads the zap shader into ordinary memory), so the fix
  was to use the aligned page inside it.
- **A frozen CPU on the first GPU client.** Mesa's first call froze a CPU so
  hard it ignored even the stall detector. Logging every step of the path
  narrowed it to the GPU SMMU: mainline programs the "PRR" registers on
  Adreno SMMUs, which this generation doesn't have, and a write to a
  non-existent register never completes. Mainline already skips SM8250 for
  exactly this reason; SM7250 needed the same one-line exclusion.
- **SMMU coherency.** The SM8250 device tree I started from marks the GPU
  SMMU's page-table walk as cache-coherent, the downstream SM7250 tree doesn't,
  so I followed downstream.
- **Faults on the first shared buffer.** The first kmscube run after every
  boot faulted on its whole framebuffer. A GPU crash dump showed the
  framebuffer was in the job's buffer list but completely absent from the GPU
  page table. The framebuffer is shared with the simpledrm display driver, and
  logging the import showed a scatter list whose entries all had length 0:
  `CONFIG_DMABUF_DEBUG`, on by default in debug kernels, deliberately blanks
  the CPU side of imported buffers to catch drivers that look at it, and
  `msm` maps imported buffers from exactly that. Turning the option off fixed
  it.

With that, `eglinfo` reports `FD620` with OpenGL 4.6 and OpenGL ES 3.2,
offscreen rendering runs at about 16,000 FPS, kmscube draws, and KWin in
Plasma Mobile composites on the GPU (`OpenGL renderer string: FD620`) with no
GPU faults. The display path is still simpledrm, so each frame is copied into
the bootloader framebuffer with no vsync.

## Kernel headers for DKMS

The kernel is cross-compiled, so a plain copy of the build tree can't build
modules on the phone: the helper programs an external module build runs are
x86 binaries. A packaging script merges the source and build trees into
`/usr/lib/modules/<version>/build` and rebuilds the two helpers this config
needs (`fixdep` and `modpost`) for aarch64. With no module versioning, no
module signing and no active GCC plugins, nothing else has to run on the
phone, and DKMS works with the phone's own gcc.

## Plasma Mobile

Install sddm, plasma-mobile, plasma-mobile-settings
using yay install

```
yay -S spacebar kclock kweather qmlkonsole neochat angelfish kasts calindori tokodon audiotube plasmatube krecorder haruna okular gwenview
```

Then enable autologin

```/etc/sddm.conf.d/autologin.conf

[Autologin]
User=alarm
Session=plasma-mobile
```

and can start sddm `systemctl start sddm`.

To fix the screen display, do

```/usr/share/plasma-mobile-device-presets/motorola,nairo.conf
# SPDX-FileCopyrightText: 2026 Stefan Riesenberger <stefan.riesenberger@gmail.com>
# SPDX-License-Identifier: GPL-2.0-or-later

[Device]
name=Motorola Moto G 5G Plus XT2075-3
formFactor=phone
codename=nairo

[Display]
# Force the system to use the correct screen dimensions
width=1080
height=2520
# A scaling factor of 2.0 (200%) or 2.25 works best for a 6.7" screen to prevent tiny text
scale=2.5
# Matches the native 90Hz smooth refresh rate of the device
refreshRate=90

[Shell]
# Adjust panel sizing relative to the ultra-tall 21:9 screen layout
topPanelHeight=60
bottomPanelHeight=72

[Input]
# Configures default touch haptic and virtual keyboard setup
virtualKeyboard=org.kde.plasma.maliit

[Panels]
statusBarHeight=106
navigationPanelHeight=90
leftPadding=30
rightPadding=250

[Panels][Top]
centerSpacing=60
```

and enable it as

```/etc/xdg/plasmamobilerc
[Device]
device=motorola,nairo
```

and restart sddm

## Waydroid - WIP

install kernel headers, dkms, waydroid, git, subversion, base-devel, go, ninja, meson
Then, install waydroid.
Then, from the aur, install binder_linux-dkms (change the architecture to "any")

## What doesn't work yet

- **Display:** only the bootloader framebuffer. A real display needs the
  DPU/DSI pipeline and a driver for the NT36672C panel (DSC, video mode).
- **Wi-Fi and Bluetooth:** the chip is a WCN399x-class part. Wi-Fi depends on
  the modem DSP, so it needs the whole remoteproc / shared-memory / AOSS base
  layer, which the SM7250 device tree doesn't have yet.
- **Modem, audio, cameras, sensors, suspend.**
- **GPU frequency scaling:** only the two lowest levels (275 and 400 MHz) are
  enabled until the speed-bin fuse is described.
- **Reboot to bootloader or recovery:** the standard PMIC and IMEM reboot
  reasons are ignored. Motorola sets an extra PMIC flag and forces a hard reset,
  which mainline doesn't do yet.
- **UFS speed:** HS-G1 instead of HS-G3, and 32-bit DMA until the SMMU stream
  ID is known.
