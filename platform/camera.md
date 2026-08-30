# Camera Hardware — Measured Live on RMX3760

Where the rest of the platform section reads the display, touch and battery from
the live unit, this page does the same for the camera stack. Direct observation
was done over the Android Camera2 API (`dumpsys media.camera`) and the kernel's
I2C / devicetree nodes on a rooted unit, so the values below are the silicon's
actual negotiated capabilities rather than marketing copy.

Observations come from firmware `RMX3760export_15.H.05` unless noted otherwise.

## What the board actually mounts

The camera tuning table (`persist.vendor.cam.sensor.tuning.*`) names the two real
image sensors mounted on this board:

| Role | Sensor tuning id | Underlying sensor |
|---|---|---|
| Rear main | `id0 = s5kjn1_sunny` | Samsung ISOCELL **S5KJN1** |
| Front selfie | `id1 = ov8856_front_ly` | OmniVision **OV8856** |

Both are exposed over the Camera HAL (v3.5) as the two physical logical IDs
`legacy/0` (Rear) and `legacy/1` (Front). The software then advertises many extra
logical camera IDs that all map back onto these two sensors for multi-camera
compositions.

## 1. Physical routing — identified over I2C without opening the app

Each sensor sits on its own I2C bus with a fixed 7-bit address, described in the
devicetree under `soc/ap-apb/i2c@*.` Read straight from `/sys/bus/i2c/devices/`:

| I2C node | Bus @ addr | Kernel driver | Devicetree node | Sensor |
|---|---|---|---|---|
| `sensor-main` | i2c-0 @ `0x5a` | `sensor_main` | `i2c@0x200d0000/sensor-main@5a` | S5KJN1 (rear main) |
| `sensor-sub` | i2c-1 @ `0x20` | `sensor_sub` | `i2c@0x200e0000/sensor-sub@20` | OV8856 (front) |
| `sensor-main2` | i2c-1 @ `0x43` | `sensor_main2` | `i2c@0x200e0000/sensor-main2@43` | rear depth / auxiliary |
| `sensor-sub2` | i2c-6 @ `0x6e` | `sensor_sub2` | `i2c@0x20220000/sensor-sub2@6e` | unpopulated slot |

The devicetree hints confirm each role:

- **sensor-main** lists `vddcama`, `vddcamd`, **`vddcammot`** (DSP / AF-motor rail) and
  `dvdd-gpios` + `reset-gpios` — a driven, motorised main module (AF).
- **sensor-main2** (the depth assistant) lists `power-domains`, `reset-gpios` and
  `sprd,vddldo1-voltage`, but **no `vddcammot` and no `vddcamd`** — confirmed at boot
  by `sensor_main2 1-0043: supply vddcamd not found, using dummy regulator`.
  A fixed-focus, low-power auxiliary lens, exactly what a bokeh depth sensor is.
  It is absent from the tuning table (`id2 = default`), so it is a passive helper.
- **sensor-sub2** at `0x6e` is a second front-position slot that is **not mounted**
  on this hardware revision (no tuning entry).

So the data path is: S5KJN1 controls the image on i2c-0, OV8856 on i2c-1,
the depth lens shares bus i2c-1 at `0x43`, and the fourth slot is dead silicon.

## 2. Rear camera — Samsung S5KJN1 (the "50 MP")

The marketing name says 50 MP. The Camera2 characteristics expose the sensor's
operating arrangement as reported by the HAL:

