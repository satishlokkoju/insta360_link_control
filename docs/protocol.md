# Extension-unit protocol notes

Notes on the Insta360 Link camera's UVC extension units. The **Verified** column is "yes" when
the row was checked on an Insta360 Link (`2e1a:4c01`, firmware `v1.4.8.8_build2`). The results
are in [hardware-verification.md](hardware-verification.md). Rows marked "no" have not been tested
on hardware yet: treat them as a starting point, not as fact. Nothing here has been checked on the
Link 2.

Selector lengths can differ between models and firmware versions. Always ask the device with
`GET_LEN`.

## Main unit (`faf1672d…`, unit 9)

| Sel | Len (Link 1) | Meaning | Verified |
|---|---|---|---|
| 1 | 4 | Factory/service hook. Do not write. | no |
| 2 | 52 | Mode record, described [below](#selector-2-mode-record) | partly |
| 3 | 170 | Identity block. The firmware version string is at bytes `0x61`–`0x6F`. Writable and persisted, so never write it. | read only |
| 4 | 262 | Raw command channel to the gimbal controller. Writes can miscalibrate the gimbal. Never write. | no |
| 5 | 1 | Gesture-group enable mask. The readback shows only the groups that were actually enabled. | yes |
| 6 | 5 | Gesture bindings, one byte per slot. Only `0`, `1` and `2` are accepted; other values are ignored. | no |
| 7 | 1 | Microphone noise cancellation (`0`/`1`). Reset sets it to `1`. | yes |
| 9 | 2 | EV compensation. Older firmware: int8 EV×10. Newer firmware: int16 EV×100. | no |
| 10 | 129 | USB personality / photo-mode switch. Writes restart the camera. Never write. | no |
| 11 | 5 | Status, read only. Bytes 0–1 look like temperatures. Byte 4 is set when a firmware update file is present. | read only |
| 12 | 32 | Serial number. Writable and persisted, so never write it. | no |
| 13, 23 | 129 | Licence/key blobs. Never write. | no |
| 14 | 1 | **Factory settings reset**, applied at the next power-up. Not a gimbal re-home. | yes |
| 15 | 12 | Autofocus mode and lens information | no |
| 16 | 255 | Tone-curve upload | no |
| 17 | 1 | USB mode switch (`0` uvc, `1` photo, `2` mass storage, …). The camera restarts. Never write. | no |
| 18 | 1 | Tracking speed, `0`–`3`. Reset sets it to `1`. | yes |
| 19 | 1 | Tracking framing: `1` head, `2` upper body, `3` full body | yes (Link 2) |
| 20 | 240 | Detected people, read only: a count followed by normalised rectangles | no |
| 21 | 8 | Tracking target point (2 × float x, y) | no |
| 22 | 4 | Continuous pan/tilt `[pan_dir, pan_speed, tilt_dir, tilt_speed]`. Direction: `0` stop, `1` negative, `0xFF` positive. Speed: `1`–`10`. | no |
| 24 | 4 | Framing offset, 2 × int16. Writing `(0x7FFF, 0x7FFF)` clears it. | no |
| 25 | 2 | **ISO**, uint16. In auto exposure it reads back the current ISO. | read only |
| 26 | 8 | Absolute position `{int32 tilt, int32 pan}` in arcseconds | yes |
| 27 | 2 | **Feature flags**, a uint16 bitmask described [below](#selector-27-feature-flags) | partly |
| 28 | 10 | Tracked-target info, read only | no |
| 29 | 2 | Shutter, uint16 `N` for an exposure of 1/N s | read only |
| 30 | 1 | Exposure mode. `1` is manual (ISO and shutter come from selectors 25 and 29). | read only |

### Selector 27: feature flags

A little-endian uint16. `GET_CUR` returns the current state. `SET_CUR` replaces **every** flag at
once, so always read the value, change one bit, and write it back. The flags are persisted across
power cycles.

| Bit | Feature | Verified |
|---|---|---|
| 0 | AI zoom | stored |
| 1 | unused | – |
| 2 | HDR | stored |
| 3 | Horizontal mirror | yes |
| 4 | Gestures (master switch) | yes (set by reset) |
| 5 | Portrait resolutions (9:16 formats). **Changing it restarts the camera.** | yes |
| 6 | Save current parameters as startup parameters | stored |
| 7 | Single-tap tracking | yes (set by reset) |
| 8 | Auto tracking. Does not start subject tracking by itself (use mode 1). | stored |
| 9 | Smart adjustment | stored |
| 10 | Lock the gimbal in vertical orientation (Link 1). No visible effect observed. | stored |
| 11 | Privacy: gimbal points down. Ignored by the Link 1. | yes (Link 1 ignores) |
| 12 | Vertical flip. Ignored by the Link 1. | yes (Link 1 ignores) |
| 13 | Low-resolution mode. **Changing it restarts the camera** (not on Link 1). | no |
| 14 | Pause-tracking area | no |
| 15 | Audio volume | no |

"Stored" means the bit can be set and cleared and reads back, without restarting the camera,
but its effect has not been confirmed. After a factory reset the value is `0x0190`
(bits 4, 7, 8).

When bit 5 changes, the camera saves the setting and restarts. The `SET_CUR` request fails with
`EPROTO` (`-71`) and the device re-enumerates about 6 s later. That is expected behaviour, not a
crash. With bit 5 set, the camera adds 1080×1920, 1088×1920 and 736×1280 formats. Streaming one of
them gives a correctly oriented portrait image; landscape formats are not offered while it is set.

### Selector 2: mode record

On the Link 1 it is 52 bytes. The length differs on other models and firmware versions.

| Offset | Content |
|---|---|
| 0 | Mode. On read, `0xFF` means a mode change is still being applied. |
| 1 | Sub-state on read. Ignored on write. |
| 2–37 | Whiteboard corner data. Writing zeros clears it. |
| 38–49 | 3 × int32 gimbal position. The value `3610` means "leave unchanged". |
| 50–51 | Zoom, 100–400 |

A write also applies bytes 38–51. To change only the mode, put `3610` in the position fields, or
write back the current values. After a mode change, wait until byte 0 reads back the new mode
before sending zoom or pan/tilt.

| Mode | Meaning | Verified |
|---|---|---|
| 0 | Normal | yes |
| 1 | AI tracking | yes (Link 2) |
| 4 | Whiteboard | yes (Link 2) |
| 5 | Overhead | yes (Link 2) |
| 6 | DeskView | yes (Link 2) |
| 7 | Auto-framing (multi-person) | no |
| 8–11 | Smart whiteboard | no |

## Other units

- **Unit 10 (`e307e649…`):** tracking metadata and diagnostics. Do not write.
- **Unit 11 (`a8bd5df2…`):** auto-framing on/off (sel 2), six on-camera presets (sel 3), save
  preset 0–5 (sel 4), go to preset 0–5 (sel 5). Not verified.

## Standard UVC controls

Pan, tilt, zoom, focus, brightness, contrast, saturation, sharpness, white balance and power-line
frequency are ordinary V4L2 controls. Quirks on the Link 1:

- `power_line_frequency` reports `max=2 default=3`. Value 3 (auto) is what a reset stores, but
  `uvcvideo` refuses to set it.
- `pan_absolute` reads `34116348` after power-up until the first pan/tilt write. Selector 26 has
  the real position.
- Hue is accepted but has no visible effect. (Not verified.)

## Model differences (not verified)

| | Link 1 (`4c01`) | Link 2 (`4c04`) |
|---|---|---|
| Lock vertical (sel 27 bit 10) | yes, firmware ≥ v1.3.1.8 | no |
| Low resolution (sel 27 bit 13) | no | yes |
| Factory reset | sel 14 = `01` | sel 14 = `0x81` |
| Detected people | unit 9 sel 20 | unit 10 sel 24 |
| Gesture "rock" (sel 5 bit 4) | no | yes |

Other product IDs: `4c03` Link 2C (no gimbal), `4c05` Link 2C Pro, `4c06` Link 2 Pro.
