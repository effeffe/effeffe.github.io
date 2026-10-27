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

Current state: the phone boots Arch Linux ARM from its internal UFS storage,
with USB networking and SSH, Wi-Fi, the SD card, touch, the backlight, battery
monitoring with an off-mode charging screen, a real display driver (DPU, DSI
and the panel) and a working Adreno GPU: Plasma Mobile composites with
OpenGL on it. Audio works: the loudspeaker, the earpiece, the headset
jack and both microphones, and calls have sound both ways, with mute and
speakerphone. The modem registers on the mobile network (LTE) and
ModemManager sees it: calls, SMS in both directions and mobile data work.

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

The SD card works at UHS-I SDR104, also with 64-bit DMA masked off for the
same reason.

The userdata partition uses Android metadata encryption and is unreadable from
mainline, so at first Arch Linux ARM lived on the **SD card**: an exFAT
partition for media and an ext4 partition with the Arch Linux ARM tarball
extracted. UFS turned out to need much more work before it could be trusted
with writes; see [Making UFS reliable](#making-ufs-reliable) below.

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
GPU faults. At that point the display path was still simpledrm, so each frame
was copied into the bootloader framebuffer with no vsync; see below for the
real display driver.

### Clocks and temperatures

The first GPU table had only 275 and 400 MHz, the two levels every
downstream speed bin shares. The downstream kernel picks its table from a
speed-bin fuse: bins `0x00` and `0x84` go up to 625 MHz, bin `0x73` to
540 MHz. Reading the fuse (bits 5–12 at offset `0x1ea` of the QFPROM at
`0x784000`) gave `0x73`, which fits a plain Snapdragon 765: Qualcomm sold
the same SM7250 die as the 765, the 765G (625 MHz GPU) and the 768G
(750 MHz). The device tree now describes the fuse, every level is tagged
with the bins it belongs to, and `msm` picks 275, 400 and 540 MHz on this
phone. Some spec databases list 750 MHz for this chip; that is the top of
Qualcomm's shared frequency list, which no bin on this phone uses. As a
test, I also enabled 625 MHz, which uses the same voltage level as 540: the
GPU management unit then failed to bring the GPU up at boot (`Timeout
waiting for GMU OOB set GPU_SET`), so 540 MHz it is.

Running faster needed temperature readings first, and the SoC had none:
the mainline SM7250 device tree had no TSENS. The two controllers sit at
the same addresses as on SM8250, so they and 27 thermal zones, named after
the downstream ones, went in: every zone shuts the phone down at 110 °C,
and the two GPU zones throttle the GPU from 85 °C. At idle the sensors read
33–42 °C.

Under FurMark, for almost five minutes, the GPU stayed at 540 MHz and its
sensors levelled off at 53–57 °C, far from the 85 °C throttle point.
FurMark draws about 4 frames per second at 1080p, roughly what a laptop's
integrated graphics manages in the same test. `peglgears` went from about
16,000 to 20,000–24,700 frames per second. Vulkan works too: Mesa's Turnip
driver reports the Adreno 620 with Vulkan 1.3, and `vkcube` runs.

## Memory bandwidth: an interconnect driver

The GPU still never asked for memory bandwidth: the A620 support sends the
GPU management unit a single "off" bandwidth level, and mainline doesn't
vote for the GPU's DDR bandwidth on its close relatives either. Looking
into it showed a bigger gap: SM7250 had no interconnect driver at all, so
nothing in Linux voted for DDR or bus bandwidth. A `memcpy` from one core
still ran at 14 GB/s, because the bootloader leaves DDR at a high speed.
That is fine for speed and wasteful for power.

The interconnect driver describes every network-on-chip on the SoC, which
masters connect to which slaves, and which RPMh "bus clock manager" votes
for each link. Motorola's kernel source has that topology as
`lito-bus.dtsi`, next to `kona-bus.dtsi` for SM8250, whose mainline
driver was derived from the latter. A script parsed both files, matched
the nodes by their downstream IDs, and generated an SM7250 driver in the
mainline style: 12 providers, 138 nodes and 26 bus clock managers, all of
which exist in the phone's command DB. SM7250 differs from SM8250 in more
places than expected: two DDR channels instead of four, a narrower link
from the GEM NoC to the system NoC, an eMMC master, and no PCIe.

The catch with an interconnect driver is the hand-over. Once the providers
are up, every path is voted at whatever its consumers ask for, and a
device without a vote loses its bandwidth. So the first test kernel also
gave the main users their votes:

- each CPU frequency a DDR bandwidth, taken from the downstream "memlat"
  tables (in MiB/s, which convert exactly to mainline's kB/s: 7980 MiB/s
  is DDR at 2092 MHz);
- each GPU level the DDR vote of its downstream power level: 2.19, 3.07
  and 4.07 GB/s for 275, 400 and 540 MHz;
- UFS, USB, the SD card, the modem's data path (IPA) and the serial
  engines that the touchscreen uses, with the same paths as SM8250.

That kernel booted with every vote in place, but the screen flickered and
showed artifacts. The display engine was still scanning out the
bootloader's framebuffer with no driver behind it, so nothing voted for
its bandwidth, and whenever the CPUs went idle the memory paths slowed
down enough for the scanout to underrun. The stock device tree shows what
Motorola's display driver asks for while the screen is on: 6.4 GB/s on
each of the display engine's two memory ports, with a DDR floor of
0.8 GB/s. A small local patch to `simpledrm` now holds that vote for the
framebuffer, until a real display driver takes over.

With that, the display is clean, a `memcpy` runs at 13.6 GB/s (10.1 GB/s
on a small core), UFS reads at 475 MB/s, and mobile data and the
GPU all work with their own votes.

## A real display: DPU, DSI and the panel

Until now every frame went through `simpledrm`: KWin drew into the buffer
the bootloader had left on screen, and the display engine scanned it out
without knowing Linux existed. A real display driver needs four pieces,
and SM7250 had none of them upstream.

- **Display clock controller.** Every clock it shares with SM8250 sits at
  the same register offset, so it became a variant of the SM8250 driver.
  Two differences: SM7250 has no second display PLL (the mux input it used
  carries a 600 MHz branch of the global PLL instead), and no MMCX power
  rail, so the display runs from CX. The first test kernel blanked the
  screen as soon as the clock driver loaded: with no consumers, it switched
  the display power domain off. The simple-framebuffer node now lists the
  power domain and the clocks it needs, so the splash stays powered until
  the real driver takes over.
- **Display engine (DPU).** The hardware reports version 6.1, the one gap
  between SM8250 (6.0) and SC7180 (6.2) in mainline. Its catalog entry, two
  layer pipes for video and three for graphics, four mixers, two DSC
  encoders and so on, comes straight from the downstream device tree, down
  to the QoS tables. The display engine also needed adding to the IOMMU
  driver's list of devices that start in bypass mode: without that it lost
  access to the splash buffer at 0.2 seconds into boot.
- **DSI and its PHY.** Same addresses and 7 nm PHY as SM8250.
- **The panel.** A Tianma 1080x2520 panel on a Novatek NT36672C, in video
  mode with display stream compression (DSC). Mainline had no driver for
  it, so I wrote one from the downstream panel description: init sequence,
  timings and DSC parameters.

The first full boot lit the panel through the new driver, but the desktop
never started: the GPU and the display had come up as two separate DRM
devices, because a module option from the GPU bring-up still asked for that.
With them merged into one device, Plasma Mobile came up on the real display:
60 Hz with vsync, no CPU copy, no artifacts.

Touch did not. The NT36672C is a touch and display chip, and the panel
driver's reset also resets the touch controller, which loses its firmware.
Reloading the firmware after the panel's init sequence was not enough
either, until the init sequence matched downstream exactly: the DSC
parameters go out after the display is switched on, not before. Two
changes fixed it for good. The panel driver now keeps the panel as the
bootloader left it on the first power-up (no blink at boot), and the touch
driver registers as a "panel follower", the mainline mechanism for this
kind of chip: it reloads its firmware after every panel power-up and puts
the touch side to sleep before the panel goes off. Touch now survives
turning the screen off and on.

### 90 Hz, the bias rails and the backlight

- **90 Hz.** The panel changes refresh rate the way the downstream driver
  does it: same pixel clock, shorter vertical front porch (27 lines instead
  of 1316). The driver now offers both modes. Plasma Mobile doesn't ship the
  display settings module, so switching needs `kscreen` installed; with it,
  90 Hz works from the settings, and `glxgears` runs at 88 frames per
  second.
- **LCD bias.** The panel's ±5.5 V supplies come from the LCDB block of the
  PM8150L PMIC, which had no mainline driver. The new one follows the
  downstream register layout (checked against the live registers: both
  outputs at 5.5 V, the boost 150 mV above), shares the block's single
  enable between the two outputs, and adopts the bootloader's state at
  boot. The panel driver now switches the bias off with the screen and
  back on with it, in the downstream order.
- **Backlight.** Switching to 90 Hz dimmed the screen: the bootloader
  leaves the panel's own brightness register at its maximum, which
  downstream only uses as a sunlight mode, and the init sequence sets the
  normal 80%. The driver now sets the normal level at boot too, and the
  backlight LEDs run at the 25 mA per string that the downstream driver
  uses on this PMIC revision instead of a cautious 20 mA, so the screen is
  as bright as stock and stays the same across boot, mode switches and
  screen off/on.
- **Panel health check.** Like the downstream driver, the panel driver
  reads the panel's power mode every 5 seconds and power-cycles the panel
  if it stops answering correctly twice in a row, for example after an
  electrostatic discharge.

Plasma's night light works on top of this. There is no factory colour
profile to load: the downstream panel description only has rounded design
values, an sRGB gamut with a white point around 7500-8000 K (x 0.290,
y 0.310) and a 420-nit peak. A small script turns them into an ICC display
profile; its calibration curves bring the white to D65 by scaling green to
0.80 and blue to 0.68, and KWin applies it per screen. Whites look clearly
warmer and less blue, at the cost of about 20% of the light; it's a generic
correction from design values, not a calibration of this particular panel.
DisplayPort over USB-C is not an option on this phone: the port is USB 2.0
only.

The GPU now also votes its own memory bandwidth through its management unit,
as its closest relative (A621) does upstream. A texture-copy test sized to
defeat the GPU's framebuffer compression gives the same 6-7 GB/s either
way.

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

## Modem and Wi-Fi

The Wi-Fi chip (WCN3990, driven by `ath10k_snoc`) is controlled by firmware
running on the modem DSP, so Wi-Fi needs the modem up first. That meant adding
a whole base layer to the SM7250 device tree: the IPCC mailbox, the AOSS QMP
messaging block, the modem's shared-memory state bits (smp2p) and the modem
remoteproc itself, plus the Wi-Fi node with its five supplies. The PD mapper
(the service that tells the modem where its protection domains live) needed an
SM7250 table, and the bootloader's device-tree overlay renames the board to
`qcom,lito`, so that compatible had to be matched too.

Userspace is the usual Qualcomm trio: `rmtfs` (the modem's EFS storage),
`tqftpserv` (serves firmware files to the modem) and `qrtr`. I run rmtfs
read-only from file copies of the EFS partitions, so a buggy modem can't
corrupt the real ones. A restart of the modem in the same boot only works if
`tqftpserv` is restarted and the PD mapper reloaded first, so the rmtfs unit
does both before starting. NetworkManager with the iwd backend manages Wi-Fi;
systemd-networkd keeps the USB interface. The real Wi-Fi MAC address comes
from the bootloader's `wifimacaddr=` command-line argument via a `.link` file.

### The modem restarting every minute

For a while the modem firmware's watchdog fired exactly 65 seconds after every
start ("DOG detects stalled initialization"), taking Wi-Fi down with it. Two
things were missing, and each hid the other:

- **A firmware file.** The `tqftpserv` log showed the modem asking, over and
  over, for `/readonly/firmware/image/modem_pr/so/819_0_0.mbn`: a "software
  options" file that lives in the modem partition next to the main firmware.
  Serving the `modem_pr` tree from the modem's firmware folder got it past that
  point, straight into the next problem.
- **The IPA driver.** IPA is the hardware block that carries mobile data
  between the modem and Linux, and the modem firmware waits for the AP's IPA
  driver to complete a QMI handshake during startup. Without it the modem now
  asserted three seconds after starting (`ipa_hwp_init ... didnt rx any ind frm
HWP`). SM7250 has the same IPA v4.7 as SM6350, which mainline already
  supports, so the driver needed only a new entry with a different IMEM address
  for the modem tables, plus a device-tree node with the SM7250 SMMU stream IDs.
  The value I first took from downstream for the modem's shared-memory region
  was wrong; the driver's own check ("SMEM item has size 36864, expected
  26624") showed the right one.

The order matters: the IPA driver has to be loaded **before** the modem starts,
because it gets the IPA microcontroller's "ready" signal through the modem's
startup. Loaded after, the handshake never completes. At boot this is now
automatic: `ipa` is in `modules-load.d`, the rmtfs unit (which starts the
modem) runs after `systemd-modules-load`, and the `modem_pr` files ship with
the rest of the firmware. With IPA loaded first and
the files served, the modem stays up once it is running.

What remains is a race inside the modem firmware. On roughly every other boot
the modem stops very early in its first start: it publishes only its first two
QRTR services, never says hello to Linux's name service and never asks rmtfs for
its EFS, and its watchdog restarts it 65 seconds later. Debug output in the
QRTR name service showed that Linux's side is complete at that point, and
sending Linux's hello a second time didn't wake the modem up either. Its second
start has worked every single time, so a small boot service now checks the
modem 15 seconds after it starts and restarts it right away if it is stuck; on a
bad boot the modem and Wi-Fi are up after about 35 seconds instead of 70. Along
the way I made the QRTR name service carry on after a failed announcement
instead of silently dropping every service after it.

### Wi-Fi board data

The WCN3990 needs board data (RF calibration, power limits, regulatory code)
from the host. The chip reports board ID `0xff`, so ath10k used the generic
entry from `linux-firmware`: a "world" regulatory code with every 5 GHz
channel passive until an access point announced its country.

Motorola's own files sit next to the modem firmware, `bdwlan_nairo_ipa.bin`,
`bdwlan_nairo_ipae.bin` and `bdwlan_nairo_ipav.bin`, and on Android
`cnss-daemon` picks one from the boot arguments. Its table maps radio `NA` to
`ipa`, `VZW` to `ipav` and `EMLA` to `ipae` (anything else `ipa`), and it
switches to a separate `bdwlanu_*` set only for chip ID `0x4320` (this one
reports `0x320`). I found that by reading the strings and one comparison in
the stock vendor image's `cnss-daemon`.

ath10k looks board data up by name in `board-2.bin`. A
`qcom,calibration-variant = "motorola_nairo"` in the device tree makes it ask
for `bus=snoc,qmi-board-id=ff,qmi-chip-id=320,variant=motorola_nairo`; a small
script wraps each Motorola file into a one-entry `board-2.bin`, and the
initramfs installs the one matching `androidboot.radio` under
`/usr/lib/firmware/updates/`, which the kernel searches before the
`linux-firmware` copy.

Motorola's board data leaves the regulatory code at 0, which the `ath` code
treats as "US"; with the system's regulatory domain set to GB
(`/etc/conf.d/wireless-regdom`), cfg80211 applies the stricter of the two, so
the effective rules are the UK ones and the 5 GHz channels outside the radar
bands no longer need passive scanning.

## Making UFS reliable

Before moving Arch onto UFS I backed up every partition table and the small
LUNs. The very first `dd` killed the controller: the link failed to come out of
hibernate (`pwr ctrl cmd 0x18 ... upmcrs:0x5`) and every LUN returned I/O
errors until a reboot.

Keeping the link awake through sysfs (no runtime suspend, no auto-hibernate,
no clock gating) got the backup done, but showed a second pattern: physical
errors on **both lanes at once** (`pa_err 0x80000003`, data-link replay
timeouts), about one every few seconds, and only while Wi-Fi was associated.
The driver recovered from most of them by resetting the link.

The dangerous part was what happened when it needed a full host reset: the PHY
never came back (`phy initialization timed-out`). A debug dump added to the PHY
driver showed the SerDes PLL not locking, or taking up to a second instead of
well under a millisecond. Forcing that path on demand (writing
`CONTROLLER_FATAL_ERROR` to the UFS debugfs `saved_err` file) made it
reproducible. Along the way I ruled out the PHY init tables (identical to
downstream, values and register offsets), clocks, supplies and the link gear.

The cause was a power rail. Downstream votes an RPMh resource called
`qphy.lvl` whenever the UFS PHY is powered; mainline never did, so the PHY
lived on other subsystems' votes, most likely the modem's. Every modem crash
or restart took the rail away for a moment. With `qphy.lvl` voted on
permanently (mainline already has the power domain for other SoCs), forced
resets re-lock the PLL in under 10 ms, and long direct reads across a modem
restart, a modem crash and a Wi-Fi firmware recovery show no UFS error at all.
I also kept downstream's PHY reset sequence (assert the PCS software reset
before writing the tables, allow up to 1 s for the PLL). A 4 GiB random-data
write with direct I/O read back identically.

With the rail fixed, the "keep the link awake" workaround could go again:
with the driver's defaults (clock gating with hibernate, auto-hibernate after
150 ms, runtime suspend) the link went through hundreds of hibernate cycles
without an error. And one more mapping fix: mainline turns a 150 MHz UniPro
clock into HS-G2, which suits SoCs whose top level is 300 MHz, but SM7250 runs
HS-G3 at 150 MHz. With that, clock scaling reaches HS-G3 rate B on two lanes
under load and direct reads go from 434 MB/s to 612 MB/s.

