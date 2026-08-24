# RMX3760 Documentation Hub

Technical documentation for the realme C53 (RMX3760) — Unisoc T612 platform,
Android 15. Collected from hands-on investigation rather than speculation, written
to be useful to owners, modders, and repair technicians working on this device.

## Current sections

### [fingerprint/](fingerprint/) — Side fingerprint sensor investigation

Complete mapping of the JIIOV/ANC fingerprint stack as shipped on Android 15:
boot-chain verification against official firmware, HAL and TEE architecture,
storage topology shared with gatekeeper, observed failure signatures with numeric
codes, differential diagnosis of four suspect categories, and a hard-earned
warning about the storage that screen-lock credentials depend on.

English · CC BY-SA 4.0 · no binaries, no personal data.

Start with [`fingerprint/README.md`](fingerprint/README.md).

## Planned sections

Placeholders for future documentation on this device:

- Bootloader unlock / root / recovery notes
- Firmware layout and OTA structure
- Engineering menu reference
- Storage partition map

Contributions welcome via pull request.
