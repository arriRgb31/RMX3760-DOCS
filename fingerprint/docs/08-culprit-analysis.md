# 08 — Differential diagnosis: four suspects

Every failed-software investigation eventually reduces to naming suspects and
weighing them honestly. This failure has four candidate categories. None can be
convicted with the evidence available, but the evidence does sort them into
different weights.

## Suspect one: damage from module use, including integrity-bypass tooling

The owner installed Magisk modules in the period immediately before the fingerprint
faults began — among them tooling of the Play Integrity bypass variety. That
timeline correlation is real and cannot be waved away.

The mechanism deserves precision. Integrity-bypass modules work by reshaping what
the system appears to be: spoofed build properties, adjusted security-patch
strings, masked root artifacts, sometimes hooked attestation paths. None of that
touches the TEE directly — trusted applications live behind hardware enforcement
and see none of it. But vendor daemons do observe their environment, and a
fingerprint stack that validates its surroundings can behave badly when its
surroundings stop being self-consistent. More importantly for this case: whatever
a confused daemon or trustlet writes into its persistent store during such a
period *stays written*. Modules can be removed; persistent state does not come
with an undo.

That is the strongest version of this suspect: not ongoing interference — the boot
chain was later verified fully stock, ruling that out — but a one-time corruption
event whose damage persists in the shared store where no user-accessible reset can
reach it (document 07). The flat match-score signature fits damaged reference
state well.

Against it: pure inference. No log captured the moment of damage, and no test can
distinguish corrupted state from other causes without safely resetting that state,
which is precisely what nobody can do. This suspect is unfalsifiable from userland,
which should temper any confidence placed in it.

## Suspect two: firmware bug

The matched system/vendor pair verified byte-identical to official images, so this
is not about modified firmware. It is about the firmware's own quality. The
investigation documented genuine engineering strain in this build: trustlet code
paths aimed at filesystem locations that never existed on this SKU, a permitted
data directory the stack never uses, one filesystem mounted at two paths with no
consumer of the second. A stack with dead calibration persistence and half-wired
storage plumbing is a stack where latent bugs are unsurprising.

Supporting this suspect further, the device carries an engineering-lineage build
flag, and public forums hold years of Unisoc-platform fingerprint complaints across
brands — including the factory-reset-doesn't-help genre that document 07 explains.
Against it: nothing identifies a specific defect, and identical stacks demonstrably
work on healthy units of the same model.

## Suspect three: hardware degradation of the sensor assembly

The owner reports the power button — which carries the sensor array — bears a
scratch. This suspect earns its weight by explaining everything at once.

Capacitive micro-arrays fail gracefully rather than absolutely. Damaged elements
leave image formation largely intact — raw quality scores stay respectable, which
is exactly what the engineering menu measured (70–93) — while ridge data across the
damaged region becomes subtly unstable between touches. An algorithm trying to
accumulate a template from features that never stabilize repeats its reposition
guidance forever; an algorithm comparing against any reference rejects everything.
Both observed symptoms, one cause. The sensor-init failures under sustained load
also fit marginal electrical behavior in a stressed assembly.

Against it: the timeline. The owner associates onset with module installation
rather than gradual worsening, though a scratch small enough to escape daily notice
may also predate anyone's attention. The decisive test costs little: replace the
assembly, observe. No software experiment offers equivalent information density.

## Suspect four: inherent vendor-stack fragility

Less a competing suspect than ambient context. The trustlet ships machinery —
self-healing databases, ghost caches, debase normalization — that exists only
because its authors expected instability modes like the ones under investigation.
Resilience designed to heal into empty state (document 04) converts transient
corruption into permanent amnesia while reporting perfect health throughout. Any
of the first three suspects acting through this machinery produces the same
terminal picture. The stack's architecture amplifies whatever initiated the fault.

## Weighting

| Suspect | Fits symptoms | Explains timeline | Falsifiable by owner | Weight |
|---|---|---|---|---|
| Module-era state damage | yes | yes (onset) | no — needs safe state reset | strong, unprovable |
| Firmware bug | partially | weakly | no | moderate |
| Hardware degradation | yes | partially | yes — part swap | strong |
| Vendor-stack fragility | amplifier | n/a | n/a | contextual |

If forced to bet: an initiating event during the module era left reference state
damaged inside a store that heals by forgetting, and the scratched assembly may be
quietly contributing instability on top. The part swap tests the second clause
cheaply; nothing available to an owner tests the first. That asymmetry defines the
recommendation in document 09 — and it is why this documentation exists at all:
when neither remaining hypothesis can be confirmed from userland, publishing the
map is worth more than guarding it.
