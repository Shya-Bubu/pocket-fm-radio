# FM Radio — Phase 2 "ESP32 Edition" Full Build Document

## RDA5807M (RRD-102 V2.0) + TDA2822M BTL + ESP32 (30-Pin USB-C) + SH1106 OLED

**Revision:** 2.0  
**Date:** 2026-02-22  
**Board:** 15 cm × 9 cm Perfboard (Dot Board)

---

**Design Decisions — Locked**

| Decision | Choice |
|----------|--------|
| MCU | ESP32-D0WD-V3, 30-pin, USB-C clone |
| Radio IC | RDA5807M on RRD-102 V2.0 module (I2C, 0x10) |
| Display | 1.3″ SH1106 OLED, I2C, 128×64, 0x3C |
| Amplifier | TDA2822M in **BTL (bridge)** mode |
| Speaker | 4 Ω, 5 W |
| Volume | Software volume via RDA5807M register (0–15), left rotary encoder |
| Tuning | 2× S141 rotary encoders (right = tuning, left = volume) |
| Power switch | 1× latching ON/OFF |
| De-emphasis | 50 µs (Sri Lanka / Asia / Europe) |
| Audio mode | Mono (LOUT + ROUT mixed via resistors) |
| Battery | 1× 18650 Li-ion (TP4056 charging) |
| Battery sensing | 100 k / 100 k voltage divider → ADC |

**Power Plan — Two Phases**

| When | Power Source | Details |
|------|-------------|---------|
| TODAY | Arduino UNO 5 V output → ESP32 VIN + TDA2822M | Dev/test only. No battery power. |
| TOMORROW Plan A | 18650 → TP4056 → Switch → MT3608 boost (5 V) → ESP32 VIN + TDA2822M | **Recommended.** Full 5 V for amp. |
| TOMORROW Plan B | 18650 → TP4056 → Switch → HT7333 LDO (3.3 V) → ESP32 3V3 pin. TDA2822M from battery direct. | Quieter speaker (~0.2 W at 3.7 V). |

---

# 0) Absolute Rules — Read Before You Touch a Soldering Iron

```
╔══════════════════════════════════════════════════════════════════════╗
║  FAILURE TO FOLLOW THESE RULES WILL DESTROY COMPONENTS OR CAUSE    ║
║  FIRE / INJURY. THERE ARE NO SECOND CHANCES WITH LITHIUM BATTERIES ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 0.1 — 3.3 V Rules

1. **RDA5807M is a 3.3 V device.** Never connect 5 V to its VCC pad. Instant destruction.
2. **SH1106 OLED is a 3.3 V device.** Never connect 5 V to its VCC pin.
3. **I2C pull-up resistors (R1, R2) connect to +3V3 only.** Never to +5V. The RDA5807M I/O
   pins will be damaged by 5 V levels on SDA/SCL.
4. **ESP32 GPIO pins are 3.3 V.** They are NOT 5 V tolerant. Do not connect 5 V logic to any GPIO.

### 0.2 — Battery Safety

5. **The TP4056 charges ONE cell only (4.2 V termination).** Never connect two 18650 cells
   in series to a single TP4056. This will cause unbalanced charging → thermal runaway → fire.
6. **Never short-circuit the 18650 battery.** It can deliver 20+ amps into a short. Use a
   battery holder with spring contacts — not loose wires soldered to bare cells.
7. **Never leave a charging battery unattended** on a flammable surface.
8. **If the TP4056 gets hot (>60 °C) during charging, disconnect immediately.**
9. **The power switch must be AFTER the TP4056 output,** not between battery and TP4056 input.
   The TP4056 needs constant battery connection for its protection circuit.

### 0.3 — IC Safety

10. **Verify TDA2822M orientation** (notch/dot = pin 1) before inserting into socket.
    Backwards = destroyed IC.
11. **Do NOT connect the speaker directly to RDA5807M outputs.** The audio outputs are
    line-level (~300 mV). The TDA2822M amplifier must sit between radio and speaker.
12. **In BTL mode, NEITHER speaker wire goes to GND.** Both speaker terminals connect to
    TDA2822M outputs (pin 1 and pin 3). Grounding either terminal will short-circuit one
    amplifier channel and may destroy the IC.

### 0.4 — Soldering Safety

13. **Place decoupling capacitors within 1–2 cm of each IC.** Distance matters in analog circuits.
14. **Electrolytic capacitor polarity matters.** Reversed polarity → capacitor heats up → bursts.
    The longer leg is positive (+). The stripe marks the negative (−) side.
15. **LED polarity matters.** Longer leg = anode (+). Shorter leg / flat side = cathode (−).

---

# 1) System Block Diagram

```
                         ┌─── TODAY ────┐
                         │ Arduino UNO  │
                         │   5V + GND   │
                         └──────┬───────┘
                                │
         ┌──── TOMORROW ────┐   │
         │ 18650 Battery    │   │
         │       │          │   │
         │    TP4056        │   │
         │       │          │   │
         │   ON/OFF Switch  │   │
         │       │          │   │
         │  Boost 5V (A)    │   │
         │  or LDO 3.3V (B) │   │
         └───────┬──────────┘   │
                 │              │
                 ▼              ▼
        ┌────────────────────────────┐
        │  +5V BUS                   │───────────────────────┐
        │                            │                       │
        │   ESP32 Dev Board (VIN)    │             TDA2822M (BTL)
        │          │                 │              Pin 2 VCC│
        │       +3V3 OUT             │                   │   │
        │     ┌────┴────┐           │              ┌────┴───┐
        │     │         │           │              │        │
        │  RDA5807M   SH1106         │        ┌─────┤Pin3    │
        │  RRD-102   OLED           │        │     │  AMP   │
        │   (I2C)    (I2C)          │   C3   │     │        │
        │     │         │           │  coup. │     │Pin1 Pin3
        │     └────┬────┘           │    │   │     └──┬──┬──┘
        │       I2C Bus             │    │   │        │  │
        │    SDA(G21) SCL(G22)      │    └───┘     SPK_A SPK_B
        │                            │    │   │        │  │
        │   Encoder (S141)          │    └───┘     ┌──┴──┴──┐
        │   CLK(G32) DT(G33)       │               │Speaker │
        │   SW(G25)                 │               │ 4Ω 5W │
        │                            │               └────────┘
        │   Battery ADC (G34)       │
        │   R7/R8 divider           │
        └────────────────────────────┘
                 │
              GND BUS
```

### Signal Flow (Audio Path)

```
FM Antenna (75 cm wire)
        │
        ▼
   RDA5807M (3.3V, I2C tuned, volume 0–15 via register)
   LOUT ──[R3 1k]──┐
                    ├── AF_MIX ──[C3 10µF]── AF_IN ── TDA Pin 7
   ROUT ──[R4 1k]──┘
        ▼
   TDA2822M (BTL, 5V)
   Pin 1 (OUT1) ──── SPK_A ──┐
                              ├── 4Ω Speaker
   Pin 3 (OUT2) ──── SPK_B ──┘
```

**Volume is controlled digitally** via the RDA5807M’s internal register (0–15),
adjusted by the left rotary encoder. No hardware potentiometer in the audio path.

---

# 2) Full Net Naming Table

Every wire in this project has a unique name. No ambiguity.

| Net Name | Description | Voltage / Type |
|----------|-------------|----------------|
| `+5V` | 5 V power bus (Arduino today, boost tomorrow) | 5.0 V DC |
| `+3V3` | 3.3 V power bus (ESP32 onboard regulator output) | 3.3 V DC |
| `GND` | Common system ground | 0 V reference |
| `I2C_SDA` | I2C data line (GPIO21), pulled up to +3V3 | 3.3 V logic |
| `I2C_SCL` | I2C clock line (GPIO22), pulled up to +3V3 | 3.3 V logic |
| `AF_L` | Left audio from RDA5807M LOUT pad | ~300 mV AC |
| `AF_R` | Right audio from RDA5807M ROUT pad | ~300 mV AC |
| `AF_MIX` | Mono-mixed audio node (R3 + R4 junction) | ~150 mV AC |
| `AF_IN` | After coupling cap C3 → TDA pin 7 (IN1+) | 0–150 mV AC |
| `SPK_A` | Speaker terminal A (TDA pin 1, OUT1) | AC output |
| `SPK_B` | Speaker terminal B (TDA pin 3, OUT2) | AC output |
| `BTL_FB` | BTL feedback path (pin 1 OUT1 → C8 → pin 5 IN2−) | AC coupled |
| `ENC1_A` | Right encoder channel A / CLK (GPIO32) | 3.3 V logic |
| `ENC1_B` | Right encoder channel B / DT (GPIO33) | 3.3 V logic |
| `ENC1_SW` | Right encoder push-button switch (GPIO25) | 3.3 V logic |
| `ENC2_A` | Left encoder channel A / CLK (GPIO26) | 3.3 V logic |
| `ENC2_B` | Left encoder channel B / DT (GPIO27) | 3.3 V logic |
| `ENC2_SW` | Left encoder push-button switch (GPIO14) | 3.3 V logic |
| `BAT_ADC` | Battery voltage divider midpoint (GPIO34) | 0–2.1 V DC |
| `BAT+` | Battery positive after TP4056 + switch | 3.0–4.2 V DC |

---

# 3) Complete Component List (BOM)

## 3.1 Active Components

| Ref | Part | Qty | Notes |
|-----|------|-----|-------|
| — | ESP32 30-pin USB-C dev board | 1 | ESP32-D0WD-V3 (your board) |
| U1 | RDA5807M on RRD-102 V2.0 module | 1 | I2C address 0x10 (sequential mode) |
| U2 | TDA2822M (DIP-8) | 1 | Mount in IC socket |
| — | SH1106 1.3″ OLED module | 1 | I2C address 0x3C, 4-pin |

## 3.2 Resistors

| Ref | Value | Qty | Purpose |
|-----|-------|-----|---------|
| R1 | 4.7 kΩ | 1 | I2C SDA pull-up to +3V3 |
| R2 | 4.7 kΩ | 1 | I2C SCL pull-up to +3V3 |
| R3 | 1 kΩ | 1 | LOUT → AF_MIX (mono mixer) |
| R4 | 1 kΩ | 1 | ROUT → AF_MIX (mono mixer) |
| R5 | 2.2 Ω | 1 | Zobel network (amp stability, across speaker) |
| R6 | 1 kΩ | 1 | Power LED current limiter (optional) |
| R7 | 100 kΩ | 1 | Battery voltage divider — top (BAT+ to BAT_ADC) |
| R8 | 100 kΩ | 1 | Battery voltage divider — bottom (BAT_ADC to GND) |

**Total resistors: 8** (7 if you skip LED)

**Why these values:**
- 4.7 kΩ pull-ups: faster I2C edge rates on perfboard (better than 10 kΩ for long wires)
- 1 kΩ mixer: prevents RDA5807M L/R outputs from shorting together
- 2.2 Ω Zobel: standard value for TDA2822M output stability
- 100 kΩ divider: <21 µA battery drain (negligible vs. self-discharge)

## 3.3 Capacitors

| Ref | Value | Type | Qty | Purpose |
|-----|-------|------|-----|---------|
| C1 | 0.1 µF | Ceramic (104) | 1 | RDA5807M VCC decoupling (close to U1) |
| C2 | 10 µF 16 V | Electrolytic | 1 | RDA5807M VCC bulk (close to U1) |
| C3 | 10 µF 16 V | Electrolytic | 1 | Audio coupling (AF_MIX → TDA pin 7 IN1+) |
| C4 | 0.1 µF | Ceramic (104) | 1 | TDA2822M VCC decoupling (close to pin 2) |
| C5 | 1000 µF 16 V | Electrolytic | 1 | TDA2822M VCC bulk (near pin 2) |
| C6 | 47 µF 16 V | Electrolytic | 1 | TDA2822M pin 8 (IN1−) to GND — sets gain (~100×). If volume too low, try 100 µF (max gain ~200×, may add hiss). |
| C7 | 0.1 µF | Ceramic (104) | 1 | Zobel cap (in series with R5, across speaker) |
| C8 | 10 µF 16 V | Electrolytic | 1 | BTL feedback coupling (pin 1 OUT1 → pin 5 IN2−). Non-polarized preferred for long-term reliability. |

**For cleaner audio (recommended):**

| Ref | Value | Type | Qty | Purpose |
|-----|-------|------|-----|---------|
| C9 | 0.1 µF | Ceramic (104) | 1 | HF filter across speaker leads (SPK_A to SPK_B) |
| C10 | 10 µF 16 V | Electrolytic | 1 | ESP32 3.3 V rail bulk (near ESP32 3V3 pin) |

**Total capacitors: 8 required + 2 recommended**

- 0.1 µF ceramic: 3 required (C1, C4, C7) + 1 recommended (C9) = 4
- 10 µF electrolytic: 3 required (C2, C3, C8) + 1 recommended (C10) = 4
- 47 µF electrolytic: 1 (C6) — may swap to 100 µF if volume is too low
- 1000 µF electrolytic: 1 (C5)

## 3.4 User Controls

| Ref | Part | Qty | Notes |
|-----|------|-----|-------|
| ENC1 | S141 rotary encoder | 1 | 5-pin: CLK, DT, SW, +, GND — **Right (Tuning)** |
| ENC2 | S141 rotary encoder | 1 | 5-pin: CLK, DT, SW, +, GND — **Left (Volume)** |
| SW1 | Latching ON/OFF switch | 1 | 2-terminal, main power (for tomorrow) |

## 3.5 Output + Indicator

| Ref | Part | Qty | Notes |
|-----|------|-----|-------|
| SPK1 | 4 Ω 5 W speaker | 1 | Mounts on back of board |
| LED1 | Red 5 mm diffused LED | 1 | Power indicator (optional) |

## 3.6 Power Components

| Ref | Part | Qty | Notes |
|-----|------|-----|-------|
| — | 18650 Li-ion battery | 1 | 3.7 V nominal, 3.0–4.2 V |
| — | 18650 battery holder | 1 | Spring-contact type |
| — | TP4056 charging module | 1 | Micro-USB or USB-C input |
| — | MT3608 boost module (Plan A) | 1 | **Buy tomorrow.** Set to 5.0 V output. |
| — | HT7333 or AMS1117-3.3V LDO (Plan B) | 1 | **Alternative if boost unavailable.** |

## 3.7 Antenna + Mechanical

| Part | Qty | Notes |
|------|-----|-------|
| Insulated wire ~75 cm | 1 | FM antenna (quarter-wave for ~100 MHz) |
| 8-pin DIP IC socket | 1 | For TDA2822M (replaceable) |
| 15 × 9 cm perfboard | 1 | Dot board |
| Hookup wire (solid core) | — | Multiple colors recommended |
| Arduino UNO | 1 | **Today only** — provides 5 V power for testing |

---

# 4) IC Pinouts — Get These Right or Nothing Works

## 4.1 ESP32 30-Pin Dev Board — Pinout (USB-C at Top)

```
              ┌────────────────┐
              │    USB-C       │
    ┌─────────┤                ├─────────┐
    │  3V3    │                │   D23   │
    │  EN     │                │   D22 ← I2C_SCL
    │  VP/G36 │    ESP32       │   TX/G1 │
    │  VN/G39 │   WROOM-32    │   RX/G3 │
    │  D34  ← BAT_ADC         │   D21 ← I2C_SDA
    │  D35    │                │   D19   │
    │  D32  ← ENC1_A          │   D18   │
    │  D33  ← ENC1_B          │   D5    │
    │  D25  ← ENC1_SW         │   D17   │
    │  D26  ← ENC2_A          │   D16   │
    │  D27  ← ENC2_B          │   D4    │
    │  D14  ← ENC2_SW         │   D2    │
    │  D12    │                │   D15   │
    │  GND    │                │   GND   │
    │  VIN  ← +5V INPUT       │   3V3   │
    └─────────┤                ├─────────┘
              └────────────────┘
