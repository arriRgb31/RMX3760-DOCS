# RMX3760 Documentation Hub

Technical documentation for the **realme C53** family — Unisoc T612 platform,
updated to Android 15. Collected from hands-on investigation and verified public
sources, written to be useful to owners, modders, and repair technicians working
on this device.

---

## realme C53 — general overview (public information)

The realme C53 launched in mid-2023 as a budget-focused 4G smartphone positioned
around an unusually slim body (7.49 mm), a large 90 Hz display, and fast 33 W
charging at its price point. It shipped with Android 13 under realme UI T Edition
and has since been updated through major Android releases to Android 15.

Two hardware generations exist under the C53 name:

| Model | Market | Distinguishing features |
|---|---|---|
| **RMX3760** | Global / Export | 50 MP main camera, thinner 7.49 mm body |
| **RMX3762** ("C53 India") | India | 108 MP main camera, slightly thicker/heavier body |

### Shared platform

Both models share the same core platform: Unisoc Tiger T612 (12 nm, T7225),
octa-core CPU (2× Cortex-A75 @1.8 GHz + 6× Cortex-A55 @1.8 GHz) with Mali-G57 GPU,
side-mounted capacitive fingerprint sensor integrated into the power button,
5000 mAh battery with 33 W SUPERVOOC wired charging, dual nano-SIM standby,
dedicated microSD slot (up to 2 TB), 3.5 mm headphone jack, USB-C 2.0 with OTG,
Wi-Fi 802.11ac dual-band, Bluetooth 5.0. No 5G on either model.

### Display and build

6.74-inch IPS LCD, 20:9, 90 Hz refresh rate, peak brightness around 560 nits,
roughly 85.5% screen-to-body ratio. Glass front, plastic frame and back.
RMX3760 measures 167.3 × 76.7 × 7.49 mm at 182 g; the Indian RMX3762 is
167.2 × 76.7 × 7.99 mm at 186 g. Colors: Mighty Black, Champion Gold.

### Memory variants

| Model | RAM options | Storage options |
|---|---|---|
| RMX3760 | 6 GB, 8 GB (LPDDR4X, Dynamic RAM expansion available by region) | 128 GB, 256 GB (UFS 2.2) |
| RMX3762 | 4 GB, 6 GB | 64 GB, 128 GB |

### Cameras

- **RMX3760**: rear 50 MP f/1.8 wide with PDAF plus auxiliary lens, 1080p video;
  front 8 MP, 720p video
- **RMX3762**: rear 108 MP main camera; front 8 MP

NFC availability depends on variant and market — not present in all regions.

### Software status (RMX3760)

Launched on Android 13 (realme UI T Edition); currently receiving updates based
on **Android 15 / realme UI**:

- Latest observed build: `RMX3760export_15.H.07`
- Android security patch level: **2026 #7**
- Official changelog: fixes known issues; improves overall system stability

The fingerprint investigation documented in this repository was conducted while
the device ran the earlier `15.H.05` maintenance line; findings are annotated
accordingly.

---

## Repository sections

### [fingerprint/](fingerprint/) — Side fingerprint sensor investigation

Complete mapping of the JIIOV/ANC fingerprint stack on this platform: boot-chain
verification against official firmware, HAL and TEE architecture, storage topology
shared with gatekeeper, observed failure signatures with numeric codes,
differential diagnosis of four suspect categories, and a hard-earned warning about
the storage screen-lock credentials depend on.

English · CC BY-SA 4.0 · no binaries, no personal data.

Start with [`fingerprint/README.md`](fingerprint/README.md).

## Planned sections

Placeholders for future documentation:

- Bootloader unlock / root / recovery notes
- Firmware layout and OTA structure
- Engineering menu reference
- Storage partition map

Contributions welcome via pull request.
