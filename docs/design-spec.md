# Hardware Design Specification: 2-Port-to-2-Port 40G/100G QSFP/QSFP28 Managed Media Converter Board

## 1. System Overview & Objective

* **Purpose**: Act as a standalone, externally powered, managed physical-layer bridge between two passive QSFP+/QSFP28 DAC cables (connected to OEM Mellanox ConnectX-5 NICs) and two active 40G/100G single-mode optical transceivers (supporting up to Power Class 7 / 5.0W for 40km–80km ER4/ZR4 modules).
* **Key Features**:
* Bypasses NIC PCIe slot power-budget limits by supplying isolated, high-current 3.3V power directly to the optical transceivers.
* Hardware `LPMode` override + I2C SFF-8636 high-power class unlocking (Power Class 4–7).
* Per-port over-current / short-circuit isolation (PPTC / eFuse) so a faulty transceiver does not brown out the board or adjacent ports.
* Integrated **ESP32-S3-WROOM-1** controller (native USB-C for flashing/console) providing a real-time WebUI (DDM optical power, temperature, voltage telemetry), automated 4-pin PWM fan control (with MOSFET full-off capability), and per-port status LEDs.
* Pure passive high-speed differential pass-through (no Retimer/Redriver required; relies on NIC SerDes CTLE/DFE and optical module internal CDR).



---

## 2. System Block Diagram

```text
[ Host NIC (CX5) ]
       │  (2x 0.5m Passive QSFP+/QSFP28 DAC Cables)
       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  DAC-Side Ports (2x QSFP28 Receptacles: DAC_P1, DAC_P2)                  │
│   ├─ High-Speed Differential Pairs (8x TX/RX pairs total)               │
│   │   └─► 16x 0402 100nF High-Freq AC-Coupling Caps (DC Isolation)      │
│   ├─ ModPrsL (Pin 27) ──► ESP32 GPIOs (Presence Detection)              │
│   └─ I2C (SCL/SDA)    ──► Unconnected / Floating                        │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │ (100Ω Differential Stripline/Microstrip)
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Optical-Side Ports (2x QSFP28 Receptacles + Heatsinks: OPT_P1, OPT_P2)  │
│   ├─ High-Speed Differential Pairs (Connected to AC-Coupling Caps)      │
│   ├─ LPMode (Pin 31)  ──► Hard-tied to GND (Forces High-Power Mode)     │
│   ├─ ModPrsL (Pin 27) ──► ESP32 GPIOs (Presence Detection)              │
│   ├─ VCC (3.3V)       ──► Per-Port 2.5A PPTC/eFuse + Bulk Decoupling    │
│   └─ I2C (SCL/SDA)    ──► Direct to ESP32 Hardware I2C0 & I2C1          │
└─────────────────────────────────────────────────────────────────────────┘
                                    ▲
                                    │
┌───────────────────────────────────┴─────────────────────────────────────┐
│  Power & Management Subsystem                                           │
│   ├─ Input: 12V DC Barrel Jack (12V 3A / 36W)                           │
│   ├─ Buck Converter: 12V -> 3.3V @ 6A+ (TI TPS56C215, 12A, <30mV ripple)│
│   ├─ MCU: ESP32-S3-WROOM-1 (Wi-Fi WebUI, AsyncWebServer + WebSockets)   │
│   ├─ USB-C (data only): native USB-Serial/JTAG for flashing + console   │
│   ├─ Status LEDs: 4x Independent 0603 LEDs (1 per port) via ESP32 GPIOs │
│   └─ Cooling: 12V 4-Pin PWM Fan (25kHz PWM + TACH + Low-Side MOSFET)    │
└─────────────────────────────────────────────────────────────────────────┘

```

---

## 3. Core Subsystem Specifications

### 3.1 High-Speed Signal Path (40G & 100G Compatible)

* **Topology**: Point-to-point passive pass-through between `DAC_P1 <-> OPT_P1` and `DAC_P2 <-> OPT_P2`.
* **Lane Mapping (Per Port)**:
* `DAC_TX[1..4]_P/N` $\rightarrow$ `0402 100nF Cap` $\rightarrow$ `OPT_RX[1..4]_P/N`
* `OPT_TX[1..4]_P/N` $\rightarrow$ `0402 100nF Cap` $\rightarrow$ `DAC_RX[1..4]_P/N`


