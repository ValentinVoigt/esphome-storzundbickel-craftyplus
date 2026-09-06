# AGENTS.md — CRAFTY+ BLE / ESPHome project

## Purpose

This repository is a reverse-engineered ESPHome integration for a **STORZ & BICKEL CRAFTY+** device over Bluetooth Low Energy (BLE). It maintains a BLE client connection to one CRAFTY+ so Home Assistant can read device state and control supported settings.

The implementation has been tested on the target device. Preserve working behavior and keep ESPHome as the integration layer.

## Source of truth

When information conflicts, use this order:

1. Observed behavior on the user's physical CRAFTY+.
2. The supplied STORZ & BICKEL web-app JavaScript (`crafty.js` and relevant helpers from `main.js`).
3. The current working ESPHome YAML.
4. ESPHome documentation and implementation details.
5. Inference.

Always label inference. Use only devices and vendor material accessed lawfully or with authorization. Publish independently written implementation and protocol facts only; never copy vendor source or substantial excerpts into the implementation or public documentation.

## Home Assistant device model

CRAFTY+ entities belong to their own ESPHome logical device:

```yaml
esphome:
  devices:
    - id: crafty_device
      name: "${crafty_name}"
```

Every user-facing CRAFTY+ entity must use `device_id: crafty_device`.

## Protocol basics

CRAFTY+ uses three proprietary services:

| Service | UUID |
|---|---|
| Service 1 | `00000001-4c45-4b43-4942-265a524f5453` |
| Service 2 | `00000002-4c45-4b43-4942-265a524f5453` |
| Service 3 | `00000003-4c45-4b43-4942-265a524f5453` |

Unless stated otherwise, numeric values are unsigned 16-bit little-endian. Bounds-check payloads and encode/decode bytes explicitly.

## Known GATT characteristics

### Service 1 — live values and controls

| Characteristic | Meaning | Direction | Encoding / behavior |
|---|---|---|---|
| `00000011-...` | Current temperature | Read + notify | `uint16 LE`, °C × 10 |
| `00000021-...` | Target temperature | Read + write | `uint16 LE`, °C × 10 |
| `00000031-...` | Boost offset | Read + write | `uint16 LE`, °C × 10 |
| `00000041-...` | Battery level | Read + notify | `uint16 LE`, percent |
| `00000051-...` | LED brightness | Read + write | `uint16 LE`, percent |
| `00000061-...` | Automatic shutoff timeout | Read + write | `uint16 LE`, seconds; protected write |
| `00000071-...` | Automatic shutoff remaining | Read + notify | `uint16 LE`, seconds |
| `00000081-...` | Heater ON | Write | two zero bytes |
| `00000091-...` | Heater OFF | Write | two zero bytes |

### Service 2 — identity and firmware

| Characteristic | Meaning | Encoding |
|---|---|---|
| `00000032-...` | Device firmware | UTF-8-ish text |
| `00000052-...` | Serial number | UTF-8 text; first 8 characters |
| `00000072-...` | BLE firmware | three bytes: major, minor, patch |

### Service 3 — status, diagnostics, and usage

| Characteristic | Meaning | Direction | Encoding / behavior |
|---|---|---|---|
| `00000023-...` | Accumulated usage hours | Read | `uint16 LE`, hours |
| `00000063-...` | Battery status register 1 | Read | `uint16 LE` bitfield |
| `00000073-...` | Battery status register 2 | Read | `uint16 LE` bitfield |
| `00000083-...` | System status register | Read | `uint16 LE` bitfield |
| `00000093-...` | Project status register 1 | Read + notify | `uint16 LE` bitfield |
| `000001b3-...` | Security code | Write | `uint16 LE` |
| `000001c3-...` | Project status register 2 | Read + write + notify | `uint16 LE` bitfield |
| `000001d3-...` | Factory reset | Write | `uint8 0` after unlock; intentionally unexposed |
| `000001e3-...` | Accumulated usage minutes | Read | `uint16 LE`, minute component |

