# MatrixChain Power Board

Custom power board that feeds **four 32×32 HUB75 LED panels (4096 LEDs)** plus the **Adafruit
MatrixPortal S3** from one USB-C PD source (requested contract **20V/3A = 60W**, standard 3A cable).

The MatrixPortal S3's MCU can drive up to 4096 LEDs, but its own USB-C port only supplies 5V/3A —
about 1500 LEDs at full white. This board lets the Portal run at full capacity: the Portal drives the
HUB75 data lines, the board supplies the panel power externally.

- **[DATASHEET.md](DATASHEET.md)** — the board's user-facing spec: electrical characteristics, connector
  pinouts, LED meanings, firmware interface, key components, assembly/bring-up.
- **[DESIGN.md](DESIGN.md)** — full design rationale, net list, per-pin hookup tables, BOM, layout and
  safety notes, open items. Source of truth for the schematic.

## Architecture (as built, Rev A)

```
USB-C J1 ──► VBUS ──► Q1 AOSP21357 PMOS (HUSB238 GATE-driven OVP switch) ──► VDC (switched 20V rail, C2 100µF)
   │  └─ R1 10Ω ─► U1 HUSB238 VIN + C1 (straps: VSET open = 20V, ISET R4 22.6k = 3A, I2C 0x08)
   │
VDC ─┬─► U2 AP63205 ──► VCC ──► U5 LM66100 ──► +5V ──► F1 ──► J3 → Portal M3 power terminals (≤1A)
     ├─► U3 LMR51450SDRRR CH1: SW ── L2 ──► +5VA ──► R_SH1 ──► /5V_CH1 ──► C19 ──► F2 ──► J4 (panels 1–2)
     ├─► U4 LMR51450SDRRR CH2: SW ── L3 ──► +5VD ──► R_SH2 ──► /5V_CH2 ──► C20 ──► F3 ──► J5 (panels 3–4)
     │            both EN = /POW_EN; U7/U8 INA226 (0x40/0x44) Kelvin-sense across R_SH1/R_SH2, on QT I2C
     └─► R13/R14 divider ──► U6 LM397 (IN+ vs 0.4V ref from R8/R9) ──► OUT + R17 pull-up = /POW_EN
            D1 red LED on /POW_EN (input invalid); D2/D3 green per channel
```

Portal links: **J3** (5V tap, ≤1A, to the M3 power screw terminals) and **J2** STEMMA QT
(GND, +3.3V, SDA, SCL). The HUB75 link to the Portal carries **data + GND only, no power** — do not
connect its 5V pins. There is no CH_EN/PG line: channels are gated purely by the supervisor
(input ≥ ≈8V ⇒ `/POW_EN` high). Panel power plugs are not reverse-protected — observe polarity.

## Behavior in one table

| Condition | Input | D1 (red) | D2/D3 (green) | Channels | Firmware |
|---|---|---|---|---|---|
| Good PD source | ≈20V | off | on | on | read HUSB238 contract, cap power at 85% of V×I, run governor on INA226s |
| Lower contract (e.g. 15V/3A) | ≈15V | off | on | on | cap to 85% of V×I, log contract |
| Overcurrent | ≈20V | off | off (faulted ch) | limit/hiccup | INA226 alert (register) + governor |
| Dumb 5V source | ≈5V | **on** | off | off | Portal may still boot; report missing contract |
| Unplugged | 0V | off | off | off | — |

## Firmware interface (summary)

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

## Files

- `powerboard.kicad_pro` / `.kicad_sch` / `.kicad_pcb` — KiCad project (schematic + layout)
- `libs/` — project symbol/footprint libraries (`sym-lib-table`, `fp-lib-table` at root)
- `datasheets/` — PDFs for every IC
- `DATASHEET.md` — board spec · `DESIGN.md` — design detail · this file — overview
