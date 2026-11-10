# MatrixChain Power Board: Design Document

Revision A — matches the captured schematic (`powerboard/powerboard.kicad_sch`, board rev "A"),
verified by netlist + ERC. Written 2 Nov 2026; updated 7 Nov 2026.

This file is the source of truth for the design and the connection-level detail of the
schematic. Anything marked **(suggested)** is an implementation detail filled in to close a
gap; anything marked **(verify)** comes from memory and must be checked against the
datasheet before you rely on it. Pin numbers are deliberately omitted: take them from the
manufacturer symbols and datasheets (PDFs are in `powerboard/`).

## 1. What the board does

A custom power board feeds four 32×32 HUB75 LED panels (4096 LEDs) and the Adafruit MatrixPortal S3 from one USB-C Power Delivery source (up to 20V/3A, 60W).

The problem this board solves: the MatrixPortal S3's MCU (ESP32-S3) is powerful enough to drive up to **4096 LEDs (64×64)**, but its own USB-C port can only supply 5V/3A — enough for only about **1500 LEDs at full white** (≈2mA per LED). Four panels at full white draw about 8.4A at 5V (2.1A each, measured), so the Portal alone cannot power them. This board makes the most of the Portal's capacity: the Portal drives the HUB75 data lines, and the board supplies the panel power externally.

- Two independent 5V channels (LMR51450 buck converters, U3/U4) each feed two panels through the panels' dedicated 2-pin power connectors. Panel power never goes through the HUB75 data connector.
- A separate housekeeping buck (U2, AP63205, fixed 5V) powers the Portal through a 1A fused tap, plus the status LED and the supervisor. It must work before PD negotiation completes and on a dumb 5V charger.
- Two INA226 current sensors (U7/U8) with external 2mΩ shunts (R_SH1/R_SH2) report per-channel voltage and current to the Portal over I2C (STEMMA QT).
- A comparator supervisor (U6, LM397 + external 0.4V reference) disables the panel channels when the input is not a valid PD contract. There is no CH_EN line from the Portal — the channels are gated by the supervisor alone.


## 2. Architecture

```
USB-C J1 ──► VBUS ──► Q1 AOSP21357 PMOS (HUSB238 GATE-driven OVP switch) ──► VDC (switched 20V rail)
   │                    gate = U1.GATE ──R3 5.1k──┬── R2 49.9k pull-up to VBUS (Q1 source)
   └─ R1 10Ω ─► U1 HUSB238 VIN + C1 1µF/50V        │   (U1 straps: VSET open = 20V, ISET R4 22.6k = 3A, I2C 0x08)
        CC1/CC2, D+/D− → J1 pins                   │
                                                   ▼
VDC ─┬─► U2 AP63205 ──► VCC (housekeeping 5V) ──► U5 LM66100 ideal diode ──► +5V ──► F1 ──► J3 → Portal M3 power terminals (+5V, GND)
     │      VCC also feeds: C7/C8/C18, U6 VDD, R17 pull-up, D1 anode
     ├─► U3 LMR51450SDRRR CH1 (EN = /POW_EN): SW ── L2 ──► +5VA ──► R_SH1 ──► /5V_CH1 ──► U7 INA226 (Kelvin) ──► C19 220µF ──► F2 ──► J4 (panels 1–2)
     ├─► U4 LMR51450SDRRR CH2 (EN = /POW_EN): SW ── L3 ──► +5VD ──► R_SH2 ──► /5V_CH2 ──► U8 INA226 (Kelvin) ──► C20 220µF ──► F3 ──► J5 (panels 3–4)
     └─► R13 820k / R14 43k divider ──► U6 LM397 IN+ (≈8V threshold on VDC); Vref ≈0.4V from VCC via R8/R9
            U6 OUTA (open-drain) + R17 10k pull-up to VCC = /POW_EN ──► EN of U3, U4
            D1 (anode VCC) + R15 1k ──► /POW_EN : fault LED, lit when input invalid
```

Portal links: `J3` 2-pin 5.08mm screw terminal (`+5V` via F1, `GND`) wired to the MatrixPortal's
**M3 power screw terminals** (+5V/GND — the pair meant to power the matrix), and `J2` STEMMA QT
(GND, +3.3V, /SDA, /SCL). The HUB75 port carries data + GND only, no power; the 3-pin A0 JST
is unused (available if a GPIO link such as PG is ever wanted). There is no CH_EN or PG line.

I2C bus: U1 (HUSB238, 0x08), U7 (INA226, 0x40), U8 (INA226, 0x44) on /SDA, /SCL; pull-ups come
from the Portal's QT port.

Grounding: one star ground on the power board. Panel grounds return through their power
terminals; the Portal's ground ties in through its HUB75 ribbon's GND (data return only, no
panel power). The J1 shell (S1–S4) ties to GND.

## 3. Nets and rails

| Net | Description |
|---|---|
| `VBUS` | USB-C VBUS at J1. Feeds R1→U1 VIN, Q1 source, R2. No input fuse (see §9). |
| `VDC` | Switched 20V rail after Q1. Feeds U2 IN/EN, U3/U4 VIN, R13. Bulk C2 100µF/35V + ceramics C3–C5. |
| `VCC` | Housekeeping 5V from U2 (≈4.8–4.9V on a 5V fallback). |
| `+5V` | After the U5 ideal diode; C17 1µF here, then polyfuse F1 → J3 (Portal tap). |
| `+5VA`, `+5VD` | Channel outputs before the shunts (C11/C12, C15/C16, FB dividers). |
| `/5V_CH1`, `/5V_CH2` | After shunts R_SH1/R_SH2 and U7/U8; C19/C20 220µF bulk here, then polyfuses F2/F3 → J4/J5. |
| `/POW_EN` | U6 OUTA (open-drain) + R17 10k pull-up to VCC; drives U3/U4 EN. High when VDC ≥ ≈8V. |
| `+3.3V` | 3.3V from the Portal over J2 pin 2. Powers U7/U8 VS (C21/C22 0.1µF local). |
| `/SDA`, `/SCL` | I2C on the QT bus (J2, U1, U7, U8). |
| `GND` | Single net; star point near the channel bulk capacitors. |

## 4. Schematic hookup (per sheet)

Reference designators are the ones on the schematic; the BOM is in section 6. Pin names refer
to datasheet function names.

### Sheet 1: Input (USB-C, PD sink, OVP switch, bulk)

**J1 USB-C receptacle, 16-pin (USB4085-GF-A).** VBUS (A4/A9/B4/B9) → `VBUS`; GND pins and
shell (S1–S4) → `GND`; CC1/CC2 and D+/D− (A6/A7) → U1; DP2/DN2/SBU unconnected. The HUSB238
provides the CC termination (Rd) internally — no external Rd needed.

