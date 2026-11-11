# Todimart T3 — unlocking the 25 km/h speed cap by under-declaring the wheel diameter

**Scope:** Todimart **T3** electric scooter (sold under various names, e.g. `YY-XMPRO`), fitted with a
Todimart / Vicont `039ZQ-34` controller kit and the Vicont Android app (`com.vicontout.benben`).

**Result:** the scooter goes from **25 km/h to ~31 km/h real speed** without touching the firmware,
by changing one EEPROM parameter (wheel diameter). The speedometer keeps reading 25.

> ⚠️ This is not a "speed limit setting". The limit stays at 25 and is never bypassed. What changes
> is the controller's *measurement* of speed: it is told the wheel is smaller than it is, so it lets
> the motor spin faster while still believing it is at 25 km/h.

---

## ⚠️ Warnings — read before doing anything

- This **deliberately falsifies the scooter's instrumentation**. After the change:
  - the **speedometer lies** (shows 25 while you ride at ~31),
  - the **odometer and remaining-range estimate read ~19 % low**.
- A modified scooter is **no longer compliant with its type approval** (25 km/h, L1e-B / EPAC-class
  rules depending on country). Insurance, warranty and liability in case of an accident are affected.
- Braking distance, handling and battery/motor stress are no longer covered by the manufacturer
  (36 V, 350 W, 15 A).
- Every write described here is **reversible** and happens in EEPROM. Nothing is flashed, and no
  firmware byte is modified.
- The numbers below are for **one specific model**. On another unit, measure first (§4) instead of
  copying values.

---

## 1. Hardware and software identified

| Item | Value |
|---|---|
| Commercial model | **Todimart T3** |
| Cloud brand | Todimart (`todimart.com`) |
| Controller kit | `039ZQ-34` |
| Free-form name in cloud | `YY-XMPRO` (M365 Pro clone) |
| Motor | `YY-039ZQ34-D36V6G17A-S-6528` |
| Battery / power | 36 V, 350 W, 15 A max |
| Dashboard (AD) | hardware **104**, firmware **58**, board **0528** |
| Motor controller (AE) | hardware **10**, firmware **50**, board **0000** |
| BLE | name `vA-<MAC>`, service `FEE0`, write/notify `FEE2` |
| Stock app | Vicont `com.vicontout.benben` v2.0.4 |

---

## 2. Why the app's speed setting does nothing

The Vicont app sends command `0x3C` ("maximum speed", 1..99, where `99` = *不限速* "no limit"). The
frame is well formed — verified in the app's own BLE log:

```
FA AF A5 5A 3C 01 63 FA      →  code 0x3C, value 0x63 = 99, checksum valid
```

The controller **silently clamps** it and replies with the applied value *and* its ceiling:

```
5A 3C 02 <value> <ceiling> <checksum> <status>
         ↑ applied  ↑ firmware ceiling
```

The app treats `value == 0` as success; receiving `25`, it shows « 设置失败 » and **reverts to the
old value** — exactly the blinking behaviour users report. The app is not the culprit: the firmware
refuses.

**Direct EEPROM write confirms it.** Writing P10 = 20 makes the scooter genuinely top out at 20 km/h.
Writing P10 = 30 or 35 brings it back to 25. The 25 km/h ceiling is a hard-coded firmware value, not
a configuration mistake.

No firmware is published for this model either: `GET /api/upgrade/last/info` returns
`升级类型信息不存在` for both types (`AD` dashboard, `AE` controller), and no image is embedded in the
APK.

---

## 3. The bypass: under-declaring the wheel diameter

The controller derives both the displayed speed **and** the limiting point from the motor rpm and a
**wheel diameter stored in EEPROM**:

```
speed_kmh = rpm × π × D_metres × 60 / 1000
```

Declare a **smaller** wheel than reality, and for the same rpm the controller computes a lower speed.
It therefore allows a higher rpm before reaching its 25 km/h limit — while the wheel really spins
faster.

The parameter is **P14** ("Wheel diameter"), expressed in **tenths of an inch**:

