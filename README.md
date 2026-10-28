# NIU BMS over WebUSB

Read a NIU battery BMS (N1S protocol, RS485 9600 8E1) from Chrome on Android with a
CP2102 USB-RS485 dongle and a USB OTG adapter. No app to install.

**Open:** https://niu-tools.github.io/niu-bms-webusb/

- Live pack voltage, current, SOC, temperatures, cycles and all 17 cell voltages
- Cell balance rating (Δ < 50 mV good, 50–100 mV medium, > 100 mV bad)
- Battery selector with aliases; live view or the last stored reading per battery
- **BMS lockout detection**: when the battery's button LED flashes and the scooter refuses it,
  shows the fault code, when the BMS logged it, the likely cause (high cell after charge, low
  voltage, temperature) and what to do
- Health summary with suggested actions (balance, weak cell, voltage limits, temperature, sensing, wear)
- Statistics per battery and time range: first/last reading, span, counts, read success,
  pack/Δ/current/temperature averages and ranges, weakest cell, per-cell avg/min/max
- Voltage charts and CSV export
- All readings are kept on the phone (IndexedDB, persistent storage requested) until you
  clear them; nothing is uploaded anywhere

## What you need

| Part | Notes |
|---|---|
| USB-RS485 dongle, **CP2102 chip** | Must be CP2102/CP210x: the page talks to that chip directly. CH340 and FTDI dongles will **not** work. |
| USB-A (female) → USB-C (male) OTG adapter | Use a micro-USB OTG adapter instead if your phone has micro-USB. |
| 3 jumper wires | To go from the dongle's screw terminals to the battery's data connector. |
| Android phone with Chrome | WebUSB is not available on iPhone (any browser) or in Firefox. |

## Wiring

```
 ┌──────────────┐     ┌─────────────┐     ┌──────────────────────┐
 │ Android      │     │ OTG adapter │     │ USB-RS485 dongle     │
 │ phone        │     │ USB-C (m) → │     │ (CP2102)             │
 │ Chrome       ├─────┤ USB-A (f)   ├─────┤ USB-A plug           │
 │  USB-C port  │     └─────────────┘     │                      │
 └──────────────┘                         │  screw terminals     │
                                          │  ┌───┬───┬─────┐     │
                                          │  │ A │ B │ GND │     │
                                          └──┴─┬─┴─┬─┴──┬──┴─────┘
                                               │   │    │
                        data wire "A" / "485+" │   │    │ battery negative / GND
                                               │   │    │
                        data wire "B" / "485-" │   │    │
                                               │   │    │
 ┌─────────────────────────────────────────────┴───┴────┴───────────┐
 │ NIU N1S battery — data connector                                 │
 │                                                                  │
 │  A, B and GND pins (battery-side pinout to be added)             │
 └──────────────────────────────────────────────────────────────────┘
```

| Dongle terminal | Battery side |
|---|---|
| **A** (sometimes `D+` or `485+`) | RS485 **A** |
| **B** (sometimes `D-` or `485-`) | RS485 **B** |
| **GND** | battery **GND** (pack negative) |

Tips:

- **Never connect the 60 V pack + to anything on the dongle.** Only A, B and GND.
- No reply? Swap A and B first: it can't damage anything and is the most common mistake.
- Don't skip GND: without a common ground the readings are unreliable or missing.

## Use

1. Wire the dongle to the battery, then plug it into the phone through the OTG adapter.
2. Open the page in Chrome and tap **Connect**. Pick the CP2102 device in the popup.
3. Readings start every 3 s and are saved on the phone. Give the battery an alias with
   **Rename** so you can tell packs apart later.
4. Adding the page to the home screen makes Chrome less likely to evict the stored data.

## Lockout: flashing button LED, scooter won't accept the battery

The BMS can lock the battery out. When it does, the button on the pack flashes instead of
showing charge on its 5 LEDs, and the scooter won't recognise the battery. The page shows
this as **Locked out by the BMS**, with the status code from the BMS.

Observed on a real pack (N1S, 17S, 82 cycles):

- After a full charge (charger LED green), cell #12 sat at **4.161 V** with the others
  around 4.137 V (+24 mV). The BMS reported status **`0x04`** and the LED flashed.
- About 300 mWh was bled from cell #12 alone, bringing it to **4.154 V**. The status went back
  to `0x00`, the fault record was erased, and the LED showed the normal charge level.

So `0x04` is a high-cell condition at the end of a charge. The exact trip and release
voltages aren't known yet. Lockouts at low voltage (a deeply discharged cell) are likely as
well, but haven't been captured yet. The page's advice for each case:

| Likely cause | What the readings show | What to do |
|---|---|---|
| High cell after charge | one cell ≥ 4.15 V and ≥ 10 mV above average | bleed that cell group slightly (≈ 50–300 mWh), reconnect, check status is OK |
| Low voltage | a cell < 3.3 V, or ≥ 50 mV below the rest near empty | charge; if the charger won't start, top up that cell alone at low current to ~3.3 V first |
| Temperature | pack above 50 °C or below 0 °C | bring the pack to room temperature |
| Other code | nothing above fits | note the code, try a charger plug-in, the button and a key-on, read again |

If you capture a different status code, please open an issue with the code and the
readings (Export CSV): it is the only way to map the remaining codes.

## Protocol

NIU did not publish this protocol: everything below was worked out from real NIU N1S packs
(17S, 60 V). The framing is **DL/T 645-1997**, the Chinese electricity-meter protocol. Every
captured frame's checksum matches that standard.

**Serial:** RS485, 9600 baud, 8 data bits, **even** parity, 1 stop bit (8E1). Half duplex:
the phone asks, the BMS answers.

### It's a memory map