* **AC-Coupling Capacitors**: 16x `0402` `100nF (0.1µF)` high-frequency MLCCs (e.g., Murata GRM155R71C104KA88D or GJM series). Place symmetrically near the receiving pins; use anti-pad cutouts on reference planes directly beneath the 0402 pads to minimize impedance discontinuity.
* **Signal Integrity Rules**:
* **Impedance**: $100\Omega \pm 10\%$ differential impedance.
* **Trace Length**: Keep total differential trace length between DAC and Optical receptacles $\le 25\text{mm}$ (ideal $< 20\text{mm}$).
* **Intra-Pair Skew**: Length-match `P` and `N` within $\le 2\text{mil}$ ($0.05\text{mm}$).
* **Layer Routing**: Route exclusively on the Top Layer (`L1`) with an unbroken Ground Plane (`L2`) directly underneath. Avoid vias on high-speed pairs.
* **Via Fencing**: Surround all differential pairs with stitched GND via fences spaced every $1.2\text{mm} - 1.5\text{mm}$.



### 3.2 Power Distribution & Per-Port Protection (Up to 80km ZR4 Support)

* **Power Budget**:
* 2x 5.0W Optical Transceivers (Power Class 7, e.g., 40km ER4 / 80km ZR4) = $10\text{W}$ ($3.03\text{A}$ @ 3.3V).
* ESP32 + Peripherals = $\sim 1.5\text{W}$ ($0.45\text{A}$ @ 3.3V).
* 12V PWM Fan = $\sim 2.4\text{W}$ ($0.2\text{A}$ @ 12V).
* **Design Target**: 3.3V rail rated for **6A continuous**; 12V DC input rated for **12V / 3A (36W)**.


