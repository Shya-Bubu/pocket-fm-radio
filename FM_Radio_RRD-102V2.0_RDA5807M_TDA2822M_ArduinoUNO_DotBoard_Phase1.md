# FM Radio — Dot Board Phase-1 "No Regrets" Build

## RRD-102V2.0 (RDA5807M) + TDA2822M + Arduino UNO + 4-Ohm Speaker

**Revision:** 3.5 (added C9 AF_MIX RF filter, C10 Arduino 3V3 decoupling, SPK− warning emphasis)
**Date:** 2026-02-22

---

**Design choices locked (Phase-0):**

| Decision | Choice |
|----------|--------|
| Power | Arduino USB 5V |
| I2C safety | External pull-ups to 3.3V (4.7k preferred; 10k OK) |
| Mono audio | Mix LOUT+ROUT via 2x 1k resistors |
| Amplifier mode | Single-ended Channel 1 (safest for dot board) |
| De-emphasis | 50 us (Sri Lanka / Asia / Europe standard) |
| Tuning | Two push buttons: Seek Up + Seek Down (auto-find stations) |
| Volume | B100K potentiometer (hardware, in audio signal path) |
| Power indicator | Red 5mm LED (always on when powered) |

---

# 0) Absolute Rules — Read Before You Touch a Soldering Iron

1. **Never connect 5V to the RRD-102V2.0 VCC.** It is a 3.3V-only module. Connecting 5V will destroy the RDA5807M chip permanently.
2. **I2C pull-ups go to 3.3V only.** Do not pull SDA or SCL to 5V.
3. **Do not connect the speaker to the radio module.** The RDA5807M audio outputs are line-level (~360mV). They cannot drive a 4-ohm speaker. The TDA2822M amplifier must sit between the module and the speaker.
4. **Keep the antenna wire physically far from the speaker wires and amplifier section.** RF is sensitive; amplifier output current creates noise.
5. **Place decoupling capacitors close to each IC.** Within 1-2 cm. Distance matters in analogue circuits.
6. **Verify the TDA2822M orientation (notch/dot = pin 1) before soldering.** If the IC is backwards, it will not work and may be destroyed.
7. **LED polarity matters.** The longer leg is the anode (+). The shorter leg (or flat side of the body) is the cathode (-).

---

# 1) What You Are Building

## 1.1 Signal Path

```
FM Antenna (~75 cm wire)
        |
        v
  RRD-102V2.0  (RDA5807M FM tuner, 3.3V)
   LOUT + ROUT
        |
   R3 + R4 (1k + 1k mono mixer)
        |
       C1 (10uF coupling capacitor)
        |
      RV1 (B100K volume pot, wiper to amp)
        |
        v
  TDA2822M  (audio amplifier, 5V)
   pin 3 (IN1+)  -->  pin 1 (OUT1)
        |
        v
   4-ohm Speaker
```

## 1.2 Control Path

```
BTN1 (Seek Up)   --> Arduino D2  (find next station upward)
BTN2 (Seek Down)  --> Arduino D3  (find next station downward)
Arduino A4 (SDA)  --> RDA5807M SDA  (I2C data)
Arduino A5 (SCL)  --> RDA5807M SCL  (I2C clock)
```

## 1.3 Power + Indicator

| Rail | Source | Supplies |
|------|--------|----------|
| +5V | Arduino USB | TDA2822M, LED1 (power indicator) |
| +3V3 | Arduino 3.3V pin | RRD-102V2.0, I2C pull-ups |

LED1 (red, 5mm diffused) is always on when the system is powered. It is wired directly across the +5V rail through a current-limiting resistor (R6, 1k). No Arduino pin is used.

---

# 2) Parts List (BOM)

Everything below is the minimum for a reliable dot-board build. Tick off each part before you start.

## 2.1 Active Components

| Ref | Part | Qty | Notes |
|-----|------|-----|-------|
| — | Arduino UNO | 1 | Any compatible board |
| U1 | RRD-102V2.0 (RDA5807M module) | 1 | Your module |
| U2 | TDA2822M (DIP-8) | 1 | Use the 8-pin IC socket |

## 2.2 Resistors

| Ref | Value | Qty | Purpose |
|-----|-------|-----|---------|
| R1 | 4.7k (or 10k) | 1 | SDA pull-up to 3.3V (4.7k preferred) |
| R2 | 4.7k (or 10k) | 1 | SCL pull-up to 3.3V (4.7k preferred) |
| R3 | 1k | 1 | LOUT to mono mixer |
| R4 | 1k | 1 | ROUT to mono mixer |
| R5 | 2.2 ohm | 1 | Zobel network (output stability) |
| R6 | 1k | 1 | LED current limiting (gives ~3mA at 5V) |

## 2.3 Capacitors

| Ref | Value | Type | Qty | Purpose |
|-----|-------|------|-----|---------|
| C1 | 10uF 16V | Electrolytic | 1 | Audio coupling (mixer to pot) |
| C2 | 0.1uF (marked 104) | Ceramic | 1 | U1 VCC decoupling |
| C3 | 0.1uF (marked 104) | Ceramic | 1 | U2 VCC decoupling |
| C4 | 1000uF 16V | Electrolytic | 1 | 5V bulk (near amplifier). 1000–2200uF is ideal. 4700uF works but risks USB inrush current on some laptops. |
| C5 | 0.1uF (marked 104) | Ceramic | 1 | Zobel network |
| C6 | 100uF 35V | Electrolytic | 1 | U2 pin 2 inverting-input bypass |
| C7 | 10uF 16V | Electrolytic | 1 | +3V3 rail bulk (RF stability, near U1) |
| C8 | 0.1uF (marked 104) | Ceramic | 1 | **Optional.** In parallel with C6 at pin 2 — improves high-frequency gain stability |
| C9 | 0.1uF (marked 104) | Ceramic | 1 | **Optional.** AF_MIX to GND — RF low-pass filter before amp. Kills residual I2C hash in audio |
| C10 | 0.1uF (marked 104) | Ceramic | 1 | **Optional.** Near Arduino 3.3V pin — extra decoupling at source of 3V3 rail |

## 2.4 Output + Indicator

| Ref | Part | Qty | Notes |
|-----|------|-----|-------|
| SPK1 | 4-ohm 5W speaker | 1 | Works but chip runs warm at high volume (~0.8–1W max clean). 8Ω speaker is safer/cooler. |
| LED1 | Red 5mm diffused LED | 1 | Power indicator (always on) |

## 2.5 User Controls

| Ref | Part | Qty | Notes |
|-----|------|-----|-------|
| RV1 | B100K potentiometer (linear) | 1 | Volume control (in audio path) |
| BTN1 | Push button micro switch | 1 | Seek Up (next station) |
| BTN2 | Push button micro switch | 1 | Seek Down (previous station) |

## 2.6 Antenna

| Part | Qty | Notes |
|------|-----|-------|
| Insulated wire ~75 cm | 1 | Quarter-wave for ~100 MHz centre |

## 2.7 Mechanical / Misc

| Part | Qty | Notes |
|------|-----|-------|
| 8-pin DIP IC socket | 1 | Mount TDA2822M in socket (replaceable) |
| Dot board (perfboard) | 1 | Your 15x18 cm board |
| Hookup wire (solid core) | — | Multiple colours recommended |

---

# 3) IC Pinouts — Get These Right or Nothing Works

## 3.1 TDA2822M (DIP-8) — Correct Pinout