Two bad turns worth knowing about: a change that mapped the 150 MHz UniPro
clock to HS-G3 made the phone crash into Qualcomm's 9008 download mode during
boot (Power + Volume Down into fastboot recovers it), and a phone that loops
on boot is far easier to rescue if you've kept the last good boot image.

## Moving Arch to UFS

The plan was to split userdata in two: 50 GiB for LineageOS, the rest for
Arch. The GPT edits themselves were easy, but Motorola's bootloader (ABL is
Motorola's own "MotoBootModule") **loses every partition on every LUN** as soon
as the table differs from stock in anything but the end of the last partition:
no boot logo, `slot-count: 1`, no partition sizes in fastboot, and nothing to
boot. Every variant failed: 128 entries, 28 entries, reusing the unused `padA`
entry behind userdata, and putting Arch in front of a shifted userdata. Only
shrinking userdata, with nothing after it, was accepted. The boot-time check is
not the signed-GPT validation (that only runs when flashing `gpt.bin`); I
didn't find the exact rule.

Two things made the experiments safe:

- **The stock `gpt.bin`** from the matching firmware. `fastboot flash partition
gpt.bin` rewrites all six LUN tables to the factory layout in a second,
  leaving partition contents alone.
- **Never letting sfdisk write the table.** Motorola sets vendor attribute bits
  (3 to 6) on most partitions, and sfdisk silently drops them. I edited the
  table bytes in place instead, recomputing only the CRCs.