* **3.3V Buck Regulator**:
* Use synchronous buck converter **TI TPS56C215** (12A, LCSC C473372; MP8759 not stocked at LCSC, TPS5450 is only 5A non-synchronous) at 400kHz FCCM with a 2.2µH / Isat ≥16A shielded inductor.
* Input protection: AO4407A P-MOSFET reverse-polarity protection + SMAJ12A TVS (clamp must stay below the TPS56C215's 20V abs-max VIN).
* Output ripple must be $\le 30\text{mV}_{p-p}$ to prevent APD receiver sensitivity degradation and TEC startup brownouts.
* Route 3.3V power using wide copper pours ($\ge 2.5\text{mm}$ / $100\text{mil}$) and $\ge 4$ parallel stitching vias per transition.


* **Per-Port Short-Circuit Isolation & Decoupling**:
* Place an independent **2.5A Hold / 4.0A Trip 1206 PPTC Resettable Fuse** (or **TPS2595 / SY6280 eFuse** set to 2.5A limit) in series with the 3.3V supply of each optical port (`OPT_P1`, `OPT_P2`).
* **Local Decoupling (Per Optical Port)**: `1x 47µF X5R` (bulk TEC inrush) + `1x 10µF 0805` + `2x 0.1µF 0402` placed immediately adjacent to `VccTx`, `VccRx`, and `Vcc1` pins.



### 3.3 QSFP28 Control Pins & Hardware Strapping

* **Optical Ports (`OPT_P1`, `OPT_P2`)**:
* `Pin 31 (LPMode)`: **Tie directly to GND** (hardware override to allow >1.5W operation).
* `Pin 9 (ResetL)`: Pull up to 3.3V via $4.7\text{k}\Omega$ resistor (or connect to ESP32 GPIO for hardware reset capability).
* `Pin 8 (ModSelL)`: **Tie directly to GND** (since each port uses a dedicated ESP32 hardware I2C bus, the module is always selected).
* `Pin 27 (ModPrsL)`: Connect to ESP32 GPIO with $10\text{k}\Omega$ pull-up to 3.3V (reads `LOW` when module is inserted).
* `Pin 11 (SCL)` & `Pin 12 (SDA)`: Pull up to 3.3V via $4.7\text{k}\Omega$ resistors; connect directly to ESP32 hardware I2C controllers.


* **DAC Ports (`DAC_P1`, `DAC_P2`)**:
* `Pin 27 (ModPrsL)`: Connect to ESP32 GPIO with $10\text{k}\Omega$ pull-up to 3.3V (presence detection).
* `SCL`, `SDA`, `LPMode`, `ModSelL`, `ResetL`: Leave unconnected / default pull-ups (passive DAC cables do not require power or I2C configuration from this end).



---

## 4. ESP32-S3-WROOM-1 Pinout Allocation

Module: **ESP32-S3-WROOM-1-N8R2** (8MB quad flash, 2MB quad PSRAM). The ESP32-S3 has **two hardware I2C controllers (`I2C0` and `I2C1`)** routable to any GPIO via the GPIO matrix; they connect to `OPT_P1` and `OPT_P2` respectively, so no PCA9548A mux is needed and both optical I2C buses stay physically isolated.

**Pins deliberately left unused** (so the map works on any WROOM-1 variant):
* GPIO 26–32: internal SPI flash/PSRAM (not bonded out on WROOM-1).
* GPIO 33–37: used by octal PSRAM on `-N8R8`/`-N16R8` variants; avoided so the module can be swapped.
* GPIO 47/48: 1.8V I/O on the `-xxV` (1.8V VDD_SPI) variants; avoided.
* GPIO 0, 3, 45, 46: strapping pins. GPIO0 = BOOT button only; GPIO 3/45/46 left floating (internal defaults).
* GPIO 19/20: native USB D-/D+ (reserved for USB-C).
* GPIO 43/44: UART0 TX/RX (exposed on test pads as a fallback console).

| Subsystem | Signal Name | ESP32-S3 Pin | Direction | Circuit / Notes |
| --- | --- | --- | --- | --- |
| **Optical Port 1 I2C** | `OPT1_SDA` | **GPIO 8** | I/O | `I2C0` SDA + $4.7\text{k}\Omega$ pull-up to 3.3V |
| **Optical Port 1 I2C** | `OPT1_SCL` | **GPIO 9** | Output | `I2C0` SCL + $4.7\text{k}\Omega$ pull-up to 3.3V |
| **Optical Port 2 I2C** | `OPT2_SDA` | **GPIO 10** | I/O | `I2C1` SDA + $4.7\text{k}\Omega$ pull-up to 3.3V |
| **Optical Port 2 I2C** | `OPT2_SCL` | **GPIO 11** | Output | `I2C1` SCL + $4.7\text{k}\Omega$ pull-up to 3.3V |
| **Presence Detect** | `DAC1_PRSL` | **GPIO 4** | Input | $10\text{k}\Omega$ pull-up to 3.3V (`LOW` = inserted) |
| **Presence Detect** | `DAC2_PRSL` | **GPIO 5** | Input | $10\text{k}\Omega$ pull-up to 3.3V (`LOW` = inserted) |
| **Presence Detect** | `OPT1_PRSL` | **GPIO 6** | Input | $10\text{k}\Omega$ pull-up to 3.3V (`LOW` = inserted) |
| **Presence Detect** | `OPT2_PRSL` | **GPIO 7** | Input | $10\text{k}\Omega$ pull-up to 3.3V (`LOW` = inserted) |
| **Module Interrupt** | `OPT1_INTL` | **GPIO 17** | Input | QSFP Pin 28 (open-drain); $10\text{k}\Omega$ pull-up to 3.3V |
| **Module Interrupt** | `OPT2_INTL` | **GPIO 18** | Input | QSFP Pin 28 (open-drain); $10\text{k}\Omega$ pull-up to 3.3V |
| **Module Reset** | `OPT1_RSTL` | **GPIO 15** | Output | QSFP Pin 9; $4.7\text{k}\Omega$ pull-up keeps module out of reset while ESP32 boots |
| **Module Reset** | `OPT2_RSTL` | **GPIO 16** | Output | QSFP Pin 9; $4.7\text{k}\Omega$ pull-up keeps module out of reset while ESP32 boots |
| **Port Status LED** | `LED_DAC1` | **GPIO 38** | Output | Series $1\text{k}\Omega$ resistor + 0603 Blue LED to GND |
| **Port Status LED** | `LED_DAC2` | **GPIO 39** | Output | Series $1\text{k}\Omega$ resistor + 0603 Blue LED to GND |
| **Port Status LED** | `LED_OPT1` | **GPIO 40** | Output | Series $1\text{k}\Omega$ resistor + 0603 Green LED to GND |
| **Port Status LED** | `LED_OPT2` | **GPIO 41** | Output | Series $1\text{k}\Omega$ resistor + 0603 Green LED to GND |
| **4-Pin Fan Control** | `FAN_PWM` | **GPIO 12** | Output | 25kHz LEDC PWM to fan PWM wire (fan has internal pull-up; drive via open-drain or series $1\text{k}\Omega$) |
| **4-Pin Fan Control** | `FAN_PWR_EN` | **GPIO 13** | Output | Drives AO3400A low-side N-MOSFET on fan GND; $100\text{k}\Omega$ gate pull-down |
| **4-Pin Fan Control** | `FAN_TACH` | **GPIO 14** | Input | $10\text{k}\Omega$ pull-up to 3.3V; counted with PCNT (2 pulses/rev) |
| **Input Monitor** | `VIN_SENSE` | **GPIO 1** | Analog | ADC1_CH0; 12V via $100\text{k}\Omega$ / $10\text{k}\Omega$ divider + 100nF |
| **USB** | `USB_D-` / `USB_D+` | **GPIO 19 / 20** | I/O | USB-C receptacle (data only), USBLC6-2SC6 ESD, $5.1\text{k}\Omega$ CC1/CC2 pull-downs |
| **Boot / Reset** | `BOOT` / `EN` | **GPIO 0 / EN** | Input | Tactile buttons to GND; EN has $10\text{k}\Omega$ pull-up + $1\mu\text{F}$ to GND |
| **Debug UART** | `U0TXD` / `U0RXD` | **GPIO 43 / 44** | I/O | Test pads only (fallback console) |

*(Note: Initial flashing and console use the native USB-Serial/JTAG over USB-C — no external USB-UART bridge needed; subsequent updates via OTA. The USB-C port is data-only: VBUS is not connected to the 3.3V rail, so the 12V input must be present while flashing. Spare GPIOs remaining: 2, 21, 42.)*

---

## 5. PCB Manufacturing & Stackup Requirements (JLCPCB)

* **Board Dimensions**: Target **~80mm × 95mm** (fits comfortably within JLCPCB's low-cost $100\text{mm} \times 100\text{mm}$ tier).
* **Layer Count**: **4-Layer PCB** (`JLC04161H` impedance-controlled stackup).
* `L1 (Top)`: High-speed 100Ω differential pairs, components, short control traces.
* `L2 (Inner 1)`: Solid, uninterrupted **GND reference plane** (critical for 100Ω impedance).
* `L3 (Inner 2)`: **3.3V Power Plane** + low-speed I2C/GPIO routing.
* `L4 (Bottom)`: Secondary **GND plane** + thermal relief pours.


* **Board Thickness**: `1.6mm` (standard for receptacle-only boards; all 4 ports use female QSFP28 cages/connectors).
* **Surface Finish**: **ENIG (Immersion Gold)** for flat pads and optimal high-frequency skin-effect performance.
* **Impedance Control**: Enable **100Ω Differential Impedance** (`L1` referenced to `L2`).

---

## 6. Firmware & WebUI Functional Requirements (for Reference)

1. **SFF-8636 High-Power Unlock**: Upon detecting `OPTx_PRSL == LOW`, wait 100ms, then write `0x04` (or appropriate Power Class 5–7 enable bits) to **Page 00h, Byte 93 (`Power Set`)** at I2C address `0x50` to unlock full TEC/laser power for 40km/80km transceivers.
2. **DDM Telemetry Polling**: Read **Page 00h, Bytes 22–57** every 1000ms to parse Module Temperature, Supply Voltage (Vcc), TX Bias Current, TX Optical Power (dBm), and RX Optical Power (dBm).
3. **LED State Machine**:
* `OFF`: Port empty (`ModPrsL == HIGH`).
* `BLINKING`: Module inserted, initializing or RX optical power below warning threshold (e.g., $< -12\text{dBm}$) / temperature $> 65^\circ\text{C}$.
* `SOLID ON`: Link-ready, power and thermal telemetry within nominal range.


4. **Closed-Loop Thermal Control**: Map maximum module temperature to 25kHz PWM duty cycle; assert `FAN_PWR_EN = LOW` to completely cut fan power via MOSFET when both modules are absent or below $40^\circ\text{C}$.
5. **WebUI**: Host a Gzip-compressed Single-Page Application in LittleFS served via `ESPAsyncWebServer` with WebSocket telemetry streaming and `/update` OTA firmware upload.

---

## 7. Bill of Materials (BOM) Highlights

* **QSFP28 Connectors & Cages**:
* 2x QSFP28 38-Pin SMT Receptacle + Single-Port Cage **with High-Profile Heatsink & Clip** (for `OPT_P1`, `OPT_P2`).
* 2x QSFP28 38-Pin SMT Receptacle + Standard EMI Cage (for `DAC_P1`, `DAC_P2`).


* **MCU & Power**:
* 1x `ESP32-S3-WROOM-1-N8R2` (8MB Flash, 2MB PSRAM).
* 1x USB-C 16-pin receptacle (data only) + 1x `USBLC6-2SC6` ESD + 2x `5.1kΩ` CC resistors.
* 2x tactile switches (`BOOT`, `EN`).
* 1x `TPS56C215RNNR` Synchronous Buck Converter + `2.2µH` Shielded Inductor (≥16A saturation current).
* 1x `AO4407A` P-MOSFET + `BZT52C12` zener (reverse-polarity protection), 1x `SMAJ12A` TVS.
* 2x `1206` `2.5A` PPTC Resettable Fuses (or 2x `TPS2595` eFuse ICs).
* 1x `AO3400A` N-Channel MOSFET (for 12V fan low-side power switching).
* 1x `DC-005` 5.5×2.1mm Barrel Jack (12V input).


* **Passives & Indicators**:
* 16x `0402` `100nF` High-Frequency MLCCs (AC coupling).
* 2x `47µF` X5R MLCCs + 4x `10µF` `0805` MLCCs + 10x `0.1µF` `0402` MLCCs (decoupling).
* 2x `0603` Blue LEDs + 2x `0603` Green LEDs + 4x `0402` `1kΩ` Resistors.