**TOP VIEW — notch/dot at pin 1 end:**

```
            ____
    OUT1  1|o   |8  VCC  (+5V)
    IN1-  2|    |7  OUT2  (not used)
    IN1+  3|    |6  IN2-  (tie to GND)
     GND  4|____|5  IN2+  (tie to GND)
```

| Pin | Name | Function |
|-----|------|----------|
| 1 | OUT1 | Audio output channel 1 --> speaker + |
| 2 | IN1- | Inverting input ch1 --> 100uF cap to GND |
| 3 | IN1+ | Non-inverting input ch1 --> AUDIO IN (this is where your signal goes) |
| 4 | GND | Ground (only true GND pin) |
| 5 | IN2+ | Non-inverting input ch2 --> tie to GND (unused) |
| 6 | IN2- | Inverting input ch2 --> tie to GND (unused) |
| 7 | OUT2 | Output ch2 --> leave unconnected |
| 8 | VCC | Power supply --> +5V |

**WARNING:** Many internet diagrams label pins 3/4/5 incorrectly. Trust the table above (source: ST Microelectronics TDA2822 datasheet, DIP-8 package).

## 3.2 RRD-102V2.0 Module Pad Order (Your Board)

**Left side (top to bottom):**

| Pad | Label | Used? |
|-----|-------|-------|
| 1 | SDA | YES |
| 2 | SCL | YES |
| 3 | NC | No |
| 4 | NC | No |
| 5 | VCC | YES (3.3V only!) |

**Right side (top to bottom):**

| Pad | Label | Used? |
|-----|-------|-------|
| 6 | ANT | YES |
| 7 | NC | No |
| 8 | LOUT | YES |
| 9 | ROUT | YES |
| 10 | GND | YES |

> **Stop and check:** If the silkscreen labels on your physical board differ from this table, follow your board's silkscreen. The pad order above is what you reported.

---

# 4) Complete Circuit — Every Connection

All connections are expressed as named nets so there is zero ambiguity.

## 4.1 Net Names

| Net | Description |
|-----|-------------|
| +5V | Arduino 5V pin (USB power) |
| +3V3 | Arduino 3.3V pin |
| GND | Common system ground |
| SDA_3V3 | I2C data line (pulled up to 3.3V) |
| SCL_3V3 | I2C clock line (pulled up to 3.3V) |
| AF_L | Left audio output from radio module |
| AF_R | Right audio output from radio module |
| AF_MIX | Mono-mixed audio node (after R3+R4) |
| AF_IN | Amplifier input node (RV1 wiper output to TDA pin 3) |
| SPK+ | Speaker positive terminal |
| SPK- | Speaker negative terminal (= GND) |
| BTN_UP | Seek-up button line (Arduino D2, pulled HIGH internally) |
| BTN_DN | Seek-down button line (Arduino D3, pulled HIGH internally) |

## 4.2 Reference Designator Summary

| Ref | Part | Value / Type |
|-----|------|--------------|
| U1 | RRD-102V2.0 | RDA5807M FM module |
| U2 | TDA2822M | DIP-8 audio amplifier |
| R1 | Pull-up | 4.7k preferred / 10k OK (SDA to +3V3) |
| R2 | Pull-up | 4.7k preferred / 10k OK (SCL to +3V3) |
| R3 | Mixer | 1k (AF_L to AF_MIX) |
| R4 | Mixer | 1k (AF_R to AF_MIX) |
| R5 | Zobel | 2.2 ohm |
| R6 | LED limiter | 1k (5V to LED1 anode) |
| C1 | Coupling | 10uF electrolytic |
| C2 | Decoupling | 0.1uF ceramic (U1) |
| C3 | Decoupling | 0.1uF ceramic (U2) |
| C4 | Bulk | 1000uF electrolytic (1000–2200uF OK; 4700uF if already bought) |
| C5 | Zobel | 0.1uF ceramic |
| C6 | Bypass | 100uF electrolytic (U2 pin 2) |
| C8 | HF bypass | 0.1uF ceramic (across C6, optional) |
| C9 | RF filter | 0.1uF ceramic (AF_MIX to GND, optional) |
| C10 | 3V3 decoupling | 0.1uF ceramic (near Arduino 3.3V pin, optional) |
| C7 | Bulk (3V3) | 10uF electrolytic (near U1, +3V3 to GND) |
| RV1 | Volume | B100K potentiometer |
| LED1 | Power indicator | Red 5mm diffused |
| BTN1 | Seek Up | Push button micro switch |
| BTN2 | Seek Down | Push button micro switch |
| SPK1 | Speaker | 4 ohm, 5W (8Ω recommended for cooler running) |

## 4.3 U1 — RRD-102V2.0 (RDA5807M) Connections

| Module Pad | Net / Destination |
|------------|-------------------|
| VCC | +3V3 |
| GND | GND |
| SDA | SDA_3V3 |
| SCL | SCL_3V3 |
| LOUT | AF_L (to R3) |
| ROUT | AF_R (to R4) |
| ANT | 75 cm antenna wire |
| NC pads | Leave unconnected |

Place **C2 (0.1uF ceramic)** between +3V3 and GND, as close to U1 VCC/GND as possible.
Place **C7 (10uF electrolytic)** between +3V3 (+) and GND (-), right next to C2. This bulk cap smooths USB switching ripple on the 3.3V rail, reducing I2C noise and random seek failures.

## 4.4 I2C Pull-Ups

| Resistor | From | To |
|----------|------|----|
| R1 (4.7k preferred, 10k OK) | SDA_3V3 | +3V3 |
| R2 (4.7k preferred, 10k OK) | SCL_3V3 | +3V3 |

**Do NOT pull up to 5V.** The RDA5807M is a 3.3V device. 5V on SDA/SCL will damage it.

> **Why 4.7k is better than 10k:** Faster rise times, cleaner SDA/SCL edges, and better noise immunity on perfboard with long hookup wires. If you already soldered 10k, it will work — no need to replace.

## 4.5 Arduino UNO Connections

| Arduino Pin | Net / Destination |
|-------------|-------------------|
| 3.3V | +3V3 bus |
| 5V | +5V bus |
| GND | GND bus |
| A4 (SDA) | SDA_3V3 |
| A5 (SCL) | SCL_3V3 |
| D2 | BTN1 (Seek Up) — other leg to GND |
| D3 | BTN2 (Seek Down) — other leg to GND |

No external pull-up resistors are needed for D2/D3. The Arduino code enables internal pull-ups (~20k to 5V) on these pins.

## 4.6 Seek Buttons (BTN1 + BTN2)

Each button has two legs. One leg goes to an Arduino digital pin, the other leg goes to GND.

```
+5V (internal pull-up via INPUT_PULLUP)
 |
 [~20k internal]
 |
Arduino D2 ----+---- BTN1 ---- GND

+5V (internal pull-up via INPUT_PULLUP)
 |
 [~20k internal]
 |
Arduino D3 ----+---- BTN2 ---- GND
```

| Button | One Leg | Other Leg | Arduino Pin | Action |
|--------|---------|-----------|-------------|--------|
| BTN1 | Arduino D2 | GND | D2 | Seek Up (find next higher station) |
| BTN2 | Arduino D3 | GND | D3 | Seek Down (find next lower station) |

When not pressed: pin reads HIGH (pulled up internally).
When pressed: pin reads LOW (connected to GND through button).

## 4.7 Power LED (LED1)

The LED is a simple power indicator — always on when the system is powered. It does not use any Arduino pin.

