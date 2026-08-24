# Notes on a broken fingerprint sensor: realme C53 (RMX3760)

This is the record of an investigation into a side-mounted fingerprint sensor that
stopped working properly on a realme C53 running Android 15 on Unisoc T612 silicon.
The phone in question enrolls fingerprints badly — most attempts end in endless
"reposition your finger" guidance that never converges — and on the rare occasion a
finger does get enrolled, authentication rejects it every single time.

We searched for anyone documenting this failure before us and found nobody. Not on
XDA, not on GitHub, not in realme's community forums, not in repair-shop circles.
The closest matches were other realme models with stubborn fingerprint enrollment
problems, and those threads all end the same way: someone suggests cleaning the
sensor and doing a factory reset, none of it works, thread dies.

There is also a practical reason this documentation matters. The phone has an
unlocked bootloader and Magisk root, which means the official support route is
effectively closed: a service center sees root, quotes the standard script
(uninstall root, flash stock, lock the bootloader), and considers the case closed
without ever looking at the actual fault. When the official channel refuses to
diagnose anything, the only way knowledge moves forward is if people who do
diagnose things write it down. That is what this repository is.

A few things worth knowing up front:

The entire secure boot chain was verified against official firmware images before
any debugging started, so the modified-software explanation was ruled out early.
Only the owner's own additions exist — a Magisk-patched boot image and TWRP in
vendor_boot — and neither touches the fingerprint stack.

The fingerprint software on this device is not a generic Android stack. It is a
vendor stack from JIIOV/ANC with its own HAL service, its own Trusted Application
running inside TrustOS, and its own storage arrangement that ignores where Android
conventions say fingerprint data should live. Document 03 covers that storage
arrangement, and document 07 covers the moment we discovered the hard way that
screen-lock credentials share it. That discovery alone justifies publishing this.

The root cause remains unproven. What we have instead is a complete map of the
stack, a precise behavioral signature of the failure, and a differential analysis
of four suspect categories — leftover damage from integrity-bypass modules,
firmware bugs, hardware degradation of the scratched power button assembly, and
inherent vendor-stack fragility. Documents 08 and 09 lay out that analysis honestly,
including where the evidence stops.

## Reading order

01 — Device identity and boot chain verification
02 — The fingerprint software stack, layer by layer
03 — Storage topology: where fingerprint data actually lives
04 — Inside the fingerprint trustlet
05 — Result codes observed during the failure
06 — How to observe this stack yourself
07 — The gatekeeper warning: how screen-lock auth almost got destroyed
08 — Differential diagnosis of the four suspect categories
09 — Open questions and what we would try next

All documents live under `docs/`. Nothing here requires programming knowledge;
it is written as plain description. There are deliberately no scripts, no commands
to copy-paste, and no data identifying the original device or its owner.

Topics: fingerprint biometrics · file-based encryption · TEE · Play Integrity API ·
realme UI / Android 15 · Unisoc platform quirks.

Licensed CC BY-SA 4.0 — see `LICENSE`.
