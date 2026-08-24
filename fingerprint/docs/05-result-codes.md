# 05 — Result codes observed during the failure

Every numeric outcome below was captured live from the debug-enabled HAL during
controlled enrollment and authentication sessions, then cross-checked against the
trustlet's own error vocabulary. Codes follow a modular scheme: the upper bits
select the subsystem, the lower bits the specific outcome. Within the enrollment
subsystem, for example, outcomes sit just above the value 4096, which is sixteen
times 256 — a module boundary.

## Enrollments

Three controlled enrollment sessions were captured end to end. Image capture worked
every time and feature extraction consistently returned success, code 300. The
algorithm's enroll-feature step, however, returned a rotating cast of failures:
guidance code 303 appeared repeatedly — the same *reposition your finger* /
*duplicate area* instruction users see on screen — flanked by occasional 401 and
406 processing failures. Progress never accumulated. The session-level outcomes
landed on 4097 twice, 4101 once, and 4105 three times; 4105 corresponds to the
reposition guidance and is the number behind the endless loop users experience.
Final telemetry told the same story bluntly: zero fingers stored despite four
enrollment attempts, with template version six reported by the engine.

## Authentication

Authentication sessions were even more consistent, in the worst way. Capture and
extraction again succeeded — code 300 every time. The compare stage then failed
with alternating codes 501 and 502 on every single attempt, producing the negative
authentication result 8197. Retry-reason codes 10241, 10251, and 10252 surfaced
alongside, corresponding to the acquire-guidance family users see as *try again*
messages.

## The anomaly that points somewhere

Telemetry records two scores side by side. Quality score measures the raw image;
match score is what the comparator produces. Across our captures, quality varied
widely — readings of roughly 49 and 85 appear in adjacent attempts — yet the match
score sat at effectively the same value, around 427 to 428, regardless. A working
matcher produces scores that move with input quality and finger variation. A flat
score against every input indicates the comparison is anchored to something static:
an invalid, empty, or corrupted reference rather than the presented finger.

## Sensor-side instability

One earlier enrollment attempt ended worse than guidance: after a cancellation and
retry the sensor failed to initialize, reporting failure 34 with an accompanying
hardware-reset message, and the framework surfaced a hardware-unavailable error to
the user. The sensor recovered after a reboot. Combined with engineering-menu
freezes after repeated touches (document 06), this suggests the stack has marginal
behavior under sustained interaction — relevant background for the hardware
hypothesis in document 08, though not itself the core failure, since the loop and
rejection reproduce cleanly on freshly booted sensors.

## Consolidated table

| Value | Stage | Meaning as observed |
|---|---|---|
| 300 | extraction | success |
| 303 | enroll feature step | duplicate area / reposition guidance |
| 34 | sensor init | hardware reset failure |
| 401, 406 | enroll feature step | processing failures, intermittent |
| 4097 | enrollment session | progress-level outcome |
| 4101 | enrollment session | progress-level outcome |
| 4105 | enrollment session | reposition guidance; the endless-loop signature |
| 8197 | authentication | negative result |
| 501, 502 | compare | rejection codes, always paired with failure |
| 10241, 10251, 10252 | acquisition | retry guidance reasons |

Interpretation confidence varies: the guidance meanings for 303 and 4105 come
directly from user-visible messages tied to them, while 4097 and 4101 were never
accompanied by distinctive on-screen text and may simply be intermediate progress
markers. None of the numbers contradict the central finding — the pipeline works
through capture and extraction and breaks exactly where templates must accumulate
or be compared.
