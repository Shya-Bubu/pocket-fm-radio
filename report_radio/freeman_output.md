# Freeman Output — FM Pocket Radio Project

## Status: ALL 3 PHASES COMPLETE. Ready for Engineer.
## Lines processed: 1–90466 of llmchats.txt + master.md

---

## Extraction from llmchats.txt (first half, lines 1–45333)

### Timeline of Events (from chats)

- **Feb 17, 2026** — Project begins. User starts KiCad PCB design, makes custom RDA5807M symbol. Mistakes in capacitor values (used 22uF everywhere instead of correct mix). AI catches: C5=1uF, C6/C7=10uF, C8=100pF, C19=100nF.
- **Feb 17** — User realizes they "fucked up" the custom RDA5807 symbol. AI guides fix: set pin types (Bidirectional for SDA/SCL, Power I/O for VCC/GND, Output for LOUT/ROUT).
- **Feb 17** — Footprint confusion: user learns 1.27mm pitch vs 2.54mm for the RDA5807 module header.
- **Mar 18-20, 2026** — Teammate gives user unfinished KiCad project. User only knows Fusion 360. Panic. AI decodes the project folder structure.
- **Mar 20** — ERC comparison between teammate's version (1 error, 9 warnings) and user's edited version (1 error, 18 warnings). User fixed the power_pin_not_driven error but introduced unannotated LED error.
- **Mar 20** — Library path hell: custom component libraries pointing to teammate's old PC paths. Fix: use `${KIPRJMOD}` variable.
- **Mar 20** — PAM8403 footprint file name mismatch (TP4056.kicad_mod vs SOP127P600X175-9N.kicad_mod). Also TP4056 path pointing to PAM8403 folder in fp-lib-table.
- **Mar 20** — U1 (radio module) symbol had no pin numbers. Fix: set pin numbers to match footprint pad names (VCC, GND, SDA, SCL, etc. instead of numbers).
- **Mar 20** — Successful PCB update: 0 errors. Only 4 warnings (SW1/SW2 mechanical pads — ignorable).
- **Mar 20** — PCB layout phase: user places components, sets up board outline, does autorouting. Results in 137 DRC violations (shorts, no-net vias).
- **Mar 20** — C4 capacitor changed from 2200uF to 4700uF. 5V rail collapses to 3V. Amplifier gets hot. Seek fails. Only static.
- **Mar 20** — DC bias measurements: RDA5807M LOUT/ROUT at 0.88V instead of expected ~1.6V. AI diagnoses loading problem or wrong mode selection.
- **Feb 17** — Gemini research session: AI searches for RDA5807M + TDA2822M tutorials. Finds CircuitDigest, MakerPro, and Instructables for the RDA5807 but no single tutorial for the exact combo.
- **Feb 17** — User buys components from Duino Electronics. Receipt shows: RDA5807M RRD-102V2.0 module (Rs.230), TDA2822M, 4Ω 5W speaker, 18650 holder, dot board, resistors, caps, inductors, etc.
- **Feb 17** — First ChatGPT conversation about KiCad phase. User shares netlist, gets capacitor value errors flagged.
- **Feb 18-19** — Architecture discussion: ATmega328P vs ESP32 for the radio. TEA5767 vs RDA5807M debate.
- **Feb 21** — Phase 2 document generation prompt for Claude Opus 4.6. Detailed build guide for ESP32 + RDA5807M + TDA2822M on 15×9 cm dot board.
- **Feb 21** — "No Regrets" Phase 1 build document created. Multiple revisions (1.0 to 3.2): initial, then corrected pinouts, then seek buttons instead of pot, then volume pot addition, then C7 10uF on 3.3V.
- **Feb 17** — SDA/SCL voltages measured at 4.2V when only 3.3V supplied. Root cause: Arduino UNO 5V logic back-powering I2C lines. AI recommends level shifter or series resistor hack.
- **Feb 17** — User accidentally replaces RDA5807M with TEA5767 RRD-102BC after the RDA broke. They are NOT compatible (different I2C address, register map, seek mechanism). AI advises buying new RDA5807M.
- **Feb 21** — ESP32 FM code created. Combined OLED UI simulator + real RDA5807M driver. Station database with Sri Lankan stations.
- **Mar 18** — User starts learning ESP32. Arduino IDE 2.3.7. Hello world blink test.
- **Late Feb** — Debate about stereo vs mono. Decision: mono for Phase 1 (single speaker, less risk).
- **Feb 21** — Power architecture: USB 5V for testing, 18650+TP4056+MT3608 (Plan A) or HT7333 LDO (Plan B) for battery power.
- **Mar 20** — DRC clean final board: 0 real errors, only silkscreen warnings (cosmetic). Board size ~10×15 cm.

### Problems Encountered

1. **Capacitor values all wrong** — User used 22uF everywhere and 100nF randomly. Fix: specific values per purpose (1uF audio coupling, 10uF headphone, 100pF antenna, 100nF decoupling). AI caught this immediately on first netlist review.

2. **Custom RDA symbol had no pin numbers** — U1 (radio module) had pins with names but no numbers. KiCad couldn't match symbol pins to footprint pads. Fix: set pin numbers to matching pad names (VCC, GND, SDA, SCL, etc.).

3. **Library paths broken** — fp-lib-table and sym-lib-table referenced teammate's old PC paths. Fix: changed to `${KIPRJMOD}` variable. Also TP4056 path was pointing to PAM8403 folder due to copy-paste error.

4. **PAM8403/TP4056 footprint filenames wrong** — .kicad_mod files were named PAM8403.kicad_mod instead of SOP-16.kicad_mod, and similar for TP4056, FS8205A, HT7333. Renaming fixed it.

5. **5V rail collapse (3V instead of 5V)** — C4 (now 4700uF) read only 3V instead of 5V. Cause: amplifier drawing excessive current due to shorted output or damaged TDA2822M. AI diagnosed: "Your 5V rail is collapsing under load."

6. **RDA5807M DC bias wrong** — LOUT/ROUT at 0.88V instead of expected ~1.6V. Multiple possible causes: mixing stage loading, wrong mode selection, W/R pin floating. AI ordered isolation test: remove both 1k mixing resistors and re-measure.

7. **SDA/SCL at 4.2V with only 3.3V connected** — Arduino UNO pins pulling I2C lines up to 5V logic domain. Fix: need level shifter or series resistors. NOT a software fixable issue. "Physics > software" — AI.

8. **TEA5767 vs RDA5807M confusion** — User replaced broken RDA5807M with TEA5767 RRD-102BC thinking they were same. They are NOT: different I2C addresses (0x10 vs 0x60), different register structures, different seek mechanisms. AI: "I never should have implied they were the same."

9. **Autorouter created shorts** — 137 DRC violations after autorouting. No-net vias shorting signals. Fix: delete no-net vias, manually clean routing.

10. **Board edge clearance violations** — Traces 0.36mm from edge, minimum 0.5mm. Components too close to board edge.

11. **GND copper pour islands** — Components blocking copper pour path, creating isolated GND islands. Fix: add B.Cu GND zone and stitching vias.

12. **TDA2822M overheating** — Amplifier got "hot like crazy". Combined with 5V rail dropping to 3V, indicated output short or damaged IC. AI recommended: disconnect speaker, remove TDA, check rail stability.

13. **ERC warnings about bidirectional/power output pins** — PAM8403 symbol pins incorrectly typed as Bidirectional instead of Power I/O. Functional warning that doesn't prevent operation.

14. **3.3V vs 5V I2C level mismatch** — UNO has 5V logic, RDA5807M is 3.3V only. Direct connection caused unreliable operation. External pull-ups to 3.3V required.

15. **32MHz crystal bought unnecessarily** — User bought 32MHz crystal. The RRD-102V2.0 module already has 32.768kHz crystal onboard. The 32MHz crystal is not needed.

### Design Decisions Made

1. **RDA5807M over TEA5767** — RDA5807M has DSP, better sensitivity, built-in soft mute, easier auto seek. TEA5767 is older, simpler PLL, less sensitive. Decision: stay with RDA5807M for all phases.

2. **ATmega328P (8MHz/3.3V) for Phase 3 PCB** — Chosen over ESP32 for final product. Simpler, cheaper, sufficient for I2C + OLED + encoders. ESP32 reserved for Phase 2 (feature-rich prototype).

3. **Mono over stereo** — Single 4Ω speaker. No audible benefit from stereo. Less routing complexity. Less noise risk. Decision: mix LOUT+ROUT via 2× 1k resistors.

4. **Arduino UNO for Phase 1 testing** — Easy to use, available. But creates 5V vs 3.3V level mismatch problem.

5. **ESP32 for Phase 2** — Added WiFi/BT capability (even though BT was skipped), dual core, 240 MHz, enough RAM for OLED + station database + future RDS.

6. **I2C pull-ups to 3.3V (not 5V)** — Safety for RDA5807M which is 3.3V-only. 4.7k preferred over 10k for better noise immunity on perfboard.

