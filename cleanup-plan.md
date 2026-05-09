# Sensor Cleanup Plan

Target file: `sensor/src/sensor.cpp`

Findings are based on manual reverse-engineering of the FA14 RS485 message by capturing hex dumps
and comparing before/after button presses. The key insight is that the status bytes are bitfields —
reading them in binary rather than as hex nibble strings makes the structure much cleaner and the
code more robust.

---

## 1. Remove debug MQTT sensors

The following HA sensors exist only for protocol debugging and should be removed entirely:
declarations, `setup()` config, and all `setValue()` calls.

| Variable | HA entity name | Notes |
|---|---|---|
| `rawData` / `lastRaw` | "Raw data" | full FA14 payload as hex string |
| `rawData2` / `lastRaw2` | "Non-Temp" | FA14 messages where byte 5 is not C/F/- |
| `rawData3` / `lastRaw3` | "CMD" | command echo bytes 17-21 |
| `fbData` / `lastFB` | "FB" | raw FB06 and FA14 tail |
| `lastRaw4`–`lastRaw6` | (not published) | AE0D non-idle messages, dead code |

---

## 2. Replace hex-string parsing with direct byte operations

Currently `handleMessage` converts `buf[]` → hex string `result` → `result.substring()` to extract
nibbles. Since `buf[]` is already available, this adds heap allocation on every message and makes
the byte positions harder to verify.

Replace with direct reads from `buf[]`:

```cpp
// Byte 5: temperature unit ('C'=0x43, 'F'=0x46, '-'=0x2D)
uint8_t unit = buf[5];

// Byte 6: pump bitfield
//   bit 0 = P1 low speed
//   bit 1 = P1 high speed
//   bit 2 = P2 low speed
//   bit 3 = P2 high speed
uint8_t p1 = buf[6] & 0x03;        // 0=off, 1=low, 2=high
uint8_t p2 = (buf[6] >> 2) & 0x03;

// Byte 7: heater (bits 5-4) and light (bits 1-0)
//   0x10 = heater on, 0x20 = heater verifying, 0x03 = light on
uint8_t heaterVal = (buf[7] >> 4) & 0x03;  // 0=off, 1=on, 2=verifying
bool lightState   = (buf[7] & 0x03) != 0;

// Byte 8: operating mode (low nibble)
uint8_t mode = buf[8] & 0x0F;

// Byte 9: menu mode (0x00=idle, 0x46=set temp, 0x4C=set mode, 0x5A=standby)
uint8_t menu = buf[9];

// Bytes 14-15: time HH MM (0xFF = unknown)
uint8_t hh = buf[14], mm = buf[15];

// Byte 16: current temperature in °F (0xFF when in menu)
uint8_t rawTempF = buf[16];

// Byte 17: command echo (0x01=temp up, 0x02=temp down, 0x00=idle)
uint8_t cmdEcho = buf[17];
```

The `result` hex string and `HexString2ASCIIString()` can be kept for telnet debug output, but
removed from all parsing logic.

---

## 3. Simplify pump state logic

The current 10-branch `if/else if` block on hex nibble strings collapses to 4 lines using the
2-bit extraction above:

```cpp
pump1State = p1;  // 0=off, 1=low, 2=high — maps directly to HASelect index
pump2State = p2;

if (p1 == 1) tubpowerCalc += POWER_PUMP1_LOW;
if (p1 == 2) tubpowerCalc += POWER_PUMP1_HIGH;
if (p2 == 1) tubpowerCalc += POWER_PUMP2_LOW;
if (p2 == 2) tubpowerCalc += POWER_PUMP2_HIGH;
```

This also makes the `PUMP1_STATE_HIGH` / `PUMP2_STATE_HIGH` macros unnecessary — `2` always means
high speed naturally from the 2-bit field — but leave the macros in `constants.h` for now since
they are used by the library version.

---

## 4. Simplify mode / menu parsing

Replace `result.substring(17,18)` string comparisons with direct byte compares using `switch`:

```cpp
switch (mode) {
    case 0x1: state = "Standard";             tubMode.setState(MODE_IDX_STD); break;
    case 0x2: state = "Economy";              tubMode.setState(MODE_IDX_ECO); break;
    case 0x4: state = "Sleep";                tubMode.setState(MODE_IDX_SLP); break;
    case 0x9: state = "Circulation";          tubMode.setState(MODE_IDX_STD); break; // TODO: confirm index
    case 0xA: state = "Cleaning";             break; // mode unknown during cleaning
    case 0xB:
    case 0x3: state = "Std in Eco";           tubMode.setState(MODE_IDX_STD); break;
    case 0xC: state = "Circulation in sleep"; tubMode.setState(MODE_IDX_SLP); break;
    default:  state = "Unknown mode";         break;
}

switch (menu) {
    case 0x00:                   break; // idle — keep mode state set above
    case 0x46: state = "Set Temperature"; break;
    case 0x4C: state = "Set Mode";        break;
    case 0x5A: state = "Standby";         break;
    default:   state = "Menu";            break;
}
```

---

## 5. Simplify temperature parsing

Replace `HexString2ASCIIString(result.substring(4, 10))` with direct ASCII-to-int from `buf[]`.
`buf[2]`–`buf[4]` are ASCII digit characters; `buf[5]` is `'C'` or `'F'`.

```cpp
char digits[4] = { (char)buf[2], (char)buf[3], (char)buf[4], '\0' };
double tmp = atoi(digits) / 10.0;
```

---

## 6. Promote heater to 3-state int

`heaterState` is currently a `boolean`, losing the distinction between *on* and *verifying*.
The verifying state is already handled specially in the power calc (counted as on) but the HA
heater binary sensor could benefit from knowing the difference in the future.

Change to:

```cpp
int heaterState = 0; // 0=off, 1=on, 2=verifying
```

Update the one place that checks it as a bool (`heater.setState(heaterState)`) — `HABinarySensor`
accepts non-zero as true so the HA entity behaviour is unchanged.

---

## 7. Document the FB06 command byte structure in `constants.h`

Add a comment block above the `#define` command list explaining the 9-byte structure, so the
magic hex strings can be verified against the protocol without cross-referencing notes.

```
FB06 command frame (9 bytes):
  [0]    0xFB  start byte
  [1]    0x06  length
  [2]    0x03 or 0x43  destination / device class
  [3]    0x45 or 0x43  device type (0x45=panel, 0x43=jets)
  [4]    0x0E or 0x06  command category
  [5]    0x00  always zero
  [6]    cmd   command code (see below)
  [7]    0xFF ^ cmd  one's complement of cmd
  [8]    checksum (algorithm not yet confirmed)
```

---

## Out of scope

- `sendCommand()` — timing logic is correct, leave it
- `setOption()` / `onTargetTemperatureCommand()` — correct, leave them
- `aux` HASelect — byte 6 high nibble meaning not yet confirmed, leave as-is
- `balboaGL.cpp` library — separate codebase, not touched by this plan
