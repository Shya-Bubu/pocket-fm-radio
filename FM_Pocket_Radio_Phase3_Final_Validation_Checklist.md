# FM Pocket Radio Phase 3: Final Validation Checklist

Use this only after the teammate shares the real design files.

## Files Required

- [ ] `.kicad_sch`
- [ ] netlist export
- [ ] `.kicad_pcb`
- [ ] exact part numbers or photos for the module and connectors

## Subsystem Review

### 1. Power and Battery

- [ ] Battery positive path is consistent from cell to switch to VBAT.
- [ ] Battery negative path and protection topology match a real datasheet circuit.
- [ ] No fake "sense resistor" is placed in the main battery current path.
- [ ] LDO input/output pins and capacitors match the regulator datasheet.
- [ ] 3.3 V rail powers only parts that are safe at 3.3 V.
- [ ] VBAT-powered parts are safe across the full battery range.

### 2. TP4056 Charger

- [ ] Symbol pinout matches the datasheet.
- [ ] `BAT` is on the correct pin.
- [ ] `PROG`, `CHRG`, `STDBY`, `CE`, `TEMP`, `VCC`, and `GND` are all checked.
- [ ] Status LEDs are wired as sink outputs, not as source outputs.
- [ ] Charge current is appropriate for the battery capacity and USB source.
- [ ] Team agrees whether charge-while-use is accepted or whether radio-off charging is required.

### 3. DW01A + FS8205A Protection

- [ ] Protection block matches one exact datasheet schematic.
- [ ] `VCC`, `VM`, `OD`, `OC`, and `GND` are all checked against the datasheet.
- [ ] FS8205A source/drain orientation is verified.
- [ ] Resistor values and locations match the reference circuit.

### 4. CH340C and Programming

- [ ] Symbol pinout matches the datasheet.
- [ ] `V3` is handled correctly for 3.3 V operation.
- [ ] `TXD` goes to MCU `RX`, and `RXD` goes to MCU `TX`.
- [ ] `DTR#` auto-reset path is correct.
- [ ] USB `D+` and `D-` go only to the CH340C.
- [ ] USB-C CC resistors and grounding are present.
- [ ] Team understands that the switched-power design may require the radio to be ON for USB enumeration.

### 5. ATmega328P Core

- [ ] `VCC`, `AVCC`, `GND`, and `AREF` are correct.
- [ ] Clock choice is final: crystal or internal RC.
- [ ] Reset pull-up and ISP header wiring are correct.
- [ ] Bootloader/programmer workflow matches the actual hardware.

### 6. RDA5807M Module

- [ ] Exact module pin order is confirmed from the real module in hand.
- [ ] `VCC`, `GND`, `SDA`, `SCL`, `ANT`, `LOUT`, and `ROUT` are mapped to the real module pads.
- [ ] I2C pull-ups are present and tied to 3.3 V.
- [ ] Antenna path is consistent with the chosen headphone-jack approach.

### 7. Audio Path

- [ ] PAM8403 symbol pinout matches the datasheet.
- [ ] Mono-speaker topology is intentionally chosen and wired consistently.
- [ ] Headphone path remains stereo if that is the intent.
- [ ] Speaker is connected only across a valid BTL output pair.
- [ ] No speaker lead is tied to ground.
- [ ] Mute/shutdown control uses the correct control pin.

### 8. Headphone Detect

- [ ] Exact jack variant is identified, not just the family name.
- [ ] Detect-switch behavior is measured with a multimeter.
- [ ] Firmware logic matches the actual switch behavior.
- [ ] Antenna-coupling connection to the jack is intentional and reviewed.

### 9. OLED and Other Modules

- [ ] Exact OLED module pin order is confirmed.
- [ ] Connector silkscreen matches the real module orientation.
- [ ] No module assumes a generic pin order without verification.

## PCB Layout Review

### 10. Placement

- [ ] USB, jack, switch, and headers are aligned to the intended enclosure edges.
- [ ] No footprint is based on a guessed connector body.
- [ ] Radio section is kept away from the noisiest digital and class-D switching areas.

### 11. Routing

- [ ] Power traces are wide enough for current.
- [ ] USB `D+` and `D-` are short and routed as a pair.
- [ ] Audio traces are short and kept out of noisy areas.
- [ ] Battery and amplifier current loops are not daisy-chained through sensitive nodes.

### 12. Grounding and RF

- [ ] Ground plane exists and is continuous where intended.
- [ ] Class-D return currents have short local returns.
- [ ] Antenna area is kept clear enough to avoid obvious self-jamming.
- [ ] No unnecessary copper is placed directly under the sensitive antenna path.

## CAD and Manufacturing Checks

- [ ] ERC is clean or all remaining warnings are intentionally waived.
- [ ] DRC is clean or all remaining violations are intentionally waived.
- [ ] Footprints match the exact physical parts.
- [ ] 3D viewer shows reasonable placement and orientation.
- [ ] STEP export from KiCad works if Fusion is needed.
- [ ] Fusion import is checked after conversion, not blindly trusted.

## Bring-Up Checklist

- [ ] Battery voltage appears where expected.
- [ ] 3.3 V rail is correct before inserting sensitive modules if applicable.
- [ ] Charger status behaves as expected.
- [ ] USB device enumerates.
- [ ] Bootloader upload works.
- [ ] I2C scan sees the expected devices.
- [ ] Radio tunes and produces audio.
- [ ] Headphone insertion changes behavior correctly.
- [ ] Speaker mute logic works.
- [ ] Battery readout is plausible.

## Final Yes Rule

Only give the final "YES, this should work if assembled correctly" after all of the following are true:

- [ ] real schematic reviewed
- [ ] real netlist reviewed
- [ ] real PCB reviewed
- [ ] exact parts confirmed
- [ ] ERC/DRC checked
- [ ] hard blockers resolved

If any of those boxes are open, the answer is still "not yet."