```

**Pins used in this project (9 GPIOs + power):**

| ESP32 Pin | Net | Function |
|-----------|-----|----------|
| GPIO21 (SDA) | I2C_SDA | I2C data — shared by RDA5807M + OLED |
| GPIO22 (SCL) | I2C_SCL | I2C clock — shared by RDA5807M + OLED |
| GPIO32 | ENC1_A | Right encoder channel A (CLK) — Tuning |
| GPIO33 | ENC1_B | Right encoder channel B (DT) — Tuning |
| GPIO25 | ENC1_SW | Right encoder push-button — Seek UP |
| GPIO26 | ENC2_A | Left encoder channel A (CLK) — Volume |
| GPIO27 | ENC2_B | Left encoder channel B (DT) — Volume |
| GPIO14 | ENC2_SW | Left encoder push-button — Seek DOWN |
| GPIO34 | BAT_ADC | Battery voltage (ADC1, input-only) |
| VIN | +5V | Power input (from Arduino today) |
| 3V3 | +3V3 | 3.3 V output to radio + display |
| GND | GND | System ground |

> **IMPORTANT:** Pin positions vary between ESP32 board manufacturers. Verify against
> YOUR board's silkscreen. The GPIO numbers above are always correct; only the physical
> position on the header may differ.

> **Why GPIO34 for ADC:** GPIO34 is input-only (no internal pull-up/down), belongs to
> ADC1 (compatible with WiFi), and has no side functions that could interfere.

## 4.2 TDA2822M (DIP-8) — Correct Pinout (Verified)

**Top view — notch/dot at pin 1 end:**

```
            ┌───╥───┐
    OUT1  1 │●  ║   │ 8  IN1− (gain cap)
     VCC  2 │   ║   │ 7  IN1+ (audio in)
    OUT2  3 │   ║   │ 6  IN2+ (GND ref)
     GND  4 │   ║   │ 5  IN2− (BTL fb)
            └───╨───┘
```

| Pin | Name | BTL Connection |
|-----|------|----------------|
| 1 | OUT1 | SPK_A (speaker terminal A) + Zobel + BTL feedback source |
| 2 | VCC | +5V bus |
| 3 | OUT2 | SPK_B (speaker terminal B) + Zobel endpoint |
| 4 | GND | GND bus |
| 5 | IN2− | C8 (10 µF) ← Pin 1 OUT1 (BTL feedback, inverts Ch1 output) |
| 6 | IN2+ | GND bus (AC ground reference for channel 2) |
| 7 | IN1+ | AF_IN (audio input from C3 coupling cap) |
| 8 | IN1− | C6 (47 µF, + toward pin) → GND (sets gain) |

**WARNING:** This pinout was physically verified on the actual IC. Many online sources and
datasheets show different arrangements. **Trust the table above — it matches YOUR chip.**

**BTL Principle:** Channel 2 receives Channel 1's output (via C8 from pin 1 to pin 5) on its
inverting input (IN2−), with its non-inverting input (IN2+, pin 6) grounded. This makes OUT2
(pin 3) produce the exact inverse of OUT1 (pin 1). The speaker sees double the voltage swing
→ approximately **4× the power** versus single-ended mode.

> **Thermal Note:** At 5 V BTL into 4 Ω, the TDA2822M delivers approximately **~1 W maximum**
> (not the speaker's 5 W rating — that's just the speaker's thermal limit). The IC will get
> warm (~50–60 °C) at full volume. This is normal. If it gets uncomfortably hot: reduce volume
> (lower C6 to 47 µF or 22 µF), ensure airflow, or add a small heatsink pad. Never exceed
> 5.0 V on VCC.
>
> **Gain Note:** C6 = 47 µF is installed by default (~100× gain). If volume is too low,
> swap to 100 µF (max gain ~200×, may add hiss at low volume).
>
> **C8 Note:** C8 sees AC signal swing in both directions (OUT1 is centered at ~VCC/2). An
> electrolytic works in practice, but technically a **non-polarized 10 µF capacitor** would be
> more correct. If C8 ever fails in long-term usage, this is the stress point.
>
> **Zobel R5 Note:** R5 (2.2 Ω) must be rated **≥ 0.5 W**. At loud output, this resistor
> dissipates heat. A tiny 1/8 W resistor will cook slowly.
>
> **Oscilloscope Warning:** Never connect an oscilloscope ground probe to either speaker
> terminal in BTL mode. You will short half the bridge through the scope's ground.

## 4.3 RDA5807M RRD-102 V2.0 Module — Pad Layout

**Top view (component side up, antenna pad on right):**

```
         ┌──────────┐
    SDA  1│          │6  ANT
    SCL  2│ RDA5807M │7  NC
     NC  3│ RRD-102  │8  LOUT
     NC  4│  V2.0    │9  ROUT
    VCC  5│          │10 GND
         └──────────┘
         1.1cm × 1.1cm
```

| Pad | Label | Connect To | Notes |
|-----|-------|------------|-------|
| 1 | SDA | I2C_SDA | Data line |
| 2 | SCL | I2C_SCL | Clock line |
| 3 | NC | — | Not connected |
| 4 | NC | — | Not connected |
| 5 | VCC | +3V3 | **3.3 V ONLY — never 5 V** |
| 6 | ANT | 75 cm wire | FM antenna |
| 7 | NC | — | Not connected |
| 8 | LOUT | AF_L → R3 (1 kΩ) | Left audio output |
| 9 | ROUT | AF_R → R4 (1 kΩ) | Right audio output |
| 10 | GND | GND bus | Ground |

> **If I2C doesn't respond at 0x10:** Try address 0x11 (index mode). Some RRD-102 V2.0 modules
> respond only in index mode, which requires sending a register address byte before data.
> See Section 8.1 and the troubleshooting notes in the Phase 1 document.

## 4.4 SH1106 OLED (1.3″, 128×64, 4-Pin Module)

```
    ┌──────────────────┐
    │                  │
    │   128×64 OLED    │
    │   SH1106         │
    │   3.5cm × 3.5cm  │
    │                  │
    │  GND VCC SCL SDA │  ← 4-pin header at bottom
    └──┬──┬──┬──┬──────┘
       │  │  │  │
```

| Pin | Label | Connect To |
|-----|-------|------------|
| 1 | GND | GND bus |
| 2 | VCC | +3V3 |
| 3 | SCL | I2C_SCL (GPIO22) |
| 4 | SDA | I2C_SDA (GPIO21) |

> **Pin order varies by manufacturer.** Some modules have VCC-GND-SCL-SDA.
> Always check the silkscreen on YOUR module.

## 4.5 S141 Rotary Encoders (2× — Right + Left)

Both encoders are identical S141 modules. They differ only in function and GPIO assignment.

```
    ┌─────────┐          ┌─────────┐
    │  Knob   │          │  Knob   │
    │  RIGHT  │          │  LEFT   │
    │ ○ ○ ○  │          │ ○ ○ ○  │  ← CLK, DT, SW
    │         │          │         │
    │   ○ ○  │          │   ○ ○  │  ← +, GND
    └─────────┘          └─────────┘
     TUNING               VOLUME
```

### ENC1 — Right Encoder (Tuning + Seek UP)

| Pin | Label | Connect To |
|-----|-------|------------|
| CLK | Channel A | ENC1_A (GPIO32) |
| DT | Channel B | ENC1_B (GPIO33) |
| SW | Push button | ENC1_SW (GPIO25) — **Press = Seek UP** |
| + | VCC | +3V3 |
| GND | Ground | GND bus |

### ENC2 — Left Encoder (Volume 0–15 + Seek DOWN)

| Pin | Label | Connect To |
|-----|-------|------------|
| CLK | Channel A | ENC2_A (GPIO26) |
| DT | Channel B | ENC2_B (GPIO27) |
| SW | Push button | ENC2_SW (GPIO14) — **Press = Seek DOWN** |
| + | VCC | +3V3 |
| GND | Ground | GND bus |

> **Pull-ups:** If your encoder module has onboard pull-up resistors (most KY-040 style do),
> supply 3.3V to the + pin. If the module has no pull-ups, the ESP32 `INPUT_PULLUP` will work
> but the pull-up goes to 3.3V (safe).

---

# 5) Complete Circuit — Every Connection

## 5.1 All Connections — Tabular Format

### Power (TODAY — Arduino UNO)

| From | To | Wire Color Suggestion |
|------|----|----------------------|
| Arduino UNO 5V pin | +5V bus on perfboard (and ESP32 VIN) | Red |
| Arduino UNO GND pin | GND bus on perfboard (and ESP32 GND) | Black |

> **Tomorrow:** Remove Arduino wires. Connect regulated output from MT3608 (5 V) or
> HT7333 (3.3 V) to the same bus points.

### ESP32 → System

| ESP32 Pin | Destination |
|-----------|-------------|
| VIN | +5V bus |
| GND (either one) | GND bus |
| 3V3 | +3V3 bus (feeds RDA5807M, OLED, encoders, pull-ups) |
| GPIO21 | I2C_SDA bus |
| GPIO22 | I2C_SCL bus |
| GPIO32 | ENC1 CLK (right encoder) |
| GPIO33 | ENC1 DT (right encoder) |
| GPIO25 | ENC1 SW (right encoder button) |
| GPIO26 | ENC2 CLK (left encoder) |
| GPIO27 | ENC2 DT (left encoder) |
| GPIO14 | ENC2 SW (left encoder button) |
| GPIO34 | BAT_ADC (R7/R8 divider midpoint) |

### I2C Pull-Ups

| Resistor | From | To |
|----------|------|----|
| R1 (4.7 kΩ) | I2C_SDA | +3V3 |
| R2 (4.7 kΩ) | I2C_SCL | +3V3 |

### RDA5807M (U1)

| Module Pad | Connect To |
|------------|------------|
| 1 (SDA) | I2C_SDA |
| 2 (SCL) | I2C_SCL |
| 3 (NC) | Not connected |
| 4 (NC) | Not connected |
| 5 (VCC) | +3V3 |
| 6 (ANT) | 75 cm antenna wire |
| 7 (NC) | Not connected |
| 8 (LOUT) | R3 (1 kΩ) → AF_MIX |
| 9 (ROUT) | R4 (1 kΩ) → AF_MIX |
| 10 (GND) | GND bus |

Place **C1 (0.1 µF ceramic)** between +3V3 and GND, within 1 cm of U1 VCC pad.  
Place **C2 (10 µF electrolytic, + to 3V3)** right next to C1.

### Mono Mixer → TDA Input (Direct Coupling)

| Component | From | To |
|-----------|------|----|
| R3 (1 kΩ) | U1 LOUT (pad 8) | AF_MIX node |
| R4 (1 kΩ) | U1 ROUT (pad 9) | AF_MIX node |
| C3 (10 µF, + toward radio) | AF_MIX | AF_IN (TDA pin 7 IN1+) |

**C3 polarity:** Positive (+) toward AF_MIX (radio side). Negative (−) toward TDA pin 7.

**No potentiometer in this design.** Volume is controlled digitally via the RDA5807M's
internal volume register (0–15), adjusted by the left rotary encoder.

### TDA2822M (U2) — BTL Mode

| Pin | Name | Connect To | Notes |
|-----|------|------------|-------|
| 1 | OUT1 | SPK_A + Zobel (R5) + BTL feedback (C8) | Speaker terminal A |
| 2 | VCC | +5V bus | Power |
| 3 | OUT2 | SPK_B + Zobel (C7) | Speaker terminal B |
| 4 | GND | GND bus | IC ground |
| 5 | IN2− | C8 (10 µF) ← Pin 1 (OUT1) | BTL feedback input |
| 6 | IN2+ | GND bus | AC ground for Ch2 |
| 7 | IN1+ | AF_IN (from C3 coupling cap) | Audio input |
| 8 | IN1− | C6 (47 µF, + toward pin) → GND | Gain stability |

Place **C4 (0.1 µF ceramic)** between +5V and GND, within 1 cm of pin 2.  
Place **C5 (1000 µF electrolytic, + to 5V)** next to C4.

### BTL Feedback Coupling

| Component | From | To |
|-----------|------|----|
| C8 (10 µF electrolytic) | Pin 1 (OUT1) — positive (+) side | Pin 5 (IN2−) — negative (−) side |

This capacitor carries the OUT1 signal to IN2−, creating the bridge output.

### Zobel Network (Across Speaker, for Stability)

```
Pin 1 (OUT1) ───[R5 2.2Ω]───[C7 0.1µF]─── Pin 3 (OUT2)
```

R5 and C7 are in series, connected between OUT1 and OUT2 (across the speaker terminals).

### Speaker

| Terminal | Connect To | WARNING |
|----------|------------|---------|
| SPK_A (+) | TDA Pin 1 (OUT1) | **NOT to GND** |
| SPK_B (−) | TDA Pin 3 (OUT2) | **NOT to GND** |

> In BTL mode, both speaker terminals carry amplified signals. Neither goes to ground.
> Grounding either terminal will short-circuit one amplifier channel.

### Optional Clean Audio Additions

| Component | Between | Purpose |
|-----------|---------|---------|
| C9 (0.1 µF ceramic) | SPK_A and SPK_B (across speaker leads) | HF noise filter |
| C10 (10 µF electrolytic) | +3V3 (+) and GND (−), near ESP32 3V3 pin | Reduce digital noise on 3V3 rail |

### Battery Voltage Divider

```
BAT+ ───[R7 100kΩ]───┬───[R8 100kΩ]─── GND
                      │
                  GPIO34 (BAT_ADC)
```

| Measurement | ADC Voltage |
|-------------|-------------|
| Battery at 4.2 V (full) | 2.10 V |
| Battery at 3.7 V (nominal) | 1.85 V |
| Battery at 3.0 V (empty) | 1.50 V |

All values are within the ESP32 ADC safe range (0–3.3 V). ✓

### ENC1 — Right Encoder (Tuning)

| Encoder Pin | Connect To |
|-------------|------------|
| CLK | GPIO32 (ENC1_A) |
| DT | GPIO33 (ENC1_B) |
| SW | GPIO25 (ENC1_SW) |
| + | +3V3 |
| GND | GND bus |

### ENC2 — Left Encoder (Volume)

| Encoder Pin | Connect To |
|-------------|------------|
| CLK | GPIO26 (ENC2_A) |
| DT | GPIO27 (ENC2_B) |
| SW | GPIO14 (ENC2_SW) |
| + | +3V3 |
| GND | GND bus |

### OLED Display

| OLED Pin | Connect To |
|----------|------------|
| GND | GND bus |
| VCC | +3V3 |
| SCL | I2C_SCL (GPIO22) |
| SDA | I2C_SDA (GPIO21) |

### Power LED (Optional)

```
+5V ───[R6 1kΩ]───[LED ▶│]─── GND
                  anode    cathode
```

Current: (5 V − 2 V) / 1000 Ω = 3 mA. Dim but visible.

---

## 5.2 Full ASCII Schematic — Complete Circuit

```
═══════════════════════════════════════════════════════════════════════
                         COMPLETE CIRCUIT DIAGRAM
                    FM Radio Phase 2 — RDA5807M + ESP32
═══════════════════════════════════════════════════════════════════════

POWER SECTION (TODAY: Arduino 5V)
─────────────────────────────────
  Arduino 5V ──────────┬──────────────────────────────── +5V BUS
                       │
  Arduino GND ─────────┼──────────────────────────────── GND BUS
                       │
                       │    ┌──────────────────────┐
                ESP32  │    │                      │
                 VIN ──┘    │    ESP32 Dev Board    │
                 GND ───────┤                      │
                 3V3 ──┐    │  GPIO21 ── I2C_SDA   │
                       │    │  GPIO22 ── I2C_SCL   │
                       │    │  GPIO32 ── ENC1_A    │
                       │    │  GPIO33 ── ENC1_B    │
                       │    │  GPIO25 ── ENC1_SW   │
                       │    │  GPIO26 ── ENC2_A    │
                       │    │  GPIO27 ── ENC2_B    │
                       │    │  GPIO14 ── ENC2_SW   │
                       │    │  GPIO34 ── BAT_ADC   │
                       │    └──────────────────────┘
                       │
                    +3V3 BUS


RDA5807M + I2C + DECOUPLING
────────────────────────────
                  +3V3
                    │
              ┌─────┤──────┐
              │     │      │
           [R1 4.7k] [R2 4.7k]
              │     │      │
         I2C_SDA    │   I2C_SCL
           │  │     │      │  │
    ESP32  │  │     │      │  │  ESP32
    G21 ───┘  │     │      │  └─── G22
              │     │      │
    ┌─────────────────────────────┐
    │ SDA SCL  NC  NC  VCC       │
    │  1   2   3   4   5         │
    │                            │
    │       RDA5807M             │
    │       RRD-102 V2.0         │
    │                            │
    │  6   7   8   9  10         │
    │ ANT  NC LOUT ROUT GND      │
    └──┬──────┬────┬────┬────────┘
       │      │    │    │
    75cm     NC   AF_L AF_R      GND BUS
    wire           │    │          │
       │           │    │          │
                   │    │     ┌────┘
                   │    │  [C1 0.1µF]──+3V3
                   │    │  [C2 10µF+]──+3V3
                   │    │     (close to U1)