Do not write undocumented characteristics.

## Temperature behavior

Target temperature uses 40–210 °C and is written as `round(temp_C * 10)`. After changing it, rewrite boost as the vendor app does. Boost is an offset, not an absolute temperature, and must satisfy:

```text
target_temperature + boost_temperature <= 210 °C
```

The package permits a zero boost so target 210 °C remains valid. Superboost is read-only and adds 15 °C in the vendor UI. Legacy firmware may report a Fahrenheit-derived target above 210; the package intentionally supports only the tested modern Celsius representation.

## Project status register 1 (`00000093`)

| Mask | Meaning |
|---:|---|
| `0x0010` | Active / heater active |
| `0x0020` | Boost mode |
| `0x0040` | Superboost mode |

`register & 0x2008` contributes to the vendor's service-required condition. `register & 0x8000` leads the vendor analysis UI to recommend a physical-button factory reset. Do not assign unproven meanings to individual bits in these masks.

Control the heater only with the dedicated Service 1 ON/OFF characteristics, never by writing this register.

## Project status register 2 (`000001c3`)

| Mask | Meaning when set | Home Assistant semantic |
|---:|---|---|
| `0x0001` | Vibration disabled | `Vibration` is inverted |
| `0x0002` | Charge LED disabled | `Charge LED` is inverted |
| `0x0004` | Target temperature reached | Read-only |
| `0x0008` | Find-device active | Read-only state plus command |
| `0x1000` | Automatic BLE shutdown disabled | `Permanent Bluetooth` uses the same polarity |

Writes must read-modify-write the full register and preserve unrelated and unknown bits. The current implementation uses the most recent sensor state, so nearly simultaneous controls can race. Find-device reportedly clears itself after about 30 seconds.

## Battery status register 2 (`00000073`)

| Mask | Meaning when set | Evidence |
|---:|---|---|
| `0x2000` | Charger connected | Inference from two observed unplug/replug cycles |
| `0x8000` | Charger or cable problem | Vendor device-analysis logic |

The `0x2000` interpretation is not present in the vendor app and is therefore tentative. On the tested physical CRAFTY+, it tracked two unplug/replug cycles exactly, but it has not been verified at full charge. It may indicate active charging rather than physical charger presence in that state. Keep the Home Assistant entity named `Charger Connected` unless further physical-device testing disproves this interpretation.

## Protected writes

Automatic shutoff requires security code 815 (`0x032F`, bytes `0x2F 0x03`) immediately before writing timeout seconds. The known maximum is 300 seconds; the package's 30-second minimum is an inference.

Factory reset is intentionally not exposed. Do not add it or reuse the automatic-shutoff unlock code for it without verified need and physical-device testing.

## Diagnostics

The vendor app supplies these composite conditions:

```text
service_required = (battery_status_1 & 0x0600)
                || (system_status & 0x0200)
                || (project_status_1 & 0x2008)
cooling_required = battery_status_1 & 0x4100
charge_required = battery_status_1 & 0x0003
charger_or_cable_problem = battery_status_2 & 0x8000
```

Do not name individual bits inside composite masks without evidence.

## Usage and compatibility

Accumulated usage is `hours + minutes / 60.0`; the minutes register is interpreted as the component accompanying hours, matching the vendor display.

The package targets a tested modern CRAFTY+. The vendor app treats approximately `minor < 51 && major <= 2` as old firmware and disables several features. Do not claim old-firmware support without evidence.

## Implementation constraints

Keep raw writable values, usage components, registers, and packed BLE firmware internal. Factory reset remains unexposed.

Use notifications for current temperature, battery, and both project status registers. Poll automatic-shutoff remaining; enabling its notifications causes repeated updates about every 2.5 seconds, including zero while inactive.

Always preserve unknown register bits, the target-plus-boost invariant, the protected-write sequence, explicit little-endian handling, payload bounds checks, and logical-device assignment. Never commit credentials, real MAC addresses or serial numbers, or unsanitized logs and BLE captures containing device identifiers.
