# Project Master Summary: FM Pocket Radio

## 1. Project Overview
The FM Pocket Radio project is a multi-phase engineering effort to design and build a portable, product-grade FM receiver. The goal was to move from a basic functional prototype to a professional, PCB-based handheld device with integrated charging, a custom enclosure, and a user-friendly interface.

## 2. Design Evolution
### Phase 1: Proof of Concept (Dot-Board)
- **Core:** Arduino UNO + RDA5807M (RRD-102) + TDA2822M amplifier.
- **Interface:** Two push-buttons for seeking, hardware B100K potentiometer for volume.
- **Power:** USB 5V via Arduino.
- **Outcome:** Verified basic FM reception and audio amplification on a perfboard.

### Phase 2: Feature Expansion (ESP32 Edition)
- **Core:** ESP32 Dev Board + RDA5807M + TDA2822M (BTL Mode).
- **Interface:** SH1106 OLED display, two S141 rotary encoders (tuning and volume).
- **Power:** 18650 Li-ion battery, TP4056 charger, MT3608 boost converter.
- **Outcome:** Introduced digital UI, battery power, and higher audio efficiency via BTL amplification.

### Phase 3: Productization (Custom PCB)
- **Core:** ATmega328P (8MHz, 3.3V) + RDA5807M + PAM8403 Class-D Amp.
- **Interface:** SSD1306 OLED, rotary encoders, 3.5mm headphone jack with auto-detect.
- **Power:** Integrated TP4056 charging and DW01A/FS8205A battery protection on a custom 2-layer PCB.
- **Connectivity:** Single USB-C port for both charging and firmware updates via CH340C.
- **Outcome:** Transition to a compact, professional form factor ($\sim$70x50mm PCB) with refined audio and power paths.

## 3. Technical Specifications (Phase 3 Target)
### Hardware Stack
- **MCU:** ATmega328P-PU (DIP-28), running at 8MHz / 3.3V.
- **Radio:** RDA5807M (RRD-102 V2.0 module), I2C interface, 87.5--108.0 MHz.
- **Amplifier:** PAM8403 Class-D, configured for Mono BTL output into 8$\Omega$ speaker.
- **Display:** 0.96" SSD1306 OLED (128x64), I2C interface.
- **USB Interface:** CH340C USB-to-UART bridge for programming.
- **Power:** 3.7V LiPo battery, TP4056 CC/CV charger, HT7333 Low-Dropout (LDO) regulator.

### Power Architecture
- **Charging Path:** USB-C VBUS $\to$ TP4056 $\to$ LiPo Battery.
- **Protection Path:** LiPo $\to$ DW01A/FS8205A (Low-side protection) $\to$ Latching Switch $\to$ VBAT Bus.
- **Regulation:** VBAT $\to$ HT7333 LDO $\to$ +3.3V Rail (powers MCU, Radio, OLED).
- **Audio Power:** VBAT $\to$ PAM8403 VDD (Direct supply for maximum efficiency).

## 4. Key Design Implementations
### Audio Signal Chain
- **Mixing:** LOUT and ROUT from RDA5807M are summed via 10k$\Omega$ resistors to a mono node (AF_MONO).
- **Coupling:** A 1$\mu$F capacitor (C5) blocks DC and feeds the PAM8403 INL pin.
- **Amplification:** PAM8403 drives the speaker in Bridge-Tied Load (BTL) mode for increased power.
- **Headphone Path:** Stereo L/R outputs bypass the amp and go directly to a 3.5mm jack via coupling caps.
- **Auto-Mute:** A detect pin on the jack triggers the MCU (D8) to pull the PAM8403 \textasciitilde{}SHDN pin LOW, muting the speaker.

### Control & User Interface
- **Tuning:** Right rotary encoder adjusts frequency in 0.1 MHz steps; push-button triggers auto-seek UP.
- **Volume:** Left rotary encoder adjusts RDA5807M internal volume register (0--15); push-button triggers auto-seek DOWN.
- **Display:** OLED shows current frequency, station name (from database), volume bar, signal strength, and battery percentage.

### Programming & Maintenance
- **USB-C Integration:** Single port handles VBUS for charging and D+/D- for the CH340C.
- **Auto-Reset:** CH340C DTR pin connected to ATmega RESET via 100nF capacitor for seamless Arduino IDE uploads.
- **ISP Header:** 2x3 pin header provided for initial bootloader burning.

## 5. Critical Audit & Bug Log
*A comprehensive audit of the Phase 3 guide revealed several "hard blockers" that must be fixed before PCB fabrication.*

### Electrical Blockers
1. **PAM8403 Pinout Error:** The guide incorrectly mapped pins for INL, INR, SHDN, and VCC. Must be corrected to match the datasheet (e.g., INL is pin 7, not 8).
2. **TP4056 Pinout Error:** The BAT pin was listed as pin 3 (GND), whereas it is actually pin 5.
3. **Battery Protection Topology:** The guide described a high-side protection circuit using N-channel MOSFETs (FS8205A). These must be moved to the low-side (GND path) to function.
4. **CH340C V3 Reference:** Pin 4 (V3) was missing. For 3.3V operation, V3 must be tied to VCC to ensure USB enumeration.
5. **Charging LED Polarity:** LEDs were wired as source-outputs. Since TP4056 pins are open-drain, LEDs must be connected to VBUS and sink to the IC pins.

### Design Inconsistencies
- **Audio Path Contradiction:** Various sections conflicted on whether the speaker path was mono-mixed or stereo-split.
- **Footprint Mismatch:** Conflict between SOIC-16 (3.9mm) and SOP-16 (4.4mm) footprints for the PAM8403 and CH340C.
- **ADC Impedance:** The 100k$\Omega$ battery divider creates a 50k$\Omega$ source impedance, exceeding the ATmega's 10k$\Omega$ limit. Fix: add 100nF cap at A0.

## 6. Validation & Testing Strategy
### Stage 1: Power & Basic Health
- Verify +3.3V and +5V rails are not shorted to GND.
- Confirm TP4056 charging LEDs light up upon USB connection.
- Measure battery voltage at the ADC pin.

### Stage 2: Communication & Firmware
- Ensure CH340C enumerates as a COM port (requires radio to be ON).
- Upload test sketch and verify I2C communication with RDA5807M and SSD1306.
- Test rotary encoders for correct clockwise/counter-clockwise direction.

### Stage 3: Audio & RF
- Verify "hum" on amplifier input to confirm amp is alive.
- Perform frequency seeks and verify audio reception.
- Test headphone insertion to confirm speaker mute logic.

## 7. Summary BOM (Phase 3)
| Component | Quantity | Key Spec |
|---|---|---|
| ATmega328P-PU | 1 | 8MHz, 3.3V |
| RDA5807M Module | 1 | RRD-102 V2.0 |
| PAM8403 | 1 | SOP-16, Class-D |
| SSD1306 OLED | 1 | 0.96", I2C |
| TP4056 | 1 | SOP-8, Charger |
| CH340C | 1 | SOP-16, USB-UART |
| DW01A + FS8205A | 1 set | Battery Protection |
| HT7333 | 1 | SOT-89, 3.3V LDO |
| LiPo Battery | 1 | 3.7V, 800--1000mAh |
| Encoders | 2 | S141 Rotary |
| Speaker | 1 | 8$\Omega$, 0.5--1W |
| 3.5mm Jack | 1 | PJ-320A (NO Detect) |
| USB-C Port | 1 | 16-pin SMD |
| PCB | 1 | 2-layer, 70x50mm |