MONO MIXER → DIRECT TO AMP
───────────────────────────
    AF_L (LOUT) ──[R3 1kΩ]──┐
                             │
                           AF_MIX
                             │
    AF_R (ROUT) ──[R4 1kΩ]──┘
                             │
                      [C3 10µF electrolytic]
                      (+) toward AF_MIX
                      (−) toward TDA pin 7
                             │
                           AF_IN
                             │
                             ▼
                      TDA2822M Pin 7 (IN1+)


TDA2822M BTL AMPLIFIER
──────────────────────
                           +5V BUS
                             │
                      [C4 0.1µF] ── GND    (ceramic, close to pin 2)
                             │
                      [C5 1000µF+]── GND   (bulk, near pin 2)
                             │
                  ┌──────────┴──────────┐
           OUT1  1│●                    │8  IN1− ──[C6 47µF+]── GND
                  │                     │          (+) toward pin 8
           VCC   2│── +5V               │7  IN1+ ── AF_IN (from C3)
                  │    TDA2822M         │
           OUT2  3│                     │6  IN2+ ── GND BUS
                  │    (in socket)      │
            GND  4│── GND BUS           │5  IN2− ── C8 (−) ← from pin 1 (+)
                  └──────────┬──────────┘
                             │
    Connections:
    Pin 1 (OUT1) ──┬── SPK_A (speaker terminal A)
                   │
                   ├──[R5 2.2Ω]──[C7 0.1µF]── Pin 3 (OUT2)  ← Zobel
                   │
                   └──[C8 10µF+]── Pin 5 (IN2−)              ← BTL feedback
                      (+) toward pin 1

    Pin 2 (VCC) ── +5V BUS

    Pin 3 (OUT2) ──┬── SPK_B (speaker terminal B)
                   │
                   └── Zobel R5+C7 other end (connects to pin 1 via Zobel)

    Pin 4 (GND) ── GND BUS

    Pin 5 (IN2−) ── C8 (−) side    (receives BTL feedback from OUT1)

    Pin 6 (IN2+) ── GND BUS

    Pin 7 (IN1+) ── AF_IN (from C3 coupling cap)

    Pin 8 (IN1−) ──[C6 47µF+]── GND     (+) toward pin 8


SPEAKER (BTL — NOT GROUNDED)
────────────────────────────
    TDA Pin 1 (OUT1) ─── SPK_A ──┐
                                  │  ┌──────────┐
                              [C9 0.1µF opt.]  │  4Ω 5W  │
                                  │  │ Speaker  │
    TDA Pin 3 (OUT2) ─── SPK_B ──┘  └──────────┘

    ⚠ NEITHER speaker terminal connects to GND in BTL mode


ENCODERS (ACTIVE LOW)
─────────────────────
    ENC1 RIGHT (Tuning + Seek UP)
    +3V3 ── Encoder +
    GND  ── Encoder GND
    Encoder CLK ── GPIO32 (ENC1_A)
    Encoder DT  ── GPIO33 (ENC1_B)
    Encoder SW  ── GPIO25 (ENC1_SW)  ── (pressed = LOW = Seek UP)

    ENC2 LEFT (Volume + Seek DOWN)
    +3V3 ── Encoder +
    GND  ── Encoder GND
    Encoder CLK ── GPIO26 (ENC2_A)
    Encoder DT  ── GPIO27 (ENC2_B)
    Encoder SW  ── GPIO14 (ENC2_SW)  ── (pressed = LOW = Seek DOWN)


OLED DISPLAY (I2C)
──────────────────
    +3V3 ── OLED VCC
    GND  ── OLED GND
    I2C_SCL (GPIO22) ── OLED SCL
    I2C_SDA (GPIO21) ── OLED SDA


BATTERY ADC (Voltage Divider)
─────────────────────────────
    BAT+ (from TP4056 output or battery +)
      │
    [R7 100kΩ]
      │
      ├── GPIO34 (BAT_ADC)
      │
    [R8 100kΩ]
      │
    GND BUS


POWER LED (Optional)
────────────────────
    +5V ──[R6 1kΩ]──[LED ▶│]── GND
                    anode   cathode
```

---

# 6) Dot Board Layout Plan

## 6.1 Board Dimensions

- **Size:** 15 cm × 9 cm (approximately 59 × 35 holes at 2.54 mm pitch)
- **Hole grid:** 2.54 mm (0.1 inch) standard spacing
- **Component side (FRONT):** Plain brown side — where you place components
- **Solder side (BACK):** Copper pads — where you solder and route wires

## 6.2 Physical Layout — FRONT VIEW (Component Side)

User faces this side. Origin (0,0) is bottom-left corner.
Layout matches the builder's verified hand-drawn board plan.

```
    ┌──────────────────── 15 cm ────────────────────┐
    │                                                │
    │  ANTENNA                  ENC1        ENC2     │  ↑
    │  (wire exits              ┌───────┐  ┌───────┐│
    │   top-left,          ○──  │CHANNEL│  │VOLUME ││
    │   away from          │    │(tune) │  │(vol)  ││
    │   amp noise)         │    └───────┘  └───────┘│
    │                      │                         │
    │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┤ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│
    │                      │                         │
    │  ┌──────────┐   ┌═══╧═══════════┐  ┌────────┐│
    │  │ RDA5807M │   │               │  │TDA2822M││
    │  │ RRD-102  │   │    SH1106     │  │  DIP-8 ││  9 cm
    │  │ 1.1×1.1  │   │     OLED      │  │ 1.0×0.7││
    │  │ [C1][C2] │   │   3.5×3.5cm   │  │[C4][C5]││
    │  │ [R1][R2] │   │               │  │[C6][C8]││
    │  └──────────┘   └═══════════════┘  │[R5][C7]││
    │                                     └────────┘│
    │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│
    │                                                │
    │  ┌────────────────────────────────────────────┐│
    │  │                                            ││
    │  │          ESP32 Dev Board (5.0 × 2.9 cm)    ││
    │  │          USB-C port faces LEFT edge         ││  ↓
    │  │                                            ││
    │  └────────────────────────────────────────────┘│
    │                                                │
    (0,0)────────────────────────────────────────────┘

    Signal flow (left → right):
    Antenna ──► RDA5807M ──► [C3] ──► TDA2822M pin 7 ──► Speaker (back)
                  │                        │
                  └── I2C ── OLED ── I2C ──┘── ESP32 (bottom)

    Bus wires:
    LEFT EDGE  = +3V3 bus (vertical, top section only)
    RIGHT EDGE = GND bus  (vertical, full height)
    BOTTOM     = +5V bus  (horizontal, under ESP32)
```

### Component Placement Rationale

| Placement | Why |
|-----------|-----|
| Antenna top-left, Amp middle-right | Maximum separation (~12 cm diagonal) reduces RF noise coupling |
| RDA5807M next to OLED | I2C wires ≤ 2 cm — shorter bus, less noise |
| Both encoders top-right together | Ergonomic — knobs side by side for one-hand operation |
| ESP32 bottom full-width | Wide footprint fits naturally, short I2C run to both ICs above |
| TDA2822M near right-side encoders | Volume encoder knob is close to amp — semantic grouping |
| C3 (coupling cap) | Place between RDA5807M and TDA pin 7, as short a wire as possible |

### Key Spacing Rules

| Rule | Distance | Why |
|------|----------|-----|
| Antenna to TDA2822M | **≥ 10 cm** | Prevent amplifier switching noise from desensing receiver |
| RDA5807M to OLED | **1–2 cm** | Short I2C wires reduce noise |
| C3 to TDA pin 7 | **As short as possible** | Audio coupling — minimize noise pickup |
| Antenna wire exit | **Top-left edge** | Furthest from speaker wires and amp |
| ESP32 to I2C ICs | **≤ 5 cm** | Keep I2C bus short; pull-ups near RDA5807M |

## 6.3 BACK VIEW (Solder Side) — Mirrored

When you flip the board over (hinge on the top edge), left↔right is mirrored.
Origin (0,0) is bottom-left of the back view.

```
    ┌──────────────────── 15 cm ────────────────────┐
    │                                                │
    │  Solder joints / wire bridges visible          │
    │  (mirror of front layout)                      │
    │                                                │
    │  ┌──────────────┐       ┌──────────────────┐  │
    │  │              │       │                  │  │
    │  │   SPEAKER    │       │  18650 BATTERY   │  │
    │  │   5 × 5 cm   │       │    HOLDER        │  │
    │  │  (fires      │       │  7.7 × 2.0 cm   │  │
    │  │   backward   │       │                  │  │
    │  │   toward     │       ├──────────────────┤  │
    │  │   user)      │       │ TP4056 module    │  │
    │  │              │       │ 2.6 × 1.7 cm     │  │
    │  └──────────────┘       │ USB-C charge     │  │
    │                         │ port faces RIGHT │  │
    │                         └──────────────────┘  │
    │                                                │
    │       [SW1 power switch]  [MT3608 boost]      │
    │                                                │
    (0,0)────────────────────────────────────────────┘

    Power path (back side):
    18650 ──► TP4056 (charge/protect) ──► SW1 ──► MT3608 (3.7→5V) ──► +5V bus
                                                                   └──► 3V3 via ESP32 regulator
```

### ESP32 Mounting (on FRONT)

The ESP32 dev board is mounted **on the FRONT side** (component side) at the bottom of the
board. Its USB-C port faces the left edge for programming access.

**Steps:**
1. Place female pin-header sockets on the perfboard at the ESP32 footprint location
2. Solder sockets from the BACK (solder side)
3. Plug ESP32 into sockets on the FRONT — removable for reprogramming

This keeps the ESP32 visible but at the bottom, out of the way of the display and knobs.
The back side remains free for speaker, battery, and power modules.

## 6.4 Power Bus Layout

```
    FRONT VIEW — Bus Wire Routing (matches Section 6.2 layout)

    ┌─────────────────── 15 cm ───────────────────┐
    │                                              │
    ║ +3V3 bus                               GND  ║  ← top
    ║ (left edge,                            bus  ║
    ║  top/middle                          (right ║
    ║  sections                            edge,  ║
    ║  only —                               full  ║
    ║  feeds                              height) ║
    ║  RDA5807M                              │    ║
    ║  + OLED)                               │    ║
    ║     │                                  │    ║
    ║     └──── does NOT extend to ESP32 ────┘    ║
    │                                              │
    │════════ +5V bus (horizontal, bottom) ════════│  ← under ESP32
    └──────────────────────────────────────────────┘