```
+5V ---[ R6  1k ]---[>|--- GND
                    LED1
              (anode)  (cathode)
```

| Component | From | To |
|-----------|------|----||
| R6 (1k) | +5V bus | LED1 anode (longer leg) |
| LED1 cathode | LED1 cathode (shorter leg / flat side) | GND bus |

Current: (5V - 2V) / 1000 ohm = 3 mA. Dim but clearly visible on a diffused red LED. Safe, no heat.

## 4.8 Mono Mixer + Volume Control

Audio from the radio module is mixed to mono, DC-blocked by C1, then passes through the volume pot before entering the amplifier.

```
U1 LOUT ---[ R3  1k ]---+
                         |
                       AF_MIX ---[ C1  10uF ]---+--- RV1 lug 1 (top)
                         |                       |
U1 ROUT ---[ R4  1k ]---+                  RV1 wiper ---> AF_IN (TDA pin 3)
                         |                        |
                   [C9 0.1uF]              RV1 lug 3 (bottom) --- GND
                   (optional)|
                        GND

C9 (optional): A small ceramic cap from AF_MIX to GND. Creates a
passive RF low-pass filter that kills residual I2C clock hash before
it reaches the amplifier. Not required — "audiophile clean" level.
```

| Component | From | To |
|-----------|------|----|
| R3 (1k) | AF_L (U1 LOUT) | AF_MIX |
| R4 (1k) | AF_R (U1 ROUT) | AF_MIX |
| C1 (10uF) | AF_MIX (+) | RV1 lug 1 (-) |
| RV1 lug 1 | C1 negative leg | (top of pot track) |
| RV1 wiper | (middle pin) | AF_IN (TDA2822M pin 3) |
| RV1 lug 3 | (bottom of pot track) | GND bus |

**C1 polarity:** Positive (+) leg toward AF_MIX (radio side). Negative (-) leg toward RV1 lug 1.

**RV1 orientation:** Turn fully clockwise = maximum volume (wiper at lug 1, full signal). Turn fully counter-clockwise = silence (wiper at lug 3, signal shorted to GND).

## 4.9 U2 — TDA2822M Amplifier Connections

| Pin | Name | Connect To | Notes |
|-----|------|------------|-------|
| 1 | OUT1 | SPK+ (speaker positive) | Also connects to Zobel network |
| 2 | IN1- | C6 (100uF) ∥ C8 (0.1uF) to GND | Inverting input bypass. C8 optional — improves HF stability |
| 3 | IN1+ | AF_IN | Audio signal input (from RV1 wiper) |
| 4 | GND | GND bus | The only ground pin on this IC |
| 5 | IN2+ | GND | Unused channel — tie to ground |
| 6 | IN2- | GND | Unused channel — tie to ground |
| 7 | OUT2 | Leave unconnected | Unused channel output |
| 8 | VCC | +5V bus | Power supply |

**C6 polarity:** Positive (+) leg toward pin 2. Negative (-) leg toward GND.

**C8 (optional 0.1uF ceramic):** Solder directly across C6 legs (pin 2 to GND). Ceramic caps have
no polarity. This shunts high-frequency noise that the large electrolytic C6 cannot absorb.

## 4.10 Speaker

| Terminal | Connect To |
|----------|------------|
| SPK+ | U2 pin 1 (OUT1) |
| SPK- | GND |

> **4Ω Speaker Warning:** In single-ended mode at 5V, the TDA2822M delivers ~0.8–1W max clean
> output into 4Ω. At high volume, expect increased distortion and the IC will get warm
> (50–60°C). This is within spec but near the limit. **Don't max the pot for extended periods.**
> An 8Ω speaker runs cooler and stays well within the IC's comfort zone.

**Optional EMI reduction:** Solder a **100nF (0.1µF) ceramic capacitor** directly across the
speaker terminals (SPK+ to SPK−). This suppresses high-frequency switching noise.

## 4.11 Zobel Network (Output Stability)

Prevents high-frequency oscillation on dot-board wiring.

```
U2 pin 1 (OUT1) ---[ R5  2.2 ohm ]---[ C5  0.1uF ]--- GND
```

R5 and C5 are in series, connected between OUT1 and GND.

## 4.12 Power Decoupling and Bulk Capacitors

| Cap | Between | Placement |
|-----|---------|-----------|
| C2 (0.1uF ceramic) | +3V3 and GND | As close to U1 VCC/GND as possible |
| C7 (10uF electrolytic) | +3V3 (+) and GND (-) | Right next to C2, near U1. Reduces USB ripple on 3.3V rail |
| C10 (0.1uF ceramic) | +3V3 and GND | **(Optional)** Near where 3.3V wire lands from Arduino. Extra source-end decoupling |
| C3 (0.1uF ceramic) | +5V and GND | As close to U2 pin 8 and pin 4 as possible |
| C4 (1000uF electrolytic) | +5V and GND | Near U2, positive to +5V, negative to GND. 4700uF works if already bought. |

---

# 5) Dot-Board Layout Guidance

## 5.1 Board Sides

| Side | What Goes There |
|------|-----------------|
| Plain brown side | Components (ICs, caps, resistors, etc.) |
| Copper pad side | Solder joints and wire bridges |

Components sit on the plain side. Their legs poke through the holes. You flip the board and solder on the copper side.

## 5.2 FRONT VIEW — Component Side (15 × 18 cm, long side horizontal)

This is the side you look at during use. Components sit here, legs poke through to the back.
Board long side (18 cm) runs left-to-right. Short side (15 cm) runs top-to-bottom.

```
 FRONT VIEW (component side — you face this side)
 Long side horizontal, 0,0 = bottom-left

 +5V bus (left edge, vertical)              GND bus (right edge, vertical)
    ║                                                        ║
    ║  ZONE A — RF + DISPLAY  (top 5 cm)                     ║
    ║                                                        ║
    ║   ANT wire ──►  exits top edge immediately              ║
    ║        │                                                ║
    ║   ┌──────────┐                                         ║
    ║   │ U1       │    [R1 4.7k]──┐                         ║
    ║   │RRD-102   │    [R2 4.7k]──┤── +3V3 bus (short,      ║
    ║   │(RDA5807M)│               │    Zone A only)          ║
    ║   │ 1.1×1.1  │  [C2 0.1µF]──┤── GND                   ║
    ║   └──────────┘  [C7 10µF+]──┘── GND                   ║
    ║                                                        ║
    ║ ─ ─ ─ ─ ─ ─ ~3-4 cm buffer ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  ║
    ║                                                        ║
    ║  ZONE B — CONTROLS + MIXER  (middle 5 cm)              ║
    ║                                                        ║
    ║   [R3 1k]──┐                                           ║
    ║   [R4 1k]──┤── AF_MIX ──[C1 10µF+]──┐                 ║
    ║            │  [C9 0.1µF]── GND (opt) │                 ║
    ║            │     ┌───────────┐        │                 ║
    ║            │     │ RV1 B100K │   Lug 1 (top)           ║
    ║            │     │   POT     │        │                 ║
    ║            │     │           │   Wiper ──► AF_IN       ║
    ║            │     └───────────┘        │                 ║
    ║            │                     Lug 3 ── GND          ║
    ║            │                                            ║
    ║   [BTN1]──GND        [R6 1k]──[LED1>|]── GND           ║
    ║   [BTN2]──GND                                          ║
    ║                                                        ║
    ║ ─ ─ ─ ─ ─ ─ ~2 cm buffer ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  ║
    ║                                                        ║
    ║  ZONE C — AMPLIFIER  (bottom 5 cm)                     ║
    ║                                                        ║
    ║          [C3 0.1µF]   [C4 1000µF+]                     ║
    ║          (pin8─4)     (+5V ── GND)                     ║
    ║                                                        ║
    ║             ┌────────────┐                              ║
    ║    AF_IN ──►│ TDA2822M   │                              ║
    ║             │  (DIP-8    │                              ║
    ║             │  in socket)│                              ║
    ║             └─────┬──────┘                              ║
    ║                   │                                     ║
    ║   [C6 100µF+]── pin 2     [C8 0.1µF]── across C6 legs  ║
    ║   [R5 2.2Ω]──[C5 0.1µF]── GND  (Zobel from pin 1)     ║
    ║                   │                                     ║
    ║              SPK wires ──► exit bottom edge             ║
    ║                                                        ║
    ╚════════════════════════════════════════════════════════╝
     (0,0)                                          (18,0)
```

