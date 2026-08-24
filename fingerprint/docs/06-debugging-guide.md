# 06 — How to observe this stack yourself

Everything documented here came from three observation channels available on a
rooted device, plus one hardware-facing diagnostic menu. This page describes what
each channel shows and where it misleads. Consistent with this repository's scope,
it describes methods rather than reproducing commands.

## The vendor debug switch

The HAL stays nearly silent until the vendor debug property
`persist.vendor.debug.fp.support` is enabled. Once true, every operation emits its
per-step numeric outcomes — capture, extraction, enrollment steps, comparison —
into the system log under the JIIOV/ANC tags. The property persists across
reboots, which makes it suitable for capturing boot-time initialization as well.
All numeric codes discussed in document 05 were read from this channel.

## Live session capture

With debug enabled, an enrollment or unlock attempt produces a complete narrative
in the log: sensor wake, capture frames, extraction success, then the algorithm's
verdicts. The failure signature described throughout this repository is easy to
recognize once seen — extraction succeeding while enroll-step guidance repeats and
session outcomes land on the reposition code, or extraction succeeding while every
comparison returns the 501/502 pair. Watching one live attempt end to end teaches
more than any static description.

Two practical warnings from experience. First, sustained interaction destabilizes
this particular stack: repeated cancel-and-retry cycles can push the sensor into a
failed-init state that only a reboot clears, so plan captures accordingly. Second,
the kernel log is worth watching alongside the system log during any experiment
near SELinux boundaries — denials appear there and nowhere else.

## Telemetry events

Independent of debug mode, the framework emits structured telemetry when biometric
sessions conclude. Fields include final result flags, stored-finger counts,
enrollment attempt counters, template version, algorithm version, quality score,
and match score. Two uses stand out: the counters confirm objectively whether any
template actually got stored (ours read zero fingers despite multiple attempts),
and the side-by-side scores expose the flat-match anomaly — match score pinned
near 427–428 across attempts whose quality scores ranged roughly 49 to 85.

## Boot-time observation

Initialization logs confirm the stack comes up healthy: calibration reported as
loaded, module identity printed and cross-checked against the QR label beside the
physical sensor, algorithm version displayed. Capturing boot logs requires the
logging to start before user storage is decrypted — an early-boot script writing to
a location readable without unlock works; writing to credential-encrypted home
directories does not, a mistake we made once and corrected. Nothing in boot logs
ever differed between healthy-memory eras and broken ones, which is itself
evidence: this failure lives after initialization.

## The engineering menu

The dialer code opening realme's hardware diagnostics on Unisoc builds reaches a
fingerprint test suite measuring raw image quality, SPI communication, bubble and
dead-pixel checks. Our readings passed comfortably — quality scores between 70 and
93, communication clean, panel checks fine. The menu cannot see what is actually
broken here: it evaluates single-frame optics, never feature stability across time,
and never the accumulate-or-compare stages where this failure lives.

One incidental finding: the same menu froze after repeated rapid touches, echoing
the sensor-side instability seen elsewhere. Treat its pass results as necessary but
nowhere near sufficient.