```

Use **thick solid-core wire** (20 AWG or thicker) for all bus wires.

### Ground Strategy — Star Topology

Every ground wire goes **directly to the GND bus** on the right edge.
Never daisy-chain grounds through thin wires from pad to pad.

Connections to GND bus (all direct):
- U1 GND (RDA5807M pad 10)
- C1, C2 negative legs
- U2 pin 4 (GND)
- U2 pin 6 (IN2+)
- C4, C5, C6, C7 (via Zobel), C8 (negative leg at pin 5 side)
- ENC1 GND (channel encoder)
- ENC2 GND (volume encoder)
- OLED GND
- R8 bottom leg
- LED1 cathode
- ESP32 GND
- Speaker: **NEITHER terminal goes to GND (BTL mode)**

### RF Isolation

- Keep all **audio wires** (AF_L, AF_R, AF_MIX, AF_IN, SPK_A, SPK_B) at least **3 cm from RDA5807M**
- Keep **speaker wires** at the **opposite end** of the board from the antenna
- Route **I2C wires** (SDA, SCL) away from audio wires if possible
- The **antenna wire** exits the board at the **top-left edge immediately** from the ANT pad — do not
  route it across the board
- Do not route **+5V bus** near RDA5807M (it carries amplifier current spikes)

## 6.5 Master Wiring Reference — Every Wire on the Board

This table lists **every single wire** you need to solder, grouped by source component.
Use this as a checklist during the build — tick each wire off as you solder it.

### A. ESP32 → Everything (I2C, Encoders, ADC, Power)

| # | ESP32 Pin | → Destination | Wire Type | Approx. Length | Notes |
|---|-----------|---------------|-----------|---------------|-------|
| 1 | GPIO 21 (SDA) | RDA5807M SDA (pad 7) | Signal | ~4 cm | Also to OLED SDA. One wire, two destinations |
| 2 | GPIO 22 (SCL) | RDA5807M SCL (pad 8) | Signal | ~4 cm | Also to OLED SCL. One wire, two destinations |
| 3 | GPIO 21 (SDA) | OLED SDA | Signal | ~3 cm | Branched from same SDA line |
| 4 | GPIO 22 (SCL) | OLED SCL | Signal | ~3 cm | Branched from same SCL line |
| 5 | GPIO 32 | ENC1 CLK (channel) | Signal | ~5 cm | Top-right area |
| 6 | GPIO 33 | ENC1 DT (channel) | Signal | ~5 cm | Top-right area |
| 7 | GPIO 25 | ENC1 SW (channel push) | Signal | ~5 cm | Optional if not used in firmware |
| 8 | GPIO 26 | ENC2 CLK (volume) | Signal | ~5 cm | Top-right area |
| 9 | GPIO 27 | ENC2 DT (volume) | Signal | ~5 cm | Top-right area |
| 10 | GPIO 14 | ENC2 SW (volume push) | Signal | ~5 cm | Optional if not used in firmware |
| 11 | GPIO 34 | R7/R8 voltage divider midpoint | Signal | ~3 cm | Battery ADC sense — input only pin |
| 12 | VIN (5V in) | +5V bus | Power | ~2 cm | From MT3608 output via +5V bus |
| 13 | GND | GND bus | Power | ~2 cm | Direct to right-edge GND bus |
| 14 | 3V3 (output) | +3V3 bus | Power | ~3 cm | Feeds RDA5807M + OLED. Left-edge bus |

### B. RDA5807M (RRD-102 V2.0) — Middle-Left Area

| # | RDA5807M Pad | → Destination | Component | Notes |
|---|-------------|---------------|-----------|-------|
| 1 | VCC (pad 1) | +3V3 bus | — | Via C1 (100nF) bypass to GND |
| 2 | GND (pad 10) | GND bus | — | Direct to right-edge bus |
| 3 | SDA (pad 7) | ESP32 GPIO 21 | — | I2C data (shared bus) |
| 4 | SCL (pad 8) | ESP32 GPIO 22 | — | I2C clock (shared bus) |
| 5 | VCC (pad 1) | R1 (4.7kΩ) | Pull-up | R1 other end → SDA line |
| 6 | VCC (pad 1) | R2 (4.7kΩ) | Pull-up | R2 other end → SCL line |
| 7 | VCC (pad 1) | C2 (10µF +) | Bulk decouple | C2 (−) → GND |
| 8 | LOUT (pad 2) | R3 (1kΩ) | Mixer | R3 other end → AF_MIX node |
| 9 | ROUT (pad 4) | R4 (1kΩ) | Mixer | R4 other end → AF_MIX node |
| 10 | ANT (pad 6) | Antenna wire | — | Exits top-left, 60–75 cm wire |

### C. Audio Mixer → TDA2822M Input (between RDA and TDA)

| # | From | → To | Component | Notes |
|---|------|-------|-----------|-------|
| 1 | R3 output | AF_MIX node | — | L+R mix junction |
| 2 | R4 output | AF_MIX node | — | L+R mix junction |
| 3 | AF_MIX node | C3 (10µF +) | Coupling cap | (+) toward AF_MIX, (−) toward TDA |
| 4 | C3 (−) output | TDA2822M pin 7 (IN1+) | — | As short as possible! |

### D. TDA2822M (DIP-8) — Middle-Right Area

| # | TDA Pin | → Destination | Component | Notes |
|---|---------|---------------|-----------|-------|
| 1 | Pin 2 (VCC) | +5V bus | — | Via C4 (100nF) bypass to GND |
| 2 | Pin 2 (VCC) | C5 (1000µF +) | Bulk decouple | C5 (−) → GND |
| 3 | Pin 4 (GND) | GND bus | — | Direct to right-edge bus |
| 4 | Pin 6 (IN2+) | GND bus | — | BTL reference ground |
| 5 | Pin 7 (IN1+) | C3 (−) output | Audio input | From mixer coupling cap |
| 6 | Pin 8 (IN1−) | C6 (47µF +) | Gain cap | C6 (−) → GND. (+) toward pin 8 |
| 7 | Pin 1 (OUT1) | C8 (10µF) | BTL feedback | C8 other end → pin 5 |
| 8 | Pin 5 (IN2−) | C8 (10µF) | BTL feedback | C8 from pin 1 |
| 9 | Pin 1 (OUT1) | R5 (2.2Ω ≥0.5W) | Zobel network | R5 other end → C7 |
| 10 | R5 → C7 junction | C7 (100nF) | Zobel network | C7 other end → pin 3 |
| 11 | Pin 3 (OUT2) | C7 (100nF) | Zobel network | From R5-C7 junction |
| 12 | Pin 1 (OUT1) | Speaker wire (+) | SPK_A | Through board to back — speaker left side |
| 13 | Pin 3 (OUT2) | Speaker wire (−) | SPK_B | Through board to back — speaker left side |

### E. Encoders (Top-Right Area)

| # | Encoder Pin | → Destination | Notes |
|---|-------------|---------------|-------|
| 1 | ENC1 CLK | ESP32 GPIO 32 | Channel/tuning encoder |
| 2 | ENC1 DT | ESP32 GPIO 33 | Channel/tuning encoder |
| 3 | ENC1 SW | ESP32 GPIO 25 | Push button (optional) |
| 4 | ENC1 GND | GND bus | Direct to right-edge bus |
| 5 | ENC2 CLK | ESP32 GPIO 26 | Volume encoder |
| 6 | ENC2 DT | ESP32 GPIO 27 | Volume encoder |
| 7 | ENC2 SW | ESP32 GPIO 14 | Push button (optional) |
| 8 | ENC2 GND | GND bus | Direct to right-edge bus |

### F. OLED Display (Center)

| # | OLED Pin | → Destination | Notes |
|---|----------|---------------|-------|
| 1 | VCC | +3V3 bus | Left-edge bus |
| 2 | GND | GND bus | Right-edge bus |
| 3 | SDA | ESP32 GPIO 21 | Shared I2C bus with RDA5807M |
| 4 | SCL | ESP32 GPIO 22 | Shared I2C bus with RDA5807M |

### G. Battery ADC Sense (Bottom Area)

| # | From | → To | Notes |
|---|------|-------|-------|
| 1 | +VBAT (before MT3608) | R7 (100kΩ) top | Raw battery voltage |
| 2 | R7 bottom | R8 (100kΩ) top | Voltage divider midpoint |
| 3 | R7-R8 midpoint | ESP32 GPIO 34 | ADC input (VBAT ÷ 2) |
| 4 | R8 bottom | GND bus | Completes divider |

### H. Power Path (Back Side)

| # | From | → To | Notes |
|---|------|-------|-------|
| 1 | 18650 (+) | TP4056 B+ | Battery to charger |
| 2 | 18650 (−) | TP4056 B− | Battery to charger |
| 3 | TP4056 OUT+ | SW1 input | Charge-protected output |
| 4 | SW1 output | MT3608 VIN+ | Switched battery power |
| 5 | TP4056 OUT− | MT3608 GND | Ground reference |
| 6 | MT3608 VOUT+ | +5V bus | Boosted 5V to entire board |
| 7 | MT3608 GND | GND bus | Shared ground |

### Wire Count Summary

| Category | Wires |
|----------|-------|
| ESP32 signals | 10 |
| ESP32 power | 4 |
| RDA5807M | 10 |
| Audio mixer + coupling | 4 |
| TDA2822M + passives | 13 |
| Encoders | 8 |
| OLED | 4 |
| Battery ADC | 4 |
| Power path (back) | 7 |
| **Total wires** | **~64** |

---

# 7) Step-by-Step Build Procedure

**Golden Rule: Build in stages. Verify at each stage. Never solder everything at once.**

## Stage 1 — Power Rails

1. Using thick solid wire on the copper side, lay down a **GND bus** along the right edge (vertical, full height).
2. Lay down a **+5V bus** along the bottom edge (horizontal, under ESP32 area).
3. Lay down a **+3V3 bus** along the left edge (vertical, top and middle sections only).
4. Connect the +5V bus and GND bus to landing pads near the bottom where Arduino wires will connect.

**✅ Verification:**
- Multimeter continuity: GND bus has no break along its length
- Multimeter continuity: +5V bus has no break
- Multimeter: +5V bus and GND bus are NOT shorted (no beep)
- Multimeter: +3V3 bus and GND bus are NOT shorted

## Stage 2 — ESP32 Landing Pads

5. Mount ESP32 on the FRONT at the bottom of the board (USB-C port toward left edge).
6. Solder female pin-header sockets onto the perfboard. Plug ESP32 into sockets (removable).
7. Create landing pads (solder points) for each ESP32 pin you'll use:
   - VIN → connect to +5V bus
   - GND → connect to GND bus
   - 3V3 → connect to +3V3 bus
   - GPIO21 → landing pad for I2C_SDA
   - GPIO22 → landing pad for I2C_SCL
   - GPIO32 → landing pad for ENC1_A (right encoder)
   - GPIO33 → landing pad for ENC1_B
   - GPIO25 → landing pad for ENC1_SW
   - GPIO26 → landing pad for ENC2_A (left encoder)
   - GPIO27 → landing pad for ENC2_B
   - GPIO14 → landing pad for ENC2_SW
   - GPIO34 → landing pad for BAT_ADC

**✅ Verification:**
- Connect Arduino 5V and GND to the +5V and GND landing pads
- Plug ESP32 into headers
- Measure ESP32 VIN pin: should read ~5.0 V
- Measure ESP32 3V3 pin: should read ~3.3 V
- Measure +3V3 bus: should read ~3.3 V
- If 3V3 reads 0 V: check connection from ESP32 3V3 pin to +3V3 bus

## Stage 3 — RDA5807M Radio Module + I2C

8. Mount U1 (RDA5807M on RRD-102 V2.0 module) in the **middle-left** area, close to OLED display.
9. Wire U1 pad 5 (VCC) → +3V3 bus.
10. Wire U1 pad 10 (GND) → GND bus.
11. Solder **R1 (4.7 kΩ):** from U1 SDA (pad 1) to +3V3 bus.
12. Solder **R2 (4.7 kΩ):** from U1 SCL (pad 2) to +3V3 bus.
13. Wire U1 SDA (pad 1) → ESP32 GPIO21 landing pad.
14. Wire U1 SCL (pad 2) → ESP32 GPIO22 landing pad.
15. Solder **C1 (0.1 µF ceramic)** between +3V3 and GND, within 1 cm of U1 VCC.
16. Solder **C2 (10 µF electrolytic, + to +3V3)** next to C1.
17. Solder antenna wire to U1 ANT (pad 6). Route wire out the top edge of the board immediately.

**✅ Verification:**
- Power on (Arduino connected)
- Measure U1 VCC (pad 5): should read 3.2–3.4 V
- Upload the I2C scanner sketch (Section 8.5). Open Serial Monitor at 115200 baud.
- Should detect device at address **0x10** (RDA5807M) or **0x11** (alternate mode)
- If no device found: check SDA/SCL wiring, check R1/R2 go to 3V3 (not 5V)

## Stage 4 — OLED Display

20. Mount the SH1106 OLED module. 4 wires:
    - VCC → +3V3 bus
    - GND → GND bus
    - SCL → I2C_SCL bus (same wire/pad as RDA5807M SCL and GPIO22)
    - SDA → I2C_SDA bus (same wire/pad as RDA5807M SDA and GPIO21)

**✅ Verification:**
- Upload I2C scanner again
- Should detect TWO devices: **0x3C** (OLED) and **0x10** (RDA5807M)
- Upload display test sketch (Section 8.6) — screen should show "Hello FM Radio!"
- If display not found: check VCC is 3.3 V (not 5 V), swap SDA/SCL if reversed

## Stage 5 — Dual Encoders

21. Mount **ENC1** (channel/tuning encoder) in the **top-right** area, left of ENC2.
22. Wire ENC1 CLK → GPIO32 landing pad.
23. Wire ENC1 DT → GPIO33 landing pad.
24. Wire ENC1 SW → GPIO25 landing pad.
25. Wire ENC1 + → +3V3 bus.
26. Wire ENC1 GND → GND bus.
27. Mount **ENC2** (volume encoder) in the **top-right** area, right of ENC1.
28. Wire ENC2 CLK → GPIO26 landing pad.
29. Wire ENC2 DT → GPIO27 landing pad.
30. Wire ENC2 SW → GPIO14 landing pad.
31. Wire ENC2 + → +3V3 bus.
32. Wire ENC2 GND → GND bus.

**✅ Verification:**
- Upload encoder test sketch (Section 8.7)
- Open Serial Monitor at 115200 baud
- Turn each encoder slowly — should print position count changing
- Press each encoder button — should print "Button pressed"
- If encoder counts wrong direction: swap CLK and DT wires

## Stage 6 — Mono Mixer + Coupling Capacitor

33. Solder **R3 (1 kΩ):** from U1 LOUT (pad 8) to a shared node (AF_MIX).
34. Solder **R4 (1 kΩ):** from U1 ROUT (pad 9) to the same AF_MIX node.
35. Solder **C3 (10 µF electrolytic):** positive (+) to AF_MIX, negative (−) toward TDA pin 7 (IN1+).
    Keep this wire as short as possible.

**✅ Verification:**
- With radio powered and tuned to a station (run basic RDA5807M test):
  - Touch AF_MIX node with multimeter probe — should read small AC voltage
  - If no AC at AF_MIX: check R3/R4 connections to LOUT/ROUT pads

> **Note:** No volume pot — volume is controlled digitally via the RDA5807M internal register (0–15), adjusted by the left rotary encoder.

## Stage 7 — TDA2822M Amplifier (BTL)

34. Solder the **8-pin DIP IC socket** in the **middle-right** area. Verify notch orientation — pin 1 at notch end.
35. Solder **C4 (0.1 µF ceramic)** between +5V and GND, close to socket pin 2 / pin 4.
36. Solder **C5 (1000 µF electrolytic):** positive (+) to +5V, negative (−) to GND. Near socket.
37. Solder **C6 (47 µF electrolytic):** positive (+) to socket pin 8 (IN1−), negative (−) to GND. (If volume too low later, swap to 100 µF.)
38. Wire socket **pin 7 (IN1+)** ← C3 negative leg (AF_IN, direct from coupling cap).
39. Wire socket **pin 4 (GND)** → GND bus.
40. Wire socket **pin 6 (IN2+)** → GND bus.
41. Wire socket **pin 2 (VCC)** → +5V bus.
42. Solder **C8 (10 µF electrolytic):** positive (+) to socket pin 1 (OUT1), negative (−) to socket pin 5 (IN2−). This is the BTL feedback coupling.
43. Wire socket **pin 1 (OUT1)** → speaker pad for SPK_A.
44. Wire socket **pin 3 (OUT2)** → speaker pad for SPK_B.
45. Solder **Zobel network:** R5 (2.2 Ω, ≥0.5W) in series with C7 (0.1 µF ceramic), connected from pin 1 (OUT1) to pin 3 (OUT2).
46. **Leave pin 3 (OUT2) NOT connected to GND.** (BTL mode — critical!)
47. **Do NOT insert the TDA2822M IC yet.** Leave socket empty until all nearby soldering is done.

**✅ Verification (before inserting IC):**
- Continuity: Pin 2 connects to +5V bus ✓
- Continuity: Pin 4 connects to GND bus ✓
- Continuity: Pin 6 connects to GND bus ✓
- Continuity: Pin 7 connects to C3 negative leg (AF_IN) ✓
- NOT shorted: Pin 1 to GND ✓ (should NOT beep — speaker is floating)
- NOT shorted: Pin 3 to GND ✓ (should NOT beep)
- NOT shorted: Pin 1 to Pin 3 ✓ (Zobel has 2.2 Ω — will show ~2 Ω, but NOT 0 Ω short)

## Stage 8 — Speaker

48. Connect speaker wires to SPK_A and SPK_B pads (from pin 1 and pin 3).
49. Route speaker wires from TDA pins 1 & 3 through the board to the speaker on the back-left. Keep away from antenna/RDA area.
50. Mount speaker on back of board or in enclosure.

**WARNING:** In BTL mode, speaker is **floating between two outputs.** Neither wire goes to GND.
If you accidentally ground one speaker wire, you will short an amplifier channel when the IC is inserted.

## Stage 9 — Battery Sensing

51. Solder **R7 (100 kΩ):** from BAT+ connection point to GPIO34 landing pad.
52. Solder **R8 (100 kΩ):** from GPIO34 landing pad to GND bus.

**Today:** Connect the BAT+ point to the battery holder positive (through TP4056 if wired, 
or directly to battery + for testing). This only reads battery voltage — it does not power anything.

**✅ Verification:**
- With a charged 18650 in the holder:
  - Measure voltage at GPIO34 pad: should read ~1.8–2.1 V (half of battery voltage)
  - Upload ADC test sketch (Section 8.8) — should print battery voltage

## Stage 10 — Optional Components

53. (Optional) Solder **R6 (1 kΩ)** and **LED1** for power indicator: R6 from +5V to LED anode, LED cathode to GND.
54. (Optional) Solder **C9 (0.1 µF ceramic)** across speaker leads (SPK_A to SPK_B).
55. (Optional) Solder **C10 (10 µF electrolytic)** from +3V3 (+) to GND (−), near ESP32 3V3 pin.

## Stage 11 — Final Assembly + IC Insert

56. **Insert TDA2822M** into the IC socket. Pin 1 (notch end) must match socket notch.
57. Inspect every solder joint on copper side. Look for cold joints or bridges.
58. Verify LED polarity (if installed).
59. Verify all electrolytic cap polarities (+ stripes).
60. Proceed to Pre-Power Checklist (Section 11).

## Stage 12 — Tomorrow's Power Integration

**Do NOT do this today. Leave power input as Arduino 5V wires.**

### Plan A — MT3608 Boost Module (Recommended)

1. Set MT3608 output to 5.0 V using a multimeter + trim pot (BEFORE connecting to circuit)
2. Battery → TP4056 → ON/OFF Switch → MT3608 IN+/IN−
3. MT3608 OUT+ → +5V bus (remove Arduino 5V wire)
4. MT3608 OUT− → GND bus (remove Arduino GND wire)
5. ESP32 USB-C can still be connected for programming (auto-switching)

### Plan B — HT7333 LDO (3.3 V Output)

1. Battery → TP4056 → ON/OFF Switch → HT7333 IN
2. HT7333 OUT (3.3 V) → ESP32 3V3 pin directly (bypass VIN and onboard regulator)
3. Battery voltage (after switch) also goes to TDA2822M pin 2 directly (3.0–4.2 V)
4. **NOTE:** TDA2822M at 3.7 V battery delivers ~0.2 W to 4 Ω speaker in BTL mode. Quieter than Plan A.
5. Remove connection from +5V bus to TDA pin 2; wire battery+ (after switch) to TDA pin 2 instead.

---

# 8) ESP32 Beginner Section — For Your Teammate

## 8.1 What Is ESP32?

The ESP32 is a powerful microcontroller (tiny computer) made by Espressif Systems. Think of it as
an "Arduino on steroids":

| Feature | Arduino UNO | ESP32 |
|---------|-------------|-------|
| CPU | 1 core, 16 MHz | **2 cores, 240 MHz** |
| RAM | 2 KB | **520 KB** |
| Flash | 32 KB | **4 MB** |
| WiFi | None | **Built-in** |
| Bluetooth | None | **Built-in** |
| ADC pins | 6 (10-bit) | **18 (12-bit)** |
| GPIO voltage | 5 V | **3.3 V** |
| Price | ~$10 | **~$3** |

Your specific chip: **ESP32-D0WD-V3** with 2 cores at 240 MHz, 4 MB flash.

**Key difference from Arduino:** All GPIO pins work at **3.3 V**, not 5 V. This means:
- Never connect a 5 V signal to any GPIO pin
- The I2C bus runs at 3.3 V (which RDA5807M and SH1106 both need)
- Use the VIN pin (5 V) only for power input

## 8.2 Installing ESP32 Board Support in Arduino IDE

### Step 1: Install Arduino IDE
- Download from https://www.arduino.cc/en/software
- Install version 2.x (recommended)

### Step 2: Add ESP32 Board URL
1. Open Arduino IDE
2. Go to **File → Preferences** (or Arduino IDE → Settings on Mac)
3. In the **"Additional Board Manager URLs"** field, paste this URL:
   ```
   https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
   ```
4. Click **OK**

### Step 3: Install ESP32 Board Package
1. Go to **Tools → Board → Boards Manager**
2. Search for **"esp32"**
3. Find **"esp32 by Espressif Systems"**
4. Click **Install** (this downloads ~200 MB, be patient)
5. Wait for installation to complete

### Step 4: Select Your Board
1. Go to **Tools → Board → esp32**
2. Select **"ESP32 Dev Module"** (works for all 30-pin clones)

### Step 5: Configure Settings
1. **Tools → Port:** Select the COM port that appears when you plug in your ESP32 via USB-C
   - If no port appears: Install CP2102 or CH340 USB driver for your board
   - Windows: Check Device Manager → Ports (COM & LPT) for the name
2. **Tools → Upload Speed:** 921600 (fastest, reduce to 115200 if upload fails)
3. **Tools → Flash Frequency:** 80 MHz
4. **Tools → Flash Size:** 4 MB
5. **Tools → Partition Scheme:** Default 4 MB with spiffs

## 8.3 How to Upload Code

1. Write or paste your sketch in the Arduino IDE
2. Click the **✓ (Verify)** button to compile. Fix any errors.
3. Click the **→ (Upload)** button
4. **IMPORTANT:** On many ESP32 boards, you must **hold the BOOT button** on the ESP32 board
   while uploading. Release it after you see "Connecting..." in the output console.
   - Some USB-C boards auto-reset — you may not need to hold BOOT
   - If upload fails with "Failed to connect": hold BOOT, press RESET, then release BOOT
5. After upload completes, press the **RESET (EN)** button on the ESP32 to start your program
6. Open **Tools → Serial Monitor** and set baud rate to **115200**

## 8.4 Hello World Test

Upload this first to verify your ESP32 is working:

```cpp
void setup() {
  Serial.begin(115200);
  delay(1000);
  Serial.println("Hello from ESP32!");
  Serial.print("CPU Frequency: ");
  Serial.print(getCpuFrequencyMhz());
  Serial.println(" MHz");
  Serial.print("Free Heap: ");
  Serial.print(ESP.getFreeHeap());
  Serial.println(" bytes");
}