The BMS doesn't have commands. It exposes **256 bytes of memory**, and every request reads
a slice of it: **"give me LEN bytes starting at ADDR"**. Every address from 0x00 to 0xFF
answers.

```
request   FE FE FE FE | 68 | 31 CE | 68 | 02 | 02 | ADDR LEN | CS | 16
response              | 68 | 31 CE | 68 | 82 | N  | data…    | CS | 16
                        │    │       │    │    │    │          │    └ end
                        │    │       │    │    │    │          └ checksum
                        │    │       │    │    │    └ masked address + length / N data bytes
                        │    │       │    │    └ length of what follows (2 in a request)
                        │    │       │    └ control: 02 = read, 82 = read reply
                        │    │       └ start again
                        │    └ device address
                        └ start
```

- `FE FE FE FE`: wake-up preamble, sent before requests only.
- **Checksum:** the sum of every byte from the first `68` up to the end of the data,
  modulo 256.
- **Masking:** every data byte, in both directions (including ADDR and LEN), has `0x33`
  added, modulo 256. Subtract `0x33` from **each byte separately**. Subtracting `0x3333`
  from a 16-bit word looks equivalent but is wrong: a borrow from the high byte shifts some
  values by 256 (e.g. a 4.092 V cell reads as 3.836 V).
- Multi-byte numbers are big-endian after unmasking.

Example: read 15 bytes at 0x2D (the pack block) → ADDR `2D+33 = 60`, LEN `0F+33 = 42`:

```
→ FE FE FE FE 68 31 CE 68 02 02 60 42 75 16
← 68 31 CE 68 82 0F 35 EF 33 33 33 33 97 33 33 33 45 47 46 46 51 E9 16
                    └─┬─┘ └─┬─┘       └ 97-33 = 64 = 100 %   └────┬────┘
                      │     current 0 A                temps 20 19 19 30 °C
                      35-33=02, EF-33=BC → 0x02BC = 700 → 70.0 V
```

The 7 reads this project started from (earlier treated as "commands") are just fixed slices:

| Name | ADDR LEN (unmasked) | Wire bytes (masked + CS) |
|---|---|---|
| hwsw | `02 02` | `35 35 3D` |
| serial | `09 10` | `3C 43 52` |
| extra | `01 2C` | `34 5F 66` |
| cycles | `29 13` | `5C 46 75` |
| pack | `2D 0F` | `60 42 75` |
| cells_alt | `2D 37` (pack + cells) | `60 6A 9D` |
| cells | `3C 28` | `6F 5B 9D` |

### Memory map (unmasked)

| Addr | Size | Meaning | Example |
|---|---|---|---|
| 0x00 | 1 | ? (always `01`) | |
| 0x01 | 1 | ? (`31`) | |
| 0x02 | 1 | firmware version, major/minor nibbles | `20` → 2.0 |
| 0x03 | 1 | hardware version | `11` → 1.1 |
| 0x04–0x08 | 5 | ? (`55 55 55 55 0a`) | |
| 0x09–0x18 | 16 | serial number, ASCII | `BNA9PY2B64900352` |
| 0x19–0x28 | 16 | ? differs per pack, static | |
| 0x29 | u16 | probably design capacity, mAh | `65 90` → 26000 |
| 0x2B | u16 | charge cycles | `00 52` → 82 |
| 0x2D | u16 | pack voltage × 0.1 V | `02 BF` → 70.3 V |
| 0x2F | s16 | current × 0.1 A, negative while discharging | |
| 0x31–0x32 | 2 | ? | |
| 0x33 | u8 | SOC % (or 0x31; not settled) | `64` → 100 % |
| **0x35** | u8 | **status: 0 = OK, non-zero = lockout** | `04` = high cell after charge |
| 0x36–0x37 | 2 | ? / extra temperature | |
| 0x38–0x3B | 4 × s8 | temperatures °C | `13 14 13 17` → 19 20 19 23 |
| 0x3C–0x5D | 17 × u16 | cell voltages, mV | `10 42` → 4.162 V |
| 0x5E–0x63 | 6 | 3 unused cell slots (`00`) | |
| 0x64–0x9F | | 10-byte records, values change between reads (counters?), `ff ff ff ff` in empty slots | |
| 0xA0–0xBF | | two 12-byte IDs (MCU unique ID?). The flag byte at 0xB0 also gets bit `04` during a lockout | |
| **0xD3–0xDF** | 13 | **active fault record**: code, `00`, then `ss mm hh dd mo yy` on the BMS clock | `04 00 13 13 0e 1c 0a 1a` → code 4 at 2026-10-28 14:19:19 |
| **0xE1–0xEE** | 14 | second fault record, same timestamp format (`08 04 08 04 02 00` + time) | |
| 0xFF | 1 | flags; bit `04` set during a lockout | `44` locked, `40` OK |

The fault records are zeroed as soon as the fault clears: there is no stored fault history.
The BMS clock in the records did not match real time on the pack tested.

Dump your own pack's map with the `niu-cells` script in the main project (`--dump`), or
compare a map from a healthy and a locked-out state to find more fields.

### Open questions

- SOC vs time remaining: whether 0x31 or 0x33 holds the SOC.
- The `0x04` trip and release voltages, and what the other status bits mean.
- The meaning of 0x19–0x28, the records at 0x64–0x9F, and the second fault record.

### Can it write settings?

Not with anything known so far. In DL/T 645-1997, control code `04` means *write data*
(reply `84`, or `C4` on error), and the standard protects writes with a password. Nobody
has published NIU's write format or password. This page only ever sends reads (`02`).

Guessing writes on a live 60 V pack is a bad idea: one wrong byte in the map could change a
protection limit (overcharge, over-discharge, over-current, temperature) without any visible
sign. There's also no known command to turn balancing on: it appears to be automatic, at the
top of a full charge.