## 5.3 BACK VIEW — Solder Side (mirrored left-right)

When you flip the board over (hinge along the top edge), left and right are swapped.
The copper pads are on this side. All wire bridges and solder joints are made here.

```
 BACK VIEW (solder/copper side — board flipped over top edge)
 0,0 = bottom-left of back view

 GND bus (now on LEFT, was right)             +5V bus (now on RIGHT, was left)
    ║                                                        ║
    ║                                                        ║
    ║   ┌──────────────────────────────────────────────────┐  ║
    ║   │         Copper pads visible                      │  ║
    ║   │         Solder joints here                       │  ║
    ║   │                                                  │  ║
    ║   │  ZONE A (top, mirrored):                         │  ║
    ║   │    U1 pads, R1/R2/C2/C7 solder joints            │  ║
    ║   │    +3V3 bus short run (now on right side)         │  ║
    ║   │    Antenna pad — wire exits top edge              │  ║
    ║   │                                                  │  ║
    ║   │  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │  ║
    ║   │                                                  │  ║
    ║   │  ZONE B (middle, mirrored):                      │  ║
    ║   │    Wire bridges: SDA→Arduino A4, SCL→Arduino A5  │  ║
    ║   │    BTN1→D2, BTN2→D3 wire runs                    │  ║
    ║   │    R3/R4/C1/RV1 solder joints                    │  ║
    ║   │    R6/LED1 solder joints                         │  ║
    ║   │                                                  │  ║
    ║   │  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │  ║
    ║   │                                                  │  ║
    ║   │  ZONE C (bottom, mirrored):                      │  ║
    ║   │    IC socket pads (pin 1 now at RIGHT)           │  ║
    ║   │    C3/C4/C5/C6/C8 solder joints                  │  ║
    ║   │    R5 (Zobel) solder joints                      │  ║
    ║   │    Speaker wire pads — exit bottom edge          │  ║
    ║   │                                                  │  ║
    ║   │  Arduino hookup wires (7 wires):                 │  ║
    ║   │    Run along back side from landing pads         │  ║
    ║   │    to board edge, then to Arduino headers        │  ║
    ║   │                                                  │  ║
    ║   └──────────────────────────────────────────────────┘  ║
    ║                                                        ║
    ╚════════════════════════════════════════════════════════╝
     (0,0)                                          (18,0)

 IMPORTANT: Pin 1 of TDA2822M socket appears on the RIGHT
 side of the back view (mirrored from front). Double-check
 notch orientation before soldering the socket.

 Wire routing (back side):
 ┌─────────────────────────────────────────────┐
 │  GND bus ══ runs vertically on LEFT edge    │
 │  +5V bus ══ runs vertically on RIGHT edge   │
 │  +3V3 bus = short run, top-right area only  │
 │                                             │
 │  Signal wires: route BETWEEN the buses      │
 │  Keep audio wires (AF_*) short and direct   │
 │  Keep SDA/SCL twisted pair away from audio  │
 │  Keep speaker wires twisted, exit at bottom │
 └─────────────────────────────────────────────┘
```

Keep **at least 3-4 cm** between Zone A (radio) and Zone C (amplifier).

## 5.3 Ground Strategy

Create one **thick ground bus** (bare copper wire or thick hookup wire) running vertically along the right edge of the board. Bring ALL ground connections to this bus:

- U1 GND
- U2 pin 4 (GND)
- U2 pin 5, pin 6 (tied to GND)
- C2, C7, C3, C4, C5, C6, C8, C9, C10 (if fitted) negative legs
- RV1 lug 3 (bottom)
- BTN1 one leg, BTN2 one leg
- Arduino GND
- Speaker negative

**Do not daisy-chain grounds** through thin wires from pad to pad. Every ground wire goes directly to the bus.

## 5.4 Wire Routing Rules

- **Antenna wire:** Exit the board edge immediately from U1 ANT pad. Do not route it across the board.
- **Speaker wires:** Keep short. Exit at bottom edge near U2. Do not route near Zone A.
- **Audio wires (AF_L, AF_R, AF_MIX, AF_IN):** Keep short. Route away from SDA/SCL.
- **SDA/SCL:** **Twist the pair together** (2–3 turns per cm). Improves I2C reliability on perfboard.
- **Speaker wires:** **Twist together** (reduces magnetic noise pickup from amplifier current loops).
- **Power wires (+5V, +3V3):** Run along the left edge bus. Keep away from audio/antenna.

---

# 6) Step-By-Step Build Procedure

**Do not solder everything at once.** Build in stages and verify at each stage.

## Stage 1 — Lay Down Power Rails

1. Using thick wire on the copper side, create a **GND bus** along the right edge (vertical).
2. Create a **+5V bus** along the left edge (vertical).
3. Create a short **+3V3 bus** in Zone A only (a few pads near where U1 will sit).

## Stage 2 — Install U1 (RRD-102V2.0) and I2C Pull-Ups

4. Mount and solder U1 in Zone A.
5. Solder **R1 (4.7k preferred, or 10k)**: from U1 SDA pad to +3V3 bus.
6. Solder **R2 (4.7k preferred, or 10k)**: from U1 SCL pad to +3V3 bus.
7. Solder **C2 (0.1uF ceramic)**: between +3V3 bus and GND bus, close to U1.
8. Solder **C7 (10uF electrolytic)**: positive (+) to +3V3 bus, negative (-) to GND bus, right next to C2.
9. Solder antenna wire to U1 ANT pad. Route the wire out of the board edge immediately.
10. Wire U1 VCC to +3V3 bus.
11. Wire U1 GND to GND bus.

## Stage 3 — Install Mono Mixer, Volume Pot, Buttons, and LED

12. Solder **R3 (1k)**: one end to U1 LOUT pad, other end to a shared node (AF_MIX).
13. Solder **R4 (1k)**: one end to U1 ROUT pad, other end to the same AF_MIX node.
14. Solder **C1 (10uF electrolytic)**: positive (+) to AF_MIX, negative (-) toward RV1.
14b. *(Optional)* Solder **C9 (0.1uF ceramic)** from AF_MIX node to GND. No polarity. RF filter.
15. Mount **RV1 (B100K pot)** in Zone B. Connect **lug 1** (top) to C1 negative leg. Connect **lug 3** (bottom) to GND bus. The **wiper** (middle pin) becomes AF_IN.
16. Mount **BTN1** (seek up) in Zone B. Wire one leg to a pad for Arduino D2, other leg to GND bus.
17. Mount **BTN2** (seek down) in Zone B near BTN1. Wire one leg to a pad for Arduino D3, other leg to GND bus.
18. Solder **R6 (1k)**: one end to +5V bus.
19. Solder **LED1** (red): anode (long leg) to the free end of R6, cathode (short leg / flat side) to GND bus.

