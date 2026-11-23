# QSFP Booster

A 2-port QSFP28 media converter board. Each port bridges a passive QSFP+/QSFP28 DAC cable from a
host NIC to a 40G/100G optical transceiver. The board powers the optical modules from its own 12 V
supply (up to Power Class 7 / 5 W each), so long-reach modules work even when the NIC cannot power
them. An ESP32-S3 manages the modules over I2C and runs the fan and status LEDs.

![PCB top view](docs/images/pcb_top.png)

<details>
<summary>Bottom view (seen from below)</summary>

![PCB bottom view](docs/images/pcb_bottom.png)

</details>

> **Status:** first prototype, not yet built or tested. Use at your own risk.

## Features

- **2 × (DAC receptacle → optical receptacle)** QSFP28 pairs. High-speed lanes are passed straight
  through (DAC Rx → optical Tx and optical Rx → DAC Tx) on 100 Ω differential pairs, with no
  retimer and no AC-coupling capacitors (QSFP28 modules and the NIC AC-couple internally).
- **Module power:** 12 V → 3.3 V by a hand-fitted ZXDN10 (ZXDN10S1205SAW) DC-DC module, mounted
  upright on the board edge, set to 3.31 V. Its output goes through a 470 µF bulk capacitor to the inner
  3.3 V plane, and each module is fed through its own PPTC fuse and an inner-layer copper area next to its
  connector.
- **Management:** a plug-in ESP32-S3 DevKitC-1 style board (ESP32-S3-WROOM-1 with at least 8 MB quad flash:
  N8, N8R2, N8R8 or N16R8) in two 22-pin sockets. Each module has its own I2C pins plus presence, reset and
  interrupt lines, each through a 100 Ω series resistor at the socket. The ESP32 board is powered from
  the board's 3.3 V through a CH213K ideal diode (U2), so it can stay unmodified: its own USB-C ports work for
  flashing and the console with or without 12 V applied.
- **Cooling:** 4-pin PWM fan header with a high-side switch that can turn the fan fully off, plus
  tachometer input. The fan only runs when the ESP32 turns it on (off with no MCU or during reset).
- **Cages:** the same press-fit cage on all four ports, each with an exposed ground strip under its
  EMI fingers.
- **Indicators:** one LED per port and a power LED, spaced along the board edges.
- **Input protection:** reverse-polarity P-MOSFET and SMAJ12A surge suppressor.
- **Board:** 100 × 100 mm, 4 layers.

## Specifications

| Item | Value |
|---|---|
| Input | 12 V DC ± 5 % (11.4–12.6 V), 3 A, barrel jack (centre positive) |
| Module power | 3.3 V, up to 5 W per optical module (SFF-8679 Power Class 7); 10.5 W supported configuration (two 5 W optics + two passive DACs). The 3.3 V copper (U1 → C33 → inner plane) is laid out for 10 A, but the 3 A input limits the board to about 30 W in, and only the configuration above is supported |
| PCB | 100 × 100 mm, 4 layers, 1.6 mm, JLCPCB stack-up JLC04161H-3313 with impedance control |
| Differential pairs | 100 Ω, 0.13 mm trace / 0.15 mm gap on the outer layers |
| Vias | 0.3 mm drill / 0.45 mm pad |
| Mounting holes | 4 × 2.7 mm plated at (3.6, 9), (60.5, 9), (3.6, 91), (60.5, 91) mm from the board's lower-left corner (DAC cages on the y = 0 edge), 5 mm GND ring with 8 stitching vias (M2.5, head ≤ 4.7 mm); bonds board ground to a metal enclosure |

## Repository contents

| Path | What it is |
|---|---|
| `hardware/qsfp-booster.epro` | JLCEDA Pro (嘉立创EDA专业版) project in epro (V2) format: schematic and PCB (source of truth). KiCad 8+ can import it via File → Import → Non-KiCad Board File |
| `docs/design-spec.md` | Original design specification (the pin map above and the project file reflect the final board where they differ) |

Production files are attached to each [GitHub release](../../releases):

| File | What it is |
|---|---|
| `Gerber_qsfp-booster.zip` | Gerber and drill files |
| `BOM_qsfp-booster.xlsx` | Bill of materials with LCSC part numbers (JLCPCB assembly) |
| `CPL_qsfp-booster.xlsx` | Component placement (pick-and-place) file, top side |

## Ordering at JLCPCB

1. Download the production files from the latest release and upload `Gerber_qsfp-booster.zip`.
2. PCB options:
   - **Layers:** 4.
   - **Thickness:** 1.6 mm.
   - **Impedance control:** yes, stack-up **JLC04161H-3313**.
   - **Min via hole size:** **0.3 mm** (the free option).