In the end I gave all of userdata to Arch: ext4 on `sde27` (still named
`userdata`, so the partition table is exactly stock) and an rsync from the SD
card. The initramfs now also accepts `nairo.root=PARTLABEL=<GPT name>`: with no
udev in there, it finds the partition by scanning the `PARTNAME=` lines in
`/sys/class/block/*/uevent`, waiting for the UFS LUNs to appear. The command
line says `nairo.root=PARTLABEL=userdata`. TRIM works. LineageOS is gone until
the next stock flash, and the SD card stays as a fallback root.

## Rebooting into fastboot, and A/B slots

Two small things made testing much faster.

**`reboot bootloader`.** Motorola's bootloader wants three things before it
stops in fastboot: the standard Qualcomm restart reason in the PMIC
(`SOFT_RB_SPARE`), a Motorola "reboot_bl" bit in a second PMIC spare register
(`DVDD_RB_SPARE`, bit 2) and a cold reset. Mainline already does the cold
reset, but the PMIC reason was never written because the PON node had no
`mode-bootloader`/`mode-recovery` properties. With those, and a small patch to
`qcom-pon` that also sets Motorola's bit for the bootloader mode,
`systemctl reboot --reboot-argument=bootloader` lands in fastboot in about
eight seconds.

**Marking the slot successful.** The bootloader keeps the A/B slot state in
GPT attribute bits of `boot_a`/`boot_b` (priority, active, a retry counter,
successful, unbootable). Android marks a slot successful after every good
boot; nothing did on Arch, so after a handful of reboots without
`fastboot set_active` the retries ran out and the phone silently switched to
slot a. A small tool now sets the "successful" bit on the active slot's entry
(both GPT copies, nothing else), run by a timer once `systemd-analyze` reports
that boot has completed, so a boot that hangs still costs a retry.

