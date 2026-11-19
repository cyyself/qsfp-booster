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
- **Module power:** 12 V → 3.3 V buck (TI TPS56C215, ≥ 9.8 A current limit) feeding each module
  through its own PPTC fuse.
- **Management:** ESP32-S3-WROOM-1-N16R8, with a TCA9548A I2C switch giving it a separate bus to
  each of the four modules. Presence, reset and interrupt lines from every module go to the ESP32.
- **USB:**
  - USB-C to the ESP32 native USB (flashing, USB-Serial/JTAG console).
  - USB-C through a CH340C to UART0, with automatic download/reset.
- **Cooling:** 4-pin PWM fan header with a MOSFET switch that can turn the fan fully off, plus
  tachometer input.
- **Cages:** the same press-fit cage on all four ports, each with an exposed ground strip under its
  EMI fingers.
- **Indicators:** one LED per port, a power LED and an MCU status LED, all spaced along the board
  edges.
- **Input protection:** reverse-polarity P-MOSFET and SMAJ12A surge suppressor.
- **Board:** 100 × 100 mm, 4 layers.

## Specifications

| Item | Value |
|---|---|
| Input | 12 V DC (11–13 V), 3 A, barrel jack (centre positive) |
| Module power | 3.3 V, up to 5 W per module (SFF-8679 Power Class 7) |
| PCB | 100 × 100 mm, 4 layers, 1.6 mm, JLCPCB stack-up JLC04161H-3313 with impedance control |
| Differential pairs | 100 Ω, 0.13 mm trace / 0.15 mm gap on the outer layers |
| Vias | 0.3 mm drill / 0.45 mm pad |

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
4. Hand-solder the parts that are not in the BOM:

   | Ref | Part | LCSC |
   |---|---|---|
   | J5 | DC-005 barrel jack (XKB DC-005-5A-2.0) | C381116 |
   | J7 | 4-pin fan header (Molex 47053-1000) | C240840 |
   | J11–J14 | QSFP28 1×1 cages, press-fit (CND-tek QSFP28 2170703-2 1X1H), all four ports | C52677971 |

   Use this cage on every port. Heatsink cages such as TE 1888631-1 have their two rear press-fit
   pins 10.6 mm behind the connector alignment holes, while this board drills them at 12.79 mm, so
   they will not fit. The optical modules are cooled by the fan through the cage's top window; a
   clip-on QSFP heatsink can be added if needed.

To change the design, open `hardware/qsfp-booster.epro` in JLCEDA Pro (File → Open Project) and
regenerate the production files from there (File → Export).

## ESP32-S3 I/O map

| Function | GPIO |
|---|---|
| I2C SDA / SCL / mux reset | IO8 / IO9 / IO10 |
| Module present (DAC1, DAC2, OPT1, OPT2) | IO4, IO5, IO6, IO7 |
| Module reset (DAC1, DAC2, OPT1, OPT2) | IO21, IO47, IO15, IO16 |
| Module interrupt (DAC1, DAC2, OPT1, OPT2) | IO48, IO42, IO17, IO18 |
| Port LEDs (DAC1, DAC2, OPT1, OPT2) | IO38, IO39, IO40, IO41 |
| MCU status LED | IO2 |
| Fan PWM / power enable / tach | IO12 / IO13 / IO14 |
| 12 V input sense (ADC1_CH0) | IO1 |
| Native USB D− / D+ | IO19 / IO20 |
| UART0 TX / RX (CH340C) | IO43 / IO44 |
| BOOT button (readable after boot) | IO0 |

The I2C switch (TCA9548A, address 0x70) uses these channels:

| Module | Channel |
|---|---|
| DAC1 | 2 |
| DAC2 | 3 |
| OPT1 | 4 |
| OPT2 | 5 |

ModSelL and LPMode are tied low on every module, so each module is always selected and allowed
high power. High-power classes are still enabled through SFF-8636 over I2C.

## Usage notes

- The NIC sees a passive copper cable, so it will try copper auto-negotiation and link training,
  which the optical module never answers. Force the link instead, for example
  `ethtool -s <if> autoneg off speed 100000`, and set the same FEC mode on both ends.
- Keep the DAC cables short (about 0.5 m). The NIC drives the cable with copper equalisation, and a
  little cable loss brings the signal into the range the optical module expects.

## Known limitations

An independent review of the production files found the following. None of them stops the board
from working, but they are worth knowing before you rely on it:

- **Port fault isolation:** each module is fed through a PPTC fuse (2.5 A hold, 5 A trip), not an
  eFuse. A shorted module can pull the shared 3.3 V rail down for a moment until its fuse trips.
  The fuse's hold current also drops with temperature (about 1.5 A at 70 °C, close to a 5 W
  module's 1.52 A), so keep the board well ventilated with high-power optics.
- **Module supply filtering:** the per-port bulk and bypass capacitors sit about 14 mm from the
  connector (the cage covers the area right next to it), and there is no SFF-8679 style per-pin
  filter.
- **High-speed channel:** every 25G lane changes layer once between the connectors, and the CND-tek
  connector is sold as QSFP+. Expect to qualify each link with the real DAC and optics.
- **USB:** both USB ports are self-powered from the 12 V input and do not sense VBUS, so the board
  presents itself on D+ even when the host is off. Full-speed USB tolerates the loose D+/D− routing.

## License

[MIT](LICENSE)