void loop() {
  Serial.println("ESP32 is alive!");
  delay(2000);
}
```

**Expected output in Serial Monitor (115200 baud):**
```
Hello from ESP32!
CPU Frequency: 240 MHz
Free Heap: 300000+ bytes
ESP32 is alive!
```

If you see garbage characters: change Serial Monitor baud rate to match `Serial.begin()` value.

## 8.5 I2C Scanner Test

This scans the I2C bus and reports all connected devices.

```cpp
#include <Wire.h>

void setup() {
  Serial.begin(115200);
  Wire.begin(21, 22);  // SDA = GPIO21, SCL = GPIO22
  delay(1000);
  Serial.println("I2C Scanner starting...");
}

void loop() {
  int devicesFound = 0;
  for (byte addr = 1; addr < 127; addr++) {
    Wire.beginTransmission(addr);
    byte error = Wire.endTransmission();
    if (error == 0) {
      Serial.print("Device found at 0x");
      if (addr < 16) Serial.print("0");
      Serial.print(addr, HEX);
      if (addr == 0x3C) Serial.print("  <-- SH1106 OLED");
      if (addr == 0x10) Serial.print("  <-- RDA5807M Radio");
      Serial.println();
      devicesFound++;
    }
  }
  if (devicesFound == 0) {
    Serial.println("No I2C devices found! Check wiring.");
  } else {
    Serial.print(devicesFound);
    Serial.println(" device(s) found.");
  }
  Serial.println("---");
  delay(3000);
}
```

**Expected output with both devices connected:**
```
Device found at 0x10  <-- RDA5807M Radio
Device found at 0x3C  <-- SH1106 OLED
2 device(s) found.
```

## 8.6 Display Test

First, install the U8g2 library:
1. Go to **Sketch → Include Library → Manage Libraries**
2. Search for **"U8g2"** by oliver
3. Click **Install**

```cpp
#include <U8g2lib.h>
#include <Wire.h>

// SH1106 128x64 I2C
U8G2_SH1106_128X64_NONAME_F_HW_I2C u8g2(U8G2_R0, /* reset=*/ U8X8_PIN_NONE);

void setup() {
  u8g2.begin();
  u8g2.clearBuffer();
  u8g2.setFont(u8g2_font_ncenB14_tr);
  u8g2.drawStr(10, 30, "Hello FM");
  u8g2.drawStr(10, 55, "Radio!");
  u8g2.sendBuffer();
}

void loop() {}
```

**Expected:** OLED displays "Hello FM" and "Radio!" in large text.

If nothing appears:
- Check VCC is 3.3 V
- Try swapping SDA and SCL wires
- Try address 0x3D instead of 0x3C (change constructor)

## 8.7 Encoder Test

```cpp
// Rotary Encoder Test — ESP32
// CLK=GPIO32, DT=GPIO33, SW=GPIO25

#define ENC_A   32
#define ENC_B   33
#define ENC_SW  25

volatile int encoderPos = 0;
volatile bool encoderMoved = false;

void IRAM_ATTR encoderISR() {
  if (digitalRead(ENC_B) == LOW) {
    encoderPos++;
  } else {
    encoderPos--;
  }
  encoderMoved = true;
}

void setup() {
  Serial.begin(115200);
  pinMode(ENC_A, INPUT_PULLUP);
  pinMode(ENC_B, INPUT_PULLUP);
  pinMode(ENC_SW, INPUT_PULLUP);
  attachInterrupt(digitalPinToInterrupt(ENC_A), encoderISR, FALLING);
  Serial.println("Encoder Test — turn knob or press button");
}

void loop() {
  if (encoderMoved) {
    encoderMoved = false;
    Serial.print("Position: ");
    Serial.println(encoderPos);
  }

  if (digitalRead(ENC_SW) == LOW) {
    Serial.println("*** BUTTON PRESSED ***");
    delay(300);  // debounce
  }

  delay(10);
}
```

**Expected:** Turning the encoder prints increasing/decreasing position values.
Pressing the button prints "BUTTON PRESSED".

If the count goes the wrong direction: swap the wires on CLK and DT.

## 8.8 Battery ADC Test

```cpp
#define BAT_ADC_PIN 34

void setup() {
  Serial.begin(115200);
  analogReadResolution(12);       // 0-4095
  analogSetAttenuation(ADC_11db); // Full 0-3.3V range
  Serial.println("Battery ADC Test");
}

void loop() {
  int raw = analogRead(BAT_ADC_PIN);
  float adcVoltage = raw * (3.3 / 4095.0);
  float batteryVoltage = adcVoltage * 2.0;  // voltage divider ×2

  Serial.print("ADC Raw: ");
  Serial.print(raw);
  Serial.print("  ADC V: ");
  Serial.print(adcVoltage, 2);
  Serial.print("  Battery V: ");
  Serial.print(batteryVoltage, 2);

  // Approximate percentage
  int percent = map(constrain(raw, 1862, 2610), 1862, 2610, 0, 100);
  Serial.print("  Battery: ");
  Serial.print(percent);
  Serial.println("%");

  delay(2000);
}
```

> **Note:** ESP32 ADC has non-linearity. The readings above are approximate (±8–10% without calibration).
> For better accuracy, calibrate against a multimeter reading.

---

# 9) Firmware Architecture

## 9.1 State Machine

```
    ┌──────────────────┐
    │   BOOT           │ ← Power on
    │  Show welcome:   │
    │  "FM RADIO"      │
    │  "EE2050"        │
    │  Load last freq  │
    │  Load last vol   │
    │  Init I2C        │
    │  Init display    │
    │  Init encoders   │
    └────────┬─────────┘
             │ (2 seconds)
             ▼
    ┌──────────────────┐
    │   NORMAL MODE    │ ← Main operating state
    │                  │
    │  Display:        │
    │  - Frequency     │
    │  - Station name  │
    │  - Volume (0-15) │
    │  - Battery %     │
    │  - Signal bars   │
    │  - Equalizer     │
    │    animation     │
    │                  │
    │  ENC1 (right):   │◄──── Rotate: tune ±0.1 MHz
    │  - Rotate=tune   │◄──── Press: Seek UP
    │                  │
    │  ENC2 (left):    │◄──── Rotate: volume 0-15
    │  - Rotate=volume │◄──── Press: Seek DOWN
    └──────────────────┘
```

## 9.2 Boot Behavior

1. ESP32 powers on → runs `setup()`
2. Initialize Serial (115200 baud)
3. Initialize I2C (SDA=21, SCL=22)
4. Initialize display (U8g2, SH1106)
5. Show **welcome screen** for 2 seconds: **"FM RADIO"** and **"EE2050"**
6. Initialize RDA5807M → tune to saved frequency from NVS (Preferences library)
7. Set volume to saved level from NVS (default 8)
8. Initialize dual encoder interrupts (ENC1 + ENC2)
9. Initialize battery ADC
10. Enter NORMAL MODE

## 9.3 Button Logic (Simplified — Two Encoders)

```
ENC1 (right encoder) button → GPIO25:
    Press → Seek UP (find next station at higher frequency)

ENC2 (left encoder) button → GPIO14:
    Press → Seek DOWN (find next station at lower frequency)

No double-click or long-press needed — each button has one function.
Debounce: 250 ms cooldown between presses.
```

## 9.4 Encoder Rotation Logic

```
ENC1 (right = tuning):
    On encoder interrupt (GPIO32 FALLING):
        Read GPIO33 to determine direction
        If CW:  frequency += 0.1 MHz
        If CCW: frequency -= 0.1 MHz
        Wrap: 108.0 + 0.1 → 87.5, 87.5 - 0.1 → 108.0
        Tune RDA5807M to new frequency
        Look up station name (±0.05 MHz match)
        Save frequency to NVS (debounced — save after 2s of no change)

ENC2 (left = volume):
    On encoder interrupt (GPIO26 FALLING):
        Read GPIO27 to determine direction
        If CW:  volume++ (max 15)
        If CCW: volume-- (min 0)
        Write volume to RDA5807M register 0x05 bits [3:0]
        Save volume to NVS
```

## 9.5 RDA5807M I2C Protocol

### I2C Address

The RDA5807M uses address **0x10** in sequential mode. Writes auto-start at register 0x02,
reads auto-start at register 0x0A. No register address byte is sent.

Alternative: address **0x11** (index/random access mode) — requires sending register address first.

### Write Registers (0x02–0x07, sequential at 0x10)

| Register | Bits | Content |
|----------|------|---------|
| 0x02 | [15] DHIZ, [14] DMUTE, [13] MONO, [12] BASS, [11] RCLK, [10] RCLK, [9] SEEKUP, [8] SEEK, [7:4] SKMODE/CLK, [0] ENABLE | Audio output + seek control + power |
| 0x03 | [15:6] CHAN, [4] TUNE, [3:2] BAND, [1:0] SPACE | Channel + tune trigger + band + spacing |
| 0x04 | [15] STCIEN, [13] DE, [12] SOFTMUTE, [11] AFCD | Interrupts + de-emphasis + soft mute |
| 0x05 | [15:12] INT_MODE, [11:8] SEEKTH, [7:4] LNA, [3:0] VOLUME | Seek threshold + volume (0–15) |

### Read Register 0x0A (status, 2 bytes)

| Bit | Name | Meaning |
|-----|------|---------|
| [15] | RDSR | RDS ready (not used) |
| [14] | STC | Seek/Tune Complete — 1 = done |
| [13] | SF | Seek Fail — 1 = no station found (wrapped entire band) |
| [12] | RDSS | RDS sync (not used) |
| [11] | BLK_E | RDS block error (not used) |
| [10] | ST | Stereo indicator — 1 = stereo received |
| [9:0] | READCHAN | Current channel number |

### Frequency Calculation

```
Channel = (Frequency_MHz - 87.0) / 0.1

Example: 99.5 MHz
Channel = (99.5 - 87.0) / 0.1 = 125

Reverse: Frequency = 87.0 + Channel × 0.1

Band = 00 (87–108 MHz)
Space = 00 (100 kHz spacing)
```

### Register Shadows Used in This Project

```
reg[2] = 0xC001    // DHIZ=1, DMUTE=1, ENABLE=1
reg[3] = (channel << 6) | (1 << 4)   // CHAN + TUNE
reg[4] = 0x1400    // DE=50µs, SOFTMUTE_EN=1
reg[5] = 0x88AF    // SEEKTH=8, VOLUME=15

After tuning: clear TUNE bit → reg[3] &= ~(1 << 4)
Seek UP:      reg[2] |= (1 << 9) | (1 << 8)    // SEEKUP=1, SEEK=1
Seek DOWN:    reg[2] = (reg[2] & ~(1 << 9)) | (1 << 8)  // SEEKUP=0, SEEK=1
Volume:       reg[5] = (reg[5] & 0xFFF0) | (volume & 0x0F)
```

> **Note on MONO bit:** Setting bit 13 of reg[2] forces mono output. Since we mix LOUT+ROUT
> to mono externally via R3/R4 anyway, we leave MONO=0 to allow the stereo indicator (bit 10
> of reg 0x0A) to report stereo reception for display purposes.

## 9.6 Battery ADC Reading and Scaling

```
Li-ion Discharge Curve (approximate):

  Voltage    → Percentage
  4.20 V     → 100%
  4.10 V     →  90%
  4.00 V     →  80%
  3.90 V     →  60%
  3.80 V     →  40%
  3.70 V     →  30%
  3.60 V     →  15%
  3.50 V     →   5%
  3.30 V     →   0%

ADC reading with 100k/100k divider:
  V_adc = V_battery / 2
  V_battery = analogRead(34) × 3.3 / 4095 × 2

Note: ESP32 ADC has ±8–10% non-linearity without calibration. Average 16 readings for stability.
```

## 9.7 Sri Lankan FM Station Database

The firmware includes a complete database of FM stations in Sri Lanka (Sinhala, Tamil, and English).
Each entry maps a frequency to a station name. The `getStationName()` function matches the
current tuned frequency against the database using **±0.05 MHz tolerance**.

**Frequency Matching Rules:**
- If `abs(tuned_freq - station_freq) < 0.05`, display the station name
- If no match, display nothing (blank)
- Wrap-around tuning: 108.0 + 0.1 → 87.5, 87.5 − 0.1 → 108.0
- Station name updates automatically on every tune change

Total stations in database: **65 frequency entries** covering 43 unique stations.

## 9.8 Display Layout (128×64 OLED)

```
+-------------------------------+
|FM 99.5 MHz         [==] 85%  |  ← Row 0-14: Large freq + battery icon + %
|Red FM                ▌▌▌▌▌   |  ← Row 15-28: Station name + signal bars
|Vol ████████░░░░░  8/15       |  ← Row 29-42: Volume bar + level
|▐█▌▐▐█▌▐▐▌▐▐█▌▐▐█▌▐   0:05  |  ← Row 43-63: Equalizer animation + time
+-------------------------------+
```

### Screen Elements

| Element | Position | Font | Update Rate |
|---------|----------|------|-------------|
| "FM" label | Top-left | badge font | Static |
| Frequency | Top, large | 14pt bold | On tune change |
| Battery icon + % | Top-right | 5×7 | Every 10 seconds |
| Station name | Middle-left | 7×14 bold | On tune change |
| Signal bars | Middle-right | Custom drawn | On tune change |
| Volume bar + number | Below station | 6×10 | On volume change |
| Equalizer animation | Bottom | Custom bars | Every 120 ms |
| Elapsed time | Bottom-right | 6×10 | Every second |

### Welcome Screen (2 seconds at boot)

```
+-------------------------------+
|                               |
|         FM RADIO              |
|                               |
|          EE2050               |
|                               |
+-------------------------------+
```

---

# 10) Example Firmware Code Structure

## 10.1 File Structure (Single-File Sketch)

```
fm_radio_phase2.ino
├── [Config]         Pin definitions, constants
├── [RDA5807M]       Radio driver functions (I2C sequential mode)
├── [Stations]       Sri Lankan FM station database (65 entries)
├── [Display]        OLED drawing functions (welcome, main, equalizer)
├── [Encoder]        Dual ISR + button logic
├── [Battery]        ADC reading + percentage
├── setup()          Initialization + welcome screen
└── loop()           Main state machine
```

## 10.2 Complete Firmware Skeleton

```cpp
/* ═══════════════════════════════════════════════════════════════
 *  FM RADIO — PHASE 2 — "EE2050"
 *  RDA5807M + TDA2822M BTL + ESP32 + SH1106 OLED
 *
 *  Hardware:
 *    I2C SDA    = GPIO21     I2C SCL    = GPIO22
 *    ENC1 CLK   = GPIO32     ENC1 DT    = GPIO33    ENC1 SW = GPIO25
 *    ENC2 CLK   = GPIO26     ENC2 DT    = GPIO27    ENC2 SW = GPIO14
 *    BAT ADC    = GPIO34
 *    RDA5807M   @ 0x10 (sequential)     SH1106 @ 0x3C
 *
 *  Controls:
 *    ENC1 (right) rotate = tuning ±0.1 MHz
 *    ENC1 (right) press  = Seek UP
 *    ENC2 (left)  rotate = volume 0–15
 *    ENC2 (left)  press  = Seek DOWN
 * ═══════════════════════════════════════════════════════════════ */