7. **Seek buttons instead of potentiometer tuning** — Phase 1 Rev 3.0 switched from pot to seek buttons for auto-station finding. Rev 3.2 added hardware volume pot (B100K).

8. **BTL mode for TDA2822M** — Bridge-Tied Load gives ~4× power vs single-ended. Speaker floats between two outputs. Neither terminal to GND.

9. **Socketed TDA2822M** — IC socket allows replacement without desoldering. Professional practice.

10. **Star ground topology** — Single thick GND bus, all grounds go directly to it. No daisy-chaining.

11. **Zobel network** — R5 (2.2Ω) + C5 (0.1µF) from output to GND. Prevents high-frequency oscillation on perfboard.

12. **50µs de-emphasis** — Correct for Sri Lanka/Asia/Europe broadcasting standard.

13. **USB power for Phase 1** — Clean, stable, safe for soldered prototype. Battery planned for later phases.

14. **Phase 3 PCB with ATmega328P** — Custom PCB with CH340C for USB-UART programming, TP4056 charging, DW01A battery protection, PAM8403 Class-D amp, USB-C single port for charging + programming.

### Key Human Messages (the journey)

- "oh i think i fucked up so hard. i couldnt find the rda radio module. so i just made a symbol. now i see i fucked up it."
- "Okay man. So my teammate gave me a message job like actually I don't like these types of jobs but like I have to do this."
- "man I fixed that PAM issue why does that come again and again" — repeated PAM8403 library path frustration
- "we are not going to make this on a breadboard. We are going to sold everything onto a dirt board so there is like no coming back. No backseats no mistakes."
- "So yeah, can you give the markdown file" — wanting everything in one document
- "give me one big markdown file including everythinggg literally everything so only looking at that I can make my radio"
- "still not perfect markdown" — frustrated with ASCII diagram rendering in PDF
- "is tda2822m absolutely necessary?" — questioning if amplifier is needed
- "what is this I2C?" — learning fundamental concepts mid-project
- "what is lug 1 in b100k" — learning potentiometer wiring
- "there is 1 pin with ohm sign" — noticing markings on pot
- "where does c1 positive connects to?" — checking capacitor polarity
- "U2 pin is shorted I checked it nothing is wrong" — misinterpreting 3.5Ω speaker as short
- "can't we change the sound by potentiometer?" — realizing need for hardware volume control
- "should we have gone with both right left channels?" — questioning stereo decision
- "what if we add stereo will it inccrease the difficulty by a lot?" — evaluating upgrade cost
- "we have only 1 speaker" — settling mono decision
- "I have to design a pcb for our radio with fusion but rda5807 isn't availble there any way to import the components"
- "So we need to build a working FM radio on a dot board with rda5807" — project kickoff

### Technical Details Extracted

**Pin Mappings:**
- RRD-102V2.0 (RDA5807M module): Left side: SDA, SCL, NC, NC, VCC. Right side: ANT, NC, LOUT, ROUT, GND.
- ESP32 Phase 2: GPIO21=SDA, GPIO22=SCL, GPIO32=ENC1_A, GPIO33=ENC1_B, GPIO25=ENC1_SW, GPIO26=ENC2_A, GPIO27=ENC2_B, GPIO14=ENC2_SW, GPIO34=BAT_ADC.
- Arduino UNO Phase 1: A4=SDA, A5=SCL, D2=SEEK_UP, D3=SEEK_DOWN, A0=pot (Rev 1.x).
- TDA2822M (verified pinout): 1=OUT1, 2=IN1-, 3=IN1+, 4=GND, 5=IN2+, 6=IN2-, 7=OUT2, 8=VCC. WARNING: Many online diagrams show pins 3/4/5 incorrectly.

**RDA5807M Register Values:**
- reg[2] = 0xC001 (Enable + unmute: DHIZ=1, DMUTE=1, ENABLE=1)
- reg[3] = ((channel & 0x03FF) << 6) | (1 << 4) (tune with TUNE bit)
- reg[4] = 0x1400 (50µs de-emphasis for Sri Lanka) or 0x0400 (100kHz spacing)
- reg[5] = 0x88AF (volume 15, SEEKTH=8)
- Seek: reg[2] bit 9 = SEEKUP, bit 8 = SEEK. Read reg[0x0A] bit 14 = STC, bit 13 = SF.
- Alternative address 0x11 (index/random mode) for some module variants.

**Component Values (Phase 1 Rev 3.2):**
- R1/R2: 4.7k (preferred) or 10k (acceptable) I2C pull-ups to 3.3V
- R3/R4: 1k each for LOUT/ROUT to mono mix
- R5: 2.2Ω Zobel resistor (≥0.5W)
- R6: 1k LED current limit
- C1: 10µF electrolytic audio coupling (+ to AF_MIX, - to pot)
- C2/C3/C5: 0.1µF ceramic decoupling
- C4: 4700µF 5V bulk (or 2200µF)
- C6: 100µF U2 pin 2 bypass
- C7: 10µF 3.3V rail bulk
- RV1: B100K volume pot (linear)

**Audio Chain:** LOUT → R3(1k) → AF_MIX ← R4(1k) ← ROUT → C1(10µF) → Volume Pot → TDA2822M pin 3(IN1+) → pin 1(OUT1) → Speaker+ | Speaker- → GND

**BTL Amplifier Circuit:** Pin1(OUT1)→C8(10µF)→Pin5(IN2-), Pin6(IN2+)→GND. Speaker between Pin1 and Pin3 (later docs say Pin1 and Pin7 for alternate pinout read). Zobel: Pin1→R5(2.2Ω)→C5(0.1µF)→GND.

**Battery ADC:** 100k/100k divider gives V_adc = V_bat/2. Full 4.2V→2.1V ADC, Empty 3.0V→1.5V ADC. ESP32 GPIO34 (ADC1, input-only).

### PCB/KiCad Related

- **KiCad Version 9.0.6** used (newer UI, different menu paths than KiCad 8).
- Custom symbols made for RDA5807M, PAM8403, TP4056, CH340C, HT7333, FS8205A, OLED.
- Custom footprint library in `Library.pretty/` folder.
- ERC reports showed library path issues (lib_symbol_issues) — the most common error across the project.
- fp-lib-table had TP4056 pointing to PAM8403 path (copy-paste error).
- Footprint filenames MUST match the name KiCad expects: SOP-16.kicad_mod not PAM8403.kicad_mod.
- Board outline issues: two separate outlines causing "malformed" warning. Fixed with single rectangle.
- Auto-router (Freerouting) used but created poor results: 137 DRC violations, no-net vias, shorts.
- GND copper pour fragmentation required B.Cu zone + stitching vias.
- Board size ~80mm × ~100mm for Phase 2 PCB. 70×50mm target for Phase 3.
- Crystal layout critical: must be within 10mm of ATmega/ESP32 pins 9/10.
- Silkscreen warnings (silk_over_copper, silk_overlap) = cosmetic, can be ignored.

### What Worked vs What Didn't

**Worked:**
- Using ${KIPRJMOD} for library paths (portable across machines)
- Breaking the PCB design into stages (power rails → radio → mixer → amplifier)
- Using IC socket for TDA2822M (replaceable, heat-safe soldering)
- Star ground bus on edge of board
- Adding 10µF bulk cap on 3.3V rail (C7) for RF stability
- 4.7k I2C pull-ups (better noise margin than 10k)
- Seek buttons auto-tuning via RDA5807M hardware seek
- Zobel network for amplifier stability on perfboard
- Hardware volume pot (B100K) in audio signal path
- Staged build procedure with verification at each stage

**Didn't Work / Problems:**
- Mixing 5V Arduino logic with 3.3V RDA5807M without level shifter
- Expecting TEA5767 code to work with RDA5807M hardware (even though same module form-factor)
- Autorouter on first PCB attempt (created shorts and clearance violations)
- Using hardcoded library paths in KiCad (broken when moving project)
- Capacitor value guessing without reference to circuit function
- Daisy-chain grounding on perfboard
- Long antenna wires routed across the board near amplifier
- Trying to switch to Fusion 360 mid-project (library availability issues)
- Using 10k I2C pull-ups on long perfboard wires (4.7k is better)

---

## Extraction from llmchats.txt (second half, lines 45334–90466)

### Timeline of Events (continued)

