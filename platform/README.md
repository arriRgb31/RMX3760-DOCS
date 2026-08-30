# Platform & Boot Chain Reference — RMX3760

This section documents how the phone is actually put together, verified directly on
a live rooted unit rather than copied from spec sheets. Where direct observation
disagreed with popular specification databases, the disagreement is called out —
two of them turned out to matter.

All observations below come from firmware `RMX3760export_15.H.05` (security patch
2026-05-01) unless noted otherwise.

## Device identity, straight from the build

| Property | Value |
|---|---|
| Model | RMX3760 |
| PCB / device code | RE58C2 |
| Board name | `ums9230_hulk` |
| OS | Android 15, build `T.R4T2.1777915050`, release-keys |
| Security patch | 2026-05-01 |
| Slot at inspection | `_b` |

The board name confirms the Unisoc UMS9230 platform — the same silicon family sold
as Tiger T612 — running an A/B, dynamically partitioned layout under Treble.

## The kernel is GKI, and the naming proves it

`uname` reports:

```
Linux 5.15.178-android13-8-g0c749b198e8d-ab40 SMP PREEMPT aarch64
```

That version string deserves unpacking, because it explains the whole software
architecture. The suffix `android13-8` identifies the kernel's KMI — Kernel Module
Interface — branch: an Android Common Kernel from the android13-5.15 family,
generation 8. In other words the kernel itself is a Google-built Generic Kernel
Image lineage component, not something Unisoc or realme compiled freely. The
`ab40` tag marks the Android build number the kernel binary came from.

Everything else follows from that design. The system side runs Android 15 while
the vendor pair stays on its Android 13 baseline (VNDK version 33), and the two
cooperate through Treble interfaces precisely because the kernel underneath is
frozen against a stable module interface. This is why the matched system/vendor
pairing matters so much on this device, and why mixing arbitrary vendor images is
far more fragile than it looks.

