# MatrixChain Power Board — Datasheet

Rev A. Companion to `DESIGN.md` (design rationale) — this file is the board's user-facing spec.

Custom power board that feeds **four 32×32 HUB75 LED panels (4096 LEDs)** and an **Adafruit
MatrixPortal S3** from one USB-C PD source. The Portal's MCU can drive 4096 LEDs but its USB-C port
only supplies 5V/3A (~1500 LEDs at full white); this board supplies the panel power externally so the
Portal can run at full capacity.

## Features

- USB-C PD sink, requested contract **20V/3A (60W)**; lower contracts accepted (channels stay off below ≈8V input)
- HUSB238 PD controller with external PMOS (AOSP21357) hard OVP/OTP disconnect switch
- **Two independent 5V/5A channel outputs** (LMR51450SDRRR, 500kHz PFM + spread spectrum), each feeding a panel pair through its own fused screw terminal — panel power never flows over HUB75
- **Housekeeping 5V/2A rail** (AP63205) powering the Portal via a 1A fused tap, isolated by an LM66100 ideal diode; works on a dumb 5V charger
- Per-channel **INA226 current/voltage monitoring** (external 2mΩ shunt per channel) over I2C (0x40/0x44), powered from the Portal's 3.3V
- Supervisor (LM397 comparator + 0.4V reference, ≈8V threshold on the input rail) hard-gates the channels (`/POW_EN`); no firmware enable line
- Status LEDs: red = input not a valid PD contract, green per channel = channel 5V up
- PTC polyfuses on all three 5V outputs; 100µF input bulk; ESD-safe (USB-C shell to GND)
- 6 SMD test pads (VBUS, VDC, VCC, +5VA, +5VD, /POW_EN)
- Board **57.3 × 49.8 mm**, all components on the top side, 2× M2 mounting holes, 2oz copper preferred

## Electrical characteristics

| Parameter | Value |
|---|---|
| Input | USB-C PD sink, 5–20V, requested 20V/3A (60W); OVP trip 24V |
| Channel outputs | 2 × 5V, 4.2A design each (2 panels), 5A converter rating, ≈4.99V set |
| Housekeeping output | 5V/2A buck, 1A fused tap to the Portal (≈4.8–4.9V when running from a 5V input) |
| Efficiency | ≈91% per channel at 4A/24V-in (design), 90% worst case |
| Input current @20V full white | ≈2.4–2.5A (all four panels + Portal); budget margin ≈10–12W |
| Heat per channel | ≈2.1–2.4W |
| Channel enable threshold | input ≥ ≈8V (9V+ PD contracts valid, 5V not) |
| I2C | 3.3V bus (STEMMA QT), pull-ups on the Portal; HUSB238 @0x08, INA226 @0x40/0x44 |

## Connectors

| Ref | Connector | Pinout / mate |
|---|---|---|
| J1 | USB-C receptacle (GCT USB4085-GF-A, 16-pin) | PD source input |
| J2 | STEMMA QT (JST SH 4-pin) | 1 GND, 2 +3.3V (from Portal), 3 SDA, 4 SCL |
| J3 | 2-pin 5.08mm screw terminal | `+5V` (via F1, ≤1A), GND → MatrixPortal **M3 power screw terminals** |
| J4 | 2-pin 5.08mm screw terminal | `/5V_CH1` (via F2), GND → panels 1–2 dedicated power plugs (16AWG) |
| J5 | 2-pin 5.08mm screw terminal | `/5V_CH2` (via F3), GND → panels 3–4 dedicated power plugs (16AWG) |

The HUB75 link to the Portal carries **data + GND only — do not connect its 5V pins** (panel power
must flow via J4/J5 power sockets only). Panel power plugs are not reverse-protected — observe polarity.

## LEDs

| Ref | Color | Meaning |
|---|---|---|
| D1 | red | input not a valid PD contract (`/POW_EN` low) |
| D2 | green | channel 1 5V up |
| D3 | green | channel 2 5V up |

D1 off + D2/D3 off = channels disabled (bad input); D1 off + one green off = that channel faulted.

## Protection

HUSB238 OVP (24V) + Q1 PMOS disconnect (input), PTC F1 2920L150/12MR on the +5V tap (protects the
1.5A LM66100), PTC F2/F3 2920L700/12MR on the channel feeds (sustained-overload backstop; dead shorts
are cleared by the buck's cycle-by-cycle limit/hiccup). No input fuse: only use compliant PD sources
or a current-limited bench supply; VDC copper and Q1 sized for ≈5A. TVS on VDC not fitted — populate
only if bring-up scoping shows >≈28–30V transients.

## Firmware interface (MatrixPortal)

- I2C (QT): HUSB238 @0x08 (source caps, negotiated contract, PDO select — change PDOs only while
  channels are off), INA226 @0x40 (ch1) / 0x44 (ch2) with 2mΩ external shunts: program CAL (0x05) =
  0.00512/(Rshunt×Imax) at boot; shunt V 0x01 (2.5µV/LSB), bus V 0x02 (1.25mV/LSB), current 0x04,
  Mask/Enable 0x06 (enable over-current alert, ≈4.8A = 9.6mV shunt = 3840 counts)
- Boot: blank HUB75 + data pins low ASAP (channels auto-enable at input ≥ ≈8V), read HUSB238 contract,
  cap panel power at ≈85% of V×I, program the INA226 CAL register + alert, then run: read INA226s every 100ms,
  brightness down if a channel >4.6A for 3 reads, up below 4.0A sustained. Shutdown: blank HUB75 first.

## Key components (BOM excerpt)

U1 HUSB238_002DD · Q1 AOSP21357 · U2 AP63205WU-7 · U3/U4 LMR51450SDRRR · U5 LM66100DCK ·
U6 LM397 · U7/U8 INA226 + R_SH1/R_SH2 2mΩ 1% 1W 2512 shunts · L1/L2/L3 MPS MHCI06050-4R7M-R8A ·
C2 Chemi-Con EMVH350ARA101MHA0G (100µF/35V SMD) · C19/C20 Panasonic EEEFPA221XAP (220µF/10V SMD) ·
F1 2920L150/12MR · F2/F3 2920L700/12MR · J1 USB4085-GF-A · exact orderable parts (MPN + LCSC C-code) are in `powerboard.csv`.

## Assembly / bring-up

JLCPCB assembles all SMD parts; hand-solder only the THT screw terminals (J3–J5). Prefer 2oz copper. Bring-up: scope VBUS/VDC at plug/unplug and full-white on/off (TVS criterion),
bench-check AP63205 at 5V-in, confirm LMR51450 startup into 220µF, measure real input power at full
white, IR-camera the channel bucks (thermal budget), verify the ≈8V threshold and LED behavior.