**U1 HUSB238_002DD (DFN-10L).** Use this package and variant: the SOT33-6L has no I2C pins, and
the `_001`/`_003` variants drop to 5V on OTP (90/75°C); `_002DD` moves to the next PDO at 150/130°C.

| U1 pin | Connect to |
|---|---|
| VIN | `VBUS` through R1 10Ω, with C1 1µF 50V to GND at the pin (HUSB238 datasheet typical circuit) |
| CC1, CC2 | J1 CC1, CC2 |
| D+, D− | J1 D+, D− |
| VSET | Open (requests 20V) |
| ISET | R4 22.6kΩ to GND (requests 3A; the pin sources 100µA). **Never leave ISET open:** the default 3.25A request cannot be met by a 20V/3A charger. |
| SDA, SCL | `/SDA`, `/SCL` (I2C pins accept 2.5–5.5V, so the 3.3V bus is fine) |
| GATE | Drives Q1 through R3 5.1kΩ, with R2 49.9kΩ gate–source pull-up (datasheet typical circuit). Q1 opens only on OVP/OTP/hard-reset, so the housekeeping rail also works on a dumb 5V charger. |
| GND / pad | `GND` (the pad is the only ground connection) |

**Q1 AOSP21357 (SO-8, wide 5.23mm).** PMOS between `VBUS` (source) and `VDC` (drain); gate from
U1 GATE via R3, pulled up by R2. VGS rating ±25V covers −20…−24V at 20V/24V VBUS, so no zener
clamp is needed. VDS −30V leaves margin at the 24V OVP trip. **(Open item: the schematic still
has the narrow `SOIC-8_3.9x4.9mm` footprint — change to the AOS `SO8SOP-8L_AOS` footprint.)**

**Input fuse: omitted by decision.** With a compliant PD source the charger's OCP bounds a
short to ≈3A/≤60W, and Q1 gives OVP disconnect. Conditions when omitting: size `VDC` traces and
Q1 for ≈5A (charger OCP headroom), never connect an unlimited supply (always set the bench
supply's current limit if using the suggested `J_BENCH` header), and do the §10 VBUS scope check
before production. If fitted instead: one-shot slow-blow ≈3.5A hold, ≥32V rating, on `VBUS` at
J1 before Q1 and the R1 tap.

**Bulk on `VDC`:** C2 100µF/35V electrolytic/polymer across `VDC`–GND at the buck VIN cluster,
plus C3/C4 4.7µF 50V X7R and C5 10µF. Use ≥35V rating — 25V parts lose much of their capacitance
at 20V bias. This damps 20V hot-plug ringing; the HUSB238's own C1 stays at its VIN behind R1.

**TVS on `VDC`: dropped.** The HUSB238's own OVP trips at 120% of the request (24V for 20V,
50µs debounce) and opens Q1 on OVP/OTP/hard-reset, isolating `VDC`; the bucks are abs-max 38–40V,
so the ≤24V that can pass before the OVP acts is within ratings, and the 100µF bulk damps
hot-plug energy. Keep an SMBJ-class footprint DNP (standoff ≥26V, e.g. SMBJ26A, so it does not
conduct at 24V) and populate only if the §10 scope check shows peaks above ≈28–30V on
`VDC`/`VBUS` (Q1 is −30V VDS). The scope check is mandatory before production since no TVS is
fitted.

**(Suggested)** a DNP 2-pin 5.08mm header `J_BENCH` for a 20V bench supply and a 0Ω link in
series with `VBUS` to isolate the USB-C front end during bring-up.

### Sheet 2: Housekeeping rail and Portal tap

**U2 AP63205WU-7 (TSOT26, fixed 5V, 2A, 3.8–32V in).** Values from the datasheet's recommended
table for 5V out.

| U2 pin | Connect to |
|---|---|
| VIN | `VDC`, with C5 10µF to GND |
| EN | `VDC` (EN rated to 35V, can tie to VIN) |
| SW | L1 4.7µH to `VCC` |
| BST | C6 100nF to SW |
| FB | `VCC` directly (fixed-output part) |
| GND | `GND` |

Output capacitors on `VCC`: C7/C8 22µF + C18 1µF. L1: **4.7µH, Isat > 2.7A, DCR < 100mΩ — literally the
AP63205 datasheet's recommended typical-application value** (DS41326: "for most applications select an
inductor of approximately 1.2µH to 4.7µH"; typical circuit: "4.7µH, DCR < 100mΩ, Isat > 2.7A"). The
AP63205WU-EVM ships a 6.8µH/5A Wurth 74439346068, but that is the EVM BOM's choice for the full 2A eval
load, not a requirement — our housekeeping load is ≤ ≈1.2A, so 4.7µH is right-sized (smaller, cheaper,
plenty of ripple margin). The
AP63205 acts as an LDO near 100% duty, so on a 5V input expect ≈4.8–4.9V at 0.5–1A
(estimate; bench-check it).

**U5 LM66100DCK (SC-70-6) ideal diode.** `VCC` in, `+5V` out. The Portal ties its USB-C 5V and
the M3 power-terminal 5V together directly on its 5V rail, so this stage stops the AP63205 from
back-feeding a USB host and vice versa. CE is tied to VOUT — valid per the datasheet (MOSFET on
while CE < VIN, off when CE > VIN, so it also blocks back-feed). ST is left NC. A plain Schottky
would drop ≈0.4V from a rail that already sits near 4.8V on a 5V fallback.

**F1 polyfuse 2920L150/12MR** (1.5A hold, 12V) between U5 VOUT and J3 — protects the 1.5A
LM66100; the tap is ≤1A. **J3** 2-pin 5.08mm screw terminal (`+5V` via F1, `GND`) → the
MatrixPortal's M3 power screw terminals (+5V, GND). C17 1µF on the tap.

### Sheet 3: Supervisor and fault LED

**U6 LM397 (SOT-23-5).** VDD from `VCC` (5–30V single supply). Single comparator — exactly our use. Same open-collector output as the LM393 (R17 pull-up unchanged); input common-mode includes ground (0 to VCC−1.5V); lower offset than LM393. LM393DR2G (SOIC-8 dual) is the drop-in alternate if the LM397 is not stocked. Divider from `VDC` — R13 820kΩ top, R14 43kΩ bottom — to IN+. The LM397 has **no internal reference**, so Vref ≈ 0.4V comes from a divider on `VCC` (as drawn: R8 115kΩ top, R9 10kΩ bottom → 5V × 10/125 = 0.4V). Rising threshold = 0.4 × (1 + 820/43) ≈ **8.0V** (tracks `VCC` ±4% — fine, 9V+ valid / 5V invalid either way). Output is **open-collector**: R17 10kΩ pulls it up to `VCC` → `/POW_EN` high when VDC ≥ ≈8V, driving U3/U4 EN directly. Add a few mV of hysteresis with R16 ≈ 10MΩ from output back to IN+ (anti-chatter on the 20V rail). Inputs are fine (tap ≈1V, ref 0.4V, both < VCC−1.5V). Note: LM397 chosen over the TPS3700 for JLCPCB stock/price (LM393DR2G is the drop-in alternate).

