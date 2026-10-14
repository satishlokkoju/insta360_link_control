# Hardware verification

Tests run on a real camera to check the behaviour described in [protocol.md](protocol.md).

- Camera: Insta360 Link (`2e1a:4c01`), firmware `v1.4.8.8_build2` (selector 3, bytes 0x61–0x6F)
- Host: Ubuntu, kernel 7.0, stock `uvcvideo`
- Date: 2026-10-12

Selector numbers refer to the main extension unit (`faf1672d…`, unit 9).

## Selector 27 is a bitmask

| Step | Write | Readback | Kernel log | Result |
|---|---|---|---|---|
| State left by `linkctl feature portrait on` + `flip on` | – | `0x0038` (bits 3, 4, 5) | – | mirror + gestures + portrait on; camera offered 1080×1920, 1088×1920, 736×1280 |
| Clear bit 3 only (mirror) | `30 00` | `0x0030` | nothing | no reboot, change applied |
| Clear bit 5 (portrait) | `10 00` | `0x0010` after re-enumeration | `SET_CUR … control 27 on unit 9: -71`, USB disconnect, re-enumerate ~6 s later | intended reboot; portrait formats gone, value persisted |

Conclusion: the `-71` "crash" is the firmware saving the portrait bit and restarting. Every id in
`linkctl.camera.FEATURES` has bit 3 set, so every `feature` call turned mirroring on, and "on"/"off"
send the same first byte.

### Every flag, one at a time

With video streaming, each flag was set and cleared through `linkctl feature`. The readback
was checked each time, and the kernel log was watched for restarts.

| Flag | Bit | Set → readback | Visible effect |
|---|---|---|---|
| ai-zoom | 0 | `0x0011` | not checked |
| hdr | 2 | `0x0014` | not checked |
| flip | 3 | `0x0018` | image mirrored |
| gesture-control | 4 | cleared → `0x0000` | not checked |
| startup-params | 6 | `0x0050` | not checked |
| tap-track | 7 | `0x0090` | not checked |
| auto-track | 8 | `0x0110` | does not start subject tracking |
| smart-adjust | 9 | `0x0210` | not checked |
| lock-vertical | 10 | `0x0410` | none observed |
| privacy | 11 | unchanged `0x0010` | write ignored |
| vflip | 12 | unchanged `0x0010` | write ignored |

None of these restarted the camera. Setting portrait (bit 5) restarted it. The camera then offered
1080×1920, 1088×1920 and 736×1280, and a 1080×1920 capture was a correctly oriented portrait
image. Clearing bit 5 restarted it again, back to landscape formats. The other flags were kept
both times.

## Selector 26 order

Read while the gimbal was parked: `(-277200, 21960)` arcseconds = (−77°, +6.1°). The parked
position is tilt ≈ −77°, so the layout is **`{int32 tilt, int32 pan}`**, as linkctl already
assumes.

## Selector 14 is a settings reset

Procedure: snapshot all settings → change some → write `01` to selector 14 → snapshot → unplug and
re-plug → snapshot.

- The write succeeds; no reboot, no immediate change in any readback.
- After re-plug:

| Setting | Before | Changed to | After reset + re-plug |
|---|---|---|---|
| Brightness / contrast / sharpness | 50 / 50 / 50 | 60 / 55 / 40 | 50 / 50 / 50 |
| Noise cancellation (sel 7) | 1 | 0 | 1 |
| Tracking speed (sel 18) | 1 | 2 | 1 |
| Selector 27 | `0x0010` | `0x0018` (mirror on) | `0x0190` (gestures, tap-track, auto-track) |
| Gesture mask (sel 5) | `0x00` | `0x02` | `0x0E` |
| Power-line frequency | 2 (60 Hz) | 1 (50 Hz) | 3 (see below) |
| White-balance temperature | 4149 | – | 5500 |
| Pan / tilt / zoom, mode | – | – | unchanged |

Conclusion: selector 14 restores factory settings, applied at the next power-up. It is not a gimbal
re-home. The GUI "centre" button is unaffected: it uses V4L2 pan/tilt/zoom (`LinkCamera.center()`).

## Other observations

- **Power-line frequency:** the camera reports `min=0 max=2 default=3`. After the reset the stored
  value is 3 (auto), but `uvcvideo` rejects setting 3 (`Numerical result out of range`).
- **Selector 5:** writing `0x03` reads back `0x02`; the firmware reports only the gesture groups it
  actually enabled.
- **V4L2 `pan_absolute`** reads `34116348` (≈ 9477°) while selector 26 reports the real pan (≈ 6°).
- **Selector 11** (status) read `2b 27 00 00 00`; byte 0/1 look like temperatures (43 / 39 °C).
- **Selectors 25 / 29 / 30** in auto exposure: ISO 2604 → 100 after reset, shutter 30 → 1000, AE mode 2.
