# 09 — Open questions and next steps

Where the evidence runs out, say so plainly. These are the questions this
investigation leaves open, ordered by how much each would advance understanding
if answered.

## Can fingerprint state be reset without killing gatekeeper?

The blocking question. The trustlet exposes extension commands used by factory
tooling, among them a general reset whose semantics are undocumented outside the
vendor. Its strings suggest it operates inside the same persistent store gatekeeper
depends on. Determining its true behavior requires disassembling the HAL's command
dispatch and tracing what reaches the trustlet and what it touches — feasible work,
deliberately not completed here, because invoking an unknown-reset against shared
state after document 07's lesson is how afternoons get much worse.

Anyone continuing this line should treat the command table extraction as step one
and blind invocation as never.

## Would a sensor assembly replacement resolve it?

The cheapest decisive experiment remaining, and purely hardware-side. If the
scratched button assembly is degrading feature stability (suspect three, document
08), a replacement part restores function immediately. If it does not, suspicion
collapses onto persistent state damage, which narrows everything else. For most
owners this is also the only intervention with zero risk of losing lock-screen
access. Repair shops perform this swap routinely on side-sensor devices.

## Why does factory reset not fix fingerprints on this platform?

Answered, we believe, but worth independent confirmation: the fingerprint engine's
state lives in proxy-served objects on a dedicated persist partition outside
userdata, so userdata wipes never touch it. Public forums carry years of realme and
other-brand Unisoc reports where factory reset failed to cure fingerprint faults;
this storage arrangement explains them all in one stroke. Confirming would take a
controlled test on a sacrificial device — check whether the state object's bytes
change across a factory reset.

## Is the persist-tree dead code or broken wiring?

The trustlet's calibration paths point into a directory tree that has never existed
on this SKU. Either those code paths belong to another variant's build and simply
ride along unused here, or calibration persistence was supposed to work and lost
its mountpoint in porting. Distinguishing matters: if the latter, some exposure-
calibration failures across the Unisoc fleet might trace to it. Comparing trustlet
versions and storage layouts across other Unisoc realme models would settle it.

## Does an official OTA ever touch this state?

Unobserved either way. If vendor OTAs ship migrations for the persist objects, a
future update could theoretically rebuild damaged reference state — the only
software fix route that exists for suspect-one scenarios. Watching whether realme
ever addresses fingerprint complaints on this model in changelogs costs nothing and
might eventually moot the whole question.

## The honest bottom line

For an owner holding this phone today: replace the button assembly first, since it
is cheap, safe, and tests the leading physical hypothesis. If the fault survives a
known-good part, the remaining cause lives inside a store no user tooling can safely
reset, and the realistic options reduce to vendor service with factory tooling —
complicated on unlocked devices — or living with the pattern lock. Everything above
documents why that dead end exists, so the next person can spend their week
somewhere more productive than where ours went.