## Stage 4 — Install U2 (TDA2822M) and Support Components

20. Solder the **8-pin IC socket** in Zone C. Verify the notch orientation — pin 1 is at the notch end.
21. Solder **C3 (0.1uF ceramic)**: between +5V and GND, close to the socket (pins 8 and 4).
22. Solder **C4 (1000uF electrolytic)**: positive (+) to +5V, negative (-) to GND. Place close to socket. (2200uF or 4700uF also fine if already bought.)
23. Solder **C6 (100uF electrolytic)**: positive (+) to socket pin 2, negative (-) to GND.
23b. *(Optional)* Solder **C8 (0.1uF ceramic)** directly across C6 legs (pin 2 to GND). No polarity.
24. Wire socket **pin 3 (IN1+)** to AF_IN (the RV1 wiper from step 15).
25. Wire socket **pin 4** to GND bus.
26. Wire socket **pin 5** to GND bus.
27. Wire socket **pin 6** to GND bus.
28. Wire socket **pin 8** to +5V bus.
29. Wire socket **pin 1 (OUT1)** to speaker positive pad.
30. Wire speaker negative pad to GND bus.
31. Solder **Zobel network**: R5 (2.2 ohm) from pin 1 to one leg of C5 (0.1uF), other leg of C5 to GND.
32. Leave socket **pin 7** unconnected.
33. **Do NOT insert the TDA2822M IC yet.** Leave the socket empty until all soldering near it is done.

## Stage 5 — Wire to Arduino UNO

34. Run a wire from **+3V3 bus** to a pad labeled for Arduino 3.3V.
34b. *(Optional)* Solder **C10 (0.1uF ceramic)** between the Arduino 3.3V landing pad and GND. Extra source-end decoupling.
35. Run a wire from **+5V bus** to a pad labeled for Arduino 5V.
36. Run a wire from **GND bus** to a pad labeled for Arduino GND.
37. Run a wire from **U1 SDA pad** (same net as R1) to a pad labeled for Arduino A4.
38. Run a wire from **U1 SCL pad** (same net as R2) to a pad labeled for Arduino A5.
39. Run a wire from **BTN1** (the non-GND leg) to a pad labeled for Arduino D2.
40. Run a wire from **BTN2** (the non-GND leg) to a pad labeled for Arduino D3.
41. Connect these 7 pads to the Arduino using hookup wires with headers (or solder directly).

## Stage 6 — Final Assembly

42. **Insert TDA2822M** into the IC socket. Pin 1 (notch end) must match the socket notch.
43. Inspect every solder joint on the copper side. Look for cold joints or bridges.
44. Verify LED1 is oriented correctly (long leg toward R6 / +5V side).
45. Verify RV1 wiper goes to TDA pin 3, lug 3 goes to GND.
46. Proceed to pre-power checks (Section 7).

---

# 7) Pre-Power Checklist (MUST PASS Before Applying Power)

Use a multimeter.

## 7.1 Continuity Checks (Power Off)

| Test | Expected | If FAIL |
|------|----------|---------|
| +5V to GND | NOT shorted (no beep) | Find and fix solder bridge |
| +3V3 to GND | NOT shorted (no beep) | Find and fix solder bridge |
| SDA to GND | NOT shorted | Check R1 wiring |
| SCL to GND | NOT shorted | Check R2 wiring |
| U2 pin 1 to GND | NOT shorted | Check speaker/Zobel wiring |
| D2 pad to GND | NOT shorted (unless BTN1 pressed) | Check BTN1 wiring |
| D3 pad to GND | NOT shorted (unless BTN2 pressed) | Check BTN2 wiring |

## 7.2 Voltage Checks (Power On via USB)

Connect Arduino to USB. Do NOT upload sketch yet.

| Probe Point | Expected Voltage |
|-------------|-----------------|
| U1 VCC pad | 3.2V - 3.4V |
| U2 pin 8 | 4.8V - 5.2V |
| U2 pin 4 | 0V (ground) |
| LED1 | Should be glowing red |
| BTN1 D2 pad (button not pressed) | ~5V (internal pull-up) |
| BTN1 D2 pad (button pressed) | ~0V |

If U1 VCC shows ~5V, **disconnect immediately** — you have wired VCC to the wrong rail.
If LED1 is not lit, check R6 orientation and LED polarity (anode toward R6).

## 7.3 Capacitor Polarity Verification (Visual)

| Cap | Positive (+) Side | Negative (-) Side |
|-----|-------------------|-------------------|
| C1 (10uF) | AF_MIX (toward radio module) | RV1 lug 1 (toward pot) |
| C7 (10uF) | +3V3 bus (near U1) | GND bus |
| C4 (1000uF) | +5V bus | GND bus |
| C6 (100uF) | U2 pin 2 | GND bus |
| C8 (0.1uF) | *(no polarity — ceramic)* across C6 legs, optional |
| C9 (0.1uF) | *(no polarity — ceramic)* AF_MIX to GND, optional |
| C10 (0.1uF) | *(no polarity — ceramic)* Arduino 3.3V pad to GND, optional |

---

# 8) Arduino Code

Upload this sketch after passing all pre-power checks.