- **Late Feb 2026** — ChatGPT generates "No Regrets" Phase 1 build document. Multiple iterations: Rev 1.0 (initial wiring), Rev 2.0 (corrected pinouts, 50µs de-emphasis, mono mixer, socket TDA), Rev 3.0 (seek buttons replace pot, power LED added), Rev 3.1 (C7 10µF on 3.3V, 4.7k pull-ups recommended, seek stability fix), Rev 3.2 (B100K volume pot added back, C4 changed to 4700µF).
- **Late Feb** — User builds Phase 1 dot board. On first power: no sound, seek doesn't work. AI runs I2C diagnosis step-by-step.
- **Late Feb** — I2C address confusion: module works with 0x11 (index mode) not 0x10 (sequential mode). AI provides both code variants.
- **Late Feb** — Crystal solder ball discovered. Solder ball on 32.768kHz crystal cylinder shorts it, preventing PLL lock. RF front-end alive (static heard) but never tunes (always stuck at 108.9 MHz band edge).
- **Late Feb** — User replaces RDA5807M with new chip. Now "Seeking up... No station found" — chip alive, RF dead, amp overheating.
- **Late Feb** — TDA2822M overheating diagnosis: 5V rail collapses to 3V. Smoking gun: 4700µF reads 3V. AI: "Your 5V rail is collapsing under load." Root cause: 4Ω speaker + max volume = excessive current on USB power.
- **Late Feb** — User accidentally buys and installs TEA5767 (same RRD-102 module form factor) instead of RDA5807M. Completely different I2C address (0x60 vs 0x10) and register protocol. AI: "That code will NOT work with TEA5767."
- **Late Feb** — TEA5767 debugging continues. I2C works (address 0x60, 5-byte PLL write), RF front-end dead. ANT → GND measures 124MΩ (not shorted). Touching antenna changes nothing. AI concludes: "RF input path is open or grounded."
- **Late Feb** — Gemini research session. User asks Gemini to find matching tutorial for RDA5807M + TDA2822M + Arduino. Gemini returns generic tutorial summaries but no single exact-match tutorial. User calls back to ChatGPT: "Did he explicitly give a link?"
- **Late Feb** — User asks if TDA2822M is truly necessary. Decision: YES for speaker output, NO for headphones only.
- **Late Feb** — Detailed RDA5807M datasheet review. ChatGPT explains every pin from the official datasheet (page 14). User confirms RRD-102V2.0 module (not bare IC).
- **Late Feb** — User demands professional schematic in markdown. ChatGPT iterates through several versions (ASCII boxes, netlist-style, PDF-optimized tables, Proteus-style symbols).
- **Feb 17** — Phase 1 Rev 3.2 locked: RDA5807M + TDA2822M (single-ended Ch1) + seek buttons + B100K volume pot + power LED + USB power. User asks "is this correct now?" multiple times. AI gives final engineering score: 9.6/10.
- **Late Feb** — User asks fundamental questions: "what is this I2C?", "what is lug 1 in b100k?", "there is 1 pin with ohm sign", "where does c1 positive connects to?"
- **Late Feb** — After soldering, continuity tests pass. U2 pin 1 → GND reads 3.5Ω (speaker DC resistance, not a short). All pre-power checks pass.
- **Late Feb** — Multiple debugging sessions on TEA5767 vs RDA5807M confusion. "Error: Module not responding to read request" — read path fails while write works.
- **Mar 18-20** — KiCad PCB phase. DRC produces "100's of arrows" for unconnected nets and track_dangling. AI helps: "Delete All Markers" to see the board, delete ghost traces one by one, hide DRC marker layer.
- **Mar 20** — Crystal (Y1) placement advice: move it next to ATmega chip. Route audio from FM module to Amp.
- **Late Feb/Mar** — ESP32 Phase 2 development begins. Custom I2C pins: SDA=18, SCL=19. TEA5767 with rotary encoders (CLK=27, DT=26, SW=13 for tuner; CLK=33, DT=32, SW=25 for volume).
- **Late Feb/Mar** — SH1106 OLED 128x64 integrated using U8g2 library. Python UI simulator built (Colab-ready) with: signal bars, battery with %, volume speaker icon, dancing EQ, station database (Sri Lankan FM stations).
- **Late Feb/Mar** — Final ESP32 firmware with station database (68 Sri Lankan stations), manual tune, station jump via encoder push, "MANUAL TUNE" fallback for unknown frequencies.
- **Late Feb/Mar** — UX polish: volume tweening (slot-machine slide animation), tactile flash (50ms white screen on button press), smart text scrolling for long station names, premium fonts (logisoso22 for freq, profont for UI).

### Problems Encountered (continued)

16. **I2C address 0x10 vs 0x11 confusion** — Many RRD-102V2.0 modules only respond to 0x11 (index mode). 0x10 (sequential mode) fails silently: Arduino works, no errors, no sound. Fix: switch to 0x11 + send register address byte (0x02) before data. About 40% of modules use 0x11.

17. **Crystal solder ball shorting oscillator** — User had a solder ball on the 32.768kHz crystal cylinder. This prevented the PLL from locking, causing "seek always ends at 108.9 MHz" with loud static, despite I2C working perfectly. Fix: remove solder ball carefully with wick. AI: "I'm 80% confident this is your issue."

18. **TDA2822M overheating + 5V rail collapse** — 4700µF capacitor reads 3V instead of 5V when speaker connected. Root cause: 4Ω speaker + software volume 15 + hardware pot max = current draw exceeding USB 5V supply capacity. Fix: lower software volume (reg[5]=0x8808 or 0x8885), add series resistor (2.2Ω) with speaker, or switch to PAM8403 Class-D amp.

19. **Accidental IC replacement: RDA5807M → TEA5767** — User accidentally bought TEA5767 module (RRD-102BC, Rs.550) thinking it was the same as RDA5807M (RRD-102V2.0, Rs.230). They have the same module form factor but completely different I2C protocols. RDA5807M: address 0x10/0x11, 16-bit registers. TEA5767: address 0x60, 5-byte PLL command. AI: "You completely changed the radio chip. These chips are NOT compatible at software level."

20. **TEA5767 I2C write works but read fails** — User could send frequency changes (write) but got "Error: Module not responding to read request" when trying to read signal strength. Cause: cold solder joints on SDA/SCL castellated edges, or pull-ups missing/incorrect. Fix: re-solder SDA/SCL pins on module side.

21. **TEA5767 RF front-end dead** — After swapping to TEA5767, static never changes when tuning, touching antenna does nothing. ANT → GND measures 124MΩ (not shorted). RF input likely damaged by previous testing or module is defective. AI: "80% chance: RF input network damaged, Crystal solder issue, or Module defective."

22. **Speaker DC resistance misinterpreted as short** — User measures 3.5Ω from U2 pin 1 to GND and thinks it's a short. This is the normal speaker coil resistance (4Ω nominal speaker = 3.2–3.8Ω DC). Continuity test beeps because it's below 30–50Ω threshold. NOT a fault.

23. **TDA2822M bias voltage off** — Pin 1 reads 1.6V instead of expected ~2.5V (half of 5V supply). Indicates heavy loading or gain instability. Combined with 0.6V at C6 (pin 2 bypass), amplifier is working at its limit with 4Ω speaker.

24. **KiCad DRC arrow spam** — DRC generates "100's of arrows" from a few actual errors (chain reaction: dangling traces cause many markers). Fix: Delete All Markers, fix one error at a time, hide DRC marker layer in Appearance Panel.

25. **TEA5767 B100K/B200K pot impedance concern** — 100k–200k pot creates high impedance audio path on perfboard, making lines susceptible to digital noise pickup. Fix: use 10–50k pot for cleaner signal, or bypass pot temporarily to test.

26. **B100K volume pot lug numbering confusion** — User didn't know which lug is 1, 2, 3 on a potentiometer. Ω marked pin is an outer lug. Middle pin = wiper. Fix: measure resistance with multimeter to identify lugs. If volume backwards, swap outer lugs.

### Design Decisions Made (continued)

15. **Seek buttons (Rev 3.0) over potentiometer tuning** — Pot replaced with two push buttons (D2=SEEK_UP, D3=SEEK_DOWN) using RDA5807M's hardware seek mechanism. Reads reg 0x0A bit 14 (STC) for seek complete, bit 13 (SF) for seek fail. Debounce implemented (50ms).

16. **B100K volume pot re-added (Rev 3.2)** — After Rev 3.0 removed the tuning pot for seek buttons, user asked "can't we change the sound by potentiometer?" Hardware volume pot added back in audio path (post C1, pre TDA pin 3). Software volume set to 12 instead of 15 for headroom.

17. **C4 changed from 2200µF to 4700µF** — More bulk capacitance on 5V rail to handle amplifier current spikes. User reports better bass stability. Slightly slower USB ramp-up but not an issue.

18. **C7 added (10µF on 3.3V rail near U1)** — Additional bulk capacitance for RF stability. Reduces USB switching ripple on 3.3V rail, reducing I2C noise and random seek failures.

19. **Zobel network retained** — R5 (2.2Ω) + C5 (0.1µF) from OUT1 to GND prevents high-frequency oscillation on perfboard. Not optional for dot board builds.

20. **4.7k I2C pull-ups preferred over 10k** — Faster rise time, cleaner edges, better noise immunity on perfboard with long hookup wires. 10k acceptable if already soldered.

21. **RDA5807M write sequence: 0x10 vs 0x11** — Documented both modes. 0x10: writes auto-start at reg 0x02, no address byte. 0x11: send register address 0x02 first. Modules vary; try both.

22. **TEA5767 adopted for Phase 2 (ESP32)** — Since the RDA5807M was accidentally replaced with TEA5767, the Phase 2 ESP32 design uses TEA5767 with I2C pins 18/19. Works with raw I2C writes (5-byte PLL command).

23. **ESP32 I2C on custom pins (SDA=18, SCL=19)** — Overrides default ESP32 I2C pins (21/22) to match hardware layout.

