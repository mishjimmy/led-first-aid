# LED test bench

A browser-based bench tool for testing LED tape in the field. It connects over Bluetooth to a portable power supply and to one or two cheap BLE LED controllers. From one page you can set the supply voltage, read live power draw, and drive the tape through colors and test patterns.

The whole app is a single self-contained HTML file (`led-test-bench.html`): vanilla JS and CSS with no build step and no dependencies. The only external request is Google Fonts, which has system-font fallbacks.

## Running it

Open `led-test-bench.html` directly in **Chrome or Edge** (desktop, Windows 10+), or serve it from `localhost`. Two conditions apply:

- It needs **Web Bluetooth**, which requires a secure context (`file://` or `localhost` both qualify) and a Chromium browser. Firefox and Safari don't support it. On iOS, only the Bluefy browser works.
- It **must run as a top-level page**. Web Bluetooth is usually blocked inside iframes and embedded viewers, so hosted previews won't work.

Close the phone apps (Happy Lighting, PolyLink) before connecting. Each device accepts only one BLE connection at a time.

## Hardware

| Role | Device | Notes |
|---|---|---|
| Power supply | ISDT MP305 (developed against MP305B) | 0–30 V, 0–5 A, 150 W linear supply with an internal battery and BLE |
| RGB controller | Triones-firmware BLE LED controller, e.g. SUPERNIGHT RGBW/RGB (Amazon B08SJ513KR) | 12–24 V, rated 10 A on RGB and 12 A on W; used in RGB mode |
| White controller (optional) | A second identical Triones controller | Used in white mode only |

### Why two controllers

Triones firmware treats RGB and white as **mutually exclusive modes**. The mode byte in the color command is `F0` for RGB or `0F` for white, and most firmware ignores `FF` (mix). One controller therefore can't run RGB and W at the same time. The tool gets around this by using two controllers as one RGBW fixture:

- Power both controllers from the same supply so they share a ground (the outputs switch the low side).
- Strip V+ connects to controller A's V+ output.
- Strip R, G and B connect to controller A's R, G and B.
- Strip W connects to controller B's **W** terminal. Controller B stays in white mode permanently.
- Controller B's V+ output is left unconnected.

Known side effects: a few tens of milliseconds of skew between the two BLE writes, unsynchronized PWM clocks (can band on camera), and independent power-on states.

## Features

### Power supply panel
- Live readout of volts, amps and watts from 0xC3 state frames, plus the voltage setpoint and current limit.
- 5 V, 12 V and 24 V presets. The current limit is always set to the maximum available, `min(5 A, 150 W / V)`.
- An available-power figure (`V × I_limit`), a load bar (turns red above 90%), and headroom in watts. The status line shows when the supply is current-limiting.
- Output on/off toggle.
- Safety check: raising the voltage while the output is live asks for confirmation.
- A "Release to front panel" button sends `remoteCon=0`.

### Light panel
- R, G, B and W faders plus a master fader, a color picker (sets R/G/B), and an output color preview.
- An **in-use checkbox** under each color fader. An unchecked channel always outputs zero, is skipped by patterns, is excluded from All full, and its Solo button is disabled. This is for testing RGB-only tape.
- Solo buttons (one channel at full, for reading per-channel current on the PSU), All full, and Blackout.
- Test patterns with an adjustable step time (0.1–3 s):
  - **Rainbow**: crossfades between the enabled channels in order. The incoming channel ramps to full while the outgoing one holds, then the outgoing one fades out, so intermediate mixes (yellow, cyan, etc.) are visible at full brightness.
  - **Chase**: hard steps through the enabled channels.
  - **Fading chase**: each enabled channel fades up and back down in turn (gamma 2.2).
- Master applies to patterns. Touching any color fader or the picker stops the running pattern.
- Controller mapping:
  - Both connected: RGB goes to A (`F0`) and W goes to B (`0F`), for full RGBW mixing.
  - Only A connected: white-only uses `0F`, RGB-only uses `F0`, and both at once tries `FF`, which most firmware ignores.
  - Only B connected: white only.

### Log
Shows connection events, the first two raw PSU state frames, control results, and errors. It's capped at 400 lines.

## Code layout

Everything lives in the `<script>` block of `led-test-bench.html`, in this order:

1. **Helpers**: `$`, `hex`, `sleep`, `log`, `setChip`.
2. **`class Triones`**: connect, disconnect, and a latest-wins write queue. While a write is in flight, only the newest frame is kept, and duplicates of the last-sent frame are skipped. Two instances exist: `ctlA` (RGB) and `ctlB` (white).
3. **Light state**:
   - `lv` holds the manual levels plus master; `out` holds the levels currently being output, before master; `used` holds the in-use flags; `eff(k)` returns the effective level after the in-use flag.
   - Fader DOM is generated from `chans`.
   - `render()` updates the UI, and `pushOutput(forceAll)` maps levels to controller commands.
4. **Patterns**: `startPattern`, `stopPattern` and `tick()` (40 ms interval). Phase accumulates, so changing the speed mid-run doesn't jump.
5. **MP305**:
   - `psuConnect`: connect, bind, request remote control, start polling.
   - `psuRx`: dispatches notifications and resolves `waitFor` promises.
   - `parseState`, `renderPsu`, `controlFrame`, `psuRequestRemote`, `psuControl`.
   - All PSU writes are serialized through a promise chain (`psuWrite`).
6. **Connect buttons and startup.**

The styling is a dark lighting-console look (Barlow / Barlow Semi Condensed), with color tokens on `:root`.