```cpp
/*
 * FM Radio - RDA5807M via I2C + Seek Buttons
 * Phase-1 Dot Board Build (Rev 3.0)
 *
 * Wiring:
 *   Arduino A4 (SDA) --> RDA5807M SDA (with 4.7k/10k pull-up to 3.3V)
 *   Arduino A5 (SCL) --> RDA5807M SCL (with 4.7k/10k pull-up to 3.3V)
 *   Arduino D2       --> BTN1 (Seek Up)   other leg to GND
 *   Arduino D3       --> BTN2 (Seek Down) other leg to GND
 *   Arduino 3.3V     --> RDA5807M VCC
 *   Arduino 5V       --> TDA2822M pin 8, R6+LED1
 *   Arduino GND      --> common ground
 *
 * Usage:
 *   - Press BTN1 to seek to the next station (higher frequency)
 *   - Press BTN2 to seek to the previous station (lower frequency)
 *   - Open Serial Monitor at 9600 baud to see current frequency
 */

#include <Wire.h>

// ---- Pin definitions ----
#define BTN_SEEK_UP    2   // D2: seek next station (higher frequency)
#define BTN_SEEK_DOWN  3   // D3: seek previous station (lower frequency)

// ---- RDA5807M I2C address (sequential mode) ----
// Writes auto-start at register 0x02. Do NOT send register address byte.
static const uint8_t RDA_ADDR = 0x10;

// Shadow copies of registers 0x02 through 0x07
static uint16_t reg[8];

// ---- I2C Write: registers 0x02-0x07 ----
static void rdaWriteRegisters() {
  Wire.beginTransmission(RDA_ADDR);
  for (uint8_t r = 2; r <= 7; r++) {
    Wire.write((uint8_t)(reg[r] >> 8));   // high byte
    Wire.write((uint8_t)(reg[r] & 0xFF)); // low byte
  }
  Wire.endTransmission();
}

// ---- I2C Read: register 0x0A (2 bytes) ----
// In sequential read mode at 0x10, reads start at register 0x0A.
static uint16_t rdaReadReg0A() {
  Wire.requestFrom((uint8_t)RDA_ADDR, (uint8_t)2);
  uint8_t hi = Wire.read();
  uint8_t lo = Wire.read();
  return ((uint16_t)hi << 8) | lo;
}

// ---- Tune to a specific frequency (MHz) ----
static void rdaSetFrequency(float freqMHz) {
  if (freqMHz < 87.0f)  freqMHz = 87.0f;
  if (freqMHz > 108.0f) freqMHz = 108.0f;

  uint16_t channel = (uint16_t)((freqMHz - 87.0f) / 0.1f + 0.5f);

  // Register 0x03 layout:
  //   [15:6] CHAN, [5] DIRECT_MODE, [4] TUNE, [3:2] BAND, [1:0] SPACE
  reg[3] = ((channel & 0x03FF) << 6) | (1u << 4);

  rdaWriteRegisters();
  delay(100);

  reg[3] &= ~(1u << 4);  // clear TUNE bit
  rdaWriteRegisters();
}

// ---- Seek to next/previous station ----
// Returns the frequency (MHz) of the station found, or 0.0 if seek failed.
static float rdaSeek(bool seekUp) {
  // Set seek direction in reg 0x02
  if (seekUp) {
    reg[2] |= (1u << 9);   // SEEKUP = 1 (seek toward higher frequencies)
  } else {
    reg[2] &= ~(1u << 9);  // SEEKUP = 0 (seek toward lower frequencies)
  }
  reg[2] |= (1u << 8);     // SEEK = 1 (start seeking)

  rdaWriteRegisters();

  // Wait for STC (Seek/Tune Complete) flag in register 0x0A bit 14
  delay(200);
  uint16_t reg0A = 0;
  for (int attempt = 0; attempt < 80; attempt++) {
    reg0A = rdaReadReg0A();
    if (reg0A & (1u << 14)) break;  // STC = 1: seek/tune complete
    delay(50);
  }

  // Clear SEEK bit
  reg[2] &= ~(1u << 8);
  rdaWriteRegisters();

  // Dummy read to clear STC status latch (prevents rare seek locks)
  delay(50);
  rdaReadReg0A();

  // Check seek fail (bit 13 of reg 0x0A)
  if (reg0A & (1u << 13)) {
    // SF = 1: no station found (wrapped entire band)
    return 0.0f;
  }

  // Read the channel the chip stopped at
  uint16_t channel = reg0A & 0x03FF;
  float freqMHz = 87.0f + channel * 0.1f;

  // Update our shadow reg[3] to reflect the new channel
  reg[3] = (channel << 6);  // TUNE=0, BAND=00, SPACE=00

  return freqMHz;
}

// ---- Setup ----
void setup() {
  Serial.begin(9600);
  Wire.begin();
  delay(100);  // let RDA5807M power up

  // Configure button pins with internal pull-ups
  pinMode(BTN_SEEK_UP,   INPUT_PULLUP);
  pinMode(BTN_SEEK_DOWN, INPUT_PULLUP);

  // Register 0x02: DHIZ=1, DMUTE=1, ENABLE=1
  //   Bit 15: DHIZ     = 1  (audio output high impedance disabled = output ON)
  //   Bit 14: DMUTE    = 1  (not muted)
  //   Bit 13: MONO     = 0  (stereo; we mix externally with resistors)
  //   Bit 12: BASS     = 0
  //   Bits 6-4: CLK_MODE = 000  (32.768 kHz crystal, already on module)
  //   Bit 0:  ENABLE   = 1  (power up the chip)
  reg[2] = 0xC001;

  // Register 0x03: will be set by rdaSetFrequency()
  reg[3] = 0x0000;

  // Register 0x04:
  //   Bit 12: DE = 1         (50us de-emphasis, correct for Sri Lanka)
  //   Bit 10: SOFTMUTE_EN = 1
  reg[4] = 0x1400;

  // Register 0x05: volume + seek threshold
  //   Bits 3-0: VOLUME = 0xF (maximum volume = 15)
  //   Bits 11-8: SEEKTH = 0x8 (seek SNR threshold, default)
  //   Upper bits: common working defaults
  reg[5] = 0x88AF;

  reg[6] = 0x0000;
  reg[7] = 0x0000;

  rdaWriteRegisters();
  delay(200);

  // Tune to a default station (change 99.5 to a known local station)
  rdaSetFrequency(99.5);
  Serial.println(F("FM Radio Ready"));
  Serial.println(F("Tuned to 99.5 MHz"));
  Serial.println(F("Press BTN1 (D2) = Seek Up, BTN2 (D3) = Seek Down"));
}

// ---- Main Loop ----
void loop() {
  static bool lastBtnUp   = HIGH;
  static bool lastBtnDown = HIGH;

  bool btnUp   = digitalRead(BTN_SEEK_UP);
  bool btnDown = digitalRead(BTN_SEEK_DOWN);

  // Detect falling edge on Seek Up button
  if (btnUp == LOW && lastBtnUp == HIGH) {
    delay(50);  // debounce
    if (digitalRead(BTN_SEEK_UP) == LOW) {
      Serial.print(F("Seeking up... "));
      float freq = rdaSeek(true);
      if (freq > 0.0f) {
        Serial.print(freq, 1);
        Serial.println(F(" MHz"));
      } else {
        Serial.println(F("No station found"));
      }
    }
  }

  // Detect falling edge on Seek Down button
  if (btnDown == LOW && lastBtnDown == HIGH) {
    delay(50);  // debounce
    if (digitalRead(BTN_SEEK_DOWN) == LOW) {
      Serial.print(F("Seeking down... "));
      float freq = rdaSeek(false);
      if (freq > 0.0f) {
        Serial.print(freq, 1);
        Serial.println(F(" MHz"));
      } else {
        Serial.println(F("No station found"));
      }
    }
  }

  lastBtnUp   = btnUp;
  lastBtnDown = btnDown;

  delay(20);
}
```

## 8.1 If the Radio Does Not Respond

**Problem A — No I2C communication at all:**
Some RDA5807M modules respond only to address 0x11 (index mode, which requires a register address byte). Change the address and write function:

```cpp
static const uint8_t RDA_ADDR = 0x11;  // index mode

static void rdaWriteRegisters() {
  Wire.beginTransmission(RDA_ADDR);
  Wire.write((uint8_t)0x02);  // start register address
  for (uint8_t r = 2; r <= 7; r++) {
    Wire.write((uint8_t)(reg[r] >> 8));
    Wire.write((uint8_t)(reg[r] & 0xFF));
  }
  Wire.endTransmission();
}
```

If using 0x11 mode, also update `rdaReadReg0A()` to read from the correct register:

```cpp
static uint16_t rdaReadReg0A() {
  Wire.beginTransmission(RDA_ADDR);
  Wire.write((uint8_t)0x0A);  // register address
  Wire.endTransmission(false);
  Wire.requestFrom((uint8_t)RDA_ADDR, (uint8_t)2);
  uint8_t hi = Wire.read();
  uint8_t lo = Wire.read();
  return ((uint16_t)hi << 8) | lo;
}
```

**Problem B — Tunes but poor audio:**
Try enabling MONO in software by changing reg[2]:

```cpp
reg[2] = 0xE001;  // bit 13 MONO=1
```