## The modem joins the network

With the modem stable, ModemManager found it over QRTR and then gave up:
"SIM is required, but missing". The SIM was there; the modem's SIM card
service (QMI UIM) showed the card `present` and its USIM application
`detected`, but never `ready`, and no "provisioning session" existed. The
modem itself sat in `shutting-down` and was not searching for a network.

Three things were missing, and each one only became visible once the previous
one was fixed.

**The modem's own files from `persist`.** On Android the modem reads and
writes a few files through the TFTP-over-QRTR server: SIM-lock data
(`datablock/`), RF calibration (`cal_rfs/`), the IMEI SV and some
Motorola blobs. They live on the `persist` partition under `rfs/msm/mpss/`.
Copied (read-only mount, nothing written back) into the directory
`tqftpserv` serves as `/readwrite/`, the modem reads them at every start.

**The modem configs (MCFG).** At every start the modem asks for
`/readonly/vendor/fsg/mcfg_sw/mbn_sw.dig`. That is the FSG partition, mounted
at `/vendor/fsg` on Android: Motorola's carrier configurations (`o2_gbr.mbn`
and a few hundred others) plus a digest. `tqftpserv` 1.2 rejects the path;
a six-line patch maps it to `fsg/` next to the modem firmware, where the
extracted FSG partition now lives. The modem now reads the digest and is
satisfied (it already holds 25 configurations in its own storage).

