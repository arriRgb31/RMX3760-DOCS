# 03 — Storage topology: where fingerprint data actually lives

Of everything learned during this investigation, the storage layer turned out to be
the least documented anywhere and the most dangerous to get wrong. On this Unisoc
platform the fingerprint Trusted Application does not keep its templates where
Android conventions suggest, and the same storage that holds fingerprint state also
holds screen-lock credential verification state. Anyone planning to "just wipe the
fingerprint data" needs to read document 07 before doing anything.

## The proxy daemons

At boot several instances of a daemon called `sprdstorageproxyd` come up. Their job
is to bridge storage requests coming from inside the TEE out to locations in the
normal world. Four roles exist on this device, each with its own backing location.
One bridges to an object under `/mnt/vendor/productinfo/sprd_ss` and showed file
activity every time enrollment or authentication was attempted. One bridges to the
RPMB partition, the hardware replay-protected block used for tamper-resistant
counters. A third works with `/data/vendor/sprd_ss`, which held roughly a megabyte
of content during our observations. The fourth points at `/data/vendor/sprd_tee_ss`
and sat empty throughout.

The first of those, the object served by the prodnv role, is where the fingerprint
trustlet keeps its persistent state. The object has a fixed size of 109080 bytes
and is updated strictly in place — never grown or shrunk — which is what you expect
from a fixed-layout record container rather than a directory of files.

## One filesystem mounted twice

The partition `mmcblk0p1` carries a single filesystem that the system mounts at two
different places simultaneously: `/mnt/vendor` and `/mnt/prodnv`. Both mountpoints
see the same content. More interesting is what is missing: there is no
`/mnt/vendor/persist` directory on this build at all, even though the fingerprint
trustlet contains hardcoded paths expecting one to exist (document 04). Any trustlet
code path trying to store calibration files there is writing into a void.

## Locations we inspected, and what they contained

| Location | Found | Interpretation |
|---|---|---|
| `/data/vendor_de/0/fpdata` | empty | The AOSP-conventional spot; this stack never uses it |
| `/data/vendor/fingerprint` | directory exists but empty | SELinux policy fully permits the HAL to use it, yet nothing was ever written |
| `/mnt/vendor/persist/fingerprint/jiiov` | does not exist | Hardcoded trustlet paths point here; the tree was never created |
| `/mnt/vendor/productinfo/sprd_ss/0` | fixed-size object, updated during fingerprint activity | The real persistent store |
| `/data/vendor/sprd_ss` | ~1 MB of live content | Secondary normal-world state area |

The empty-but-permitted `/data/vendor/fingerprint` directory is telling. Someone at
the vendor intended file-based template storage there, wrote complete SELinux rules
for it, then shipped a build where the actual persistence goes through the proxy
object instead. This is consistent with other half-finished plumbing on this SKU
and supports the vendor-stack fragility discussion in document 08.

## Why this matters more than it seems

The objects under `sprd_ss` are shared. Gatekeeper — the component that verifies
your pattern or PIN — reads and writes through the same proxy arrangement. During
this investigation one such object was renamed away from its expected path as part
of an experiment, the phone rebooted, and pattern unlock stopped working entirely.
The exact bytes were restored and unlock came back. Document 07 tells that story in
full, including why recovery from losing that object is far harder than it sounds
on an Android 15 device with file-based encryption: your encrypted data can only be
unlocked after credential verification succeeds, so damaging the verifier damages
access to everything.

The practical conclusions are uncomfortable but simple. There is no safe,
selective way for a user to reset only fingerprint state on this platform. Guides
that recommend deleting `fpdata` directories describe places this stack never
reads. And any experiment near these storage objects must start from a verified
byte-level copy kept off-device while the device can still be accessed normally.
