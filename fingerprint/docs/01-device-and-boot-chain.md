# 01 — Device Identity and Boot-Chain Verification

## Device

| Item | Value |
|---|---|
| Model | realme C53 |
| Marketing model number | RMX3760 |
| PCB code | RE58C2 |
| Board codename (kernel cmdline) | `hulk` |
| SoC | Unisoc UMS9230 (Tiger T612 family) |
| Kernel | `5.15.178-android13` |
| OS | Android 15, realme UI, OTA `RMX3760export_15.H.05` |
| OTA incremental | `T.R4T2.1777915050-40` |
| Vendor partition build | Android 13, same incremental string (matched system/vendor pair) |
| Build variant flag | `eng_version=userdebug` |
| Slot layout | A/B, active slot `_b` at time of investigation |

## Why boot-chain verification came first

Before blaming software, every partition involved in secure boot and TEE operation was
verified byte-for-byte against the official firmware. This eliminates "modified
software broke the fingerprint stack" as a class of causes.

Method:

1. Official images downloaded from the public firmware repository:
   `https://gitlab.com/rmx3760/RMX3760-Android15-Firmware`
   (raw file access pattern:
   `https://gitlab.com/api/v4/projects/rmx3760%2FRMX3760-Android15-Firmware/repository/files/<path>/raw?ref=main`)
2. On-device partitions read directly from `/dev/block/by-name/*` (root shell).
3. Each image unpacked locally with `/data/adb/magisk/magiskboot` where needed
   (`unpack`, `extract`) and compared by hash / content diff against stock.
4. AVB state checked via vbmeta contents and verified-boot runtime state.

## Partition verdict table

| Partition | On-device state vs official firmware | Notes |
|---|---|---|
| `vbmeta_a` / `vbmeta_b` | identical to stock | AVB metadata intact, no disabled-verification flags |
| `boot_b` | **DIFFERS** | Magisk-patched kernel image (user root) — expected delta |
| `boot_a` | stock | inactive slot untouched |
| `vendor_boot_b` | **DIFFERS** | TWRP recovery stubbed into ramdisk (~148-byte ramdisk payload) — expected delta |
| `init_boot` | identical to stock | contains generic ramdisk on modern layouts |
| `dtbo` | identical to stock | no device-tree overlay tampering |
| `trustos_b` | identical to stock | TEE OS (ASTO container format) untouched |
| `teecfg_b` | identical to stock | TEE configuration untouched |
| `sml_b` | identical to stock | secure-monitor loader untouched |
| `userdata`, `metadata` | not compared | user data, encrypted |

## Conclusion

The only deltas are the two artifacts the owner intentionally installed
(Magisk root, TWRP). Neither touches the fingerprint stack, the TEE, or any HAL.
The entire fingerprint software stack — HAL binary, TA trustlet, SELinux policy,
VINTF manifests — is exactly what shipped with the matched system/vendor pair.

Therefore the failure is either:

* inside the stock stack itself (state corruption in TA storage), or
* hardware degradation of the sensor assembly,

not something a reflash of public software can be assumed to fix.

## Reference hashes

Hashes below are derived from the **public** firmware repository (safe to publish;
anyone can reproduce them):

| File | MD5 |
|---|---|
| `odm/firmware/jiiov0101.elf` (fingerprint TA) | `61569e0080aa50695c469ec485e598d6` |
| `odm/firmware/jiiov.elf` (loader/companion TA) | `8d536186ddbb266daba9b3d6831290e0` |