*Why a supervisor and not an op-amp:* an op-amp used as a comparator would need an external voltage reference (the 400mV threshold would otherwise be ratiometric to `VCC` and drift between 4.8–5V), added hysteresis (no chatter on the 20V rail), and its push-pull output complicates the `/POW_EN` pull-up + fault-LED arrangement. The TPS3700 has all of that built in — internal 400mV ±1% reference, falling hysteresis, open-drain output, inputs tolerant to 6.5V regardless of VDD (so the divider hangs directly on `VDC`) — in one SOT-23-6 covering both channels, at ~$0.30. The only true simplification would be using the LMR51450's own EN pin as a programmable UVLO (divider from `VDC` to EN), which would delete U6/R13/R14 — but you'd lose the precise threshold, the hysteresis and the fault-LED/PG indication, and each channel would need its own divider.

**D1 fault LED:** anode to `VCC`, cathode through R15 1kΩ to `/POW_EN` — lit whenever the input
is invalid.

**D2/D3 channel "power available" LEDs:** green LED from `+5VA` / `+5VD` through R18/R19 to GND
(as drawn: R18/R19 = 10kΩ → ≈0.3mA, dim but visible; use 2.2–4.7kΩ for ≈0.7–1.3mA if you want
brighter). Status scheme: D1 on = input not a valid PD contract; D2/D3 on = that channel's 5V is
up; D1 off but D2 or D3 off = that channel faulted (tripped F2/F3 or buck hiccup). If you'd rather
the LED also catch a blown polyfuse, put it on the terminal side of F2/F3 instead. U7/U8 ALERT pins are NC, so overcurrent alerts are register-only (firmware must
poll the Mask/Enable flag); if a local amber indication is wanted later: wire both ALERT pins
together, pull up to `+3.3V`, LED + 470Ω to `+3.3V`.

### Sheet 4: Panel channels (draw once, instantiate twice)

**U3 / U4 LMR51450SDRRR (WSON-12, 3×3).** PFM variant with integrated spread spectrum — chosen
for EMI (bucks next to HUB75 panels) and light-load efficiency; drop-in same package/pinout as
FNDRRR. Component values from the datasheet's Table 9-2 for 5V out at 500kHz.

| Function | Connect to |
|---|---|
| VIN | `VDC`, bulk + ceramics at the pins (§4 sheet 1) |
| EN | `/POW_EN` (R17 pull-up to VCC); supervisor-driven only |
| SW | L2/L3 4.7µH (6A RMS, 10A sat, above the 9.6A max peak limit) to `+5VA`/`+5VD` |
| BOOT | C9/C13 100nF to SW |
| FB | Divider from `+5VA`/`+5VD`: R5/R11 100kΩ top, R6/R10 19.1kΩ bottom → ≈4.99V; C10/C14 33pF in series with R7/R12 1kΩ across the top resistor |
| RT | Open = 500kHz default on the commercial part (RT shorted = 1MHz) |
| PG | Optional; NC |
| PGND / GND / thermal pad | `GND`, via array under the pad |

Output capacitors: C11/C12 and C15/C16, 2×33µF each on `+5VA`/`+5VD` (pre-shunt).

**U7 / U8 INA226 (VSSOP-10).** Unlike the INA260 it has **no internal shunt** — each channel gets an external shunt **R_SH1/R_SH2: 2mΩ 1% ≥1W, 2512**, in series between `+5VA`→`/5V_CH1` (and `+5VD`→`/5V_CH2`); the INA226's VIN+/VIN− Kelvin-connect across the shunt (VIN+ to `+5VA`, VIN− to `/5V_CH1`). Shunt drop at 4.2A = 8.4mV (0.17%), dissipated in the shunt.

