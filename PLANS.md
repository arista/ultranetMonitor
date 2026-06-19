# Plans / next steps

_ultranetMonitor project planning. General KiCad setup/library/tooling now lives in the sibling `kicad-shared` repo._

Last updated: 2026-06-19

## Board: `ui-tile-1` (UI interface tile)

RP2040-based interface tile. Eventually mounts to a mainboard. Front-panel: **4 rotary encoders** (A/B + push), an **SPI LCD**, and a **string of WS2812** LEDs. Talks to the mainboard over **SPI + IRQ**.
Project files: `ui-tile-1/ui-tile-kicad/ui-tile-kicad.kicad_sch` (+ `.kicad_pcb`).

## Status

**RP2040 core schematic: complete and ERC-clean.** Done so far:
- RP2040 (U1) + W25Q16 QSPI flash (U2) + 12 MHz crystal (Pierce, 30 pF loads + 1k series R).
- Power: USB-C (USBC1) and mainboard 5 V **diode-OR'd** (D1/D2 = B5819W Schottky) into **VSYS** → NCP1117 LDO → **+3V3**. RP2040 internal reg makes +1V1.
- USB-C: D± series Rs, 5.1k CC pulldowns. Two power-indicator LEDs.
- All JLCPCB-imported symbol pin types cleaned up; PWR_FLAGs placed; ERC clean.

## OPEN ITEMS (carry-over)

- [ ] **Mainboard connector is missing.** The `/+5V` net dead-ends at D2's anode — there is no physical connector/pads for the board-to-board interface. Need a connector carrying **+5 V, GND, and the mainboard SPI + IRQ** signals (see label list). This is the one real gap in the otherwise-clean power path.
- [ ] **LCD footprint fix** (done in lib, needs pulling into project): the symbol's Footprint field was corrected `L51.0` → `L56.0` in `kicad-shared/library/jlcpcb.kicad_sym` (it pointed at a non-existent name). Do **Tools → Update Symbols from Library**, then **Update PCB from Schematic (F8)** so the LCD footprint + 3D model appear. LCD part = `HS180S10B` (LCSC **C5329585**).

## NEXT: restructure schematic into a hierarchy

Root sheet is too full to bolt on more. Plan: make the **root a block diagram** with two sibling sub-sheets.

```
Root (block diagram)
 ├── RP2040   (move existing core here)
 └── UI       (new: encoders + LCD + WS2812)
```

Steps (KiCad has no "demote root" command — do it by cut/paste):
1. **Commit current state in git first** (checkpoint before the structural change).
2. On root: Place → Add Hierarchical Sheet → name `RP2040`, file `rp2040.kicad_sch`.
3. Select-all on root, **Shift-click to deselect the new box**, **Cut** (Ctrl+X).
4. Enter the `RP2040` box, **Paste** (Ctrl+V). Root is now just the box.
5. **PCB watch-out:** after the move, Tools → Update PCB from Schematic with **"Re-link footprints to schematic symbols based on their reference designators"** ticked (paths change when symbols move sheets).
6. Replace the no-connect flags on the GPIOs we're using with **hierarchical labels** (list below); import them as **sheet pins** on the RP2040 box.
7. Add the `UI` sub-sheet and wire the two boxes on the root. Power crosses via power symbols (global), not labels.

## GPIO pin map (PROVISIONAL — optimize during routing)

RP2040 has 30 GPIO (0–29), all free. Using **24**, 6 spare. Pin assignments are flexible (firmware-defined) — let routing drive final placement; only the hardware-SPI signals are lightly constrained, and QSPI/USB/crystal/power are fixed.

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

## Hierarchical labels to create on the RP2040 sheet (24)

Name nets by **function, not pin** so reassignment is painless. Direction is from the RP2040's view (documentation only — Bidirectional is fine if unsure).

**Encoders → UI (In):** `ENC1_A` `ENC1_B` `ENC1_SW` `ENC2_A` `ENC2_B` `ENC2_SW` `ENC3_A` `ENC3_B` `ENC3_SW` `ENC4_A` `ENC4_B` `ENC4_SW`

**WS2812 + LCD → UI (Out):** `WS2812_DIN` `LCD_SCK` `LCD_MOSI` `LCD_CS` `LCD_DC` `LCD_RST` `LCD_BL`

**Mainboard → connector** (assuming tile = SPI **slave**): `MB_SCK` (In) `MB_MOSI` (In) `MB_MISO` (Out) `MB_CS` (In) `MB_IRQ` (Out)

The `UI` sheet gets mirror copies of the 19 `ENC*`/`WS2812_DIN`/`LCD_*` names (opposite direction). The 5 `MB_*` labels become sheet pins wired to the mainboard connector on the root.

## Decisions / risks to settle

- [ ] **WS2812 level shifter:** if the string runs at 5 V, RP2040's 3.3 V data is marginal (V_IH ≈ 3.5 V). Add a single-gate **74AHCT1G125** buffer (powered from 5 V) on GPIO5's line. Lives in the UI sheet near the LEDs.
- [ ] **WS2812 power budget:** ~60 mA/pixel at full white. **How many LEDs?** Matters for the VSYS/5 V source and the USB-500 mA programming cap. (Power question, not pins.)
- [ ] **Mainboard SPI master/slave** — assumed tile = slave (IRQ = tile→mainboard). If master, flip the `MB_` SPI directions.
- [ ] **LCD backlight** — if hardwired on (not PWM), drop `LCD_BL` and reclaim GPIO7.
- [ ] Pin assignments are provisional — expect to shuffle for routing (RP2040 GPIOs are interchangeable; keep functional net names so it's painless).

## After that

- PCB layout. Watch: decoupling caps hugging power pins; crystal + load caps tight to U1.20/21; USB D± matched pair; NCP1117 tab on a generous +3V3 pour with thermal vias.
- Workflow reminder: export the netlist (`ui-tile-kicad.net`) and have Claude review after schematic changes.
- Possible later: factor the RP2040 core into a reusable design block (template already cloned to `kicad-shared/templates/rp2040-template/`).
