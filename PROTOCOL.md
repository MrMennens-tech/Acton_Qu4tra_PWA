# Acton Blink Qu4tro — Reverse Engineered BLE Protocol

Extracted from `ACTON_App_1_7_1_APKPure.apk` via static analysis of `com.acton.r.sdk`.

---

## 1. BLE Connection — No intermediate hardware required

The board contains an **HM-10 BLE module**. Connect directly from any phone or browser:

| | UUID |
|---|---|
| **Service** | `0000ffe0-0000-1000-8000-00805f9b34fb` |
| **Characteristic** | `0000ffe1-0000-1000-8000-00805f9b34fb` |

- **Write** to FFE1 → send commands to the board
- **Notify** from FFE1 → receive telemetry from the board
- Encoding: **UTF-8 ASCII text** (not a binary protocol)

---

## 2. Packet format

All messages (both TX and RX) use the following format:

```
AS | FIELD1 | VALUE1 | FIELD2 | VALUE2 | ... | AE
```

- `AS` = Array Start (packet begin marker)
- `AE` = Array End (packet end marker)
- `|` = field separator (pipe character with spaces)
- Fields and values alternate

### Example — command TX (app → board):
```
AS | M | NORMAL | AE
```

### Example — telemetry RX (board → app):
```
AS | V | 3780 | B3 | 42 | B1 | 15 | B6 | 1 | AE
```

---

## 3. Field keys

### TX (App → Board)

| Key | Meaning | Values |
|---|---|---|
| `M` | Ride mode | `BEGINNER`, `NORMAL`, `PERFORMANCE`, `QUERY`, `RESET` |
| `C` | Board control state | `GO`, `LOCK`, `UNLOCK`, `REMOTE`, `REMOTE_CONTROLLER`, `QUERY`, `BEGINNING`, `PUSH_TO_START`, `FAULT` |
| `P` | Packet type / power field | integer |
| `R` | Remote field | integer |
| `B1` | Byte field 1 | integer |
| `B2` | Byte field 2 | integer |
| `B7` | Light control | `0`=off, `1`=front, `2`=front+rear |

### RX (Board → App)

| Key | Meaning | Unit | Notes |
|---|---|---|---|
| `V` | Battery voltage | mV | Divide by 1000 for volts |
| `B3` | Battery capacity | % (0–100) | |
| `B1` | Battery temperature | °C | |
| `B4` | Current draw | mA | |
| `B6` | Current ride mode | `0`=Beginner, `1`=Normal, `2`=Performance | |
| `B7` | Light state | `0`=off, `1`=front, `2`=front+rear | |
| `R6` | Motor RPM | rpm | Divide by ~26.6 for km/h (needs calibration) |
| `R8` | Estimated remaining range | meters | |
| `R0` | Direction state | `0`=forward, `1`=reverse | |
| `BQ` | Side light state | | |
| `BS` | Direction state (secondary) | | |
| `AX` | Unknown | | |

> ⚠️ The RPM-to-speed conversion factor and some field units are inferred from field names in the SDK bytecode. Calibrate against a live board if precision matters.

---

## 4. Ride modes (SkateMode enum)

| Enum value | Description |
|---|---|
| `BEGINNER` | Limited to ~15 km/h, soft braking |
| `NORMAL` | Limited to ~25 km/h, standard response |
| `PERFORMANCE` | Full power, sharp throttle/brake response |
| `QUERY` | Request current mode from board |
| `RESET` | Reset to default settings |

---

## 5. Light modes (LightMode enum)

| Enum value | Description |
|---|---|
| `CLOSE` | All lights off |
| `NORMAL` | Default lighting |
| `OPEN` | All lights on |
| `OPENBEHIND` | Rear lights only |

> Note: The `B7` field key uses integer values (0/1/2) in the wire format, not the enum name strings. The LightMode enum is used internally by the app.

---

## 6. Heartbeat / keep-alive

The app sends a periodic heartbeat to maintain the connection and poll for status.
Send every ~2000ms:

```
AS | M | QUERY | AE
```

Or alternatively:
```
AS | C | REMOTE | AE
```

---

## 7. Command reference

### Set ride mode
```
AS | M | BEGINNER | AE
AS | M | NORMAL | AE
AS | M | PERFORMANCE | AE
```

### Control lights
```
AS | B7 | 0 | AE        (off)
AS | B7 | 1 | AE        (front lights)
AS | B7 | 2 | AE        (front + rear)
```

### Query current state
```
AS | M | QUERY | AE
```

### Lock / unlock board
```
AS | C | LOCK | AE
AS | C | UNLOCK | AE
```

---

## 8. Board advertisement names

The board advertises over BLE using names in the format:

| Device name | Model |
|---|---|
| `blink_q4` | Qu4tro (4-wheel) |
| `blink_s1` | Blink S1 |
| `blink_s2` | Blink S2 |
| `blink_lite` | Blink Lite |
| `BLINK` | Generic fallback |

Scan for devices with a name prefix of `blink` or `BLINK`.

---

## 9. Server dependencies (now offline)

The original app required two backend services which are no longer available:

| Service | URL | Purpose |
|---|---|---|
| Acton API | `https://api.actonglobal.com/v1` | User accounts, leaderboards, OTA firmware |
| Parse backend | `https://api.parse.com/1/` | Auth, social features |

**The BLE communication is entirely local and does not require any server.** A custom app only needs the UUIDs and packet format above to function fully.

---

## 10. How this was extracted

Static analysis of `classes.dex` from the APK:

1. DEX string table parsed → BLE UUIDs and class names identified
2. `com.acton.r.sdk` package located: `Skate`, `SkateMode`, `SkateControl`, `LightMode`, `SkateScanner`
3. Class definitions decoded → enum constant names extracted from static field definitions
4. Method bytecode decoded → `const-string` instructions revealed the ASCII packet format (`AS`, `AE`, `|`, field keys)
5. Instance field names cross-referenced with telemetry field keys → complete field map

---

## 11. What still needs verification

The following items were inferred from field names and bytecode structure. Verification against a live board with a BLE sniffer (e.g. nRF Sniffer or Wireshark + BLE USB dongle) would confirm or correct them:

- Exact RPM-to-speed conversion factor
- Voltage range for battery percentage calculation
- Whether `B7` accepts the LightMode enum name strings or only integer values
- The meaning of `AX`, `BQ`, and `BS` fields
- Whether the board sends a full telemetry packet on every notify or only changed fields

If you can capture BLE traffic from a working original app session, please contribute the packet dumps.

---

*Extracted via static DEX analysis — 2025*
*Contributions and corrections welcome via GitHub issues*