```
D_metres = P14 × 0.00254        (P14 = 83  →  8.3 inches  →  210.8 mm)
```

Adjustment formula:

```
P14_new = P14_stock × (indicated_speed / desired_real_speed)
```

To go from 25 to 31 km/h: `83 × 25/31 ≈ 67`.

---

## 4. Measuring the declared diameter without reading it

EEPROM parameters **cannot be read back** on this firmware: the `0xC9` "show parameter on the
dashboard" command is not implemented, and there is no read frame at all. The reliable method is to
**derive** the value from a physical measurement: motor rpm and indicated speed are both broadcast in
the `0x10` telemetry frame.

Measurement procedure:

1. Rear wheel **off the ground** (the clamp applies unloaded too, which makes the measurement
   reproducible).
2. Full throttle, **stabilised rpm** for 15–20 s.
3. Read **rpm** and **indicated speed** at the same time.

Rpm is the honest quantity: unlike the speedometer, it cannot be clamped. Then:

```
D_metres = indicated_kmh × 1000 / (60 × π × rpm)
P14      = D_metres / 0.00254
```

Real example (stock values): 24.9 km/h at 625 rpm → D = 211.4 mm → **P14 = 83**.

---

## 5. Full procedure

### Software required

The stock app **cannot** write P parameters. You need a client that exposes command `0xD2` (EEPROM
parameter write). Used here:

