# 07 — The gatekeeper warning

This document exists so nobody else has to learn it the way we did. During the
investigation, one storage experiment crossed a line that was invisible at the
time, and the phone briefly became something close to unusable. What follows is a
reconstruction of what happened, why it happened, and the rules that came out of it.

## What happened

As part of testing whether the fingerprint trustlet's persistent state was
corruptible-and-recoverable, one of the proxy-served state objects under
`/mnt/vendor/productinfo/sprd_ss` was set aside — renamed within its directory,
nothing deleted — followed by a reboot. The expectation was a clean slate for the
fingerprint engine: worst case, previously enrolled fingers stop matching.

The actual result was different in kind. After reboot, the screen lock itself
stopped accepting its credentials. The correct pattern was rejected. The phone sat
there asking for verification that could no longer succeed.

## Why it happened

The object that was moved away does not belong to the fingerprint application
alone. Gatekeeper — the component that verifies screen-lock credentials before
anything else may proceed — reads and writes through the same proxy arrangement
and keeps its state inside those same objects. Removing the object removed the
verification basis for every credential on the device.

On an Android 15 device with file-based encryption this cascades immediately.
Credential-encrypted storage only becomes readable after credential verification
succeeds. With the verifier's state gone, the encrypted data behind it is
unreachable by design — not just inconvenient, structurally inaccessible until the
verifier works again. Standard recovery environments cannot simply read past this;
they are bound by the same rule.

## How it ended

Recovery was possible only because a verified byte-level copy of the object existed
from before the experiment, taken while the device was still fully accessible. The
original bytes were restored through a PC connection, the device rebooted, and
credential verification worked again on the next attempt. Nothing else was lost.
The lesson cost a very bad afternoon and nothing more, which by the standards of
this particular mistake counts as lucky.

## The rules this bought

First: the objects under `sprd_ss` are shared infrastructure between fingerprint
and screen-lock state. There is no selective wipe. Any tool or guide describing
them as fingerprint-only storage is wrong, and on this platform being wrong there
costs your entire lock screen.

Second: a backup taken after the mistake is worthless. The only useful moment to
make a verified, off-device copy is while everything still works.

Third: rollback must be tested while you still hold working credentials. Restoring
an object is itself a write operation into sensitive territory; finding out your
restore procedure has a flaw after you have already lost access is not a recoverable
position.

Fourth: the official support channel will not meet this problem halfway. On an
unlocked, rooted device the standard response is a reflash and bootloader relock,
which resolves nothing documented here — and if credential state were ever truly
lost rather than restored, no amount of reflashing brings encrypted data back.
Prevention is the only safety net that exists.

Fifth, and most relevant to the rest of this repository: this discovery reframes
the whole investigation. The fingerprint engine's reference state cannot be safely
reset by any means available to a user. Whatever corrupted it, if state corruption
is indeed the cause (document 08), it stays corrupted — through reboots, through
module removal, and notably even through factory resets, since these objects live
outside userdata entirely. That last point quietly explains a pattern visible in
public forums for years: realme and Unisoc users reporting that factory reset did
not fix their fingerprint sensor. It never would have.