3. Enable PCB assembly (standard PCBA, top side), then upload `BOM_qsfp-booster.xlsx` and
   `CPL_qsfp-booster.xlsx`.
   In the assembly preview, check the parts by their LCSC number rather than the CPL "Device" name: the LED
   resistors R1, R2, R45, R53 and R54 are **C25104** (330 Ω), the series resistors R3–R17 and R39–R43
   **C25076** (100 Ω), and the output-voltage resistors R51/R52 **C25104** (330 Ω) / **C25118** (47 Ω).
4. Hand-solder the parts that are not in the BOM:

   | Ref | Part | LCSC |
   |---|---|---|
   | J5 | DC-005 barrel jack (XKB DC-005-5A-2.0) | C381116 |
   | J7 | 4-pin fan header (Molex 47053-1000) | C240840 |
   | J6, J8 | 22-pin 2.54 mm female headers for the ESP32-S3 board | — |
   | U1 | ZXDN10 (ZXDN10S1205SAW) 12 V → 3.3 V module, upright, label facing the board edge | salvaged |
   | C32, C33 | 470 µF 25 V radial electrolytic, 8 × 15 mm, 3.5 mm hole pitch (C32 on 12 V, C33 on 3.3 V; mind the polarity). LCSC supplies this part with leads kinked out to 5 mm for tape: straighten them so the can sits flat, or use a straight-lead 3.5 mm-pitch equivalent | C2835696 |
   | J11–J14 | QSFP28 1×1 cages, press-fit (CND-tek QSFP28 2170703-2 1X1H), all four ports | C52677971 |

   Use this cage on every port. Heatsink cages such as TE 1888631-1 have their two rear press-fit
   pins 10.6 mm behind the connector alignment holes, while this board drills them at 12.79 mm, so
   they will not fit. The optical modules are cooled by the fan through the cage's top window; a
   clip-on QSFP heatsink can be added if needed.

To change the design, open `hardware/qsfp-booster.epro` in JLCEDA Pro (File → Open Project) and
regenerate the production files from there (File → Export).

## ESP32-S3 I/O map

The MCU is a plug-in **ESP32-S3 DevKitC-1 style board** with an ESP32-S3-WROOM-1 module and at least 8 MB of
quad flash (N8, N8R2, N8R8 or N16R8). The board leaves IO33–IO37 free, so octal-PSRAM "R8" modules work
too. The pin map is verified against the YD-ESP32-S3 and Espressif's DevKitC-1
v1.0/v1.1. Each port uses five adjacent pins of one socket (DAC1's five are in reverse order).

| Port | SDA | SCL | ModPrsL | IntL | ResetL | LED | Socket pins |
|---|---|---|---|---|---|---|---|
| DAC1 | IO3 | IO18 | IO8 | IO17 | IO16 | IO21 | J6.13–9 |
| DAC2 | IO10 | IO12 | IO11 | IO13 | IO14 | IO47 | J6.16–20 |
| OPT1 | IO42 | IO40 | IO41 | IO39 | IO38 | IO1 | J8.6–10 |
| OPT2 | IO4 | IO6 | IO5 | IO7 | IO15 | IO2 | J6.4–8 |

| Function | GPIO |
|---|---|
| Fan PWM / power enable / tach | IO45 / IO46 / IO9 |
| Unused (dev-board functions) | IO0 BOOT button, IO19/IO20 native USB, IO43/IO44 UART console, IO48 RGB LED, IO26–IO37 flash/PSRAM |

Three of the used pins are boot straps, chosen so that nothing on this board lights or switches during reset or
flashing (IO0 is left free for the BOOT button):

- **IO45 (fan PWM):** only drives Q3's gate. Its internal pull-down keeps it low at reset (3.3 V flash), and
  the fan has no power then anyway.
- **IO46 (fan enable):** held low by R46 (4.7 kΩ), so the fan stays off during reset and download.
- **IO3 (DAC1 SDA):** only a JTAG-select strap; leave the JTAG eFuses at their defaults.

On Espressif's DevKitC-1 **v1.1** the on-board RGB LED's data input is wired to IO38 (OPT1 ResetL). A reset pulse is
not a colour frame, so it normally stays dark, but remove that LED (or its 0 Ω feed) if it must never light. The
YD-ESP32-S3 and DevKitC-1 v1.0 have it on the unused IO48.

Each port has its own SDA/SCL pair and status lines, every one with a 4.7 kΩ pull-up to the board's 3.3 V on the module
side and a 100 Ω series resistor next to the ESP32 socket. ModSelL and LPMode are tied low.

## Power-up and service rules

- **ESP32 board power:** use the ESP32 board unmodified (keep its regulator). Its 3V3 pins (J6.1/J6.2) are fed
  from the board's 3.3 V rail through U2, a CH213K ideal diode (about 0.1–0.2 V drop at 0.5 A, so the ESP32 sees
  about 3.15 V). When USB is plugged in, the ESP32 board's own regulator can power it too. U2 blocks current back
  into the module rail, so USB alone runs only the ESP32, never the modules. With 12 V applied and USB plugged in,
  both feeds meet at the ESP32 board's 3V3 pins, and the higher one supplies the ESP32. This relies on the ESP32
  board's LDO not sinking current into its output, which typical LDOs do not, but Espressif lists USB and 3V3-pin
  power as mutually exclusive: check the regulator on your board, and on the first board measure J6.1 with USB and
  12 V both applied. With neither USB nor 12 V the ESP32 is off. The
  board's 5V pin (J6.21) is not connected. Do not remove the ESP32 board's regulator: USB-only flashing needs it.