**Problem C — Seek never finds stations:**
- Confirm antenna is ~75 cm and connected to ANT pad.
- Lower the seek threshold (makes seek more sensitive to weak stations):
  ```cpp
  reg[5] = 0x84AF;  // SEEKTH = 4 (lower = more sensitive)
  ```
- As a last resort, try tuning manually to a known local station:
  ```cpp
  rdaSetFrequency(93.7);  // replace with your local station
  ```

---

# 9) Bring-Up Test Plan

Run these tests in order after uploading the code.

## Test A — Power and LED

1. Plug in USB.
2. **LED1 should glow red.** If not, check LED polarity and R6 connection.
3. Measure U1 VCC: should read ~3.3V.
4. Measure U2 pin 8: should read ~5V.
5. Touch the TDA2822M IC briefly — it should be cool or barely warm.

## Test B — Volume Pot and Audio Chain

1. Turn RV1 fully clockwise (maximum volume).
2. With power on, lightly touch the AF_IN node (RV1 wiper / middle pin) with your finger.
3. You should hear a brief hum or pop from the speaker.
4. Turn RV1 fully counter-clockwise. The hum should go silent.
5. This confirms the pot and amplifier are working.

**Do not short AF_IN to power rails.**

## Test C — FM Reception (Seek Buttons)

1. Open the Serial Monitor at 9600 baud.
2. Press BTN1 (D2). The serial output should print "Seeking up..." then a frequency.
3. You should hear audio from the station found.
4. Turn RV1 (volume pot) up and down — volume should change smoothly.
5. Press BTN2 (D3). It should seek down to the next lower station.
5. If seek wraps without finding anything, adjust the antenna or lower SEEKTH (see Section 8.1 Problem C).

## Test D — Button Debounce

1. Quickly tap BTN1 several times. Each press should produce exactly one seek.
2. If you see multiple seeks per press, increase the debounce delay in the code (50 → 100 ms).

---

# 10) Troubleshooting

## No sound at all

| Check | Action |
|-------|--------|
| Speaker wiring | SPK+ to U2 pin 1, SPK- to GND |
| TDA2822M orientation | Notch/dot must be at pin 1 |
| C1 installed? | Must be between AF_MIX and RV1 lug 1 |
| RV1 wiper to TDA pin 3? | Wiper (middle) must go to pin 3 |
| RV1 turned up? | Turn fully clockwise for max volume |
| C6 installed? | 100uF from U2 pin 2 to GND |
| U2 pin 4 to GND? | This is the only GND pin |

## Radio never tunes / no I2C

| Check | Action |
|-------|--------|
| R1/R2 connected to 3.3V? | Must be +3V3, never +5V |
| SDA to A4, SCL to A5? | Swap if reversed |
| Try address 0x11 | See Section 8.1 Problem A |

## Loud buzzing or digital ticking

| Check | Action |
|-------|--------|
| Audio wire length | Shorten all audio wires |
| SDA/SCL near audio? | Re-route to separate them |
| C2 and C3 close to ICs? | Move closer if possible |
| Zobel installed? | R5+C5 from pin 1 to GND |

## Weak reception

| Check | Action |
|-------|--------|
| Antenna connected? | Must be ~75 cm wire on ANT pad |
| Antenna position | Keep away from board, hold vertically |
| Try mono mode | Set reg[2] = 0xE001 in code |

## LED1 does not glow

| Check | Action |
|-------|--------|
| LED polarity | Long leg (anode) toward R6 / +5V side |
| R6 installed? | 1k between +5V bus and LED anode |
| GND connection | LED cathode (short leg / flat side) to GND bus |

## Buttons do not seek

| Check | Action |
|-------|--------|
| Wiring | One leg to D2/D3, other leg to GND |
| Serial Monitor open? | 9600 baud — look for "Seeking..." messages |
| Internal pull-up working? | Measure D2/D3 with button not pressed — should read ~5V |
| Button stuck low? | If pin always reads LOW, check for short to GND |

## Volume pot not working

| Check | Action |
|-------|--------|
| Wiper connection | RV1 middle pin must go to TDA pin 3 (AF_IN) |
| Lug 3 to GND? | Bottom lug must go to GND bus |
| Lug 1 to C1? | Top lug must connect to C1 negative leg |
| Scratchy audio when turning? | Normal for new pots; will improve with use |

## TDA2822M gets very hot

| Check | Action |
|-------|--------|
| Speaker impedance | Must be 4 ohm or higher. 8Ω runs significantly cooler |
| Volume too high? | Don't max the pot with a 4Ω speaker for extended periods — ~0.8–1W max clean output |
| Short on output? | Check pin 1 is not shorted to GND |
| VCC correct? | Should be 5V, not higher |

---

# 11) What Phase-1 Delivers

- Working FM radio reception (87-108 MHz)
- Mono speaker output through TDA2822M amplifier
- Hardware volume control (B100K potentiometer)
- Two seek buttons: BTN1 (seek up) and BTN2 (seek down)
- Red LED power indicator (always on when USB connected)
- Serial Monitor frequency readout at 9600 baud
- Safe 3.3V I2C levels (no risk to RDA5807M)
- Stable USB-powered operation
- Replaceable amplifier IC (socketed)

---

# Appendix A — One-Page Wiring Cheat Sheet

Print this page and keep it next to the dot board while soldering.

## Arduino UNO Wires (7 wires to dot board)

| Arduino Pin | Goes To |
|-------------|---------|
| 3.3V | +3V3 bus (powers U1, R1, R2, C7) |
| 5V | +5V bus (powers U2, R6 → LED1) |
| GND | GND bus |
| A4 | SDA_3V3 (U1 SDA pad) |
| A5 | SCL_3V3 (U1 SCL pad) |
| D2 | BTN1 one leg (other leg to GND) |
| D3 | BTN2 one leg (other leg to GND) |

## RRD-102V2.0 (U1) Summary

| Pad | Goes To |
|-----|---------|
| VCC | +3V3 |
| GND | GND bus |
| SDA | Arduino A4 + R1 (4.7k/10k) to +3V3 |
| SCL | Arduino A5 + R2 (4.7k/10k) to +3V3 |
| LOUT | R3 (1k) to AF_MIX |
| ROUT | R4 (1k) to AF_MIX |
| ANT | 75 cm wire antenna |

## TDA2822M (U2) Summary

| Pin | Goes To |
|-----|---------|
| 1 (OUT1) | Speaker + and Zobel (R5+C5 to GND) |
| 2 (IN1-) | C6 (100uF) ∥ C8 (0.1uF, optional) to GND |
| 3 (IN1+) | AF_IN (from RV1 wiper) |
| 4 (GND) | GND bus |
| 5 (IN2+) | GND |
| 6 (IN2-) | GND |
| 7 (OUT2) | Not connected |
| 8 (VCC) | +5V bus |

## Passive Components at a Glance