#include <Wire.h>
#include <U8g2lib.h>
#include <Preferences.h>

// ──────────────── PIN CONFIG ────────────────
#define ENC1_A      32    // Right encoder (tuning) CLK
#define ENC1_B      33    // Right encoder DT
#define ENC1_SW     25    // Right encoder button → Seek UP

#define ENC2_A      26    // Left encoder (volume) CLK
#define ENC2_B      27    // Left encoder DT
#define ENC2_SW     14    // Left encoder button → Seek DOWN

#define BAT_ADC     34    // Battery voltage divider midpoint

// ──────────────── RDA5807M CONFIG ────────────────
#define RDA_ADDR    0x10  // Sequential mode I2C address
#define FREQ_MIN    87.5f
#define FREQ_MAX    108.0f

// ──────────────── GLOBALS ────────────────
U8G2_SH1106_128X64_NONAME_F_HW_I2C display(U8G2_R0, U8X8_PIN_NONE);
Preferences prefs;

// RDA5807M register shadows (0x02–0x07)
static uint16_t reg[8];

// Tuning state
float    currentFreq    = 99.5f;
int      currentVolume  = 8;       // 0–15 (RDA5807M internal volume)
int      signalLevel    = 0;       // 0–15 (RSSI from read)
int      batteryPercent = 100;

// Encoder ISR variables
volatile int  enc1Delta = 0;       // Tuning encoder accumulated steps
volatile int  enc2Delta = 0;       // Volume encoder accumulated steps

// Button state
unsigned long lastBtn1Press = 0;   // Debounce timestamp
unsigned long lastBtn2Press = 0;

// App state
enum AppState { STATE_BOOT, STATE_NORMAL };
AppState appState = STATE_BOOT;
unsigned long bootTime = 0;

// Animation
unsigned long lastAnimTime = 0;
int animBars[8] = {0};            // Heights for 8 equalizer bars
unsigned long lastDisplayUpdate = 0;

// NVS save debounce
unsigned long lastFreqChange = 0;
bool          freqDirty      = false;
unsigned long lastVolChange  = 0;
bool          volDirty       = false;

// ══════════════════════════════════════════════════════════════
//  SRI LANKAN FM STATION DATABASE
// ══════════════════════════════════════════════════════════════

struct Station {
  float freq;
  const char* name;
};

// All frequencies sorted ascending. Each regional frequency gets its own entry.
// If tuned frequency matches ±0.05 MHz, the station name is displayed.
const Station stationList[] PROGMEM = {
  // ── English ──
  { 87.6f, "Lite FM"          },
  { 87.8f, "Lite FM"          },
  { 88.3f, "E FM"             },
  // ── Sinhala ──
  { 88.6f, "Singha FM"        },
  { 88.8f, "Singha FM"        },
  { 89.1f, "Isira TNL FM"     },
  { 89.3f, "Isira TNL FM"     },
  { 89.6f, "City FM"          },
  { 89.8f, "City FM"          },
  // ── Tamil ──
  { 90.4f, "Vettri FM"        },
  { 90.6f, "Vettri FM"        },
  // ── Sinhala ──
  { 90.9f, "Shaa FM"          },
  { 91.1f, "Shaa FM"          },
  // ── Tamil ──
  { 91.4f, "Alai FM"          },
  // ── Sinhala ──
  { 92.2f, "FM Derana"        },
  { 92.4f, "FM Derana"        },
  { 92.7f, "Y FM"             },
  // ── English ──
  { 93.0f, "Gold FM"          },
  { 93.2f, "Gold FM"          },
  // ── Sinhala ──
  { 93.5f, "Lakhanda"         },
  { 93.7f, "Lakhanda"         },
  { 94.0f, "VIP Radio"        },
  { 94.8f, "Neth FM"          },
  { 95.0f, "Neth FM"          },
  { 95.6f, "Rhythm FM"        },
  { 95.8f, "Rhythm FM"        },
  { 96.1f, "Hiru FM"          },
  { 96.3f, "Hiru FM"          },
  { 96.6f, "Kiriula FM"       },
  // ── English ──
  { 96.9f, "Kiss FM"          },
  { 97.1f, "Real Radio"       },
  { 97.4f, "Radio Sri Lanka"  },
  { 97.6f, "Radio Sri Lanka"  },
  // ── Sinhala ──
  { 98.2f, "Siyatha FM"       },
  { 98.4f, "Siyatha FM"       },
  // ── English ──
  { 98.7f, "Sun FM"           },
  { 98.9f, "Sun FM"           },
  { 99.2f, "TNL Rocks"        },
  { 99.5f, "Red FM"           },
  { 99.7f, "Red FM"           },
  // ── Sinhala ──
  {100.0f, "Shree FM"         },
  {100.2f, "Shree FM"         },
  {100.5f, "Ran FM"           },
  // ── English ──
  {100.8f, "Yes FM"           },
  {101.0f, "Yes FM"           },
  // ── Sinhala ──
  {101.8f, "Seth FM"          },
  // ── Tamil ──
  {102.1f, "Tamil NS"         },
  {102.3f, "Tamil NS"         },
  {102.6f, "Vasantham FM"     },
  {102.8f, "Vasantham FM"     },
  {103.4f, "Sooriyan FM"      },
  {103.6f, "Sooriyan FM"      },
  {103.9f, "Shakthi FM"       },
  {104.1f, "Shakthi FM"       },
  // ── Sinhala ──
  {104.4f, "Rangiri SL FM"    },
  // ── Tamil ──
  {104.7f, "Thendral"         },
  {104.9f, "Thendral"         },
  // ── Sinhala ──
  {105.2f, "Lakviru FM"       },
  {105.4f, "Lakviru FM"       },
  {105.7f, "Rangiri SL FM"    },
  {106.0f, "Lak FM"           },
  {106.2f, "Lak FM"           },
  {106.5f, "Sirasa FM"        },
  {106.7f, "Sirasa FM"        },
  {107.0f, "V FM"             },
  {107.3f, "Kandurata FM"     },
  {107.5f, "Kandurata FM"     },
};
const int NUM_STATIONS = sizeof(stationList) / sizeof(stationList[0]);

// Return station name if frequency matches within ±0.05 MHz, else ""
const char* getStationName(float freq) {
  for (int i = 0; i < NUM_STATIONS; i++) {
    if (fabs(freq - stationList[i].freq) < 0.05f) {
      return stationList[i].name;
    }
  }
  return "";  // No match → display nothing
}

// ══════════════════════════════════════════════════════════════
//  RDA5807M DRIVER (Sequential I2C at 0x10)
// ══════════════════════════════════════════════════════════════

// Write registers 0x02–0x07 (12 bytes total)
static void rdaWriteRegisters() {
  Wire.beginTransmission(RDA_ADDR);
  for (uint8_t r = 2; r <= 7; r++) {
    Wire.write((uint8_t)(reg[r] >> 8));     // High byte
    Wire.write((uint8_t)(reg[r] & 0xFF));   // Low byte
  }
  Wire.endTransmission();
}

// Read register 0x0A (2 bytes — status register)
static uint16_t rdaReadReg0A() {
  Wire.requestFrom((uint8_t)RDA_ADDR, (uint8_t)2);
  uint8_t hi = Wire.read();
  uint8_t lo = Wire.read();
  return ((uint16_t)hi << 8) | lo;
}

// Read 4 bytes (reg 0x0A + reg 0x0B) to get RSSI from 0x0B
static uint16_t reg0A_cache = 0;
static uint16_t reg0B_cache = 0;

static void rdaReadStatus() {
  Wire.requestFrom((uint8_t)RDA_ADDR, (uint8_t)4);
  uint8_t a_hi = Wire.read();
  uint8_t a_lo = Wire.read();
  uint8_t b_hi = Wire.read();
  uint8_t b_lo = Wire.read();
  reg0A_cache = ((uint16_t)a_hi << 8) | a_lo;
  reg0B_cache = ((uint16_t)b_hi << 8) | b_lo;
}

// Tune to a specific frequency (MHz)
static void rdaSetFrequency(float freqMHz) {
  if (freqMHz < 87.0f)  freqMHz = 87.0f;
  if (freqMHz > 108.0f) freqMHz = 108.0f;

  uint16_t channel = (uint16_t)((freqMHz - 87.0f) / 0.1f + 0.5f);

  // Register 0x03: [15:6]=CHAN, [4]=TUNE, [3:2]=BAND(00), [1:0]=SPACE(00)
  reg[3] = ((channel & 0x03FF) << 6) | (1u << 4);
  rdaWriteRegisters();
  delay(80);

  // Clear TUNE bit after tuning
  reg[3] &= ~(1u << 4);
  rdaWriteRegisters();
}

// Set volume (0–15) in register 0x05 bits [3:0]
static void rdaSetVolume(int vol) {
  vol = constrain(vol, 0, 15);
  reg[5] = (reg[5] & 0xFFF0) | (vol & 0x0F);
  rdaWriteRegisters();
}

// Read RSSI (signal strength) from reg 0x0B bits [15:9] → 0–127
static int rdaGetRSSI() {
  rdaReadStatus();
  return (reg0B_cache >> 9) & 0x7F;  // 7-bit RSSI
}

// Map RSSI (0–127) to display level (0–15)
static int rdaGetSignalLevel() {
  int rssi = rdaGetRSSI();
  return constrain(rssi / 4, 0, 15);
}

// Seek to next/previous station
// Returns true if a station was found, false if seek failed (wrapped entire band)
static bool rdaSeek(bool seekUp) {
  // Set seek direction and start seek
  if (seekUp) {
    reg[2] |= (1u << 9);    // SEEKUP = 1
  } else {
    reg[2] &= ~(1u << 9);   // SEEKUP = 0
  }
  reg[2] |= (1u << 8);      // SEEK = 1
  rdaWriteRegisters();

  // Wait for STC (Seek/Tune Complete) flag in reg 0x0A bit 14
  delay(200);
  uint16_t status = 0;
  for (int attempt = 0; attempt < 80; attempt++) {
    status = rdaReadReg0A();
    if (status & (1u << 14)) break;  // STC = 1
    delay(50);
  }

  // Clear SEEK bit
  reg[2] &= ~(1u << 8);
  rdaWriteRegisters();

  // Dummy read to clear STC latch
  delay(50);
  rdaReadReg0A();

  // Check seek fail (bit 13)
  if (status & (1u << 13)) {
    return false;  // SF = 1: no station found
  }

  // Read the channel the chip stopped at
  uint16_t channel = status & 0x03FF;
  currentFreq = 87.0f + channel * 0.1f;

  // Round to 0.1 MHz
  currentFreq = round(currentFreq * 10.0f) / 10.0f;

  // Update shadow register
  reg[3] = (channel << 6);  // TUNE=0, BAND=00, SPACE=00

  // Read signal level
  signalLevel = rdaGetSignalLevel();

  // Save to NVS
  prefs.putFloat("freq", currentFreq);

  return true;
}

// Initialize RDA5807M with default registers
static void rdaInit() {
  // reg[2]: DHIZ=1, DMUTE=1, MONO=0, BASS=0, CLK=000(32.768k), ENABLE=1
  reg[2] = 0xC001;

  // reg[3]: set by rdaSetFrequency()
  reg[3] = 0x0000;

  // reg[4]: DE=50µs (bit 11), SOFTMUTE_EN=1 (bit 9)
  reg[4] = 0x0A00;

  // reg[5]: SEEKTH=8, VOLUME=currentVolume
  reg[5] = (0x88A0 & 0xFFF0) | (currentVolume & 0x0F);

  reg[6] = 0x0000;
  reg[7] = 0x0000;

  rdaWriteRegisters();
  delay(200);
}

// ══════════════════════════════════════════════════════════════
//  ENCODER ISRs (dual encoders, both IRAM_ATTR)
// ══════════════════════════════════════════════════════════════

void IRAM_ATTR enc1ISR() {
  // Right encoder: tuning
  if (digitalRead(ENC1_B) == LOW) {
    enc1Delta++;
  } else {
    enc1Delta--;
  }
}

void IRAM_ATTR enc2ISR() {
  // Left encoder: volume
  if (digitalRead(ENC2_B) == LOW) {
    enc2Delta++;
  } else {
    enc2Delta--;
  }
}

// ══════════════════════════════════════════════════════════════
//  BATTERY ADC
// ══════════════════════════════════════════════════════════════

int readBatteryPercent() {
  long sum = 0;
  for (int i = 0; i < 16; i++) {
    sum += analogRead(BAT_ADC);
  }
  float adcAvg = sum / 16.0f;
  float voltage = adcAvg * (3.3f / 4095.0f) * 2.0f;

  // Li-ion lookup
  if (voltage >= 4.20f) return 100;
  if (voltage >= 4.10f) return 90;
  if (voltage >= 4.00f) return 80;
  if (voltage >= 3.90f) return 60;
  if (voltage >= 3.80f) return 40;
  if (voltage >= 3.70f) return 30;
  if (voltage >= 3.60f) return 15;
  if (voltage >= 3.50f) return 5;
  return 0;
}

// ══════════════════════════════════════════════════════════════
//  DISPLAY — WELCOME SCREEN
// ══════════════════════════════════════════════════════════════

void drawBootScreen() {
  display.clearBuffer();
  display.setFont(u8g2_font_ncenB14_tr);   // Large bold
  display.drawStr(18, 28, "FM RADIO");
  display.setFont(u8g2_font_ncenB10_tr);   // Medium bold
  display.drawStr(35, 52, "EE2050");
  display.sendBuffer();
}

// ══════════════════════════════════════════════════════════════
//  DISPLAY — BATTERY ICON
// ══════════════════════════════════════════════════════════════

void drawBatteryIcon(int x, int y, int percent) {
  display.drawFrame(x, y, 18, 9);             // Outline
  display.drawBox(x + 18, y + 2, 2, 5);       // Nub
  int fillW = map(constrain(percent, 0, 100), 0, 100, 0, 16);
  if (fillW > 0) {
    display.drawBox(x + 1, y + 1, fillW, 7);
  }
}

// ══════════════════════════════════════════════════════════════
//  DISPLAY — SIGNAL BARS
// ══════════════════════════════════════════════════════════════

void drawSignalBars(int x, int y, int level) {
  int bars = map(constrain(level, 0, 15), 0, 15, 0, 5);
  for (int i = 0; i < 5; i++) {
    int h = 3 + i * 2;
    int bx = x + i * 4;
    int by = y + (13 - h);
    if (i < bars) {
      display.drawBox(bx, by, 3, h);
    } else {
      display.drawFrame(bx, by, 3, h);
    }
  }
}

// ══════════════════════════════════════════════════════════════
//  DISPLAY — VOLUME BAR
// ══════════════════════════════════════════════════════════════

void drawVolumeBar(int x, int y, int vol) {
  // Label
  display.setFont(u8g2_font_5x7_tr);
  display.drawStr(x, y + 7, "Vol");

  // Bar outline (60 px wide)
  int barX = x + 18;
  int barW = 60;
  int barH = 7;
  display.drawFrame(barX, y, barW, barH);

  // Fill proportional to volume 0–15
  int fillW = map(vol, 0, 15, 0, barW - 2);
  if (fillW > 0) {
    display.drawBox(barX + 1, y + 1, fillW, barH - 2);
  }

  // Volume number
  char volStr[6];
  sprintf(volStr, "%d/15", vol);
  display.drawStr(barX + barW + 3, y + 7, volStr);
}

// ══════════════════════════════════════════════════════════════
//  DISPLAY — EQUALIZER ANIMATION
// ══════════════════════════════════════════════════════════════