**Setting the modem online and activating the SIM.** Two QMI requests do what
Android's radio daemon does after boot:

```sh
qmicli -d qrtr://0 --dms-set-operating-mode=online
qmicli -d qrtr://0 --uim-change-provisioning-session=\
    slot=1,activate=yes,session-type=primary-gw-provisioning,\
    aid=<the USIM application ID from --uim-get-card-status>
```

Within seconds the USIM application went `ready`, and the modem registered:

```text
Registration state: 'registered'
CS: 'attached'
PS: 'attached'
Radio interfaces: 'lte'
Current PLMN: MCC '234', MNC '10', 'O2 - UK'
LTE: RSSI -66 dBm, RSRQ -11 dB, RSRP -96 dBm, SNR 4.8 dB
```

The boot-time modem check now does both after the modem has come up, as a
oneshot unit ordered before ModemManager. ModemManager then finds a ready SIM
and reports the modem `registered` on O2 (home network, LTE with a 5G NSA cell
in range), with SMS and voice available and no data bearer connected.

From there Plasma Mobile's own apps work: an outgoing and an incoming call
both connected (silent, with no audio driver), and an SMS went out from
the messaging app. During each call the packet service detaches from LTE
and reattaches afterwards: without an IMS stack there is no VoLTE, so the
modem falls back to 3G/2G for the call (CSFB).

Two side notes from the way there. Loading a carrier configuration with
`qmicli --pdc-load-config` crashed qmicli 1.38 before it sent anything: the
code `g_free()`s the contents of a `GMappedFile`, which is memory-mapped, not
allocated. With that line removed the modem answered, but it refused the
upload; it already stores 25 configurations, and that appears to be its
limit. And twice, restarting the modem (via `rmtfs`) right after swapping its
files hung the whole SoC until the watchdog reset it, with nothing in the
kernel log. After changing anything the modem reads, I now reboot.

## Mobile data

Data goes through the same IPA path that the modem needed to boot. A single
ModemManager command brings up a bearer on the SIM's default profile
(`wap.o2.co.uk`), and the modem hands out an IPv4 address on a QMAP mux
interface on top of `rmnet_ipa0`:

```sh
mmcli -m 0 --simple-connect="profile-id=2,ip-type=ipv4"
```

```text
connected: yes
interface: qmapmux0.0
  address: 10.167.248.27
   prefix: 29
      dns: 82.132.254.2, 82.132.254.3
```

With the address set on `qmapmux0.0`, five pings to 1.1.1.1 came back in
25–34 ms over LTE, with no loss. The SIM is pay-as-you-go, so that was the
whole test: connect, ping, disconnect. Incoming SMS also work: a text sent
to the phone appeared in Plasma Mobile's messaging app.

Between this test and the previous section, the phone spent a day locked,
running LineageOS signed with my own keys. That story is in
[its own post]({% post_url 2026-10-09-relocking-moto-g-5g-plus-avbroot %}).
Coming back to Arch meant unlocking the bootloader again, which wipes the
data partition; the Arch root came back from an older copy on the SD card.
That copy predated the modem work, so the patched `tqftpserv`, the modem
configs and the persist files had to be put back. With those in place the
modem registered within a minute of boot.

### Mobile data from NetworkManager

Getting the same connection up from NetworkManager, so that Plasma can
toggle it, took four fixes:

- **The APN.** O2 pay-as-you-go wants `payandgo.o2.co.uk`, and Plasma's
  automatic APN lookup can't pick it, because O2 and the virtual operators
  on its network share the same operator code. The profile sets it by hand.
- **The attach APN.** On LTE the phone already holds a data session from
  attaching to the network, with the APN in modem profile 2. Asking for
  `payandgo` while attached with `wap.o2.co.uk` fails with
  `multiple-connection-to-same-pdn-not-allowed`, and the modem rewrites
  profile 2 from its carrier configuration at every boot. So the profile
  sets the attach APN too (`gsm.initial-eps-bearer-*`), and ModemManager
  updates it before each connect: a few seconds of detach and reattach, and
  no data.
- **The authentication methods.** By default NetworkManager asks for every
  method for the attach bearer: none, PAP, CHAP, MSCHAP, MSCHAPv2 and EAP.
  A QMI modem stores only PAP and CHAP, so ModemManager reads back a
  different setting and fails with "requested and reloaded initial EPS
  bearer settings don't match". Refusing the other methods fixes it.
