---
layout: post
title: Relocking the Moto G 5G Plus bootloader with LineageOS and avbroot
date: 2026-10-09
description: How I got LineageOS 23.2 running on a Moto G 5G Plus (nairo) with a relocked bootloader and my own verified boot key, using avbroot, and the four device-specific problems the avbroot guide doesn't cover.
tags: android lineageos avb bootloader security
categories: projects
toc:
  sidebar: left
---

The Moto G 5G Plus (XT2075-3, **nairo**) lets you register your own Android
Verified Boot (AVB) key and lock the bootloader again. The phone then boots an
OS signed with that key and shows the yellow "different operating system"
warning instead of the orange "unlocked" one. Verified boot stays enforced and
the bootloader can't be used to flash unsigned images, but you still run the OS
of your choice.

DISCLAIMER: READ THE ENTIRE POST BEFORE STARTING! It's not too long, but if you make a mistake you could leave the phone in a bricked locked state!

[avbroot](https://github.com/chenxiaolong/avbroot) is the tool for this: it
re-signs an A/B OTA zip with your own keys. Its guide is written with Pixels
in mind. On nairo with LineageOS it took four fixes that the guide doesn't
mention: one of them is an avbroot bug that makes recovery unbootable. This
post is the full procedure, from key generation to the locked boot, with every
step that was needed on this phone.

End state:

```
$ adb shell getprop ro.boot.verifiedbootstate
yellow
$ adb shell getprop ro.boot.vbmeta.device_state
locked
```

Versions used:

- bootloader `MBM-3.0-nairo_retail-ab2c614f64c-220629` (the last stock
  firmware, Android 11, RPNS31.Q4U-39-27-9-2-9)
- LineageOS 23.2 nightly, 2026-10-05 (`lineage-23.2-20261005-nightly-nairo-signed.zip`)
- avbroot 3.34.1, built from source (`cargo build --release`)
- fastboot 35 or newer (older versions break `flashall`)

**Warning.** Locking with images the bootloader doesn't accept leaves you with
a phone that boots neither Android nor recovery. You get out of that only if
the _OEM unlocking_ flag is set (see [below](#the-oem-unlocking-switch)).
Everything here wipes userdata more than once.

## Why LineageOS and not stock

avbroot patches OTA zips that contain a `payload.bin` (the A/B update
format). Motorola's stock firmware for nairo is only distributed as a
fastboot package: `boot.img`, `vbmeta.img`, `super.img_sparsechunk.*` and a
`flashfile.xml`, no payload. avbroot can pack a payload (`avbroot payload
pack`, `avbroot zip pack`), so building one from the stock images is possible,
but there's little point: Android 11 is the last stock release for this phone,
so there will never be a stock OTA to sign, and stock firmware already boots
with a locked bootloader under Motorola's own key.

LineageOS publishes regular OTA zips with a payload, so the normal avbroot
flow applies. The zip contains:

```
META-INF/com/android/metadata(.pb), otacert
apex_info.pb, care_map.pb
payload.bin, payload_properties.txt
```

and the payload holds 11 partitions: `boot dtbo odm product recovery system
system_ext vbmeta vbmeta_system vendor vendor_dlkm`.

## Keys

This part follows the avbroot README. Two RSA-4096 keys, one for AVB
(vbmeta and boot images) and one for the OTA (payload and zip):

```sh
avbroot key generate-key -t rsa4096 -o avb.key
avbroot key generate-key -t rsa4096 -o ota.key
avbroot key encode-avb -k avb.key -o avb_pkmd.bin   # public key for the bootloader
avbroot key generate-cert -k ota.key -o ota.crt     # cert that recovery checks OTAs against
```

Both private keys are passphrase-protected; avbroot asks for the passphrases
on every patch. Back them up. Losing `avb.key` means unlocking (and wiping)
to ever update again.

## Unlocking and registering the key

The bootloader was already unlocked from the mainline Linux work (Motorola
unlock code from their portal, via `fastboot oem get_unlock_data`). With the
phone in the bootloader:

```sh
fastboot flash avb_custom_key avb_pkmd.bin
```

```
Warning: skip copying avb_custom_key image avb footer (avb_custom_key partition size: 0, avb_custom_key image size: 1032).
Sending 'avb_custom_key' (1 KB)                    OKAY [  0.001s]
Writing 'avb_custom_key'                           OKAY [  0.009s]
```

The warning is harmless: `avb_custom_key` isn't a real partition, so
`getvar partition-size` returns nothing. The bootloader's fastboot handler
stores the key elsewhere (more on that [later](#does-the-bootloader-support-custom-keys)).

## Patching the OTA

The basic patch is the README's, plus two things:

```sh
avbroot ota patch \
    --input lineage-23.2-20261005-nightly-nairo-signed.zip \
    --key-avb avb.key --key-ota ota.key --cert-ota ota.crt \
    --rootless \
    --clear-vbmeta-flags
```

- `--rootless`: no Magisk. I only want the locked bootloader. The phone keeps
  LineageOS's own `adb root` (a userdebug build).
- `--clear-vbmeta-flags`: LineageOS ships its root `vbmeta` with
  `flags = 3`, which turns verification off. avbroot refuses that (`Verified
boot is disabled by vbmeta's header flags: 0x3`), since a locked phone
  with verification disabled makes no sense. The option sets the flags to 0.

This patch is **not** bootable on nairo yet: problems 3 and 4 below need two
replacement images. The final command is [at the end](#the-final-patch-command).

## Problem 1: flashing only some partitions

The README extracts and flashes the patched images like this:

```sh
avbroot ota extract --input ota.zip.patched --directory extracted --fastboot
ANDROID_PRODUCT_OUT=extracted fastboot flashall --skip-reboot
```

Without `--all`, `extract` writes only the images that avbroot might change:
`boot recovery system vbmeta vbmeta_system`. The README assumes the phone
already runs exactly the same, unpatched build (its step 3: "the device must
already be running the correct OS"). Mine was coming from stock and Arch
Linux, so `dtbo`, `odm`, `product`, `system_ext`, `vendor` and `vendor_dlkm`
were all different from what the new `vbmeta` describes. The new `vbmeta`
contains hash descriptors for all of them and has verification enabled, so the
phone didn't boot, not even into recovery.

Fix: extract everything.

```sh
avbroot ota extract --input ota.zip.patched --directory extracted --fastboot --all
```

## Problem 2: partitions missing from super

With everything extracted, `flashall` failed halfway through:

```
Rebooting into fastboot                            OKAY [  0.001s]
< waiting for any device >
Sending 'odm' (1704 KB)                            OKAY [  0.043s]
Writing 'odm'                                      FAILED (remote: 'No such file or directory')
```

`product`, `system` and `vendor` flashed fine, but `odm`, `system_ext` and
`vendor_dlkm` failed the same way. These are logical partitions inside
`super`, and the table in `super` still came from Motorola's Android 11,
which has none of the three. A normal `flashall` from a build tree starts by
writing `super_empty.img` (the empty partition table) to set up the layout,
but an OTA zip doesn't contain one, so the `fastboot-info.txt` that avbroot
writes skips that step.

LineageOS publishes `super_empty.img` for nairo next to the OTA download.
`lpdump` shows it's the right table: groups `moto_dynamic_partitions_a/_b`
with a maximum size of 4861198336 bytes each (the same size as the OTA's
`dynamic_partition_metadata`), and `odm system system_ext product vendor
vendor_dlkm` for both slots.

From fastbootd (the phone was still there after the failed `flashall`):

```sh
export ANDROID_PRODUCT_OUT=extracted
fastboot wipe-super super_empty.img
for p in odm product system system_ext vendor vendor_dlkm; do
    fastboot flash $p extracted/$p.img || break
done
fastboot -w          # erase userdata and metadata; flashall stopped before doing it
fastboot reboot
```

`wipe-super` empties every logical partition on both slots, so all six have
to be flashed again. Note that `-w` also never ran, because `flashall` stopped
at the error. With this, the patched LineageOS booted (still unlocked, so in
orange state).

## Problem 3: avbroot breaks the recovery image

The first image that booted was patched with two extra options,
`--skip-system-ota-cert --skip-recovery-ota-cert`. These stop avbroot from
replacing `otacerts.zip`, the list of certificates that system and recovery
accept OTAs from. Without the replacement, recovery only trusts the LineageOS
key and will refuse every OTA I sign myself, so it's not a usable setup.

Without those options the phone looped at boot and never got as far as
fastbootd. On nairo, fastbootd runs from the `recovery` partition, so recovery
was the suspect. Comparing the patched and original images:

```
$ avbroot boot info -i recovery.img          # original, from the payload
- Ramdisk size:         17299273
- Recovery dtbo size:   3457024
- Recovery dtbo offset: 59400192
$ avbroot boot info -i extracted/recovery.img   # patched
- Ramdisk size:         17878936
- Recovery dtbo size:   3457024
- Recovery dtbo offset: 59400192
```

nairo's recovery is a version 2 boot image. Versions 1 and 2 of the header
carry a `recovery_dtbo_offset`: the absolute offset in the image of an
overlay that the bootloader applies when it boots into recovery. Adding the
certificate makes the gzip ramdisk 579663 bytes larger, so everything after
it moves, but the offset in the header stays the same. With 4096-byte pages:

```
header + kernel + ramdisk (original): 4096 + 42094592 + 17301504 = 59400192
header + kernel + ramdisk (patched):  4096 + 42094592 + 17883136 = 59977728
```

and the bytes at each offset confirm it (`d7b7ab1e` is the DTBO table magic):

```
$ xxd -s 59400192 -l 4 -p extracted/recovery.img
cb47c4ff                                  # gzip data
$ xxd -s 59977728 -l 4 -p extracted/recovery.img
d7b7ab1e                                  # the recovery dtbo
```

So the bootloader reads ramdisk bytes as the recovery dtbo. On this phone the
bootloader aborts the boot when it can't apply an overlay (I hit the same
behaviour with the mainline kernel), hence the loop. The cause is in avbroot:
`avbroot/src/format/bootimage.rs` writes the `recovery_dtbo_offset` from
the parsed header back unchanged when it repacks the image, instead of
working it out from the new sizes. A device whose recovery has no recovery
dtbo, or that has a `vendor_boot` instead, would never notice.

### Fixing the recovery image

Fix the offset in the patched recovery and feed it back to avbroot as a
replacement image. avbroot can unpack both layers, the AVB footer and the boot
image:

```sh
mkdir fix && cd fix
avbroot avb unpack -i ../extracted/recovery.img      # -> avb.toml, raw.img
mkdir b && cd b
avbroot boot unpack -i ../raw.img                    # -> boot.toml, kernel.img, ramdisk.img.0, ...
sed -i 's/^recovery_dtbo_offset = .*/recovery_dtbo_offset = 59977728/' boot.toml
avbroot boot pack -o ../raw.img
cd ..
avbroot avb pack -o recovery-otacert-fixed.img --input-info avb.toml --input-raw raw.img
```

The `avb pack` step rebuilds the AVB footer (the image's hash descriptor)
around the edited image; no key is needed because the recovery's hash is
signed through the root `vbmeta`. The result differs from avbroot's output
in 34 bytes: the offset and the hash. The value 59977728 is specific to this
recovery: work it out again for a different build (page size plus the
page-aligned kernel and ramdisk sizes).

Then patch again with:

```
--replace recovery recovery-otacert-fixed.img --skip-recovery-ota-cert
```

`--replace` makes avbroot take recovery from the file instead of the payload.
`--skip-recovery-ota-cert` stops it from adding the certificate a second time
(and breaking the offset again); the certificate is already in the
replacement. The system copy of `otacerts.zip` is still patched normally.
Check the result before flashing:

```sh
avbroot avb verify -i extracted/vbmeta.img                   # every hash and signature
xxd -s 59977728 -l 4 -p extracted/recovery.img               # d7b7ab1e
```

To test only recovery, flash `recovery_b` and `vbmeta_b` and boot into it with
the volume keys before reflashing the rest.

## Problem 4: the rollback index

Flashing the patched `vbmeta` printed a warning:

```
Writing 'vbmeta_b'                                 (bootloader) WARNING: vbmeta_b anti rollback downgrade, 0 vs 19
OKAY [  0.007s]
```

AVB rollback protection: every `vbmeta` carries a `rollback_index`, and the
bootloader keeps the highest index it has booted for each location. It refuses
anything lower, but only while locked. The stock Android 11 `vbmeta` has
`rollback_index: 19`, and the phone booted it locked for years, so 19 is
stored for location 0. LineageOS's `vbmeta` has 0. Unlocked, that's only a
warning. Locked, the bootloader would refuse it whatever key signed it. The
stored values can only grow, so the only option is to raise the image's index.

avbroot has no option for this, but it doesn't need one. When it re-signs
the root `vbmeta` (`update_vbmeta_headers()` in `avbroot/src/cli/ota.rs`), it
takes the header of the input image as is and changes only the flags, the
descriptors and the signature. So a replacement `vbmeta` with index 19 keeps
it through the patch. Building one from the original LineageOS `vbmeta`:

```sh
avbroot ota extract --input lineage-…-signed.zip --directory orig --partition vbmeta
mkdir ri && cd ri
avbroot avb unpack -i ../orig/vbmeta.img             # -> avb.toml
sed -i 's/^rollback_index = 0$/rollback_index = 19/' avb.toml
sed -i '0,/^flags = 3$/s//flags = 0/' avb.toml          # the header's flags, first match
avbroot key generate-key -t rsa4096 -o tmp.key       # any key: avbroot re-signs it anyway
avbroot avb pack -o vbmeta-rollback19.img --input-info avb.toml -k tmp.key
avbroot avb info -i vbmeta-rollback19.img | grep -E '^\s{8}(rollback_index|flags):'
```

The last line should print `rollback_index: 19` and `flags: 0`. Then add to
the patch:

```
--replace vbmeta vbmeta-rollback19.img
```

and check the extracted `vbmeta` the same way. With flags already 0 in the
replacement, `--clear-vbmeta-flags` isn't needed any more. I compared the
new extraction with the previous one: only `vbmeta.img` differed, so I
flashed just that:

```sh
fastboot flash vbmeta_b extracted2/vbmeta.img
```

To check which `vbmeta` the phone actually booted, compare
`avbroot avb digest -i extracted2/vbmeta.img` with
`androidboot.vbmeta.digest=` on the kernel command line (`dmesg | head`).

`vbmeta_system` uses rollback location 1 and LineageOS gives it index 0. I
couldn't read the stored value, but the stock firmware has no
`vbmeta_system`, so location 1 had never been used, and indeed the locked boot
accepted it.

## The final patch command

```sh
avbroot ota patch \
    --input lineage-23.2-20261005-nightly-nairo-signed.zip \
    --key-avb avb.key --key-ota ota.key --cert-ota ota.crt \
    --rootless \
    --replace recovery recovery-otacert-fixed.img --skip-recovery-ota-cert \
    --replace vbmeta vbmeta-rollback19.img

avbroot ota extract --input lineage-…-signed.zip.patched \
    --directory extracted --fastboot --all
avbroot avb verify -i extracted/vbmeta.img
```

The flashing order for a phone that isn't already running this build:
`fastboot flashall --skip-reboot` with `ANDROID_PRODUCT_OUT=extracted` for the
non-dynamic partitions. If it stops at a missing logical partition, follow
problem 2 (`wipe-super`, then the six logical partitions, then `-w`).

## Does the bootloader support custom keys?

Not every Android bootloader accepts `avb_custom_key`, and a locked
bootloader that ignores it won't boot anything. A successful `fastboot flash
avb_custom_key` only proves that the bootloader has a handler for the
command. Locking is the only real test, but the bootloader's code can show
whether the feature exists at all.

Motorola's `bootloader.img` from the stock firmware is a container holding
`abl.elf` (the bootloader app that loads Android) along with xbl, tz and
others. The stock package has the same build as the phone,
`MBM-3.0-nairo_retail-ab2c614f64c-220629`. Its strings include the standard
Qualcomm user-key code:

```
flashing avb_custom_key / erasing avb_custom_key
StoreUserKey, UserKeySize too large!
GetUserKey failed!, %r
ValidateVbmetaPublicKey OutIsTrusted %d, UserKey %d
ValidatePartitionPublicKey OutIsTrusted %d, UserKey %d
yellow
Image %a rollback index %lld is less than the stored rollback index %lld.
```

So `vbmeta` is accepted when it is signed by either Motorola's key or the user
key, and the latter boots in yellow state. Disassembling `StoreUserKey`
shows the key going into a "DeviceInfo" structure (magic `ANDROID-BOOT!`,
0x9b0 bytes): key length at offset 0xac, key at 0xb0 (2048 bytes maximum, a
1032-byte `avb_pkmd.bin` fits), and 32 rollback indexes at 0x8b0. The
structure is saved through the firmware's verified boot service
(`VBRwDevice`).

I hoped to read the key back from Android: the `devinfo` partition starts
with the same `ANDROID-BOOT!` magic. But in a root dump of it the key length
is 0, the key area is empty, and the rollback slots are all zero, though the
bootloader just reported 19. The `devinfo` partition is therefore not the
copy the bootloader uses (that one most likely lives in RPMB, which Android
can't read), and there's no fastboot command to read it back. Before locking,
you can't confirm the key was stored. What you can make sure of is that a
failed lock can be undone.

## The OEM unlocking switch

The README warns to keep _OEM unlocking_ on. On a locked phone, `fastboot
oem unlock` (or `flashing unlock`) is only accepted if that flag is set.
In LineageOS the switch is greyed out while the bootloader is unlocked
("Bootloader is already unlocked"), and on nairo it said off. Motorola's
`fastboot flashing get_unlock_ability` doesn't report it either: it only
prints the URL of the unlock portal.

The flag is the last byte of the `frp` partition, which can be read with
`adb root` on a userdebug build:

```sh
adb shell 'dd if=/dev/block/by-name/frp bs=1 skip=$(($(blockdev --getsize64 /dev/block/by-name/frp)-1)) count=1 2>/dev/null | xxd -p'
00
```

`00` means not allowed to unlock. Under developer options, oem unlock might be grayed out with the phrase "already unlocked", yet it will show a locked state, with the X rather than the tick.

The settings switch calls the system's `oem_lock` service, and with `adb
root` the shell can call it directly. The transaction codes follow the method
order of `IOemLockService`; I checked them with the two read-only calls
before writing anything:

```sh
adb root
adb shell service call oem_lock 7        # isDeviceOemUnlocked
Result: Parcel( 00000000 00000001   '........')      # true: the order matches
adb shell service call oem_lock 5        # isOemUnlockAllowedByUser
Result: Parcel( 00000000 00000000   '........')      # false
adb shell service call oem_lock 4 i32 1  # setOemUnlockAllowedByUser(true)
adb shell service call oem_lock 5
Result: Parcel( 00000000 00000001   '........')
```

The `frp` byte then read `01`, and the greyed-out switch in Developer options
showed on. If the codes return anything else on another Android version,
stop: code 4 might be a different method.

## Locking

```sh
adb reboot bootloader
fastboot flashing lock
```

Locking wipes userdata (the bootloader's own warning says so). The `frp` flag
survives. The phone then showed the yellow "Your device is loading a
different operating system" screen and booted into LineageOS setup:

```
$ adb shell getprop ro.boot.verifiedbootstate
yellow
$ adb shell getprop ro.boot.vbmeta.device_state
locked
```

The _OEM unlocking_ switch is no longer greyed out. Leave it on: it's the
way back if something goes wrong. And it costs nothing, since `oem unlock`
still needs Motorola's code.

If the lock had failed (red screen or a loop), the way back would have been
`fastboot oem unlock <code>` and a wipe. With no OEM unlocking flag set, there
would still be one way out: a locked Motorola bootloader still accepts
Motorola-signed images. The stock firmware package, flashed with its
`flashfile.xml`, boots locked (its `vbmeta` has index 19), and from there you
can turn the switch on.

## Updating LineageOS

Each new LineageOS OTA needs the same two replacement images, rebuilt from
that OTA:

1. Extract the new OTA's `vbmeta`, set `rollback_index = 19` and `flags = 0`,
   and pack it (problem 4).
2. Patch once **without** `--replace recovery` and **without**
   `--skip-recovery-ota-cert` (keep the `vbmeta` replacement), extract the
   broken `recovery.img`, and fix its offset. The right offset is
   4096 + the kernel size + the ramdisk size, each rounded up to 4096 bytes,
   from `avbroot boot info` (problem 3).
3. Patch again with both replacements and verify.
4. Sideload the result from recovery (`adb reboot recovery`, then _Apply
   update_, then `adb sideload ota.zip.patched`). Recovery now holds my
   certificate, so it accepts my OTAs and refuses LineageOS's unpatched ones.

Never install an OTA that still has rollback index 0 while locked: the new
slot would fail to verify. A/B would fall back to the old slot after a few
tries, but better not to rely on it. For the same reason, the built-in
LineageOS updater is of no use now: it downloads unpatched OTAs, which the
patched system rejects because of the OTA certificate.

## Going back

From the README: unlock (wipe), `fastboot erase avb_custom_key`, flash
whatever firmware you want. For stock Motorola firmware, use its
`flashfile.xml`.

## To report upstream

The recovery dtbo offset bug belongs in avbroot: for v1 and v2 boot image
headers, `recovery_dtbo_offset` should be computed from the new kernel,
ramdisk and second-stage sizes when the image is repacked. The other three
problems are documentation for devices like this one (the vbmeta flags, a
super layout that differs from the target build's, a stored rollback index
higher than the custom OS's).
