# 02 — The fingerprint software stack

Understanding this failure requires knowing how unusual this stack is compared to
what most Android documentation assumes. There is no `fingerprintd`, no AOSP
fingerprint HAL implementation, and none of the standard template storage paths.
Everything from the HAL downward belongs to a biometrics vendor whose components
identify themselves as ANC and JIIOV.

## The layers, top to bottom

At the top sits the normal Android biometrics framework talking over AIDL, exactly
as on any modern device. Below that, instead of a generic vendor HAL, runs a
dedicated service binary named for the JIIOV stack alongside a companion library
that carries the sensor-side logic. That library communicates with a Trusted
Application running inside TrustOS, the Unisoc trusted execution environment, whose
own images (`trustos`, `teecfg`, `sml`) all sit untouched in their boot partitions.
Inside the secure world the work splits between a loader trustlet and the main
fingerprint trustlet, the latter carrying the matching algorithm, versioned
JV1.3.06.1492 at the time of investigation.

The trustlet talks outward through the proxy arrangement described in document 03,
and the HAL side reaches the TEE through Unisoc's TEE driver interfaces. Sensor
communication happens over SPI to the capacitive array integrated into the power
button assembly.

| Component | Where it lives | Notes |
|---|---|---|
| Biometrics framework | system partition | stock AOSP behavior |
| Vendor HAL service | vendor partition, named for the JIIOV stack | logs extensively once debug mode is enabled |
| Companion library | `/vendor/lib64/hw/anc.hal.so` | stripped ARM64 ELF, handles sensor and protocol logic |
| TEE OS | `trustos_b` / `teecfg_b` / `sml_b` partitions | ASTO container format, byte-identical to official firmware |
| Loader trustlet | `odm/firmware/jiiov.elf` | MD5 published in document 01 |
| Fingerprint trustlet | `odm/firmware/jiiov0101.elf` | carries the matching algorithm and all storage logic |
| Persistent state | proxy-served object, document 03 | shared with gatekeeper |

## What the HAL is allowed to touch

SELinux confines the HAL service to its own domain, `hal_fingerprint_default`. Its
policy grants access to a small set of labeled resources: a data type called
`jiiov_vendor_data` mapped onto `/data/vendor/fingerprint`, device nodes for the
sensor and TEE interfaces, a handful of vendor properties, and the biometric
services. Nothing in the policy mentions the persist-tree paths hardcoded inside
the trustlet, which is one more sign those paths belong to dead code on this build.

This matters when evaluating fixes. Any intervention that would have the HAL or
trustlet writing somewhere outside its allow-list will fail with denials rather
than silently working, and the denial shows up in the kernel log where it can be
checked.

## Debug surface

Two facts make this stack unusually observable for vendor code. First, setting the
vendor debug property enables verbose logging through the whole HAL, including
per-step numeric return values from capture, extraction, enrollment, and matching.
Second, the framework emits telemetry events containing structured fields — image
quality scores, match scores, algorithm version, enroll counters — which survive
even without the debug property. Document 06 describes both channels and what we
saw in them.

One boot-time observation worth recording: on every cold boot the HAL reported
successful calibration loading, and the module identity string it printed matched
the QR-code label physically etched next to the sensor on this unit. Initialization
is not where this failure lives. The stack comes up healthy and stays healthy until
the algorithm is asked to accumulate or compare features — at which point the
behavior documented in 05 begins.