| Property | Measured value |
|---|---|
| Active array | **4080 × 3072** (~12.5 MP output) |
| Full resolution | 4080 × 3072 (still OUTPUT) |
| Aperture | **f/1.8** |
| Focal length | **4.05 mm** |
| Physical sensor size | 5.20 × 3.93 mm (~1/2.74") |
| CFA pattern | RGGB |
| Optical stabilization | none (`availableOpticalStabilization = [0]`) |
| Hardware level | LIMITED |
| Digital zoom | up to 10× (`availableMaxDigitalZoom = 10.0`) |

The silicon itself (S5KJN1) is a native quad-Bayer sensor; the HAL actively
exposes a **12.5 MP (4080×3072) remosaic output** as its working resolution, from
which the "50 MP" shots are produced via pixel-binning / AI upscaling. This is
normal for quad-Bayer budget sensors — the 50 MP figure is a reconstruction, not
a per-pixel native readout.

### Rear resolutions the HAL will actually produce

`availableStreamConfigurations` (format tags: 33 = YUV, 34 = PRIVATE, 35 = RAW,
37 = JPEG) lists **every** output bucket the rear camera can negotiate:

```
4080x3072 * 4000x3000 * 4000x2240 * 4000x2000 * 4000x1896
3840x2160 *  3280x2464 * 3264x2448 * 3264x1836 * 3264x1632
3264x1552 * 3200x1440 * 2592x1944 * 2592x1458 * 2592x1456
2592x1296 * 2592x1224 * 2560x1920 * 2560x1440 * 2448x2448
2320x1740 * 2304x1728 * 2272x1080 * 2160x1080 * 2048x1536
2048x1152 * 1920x1080 * 1920x960  * 1920x896  * 1920x864
1728x1728 * 1600x1200 * 1600x1080 * 1600x720  * 1440x1080
1280x1280 * 1280x1080 * 1280x720  * 1280x640  * 960x720
800x600   * 720x720   * 720x480   * 640x480   * 352x288
320x240   * 320x144   * 304x144   * 288x144   * 256x144
240x240   * 176x144
```

(* = also present in the RAW `35` bucket, i.e. real sensor readouts, not pure crop)

**A hidden 4K.** `3840×2160` is a real negotiated output bucket on this rear
camera even though realme only advertises 1080p for the model. The silicon can
record 2160p; the shipping camera app simply does not surface it. Same story for
a handful of other sizes (e.g. 2560×1440). Capturing through a third-party
Camera2 app exposes them.

## 3. Rear video / frame-rate capability

- **AE target FPS ranges:** 10–10, 10–15, 15–15, 10–20, 20–20, 10–24, 24–24,
  **10–30, 30–30** → smooth 30 fps ceiling; no 60 fps on the main sensor.
- **High-speed (slow-motion) buckets:** `1920×1080 @ 30/120`, `1280×720 @ 120` and
  `720×480 @ 120` → **120 fps slow motion up to 1080p** is available at HAL level.
- `availableSlowMotion = [0 1 4]` (three supported slow-motion modes).
- **Video stabilization: `[0]` only** → **no EIS/OIS shipped**; software-gyro
  stabilization is not exposed by the HAL. Handheld video will be shaky.

## 4. Front camera — OmniVision OV8856 (8 MP, native)

| Property | Measured value |
|---|---|
| Active array | **3264 × 2448** (8 MP, native) |
| Aperture | **f/2.0** |
| Focal length | **2.76 mm** |
| Physical sensor size | 3.60 × 2.70 mm (~1/4") |
| CFA pattern | RGGB |
| Optical stabilization | none |
| Hardware level | LIMITED |

The front sensor's negotiated output sizes top out at **3264 × 2448** and include
full **1920×1080 (1080p)** record buckets plus `1280×720`, `720×480`, etc. — so the
front camera can record **1080p**, wider than the 720p figure most spec sheets
list. Its AE ranges climb to 30 fps and `availableVideoStabilizationModes` is
likewise `[0]` (no EIS).

## 5. AI / Scene / HDR feature gate (vendor Unisoc keys)

Read from `dumpsys media.camera` `com.addParameters.*` on the **rear** camera
(front carries the same flags where supported):

| Feature | Value | Meaning |
|---|---|---|
| `sprdAvailableAutoHdr` | `1` | **Auto HDR supported** |
| `sprdAvailableAIScene` | `2` | **AI scene recognition supported** (level 2 engine) |
| `sprdSNReady` | `1` | **SuperNight (night) mode ready** |
| `sprdAvailableGesturedetect` | `1` | gesture detection available |
| `isHdrSceneForAuto` | `1` | HDR auto-detection by scene |
| `availableMeteringMode` | `0 1 2 3` | center / spot / matrix / evaluative |
| `availableSlowMotion` | `0 1 4` | slow motion supported |
| `videoSnapshotSupport` | `1` | frame-grab during video |
| `sprd3availableAuto3dnr` | `0` | **no 3D noise reduction** |
| `sprdUltraWideId` | `255` | **no ultrawide camera** (`255` = none) |
| `sprdCamIdSensorRoleType` | `0` | main role (no wide/macro split) |
| `sprdAIScene` (current) | `0` | engine idle until a scene triggers |

`availableSceneModes` on the rear lists `1,5,2,3,4,100,101,102,103,104,18` —
the standard auto / night / portrait / landscape set plus the vendor custom
modes (`100+`) used by the realme camera app (Night, AI, etc.).

## Summary table

| Camera | Sensor | Aperture | Focal | Active array | Output sizes | Video |
|---|---|---|---|---|---|---|
| Rear main | Samsung S5KJN1 | f/1.8 | 4.05 mm | 4080×3072 | still to 4080×3072; **hidden 4K** | 30 fps; 120 fps slow-mo to 1080p |
| Rear depth | passive aux @ 0x43 | — | — | — | depth only | — |
| Front | OmniVision OV8856 | f/2.0 | 2.76 mm | 3264×2448 | still 3264×2448; 1080p | 30 fps max; no EIS |

None of the three mounted lenses is a true ultrawide or macro; the "dual rear
camera" second element is the fixed-focus depth assistant. No camera on the
device has optical or electronic stabilization.

Licensed CC BY-SA 4.0 together with the rest of this repository.
