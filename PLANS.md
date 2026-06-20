# Plans / next steps

_ultranetMonitor project planning. General KiCad setup/library/tooling now lives in the sibling `kicad-shared` repo._

Last updated: 2026-06-20

## Board: `ui-tile-1` (UI interface tile)

RP2040-based interface tile. Eventually mounts to a mainboard. Front-panel: **4 rotary encoders** (A/B + push), an **SPI LCD**, and **4 WS2812** LEDs (one per encoder, spread around the board). Talks to the mainboard over **SPI + IRQ**.
Project files: `ui-tile-1/ui-tile-kicad/ui-tile-kicad.kicad_sch` (+ `.kicad_pcb`).

## Status

**Schematic: COMPLETE and ERC-clean (all sheets).** Hierarchy: root block diagram → `rp2040`, `encoders`, `lcd`, `leds`, `headers` sub-sheets.

- **RP2040 core:** U1 + W25Q16 QSPI flash (U2) + 12 MHz crystal (Pierce, 30 pF loads + 1k series R). Boot (SW1) + Run/reset (SW2) buttons w/ 10k pull-ups.
- **Power:** USB-C (USBC1) and mainboard 5 V **diode-OR'd** (D1/D2 = B5819W Schottky) into **VSYS** → NCP1117 LDO → **+3V3**. RP2040 internal reg makes +1V1. PWR_FLAGs on +5V and VSYS.
- **USB-C:** D± series Rs, 5.1k CC pulldowns, two power-indicator LEDs.
- **Encoders:** SW3–6, A/B/SW each wired to RP2040 (see GPIO map).
- **LCD (LCD1 = HS180S10B, LCSC C5329585):** SPI1 + DC/RES/CS. VCC pins (2,11)→+3V3, GND (1,12)→GND, FCS (10,14)→+3V3 (flash held deselected), FS0 (9,15) NC. Backlight via low-side MOSFET (see below).
- **WS2812 (LED3–6):** GPIO5 → **74AHCT1G125** level-shifter (U4, on VSYS) → string. Powered from VSYS. Decoupling added (see layout notes).
- All JLCPCB-imported symbol pin types cleaned up; ERC clean.

### Backlight sub-circuit (low-side N-MOSFET, PWM on GPIO7)
```
VSYS ─[R11 68Ω 0603]─ BLA(13)   BLK(8) ─ Q1·D
                                         Q1·S ─ GND
GPIO7 ─[R10 100Ω]─ Q1·G ─[R12 10k]─ GND
```
- Q1 = 2N7002 (SOT-23: G=1, S=2, D=3). R11 carries backlight current (~26 mA, ~53 mW → 0603). R12 holds FET off while GPIO7 is Hi-Z at boot.
- R11 value assumes ~3 V backlight; verify Vf on bench, drop to 100 Ω if too bright.

## NEXT: PCB layout (starting 2026-06-21)

Export the netlist and have Claude review after any further schematic changes (`ui-tile-kicad.net`).

### General layout watch-items
- Decoupling caps hugging power pins (U1 IOVDD/DVDD, U2, U4).
- Crystal + 30 pF load caps tight to U1.20/21; keep loop small.
- USB D± as a matched pair (≈90 Ω differential), short.
- NCP1117 (U3) tab on a generous +3V3 pour with thermal vias.

### WS2812 string (distributed — one LED per encoder)
- Place each **100 nF (C18–C21) right at its LED's VDD/GND pins** — these matter now that pixels are far apart on long VDD traces.
- **C22 (10 µF) bulk** near the power feed-in / first pixel (LED3).
- **R13 (470 Ω) close to the buffer U4 output** (damps the long run to the first pixel). Inter-pixel hops (LED3→4→5→6) need no resistors — each WS2812 re-buffers.
- VSYS trunk to the LEDs wide enough for **~240 mA** worst case (4 × ~60 mA full white); solid GND return.
- Put level-shifter U4 near the first pixel / GPIO5 exit.

### Backlight / LCD
- Keep Q1 + R10/R11/R12 grouped near the LCD connector.

## Mainboard interface (headers H1/H2)
- Two 1×8 stacking headers wired **in parallel** (intentional). Pinout per header: 1=+5V, 2=CS, 3=SCK, 4=MISO, 5=MOSI, 6=IRQ, 7=GND, 8=GND.

## GPIO pin map (PROVISIONAL — optimize during routing)

RP2040 has 30 GPIO (0–29). Using **24**, 6 spare. Assignments are firmware-defined and flexible — let routing drive final placement; only hardware-SPI is lightly constrained, QSPI/USB/crystal/power are fixed.

| GPIO | Use | GPIO | Use |
|---|---|---|---|
| 0 | Mainboard MISO (SPI0 RX) | 16 | Enc1 A |
| 1 | Mainboard CS (SPI0 CSn) | 17 | Enc1 B |
| 2 | Mainboard SCK (SPI0) | 18 | Enc1 SW |
| 3 | Mainboard MOSI (SPI0 TX) | 19 | Enc2 A |
| 4 | Mainboard IRQ | 20 | Enc2 B |
| 5 | WS2812 data → level shifter | 21 | Enc2 SW |
| 6 | LCD RST | 22 | Enc3 A |
| 7 | LCD BL (PWM) | 23 | Enc3 B |
| 8 | LCD DC | 24 | Enc3 SW |
| 9 | LCD CS (SPI1 CSn) | 25 | Enc4 A |
| 10 | LCD SCK (SPI1) | 26 | Enc4 B |
| 11 | LCD MOSI (SPI1 TX) | 27 | Enc4 SW |
| 12–15 | spare | 28–29 | spare (ADC-capable) |

- Mainboard = SPI0, LCD = SPI1 (two hardware SPI blocks). Encoders + WS2812 via **PIO**.
- Encoder A/B pairs are on adjacent pins (16/17, 19/20, 22/23, 25/26) for PIO quadrature.

## Open questions / later
- [ ] **Mainboard SPI master/slave** — assumed tile = slave (IRQ = tile→mainboard). If master, flip the `MB_` SPI directions.
- [ ] Pin assignments provisional — expect to shuffle for routing; keep functional net names so it's painless.
- [ ] Possible later: factor the RP2040 core into a reusable design block (template already cloned to `kicad-shared/templates/rp2040-template/`).
