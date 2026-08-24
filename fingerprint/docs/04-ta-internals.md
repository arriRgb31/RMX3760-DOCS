# 04 — Inside the fingerprint trustlet

The main fingerprint trustlet was extracted from the `odm/firmware` directory of
the device and identified by its container header — Unisoc trusted applications on
this platform ship inside an ASTO-wrapped ELF. The binary is stripped, so the
analysis below comes from its string tables and observable behavior rather than
disassembly. Even at that level of inspection, the trustlet reveals a great deal
about how its authors expected it to fail.

## The matcher knows several ways to say no

The error vocabulary includes a set of match-failure reasons, each associated with
a floating-point score in the code's own logging: a live-finger failure mode, a
ghost-related mode, a partial-image mode, one tied to exposure status, one that
fires when a debase pattern already exists, and a fallback default. The presence of
distinct reasons for ghost impressions and pre-existing debase patterns tells us the
algorithm actively maintains corrective state about what it has previously seen,
not just a passive template list.

## Ghost cache and debase machinery

Two subsystems dominate the storage strings. The first manages files named as ghost
caches under the trustlet's root path, with backup and temporary variants, plus
explicit load-and-clear operations. Ghost impressions are the residue patterns that
capacitive sensors gradually accumulate from repeated touches; the stack is clearly
designed to learn what the sensor's "empty" state looks like and subtract it.

The second manages per-finger base-normalization data — files with a `.debase`
suffix paired with context files, loaded during matching sessions. This is
reference-state machinery: normalization baselines that comparisons depend upon.
If these references are missing or corrupt while templates technically exist, the
matcher would still run and still produce scores, but against garbage — which is
precisely the flat-score signature described in document 05.

## A self-healing database that heals into nothing

Persistent records go through an internal database file checked by magic value and
CRC32, with explicit log vocabulary for cleaning a corrupted database before
continuing. Alongside it sit calls to write whole record containers out through the
storage proxy to a fixed base name — the mechanism behind the fixed-size,
in-place-updated object described in document 03. The design intent is resilience:
if the database fails its checks, rebuild it clean. The design's blind spot is that
"clean" means empty. A self-healing event that discards corrupted reference state
would leave the engine running normally with no memory of any finger — matching our
observed behavior exactly — without producing any user-visible error anywhere.

## Paths into the void

The trustlet also carries hardcoded absolute paths under a persist tree inside the
vendor mount: a calibration base image and a normalized-calibration backup among
them. On this build that tree does not exist and never has (document 03), and the
SELinux policy gives the HAL no route to create or write it. Whatever calibration
persistence those paths were meant to provide either happens elsewhere on this SKU
or does not happen at all. Vendor firmware containing live code paths aimed at
missing filesystem locations is a small but honest indicator of engineering strain
in this stack's port.

## Factory-test surface

Finally, the trustlet advertises a wide extension command surface used by factory
tooling: module test variants, lens and defect tests, signal-to-noise measurement,
exposure calibration routines both automatic and fixed, save/load operations for
calibration data, file synchronization, and a general reset command. The reset
command's exact semantics could not be determined from strings alone, and since it
operates inside the same shared persistent store that gatekeeper depends on
(document 07), we deliberately did not invoke it blind. That restraint is
recommended to everyone.