| Ref | Value | Between |
|-----|-------|---------|
| R1 | 4.7k (or 10k) | SDA_3V3 --- +3V3 |
| R2 | 4.7k (or 10k) | SCL_3V3 --- +3V3 |
| R3 | 1k | U1 LOUT --- AF_MIX |
| R4 | 1k | U1 ROUT --- AF_MIX |
| R5 | 2.2 ohm | U2 pin 1 --- C5 |
| R6 | 1k | +5V bus --- LED1 anode |
| C1 | 10uF | AF_MIX (+) --- AF_IN (-) |
| C2 | 0.1uF | +3V3 --- GND (near U1) |
| C3 | 0.1uF | +5V --- GND (near U2) |
| C4 | 1000uF | +5V (+) --- GND (-) (near U2) |
| C5 | 0.1uF | R5 --- GND |
| C6 | 100uF | U2 pin 2 (+) --- GND (-) |
| C8 | 0.1uF | Across C6 legs (pin 2 --- GND), optional |
| C9 | 0.1uF | AF_MIX --- GND (RF filter), optional |
| C10 | 0.1uF | Arduino 3.3V pad --- GND, optional |
| C7 | 10uF | +3V3 (+) --- GND (-), near U1 |
| RV1 | B100K pot | Lug 1 ← C1, Wiper → AF_IN, Lug 3 → GND |
| LED1 | Red 5mm | Anode ← R6, Cathode → GND |
| BTN1 | Micro SW | One leg → D2, other leg → GND |
| BTN2 | Micro SW | One leg → D3, other leg → GND |

---

# Appendix B — Complete ASCII Wiring Diagram

Every component and every wire in one diagram. Pin numbers match the IC pinouts in Section 3.

```
═══════════════════════════════════════════════════════════════════════════════
                        FM RADIO — PHASE 1 — COMPLETE WIRING
═══════════════════════════════════════════════════════════════════════════════

                              ~75 cm ANTENNA WIRE
                                     │
    ┌─────────────────────────────────┤
    │   RRD-102V2.0 (RDA5807M)       │
    │   ┌─────────────────────┐      │
    │   │  Left        Right  │      │
    │   │  side        side   │      │
    │   │                     │      │
    │   │  SDA (1)   ANT (6) ─┼──────┘
    │   │  SCL (2)    NC (7)  │
    │   │   NC (3)  LOUT (8) ─┼──── AF_L ──[R3 1k]──┐
    │   │   NC (4)  ROUT (9) ─┼──── AF_R ──[R4 1k]──┤
    │   │  VCC (5)   GND(10) ─┼──── GND BUS         │
    │   └──┬──┬───────────────┘                      │
    │      │  │                                      │
    │      │  └── +3V3 BUS                       AF_MIX ──[C9 0.1µF]── GND (opt)
    │      │      │                                  │
    │      │    [C2 0.1µF]── GND                     │
    │      │    [C7 10µF+]── GND                   [C1 10µF]
    │      │                                     (+) │ (−)
    │   SDA│(1)                                      │
    │   ───┼────────────────────────────── Arduino A4 (SDA)
    │      │                                         │
    │   SCL│(2)                               RV1 (B100K POT)
    │   ───┼────────────────────────────── Arduino A5 (SCL)
    │      │                              ┌──────────┴──────────┐
    │      │                              │                     │
    │      │                         Lug 1 (top)           Lug 3 (bot)
    │      │                         ← from C1(−)           │
    │      │                              │                GND BUS
    │      │                         Wiper (mid)
    │      │                              │
    │      │                            AF_IN
    │      │                              │
    │      │                              │
    │      │                              ▼
    │      │
    │   I2C PULL-UPS                TDA2822M (DIP-8, in socket)
    │      │                        ┌──────────────────────────┐
    │   [R1 4.7k]── +3V3            │                          │
    │    (SDA─3V3)               ┌──┤  Pin 1 (OUT1)           │
    │   [R2 4.7k]── +3V3        │  │                          │
    │    (SCL─3V3)               │  │  Pin 2 (IN1−) ──[C6 100µF+]── GND
    │                            │  │                  (+) toward pin
    │                            │  │               ──[C8 0.1µF ]── GND  (optional, across C6)
    │                            │  │  Pin 3 (IN1+) ◄── AF_IN (from wiper)
    │                            │  │                          │
    │                            │  │  Pin 4 (GND) ─────── GND BUS
    │                            │  │                          │
    │                            │  │  Pin 5 (IN2+) ─────── GND BUS
    │                            │  │                          │
    │                            │  │  Pin 6 (IN2−) ─────── GND BUS
    │                            │  │                          │
    │                            │  │  Pin 7 (OUT2)    (not connected)
    │                            │  │                          │
    │                            │  │  Pin 8 (VCC) ─────── +5V BUS
    │                            │  │                          │
    │                            │  └──────────────────────────┘
    │                            │         │            │
    │                            │       [C3 0.1µF]   [C4 1000µF+]
    │                            │      (pin8─pin4)   (+5V ── GND)
    │                            │
    │    ZOBEL NETWORK           │
    │    (output stability)      │
    │                            ├──[R5 2.2Ω]──[C5 0.1µF]── GND
    │                            │
    │    SPEAKER                 │
    │    ┌──────────┐            │
    │    │  4Ω 5W   │ SPK+ ◄────┘
    │    │ Speaker   │ SPK− ──────── GND BUS
    │    └──────────┘
    │     (optional: 0.1µF ceramic across SPK+ and SPK−)
    │
    │
    │    POWER LED
    │    +5V BUS ──[R6 1k]──[>|── GND BUS
    │                       LED1
    │                  (anode) (cathode)
    │
    │
    │    SEEK BUTTONS
    │
    │    Arduino D2 ──── BTN1 ──── GND BUS     (Seek Up)
    │    Arduino D3 ──── BTN2 ──── GND BUS     (Seek Down)
    │       (internal ~20k pull-ups to 5V)
    │
    │
    │   ARDUINO UNO CONNECTIONS SUMMARY
    │   ┌────────────────────────────────────────────────────┐
    │   │  3.3V ──[C10 0.1µF]── GND (opt) ──► +3V3 BUS     │
    │   │    5V ─────────── +5V BUS                         │
    │   │   GND ─────────── GND BUS                         │
    │   │    A4 ─────────── SDA (U1 pad 1, R1)              │
    │   │    A5 ─────────── SCL (U1 pad 2, R2)              │
    │   │    D2 ─────────── BTN1 (other leg GND)            │
    │   │    D3 ─────────── BTN2 (other leg GND)            │
    │   └────────────────────────────────────────────────────┘
    │
    │
    │   POWER BUSES
    │   ═══════════
    │   +5V BUS ──── TDA pin 8, C3, C4(+), R6
    │   +3V3 BUS ─── U1 VCC, R1, R2, C2, C7(+), C10 (opt)
    │   GND BUS ──── U1 GND, TDA pin 4/5/6, C2, C3, C4(−), C5,
    │                C6(−), C7(−), C8, C9, C10 (if fitted),
    │                RV1 lug 3, BTN1, BTN2,
    │                LED1 cathode, SPK−, Arduino GND
    │
    │   ┌─────────────────────────────────────────────────────┐
    │   │ ⚠  Speaker negative (SPK−) IS GROUNDED.            │
    │   │    This is SINGLE-ENDED mode — NOT bridge (BTL).    │
    │   │    Do NOT float SPK−. It MUST connect to GND bus.   │
    │   └─────────────────────────────────────────────────────┘
    │
    │
    │   SIGNAL FLOW (left to right)
    │   ════════════════════════════
    │   ANT ► RDA5807M ► LOUT+ROUT ► R3+R4 ► AF_MIX ► C1 ► RV1 ► AF_IN
    │                                                                 │
    │              TDA pin 3 (IN1+) ◄─────────────────────────────────┘
    │                     │
    │              TDA pin 1 (OUT1) ───► SPK+ ──┐
    │                                           │ 4Ω Speaker
    │                                   SPK− ───┘──► GND
    │
═══════════════════════════════════════════════════════════════════════════════
```