| Function | Connect to |
|---|---|
| VIN+ / VIN− | Kelvin across R_SH1/R_SH2 (`+5VA` / `/5V_CH1` sides) |
| VBUS (voltage sense) | `+5VA` / `+5VD` (supply rail, per datasheet reference hookup) |
| VS | `+3.3V` from J2 pin 2, with C21/C22 0.1µF local. VS must stay at 3.3V (I2C VIH = 0.7×VS — at 5V the Portal's 3.3V pull-ups would not register high) |
| GND | `GND` |
| SDA, SCL | `/SDA`, `/SCL` |
| A1, A0 | U7: A1=GND, A0=GND → 0x40. U8: A1=VS, A0=GND → **0x44** (as drawn; A1=GND/A0=VS would be 0x41) |
| ALERT | NC (register-only alerts) |

**Firmware must program the INA226 calibration register (0x05):** CAL = 0.00512 / (Rshunt × Imax), e.g. 2mΩ and 5A → CAL = 512. Shunt-voltage reg 0x01 LSB 2.5µV, bus-voltage 0x02 LSB 1.25mV, current 0x04, power 0x03, Mask/Enable 0x06 — **(verify the alert-limit register against the INA226 datasheet)**.

**After the shunt/sensor:** `/5V_CH1`/`/5V_CH2` → C19/C20 **220µF/10V SMD polymer (Panasonic EEEFPA221XAP)** bulk → polyfuses F2/F3 2920L700/12MR (7A hold; ≈4.8A hold at 70°C per the Littelfuse derating table, survives the 4.2A channel current in a hot case; 12V rating fine at 5V, never on `VBUS`) → screw terminals J4/J5. One terminal per channel; if both panels of a pair get individual feeds, add a second terminal + ≈4A-hold PTC per feed, star-wired with short 14–16 AWG leads. Bulk is 220µF (not 2200µF) because the panels have their own onboard input caps — startup into 220µF ≈ 0.22A/channel, trivial. A PTC will not clear a dead short fast (the buck hiccups first) — it is the sustained-overload backstop.


### Sheet 5: Connectors

**J2 STEMMA QT (JST SH 4-pin):** pin 1 GND, pin 2 `+3.3V`, pin 3 `/SDA`, pin 4 `/SCL`. The
Portal's QT port has the I2C pull-ups to 3.3V; the board relies on those.

**Portal links:**

| Link | Connector | Direction |
|---|---|---|
| 5V tap (≤1A) | J3 2-pin screw terminal (`+5V` via F1, `GND`) → Portal M3 power screw terminals (+5V, GND) | out |
| I2C and 3.3V | J2 (QT cable) | in/out |

## 5. Behavior and firmware interface

### Behavior

The panel channels run whenever VDC is at or above ≈8V (the supervisor drives `/POW_EN` high);
every other state fails to off. Note: with no CH_EN, the channels come up before the Portal has
booted — blank HUB75 as fast as possible at boot to avoid a power-on flash.

| Condition | VBUS | LED | Channels | Portal | Firmware action |
|---|---|---|---|---|---|
| Good PD source, Portal booting | ≈20V | D1 off | on (/POW_EN high) | powered via tap | Read the HUSB238 contract and set the power cap; blank HUB75 ASAP. |
| Good PD source, loaded | 19–20V | D1 off | on | powered | Read INA226s every 100ms; step brightness down if a channel reads above 4.6A for 3 reads; step back up below 4.0A sustained; keep estimated input power under 85% of the contract V × I. |
| Lower PD contract (e.g. 15V/3A) | ≈15V | D1 off | on | powered | Cap panel power to ≈85% of V × I by lowering brightness; log the contract over Serial/WiFi. |
| Channel overcurrent | ≈20V | D1 off (ALERT is register-only) | cycle-by-cycle limit; rail sags; hiccup on sustained short | powered | Poll the INA226 alert flag; governor steps brightness down; log and flag. |
| Dumb 5V source / rejected request | ≈5V | **D1 on** | off (/POW_EN low: VDC < 8V) | ≈4.8–4.9V on its rail; should still boot (Q1 stays on absent a PD fault) | D1 is the local warning; if the Portal boots, report the missing contract over Serial/WiFi. |
| Unplugged or fault | 0V | off | off | dies | None. |

The matrix cannot show a warning when the channels are disabled, so the on-board LED plus
Serial/WiFi from a live Portal are the warning surface.

### I2C devices

| Address | Device | Role |
|---|---|---|
| 0x40 | INA226 #1 (U7) | Channel 1 (panels 1–2) |
| 0x44 | INA226 #2 (U8) | Channel 2 (panels 3–4) |
| 0x08 | HUSB238 (U1) | Source capabilities, negotiated contract, PDO selection |
| 0x19 | LIS3DH (on the Portal) | Onboard accelerometer; no conflict |

INA226 register notes **(verify against the datasheet)**: program CAL (0x05) = 0.00512/(Rshunt×Imax)
at boot; shunt voltage 0x01 (LSB 2.5µV), bus voltage 0x02 (LSB 1.25mV), power 0x03, current 0x04,
Mask/Enable 0x06 (alert enable). With 2mΩ, 4.8A = 9.6mV shunt = 3840 counts of 2.5µV. Firmware must
set CAL and enable the over-current alert at boot.

### Boot and shutdown sequence

1. Boot; blank the HUB75 outputs and drive the data pins low (channels are already enabled by `/POW_EN`).
2. Read the HUSB238 (source capabilities and negotiated contract). Optionally select a better PDO over I2C. **Change PDOs only while the input is invalid or the load is off**, because the rail voltage moves.
3. Compute the power cap: about 85% of the contract V × I. If the HUSB238 read fails, start at a conservative brightness.
4. Program the INA226 CAL register and alert limits (≈4.8A).
5. Run: read INA226s every 100ms; input power ≈ (5.05V × (I1 + I2) + P_housekeeping) / 20V / η with P_housekeeping ≈ 1.5–3W.
6. Shutdown: blank the HUB75 outputs and drive the data pins low before the input disappears, so the Portal does not back-power unpowered panels through their input protection.

The governor is an anomaly detector and thermal backstop, not a normal-operation limiter.
Sustained readings above 4.6A mean a panel with a different driver current, a short or a wiring fault.

## 6. Bill of materials

| Ref | Qty | Part / value | Package | Notes |
|---|---|---|---|---|
| J1 | 1 | USB-C receptacle USB4085-GF-A_REVA4 | SMD+THT | shell to GND |
| U1 | 1 | HUSB238_002DD | DFN-10L 3×3 | VSET open = 20V; ISET R4 22.6k = 3A |
| R1 | 1 | 10Ω | 0603 | VIN series R (datasheet typical) |
| R2 | 1 | 49.9kΩ | 0603 | Q1 gate pull-up (datasheet typical) |
| R3 | 1 | 5.1kΩ | 0603 | GATE series R (datasheet typical) |
| R4 | 1 | 22.6kΩ 1% | 0603 | ISET = 3A |
| C1 | 1 | 1µF 50V | 0603/0805 | at U1 VIN |
| Q1 | 1 | AOSP21357 | SOP-8 wide | PMOS OVP switch VBUS→VDC, ±25V gate; **footprint: use AOSP21357:SO8SOP-8L_AOS, not narrow SOIC-8 (open item)** |
| U2 | 1 | AP63205WU-7 | TSOT26 | fixed 5V |
| L1 | 1 | 4.7µH, ≥2.7A, DCR<100mΩ | SMD | SW→VCC |
| C2 | 1 | **Chemi-Con EMVH350ARA101MHA0G** (100µF/35V SMD polymer) | SMD Ø8×10 → `Capacitor_SMD:CP_Elec_8x10` | VDC bulk at the buck VIN cluster |
| C3, C4 | 2 | 4.7µF 50V X7R | SMD | VDC ceramics |
| C5 | 1 | 10µF 50V | SMD | U2 VIN |
| C6 | 1 | 100nF | 0603 | BST–SW |
| C7, C8 | 2 | 22µF | | VCC |
| C18 | 1 | 1µF | 0603 | VCC |
| U5 | 1 | LM66100DCK | SC-70-6 | ideal diode VCC→+5V; CE→VOUT, ST NC |
| C17 | 1 | 1µF | 0603 | +5V tap |
| F1 | 1 | Polyfuse 2920L150/12MR (1.5A hold, 12V) | 2920 | +5V tap: U5 VOUT → J3 |
| J3 | 1 | screw terminal 5.08mm | THT | +5V (via F1), GND → Portal M3 power terminals |
| U3, U4 | 2 | LMR51450SDRRR | WSON-12 3×3 | PFM + spread spectrum; RT open = 500kHz |
| L2, L3 | 2 | 4.7µH, 6A RMS, 10A sat | SMD | |
| C9, C13 | 2 | 100nF 16V | 0603 | BOOT–SW |
| C10, C14 | 2 | 33pF | 0603 | feedforward |
| C11, C12, C15, C16 | 4 | 33µF | | channel outputs, pre-shunt |
| R5, R11 | 2 | 100kΩ 1% | 0603 | FB top |
| R6, R10 | 2 | 19.1kΩ 1% | 0603 | FB bottom → ≈4.99V |
| R7, R12 | 2 | 1kΩ | 0603 | feedforward |
| U6 | 1 | **LM397** (alt LM393DR2G) | SOT-23-5 | open-collector OUT + R17 pull-up; Vref divider R8/R9; R16 hysteresis |
| R13 | 1 | 820kΩ 1% | 0603 | divider top |
| R14 | 1 | 43kΩ 1% | 0603 | divider bottom → ≈8.0V threshold @ 400mV ref |
| R17 | 1 | 10kΩ | 0603 | /POW_EN pull-up to VCC |
| R15 | 1 | 1kΩ | 0603 | D1 series |
| D1 | 1 | LED (red/amber) | 0603/0805 | fault LED, on when /POW_EN low |
| D2, D3 | 2 | LED (green) | 0603/0805 | channel power-available, on `+5VA`//`+5VD` via R18/R19 |
| R18, R19 | 2 | 10kΩ (as drawn; 2.2–4.7k for brighter) | 0603 | D2/D3 series |
| U7, U8 | 2 | **INA226** | VSSOP-10 | 0x40 / 0x44; needs external shunt + CAL register |
| R_SH1, R_SH2 | 2 | 2mΩ 1% ≥1W | 2512 | channel shunts in series `+5VA`→`/5V_CH1`, `+5VD`→`/5V_CH2`; INA226 Kelvin across each |
| C21, C22 | 2 | 0.1µF | 0603 | INA226 VS decoupling on +3.3V |
| R8, R9 | 2 | 115kΩ / 10kΩ | 0603 | U6 Vref divider (0.4V from VCC) |
| R16 | 1 | 10MΩ | 0603 | U6 hysteresis (OUT→IN+) |
| J2 | 1 | JST SH 4-pin (STEMMA QT) | SMD 1mm | GND / +3.3V / /SDA / /SCL |
| F2, F3 | 2 | Polyfuse 2920L700/12MR (7A hold, 12V) | 2920 | channel feeds: C19/C20 → J4/J5; never on VBUS |
| C19, C20 | 2 | **Panasonic EEEFPA221XAP** (220µF/10V SMD polymer) | SMD Ø6.3×7.7 → `Capacitor_SMD:CP_Elec_6.3x7.7` | channel bulk on /5V_CH1//5V_CH2 (220µF suffices — panels have their own caps) |
| J4, J5 | 2 | screw terminal 5.08mm | THT | /5V_CH1, /5V_CH2 panel feeds (via F2/F3) |

**Value-field convention (done in the latest schematic — cap ratings are in the Value fields: C3/C4 `4.7uF/50V X7R`, C5 `10uF/50V X7R`, C7/C8 `22uF/10V X7R`, C11–C16 `33uF/10V X7R`, C1 `1uF/50V`, C2/C19/C20 full part numbers; small decoupling caps C6/C9/C10/C13/C14/C17/C18/C21/C22 still generic — 16–25V X7R/C0G is fine for them):** originally: put the key rating in the Value field so
it flows into the BOM and part selection — capacitors get `µF/voltage dielectric` (voltage rating
is critical at 20V bias: a 25V ceramic loses most of its capacitance on `VDC`), precision
resistors get `1%`, inductors get their current rating, electrolytics get `low-ESR`. Suggested
values: C1 `1uF/50V` ✓, C2 `100uF/35V` ✓, C3/C4 `4.7uF/50V X7R`, C5 `10uF/50V X7R`, C6 `100nF/25V`,
C7/C8 `22uF/10V X7R` (use 0805/1206 10V parts), C9/C13 `100nF/16V`, C10/C14 `33pF/50V C0G`,
C19/C20 `EEEFPA221XAP`, C21/C22 `0.1uF/25V`;
R4/R5/R6/R10/R11 `… 1%` (ISET + FB divider), others 5%; L1 `4.7uH ±20% Irms≥2.7A Isat≥3.5A DCR<100mΩ`, L2/L3 `4.7uH ±20% Irms≥6A Isat≥8A` (see inductor note below);
D1–D3 `LED … Vf ≤ 2.2V`. F1–F3 already carry full part numbers.

**Capacitors need ratings, not brands** (except the electrolytics): dielectric **X7R** for all power
ceramics (C0G/NP0 only for the 33pF feedforward C10/C14), never X5R on the 20V rail, and voltage rating
with DC-bias margin as listed above (50V on `VDC`, ≥10V on the 5V rails). The two electrolytics are the
only ones worth a named part: C2 = Chemi-Con **EMVH350ARA101MHA0G** (100µF/35V SMD polymer, Ø8×10,
ripple-rated for the `VDC` input ripple) and C19/C20 = Panasonic **EEEFPA221XAP** (220µF/10V SMD
polymer, Ø6.3×7.7) — both SMD, so JLCPCB assembles them (hand-solder only the screw terminals now).
220µF/channel bulk is enough because the panels carry their own input caps. Everything else can be
any X7R part in the right size.

Not fitted (deliberate): input fuse on `VBUS` (§4 sheet 1 conditions), TVS on `VDC` (DNP
footprint optional, §10 scope check decides), second panel terminal per channel, green/amber
LEDs, on-board I2C pull-ups. Package notes assume JLCPCB assembles SMD and you hand-solder THT
(screw terminals only — the electrolytics are SMD now). Confirm LCSC stock for every part when exporting the BOM.

## 7. Power budget

Basis: one 32×32 panel measured at 2.08A at full white (Portal plus panel 2.20A at 5.15V; Portal
alone 0.12A), designed at 2.1A per panel. Converter efficiency is the LMR51450 datasheet figure
of 91% at 4A with a 24V input; 91% is the design value and 90% the worst case.

Four panels = 4096 LEDs ≈ 8.4A at 5V at full white — far more than the Portal's 3A USB port can deliver (≈1500 LEDs). That is the capacity this board unlocks.

| Item | Value |
|---|---|
| Per panel / channel / both | 2.1A / 4.2A / 8.4A |
| Output power at 5.05V | ≈42.4W |
| Channel input power at 91% / 90% | ≈46.6W / ≈47.1W |
| Portal, housekeeping, LEDs | ≈1.5W (≈3W with WiFi); measured Portal baseline ≈0.6W |
| Total input, typical (91%) | ≈48W |
| Total input, worst case (90% plus WiFi) | ≈50W |
| Input current at 20V | ≈2.4–2.5A (source limit 3A) |
| Margin vs. the 60W contract | ≈10–12W (16–20%) |
| Heat per channel | ≈2.1–2.4W |

- Both channels at their full 5A rating would pull about 56–59W. The LMR51450's deliverable maximum is about 5.7–7.3A per channel (6.5A typical): high-side limit 6.4–9.6A, low-side limit 5A. A double overload could pull about 72W at the input, above the 60W contract, so the source's protection, the panel polyfuses, the INA226 alert and the governor are the backstops.
- An overload gives cycle-by-cycle limiting. A short that pulls FB below 40% of the reference triggers hiccup with a 96ms off time.
- Startup: soft start ≈5ms (3.2–7.2ms), so 220µF draws only ≈0.22A per channel at startup (panels' own caps add theirs at their end).
- Verify: put a USB tester that supports 20V PD between the charger and the board at full white to read real input power, then log INA226 peaks across all four panels.

## 8. Layout and manufacturing notes

- **Board outline:** 57.3 × 49.8 mm (Edge.Cuts), single-sided component placement, 2× M2 mounting holes.

- **LMR51450 layout:** follow TI's EVM-style layout. θJA is 47.4°C/W (23°C/W on TI's EVM board). Solder the exposed pad with a via array (fill, plug or tent the vias under the pad as TI recommends), use ground planes, and keep the input capacitors right at the VIN and PGND pins. 2oz copper is preferred; check availability and restrictions with assembly at JLCPCB.
- **Buck placement / channel separation:** no fixed spacing needed between U3 and U4 — the coupling is conducted (shared `VDC`) and radiated from the SW nodes, not distance-dependent. What matters: keep each channel's hot loop (SW pins → L → output caps → GND) compact and **do not interleave the two channels' SW/inductor areas**; keep SW copper area minimal and away from FB, I2C, the INA226s/shunts and the panel terminals; per-pin input ceramics + C2 bulk handle the shared-rail coupling; spread the two bucks thermally (2.1–2.4W each) and don't share a small copper island. Shielded inductors + 500kHz spread spectrum do the rest.
- **Current paths:** use copper pours or wide traces for `VBUS`, `VDC`, `+5VA`/`+5VD`, `/5V_CH1`/`/5V_CH2` and the panel feeds (roughly 3mm or more for 5A on 1oz outer copper; check with a trace-width calculator). Keep the high-current loops (buck → shunt → INA226 → bulk → fuse → terminal) short and separate from the I2C and sense traces.
- **Copper pours (zones) in KiCad:** select the copper layer, use the **Add filled zone** toolbar
  button, click a **closed** polygon around the area, then set the zone's **Net** and **Pad
  connection = Solid** (zone properties) — thermal-relief spokes are wrong for power/thermal pads.
  Fill with **B** (or right-click → Zones → Fill All Zones); re-fill after moving parts. Whole-board
  GND plane on B.Cu; per-rail pours (VDC/+5VA/+5VD/SW hot loop) as tight F.Cu polygons on the same
  net. Stitch U3/U4 thermal pads to the plane with a via array (Add Via tool, net GND, solid).
- **Star ground:** put the star point near the channel bulk capacitors. Do not route panel return current through the Portal's ground.
- **AP63205:** θJA is 89°C/W on a small board, so each 0.5W of loss adds about 45°C. Use a ground plane, 2oz copper if available, and vias on the GND pad of the capacitors.
- **Creepage and enclosure:** 20V at 3A is 60W; fuse the rails, strain-relieve all cables.
- **JLCPCB assembly:** SMD parts are placed by JLCPCB; through-hole parts (only the screw terminals now) are hand-soldered by you. Extended parts carry a loading fee per unique part number, so keep the number of different values low and prefer Basic and Preferred parts for passives.

### Test points

✅ **Done in the schematic (netlist-verified):** six `TestPoint:TestPoint_Pad_D2.0mm` pads — TP1 `VBUS`,
TP2 `VDC`, TP3 `VCC`, TP4 `+5VA`, TP5 `+5VD`, TP6 `/POW_EN`. Nets on screw terminals (`+5V`, `/5V_CH1`,
`/5V_CH2`, and their GND) are probed via the terminal itself — no pads needed there. At layout, add
GND pads (incl. a ground-spring pad next to J4/J5 for ripple probing) and optionally SW-node pads
(U3/U4) for scoping the switch waveform. A 0Ω link (0603, Value `0R`) in the `+5V` path and in series
with `VBUS` helps isolate nets during bring-up **(suggested)**.

## 9. Safety notes

- Panels are **5V only**: never feed them 12V or 20V.
- 20V at 3A and 5V at about 8.4A are both high-energy. Protection is each converter's internal limit, the per-rail polyfuses (F1 tap, F2/F3 channels) and the part ratings; never rely on software alone for hard limits. The input fuse on `VBUS` is omitted by decision — the conditions for that are in §4 sheet 1 (compliant PD sources only, current-limited bench supply, `VDC` copper and Q1 sized for ≈5A).
- The HUB75 connector carries **data and GND only — no power**. Panel power goes through the panels' dedicated sockets. The Portal itself is powered (≤1A, via F1) from the board's `+5V` tap through its M3 power screw terminals, not through HUB75 or the A0 JST.
- The Portal ties its USB-C 5V and the M3 power-terminal 5V together directly on its 5V rail; U5 (LM66100) prevents the two sources from fighting.
- The tap cannot be dragged down by the panels: the only path from `VCC` to the Portal's 5V rail is VCC → U5 (ideal diode, reverse-blocking) → F1 (1A PTC) → J3 → M3 terminals, so ribbon back-feed from a panel is hard-capped at ≈1A by F1 and blocked from VCC by U5; upstream of F1 the AP63205 current-limits at ≈2.8A. Worst case F1 trips and only the Portal reboots — VCC (supervisor, LED) and the panel channels (VDC) are upstream and keep running. If the Portal is also on USB, U5 stops USB from back-feeding our buck.
- The lowest-rated part on the 20V rail is the HUSB238 (VIN 30V abs-max); it trips its own OVP at 24V for a 20V request.
- A standard 3A USB-C cable is enough at 20V/3A; more than 3A would need an e-marked 5A cable.

## 10. Bring-up checks

1. **VBUS/VDC scope check:** plug-in, unplug, and full-white on/off. Peaks approaching 24V would restart the board; add input damping first and populate the TVS only for spikes that threaten parts (criterion: > ≈28–30V, Q1 is −30V VDS). Mandatory since no TVS is fitted.
2. **AP63205 bench check:** measure the output on a 5V input under Portal load (about 4.8V expected at the 1A tap limit) and the regulator's temperature at the Portal's real draw.
3. **LMR51450 startup:** confirm clean startup and load steps with the 220µF bulk capacitor on each channel.
4. **Power:** measure real input power at full white and compare with the budget in section 7.
5. **Supervisor:** confirm the 8V threshold and the D1 (fault LED) behavior against the LM397 datasheet.

## 11. Open items before layout

- ✅ **Footprints all assigned** (re-verified 7 Nov 2026): zero empty Footprint fields remain; R/C/L/D/F/J per the table below; J3–J5 = `TerminalBlock_CUI:TB007-508-02`; J2 = JST SH SM04B. Connectivity unchanged vs previous save. Caveat: C5 (10µF) and C7/C8 (22µF) are 0603 — check the LCSC parts' voltage ratings (want ≥25V for C5 on the 20V rail, ≥10V X7R for C7/C8 on the 5V rail; 0603 22µF is usually 6.3V-rated with heavy DC-bias loss — prefer 0805/1206 there).
- ✅ Q1 footprint fixed: `AOSP21357:SO8SOP-8L_AOS-L` (wide SOP-8).
- ✅ Symbol mismatches all resolved (latest save, ERC-verified).
- ✅ U3 SW pin 1 reconnected (netlist-verified: `Net-(C9-Pad2)` = U3.1/2/3 + C9.2 + L2.1). ✅ All `lib_symbol_mismatch` warnings gone (custom symbols re-synced to the project libs). One cosmetic leftover: a single stray unconnected wire endpoint (1.27mm stub) — delete it in the GUI.
- ERC cosmetics (optional): PWR_FLAGs on label-only power nets; no_connect flags on the intentional NC pins (J1 SBU/DP2/DN2, U1 VSET, U3/U4 PG+RT, U5 NC+ST, U6 OUTB, U7/U8 ALERT+NC).
- ✅ `AP63200-AP63201-AP63203-AP63205.pdf` added to the datasheet folder.
- Verify INA226 register map (CAL 0x05, alert-enable) and HUSB238 register map (source capabilities, negotiated contract, PDO select).
- JLCPCB: stock and library class of every part, 2oz copper and plugged-via options, final LCSC numbers (part swaps done 8 Nov for stock/price: LM397 for TPS3700 (LM393DR2G as alternate), INA226 + 2mΩ shunts for INA260, EMVH350ARA101MHA0G / EEEFPA221XAP SMD polymer caps).
- Firmware: program INA226 CAL register (0x05) at boot before reading current; verify INA226 alert-limit register.
- **Board temperature sensor (recommended add):** TMP102/LM75A on the QT I2C bus at 0x48 (free address),
  powered from `+3.3V` (J2 pin 2), placed between U3 and U4; firmware throttles brightness above
  ≈65–70°C. Zero-part alternative: 10kΩ NTC divider from `+3.3V` to GND with the tap wired to the
  Portal's unused A0-JST pin (GPIO12 ADC).
- Firmware: blank HUB75 immediately at boot (channels auto-enable at VDC ≥ ≈8V, before the Portal is up).
- D2/D3 channel LEDs added and netlist-verified (D2: +5VA→R18→GND, D3: +5VD→R19→GND; values GREEN LED, R18/R19 10k).
- **HUB75 5V back-feed (check your panels/cables):** the MatrixPortal's HUB75 port carries 5V on the same net as USB-C 5V and the M3 terminals (which our 1A F1 tap feeds). If the panel's IDC connector and the ribbon cable carry 5V (HUB75 is a loose spec — some panels/cables parallel IDC-5V with the panel 5V rail, standard 16-wire cables usually do not), the panel chain's 5V rails get a second feed from the Portal through the ribbon: current splits between the low-resistance channel feed (J4/J5 power sockets) and the high-resistance ribbon+Portal-PCB path, so up to F1's 1A can flow through the ribbon into the panels, heating ribbon wires/Portal traces, with the return current flowing through the Portal's GND instead of the power board's star point. Note the dedicated 2-pin power socket with 16AWG wires (≈0.004Ω/m) dominates the thin HUB75 ribbon (≈0.13Ω/m for 26AWG) by ~30× in resistance, so in normal operation the ribbon's parasitic share is only a few percent; it is hard-capped at F1's 1A worst case (e.g. if the channel rail sags at current limit). If your HUB75 link carries 5V, still do not connect it at the Portal end (keep GND for the signal return) so panel power and its return flow only via the 16AWG power path to the board's star point.
- **Panel reverse polarity:** the 2-pin panel power plugs are user-wired and the panels themselves have no reverse protection. Consider a PMOS reverse-protection (or series Schottky) on each channel output, before F2/F3.
- **Thermal budget:** compute per-part Tj at full load (8.4A) in worst-case ambient from the real copper (LMR51450 θJA 47.4°C/W 1oz / 23°C/W on the EVM, AP63205 89°C/W, Q1 ≈0.05W) and verify with an IR camera at bring-up; the 2.1–2.4W/channel figure in §7 implies a large rise on 1oz.

### Footprint assignment (for Tools → Assign Footprints)

| Refs | Value(s) | Recommended footprint |
|---|---|---|
| R1–R19 (all resistors) | 10Ω … 820kΩ | `Resistor_SMD:R_0603_1608Metric` |
| C1, C6, C9, C10, C13, C14, C17, C18, C21, C22 | 33pF–1µF | `Capacitor_SMD:C_0603_1608Metric` |
| C3, C4 | 4.7µF 50V (VDC) | `Capacitor_SMD:C_0805_2012Metric` |
| C5 | 10µF 50V (U2 VIN) | `Capacitor_SMD:C_0805_2012Metric` |
| C7, C8 | 22µF (VCC) | `Capacitor_SMD:C_1206_3216Metric` (10V X7R) |
| C11, C12, C15, C16 | 33µF (channel outputs) | `Capacitor_SMD:C_1206_3216Metric` |
| C2 | 100µF 35V — **EMVH350ARA101MHA0G** (SMD polymer) | `Capacitor_SMD:CP_Elec_8x10` |
| C19, C20 | 220µF 10V — **EEEFPA221XAP** (SMD polymer, Ø6.3×7.7) | `Capacitor_SMD:CP_Elec_6.3x7.7` |
| D1, D2, D3 | LEDs | `LED_SMD:LED_0603_1608Metric` |
| F1, F2, F3 | 2920L…/12MR PTC | `Fuse:Fuse_2920_7451Metric` |
| L1, L2, L3 | 4.7µH shielded (MHCI06050-4R7M-R8A for all three) | `Inductor_SMD:L_6.3x6.3_H3` |
| L2, L3 | 4.7µH, Isat≥6.5A, Irms≥5A (channels) | `Inductor_SMD:L_6.3x6.3_H3` (MHCI06050-4R7M-R8A) |
| J3, J4, J5 | 2-pin 5.08mm screw terminals | `TerminalBlock_CUI:TerminalBlock_CUI_TB007-508-02_1x02_P5.08mm_Horizontal` (JLCPCB-standard part; if you buy Phoenix MKDS/MSTB instead, use `TerminalBlock_Phoenix:TerminalBlock_Phoenix_MKDS-1,5-2-5.08_1x02_P5.08mm_Horizontal` to match) |

Inductor footprints must match the actual inductor parts picked at BOM time (check the LCSC part's datasheet package); same for the radial electrolytics' diameter/pitch.

**Inductor selection notes:** tolerance **±20%** is standard for power inductors and is fine here — the
bucks are peak-current-mode with internal compensation, and the ripple/limit math already has margin
(at L−20%, ΔI ≈ 2.0A, worst peak ≈ 5.2A < the 6.4A min HS current limit). What matters is **saturation and heating current, not the EVM's exact numbers** — you can compromise:
L2/L3 hard floor is **Isat ≥ ≈6.5A** (must not saturate below the LMR51450's 6.4A min HS current limit;
worst peak ≈ 5.3A at L low) and **Irms ≥ ≈5A** (4.2A load + ripple). The EVM's 7A/15mΩ is conservative —
DCR can be worse, up to ≈50mΩ, at the cost of ≤0.9W extra heat in the inductor (channel budget has room).
Inductance can move within **≈3.3–6.8µH** (ripple stays 25–55% of 4.2A; internal compensation tolerates
this) — so a cheap 5.6µH or 6.8µH shielded part is fine. Shielded is **not** compromisable (panels
nearby). L1 (housekeeping, ≤1.2A load): Irms ≥ 1.5A, Isat ≥ 2.7A, DCR < 100mΩ — lots of room there. **Termination/package: SMD, shielded**
(molded metal-alloy or shielded drum core), tape & reel for JLCPCB — unshielded drum cores radiate
into the HUB75 panel wiring; on shielded parts connect the winding-start dot to the SW node for extra
shielding. L2/L3 = **MPS MHCI06050-4R7M-R8A** (decided 8 Nov, cost/sourcing): molded (shielded) power choke,
4.7µH ±20%, 6.0×6.0×5.0mm, DCR 25mΩ max, Irms 7A, Isat 6.5A (30% drop) — meets the L2/L3 floors
(Isat ≥ 6.5A = just above the 6.4A chip limit, Irms ≥ 5A, DCR ≤ 50mΩ, shielded); footprint
`Inductor_SMD:L_6.3x6.3_H3` (fits the 6.0mm body; or MPS's official land pattern). Alternatives if
out of stock: TDK SPM10040T-4R7M-HZR (10×10×4mm, Isat/Irms ≈7.9A, DCR 18mΩ — footprint
`L_Bourns_SRP1038C_10.0x10.0mm` fits), Bourns SRP1038CC-4R7M, or the EVM's Wurth 74437368047.
L1 = **same part as L2/L3: MPS MHCI06050-4R7M-R8A** (decided 8 Nov to simplify the BOM — one inductor
for all three positions): far above the AP63205 requirement (Isat > 2.7A, DCR < 100mΩ) for the ≤1.2A
housekeeping load; ripple at 1.1MHz/1.2A is ≈0.7A, fine. Footprint `Inductor_SMD:L_6.3x6.3_H3`.
Check the 5.0mm height fits the enclosure. Confirm the LCSC number/stock at BOM time.

## 12. Reference: datasheet-derived values used

| Part | Values used |
|---|---|
| HUSB238 | I2C 0x08; I2C pins 2.5–5.5V; VIN and GATE rated 30V, CC pins 25V; typical circuit: VIN via 10Ω + 1µF, GATE → PMOS via 5.1kΩ with 49.9kΩ pull-up (opens only on OVP/OTP/hard-reset); OVP at 120% of requested voltage (24V for 20V); ISET sources 100µA, 22.6kΩ → 3A; VSET open = 20V; internal Rd on CC; `_002DD` moves to the next PDO on OTP (150/130°C) — the `_001`/`_003` drop to 5V (90/75°C) |
| AP63205 | Fixed 5V, 2A, 3.8–32V in; VIN abs-max 35V DC (40V for 400ms); EN rated to 35V; 4.7µH, 10µF in, 2×22µF out, 100nF boost; soft start 4ms; peak limit ≈2.8A with hiccup after 2ms; UVLO ≈3.5V rising, 440mV hysteresis; θJA 89°C/W |
| LMR51450 | 4–36V in (38V abs-max); 91% at 4A, 24V in; 0.8V ±1% reference; RT open = 500kHz default, RT shorted = 1MHz (commercial part; fsw 200kHz–1.1MHz with RT; only the -Q1 variant forbids floating RT); soft start 5ms (3.2–7.2ms); HS limit 6.4–9.6A, LS limit 5A; hiccup 96ms; EN 1.25V rising; θJA 47.4°C/W (23°C/W on EVM); SDRRR = PFM + spread spectrum (EVM part), FNDRRR = FPWM no SS; Table 9-2 (500kHz): 4.7µH, 2×33µF, 100kΩ/19.1kΩ, 33pF + 1kΩ |
| INA226 | External shunt required (2mΩ 1% 1W per channel); CAL register 0x05 = 0.00512/(Rshunt×Imax); shunt reg 0x01 LSB 2.5µV, bus 0x02 LSB 1.25mV; VS 2.7–5.5V (I2C VIH = 0.7×VS); VBUS to supply rail; ALERT open-drain; addresses 0x40–0x4F via A1/A0 — as drawn: U7 A1=GND/A0=GND → 0x40, U8 A1=VS/A0=GND → 0x44 |
| LM66100 | 1.5–5.5V in, 1.5A, ≈79mΩ, SC-70-6; MOSFET on while CE < VIN, off when CE > VIN (CE→VOUT valid, blocks back-feed); ST open-drain status |
| LM397 | Single comparator, open-collector output (pull-up required — R17), 5–30V single supply, **no internal reference** (external 0.4V from VCC divider R8/R9), input common-mode includes ground (0 to VCC−1.5V; tap ≈1V fine), low offset; add R16 ≈10M output→IN+ for a few mV hysteresis; threshold = 0.4 × (1 + 820/43) ≈ 8.0V, tracks VCC ±4%. (LM393DR2G = dual SOIC-8 alternate, same hookup.) |
| AOSP21357 | P-ch SO-8 (wide 5.23mm), VDS −30V, VGS ±25V, Rds(on) 8.5mΩ, ID −16A; AOS's replacement for the EOL AO4455 |
| Littelfuse 2920L PTC | 2920L150/12MR: 1.5A hold; 2920L700/12MR: 7A hold (≈4.8A at 70°C), 12V max, 50A interrupt, R 4–20mΩ; never on VBUS (20–24V) |