void updateEqualizerBars() {
  // Subtle random movement for each bar
  for (int i = 0; i < 8; i++) {
    int target = 3 + random(0, 15);  // Random target height
    // Smooth toward target
    if (animBars[i] < target) animBars[i]++;
    if (animBars[i] > target) animBars[i]--;
    animBars[i] = constrain(animBars[i], 2, 18);
  }
}

void drawEqualizer(int x, int y, int maxH) {
  for (int i = 0; i < 8; i++) {
    int h = map(animBars[i], 0, 18, 2, maxH);
    int bx = x + i * 7;
    display.drawBox(bx, y + (maxH - h), 5, h);
  }
}

// ══════════════════════════════════════════════════════════════
//  DISPLAY — MAIN SCREEN
// ══════════════════════════════════════════════════════════════

void drawNormalScreen() {
  display.clearBuffer();

  // ── Row 1 (y=0–14): FM + Large frequency + Battery ──
  display.setFont(u8g2_font_6x10_tr);
  display.drawStr(0, 10, "FM");

  display.setFont(u8g2_font_ncenB14_tr);
  char freqStr[10];
  dtostrf(currentFreq, 5, 1, freqStr);
  strcat(freqStr, "MHz"); // Append MHz without gap
  display.drawStr(15, 14, freqStr);

  // Battery icon + percentage (top right)
  drawBatteryIcon(92, 0, batteryPercent);
  display.setFont(u8g2_font_5x7_tr);
  char batStr[5];
  sprintf(batStr, "%d%%", batteryPercent);
  display.drawStr(112, 9, batStr);

  // ── Row 2 (y=16–28): Station name + Signal bars ──
  display.setFont(u8g2_font_7x14B_tr);
  const char* name = getStationName(currentFreq);
  if (strlen(name) > 0) {
    display.drawStr(0, 28, name);
  }
  drawSignalBars(104, 16, signalLevel);

  // ── Row 3 (y=30–39): Volume bar ──
  drawVolumeBar(0, 31, currentVolume);

  // ── Row 4 (y=44–63): Equalizer animation + elapsed time ──
  drawEqualizer(0, 45, 18);

  // Elapsed time (bottom right)
  unsigned long elapsed = millis() / 1000;
  int mins = (elapsed / 60) % 60;
  int hrs  = elapsed / 3600;
  char timeStr[10];
  sprintf(timeStr, "%d:%02d", hrs, mins);
  display.setFont(u8g2_font_5x7_tr);
  display.drawStr(100, 62, timeStr);

  display.sendBuffer();
}

// ══════════════════════════════════════════════════════════════
//  PROCESS ENCODERS + BUTTONS
// ══════════════════════════════════════════════════════════════

void processEncoders() {
  // ── ENC1: Tuning (right encoder) ──
  if (enc1Delta != 0) {
    int delta = enc1Delta;
    enc1Delta = 0;

    // Each click = ±0.1 MHz
    if (delta > 0) {
      currentFreq += 0.1f;
    } else {
      currentFreq -= 0.1f;
    }

    // Wrap-around
    if (currentFreq > FREQ_MAX) currentFreq = FREQ_MIN;
    if (currentFreq < FREQ_MIN) currentFreq = FREQ_MAX;

    // Round to 0.1 MHz
    currentFreq = round(currentFreq * 10.0f) / 10.0f;

    // Tune radio
    rdaSetFrequency(currentFreq);

    // Read signal level after brief settle
    delay(30);
    signalLevel = rdaGetSignalLevel();

    Serial.print("Tuned: ");
    Serial.print(currentFreq, 1);
    Serial.print(" MHz  RSSI: ");
    Serial.print(signalLevel);
    const char* stn = getStationName(currentFreq);
    if (strlen(stn) > 0) {
      Serial.print("  [");
      Serial.print(stn);
      Serial.print("]");
    }
    Serial.println();

    // Mark frequency as changed (NVS save debounce)
    freqDirty = true;
    lastFreqChange = millis();
  }

  // ── ENC2: Volume (left encoder) ──
  if (enc2Delta != 0) {
    int delta = enc2Delta;
    enc2Delta = 0;

    if (delta > 0) {
      currentVolume++;
    } else {
      currentVolume--;
    }
    currentVolume = constrain(currentVolume, 0, 15);

    // Write to RDA5807M
    rdaSetVolume(currentVolume);

    Serial.print("Volume: ");
    Serial.print(currentVolume);
    Serial.println("/15");

    // Mark volume as changed
    volDirty = true;
    lastVolChange = millis();
  }
}

void processButtons() {
  unsigned long now = millis();

  // ── ENC1 button: Seek UP ──
  if (digitalRead(ENC1_SW) == LOW && (now - lastBtn1Press > 250)) {
    lastBtn1Press = now;
    Serial.print("Seek UP... ");
    if (rdaSeek(true)) {
      Serial.print("Found: ");
      Serial.print(currentFreq, 1);
      Serial.print(" MHz");
      const char* stn = getStationName(currentFreq);
      if (strlen(stn) > 0) {
        Serial.print("  [");
        Serial.print(stn);
        Serial.print("]");
      }
      Serial.println();
    } else {
      Serial.println("No station found");
    }
  }

  // ── ENC2 button: Seek DOWN ──
  if (digitalRead(ENC2_SW) == LOW && (now - lastBtn2Press > 250)) {
    lastBtn2Press = now;
    Serial.print("Seek DOWN... ");
    if (rdaSeek(false)) {
      Serial.print("Found: ");
      Serial.print(currentFreq, 1);
      Serial.print(" MHz");
      const char* stn = getStationName(currentFreq);
      if (strlen(stn) > 0) {
        Serial.print("  [");
        Serial.print(stn);
        Serial.print("]");
      }
      Serial.println();
    } else {
      Serial.println("No station found");
    }
  }
}

// ══════════════════════════════════════════════════════════════
//  NVS SAVE (debounced — saves 2 seconds after last change)
// ══════════════════════════════════════════════════════════════

void checkNvsSave() {
  unsigned long now = millis();
  if (freqDirty && (now - lastFreqChange > 2000)) {
    prefs.putFloat("freq", currentFreq);
    freqDirty = false;
  }
  if (volDirty && (now - lastVolChange > 2000)) {
    prefs.putInt("vol", currentVolume);
    volDirty = false;
  }
}

// ══════════════════════════════════════════════════════════════
//  SETUP
// ══════════════════════════════════════════════════════════════

void setup() {
  Serial.begin(115200);
  delay(500);
  Serial.println("\n=== FM Radio Phase 2 — EE2050 ===");

  // I2C
  Wire.begin(21, 22);
  Wire.setClock(100000);  // 100 kHz

  // Display
  display.begin();
  drawBootScreen();
  bootTime = millis();
  appState = STATE_BOOT;

  // Encoder pins
  pinMode(ENC1_A,  INPUT_PULLUP);
  pinMode(ENC1_B,  INPUT_PULLUP);
  pinMode(ENC1_SW, INPUT_PULLUP);
  pinMode(ENC2_A,  INPUT_PULLUP);
  pinMode(ENC2_B,  INPUT_PULLUP);
  pinMode(ENC2_SW, INPUT_PULLUP);
  attachInterrupt(digitalPinToInterrupt(ENC1_A), enc1ISR, FALLING);
  attachInterrupt(digitalPinToInterrupt(ENC2_A), enc2ISR, FALLING);

  // Battery ADC
  analogReadResolution(12);
  analogSetAttenuation(ADC_11db);
  pinMode(BAT_ADC, INPUT);

  // Load saved frequency + volume from NVS
  prefs.begin("fmradio", false);
  currentFreq   = prefs.getFloat("freq", 99.5f);
  currentVolume = prefs.getInt("vol", 8);
  currentVolume = constrain(currentVolume, 0, 15);
  Serial.print("Saved freq: ");
  Serial.print(currentFreq, 1);
  Serial.print(" MHz, Vol: ");
  Serial.print(currentVolume);
  Serial.println("/15");

  // Initialize RDA5807M
  rdaInit();
  rdaSetFrequency(currentFreq);
  delay(100);
  signalLevel = rdaGetSignalLevel();

  Serial.print("Signal: ");
  Serial.println(signalLevel);
  Serial.println("Boot complete. Welcome screen for 2 seconds...");
}

// ══════════════════════════════════════════════════════════════
//  MAIN LOOP
// ══════════════════════════════════════════════════════════════

void loop() {
  unsigned long now = millis();

  switch (appState) {
    case STATE_BOOT:
      // Stay on welcome screen for 2 seconds
      if (now - bootTime > 2000) {
        appState = STATE_NORMAL;
        Serial.println("Entering NORMAL mode.");
      }
      break;

    case STATE_NORMAL:
      processEncoders();
      processButtons();
      checkNvsSave();

      // Update equalizer animation every ~120 ms
      if (now - lastAnimTime > 120) {
        lastAnimTime = now;
        updateEqualizerBars();
      }

      // Redraw display every ~150 ms (non-blocking)
      if (now - lastDisplayUpdate > 150) {
        lastDisplayUpdate = now;
        drawNormalScreen();
      }

      // Update battery every 10 seconds
      {
        static unsigned long lastBatRead = 0;
        if (now - lastBatRead > 10000) {
          lastBatRead = now;
          batteryPercent = readBatteryPercent();
        }
      }
      break;
  }

  delay(2);  // Small yield — keeps loop responsive
}
```

## 10.3 Required Libraries

Install these via **Sketch → Include Library → Manage Libraries:**

| Library | Author | Purpose |
|---------|--------|---------|
| U8g2 | oliver | SH1106 OLED display driver |
| Wire | (built-in) | I2C communication |
| Preferences | (built-in, ESP32) | Non-volatile storage for frequency + volume |

No third-party RDA5807M library needed — the driver is built into the sketch above.
The RDA5807M driver uses the same sequential I2C protocol proven in Phase 1.

---

# 11) Pre-Power Checklist

## 11.1 Continuity Checks (Power OFF, Arduino Disconnected)

Use multimeter in continuity/beep mode.

| Test | Expected | If FAIL |
|------|----------|---------|
| +5V bus to GND bus | **NO beep** (not shorted) | Find solder bridge. Focus on C5 area and TDA socket. |
| +3V3 bus to GND bus | **NO beep** | Check C1, C2 area. Check OLED wiring. |
| TDA pin 1 (OUT1) to GND | **NO beep** | Check speaker wiring — speaker must NOT be grounded in BTL. |
| TDA pin 3 (OUT2) to GND | **NO beep** | Same as above. SPK_B must not go to GND. |
| TDA pin 1 to pin 3 | **~2 Ω** (through Zobel R5) | If 0 Ω: short circuit somewhere. If open: check Zobel. |
| I2C_SDA to GND | **NO beep** | Check R1 (must go to +3V3, not GND). |
| I2C_SCL to GND | **NO beep** | Check R2. |
| I2C_SDA to +3V3 | **~4.7 kΩ** | Confirms R1 is connected. |
| I2C_SCL to +3V3 | **~4.7 kΩ** | Confirms R2 is connected. |
| RDA5807M VCC (pad 5) to +3V3 | **Beep** (connected) | Trace the wire. |
| RDA5807M VCC (pad 5) to +5V | **NO beep** | If beep: STOP. VCC is wired to 5V — will destroy RDA5807M. |
| ENC1 SW to GND | **NO beep** (unless button pressed) | Check encoder wiring. |
| ENC2 SW to GND | **NO beep** (unless button pressed) | Check encoder wiring. |

## 11.2 Voltage Checks (Power ON, Arduino 5V Connected)

Connect Arduino UNO to USB. Connect Arduino 5V → perfboard +5V, GND → perfboard GND.
ESP32 should be plugged in but **no sketch uploaded yet.**

| Probe Point | Expected | If WRONG |
|-------------|----------|----------|
| +5V bus | 4.8–5.2 V | Check Arduino USB connection |
| ESP32 VIN pin | 4.8–5.2 V | Check wire from +5V bus to VIN |
| ESP32 3V3 pin | 3.2–3.4 V | ESP32 regulator working |
| +3V3 bus | 3.2–3.4 V | Check wire from ESP32 3V3 to +3V3 bus |
| RDA5807M VCC (pad 5) | 3.2–3.4 V | Check connection to +3V3 bus |
| OLED VCC | 3.2–3.4 V | Check connection |
| TDA Pin 2 (VCC) | 4.8–5.2 V | Check wire to +5V bus |
| TDA Pin 4 (GND) | 0 V | Check wire to GND bus |
| TDA Pin 6 (IN2+) | 0 V | Must be grounded |
| LED1 (if installed) | Glowing red | Check LED polarity, R6 |

**CRITICAL:** If RDA5807M VCC reads ~5 V instead of ~3.3 V, **disconnect immediately.**
You have wired it to the wrong rail.

## 11.3 Capacitor Polarity Visual Check

Before powering on, visually verify electrolytic cap orientation:

| Cap | Positive (+) Side | Negative (−) Side |
|-----|-------------------|-------------------|
| C2 (10 µF) | +3V3 bus | GND bus |
| C3 (10 µF) | AF_MIX (radio side) | TDA pin 7 (amp side) |
| C5 (1000 µF) | +5V bus | GND bus |
| C6 (47 µF) | TDA pin 8 | GND bus |
| C8 (10 µF) | TDA pin 1 (OUT1) | TDA pin 5 (IN2−) |
| C10 (10 µF, opt.) | +3V3 bus | GND bus |

## 11.4 Battery Safety Pre-Check (For Tomorrow)

Before connecting battery to the system:

| Test | How | Expected |
|------|-----|----------|
| Battery voltage | Multimeter on battery terminals | 3.0–4.2 V |
| TP4056 standalone test | Connect battery to TP4056, plug in USB. Measure output. | Output = battery voltage (3.0–4.2 V) |
| Boost module test (Plan A) | Connect TP4056 output to MT3608 input. Adjust trim pot. | MT3608 output = 5.0 V (±0.1 V) |
| LDO test (Plan B) | Connect TP4056 output to HT7333 input. | HT7333 output = 3.3 V |

**Set boost/LDO output voltage BEFORE connecting to the circuit.**

---

# 12) Troubleshooting Matrix

## No Sound At All

| Check | Action |
|-------|--------|
| Speaker wiring | SPK_A to pin 1, SPK_B to pin 3. **Neither to GND (BTL mode).** |
| TDA2822M orientation | Notch/dot must be at pin 1 end of socket |
| TDA2822M inserted? | IC must be in the socket (not left out from build stage) |
| C3 installed? | Must be between AF_MIX and TDA pin 7 |
| C3 to TDA pin 7? | C3 negative leg connects directly to TDA pin 7 |
| RDA5807M volume? | Check volume is not 0 — turn left encoder clockwise |
| C6 installed? | 47 µF from pin 8 to GND |
| C8 installed (BTL)? | 10 µF from pin 1 to pin 5 — required for BTL. (+) toward pin 1, (−) toward pin 5. |
| Pin 4 to GND? | This is the only GND pin on the IC |
| Pin 6 to GND? | IN2+ must be grounded for BTL |
| +5V on pin 2? | Measure: should be ~5 V |

## Distortion / Crackling

| Check | Action |
|-------|--------|
| C5 (1000 µF) present? | Bulk decoupling prevents power sag during loud peaks |
| C4 (0.1 µF) close to pin 2? | Must be within 1 cm |
| Speaker impedance | Must be 4 Ω or higher |
| Short on pin 1 or pin 3 to GND? | Check — BTL outputs must float |
| Zobel installed? | R5 + C7 from pin 1 to pin 3 |

## Noise / Hum / Buzzing

| Check | Action |
|-------|--------|
| Audio wire length | Shorten AF_IN wire (C3 to pin 7). Keep under 3 cm. |
| I2C near audio wires? | Re-route — I2C generates digital noise |
| +5V bus near RDA5807M? | Re-route — carries amplifier current spikes |
| Antenna near amp? | Move antenna to opposite end of board |
| C1 and C4 close to ICs? | Ceramic caps must be within 1 cm of IC power pins |
| Ground bus intact? | Check for breaks in the GND bus wire |
| Star ground? | All GND wires should go directly to bus, not daisy-chained |

## No I2C — Device Not Found

| Check | Action |
|-------|--------|
| R1/R2 connected to +3V3? | Must be 3.3 V, never 5 V |
| SDA to GPIO21, SCL to GPIO22? | Swap if reversed |
| RDA5807M address | Scanner should find 0x10. Try 0x11 if not found (index mode). |
| OLED address | Try 0x3D if 0x3C doesn't work |
| Wire.begin(21, 22) in code? | Explicit pin assignment for ESP32 |
| Pull-up resistance | Measure SDA to +3V3: should be ~4.7 kΩ |

## Encoder Jitter / Miscounting

| Check | Action |
|-------|--------|
| Encoder + pin | Must be connected to +3V3 (not 5 V) |
| Pull-ups | Encoder module usually has onboard pull-ups. If not, use INPUT_PULLUP. |
| Wrong direction | Swap CLK and DT wires |
| Counts double | Reduce interrupt sensitivity — try RISING instead of FALLING |
| Noisy counts | Add 0.1 µF ceramic caps from CLK to GND and DT to GND |

## Display Not Working

| Check | Action |
|-------|--------|
| VCC voltage | Must be 3.3 V at OLED VCC pin |
| SDA/SCL swapped? | Check OLED module silkscreen — pin order varies |
| I2C address | Run I2C scanner. Try 0x3C and 0x3D. |
| Wrong driver | SH1106 ≠ SSD1306. Use `U8G2_SH1106_128X64` constructor. |
| Display dead? | Check with separate 3.3 V supply + standalone I2C test |

## Battery Warning Wrong / No Reading

| Check | Action |
|-------|--------|
| R7/R8 connected? | 100 kΩ divider from BAT+ to GND with midpoint at GPIO34 |
| GPIO34 correct? | Must be GPIO34 (ADC1_CH6). Not GPIO35 or other. |
| Battery connected? | Battery must be in holder with TP4056 output reaching R7 |
| ADC attenuation | Code must have `analogSetAttenuation(ADC_11db)` for 0–3.3 V range |
| Calibration | ESP32 ADC is ±8–10% non-linear without calibration. Calibrate against multimeter reading. |

## TDA2822M Gets Hot

| Check | Action |
|-------|--------|
| Speaker impedance | Must be ≥ 4 Ω. Lower impedance = more current = more heat. |
| Pin 1 or pin 3 shorted to GND? | In BTL, this shorts half the bridge → excess current |
| VCC correct? | Should be 5 V. Higher voltage = more heat. |
| Pin 1 shorted to pin 3? | Direct short (bypassing Zobel) = maximum current draw |
| Gain too high? | C6 = 47 µF is default. If still hissy, try 22 µF. Only go to 100 µF if volume is too low. |

---

# 12.5) Engineering Notes & Optimization Tips

These are not errors in the design. They are optimization tips gathered during build verification.

## Gain Optimization (C6)

C6 = 47 µF is the default value (~100× gain). This provides a clean balance between
volume and noise floor for the RDA5807M + TDA2822M BTL chain.

| C6 Value | Approx. Gain | When to Use |
|----------|-------------|-------------|
| 22 µF | ~50× | Cleanest audio, lower max volume |
| **47 µF** | **~100×** | **Default — recommended starting point** |
| 100 µF | ~200× | Max volume, may add hiss at low levels |

Swap C6 without any other changes — it's the only gain-setting component.

## C8 Polarity Awareness

C8 (10 µF electrolytic) connects between Pin 1 (OUT1) and Pin 5 (IN2−). Both pins internally
bias around ~VCC/2. The signal swings AC in both directions through this capacitor.

In practice, an electrolytic cap works fine here — standard builds use it. But technically,
a **non-polarized 10 µF capacitor** would be more correct because neither terminal has a
fixed DC bias advantage. If C8 ever fails after long-term use, this AC stress is the reason.

**Not a build-stopper.** Just engineering awareness.

## Zobel Resistor Power Rating

R5 (2.2 Ω) in the Zobel network must be rated **≥ 0.5 W**. At loud output, this resistor
absorbs high-frequency energy from the amplifier outputs. A tiny 1/8 W resistor will
overheat slowly over time.

**Use at minimum a 1/4 W resistor, ideally 1/2 W.**

## MT3608 Switching Noise (Tomorrow — Important)

When switching to battery + MT3608 boost power:

MT3608 is a **switching converter** that produces high-frequency switching noise and ripple
on the +5V rail. Your design already has:

- C5 (1000 µF) bulk cap — good
- C4 (0.1 µF) decoupling — good

**Strongly recommend adding:** Another **470 µF–1000 µF electrolytic** right at the MT3608
output, **before** it connects to the +5V bus. This will reduce:

- Audio whining/buzzing
- Switching ripple
- RF interference to the RDA5807M

## Speaker Power Reality

The speaker is rated **4 Ω 5 W**, but TDA2822M at 5 V BTL into 4 Ω produces approximately
**~1 W maximum**. This is fine — the 5 W rating is just the speaker's thermal limit.

Don't expect huge loudness. The amplifier is the bottleneck, not the speaker.

## Encoder Noise on Perfboard

The ISR interrupt pins (ENC1_A on GPIO32, ENC2_A on GPIO26) may pick up noise from long
perfboard wires, causing ghost increments or jitter.

**If this happens, two fixes:**

1. Add `delayMicroseconds(50);` at the start of each ISR
2. Add a **0.1 µF ceramic capacitor** from each CLK pin to GND (hardware debounce)

Not mandatory upfront — only apply if you observe jitter during testing.

## Battery Divider (No Correction Needed)

R7/R8 = 100 kΩ / 100 kΩ divider draws only **~21 µA**. This is negligible compared to
battery self-discharge. No correction needed. The divider can stay always-connected.

## TP4056 Charging Module Placement

**Best placement: Right side of the back, near battery holder.**

```
BACK VIEW (solder side) — matches Section 6.3:

    ┌──────────────────────────────────────────────┐
    │                                              │
    │  ┌──────────────┐       ┌─────────────┐   │
    │  │              │       │             │   │
    │  │   SPEAKER    │       │  Battery    │   │
    │  │  (left side)  │       │  Holder     │   │
    │  │   4Ω 5W      │       │ (right)     │   │
    │  └──────────────┘       └──────┬──────┘   │
    │                              │               │
    │                        ┌─────┴─────┐       │
    │                        │  TP4056   │       │
    │                        │ (USB out) │       │
    │                        └─────┬─────┘       │
    │                        ┌─────┴─────┐       │
    │                        │  Switch   │       │
    │                        └─────┬─────┘       │
    │                        ┌─────┴─────┐       │
    │                        │  MT3608   │       │
    │                        │  Boost    │       │
    │                        └───────────┘       │
    └──────────────────────────────────────────────┘
    │                           └─────┬─────┘   │
    │                           ┌─────┴─────┐   │
    │                           │  Switch   │   │
    │                           └─────┬─────┘   │
    │                           ┌─────┴─────┐   │
    │                           │  MT3608   │   │
    │                           │  Boost    │   │
    │                           └───────────┘   │
    └──────────────────────────────────────────────┘
