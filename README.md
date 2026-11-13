# MatrixChain Power Board

Custom power board that feeds **four 32×32 HUB75 LED panels (4096 LEDs)** plus the **Adafruit
MatrixPortal S3** from one USB-C PD source (requested contract **20V/3A = 60W**, standard 3A cable).

The MatrixPortal S3's MCU can drive up to 4096 LEDs, but its own USB-C port only supplies 5V/3A —
about 1500 LEDs at full white. This board lets the Portal run at full capacity: the Portal drives the
HUB75 data lines, the board supplies the panel power externally.

- **[DATASHEET.md](hardware/DATASHEET.md)** — the board's user-facing spec: electrical characteristics, connector
  pinouts, LED meanings, firmware interface, key components, assembly/bring-up.
- **[DESIGN.md](hardware/DESIGN.md)** — full design rationale, net list, per-pin hookup tables, component-selection rationale, layout and
  safety notes, open items. Source of truth for the schematic.

## Connections

```
USB-C adapter ──► PowerBoard USB-C connector (J1) ──► D1 red LED on /POW_EN (input invalid); D2/D3 green per channel
					└─ I2C connector (J2) ─► Adafruit MatrixPortal I2C connector
					└─ Screw terminals (J3) ─► Adafruit MatrixPortal power terminals (≤1A)
					└─ Screw terminals (J4) ─► HUB75 panels 1-2 (<5A)
					└─ Screw terminals (J5) ─► HUB75 panels 3-4 (<5A)
          
```

Portal links: **J3** (5V tap, ≤1A, to the M3 power screw terminals) and **J2** STEMMA QT
(GND, +3.3V, SDA, SCL). The HUB75 link to the Portal carries **data + GND only, no power** — do not
connect its 5V pins. There is no CH_EN/PG line: channels are gated purely by the supervisor
(input ≥ ≈8V ⇒ `/POW_EN` high). Panel power plugs are not reverse-protected — observe polarity.

## Behavior

| Condition | Input | D1 (red) | D2/D3 (green) | Channels | Firmware |
|---|---|---|---|---|---|
| Good PD source | ≈20V | off | on | on | read HUSB238 contract, cap power at 85% of V×I, run governor on INA226s |
| Lower contract (e.g. 15V/3A) | ≈15V | off | on | on | cap to 85% of V×I, log contract |
| Overcurrent | ≈20V | off | off (faulted ch) | limit/hiccup | INA226 alert (register) + governor |
| Dumb 5V source | ≈5V | **on** | off | off | Portal may still boot; report missing contract |
| Unplugged | 0V | off | off | off | — |

## Firmware interface

I2C on the QT bus (3.3V): HUSB238 **0x08** (source caps / negotiated contract / PDO select),
INA226 **0x40** (ch1) and **0x44** (ch2) — program CAL (0x05) at boot, alert ≈4.8A via Mask/Enable 0x06.
Boot: blank HUB75 ASAP (channels auto-enable at input ≥ ≈8V) → read contract → set power cap →
program INA226 → run (read sensors every 100ms; brightness down if a channel >4.6A for 3 reads).
Details and register maps: DATASHEET.md; rationale: DESIGN.md §5.

## Safety

Panels are **5V only**. 20V@3A and 5V@8.4A are high-energy: protection is the HUSB238 OVP (24V) + Q1
disconnect, per-rail PTC fuses (F1 tap, F2/F3 channels), converter current limits — never rely on
software for hard limits. No input fuse (compliant PD sources only / current-limited bench supply).
Strain-relieve all cables. Full notes: DESIGN.md §9.
