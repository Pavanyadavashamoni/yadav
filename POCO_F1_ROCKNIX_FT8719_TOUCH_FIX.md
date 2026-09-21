# POCO F1 ROCKNIX Touchscreen Fix — Tianma + FocalTech FT8719

This is the troubleshooting record and final working fix developed for a Xiaomi Poco F1 (beryllium) using the unofficial ROCKNIX `sdm845-beryllium` port.

## Hardware

- Device: Xiaomi Poco F1 (beryllium)
- Display: Tianma
- Touch controller: FocalTech FT8719
- ROCKNIX branch: `sdm845-beryllium`

## 1. Initial ROCKNIX installation

The original installation used:

```powershell
fastboot flash boot boot-tianma.img
fastboot flash system system.img
fastboot flash userdata storage.img
fastboot reboot
```

The phone initially failed to boot cleanly and produced an init panic. Debugging later exposed a filesystem problem on the Android `cust` partition.

## 2. Filesystem repair

The affected partition was `/dev/block/sda18` (cust). The recovery partition `sda19` was not an ext4 filesystem and was not treated as the repair target.

Check:

```sh
e2fsck -fn /dev/block/sda18
```

The filesystem was repaired and then verified clean. After this, the ROCKNIX menu became visible and touchscreen investigation could continue.

## 3. Proving the touchscreen hardware works

TWRP detected the real touchscreen as FocalTech FT8719.

The input device was `/dev/input/event2`. Running:

```sh
getevent -lt /dev/input/event2
```

while touching the display produced normal:

- `BTN_TOUCH`
- `ABS_MT_POSITION_X`
- `ABS_MT_POSITION_Y`
- `SYN_REPORT`

This proved the touchscreen hardware and basic I²C/IRQ path were functional.

TWRP also showed:

- I²C bus: 3
- I²C address: `0x38`
- IRQ GPIO: 31
- Reset GPIO: 32
- Controller ID: `0x8719`
- Firmware: valid, version 16/16

## 4. ROCKNIX failure with the original Tianma boot image

Under the original ROCKNIX Tianma image, the FocalTech driver existed but touch events were not generated.

Relevant logs included:

```text
[FTS]fts_ts_probe
[FTS][Info]display x(0 1080) y(0 2246)
[FTS][Info]max touch number:10, irq gpio:31, reset gpio:32
[FTS][Error]Pin state[release] not found
[FTS]verify id:0x8719
[FTS][Info]TP Ready, Device ID = 0x87
[FTS][Info]fw valid
[FTS][Info]fw version in tp:16, host:16
[FTS][Info]fw don't need upgrade
[FTS][Error][IIC]: i2c_transfer(write) error, ret=-107
[FTS][Error]read touchdata failed, ret=-107
```

The important issue was identified by inspecting the DTB embedded inside the boot image.

## 5. Root cause: wrong touchscreen definition in the Tianma DTB

The embedded Tianma DTB described the touchscreen as a Novatek device:

```dts
touchscreen@1 {
    compatible = "novatek,nt36672a-ts";
    reg = <0x00000001>;
    interrupts-extended = <0x0000004f 0x0000001f 0x00000001>;
    reset-gpios = <0x0000004f 0x00000020 0x00000001>;
    panel = <0x00000071>;
    iovcc-supply = <0x00000072>;
    vcc-supply = <0x00000073>;
    pinctrl-0 = <0x00000074 0x00000075>;
    pinctrl-1 = <0x00000076 0x00000077>;
    pinctrl-names = "default", "sleep";
    touchscreen-size-x = <0x00000438>;
    touchscreen-size-y = <0x000008c6>;
};
```

But the actual phone contains a FocalTech FT8719 at I²C address `0x38`.

The EBBG reference DTB contained the correct FT8719 definition:

```dts
touchscreen@38 {
    compatible = "focaltech,ft8719";
    reg = <0x38>;
    interrupts-extended = <0x4F 0x1F 0x1>;
    reset-gpios = <0x4F 0x20 0x1>;
    panel = <0x71>;
    iovcc-supply = <0x72>;
    vcc-supply = <0x73>;
    pinctrl-0 = <0x74 0x75>;
    pinctrl-1 = <0x76 0x77>;
    pinctrl-names = "default", "sleep";
    touchscreen-size-x = <0x438>;
    touchscreen-size-y = <0x8C6>;
};
```

## 6. DTB patch

The clean Tianma DTS was patched with exactly three touchscreen changes:

```text
touchscreen@1  -> touchscreen@38
novatek,nt36672a-ts -> focaltech,ft8719
reg = <0x1> -> reg = <0x38>
```

The Tianma display/panel configuration was kept unchanged.

The patched DTB packed successfully and remained exactly **151,175 bytes**, matching the original Tianma DTB size.

## 7. Rebuilding the boot image

The corrected DTB was inserted back into a copy of the original Tianma boot image at:

```text
DTB offset: 0x11516c0
```

Resulting image:

```text
boot-tianma-ft8719.img
```

Verification showed:

```text
focaltech,ft8719   = present
novatek,nt36672a-ts = absent
reg = 0x38         = present
DTB magic          = d00dfeed
```

Boot image size remained **18,317,312 bytes**.

## 8. Safe first test

The patched image was first tested without permanently overwriting the boot partition:

```powershell
fastboot boot .\boot-tianma-ft8719.img
```

ROCKNIX booted and **touch worked**.

Several screen-off/wake cycles and an extended idle test were performed; touch remained functional.

## 9. Hardware reset recovery

When the touchscreen had previously entered a failed state, the TWRP FocalTech driver exposed:

```sh
/sys/bus/i2c/devices/3-0038/fts_hw_reset
```

Executing it returned:

```text
hw reset executed
```

Immediately afterward, `getevent` again produced normal touch events.

This confirmed that the FT8719 controller itself was recoverable and that the reset path worked.

## 10. Permanent installation of the working fix

After the temporary test succeeded, the patched image was permanently written to the boot partition:

```powershell
fastboot devices
fastboot flash boot .\boot-tianma-ft8719.img
fastboot reboot
```

No system or userdata reflash was required for the touchscreen correction.

## 11. Final validation

After a real reboot from the installed boot partition:

**Touch worked.**

The patched FT8719 DTB therefore works not only with temporary `fastboot boot`, but also as the installed ROCKNIX boot image.

## Final fix summary

For a Poco F1 with **Tianma display + FocalTech FT8719 touch**, the working fix is:

1. Use the Tianma ROCKNIX boot image as the base.
2. Extract its embedded DTB.
3. Keep the Tianma display/panel configuration.
4. Change the touchscreen node:
   - `touchscreen@1` → `touchscreen@38`
   - `compatible = "novatek,nt36672a-ts"` → `compatible = "focaltech,ft8719"`
   - `reg = <0x1>` → `reg = <0x38>`
5. Pack the DTB and replace it at the original DTB offset.
6. Validate the DTB.
7. Test with `fastboot boot`.
8. Once confirmed, flash only the corrected boot image.

## Useful values

- Original boot image SHA256: `F35B70D0FA20C8AF1B1CE926B468D30998B5DC815FEB1E10C6F8CA6E6819A40C`
- DTB offset: `0x11516c0`
- Tianma DTB size: `151175` bytes
- Boot image size: `18317312` bytes

## Remaining work

Other ROCKNIX port issues observed during the session are separate from the touchscreen fix, including ADB availability, USB/OTG behavior, audio, Wi-Fi behavior, rotation, and suspend/power-management details.

This document records the touchscreen fix only.