- **First power-up of every board:** with no ESP32 board and no modules fitted, apply 12 V and measure the
  3.3 V rail across C33. It must read about 3.25–3.35 V before anything else is plugged in. The ZXDN10's
  output is set by R51 + R52 (377 Ω, Vout = 0.591 × (3580 + R) / (330 + R) ≈ 3.31 V); a short there gives
  6.4 V, which would destroy the optics and the ESP32. Do not probe the ADJ resistors with power on. With
  modules running, also check the ripple across C33 with a scope: the shared 3.3 V plane has only C33 and the
  ZXDN10's own output capacitors as bulk; every other ceramic sits behind the port fuses.
- **12 V jack:** centre positive, a regulated 12 V ± 5 % (11.4–12.6 V), 3 A. Do not go higher: the SMAJ12A
  surge suppressor across the input is rated for 12.0 V continuous (stand-off) and breaks down from 13.3 V, so
  a supply at the top of the range already sits slightly above its stand-off rating. Prefer a supply close to
  12.0 V, and check D2 stays cold on the first board. The fan output's clamp D8 is the same SMAJ12A (forward-biased
  only for the fan's switch-off spike), so check it stays cold with the fan running too. The reverse-polarity
  P-MOSFET Q1 (AO3407A) is rated ±20 V gate-source, so it needs no gate clamp across that input range.
  Fit C32 before the first power-up: plugging into a live adapter makes the cable's inductance ring against the
  input ceramics, and C32 damps that spike so D2 only has to clip what is left. Where possible, plug the DC jack
  in first and then switch the adapter on at the wall, so the 12 V ramps up instead of stepping.
- **Fan (J7):** GND, 12V, TACH, PWM (silkscreened). The fan's 12 V is switched on the high side by Q6
  (AO3407A), which Q2 (AO3400A) turns on when IO46 goes high, so the fan's ground is always connected. The fan is off unless the ESP32 is running and turns it on. Use a fan drawing
  at most 0.6 A (start-up included): its 12 V feed goes through a 1 A-hold PPTC fuse (F5) that derates when warm.
- **Supported module configuration:** passive DAC cables on the DAC ports (J1/J2) and optics up to 5 W (power
  class 7) on the optical ports (J3/J4), 10.5 W in all. Nothing on the board limits a module's draw, so stay within
  this configuration.

## Usage notes

- The NIC sees a passive copper cable, so it will try copper auto-negotiation and link training,
  which the optical module never answers. Force the link instead, for example
  `ethtool -s <if> autoneg off speed 100000`, and set the same FEC mode on both ends.
- Keep the DAC cables short (about 0.5 m). The NIC drives the cable with copper equalisation, and a
  little cable loss brings the signal into the range the optical module expects.

## Known limitations

Independent reviews of the production files found the following. None of them stops the board from
working, but they are worth knowing before you rely on it:

- **Salvaged power module:** the ZXDN10 has no published ratings and no output overvoltage protection on
  this board; qualify each module (see the first power-up rule) and keep the total load within the supported
  configuration.
- **Port fault isolation:** each module is fed through a PPTC fuse (2.5 A hold, 5 A trip), not an
  eFuse. A shorted module can pull the shared 3.3 V rail down for all ports until the fault is removed: how fast
  (or whether) its fuse trips depends on the fault current and on how the salvaged ZXDN10 behaves in overload, so
  automatic clearing is not guaranteed. The fuse's hold current also drops with temperature (about 1.5 A at 70 °C,
  close to a 5 W module's 1.52 A), so keep the board well ventilated with high-power optics.
- **Module supply filtering:** each port's fuse output runs on the top layer down a column of bulk and bypass
  capacitors (47 µF, 10 µF, 3 × 100 nF) at the board edge (each fuse input takes the same 3 vias from the
  main 3.3 V plane), and drops through 5 vias spread along that capacitor column into
  an L-shaped copper area on inner layer 3 that runs along the cage edge to the connector's VCC vias. There is no
  direct path from the fuse to the connector, so the supply current flows past the capacitors instead of bypassing
  them. The cage covers the area right next to the
  connector, so the capacitors cannot sit closer. There is no SFF-8679 style per-pin filter. Inner layer 3 is
  GND, not 3.3 V, under the 25G lane transitions, so the bottom-layer lane segments keep a ground reference.
- **High-speed channel:** every 25G lane goes top → bottom → top (two vias) between the connectors, and the CND-tek
  connector is sold as QSFP+. Expect to qualify each link with the real DAC and optics.

## License

[MIT](LICENSE)
