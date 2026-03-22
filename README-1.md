# Qu4tro BLE Dashboard

A reverse-engineered web app for the **Acton Blink Qu4tro** electric skateboard, built after Acton went bankrupt and their servers went offline.

This project gives you back full control over your board — **no app, no account, no servers needed**.

![Dashboard screenshot placeholder](screenshot.png)

> ⚠️ **Work in progress — not yet tested against a real board.**
> This project was built with the help of [Claude](https://claude.ai) (Anthropic AI) through static analysis of the official APK. The protocol, field mappings, and dashboard code have not yet been verified against a live Qu4tro. If you try it and it works (or doesn't), please open an issue with your findings.

---

## Background

Acton went out of business and took their backend servers with them. The official app requires login via their servers (`api.actonglobal.com` and `parse.com`), both of which are now offline. This project reverse-engineers the BLE protocol directly from the official APK so you can configure and monitor your board without any server dependency.

The BLE communication itself was always local — the server was only needed for login, leaderboards, and OTA updates. Everything else works fine offline.

---

## Features

- 🔋 Live battery percentage and voltage
- ⚡ Speed readout (from RPM)
- 🌡️ Battery temperature and current draw
- 📏 Estimated remaining range
- 🎚️ Ride mode switching (Beginner / Normal / Performance)
- 💡 Light control (off / front / front+rear)
- 📱 Runs in Chrome/Edge on Android — no app install needed

---

## Requirements

- Android phone with **Chrome** or **Edge** browser
- The HTML file hosted on **HTTPS** (Web Bluetooth does not work from local files)
- Acton Blink Qu4tro board powered on

> **Web Bluetooth is not supported in Safari, Firefox, or any iOS browser.**

---

## Quick Start

### Option 1 — GitHub Pages (recommended)

1. Fork or clone this repository
2. Go to **Settings → Pages → Branch: main → Save**
3. Visit `https://yourusername.github.io/qu4tro-dashboard` on your Android phone in Chrome
4. Tap **Connect via Bluetooth** and select your board from the list

### Option 2 — Any HTTPS host

Upload `index.html` (rename `qu4tro-dashboard.html` to `index.html`) to any static hosting service:
- [Netlify Drop](https://app.netlify.com/drop) — drag and drop, instant HTTPS
- [Vercel](https://vercel.com)
- Your own web server with a valid SSL certificate

### Option 3 — Local development

```bash
# Python 3
python -m http.server 8080
```

Then open `http://localhost:8080` in Chrome on the **same machine**. `localhost` counts as a secure context. For access from your phone on the same WiFi, use your PC's local IP — note that `192.168.x.x` is **not** a secure context, so Web Bluetooth will be blocked. Use GitHub Pages or Netlify for phone testing.

---

## Connecting to the Board

1. Power on your Qu4tro
2. Open the dashboard URL in Chrome on Android
3. Tap **Connect via Bluetooth**
4. The Qu4tro advertises as `BQ` followed by its serial number (e.g. `BQ451800617`). Other Blink models may use `blink_s1`, `blink_s2`, `blink_lite`, or `BLINK`
5. Select it from the list and tap **Pair**

If no devices appear in the list:
- Make sure the board is powered on and not already connected to another device
- Grant Chrome **Location permission** in Android settings (required for BLE scanning)
- Toggle Bluetooth off and on
- If you still see nothing, try the diagnostic mode below

### Diagnostic mode — find your board's name

If the device list is empty, the board may advertise under a different name. Edit the `connect()` function in the HTML and temporarily replace the filters with:

```javascript
device = await navigator.bluetooth.requestDevice({
  acceptAllDevices: true,
  optionalServices: [SERVICE_UUID]
});
```

This shows all nearby BLE devices. Once you know your board's name, update the `filters` array accordingly.

---

## Protocol

The full reverse-engineered protocol is documented in [`PROTOCOL.md`](PROTOCOL.md).

**Short version:** The board uses an HM-10 BLE module with standard UUIDs:

| | UUID |
|---|---|
| Service | `0000ffe0-0000-1000-8000-00805f9b34fb` |
| Characteristic | `0000ffe1-0000-1000-8000-00805f9b34fb` |

All messages are **plain ASCII text** in a pipe-delimited format:

```
AS | KEY | VALUE | KEY | VALUE | AE
```

Examples:
```
AS | M | NORMAL | AE          ← set ride mode
AS | B7 | 1 | AE              ← lights on
AS | M | QUERY | AE           ← request status
```

---

## Files

| File | Description |
|---|---|
| `qu4tro-dashboard.html` | The dashboard PWA — rename to `index.html` for deployment |
| `PROTOCOL.md` | Full reverse-engineered protocol reference |
| `README.md` | This file |

---

## How it was reverse-engineered

The official Android APK (`ACTON_App_1_7_1`) was decompiled using static analysis of the DEX bytecode:

1. The APK was unpacked and `classes.dex` / `classes2.dex` were extracted
2. The DEX string table was parsed to identify BLE UUIDs, class names, and field names
3. The `com.acton.r.sdk` package was located, containing `Skate`, `SkateMode`, `SkateControl`, `LightMode`, and `SkateScanner`
4. Enum values were extracted directly from static field definitions
5. Method bytecode was decoded to identify the ASCII packet format (`AS | ... | AE`) and field keys

Key findings:
- No binary protocol — all communication is UTF-8 ASCII
- No authentication required for BLE — the server was only used for the social/account features
- The HM-10 module uses standard off-the-shelf UUIDs

---

## Known limitations

- **Telemetry field scaling** (RPM → km/h, voltage scaling) was inferred from field names and may need calibration against a live board
- **Pairing** uses standard BLE — no custom pairing sequence required
- The Qu4tro advertises as `BQ` + serial number (confirmed). Other Acton Blink boards (`blink_s1`, `blink_s2`, `blink_lite`) are untested

---

## Contributing

If you have an Acton board and can capture BLE traffic, please open an issue or PR with:
- Your board model
- Raw packet dumps from a BLE sniffer (nRF Sniffer, Wireshark + BLE USB dongle)
- Any corrections to the field mappings in `PROTOCOL.md`

---

## Disclaimer

This project is the result of reverse engineering for interoperability purposes under the EU Software Directive (2009/24/EC) and similar provisions. Acton is no longer in business. No proprietary code is included — only protocol documentation and an independently written client.

Use at your own risk. The ride controls (throttle/brake) have been deliberately excluded from this app for safety reasons.

---

## License

MIT — do whatever you want with it, just don't blame me if something goes wrong.