A native Android 15 kernel source tree and custom image for this exact board are
maintained publicly:
[android_kernel_realme_ums9230_a15](https://github.com/arriRgb31/android_kernel_realme_ums9230_a15).

## Boot chain, stage by stage

Unisoc boots differ from Qualcomm/MediaTek layouts in naming but follow the same
trust flow. From reset to unlock:

1. **BootROM** in silicon loads the first-stage loader from eMMC boot hardware
   partitions (`mmcblk0boot0/1` exist alongside the 82 named partitions).
2. **U-Boot** (`uboot_a/b`) initializes DRAM and storage, then verifies and jumps
   onward. Unisoc keeps its own log region (`uboot_log`) and first-logo image
   (`fbootlogo`) as dedicated partitions.
3. **AVB verification** fans out from `vbmeta_a/b` into per-image hash descriptors:
   this device carries separate `vbmeta_odm`, `vbmeta_product`, `vbmeta_system`,
   `vbmeta_system_ext`, and `vbmeta_vendor` partitions plus rollback metadata
   (`avbmeta_rs`). Each dynamic-partition image is independently attested.
4. **TEE early load**: `sml_b` (secure monitor loader) brings up TrustOS from
   `trustos_b` configured by `teecfg_b`; trusted applications load later from odm.
5. **Generic Kernel Image** starts, mounts the dynamic super partition, and launches
   init with `slot_suffix=_b` from the kernel command line.
6. **Vendor HALs** — including the JIIOV fingerprint service documented in the
   fingerprint section — attach to their drivers and proxy daemons.

### Partition group overview

| Group | Partitions | Role |
|---|---|---|
| Bootloader | `uboot_a/b`, `mmcblk0boot0/1`, `fbootlogo`, `uboot_log` | Unisoc boot flow |
| Verification | `vbmeta_*` (×2 slots each), `avbmeta_rs_a/b` | AVB root + per-image attestations |
| Kernel | `boot_a/b`, `init_boot_a/b`, `vendor_boot_a/b`, `dtbo_a/b` | GKI kernel, generic ramdisk, vendor ramdisk, overlays |
| Secure world | `trustos_a/b`, `teecfg_a/b`, `sml_a/b` | TrustOS TEE stack |
| Radio | `l_modem_a/b`, `l_ldsp_a/b` | LTE modem and DSP firmware |
| Dynamic (inside `super`) | `system`, `vendor`, `product`, `system_ext`, `odm`, plus dlkm trees | Android OS images, VNDK 33 vendor |
| Device-specific state | `prodnv`, `persist`, `misc`, `miscdata` | TA/gatekeeper store, calibration, bootloader state |

Note where `prodnv` sits — outside userdata, outside factory reset's reach — which
is the foundation of the gatekeeper warning in the fingerprint section.

## Display subsystem — measured, not assumed

Public specification databases disagree about this panel, and the disagreement is
resolvable only by asking the panel itself. Asked directly, the display stack
reported:

| Property | Measured value |
|---|---|
| Resolution | **720 × 1600 (HD+)** |
| Refresh modes | 60 Hz and 90 Hz, switchable |
| Peak luminance (framework-reported) | 500 nits |
| Density | 320 dpi (~260 ppi) |
| Cutout | top-center drop, ~64 px deep |
| Backlight control | `sprd_backlight`, 4095 PWM steps |
| Pipeline | Unisoc DPU (`dispc0`) driving MIPI-DSI panel via `sprd-mipi-panel-drv` |

So for this variant the HD+ listings were right and the widely-copied FHD+ claim
wrong. The brightness ladder exposed to auto-brightness tops out near 366 nits in
normal operation with higher-stress values reserved beyond it — consistent with a
budget LCD rather than marketing peak figures.

## Touch controller

The devicetree wires the touchscreen as an OmniVision TCM controller on SPI
(`omnivision_tcm`), with compatibility coordinate nodes referencing Jadard and
Himax parts. Worth remembering when sourcing replacement panels: digitizer
firmware expectations ride along the display assembly supply chain.

## Camera stack — two real sensors, mapped over I2C

The rear "50 MP" and front "8 MP" are driven by a Samsung **S5KJN1** (rear) and an
OmniVision **OV8856** (front), plus a fixed-focus depth lens — all identified
directly from the devicetree I2C nodes without opening the camera app. Notably
the main sensor exposes a hidden **3840×2160 (4K)** output bucket and **120 fps
slow motion**, both unsurfaced by the stock camera app, while no lens on the
device carries image stabilization and there is no ultrawide.

Full measurement (sensor routing, every negotiated size, frame-rate and AI/HDR
gates) lives in [`platform/camera.md`](camera.md).

## Battery — what sysfs says versus the box

Marketing says 5000 mAh typical. The fuel gauge reports a **design capacity of
4775 mAh** with lithium-ion chemistry, exposing voltage, temperature, current and
remaining-charge counters over the standard power-supply interface. Charging runs
the advertised 33 W SUPERVOOC path through the USB power-supply node. None of the
usual hidden-capacity games here — just the normal gap between electrochemical
design value and rounded retail figure.

## Capability flags actually present

Framework features confirmed enabled on this unit:

- **NFC, complete stack**: controller plus card-emulation modes (HCE, off-host
  eSE, UICC-based) — several databases list this model NFC-less; this hardware
  disagrees, matching realme's "depends on market" caveat.
- Side-mounted **fingerprint** (see the fingerprint investigation section).
- Bluetooth LE, Wi-Fi Direct, USB host mode.
- Motion and environment sensors: accelerometer, gyroscope, magnetometer/compass,
  light, proximity, step counter and detector.

## Companion repositories

Working code and tooling maintained alongside this documentation:

| Repository | Contents |
|---|---|
| [android_kernel_realme_ums9230_a15](https://github.com/arriRgb31/android_kernel_realme_ums9230_a15) | Native Android 15 (5.15.178) kernel source and custom image for RMX3760 |
| [RE58C2_device_base](https://github.com/arriRgb31/RE58C2_device_base) | Generated device-tree base for the board |
| [RMX3760](https://github.com/arriRgb31/RMX3760) | Build makefiles |
| [RMX3760-tools](https://github.com/arriRgb31/RMX3760-tools) | Unlock, AVB, DM-Verity, SELinux, root, TWRP, ADB/fastboot tool suite |
| [spreadtrum_flash](https://github.com/arriRgb31/spreadtrum_flash) | Unisoc flashing protocol tooling (research fork) |

See [fingerprint/](../fingerprint/) for the deep dive into the biometrics stack,
including the storage arrangement shared with screen-lock credentials.

Licensed CC BY-SA 4.0 together with the rest of this repository.