- **[AovoControl](https://github.com/YouRooni/AovoControl)** (open source, Android, ViCont support)
- enable **Settings → Expert mode**, then open **Settings → Engineering menu**
- add the **Motor speed** tile to the dashboard (Edit dashboard → Add tile → Motor speed); it is the
  measurement instrument

### Steps

1. **Record the stock values** before writing anything:
   - Engineering menu → **P14** (it displays `10` by default: the app reads nothing, that is a filler
     value, not the real one),
   - **P10** (Sport limit) — readable, this one comes from the `0x3C` reply,
   - **P15** (speedometer source),
   - starting torque: *Controller parameters* → *Throttle response* (1..10 scale).
2. **Measure the real diameter** as in §4 (wheel up, full throttle, rpm + speed).
3. **Compute** `P14_new = P14_stock × 25 / desired_speed`.
4. **Write**: Engineering menu → P14 → slider → *Apply*.
   Frame sent: `FA AF A5 5A D2 02 0D <value> <sum>`
5. **Verify**: repeat the §4 measurement. Rpm must **increase** by the predicted ratio while the
   indicated speed stays at **25**. That is the signature of success.
6. Final check **while riding**, on a closed road, against GPS: real speed must match the calculation.

### Values measured on this unit (reference)

| P14 | Max rpm (wheel up) | Real speed |
|---|---|---|
| **83** (stock) | 625 rpm | **24.9 km/h** |
| 80 | 660 rpm | 26.3 km/h |
| **67** (retained) | 768–781 rpm | **≈31 km/h** |

---

## 6. Torque (optional, independent of speed)

The two torque settings are not subject to the speed clamp. Direct commands `0x3D` (starting torque)
and `0x3E` (max driving torque), raw values **0..200** (200 = 100 %):

```
FA AF A5 5A 3D 01 C8 60     → starting torque = 200/200
FA AF A5 5A 3E 01 C8 61     → driving torque  = 200/200
```

Verifying `0x3D`: *Controller parameters* → *Throttle response* goes from `8` to `10` (1..10 scale
rebuilt from 0..200). `0x3E` has no dedicated readout on that screen.

Effect: sharper acceleration, higher inrush current, more heat and wear.

---

## 7. Returning to stock

| Parameter | Stock value |
|---|---|
| **P14** (diameter) | **83** |
| **0x3D** (starting torque) | **160** (`0xA0`, i.e. 8/10) |
| **P15** (speedometer source) | **0** (documented default) |
| P10 (limit) | 25 (untouched by this procedure) |

Corresponding frames:

```
FA AF A5 5A D2 02 0D 53 <sum>     → P14 = 83
FA AF A5 5A 3D 01 A0 <sum>        → starting torque = 160
FA AF A5 5A D2 02 0E 00 <sum>     → P15 = 0
```

After restoring 83, the speedometer, odometer and limit are consistent again.

---

## 8. Protocol reference (as observed on the device)

### Framing

```
FA AF A5 | zt(0x5A) | command | length | payload… | checksum
```

`checksum = (zt + command + length + Σpayload) & 0xFF`

Replies carry **no** preamble:

```
zt | command | length | payload… | checksum | status
```

### Useful commands

| Code | Direction | Payload |
|---|---|---|
| `0x01` | TX | start telemetry |
| `0x10` | **RX** telemetry | speed `u16/10` km/h, rpm `u16`, voltage `u16/100`, current |
| `0x11` | **RX** state | gear, lights, fault bits |
| `0x12` | **RX** versions | dashboard/controller hw+sw, gear mask |
| `0x3C` | TX | max speed 1..99 (**clamped to 25**) · reply `[value, ceiling]` |
| `0x3D` | TX | starting torque 0..200 |
| `0x3E` | TX | max driving torque / current 0..200 |
| `0x3F` | TX | electromagnetic brake 0..200 |
| `0x42` | TX | gear 1..5 |
| `0xAB` | TX | versions: `AD` dashboard, `AE` controller |
| `0xC9` | TX | show a parameter on the dashboard (**not implemented here**) |
| `0xD2` | TX | **write an EEPROM parameter**: `index, value` |

`index = P_number - 1` (P10 → 9, P14 → 13, P15 → 14).

### Relevant P parameters

| P | Index | Meaning | Range | Unit |
|---|---|---|---|---|
| P8 | 7 | Eco limit | 1..99 | km/h |
| P9 | 8 | Drive limit | 1..99 | km/h |
| **P10** | **9** | **Sport limit** (= `0x3C`) | 1..99 | km/h |
| P11 | 10 | Gear 4 limit | 1..99 | km/h |
| P12 | 11 | Gear 5 limit | 1..99 | km/h |
| P13 | 12 | Motor pole pairs | 1..99 | — |
| **P14** | **13** | **Wheel diameter** | 1..99 | 1/10 inch |
| P15 | 14 | Speedometer source | 0..9 | — |

---

## 9. Dead ends (so nobody repeats them)

| Attempt | Outcome |
|---|---|
| `0x3C` above 25 | clamped to 25 by the firmware |
| `0xD2` P10 above 25 (direct EEPROM write) | clamped to 25 as well |
| Vicont OTA firmware (`AD`/`AE`) | `升级类型信息不存在` — nothing published |
| Firmware inside the APK | no `.bin`; `demo1.8` is only a default filename string |
| Public firmware (GitHub, scooterhacking.org) | nothing for `039ZQ` / board `0528` |
| Xiaomi firmware (M365, Pro, 1S, Max…) | different MCU, different protocol (`55 AA`), incompatible — bricks the controller |
| Scooter Hacking Utility / DownG | speak `55 AA` / `5A A5` on services `FFE0`/`FFE1` → "unknown device" |
| P15 = 6 ("real speed" source) | no effect: the same diameter constant is used everywhere |
| P13 (pole pairs) | same family of effect as P14, not tested on this unit |

---

## 10. Why the speedometer cannot be fixed

Every calculation — indicated speed, odometer, limiting point — derives from the **same** declared
diameter. There is no separate "true" constant. As soon as P14 lies, everything lies consistently.
The trade-off is therefore **structural**, not a missed setting:

- maximum real speed; **or**
- accurate speedometer and odometer.

---

## Credits

- **[YouRooni/AovoControl](https://github.com/YouRooni/AovoControl)** — the open-source client that
  documents the ViCont engineering menu and command `0xD2`, without which this bypass would have been
  far harder to find. The P-parameter descriptions quoted here come from its `EngineeringParams.kt`.
- The `0x3C`/`0x3D`/`0x3E`/`0x3F` command map and the frame layout were confirmed by reverse
  engineering the uni-app bundle of the Vicont app `com.vicontout.benben` v2.0.4.