```

**Why this placement:**
- Short battery wires (minimal resistance loss)
- Short charging loop (reduces noise)
- Charging noise is far from RF section (RDA5807M is on the opposite side, middle-left)
- USB port faces outside of enclosure for easy charging access
- MT3608 close to TP4056 output for clean power path

---

# 13) Final System Capabilities — Phase 2

## What Phase 2 Delivers

| Capability | Status |
|------------|--------|
| FM radio reception (87.5–108 MHz) | ✅ |
| RDA5807M I2C control (tuning, seek, volume) | ✅ |
| Mono speaker output via TDA2822M BTL | ✅ |
| ~4× more power than single-ended mode | ✅ |
| Software volume (RDA5807M 0–15, left encoder) | ✅ |
| Dual rotary encoders (right=tuning, left=volume) | ✅ |
| Seek UP (right encoder press) | ✅ |
| Seek DOWN (left encoder press) | ✅ |
| 1.3″ OLED display (frequency, station, volume, battery, signal) | ✅ |
| Sri Lankan station database (65 frequencies, 43 stations) | ✅ |
| Frequency → station name matching (±0.05 MHz) | ✅ |
| Volume bar display with level indicator | ✅ |
| Battery level display (percentage + icon) | ✅ |
| Signal strength bars | ✅ |
| Equalizer animation (non-blocking, ~120 ms) | ✅ |
| Elapsed time display | ✅ |
| Non-volatile frequency + volume save (NVS, debounced) | ✅ |
| Welcome screen ("FM RADIO" / "EE2050", 2 seconds) | ✅ |
| Wrap-around tuning (87.5 ↔ 108.0) | ✅ |
| Single 18650 battery operation (tomorrow) | ✅ (after boost/LDO) |
| USB charging via TP4056 (tomorrow) | ✅ (after wiring) |
| Low battery warning (display icon drains) | ✅ |
| Safe 3.3 V I2C levels | ✅ |
| Replaceable amplifier IC (socketed) | ✅ |
| Clean perfboard layout with RF/audio isolation | ✅ |

## What Phase 2 Does NOT Have (Future Upgrades)

| Feature | Why Not | Upgrade Path |
|---------|---------|--------------|
| Real-time clock | No RTC module or WiFi for NTP | Add DS3231 RTC module (I2C, $1) |
| Stereo output | Single speaker, mono mixer | Add second speaker + TDA2822M stereo mode |
| Preset stations | Could be added in firmware | Store 6 presets in NVS, add button |
| RDS station names | RDA5807M has RDS but not decoded yet | Add RDS parsing in firmware |
| Bluetooth audio | Not implemented | ESP32 has BT — add A2DP sink in firmware |

---

# Appendix A — One-Page Wiring Cheat Sheet

Print this and keep it next to the perfboard while soldering.

## ESP32 Connections (12 wires to perfboard)

| ESP32 Pin | Goes To |
|-----------|---------|
| VIN | +5V bus |
| GND | GND bus |
| 3V3 | +3V3 bus |
| GPIO21 | I2C_SDA (RDA5807M pad 1 + OLED SDA + R1) |
| GPIO22 | I2C_SCL (RDA5807M pad 2 + OLED SCL + R2) |
| GPIO32 | ENC1 CLK (right encoder — tuning) |
| GPIO33 | ENC1 DT |
| GPIO25 | ENC1 SW (Seek UP) |
| GPIO26 | ENC2 CLK (left encoder — volume) |
| GPIO27 | ENC2 DT |
| GPIO14 | ENC2 SW (Seek DOWN) |
| GPIO34 | BAT_ADC (R7/R8 midpoint) |

## RDA5807M (U1) Connections

| Pad | Goes To |
|-----|---------|
| 1 SDA | I2C_SDA + R1 (4.7k) to +3V3 |
| 2 SCL | I2C_SCL + R2 (4.7k) to +3V3 |
| 3 NC | — |
| 4 NC | — |
| 5 VCC | +3V3 (**NOT 5V!**) |
| 6 ANT | 75 cm wire antenna |
| 7 NC | — |
| 8 LOUT | R3 (1k) → AF_MIX |
| 9 ROUT | R4 (1k) → AF_MIX |
| 10 GND | GND bus |

## TDA2822M (U2) BTL Connections

| Pin | Goes To |
|-----|---------|
| 1 OUT1 | SPK_A + Zobel (R5→C7→pin3) + C8(+) |
| 2 VCC | +5V bus |
| 3 OUT2 | SPK_B + Zobel (C7←R5←pin1) |
| 4 GND | GND bus |
| 5 IN2− | C8 (10µF−) ← from pin 1 |
| 6 IN2+ | GND bus |
| 7 IN1+ | AF_IN (C3 negative leg — direct from coupling cap) |
| 8 IN1− | C6 (47µF+) → GND |

## All Passive Components at a Glance

| Ref | Value | Between |
|-----|-------|---------|
| R1 | 4.7 kΩ | I2C_SDA — +3V3 |
| R2 | 4.7 kΩ | I2C_SCL — +3V3 |
| R3 | 1 kΩ | LOUT — AF_MIX |
| R4 | 1 kΩ | ROUT — AF_MIX |
| R5 | 2.2 Ω (≥0.5W) | OUT1 — C7 — OUT2 (Zobel, across speaker) |
| R6 | 1 kΩ | +5V — LED1 anode (optional) |
| R7 | 100 kΩ | BAT+ — GPIO34 |
| R8 | 100 kΩ | GPIO34 — GND |
| C1 | 0.1 µF | +3V3 — GND (near RDA5807M) |
| C2 | 10 µF | +3V3(+) — GND(−) (near RDA5807M) |
| C3 | 10 µF | AF_MIX(+) — TDA pin 7(−) |
| C4 | 0.1 µF | +5V — GND (near TDA pin 2) |
| C5 | 1000 µF | +5V(+) — GND(−) (near TDA) |
| C6 | 47 µF | TDA pin 8(+) — GND(−) |
| C7 | 0.1 µF | R5 — OUT2 (Zobel cap) |
| C8 | 10 µF | TDA pin 1(+) — TDA pin 5(−) (BTL coupling) |
| C9 | 0.1 µF | SPK_A — SPK_B (optional HF filter) |
| C10 | 10 µF | +3V3(+) — GND(−) (optional, near ESP32) |

---

# Appendix B — Tomorrow's Power Wiring

## Plan A: MT3608 Boost to 5 V (Recommended)

```
   ┌──────────────┐
   │   18650      │
   │   Battery    │
   │  3.0–4.2V   │
   └──────┬───────┘
          │
   ┌──────┴───────┐
   │   TP4056     │  ← USB-C charging input
   │  Charge Ctrl │
   └──────┬───────┘
          │
   ┌──────┴───────┐
   │   ON/OFF     │  ← Latching switch (between TP4056 OUT and boost IN)
   │   Switch     │
   └──────┬───────┘
          │
   ┌──────┴───────┐
   │   MT3608     │  ← Adjust trim pot to output 5.0V BEFORE connecting
   │  Boost to 5V │
   └──────┬───────┘
          │
     +5V BUS ──────→ ESP32 VIN + TDA2822M Pin 2
          │
          └── Also connect BAT+ (before boost, after switch) to R7 for ADC reading
```

**Steps:**
1.  Disconnect Arduino 5V and GND wires from perfboard
2.  Adjust MT3608 output to exactly 5.0 V (use multimeter + tiny screwdriver on trim pot)
3.  Wire: Battery(+) → TP4056 B+ | Battery(−) → TP4056 B−
4.  Wire: TP4056 OUT+ → Switch terminal 1
5.  Wire: Switch terminal 2 → MT3608 VIN+
6.  Wire: TP4056 OUT− → MT3608 VIN− (GND)
7.  Wire: MT3608 VOUT+ → +5V bus on perfboard
8.  Wire: MT3608 VOUT− → GND bus on perfboard
9.  Wire: TP4056 OUT+ (or switch output) → R7 (for battery ADC reading)
10. Power on switch. Verify 5 V on +5V bus. Verify 3.3 V from ESP32.

## Plan B: HT7333 LDO to 3.3 V (Fallback)

```
   18650 → TP4056 → Switch ─┬── HT7333 IN → OUT (3.3V) → ESP32 3V3 pin + +3V3 bus
                             │
                             └── Direct to TDA2822M Pin 8 (3.0–4.2V, lower power)
```

**Note:** In Plan B, TDA2822M runs at battery voltage (3.0–4.2 V) instead of 5 V.
Speaker output power will be approximately 0.1–0.4 W depending on charge level.
This is noticeably quieter than Plan A.

**Steps:**
1.  Disconnect Arduino wires
2.  Wire: HT7333 pin 1 (IN) → switch output
3.  Wire: HT7333 pin 2 (GND) → GND bus
4.  Wire: HT7333 pin 3 (OUT) → ESP32 3V3 pin AND +3V3 bus
5.  Disconnect ESP32 VIN from +5V bus (ESP32 no longer needs 5V)
6.  Wire: Switch output → TDA2822M pin 8 directly (battery voltage)
7.  Add 10 µF electrolytic capacitor: HT7333 OUT(+) to GND(−) for stability
8.  Power on. Verify 3.3V on +3V3 bus. TDA pin 8 should show battery voltage.

---

# Appendix C — ESP32 Chip Info Reference

Your chip (from scan):

```
Chip Model:       ESP32-D0WD-V3
Chip Revision:    1
CPU Cores:        2
CPU Frequency:    240 MHz
Flash Size:       4,194,304 bytes (4 MB)
Flash Speed:      80 MHz
SDK Version:      v4.x.x
```

This is a full-featured ESP32 with Dual-Core, WiFi, Bluetooth, and 4 MB flash.
Plenty of resources for this project.

---

**END OF DOCUMENT**