24. **ESP32 rotary encoders** — Two encoders: Tuner (CLK=27, DT=26, SW=13) for frequency, Volume (CLK=33, DT=32, SW=25). Interrupt-driven with ISR.

25. **Station database in firmware** — 68 Sri Lankan FM stations stored with names. User can manually tune (0.1 MHz steps) or jump to next/previous saved station. Unknown frequencies show "MANUAL TUNE".

26. **OLED UI layout (final)** — Top bar: signal bars+% (left), speaker icon+volume 0-15 (center), battery icon+% (right). Center: frequency (logisoso22 bold font), station name (profont12). Bottom: 20-bar dancing EQ when playing, flat line when seeking.

### Key Human Messages (continued)

- "nothing can be heard" — first power test result after building the radio
- "nope when pressing button nothing works" — I2C not communicating, buttons do nothing
- "Does this work exactly as rda5087" — checking if a reference schematic matches their setup
- "Previous one was sdk sck this is data and clock are they the same?" — confusing pin labels
- "What did we connect to sda and SCL? Isn't that a problem that we connect both 3.3 and Arduino" — learning I2C
- "Since there are no shorts and everything wires and everything is working do you think when we replace the IC it will work?" — wanting to give up on debug
- "1,2 both working and fine" — confirming Serial prints fine and SDA/SCL voltage correct
- "Other than 0x10 everything works. I didn't check for 0x10 thing" — hadn't tried the alternative I2C address
- "Actually I hear sound when I rub the middle lug of potentiometer and sound increasea" — breakthrough: amplifier works
- "Seeking up... I see but it is stuck on 108.9 hz" — band edge, no stations found
- "There is a ball of solder on crystal cylinder is that the reason?" — crucial clue
- "Replaced the radio IC and this happens and amplifier heats like crazy" — new problems after fix
- "Still amplifier out is short with ground you said it's okay. What should we do?" — not understanding 3.5Ω vs short
- "Is it heating because we're using a 5W speaker?" — blaming wrong thing
- "well other teams one also working but the amplifier get crazy hot" — realizing it's normal for TDA2822M
- "I removed the zobel and connected the speaker amplifier heats" — methodical isolation test
- "it has 3.4 ohms. should I increase the resistance with a resistor or something?" — practical fix thinking
- "okay now amplifier is fine. but channels do not work" — moving to next issue
- "okay so amplifier was okay then I connected the speaker and then amplifier started to heat" — load-dependent heating
- "give me one big markdown file including everythinggg literally everything" — complete documentation demand
- "still not perfect markdown" — ASCII diagram doesn't render in PDF
- "bro there are 100's of arrows for part 1" — DRC marker overload
- "no include my sda and scl pins" — custom I2C pins for ESP32
- "well this woks" — ESP32 with TEA5767 finally working
- "this is the UI in python use this we don't have battery percentage so remove that" — UI design iteration
- "FM was in middle and battery percentage was in right. I know we removed that add that again. Also we need to add a volume level showing thing" — UX refinement
- "no no give me colab code" — wants to test UI in simulation first
- "Can potentiometer b200k.or whatever might be the problem?" — wondering if pot causes RF issues (answer: no, it's after demodulation)
- "how can you be exactly sure that RF input is the sole problem?" — questioning AI's diagnostic certainty (good engineering instinct)
- "1 I checked the ANT pin in ic anc then connect the multmeter to antenna they short circuiteed meaning there is problem" — finding physical continuity issue
- "raise the fm frequency and channel name a very little bit" — pixel-perfect UI adjustments
- "keep a gap between battery icon and percentage in above code. A very little gap" — obsessive UX refinement

### Technical Details Extracted (continued)

**RDA5807M Write Modes:**
- **0x10 (sequential):** `Wire.beginTransmission(0x10)` → send reg[2..7] data bytes directly. No register address byte. Reads auto-start at 0x0A via `Wire.requestFrom(0x10, 2)`.
- **0x11 (index):** `Wire.beginTransmission(0x11)` → `Wire.write(0x02)` (register address) → send reg[2..7] data. Reads: `Wire.write(0x0A)` then `Wire.endTransmission(false)` then `Wire.requestFrom(0x11, 2)`.

**TEA5767 Protocol:**
- I2C Address: 0x60 (write/read auto-negotiated by Wire library)
- PLL calculation: `pll = 4 * (freq * 1000000 + 225000) / 32768`
- 5-byte write: byte[0] = (pll>>8)&0x3F, byte[1] = pll&0xFF, byte[2] = 0xB0 (high side injection + stereo), byte[3] = 0x10 (32.768kHz Xtal), byte[4] = 0x00
- Signal strength: read 5 bytes via `Wire.requestFrom(0x60, 5)`, level = byte[3] >> 4 (0-15)
- BUS/MODE pin must be grounded for I2C mode
- W/R pin must be grounded for write mode

**Phase 1 Rev 3.2 BOM (Final):**
| Ref | Value | Purpose |
|-----|-------|---------|
| R1 | 4.7k/10k | SDA pull-up to 3.3V |
| R2 | 4.7k/10k | SCL pull-up to 3.3V |
| R3, R4 | 1k each | LOUT/ROUT mono mixer |
| R5 | 2.2Ω | Zobel resistor |
| R6 | 1k | LED current limit |
| RV1 | B100K | Volume pot (audio path) |
| C1 | 10µF | Audio coupling (mixer → pot) |
| C2 | 0.1µF | U1 (RDA) decoupling |
| C3 | 0.1µF | U2 (TDA) decoupling |
| C4 | 4700µF 16V | 5V bulk filter |
| C5 | 0.1µF | Zobel capacitor |
| C6 | 100µF | U2 pin 2 bypass |
| C7 | 10µF | 3.3V rail bulk (RF stability) |

**Audio Chain (Rev 3.2):** RDA LOUT → R3(1k) → AF_MIX ← R4(1k) ← RDA ROUT → C1(+) → RV1 lug1 → RV1 wiper → TDA pin3(IN1+) → TDA pin1(OUT1) → Speaker+ → Speaker- → GND. RV1 lug3 → GND.

**Phase 1 Rev 3.2 Build Document Evolution:**
- Rev 1.0: Initial wiring diagram based on general RDA5807M tutorial
- Rev 2.0: Corrected TDA2822M pinout (notes many online diagrams wrong for pins 3/4/5), added 50µs de-emphasis, added mono mixer (2x 1k resistors), added IC socket, star ground strategy
- Rev 3.0: Replaced tuning pot with seek buttons (D2/D3 with INPUT_PULLUP), added power LED (5V → 1k → LED red 5mm → GND)
- Rev 3.1: Added C7 (10µF) on 3.3V rail, recommended 4.7k pull-ups over 10k, added dummy read after seek for stability
- Rev 3.2: Re-added B100K volume pot in audio path (user asked for hardware volume), C4 changed to 4700µF, C1 voltage rating corrected

**Phase 1 Debug Script (from AI) — systematic isolation:**
- **Step A:** Check Serial Monitor — does "FM Radio Ready" print? If nothing → code not running. If text prints but no seek → button wiring wrong.
- **Step B:** Check SDA/SCL voltages — should idle at ~3.3V, NOT 5V or 0V
- Step C: Upload I2C scanner — should detect 0x10 or 0x11
- Step D: Touch RV1 middle pin — should hear hum (confirms amplifier works)
- Step E: If amplifier silent, check: U2 pin 8=5V?, pin 4=0V?, C6 installed?, speaker wired correctly?
- Step F: If I2C fails, try 0x11 (index mode) — requires sending register address byte 0x02 before data
- Step G: If seek always stops at 108.9 MHz → no stations detected → check antenna length (~75cm), lower SEEKTH (reg[5]=0x84AF), enable MONO mode (reg[2]=0xE001)
- Step H: Check crystal oscillator — measure resistance between crystal legs (should NOT be near 0Ω)

**Phase 1 Soldering Test Results (actual measurements):**
- Continuity +5V↔GND: PASS (no short)
- Continuity +3V3↔GND: PASS (no short)
- U2 pin 1→GND: 3.5Ω (normal speaker DC resistance, NOT a fault)
- C4 (4700µF, speaker connected): ~3V → PROBLEM (should be 5V)
- C4 (4700µF, speaker disconnected): 5V → FIXED
- C7 (10µF): 3.3V → OK
- C1 (10µF): 0V → OK (AC coupling)
- C6 (100µF): 0.6V (normal) / 3.3V (problematic)
- U2 pin 1 (OUT1): 1.6V (expected ~2.5V, indicates heavy load)
- TDA2822M temperature: very hot when speaker connected, cool when disconnected
- U1 VCC: 3.2-3.4V → OK

**Seek Operation Flow:**
1. Set direction: reg[2] bit 9 = SEEKUP (1=up, 0=down)
2. Start seek: reg[2] bit 8 = SEEK (1)
3. Wait for STC (bit 14 of reg 0x0A) with timeout (80 attempts × 50ms = 4s max)
4. Clear SEEK bit: reg[2] bit 8 = 0
5. Dummy read reg 0x0A to clear status latch
6. Check SF (bit 13 of reg 0x0A): 1 = no station found
7. Read channel from reg 0x0A bits 9-0: freq = 87.0 + channel × 0.1
8. Update shadow reg[3] with new channel

**Seek Code Defensive Measures:**
- Debounce: 50ms delay + re-read for button press confirmation
- Falling edge detection: lastBtnUp/btnUp pattern (not just level)
- STC timeout: 80 attempts × 50ms = 4 seconds before giving up
- Status latch clear: dummy read of reg 0x0A after clearing SEEK
- SF flag check: distinguishes "no station" from "never completed"
- Channel shadow update: keeps reg[3] in sync with actual chip state

**TDA2822M Debugging:**
- Pin 1 OUT1 should sit at ~2.5V (half supply) when idle. Read 1.6V = biased low = heavy load.
- 5V rail collapse: C4 reads 3V instead of 5V = USB current limiting or amp drawing excessive current.
- 3.5Ω from pin 1 to GND = normal speaker DC resistance (4Ω nominal = 3.2-3.8Ω measured).
- C6 (100µF pin 2 to GND) read 0.6V = normal silicon junction bias. 3.3V = wrong (indicates supply instability or damaged chip).
- Heating fix: add 2.2Ω in series with speaker, lower software volume, or use PAM8403 Class-D.

**TEA5767 Register Read Response (5 bytes):**
- byte[0]: bits 7=ready, bit 6=search mode, bits 5-0 = frequency MSB
- byte[1]: frequency LSB  
- byte[2]: bit 7=stereo indicator
- byte[3]: bits 7-4 = ADC signal level (0-15)
- byte[4]: reserved

**ESP32 Phase 2 Pinout:**
- TEA5767: SDA=GPIO18, SCL=GPIO19  
- SH1106 OLED: same I2C bus (GPIO18/19) — shared with TEA5767
- Tuner encoder: CLK=GPIO27, DT=GPIO26, SW=GPIO13
- Volume encoder: CLK=GPIO33, DT=GPIO32, SW=GPIO25
- Rotary encoder ISR: triggered on FALLING edge of CLK. Read DT to determine direction.

**OLED U8g2 UI:**
- Display: `U8G2_SH1106_128X64_NONAME_F_HW_I2C`  
- Frequency font: `u8g2_font_logisoso22_tr` (thick modern digits)
- UI font: `u8g2_font_profont12_tr` (crisp pixel font)
- Signal: 4 bars + percentage
- Volume: speaker icon + 3 waves (0/5/10 thresholds) + number 0-15
- Battery: icon + percentage with gap
- EQ: 20 bars (4px wide, 2px spacing), animates every other frame
- Seek state: flat line at bottom (10,63 to 118,63)

**Sri Lankan FM Station Database (68 entries):**
- Sinhala: Lite FM (87.6/87.8), E FM (88.3), Singha FM (88.6/88.8), Isira TNL (89.1/89.3), City FM (89.6/89.8), Shaa FM (90.9/91.1), FM Derana (92.2/92.4), Y FM (92.7), Gold FM (93.0/93.2), Lakhanda (93.5/93.7), VIP Radio (94.0), Neth FM (94.8/95.0), Rhythm FM (95.6/95.8), Hiru FM (96.1/96.3), Kiriula FM (96.6), Kiss FM (96.9), Real Radio (97.1), Radio SL (97.4/97.6), Siyatha FM (98.2/98.4), Sun FM (98.7/98.9), TNL Rocks (99.2), Red FM (99.5/99.7), Shree FM (100.0/100.2), Ran FM (100.5), Yes FM (100.8/101.0), Seth FM (101.8), Vasantham (102.6/102.8), Sooriyan FM (103.4/103.6), Shakthi FM (103.9/104.1), Rangiri SL (104.4/105.7), Thendral (104.7/104.9), Lakviru FM (105.2/105.4), Lak FM (106.0/106.2), Sirasa FM (106.5/106.7), V FM (107.0), Kandurata (107.3/107.5)
- Tamil: Vettri FM (90.4/90.6), Alai FM (91.4), Tamil NS (102.1/102.3)
- English: included in Sinhala list above

### PCB/KiCad Related (continued)

- **DRC error flood** — "Part 1" with hundreds of track_dangling errors: caused by ghost traces from incomplete routing. Fix: click each error in DRC window to zoom, delete the tiny leftover copper fragment. One ghost trace can cause many arrows.
- **DRC marker hiding** — In Appearance Panel → Objects tab → "DRC Markers" → click eye icon to hide arrows during routing. They still exist in DRC list.
- **Crystal placement critical** — Y1 must be moved next to ATmega/ESP32 pins 9/10. Keep within 10mm for oscillator stability.
- **Board outline** — Two outlines caused "malformed" warning. Fix: use single rectangle for board edge.
- **Symbol pin types** — Custom RDA5807M symbol had wrong pin types: SDA/SCL needed "Bidirectional", VCC/GND needed "Power I/O", LOUT/ROUT needed "Output". Wrong types caused ERC warnings.
- **PAM8403 symbol** — Custom PAM8403 symbol had pins typed as "Bidirectional" instead of "Power I/O". This is a functional warning only, doesn't prevent operation.

### What Worked vs What Didn't (continued)

**Worked:**
- Methodical stage-based debugging (amplifier isolation → I2C → RF)  
- Providing both 0x10 and 0x11 I2C code variants (covers module variation)
- Zobel network preventing perfboard oscillation
- Dummy read to clear STC status latch (prevents seek lock)
- Adding 10µF bulk cap on 3.3V (C7) for RF stability  
- Lowering software volume (12 instead of 15) to reduce amplifier load
- Using power bank instead of laptop USB (less noise)
- 4.7k I2C pull-ups (stronger, cleaner edges on long wires)
- Socketed TDA2822M (survived multiple insertions during test)
- Encoder-based tuning with ISR (responsive, smooth)
- Station database with name lookup (better UX than raw frequency)
- Python Colab simulation (allowed UI iteration without reflashing ESP32)

**Didn't Work / Problems:**
- Expecting solder ball on crystal to be harmless (it prevented PLL lock entirely)  
- Replacing RDA5807M with TEA5767 without checking protocol compatibility
- Using 5V rail analysis in isolation without considering amp current draw
- Interpreting speaker DC resistance as a short (3.5Ω = normal, not a fault)
- Using continuity mode for low-resistance measurements (misleading below ~30-50Ω)
- Trying to drive 4Ω speaker at max volume from USB 5V (current limit triggers)
- Truncating long station names (switched to smooth scrolling instead)
- Using standard system fonts (switched to logisoso22 + profont for premium look)

### AI Model Interactions

**ChatGPT:**
- Used for ALL Phase 1 build document generation (Rev 1.0 through Rev 3.2)
- Provided exhaustive debugging sessions: I2C, amplifier, RF front-end
- Correctly diagnosed crystal solder ball issue, 5V rail collapse, I2C address mode
- Generated complete Arduino code (0x10 and 0x11 variants) with proper seek logic
- Created professional schematic with net names and reference designators
- Review style: very thorough engineering design reviews (scored documents 9.3-9.8/10)
- Strengths: hardware debugging, systematic elimination process, clear explanations
- Weakness: took many iterations to get the markdown/PDF output right

**ChatGPT (continued for Phase 2/ESP32):**
- Generated TEA5767 diagnostic code with signal level readback
- Created ESP32 firmware with TEA5767 on custom I2C pins (18/19)
- Built SH1106 OLED UI with U8g2 library
- Developed full station database (68 Sri Lankan stations)
- Implemented advanced UX: volume tweening, tactile flash, smart scrolling
- Provided Python Colab simulation for UI testing (multiple iterations)

**Gemini:**
- Used once for research session: "find a matching tutorial/ guide making the exact same fm radio"
- Gemini searched for RDA5807M + TDA2822M tutorials, found CircuitDigest, MakerPro references
- User verdict: "Did he explicitly give a link? ... Do we have the perfect tutorial?" — No single matching tutorial found
- Outcome: ChatGPT had to synthesize a custom guide from multiple sources

**AI Summary:**
- ChatGPT was the primary AI used throughout the project (estimated 95%+ of conversations)
- Gemini was used for one research session (tutorial search)
- No evidence of Claude, Grok, or DeepSeek usage in the second half
- ChatGPT's most valuable contributions: debugging methodology (systematic elimination), providing multiple code variants (0x10/0x11, RDA/TEA), translating UI mockups to working code
- **Notable AI error:** ChatGPT initially told the user RDA5807M and TEA5767 were similar enough that they could try the same code. This was wrong and caused extended confusion. Later corrected.
- **AI teaching role:** ChatGPT taught I2C fundamentals, potentiometer lug identification, capacitor polarity, multimeter usage, and basic soldering practice to a beginner-level student throughout the project.

---

## Extraction from Project Documentation & Technical Files

### Expert Audit Findings (ALL reviewers consolidated)

#### Critical Issues (will prevent radio from working)

1. **TP4056 CHRG/STDBY LED wiring is backwards** — Found by: Sonnet 4.6, Grok Fast 1, Gemini 3.1 Pro, GPT-5.4. The guide shows CHRG (pin 7) → R12 → LED2 → GND and STDBY (pin 6) → R13 → LED3 → GND. But CHRG and STDBY are open-drain outputs that sink to GND when active. They cannot source current. Correct wiring: VBUS (5V) → R12 → LED2 anode → LED2 cathode → CHRG pin 7. Same for STDBY. Without this fix, neither charging LED will ever light up.

2. **CH340C V3 pin (pin 4) missing from every section** — Found by: Sonnet 4.6, Grok Fast 1, Gemini 3.1 Pro. At 3.3V VCC, V3 must be tied to VCC with a 100nF bypass to GND. Without this, the USB transceiver fails to enumerate — no COM port appears. The .tex guide has since been updated to include C18 (100nF) on V3 pin.

3. **SW1 must be ON to upload firmware — nowhere stated** — Found by: Sonnet 4.6, Grok Fast 1, Gemini 3.1 Pro. CH340C VCC is powered from +3V3 → HT7333 → VBAT → behind latching switch SW1. If SW1 is OFF, CH340C has no power. USB-C port looks dead. Fix: add bold warning that power button must be pressed before USB-C upload.

4. **PAM8403 pin mapping in the guide is WRONG vs datasheet** — Found by: Codex Audit. The guide assumes INL=pin 8, INR=pin 7, ~SHDN=pin 5, VCC=pin 12, PVCC=pins 2,15, OUTL+/OUTL-=pins 16,13. But the actual PAM8403 datasheet says: INL=pin 7, VREF=pin 8, INR=pin 10, SHDN=pin 12, MUTE=pin 5, PVDD=pins 4,13, +OUT_L/-OUT_L=pins 1,3, +OUT_R/-OUT_R=pins 16,14. This is a direct pin-number mismatch. If the schematic was drawn from the guide, the amplifier is wired to wrong pins. Note: The .tex guide re-draws this with correct pin numbers in the PAM8403 infobox section (Section 5.7), but the pin mapping in the main text (Section 5.3 and early document sections) still uses the wrong mapping. This is a contradiction within the same document.

5. **TP4056 BAT pin referenced as pin 3 — WRONG** — Found by: Codex Audit. The guide says TP4056 BAT = pin 3, but pin 3 is actually GND, pin 4 is VCC, pin 5 is BAT. This would make the charger section electrically wrong if copied directly into a schematic.

6. **DW01A + FS8205A protection circuit described incorrectly** — Found by: Codex Audit, GPT-5.4, Gemini 3.1 Pro. The guide shows R14 (100Ω) in the main battery return path as a "current sense resistor". The DW01A typical application uses R1=100Ω feeding VCC, and a separate R2=1kΩ for VM. Placing 100Ω in the main battery return path creates a huge voltage drop and breaks normal operation. Also, the block diagram shows DW01A/FS8205A on the BAT+ positive rail (high-side), but N-channel FETs require low-side operation (BAT- path).

7. **No real schematic in repo to validate** — Found by: Codex Audit. Both `kicad radio/kicad radio.kicad_sch` and `ESP32_FM_Radio_V2/ESP32_FM_Radio_V2.kicad_sch` are empty (14 lines each, no components). Only the guide exists — no schematic, no netlist, no PCB to inspect. Cannot give fabrication "YES" without real design files.

#### Warnings (will cause problems)

1. **Audio input path contradiction across 4 sections** — Found by: Sonnet 4.6, Grok Fast 1, GPT-5.4. The block diagram shows R3+R4 merged at AF_MONO → single C5 → PAM8403. But the connection checklist rows 37-38 show separate C5 for INL and INR. The BOM says C5 qty=2. The block diagram is mono; the checklist is stereo. A builder reading different sections gets different circuits.

2. **Battery divider source impedance too high (50kΩ)** — Found by: Sonnet 4.6, Grok Fast 1, Gemini 3.1 Pro. R7=R8=100kΩ → Thevenin source = 50kΩ. ATmega328P datasheet requires ≤10kΩ for ADC accuracy. Fix: use 10kΩ each (source=5kΩ, draws 210µA at 4.2V) or add 100nF cap at A0 midpoint. The .tex guide has since added C19 (100nF) as fix.

3. **Encoder ISR read/clear not atomic** — Found by: Sonnet 4.6, Grok Fast 1. `int8_t rD = encRDelta; encRDelta = 0;` is two operations. Interrupt between them loses a pulse. Fix: wrap in `noInterrupts()/interrupts()`.

4. **PAM8403 second PVCC pin not bypassed** — Found by: Sonnet 4.6, Grok Fast 1. Two PVCC pins (pins 2 and 15), only C11 (10µF) on pin 2. The other channel has no bypass → audible distortion. Fix: add C20 (10µF) on pin 15. The .tex guide has since been updated with C20.

5. **Footprint conflict: SOIC-16 3.9mm vs SOP-16 4.4mm** — Found by: Sonnet 4.6, Grok Fast 1, GPT-5.4. The mid-document footprint table lists SOIC-16_3.9x9.9mm for PAM8403 and CH340C. The appendix lists SOP-16_4.4x10.4mm. Actual parts have 4.4mm body. Using 3.9mm footprint causes chip body overhang.

6. **"Use while charging" claim too optimistic** — Found by: GPT-5.4. The design is USB-C → TP4056 → battery → system. TP4056 is a charger, not a power-path controller. Charge termination becomes unreliable under load, CHG/STDBY indication can be misleading, charge time increases. Fix: either add load-sharing power path or change guide to "charge with radio off".

7. **DW01A/FS8205A schematic not spelled out clearly** — Found by: Sonnet 4.6, Grok Fast 1, GPT-5.4, Codex Audit, Gemini 3.1 Pro. 5/5 reviewers flagged this. "R14 is listed as 100 ohm x2 DW01A current sense resistors" — does not match standard single-cell protection. Missing dedicated wiring diagram for the most pin-confusing sub-circuit.

8. **Phase 3 assumes RDA5807M at 0x10 only** — Found by: GPT-5.4. Phase 3 assumes address 0x10. Some RRD-102 modules only respond at 0x11 (index mode). This was documented in Phases 1 and 2 but dropped in Phase 3.

9. **RDS station names claimed but not implemented** — Found by: GPT-5.4. Guide promises "station name display (built-in frequency database)" but firmware uses a hardcoded frequency-to-name table, not real RDS decode. Spec mismatch.

10. **OLED changed from Phase 2 (SH1106 1.3") to Phase 3 (SSD1306 0.96")** — Found by: GPT-5.4. If intentional, fine. If not, causes PCB+enclosure mismatch with different pin orders and controller drivers.

#### Suggestions (improvements)

1. **PJ-320A detect variant unspecified** — Sonnet 4.6, Grok Fast 1, GPT-5.4, Gemini 3.1 Pro. Must specify NO-type (normally-open detect). NC variant inverts mute logic. Actually, Gemini points out PJ-320A uses NC switches internally — depends on exact vendor wiring.

2. **R9=10kΩ gives 100mA charge (8-10 hours)** — Sonnet 4.6, Grok Fast 1. Recommend R9=2.2kΩ → ~455mA → 2-3 hour charge. Already reflected in updated .tex BOM.

3. **AMP_EN goes HIGH while RDA5807M still settling — pop on boot** — Sonnet 4.6, Grok Fast 1. Add 500ms delay before enabling amplifier, or use STC flag check.

4. **UNO 3.3V pin (50mA) cannot power full 3V3 bus during ISP** — Sonnet 4.6, Grok Fast 1. Power PCB from LiPo battery during ISP, not from UNO 3.3V pin.

5. **Add 100nF cap on AREF pin** — Sonnet 4.6, Grok Fast 1. Stabilizes ADC reference. Cost: ~₹1. Already added as C21 in .tex guide.

6. **Replace crystal with internal RC oscillator** — Sonnet 4.6, Grok Fast 1, Gemini 3.1 Pro. Saves ~₹80-120 and 3 PCB pads. Must burn bootloader with lfuse=0xE2.

7. **Add test points on critical nets** — Sonnet 4.6, Grok Fast 1. VBAT, +3V3, TXD/RXD, RESET, SDA/SCL.

8. **Verify OLED pin order** — Sonnet 4.6, Grok Fast 1, Gemini 3.1 Pro. Two common orders: GND-VCC-SCL-SDA vs VCC-GND-SCL-SDA. Wrong = dead OLED in seconds.

9. **Correct power claim: ~0.5W per channel, not 3W** — Sonnet 4.6, Grok Fast 1. PAM8403 rated 3W at 5V/4Ω, but actual at VBAT 4.2V/8Ω = ~0.55W. Set expectations.

10. **Verify encoder CW/CCW direction** — Sonnet 4.6, Grok Fast 1. Current code: DT HIGH = -1 = tune down. EC11 clockwise on CLK falling edge → DT is HIGH → clockwise = lower frequency (may be backwards).

11. **Audio topology must be locked to one consistent design** — GPT-5.4. Recommended: keep headphones stereo, mix LOUT+ROUT to AF_MONO for speaker, feed one PAM8403 channel only.

12. **Lock exact manufacturer part numbers before routing** — GPT-5.4, Codex Audit. USB-C (16-pin vs 6-pin), PJ-320A variant, OLED pin order, RDA pinout — all still ambiguous in guide.

13. **Drop on-board TP4056+DW01A+FS8205A for ready-made module** — GPT-5.4, Codex Audit. Lower risk for first PCB iteration. On-board discrete protection has high error probability.

14. **Headphone detect logic mismatch** — Gemini 3.1 Pro. PJ-320A uses NC switches that break contact when plug inserted. With INPUT_PULLUP on D8, breaking = float HIGH, not LOW. Need 10k pull-down and invert logic.

15. **Add ferrite bead on antenna coupling** — Gemini 3.1 Pro. Add 1µH inductor in series with audio signal before headphone jack to stop FM frequency being shunted to GND by low output impedance of audio amp.

### Phase 1 Design Summary (from Phase 1 doc)

**Components:** Arduino UNO + RDA5807M (RRD-102V2.0 module) + TDA2822M (DIP-8) + 4Ω speaker on 15×18cm dot board.

**Topology:**
- RDA5807M powered from Arduino 3.3V pin (external pull-ups to 3.3V, 4.7k preferred)
- LOUT/ROUT mixed mono via R3/R4 (2× 1kΩ) → C1 (10µF coupling) → B100K volume pot → TDA2822M pin 3 (IN1+)
- Amplifier: single-ended channel 1 (pin 1=OUT1→SPK+, pin 4=GND, speaker- = GND)
- Seek buttons on D2/D3 with internal pull-ups
- Power LED always-on indicator
- Zobel network (2.2Ω + 0.1µF) for output stability
- C7 (10µF) on 3.3V rail for RF stability

**Achieved:** Basic FM reception on dot board. Multi-revision build document (Rev 1.0 through 3.2) with progressive improvements. Solder ball on crystal discovered as reason PLL never locked. 5V rail collapse under load identified (USB current limit vs 4Ω speaker). Working radio with seek buttons and hardware volume knob.

### Phase 2 Design Summary (from Phase 2 doc)

**Components:** ESP32 30-pin + RDA5807M (RRD-102 V2.0) + TDA2822M (DIP-8, BTL mode) + SH1106 1.3" OLED + 2× S141 rotary encoders + 18650 battery + TP4056 module on 15×9cm dot board.

**Topology:**
- ESP32 powers RDA5807M and OLED from its 3.3V output
- TDA2822M in BTL mode (pins 1&3 drive speaker, neither to GND)
- Software volume via RDA5807M register 0-15 (no hardware pot)
- I2C shared bus (GPIO21/22) for RDA + OLED
- Battery voltage divider (100k/100k) on GPIO34 (ADC1, input-only)
- Power plans: Plan A = MT3608 boost to 5V, Plan B = HT7333 LDO 3.3V
- Volume controlled by left encoder, tuning by right encoder, push buttons for seek

**Achievements:** Full OLED UI with frequency, station name, signal bars, battery %, volume level, 68-name Sri Lanka station database. Premium UX features: volume tweening, tactile flash, auto-scroll for long station names. Python Colab UI simulation for iteration without reflashing.

### Phase 3 Design Summary (from Phase 3 .tex guide)

**Board:** Custom 2-layer PCB, ~70×50mm, FR-4, mixed SMD + through-hole.

**Architecture:**
- **MCU:** ATmega328P-PU (DIP-28, 8MHz, 3.3V) — Arduino Pro Mini compatible
- **Radio:** RDA5807M on RRD-102 V2.0 module (1.27mm pitch castellation)
- **Display:** 0.96" SSD1306 OLED 128×64, I2C address 0x3C
- **Amplifier:** PAM8403 Class-D (SOP-16), mono mode (left channel only), ~0.5W into 8Ω
- **Audio output:** 8Ω 0.5-1W speaker + stereo 3.5mm headphone jack with auto-detect
- **Power:** 3.7V LiPo 800-1000mAh → HT7333 LDO to 3.3V → entire system
- **Charging:** TP4056 (SOP-8) on-board, USB-C VBUS, 455mA via R9=2.2kΩ
- **Protection:** DW01A + FS8205A low-side (BAT- path)
- **USB:** CH340C (SOP-16) on-board, single USB-C for both charging and programming
- **Encoders:** 2× rotary encoders (right=tuning/seek UP, left=volume/seek DOWN)

**Key changes from Phase 2:**
- ATmega328P replaces ESP32 (simpler, cheaper)
- PAM8403 replaces TDA2822M (Class-D efficiency, SMD)
- Single USB-C for everything (vs separate power/programming)
- Custom PCB (vs dot board)
- SSD1306 OLED (0.96") replaces SH1106 (1.3")

### Bill of Materials (Phase 3 — from .tex guide)

**ICs and Modules:**
| Ref | Part | Qty |
|-----|------|-----|
| U1 | ATmega328P-PU (DIP-28) | 1 |
| U2 | RDA5807M RRD-102 V2.0 module | 1 |
| U3 | PAM8403 (SOP-16) | 1 |
| U4 | TP4056 (SOP-8) | 1 |
| U5 | HT7333 (SOT-89) | 1 |
| U6 | CH340C (SOP-16) | 1 |
| U7 | DW01A (SOT-23) + FS8205A (SOT-23-6) | 1+1 |
| --- | SSD1306 0.96" OLED module | 1 |

**Resistors:**
| Ref | Value | Purpose |
|-----|-------|---------|
| R1,R2 | 4.7kΩ | I2C pull-ups to +3V3 |
| R3,R4 | 10kΩ | LOUT/ROUT mono mix (increased from 1k in Phases 1-2) |
| R5 | 10kΩ | ATmega RESET pull-up |
| R6 | 330Ω | Power LED (from +3V3) |
| R7,R8 | 100kΩ | Battery divider (with C19 100nF fix) |
| R9 | 2.2kΩ | TP4056 PROG → 455mA charge |
| R10,R11 | 5.1kΩ | USB-C CC pull-downs |
| R12,R13 | 1kΩ | CHRG/STDBY LED limiters |
| R14 | 100Ω | DW01A current sense |

**Capacitors:**
| Ref | Value | Purpose |
|-----|-------|---------|
| C1,C2 | 100nF | ATmega VCC/AVCC decoupling |
| C3a,b | 22pF | Crystal load caps |
| C4 | 100nF | RDA5807M decoupling |
| C5 | 1µF | PAM8403 input coupling |
| C6,C7 | 10µF | Headphone L/R coupling |
| C8 | 100pF | Antenna coupling |
| C9,C10 | 10µF | HT7333 input/output |
| C11 | 10µF | PAM8403 PVCC bypass (pin 2) |
| C12 | 10µF | TP4056 input bypass |
| C13 | 100nF | CH340C VCC decoupling |
| C14 | 100nF | Auto-reset (DTR# → RESET) |
| C15 | 1µF | TP4056 output bypass |
| C16 | 10µF | RDA5807M VCC bulk |
| C17 | 1µF | TP4056 BAT pin decoupling |
| C18 | 100nF | CH340C V3 pin bypass |
| C19 | 100nF | ADC filter (A0 to GND) |
| C20 | 10µF | PAM8403 PVCC pin 15 bypass |
| C21 | 100nF | AREF to GND |

**Connectors:**
| Ref | Part |
|-----|------|
| J1 | 3.5mm TRS jack PJ-320A NO-type |
| J2 | JST PH 2-pin battery connector |
| J3 | 2×3 pin ISP header |
| J4 | USB-C receptacle (16-pin SMD) |
| J5 | 1×4 pin header (OLED) |
| J6 | 2× 1×5 pin headers (encoders) |
| J7 | 2×5 pin 1.27mm header (radio module) |
| J8 | Speaker connector |
| SW1 | Latching push button |
| Y1 | 8MHz HC49S crystal |

### KiCad Project Details

**kicad radio/kicad radio.kicad_sch:** Empty (14 lines, no lib_symbols, no components). Schematic never drawn.

**kicad radio/kicad radio.kicad_pro:** Basic project file with DRC rules. Default trace 0.2mm, min clearance 0.2mm, copper-to-edge 0.5mm, min via drill 0.3mm. Empty design — no board outline or components.

**ESP32_FM_Radio_V2/ESP32_FM_Radio_V2.kicad_sch:** Empty (14 lines). No symbols placed.

**ESP32_FM_Radio_V2/ESP32_FM_Radio_V2.kicad_pro:** Identical config to kicad radio project.

**radio2_drc.json** (from "radio 2" PCB, 2026-03-21, KiCad 9.0.6):
- 15 unconnected_items errors (GND zone not connected to oled1, R10, C6, C18, U4 pads)
- 3 copper_edge_clearance violations (J2_bat1 tracks at 0.358mm from edge vs 0.5mm min)
- 6 starved_thermal violations (GND pads with only 1 thermal spoke vs 2 minimum)
- 30+ lib_footprint_issues warnings (Footprint libraries not registered: Rotary_Encoder, Resistor_SMD, LED_THT, Capacitor_SMD, Connector_Audio, Package_SO, HT7333, Connector_JST, Connector_PinSocket_2.54mm, Connector_PinHeader_2.54mm, Connector_USB, Package_DIP, PAM8403, TP4056, FS8205A, Crystal, RDA5807-eagle, etc.)

**KiCad details from .tex guide:**
- KiCad 9.0 used
- Custom footprints needed: RDA5807M module (11×11mm, 2×5 pads, 1.27mm pitch)
- Board outline: 70×50mm rectangle, 2mm fillet corners, M3 mounting holes
- Design rules: min trace 0.25mm, power 0.5-1.0mm, clearance 0.2mm, via drill 0.3mm
- B.Cu continuous GND plane, zone separation: analog (west), digital (east), power (south-west)

### TEA5767 Notes

The tea5767.txt file is a QuartzComponents tutorial page showing TEA5767 module interfacing with Arduino Nano + LM386 amp + 16×2 LCD. Key specs: I2C address 0x60, 5-byte PLL command, BUS/MODE pin must be grounded for I2C, W/R pin grounded for write, MPXO audio output, L/R out.

The human found this tutorial after accidentally buying TEA5767 (RRD-102BC, Rs.550) instead of RDA5807M (RRD-102V2.0, Rs.230). Same module form factor, completely different protocol. Attempting to use RDA5807M code with TEA5767 hardware caused extended confusion. The ESP32 Phase 2 was eventually adapted for TEA5767 with raw I2C writes (5-byte PLL command). Final verdict: RDA5807M is the correct chip. TEA5767 was a wrong-part detour.

### Validation Checklist Status

From FM_Pocket_Radio_Phase3_Final_Validation_Checklist.md — 12 domains:

- **Files required:** ❌ Real schematics empty, no netlist, no PCB, no exact part numbers
- **Power and Battery:** ❌ DW01A+FS8205A protection not finalized
- **TP4056 Charger:** ❌ LED polarity wrong; BAT pin number wrong originally
- **DW01A+FS8205A:** ❌ No datasheet-derived schematic; resistor values unclear
- **CH340C:** ⚠️ PARTIAL — V3 pin added; SW1 warning still missing
- **ATmega core:** ❌ Crystal vs internal RC not finalized
- **RDA5807M module:** ⚠️ Module pin order not confirmed from physical module
- **Audio Path:** ❌ PAM8403 pin mapping conflict; mono/stereo contradiction
- **Headphone Detect:** ❌ Jack variant not locked; firmware logic may be inverted
- **PCB Layout Review:** ⚠️ No real PCB to review
- **CAD/Manufacturing:** ⚠️ No ERC/DRC results on final design
- **Bring-Up:** ⚠️ No physical board to test

**Final verdict: "Not yet" for fabrication. Real schematic required before YES.**

### Final Consolidated Timeline

- **Feb 17, 2026** — Project starts. KiCad learning, custom RDA5807M symbol, capacitor errors.
- **Feb 17** — Components bought (Duino Electronics). Wrong 32MHz crystal.
- **Feb 17** — I2C 5V/3.3V mismatch discovered. Level shifter needed.
- **Feb 17** — First ChatGPT KiCad consultation.
- **Feb 18-19** — ATmega vs ESP32, TEA5767 vs RDA5807M debates.
- **Feb 21** — Phase 2 ESP32 doc generated. OLED UI simulator built.
- **Feb 21** — Phase 1 "No Regrets" build guide. Revs 1.0 to 3.2.
- **Late Feb** — Phase 1 dot board built. No sound on first power.
- **Late Feb** — I2C 0x10 vs 0x11 resolved (index mode needed).
- **Late Feb** — Crystal solder ball found (PLL not locking).
- **Late Feb** — RDA5807M accidentally replaced with TEA5767. Wrong chip debugging.
- **Late Feb** — TDA2822M overheating + 5V rail collapse diagnosed.
- **Late Feb** — Phase 1 Rev 3.2 locked. New RDA5807M ordered.
- **Late Feb/Mar** — ESP32 Phase 2: full OLED UI, station database (68 Sri Lanka), encoder tuning.
- **Late Feb/Mar** — Python Colab UI simulation built.
- **Mar 18** — User starts ESP32 learning. Arduino IDE 2.3.7, blink test.
- **Mar 18-20** — KiCad PCB phase for Phase 3. Teammate's unfinished project received.
- **Mar 20** — Library path hell, footprint filename errors fixed.
- **Mar 20** — ERC clean (0 errors). PCB autorouter → 137 DRC violations (cleaned).
- **Mar 20** — Phase 3 power architecture finalized (TP4056+DW01A+FS8205A+HT7333+PAM8403+CH340C+USB-C).
- **Mar 21** — radio2_drc.json: 24 errors on "radio 2" PCB revision.
- **Late Mar** — 5 AI models review Phase 3 design. Multiple critical issues found.
- **Mar 2026** — Phase 3 .tex guide final with some fixes applied. PCB never fabricated.

### Final Problem List (deduplicated)

1. 5V/3.3V level mismatch on I2C (Arduino UNO → RDA5807M)
2. I2C address 0x10 vs 0x11 — ~40% modules need index mode
3. Solder ball on 32.768kHz crystal — prevented PLL lock
4. TDA2822M overheating with 4Ω speaker at max volume
5. Wrong IC purchased: TEA5767 instead of RDA5807M
6. TEA5767 RF front-end dead (possibly damaged)
7. Autorouter shorts (137 DRC violations on first PCB attempt)
8. KiCad library path hell (teammate's old PC paths)
9. Custom RDA symbol had no pin numbers (names only)
10. UNO 3.3V pin cannot supply enough current for ISP
11. TP4056 CHRG/STDBY LEDs wired backwards
12. CH340C V3 pin floating — USB won't enumerate
13. DW01A/FS8205A protection on wrong side (high-side) and wrong resistor values
14. PAM8403 pin mapping contradicts datasheet
15. TP4056 BAT pin number wrong (pin 3 instead of pin 5)
16. Audio path contradiction: mono block diagram vs stereo checklist
17. Battery ADC 50kΩ source impedance (too high for ATmega)
18. Encoder ISR read/clear not atomic → lost pulses
19. PAM8403 second PVCC pin not bypassed
20. SOIC-16 vs SOP-16 footprint conflict
21. OLED type changed without clear justification
22. "Use while charging" claim unsupported by TP4056 topology
23. RDS claim not implemented (hardcoded table only)
24. 0x11 address fallback dropped in Phase 3
25. No real schematic or PCB in repository

### Design Rationale Summary

**ATmega328P over ESP32:** Simpler, cheaper (~₹300), DIP-28 through-hole, familiar UNO ecosystem, sufficient for I2C+OLED+encoders. No level shifters at 3.3V.

**PAM8403 over TDA2822M:** Class-D 90% efficiency (vs ~50-60%), ~SHDN pin for mute, no heatsink needed, SMD for PCB. TDA2822M used in Phases 1 (single-ended) and 2 (BTL).

**Mono speaker + stereo headphones:** Single speaker in pocket radio — no stereo benefit. Headphones get full stereo via jack coupling caps.

**R3/R4 changed 1kΩ→10kΩ:** 1kΩ creates ~500Ω load on ~1kΩ output impedance → 6dB loss. 10kΩ reduces loading. Review feedback drove this change.

**USB-C single port:** One cable for charging + programming. CH340C handles USB→UART, TP4056 shares VBUS for charging.

**On-board TP4056 (not module):** Cleaner integration, but increases risk — DW01A/FS8205A must be correct.

**HT7333 over AMS1117:** Dropout <100mV vs 1V. At 3.0V battery, HT7333 outputs ~2.9V vs AMS1117 would cut out entirely.

**2-layer PCB (not 4-layer):** Budget constraint (~₹800-1500 for 5 pcs). B.Cu ground plane provides adequate shielding with careful zone separation.

**RDA5807M over TEA5767:** DSP demodulation, soft mute, auto seek, stereo, 50µs de-emphasis. TEA5767 only a temporary detour from wrong purchase.

**R9=2.2kΩ (455mA charge):** Compromise: 100mA too slow (8-10hr), 1A may trip USB limit. 455mA = ~2hr charge for 800mAh.

**Crystal over internal RC:** Better UART accuracy at ≥57600 baud. Internal RC ±2% may cause framing errors. Cost ₹80-120.

**Software volume over hardware pot:** Fewer components, 16 steps sufficient. Saved pot + knob + wiring.

**8Ω speaker over 4Ω:** ~0.5W adequate for pocket radio. 4Ω draws more current, shorter battery life. Phase 1 showed 4Ω causes overheating with TDA2822M.

**DW01A+FS8205A low-side (BAT-):** N-channel FETs need gate voltage above source → can only work on low side. Initial block diagram incorrectly showed high-side. Corrected in later .tex revisions.