- **A ModemManager crash.** ModemManager 1.24.2 segfaulted while setting up
  the data interface: it freed a pending netlink request and then called
  its callback. That was the "reconnect" crash from the first test. It is
  fixed upstream
  ([merge request 1509](https://gitlab.freedesktop.org/mobile-broadband/ModemManager/-/merge_requests/1509),
  4 October 2026) but not in a release yet, so the phone runs ModemManager
  and libqmi built from git.

One more trap: `nmcli c modify` on just one of the two passwords
(`gsm.password` and `gsm.initial-eps-bearer-password`) dropped the other
from the saved profile, and the connection then sat waiting for a password.
Setting both in the same command keeps both.

The profile, without autoconnect because of the pay-as-you-go tariff:

```sh
nmcli c add type gsm ifname '*' con-name PayAndGo \
  gsm.apn payandgo.o2.co.uk gsm.username payandgo gsm.password password \
  gsm.initial-eps-bearer-configure yes \
  gsm.initial-eps-bearer-apn payandgo.o2.co.uk \
  gsm.initial-eps-bearer-username payandgo \
  gsm.initial-eps-bearer-password password \
  gsm.initial-eps-bearer-noauth no \
  gsm.initial-eps-bearer-refuse-mschap yes \
  gsm.initial-eps-bearer-refuse-mschapv2 yes \
  gsm.initial-eps-bearer-refuse-eap yes \
  ipv4.method auto ipv6.method disabled connection.autoconnect no
```

`nmcli c up PayAndGo` then brings up `qmapmux0.0` with its address, route
and DNS servers, and three pings to 1.1.1.1 came back in 25–64 ms.
NetworkManager warns that it can't set the 1500-byte MTU the modem
reports, because the IPA interface underneath is limited to 1496 bytes;
the connection works anyway.

### Patches and upstream reports

Everything the phone's userspace needed is in
[nairoArchLinuxArmTools](https://github.com/effeffe/nairoArchLinuxArmTools),
under `userspace/`: the `tqftpserv` patch for the FSG path, the `qmicli`
fix, how the fixed ModemManager is built, and a script that creates the
NetworkManager profile above (`nm-mobile-profile.sh`, with the APN as an
argument for other networks). Six upstream reports came out of this, and
their drafts are in `upstream-reports/`:

- **NetworkManager:** `nmcli connection modify` on one gsm password drops
  the other passwords from the saved profile. `nmcli` doesn't fetch the
  stored secrets before saving, and the daemon only merges them back when
  an update carries no secrets at all.
- **ModemManager:** setting the attach APN on a QMI modem fails whenever
  the request includes authentication methods other than PAP and CHAP.
  ModemManager drops the extra methods when writing, then compares what it
  reads back against the original request.
- **ModemManager:** the development branch needs libqmi 1.40 but its build
  accepts 1.37.95, so the build fails halfway instead of at the dependency
  check.
- **libqmi:** `qmicli --pdc-load-config` frees memory owned by a
  `GMappedFile` and crashes before sending anything. The fix is one line.
- **tqftpserv:** a patch to serve `/readonly/vendor/fsg/`, where this
  modem looks for its carrier configurations.
- **plasma-nm:** editing a mobile connection in Plasma's network settings
  drops all of NetworkManager's `gsm.initial-eps-bearer-*` settings, which
  breaks the profile above.

The ModemManager crash itself needed no report: it was fixed upstream a
few days before I hit it.

## Off-mode charging

Plugging the charger into a phone that is off doesn't start the system: the
bootloader boots the same boot image with `androidboot.mode=charger` on the
kernel command line ("Entering charging mode" in its log). Android shows a
charging animation and nothing else. Until now, Arch simply booted.

The charging screen lives in the initramfs, in the small C program that is
its `init`. When it sees `androidboot.mode=charger` it never looks for the
root filesystem. Nothing is mounted and nothing is written, so a hard reset
can't harm anything. It:

- loads the charger and fuel-gauge modules, which now ship in the initramfs
  for this purpose only;
- switches the console to `KD_GRAPHICS`, so fbcon stops drawing, and draws
  a battery on the bootloader framebuffer through simpledrm's `/dev/fb0`:
  the charge level (red below 15%, amber below 30%), the percentage in a
  5×7 bitmap font scaled up 20 times, and a lightning bolt while charging;
- turns the backlight off after 20 seconds; a press of the power key lights
  it again;
- reboots when the power key is held for 5 seconds. A warm reboot doesn't
  enter charger mode again, so the phone comes up in Arch;
- powers the phone off when the charger has been unplugged for 10 seconds.

The USB gadget and SSH still come up, which made it easy to debug. Once
there is a real display driver, the screen can switch the panel off rather
than just the backlight, and the drawing can move to the GPU.

## Suspend

Until now the kernel booted with `clk_ignore_unused` and `pd_ignore_unused`,
which keep every clock and power domain the bootloader left on running for
as long as the phone is up. That made bring-up easy, but a SoC with half its
clocks running never reaches its low-power states. Dropping them took three
fixes:

- **Clock controllers that never let go.** A clock controller only turns off
  what nobody uses once all its consumers have probed (`sync_state`). The
  GPU's management unit has no driver, so the global and GPU clock
  controllers waited forever and kept everything on. Booting with
  `fw_devlink.sync_state=timeout` makes them stop waiting after a timeout.
- **The boot splash.** Without `clk_ignore_unused`, the kernel switches off
  every clock nobody has claimed shortly after boot, including the ones
  feeding the framebuffer the bootloader set up. The display clock
  controller was a module and loaded too late for the framebuffer driver to
  claim them; it is built in now.
- **USB after resume.** The USB controller came back dead after every
  suspend (`failed to enable ep0out`). On this board the controller is set
  to take its PHY clock from the USB 2.0 PHY, and that setting is lost
  while it sleeps. The driver now sets it again on resume.

The SoC's always-on processor keeps sleep statistics, which mainline can
read once the device tree points at them. They show where the time goes:
in a 20-second suspend the whole chip, including the DDR, spends 17-20
seconds in its deepest state. The phone suspends to idle (`s2idle`), the
only mode this firmware offers to Linux. Plasma now suspends it on its own
when idle, the power key wakes it, and USB, Wi-Fi, the modem, the display
and touch all come back.

### Waking up for calls and texts

The first test of an incoming call failed in two ways. First,
ModemManager forgot the modem: by default it drops every modem when the
system suspends and searches again on resume, but this modem stays
powered in suspend and is never announced again, so after the first
suspend ModemManager reported no modems at all. Its
`--test-quick-suspend-resume` option, made for modems that stay on, keeps
the modem across a suspend; a systemd drop-in in the tools repository
enables it.

Second, the modem could not wake the phone. Messages from the modem arrive
as an interrupt, and the kernel keeps that interrupt enabled during suspend
so the link keeps working. An interrupt that stays enabled is never counted
as a wake-up, though: the CPU woke up, handled the message, and went
straight back to sleep with ModemManager frozen, so the call rang on the
caller's side and nothing happened on the phone. The downstream kernel
makes every message from any coprocessor wake the system. A kernel patch
now lets the modem's link report a wake-up, off by default and turned on
for the modem only by a udev rule. The first two versions were wrong:

- reporting a wake-up for every message cancelled every suspend: while the
  system is going to sleep, ModemManager and other drivers are still
  talking to the modem (66 messages in one attempt), so the screen came
  straight back on. Wake-ups are now only reported once the system is
  asleep;
- the first version crashed the kernel whenever the modem restarted,
  because the wake-up source is registered as a child of the link, and the
  link's teardown removes all of its children before removing it a second
  time. It is now removed first.

With both in place an incoming call and an SMS each wake the phone, it
rings and the notification appears, and the modem otherwise left the phone
asleep for ten minutes straight.

## Audio

On this SoC the CPU never talks to the codec directly. Audio goes through
the audio DSP (ADSP): Linux streams PCM to one of the DSP's front ends, and
the DSP routes it to a backend port. The earpiece, the headset jack and the
two microphones hang off a WCD9380 codec, reached through the LPASS RX and
TX macros and SoundWire; the loudspeaker has its own amplifier on a
secondary I²S link (MI2S). During a call the DSP exchanges the audio with
the modem itself, and Linux only sets the path up.

### Starting the audio DSP

The device tree gained the ADSP, its services (core, AFE, ASM and ADM
routing), the LPASS codec macros, both SoundWire buses and the LPASS pin
controller, which turned out to be SM6350's. The DSP then loaded, came out
of reset and never reported ready. The downstream kernel failed the same
way when I started the ADSP by hand from LineageOS's recovery, which ruled
out the mainline loader. The answer was in the modem partition: it carries
two ADSP images, `adsp` and `adsp2`, and a fuse (`adsp_variant`) decides
which one a phone runs. This one needs `adsp2`. With it, the DSP comes up
its four services register and the WCD9380
binds on both SoundWire buses. (The compute DSP, which audio doesn't need,
is still refused by TrustZone with `-EINVAL`.)

### Earpiece and microphones

The sound card registered once the DSP's stream front ends were in the
device tree. Each front end is one DSP session that serves both
directions, so playback and recording on the same one collide ("Audio
Client already active"): every front end is now fixed to one direction.

A test tone played straight to the card came out of the earpiece; the
same tone through PipeWire was silent, although its stream ran without errors.
Three tones in a row, one per sample format, settled it: 16-bit samples
play, and 24- and 32-bit ones, which PipeWire prefers, play as silence.
The ADSP firmware's wider formats don't work here, so a WirePlumber rule
opens the card as `S16LE`.

The rest was getting the mixer right, in a UCM profile (the description
of the card that PipeWire reads):

- The **earpiece** is a single transducer on the left channel, so the
  right-channel test in Plasma was silent. Both channels are now mixed
  into it, (L + R) / 2.
- It was also far too quiet at 100%: the codec's `EAR_PA Volume` counts
  _down_, from +6 dB at 0 to -18 dB at 16, and the profile had it the wrong
  way round.
- The **microphones** are at the bottom (AMIC1) and the top (AMIC3), next
  to the earpiece. They are read as one stereo stream, and a PipeWire
  loopback splits it into "Bottom Microphone" and "Top Microphone"
  sources, so Plasma offers both separately as well as the stereo pair.
- `ADC1 Switch` and `ADC2 Switch` look like mute switches, but they
  connect the microphones to their SoundWire ports. When PipeWire was
  allowed to drive them as the source's mute, it disconnected the
  microphone; they are now set once and left alone.

### The loudspeaker: a new amplifier driver

The loudspeaker amplifier answers on I²C with chip ID `0x1852`: an Awinic
AW882xx from a generation that none of the mainline Awinic drivers
support. I wrote a new ASoC codec driver for it, following the downstream
one:

- it loads the register table Motorola ships as firmware
  (`aw882xx_spk_reg.bin`, 36 register writes), sets the current and
  voltage sensing gain from the chip's efuses, and only unmutes once its
  PLL has locked onto the I²S clocks;
- a **volume control**, 0.5 dB steps from -90 dB to 0 dB. The first
  version had none, and the first test tones came out at full gain, very
  loud;
- a **channel select**: downstream sends the speaker a single mono
  channel, so the driver plays the average of left and right by default;
- like downstream, a **monitor** reads the battery voltage and the die
  temperature every 3 seconds while playing, and below Motorola's
  thresholds (3.9 V, and 5 °C) adds attenuation and lowers the boost
  converter's peak current.

The DSP's speaker protection, which downstream feeds with the amplifier's
current and voltage readings over a second I²S line, isn't implemented.

The loudspeaker, the earpiece and the headphones share one front end.
PipeWire turns every output of the UCM profile into its own sink, and
outputs that share a front end into separate profiles, so the profile
order decides the default output: the loudspeaker comes first. I also
tried the loudspeaker and the earpiece as a stereo pair, as some phones
do. It works through a PipeWire combined sink, but downstream never plays
media on the earpiece, and the configuration stays in the repository,
disabled.

### Vibration

The vibration motor is a linear resonant actuator driven by an Awinic
AW8695, which has no mainline driver either. The new input driver plays
force-feedback rumble effects in the chip's continuous mode, which drives
the motor at its resonant frequency and follows it through zero-crossing
detection; the drive levels and the frequency come from Motorola's device
tree, where they are tuned per phone. The downstream waveform library and
calibration are not implemented.

### Call audio

Mainline has no driver for the DSP's voice services (MVM, CVS, CVP), the
part that connects a call to the codec. The sdm845-mainline kernel and
postmarketOS carry one, `q6voice`, written by Stephan Gerhold, Richard
Acayan, Joel Selvaraj, Vincent Knecht and Otto Pflüger, including the
start-up sequence that newer DSPs like this one need (CVD 2.3). I took
their commits as they are and added the codec DMA ports this SoC uses.
Two userspace pieces drive it:

- **q6voiced**, from postmarketOS, opens the voice "PCM" while
  ModemManager reports a call. No samples pass through it: opening it is
  what starts the voice sessions in the DSP. I ported it from tinyalsa to
  alsa-lib, and it retries the open, because the first attempt races the
  switch of the card to its call profile;
- **callaudiod**, which Plasma Mobile's dialer already uses, switches the
  card to the "Voice Call" profile of the UCM description.

The first calls connected with no sound in either direction. Three things
were wrong:

- The machine driver forced every port to two channels, while the voice
  processor was set up, as downstream does for single-microphone calls,
  for a mono uplink. The capture ports now take their channel count from
  the stream that opens them: one channel for a call, two for a stereo
  recording.
- PipeWire was switching off `ADC1 Switch` (above) as part of the call
  profile's microphone volume.
- The DSP's default uplink processing (echo cancellation and noise
  suppression) needs calibration data that Android loads from userspace
  and Linux doesn't have, and without it the uplink stays silent. Calls
  now use the processing topology "none".

With these, both sides could hear each other. Speakerphone and mute
didn't work, for one reason: callaudiod drives both through PipeWire, but
the call's audio never passes through PipeWire.

- **Speakerphone:** callaudiod switches the port of the call's sink, but
  the loudspeaker is in a different profile from the earpiece (see
  above), never a port of the same sink. A patch to callaudiod switches
  the profile instead. In the kernel, moving a running call to the
  loudspeaker used to leave the voice processor attached to the earpiece's
  port, and the call went silent; it is now restarted on the new port.
- **Mute:** muting PipeWire's microphone source mutes nothing, since the
  microphone goes straight into the DSP. The voice driver now has mute
  switches that send the DSP's own mute command, and the UCM profile names
  them as the volume elements of the call devices, so PipeWire's mute
  mutes the call.
- **Hanging up:** a call that ended while callaudiod was still switching
  profiles could leave the card stuck in the call profile. The patch
  fixes that too.

The tools repository has the UCM profile, the PipeWire and WirePlumber
configurations, q6voiced and the callaudiod patch, under `userspace/`. The
amplifier and voice modules are still loaded by hand after boot, as every
new driver is at first.

## What doesn't work yet

- **Display:** no content-adaptive backlight (CABC), which the downstream
  driver switches off at low brightness, something the mainline backlight
  driver can't do yet.
- **Bluetooth:** the other half of the WCN chip, over UART; not started.
- **Modem:** registers on LTE at boot and ModemManager sees it. On about half
  the boots its first start stalls and is restarted after 15 seconds. The IPA
  data path works: calls (in and out), SMS (in and out) and mobile data
  from NetworkManager. No VoLTE.
- **Audio:** no in-call volume control yet, and the dialer can show the
  microphone as muted when a call starts. Calls have no echo cancellation
  or noise suppression (no calibration data), the loudspeaker has no DSP
  speaker protection, and music can't stay on the loudspeaker while a
  call is on the earpiece (a stream feeds one output at a time). The
  amplifier and voice modules are loaded by hand.
- **Vibration:** rumble only, without the downstream waveform library.
- **Cameras, sensors.**
- **UFS:** 32-bit DMA until the SMMU stream ID is known.
