# usbmuxd (ideacatlab fork) — per-device AV-mode reset for QuickTime-over-USB

Fork base: **libimobiledevice/usbmuxd @ `3ded00c`** (= the exact build running on ALI
pod `ascensionpod1`: `usbmuxd 1.1.1-72-g3ded00c`). The Valeria / device-mode support is
**already upstream** here — this fork adds an **on-demand, per-device AV-session reset**
so the ALI QuickTime-USB H.264 stream ([[ideacatlab/quicktime_video_hack]]) can be
(re)armed reliably while WDA control keeps running.

## How Apple device-modes work in usbmuxd (src/usb.c, src/usb.h)
Apple phones expose USB "modes" selected by **vendor-specific control transfers**:
- `APPLE_VEND_SPECIFIC_GET_MODE = 0x45`, `APPLE_VEND_SPECIFIC_SET_MODE = 0x52` (usb.h:54-55)
- Modes (usb.c comments): **1 = initial** (4 configs, normal usbmux on config 4),
  **2 = "Valeria"** (adds **config 5** with the H.264/H.265 screen-capture AV interface —
  this is what macOS QuickTime screen-recording activates), 3 = CDC-NCM, etc.
- `USBMUXD_DEFAULT_DEVICE_MODE` env (`ENV_DEVICE_MODE`) sets the desired mode. We run
  the fleet with `=2`.

Flow on device add (`usb_device_add` ~l.700 → `get_mode_cb` l.684 → `switch_mode_cb`):
1. usbmuxd sends GET_MODE (0x45), reads current mode.
2. If `guessed_mode != desired_mode (2)`, it sends **SET_MODE (0x52) wIndex=desired_mode**
   via `submit_vendor_specific` (l.368) → device re-enumerates into Valeria/config 5.
3. Both the usbmux/control iface (`5.1`, subclass 0xFE) and the AV iface (`5.2`, 0x2A)
   then live on config 5 → **WDA control + QuickTime video coexist**.

## The problem we hit (root cause)
- QVH's `EnableQTConfig` sends `Control(0x40, 0x52, 0x00, 0x02)` — that is **the exact
  same SET_MODE 0x52 command** usbmuxd uses. So QVH and usbmuxd both drive the mode, and
  fight.
- After a QVH session the device gets **stuck "audio-only"**: it answers PING + sends
  sparse `SYNC_TIME` but never a fresh `CWPA` (audio-clock anchor) → no `HPD1` → no video.
- QVH's soft re-cycle (`SET_MODE 0` then `SET_MODE 2`) **cannot** clear it — `wIndex 0` is
  not a real mode, so it never performs a true mode transition. QVH also cannot
  `SetConfiguration`/USB-reset the device because **usbmuxd holds the interface** (`busy
  [-6]`, and direct libusb hits `timeout [-7]` from contention).
- Only a full `usbmuxd` restart (which re-runs GET_MODE→SET_MODE on every device) reliably
  produces a fresh AV session — too coarse, and it blips the whole fleet + the WDA control
  path (caused a real control-lag regression on the pod).

## The fix (this fork)
**Do the AV-session reset where it belongs — inside usbmuxd, the one process that owns the
device** — as a clean `SET_MODE 1 → SET_MODE 2` transition (initial → Valeria), on demand,
**per device**, without touching the rest of the fleet.

Design:
1. Add an internal `device_reset_mode(usb_device *dev)` in `usb.c` that issues
   `SET_MODE wIndex=1` then (after the device settles / on the switch callback)
   `SET_MODE wIndex=2`, reusing `submit_vendor_specific` + the existing `switch_mode_cb`
   machinery. This forces a real Valeria re-entry → fresh `CWPA` on the next QVH connect.
2. Expose it on demand. Two options (pick per integration taste):
   - **New usbmux client request** `"ResetDeviceMode"` (plist over the usbmux socket,
     handled in `client.c` → `device.c`), keyed by `DeviceID`/udid. Cleanest; lets the
     capture manager call it just before `qvh record`.
   - **Or** a lightweight control: SIGUSR1 + a udid file, or a tiny `tools/` helper that
     talks the usbmux protocol. Faster to prototype.
3. Capture manager ([[ideacatlab/quicktime_video_hack]] `deploy/server.js`) calls reset →
   waits for re-attach → `qvh record`. The CWPA-gated retry in QVH then arms first try.

Code touch-points: `src/usb.c` (`usb_device_add`, `get_mode_cb`, `switch_mode_cb`,
`submit_vendor_specific`), `src/usb.h` (`APPLE_VEND_SPECIFIC_SET_MODE`), and for the
client command `src/client.c` + `src/device.c` + `src/usbmuxd-proto.h`.

## Test plan — on a DEDICATED rig, NOT a control-serving pod
usbmuxd contention is the enemy of clean claims; doing this RE on a busy pod degrades the
fleet's control path. Use a spare Linux box + 2-3 jailbroken iOS≤16 phones where usbmuxd
can be stopped/instrumented. Validate: stuck device → `ResetDeviceMode` → fresh CWPA →
video, repeatable, with WDA control unaffected.

## Bigger picture (why fork usbmuxd at all)
The ALI fleet has chronic usbmuxd pain (goes "blind", contention, distro-version breakage,
external watchdog hacks). Owning a fork lets us also: add a clean health/restart RPC, fix
the blind-state detection, and converge the fleet on one known-good build — alongside this
AV-reset feature. See the master POC writeup in [[ideacatlab/quicktime_video_hack]]
`FINDINGS_IDEACATLAB.md` + ALI-pod `docs/research/`.