## Protocol reference

### Triones (Happy Lighting) controllers

- Service `0xFFD5`, write characteristic `0xFFD9` (write without response when supported).
- Notify service `0xFFD0`, characteristic `0xFFD4` (status replies; not used by the current tool).
- Advertised names start with `Triones`, `LEDBLE`, `Dream`, `BRGlight` or `QHM`. Clones vary, which is why there's a "Show all Bluetooth devices" option.

| Command | Bytes |
|---|---|
| Power on | `CC 23 33` |
| Power off | `CC 24 33` |
| Color | `56 RR GG BB WW MM AA`, where `MM` = `F0` RGB, `0F` white, `FF` mix (usually ignored) |
| Status query | `EF 01 77` → `66 … 99` (12 bytes: device type, power `23`/`24`, mode, …, R, G, B, W, firmware version) |

This controller family is **verified working** with this tool.

### ISDT MP305

Source: [nemanjan00/pymp305](https://github.com/nemanjan00/pymp305) (`PROTOCOL.md` and `CHANGELOG.md`), which was reverse-engineered from ISDT's WebLink app and firmware. Its BLE reads and control are hardware-verified upstream on an MP305B.

**GATT**
- Service `0000af00-…`.
- `AF01`: commands and notifications.
- `AF02`: binding and hardware info.
- `FEE0`/`FEE1`: OTA. Don't touch.
- Advertised name: `0000MP305B` / `0000MP305A`.

**Framing**
- Commands to `AF01` are `[0x12, cmd, …payload]`, with a little-endian payload and no length or checksum.
- `AF01` responses start with **`0x31`**, not `0x12`: `[0x31, cmd, …payload]`, with the command at index 1 and the payload at index 2.
- `AF02` handshake frames are bare, with the command at index 0.

**Session**
1. Bind: write `[0x18, 16 random bytes, 0, 0]` to `AF02`. The supply shows a prompt on its screen; after the user accepts, it replies `[0x19, …]`.
2. After a 0.5 s gap, write `[0xE0]` to `AF02` to get the hardware info reply `[0xE1, …]`.
3. **Take control**: send `0xC8` with `remoteCon=2` and every other field zero. Over BLE, the supply shows an "allow remote control" prompt, and `0xC9` status `0` arrives only after the user accepts. Wait up to 45 s.
4. Control: send `0xC8` with `remoteCon=1`. If the reply is status `1`, control was dropped; re-request it once and retry.
5. Poll `0xC2` for `0xC3` state frames every **500 ms**. Don't poll faster, because the firmware stops answering. `0xBD` ("realtime") never replies on this firmware; don't use it.
6. Release control with `remoteCon=0`, which also reverts the supply to DC mode.

**`0xC8` control payload (13 bytes after `0x12 0xC8`)**
```
remoteCon   u8   2=request, 1=apply (holding control), 0=release
setVoltage  u16  V × 100
setCurrent  u16  A × 1000
realChange  u8   1=V, 2=I, 3=both
voltageSlow u8
currentOver u8   OCP enable
output      u8   1=on
model       u8   0=DC, 1=programmable, 2=USB-PD, 3=charge
refresh     u8
```
The `0xC9` reply status byte (index 2) is 0 for accepted, 1 for rejected or no remote control, and 2 for pending.

**`0xC3` state frame** (fields start at index 2)
```
outState u8 (1=CV, 2=CC), battState u8, battPct u8,
voltage u16 /100, setVoltage u16 /100, current u16 /1000, setCurrent u16 /1000,
workingTime u32 (s), energy u32 /1000 (Wh), power u16 /100,
currentOver u8, realChange u8, voltageSlow u8, output u8, model u8,
voltageBoard u8, currentBoard u8, temperature u8 (°C)
```
A frame needs at least 30 bytes. Shorter frames mean the BLE MTU is too small, and the tool logs this.

**Quirks**
- Lowering the current limit while the output is on doesn't engage CC until the output is cycled.
- USB-HID uses the same command set with length, `0xAA` stuffing and a checksum. That path is fully verified upstream and is a fallback option (WebHID, VID `0x28E9`).

## Status and known issues

- **The MP305 remote-control handshake is not yet confirmed in this tool.** The two-step `remoteCon=2`/`remoteCon=1` flow and the on-screen prompts follow the upstream notes. Earlier builds got `31 C9 01` (rejected) because they skipped the request. If control still fails, check the log for "Remote control granted".
- The MP305's `outState` (CV/CC) is parsed but not displayed. CC is inferred from measured current versus the limit instead.
- With a single Triones controller, W combined with RGB depends on `FF` support, which most units lack.
- Reconnecting always goes through the browser device picker; `navigator.bluetooth.getDevices()` isn't used.
- No persistence: presets, the in-use flags and pattern speed reset on reload.

## Development notes

- Keep it a **single self-contained HTML file** with no build step and no npm. External scripts, if ever needed, should load from a CDN with pinned versions.
- Test manually against real hardware. The log is the main debugging tool. Protocol changes should log raw frames (`hex()`) the first time they fire.
- BLE writes: always go through the per-device queue (`Triones.send`/`force`, `psuWrite`). Parallel GATT operations on one device throw "GATT operation already in progress".
- The UI must stay usable at narrow widths (layout collapses below 900 px and 700 px) and keep visible keyboard focus.
- Safety conventions: the current limit always goes to maximum, voltage increases while live require confirmation, and a single-controller setup must never silently drop a requested channel without logging or showing it in the mode line.

## Roadmap

_Add planned features here._
