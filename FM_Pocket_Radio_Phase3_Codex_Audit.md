# FM Pocket Radio Phase 3: Codex Audit and Submission Workflow

## Scope

This file implements stage 1 of the two-stage workflow:

1. Review the guide and repo for fabrication-blocking issues.
2. Define the fastest safe KiCad-to-Fusion workflow for the March 19, 2026 submission.
3. Defer the final fabrication "YES" until the real schematic/netlist/layout exists.

## Repo Status

- `FM_Pocket_Radio_Phase3_Guide.tex` is detailed, but it is still only a document.
- `ESP32_FM_Radio_V2/ESP32_FM_Radio_V2.kicad_sch` is effectively empty.
- `ESP32_FM_Radio_V2/ESP32_FM_Radio_V2.kicad_pcb` is effectively empty.
- `kicad radio/kicad radio.kicad_sch` is effectively empty.
- `kicad radio/kicad radio.kicad_pcb` is effectively empty.

Bottom line: there is no real design in the repo yet to validate pin-by-pin. The guide can be audited, but it cannot earn a final fabrication sign-off by itself.

## Hard Blockers in the Guide

These are the issues that can cause a dead board, a wrong schematic, or a likely respin. They should be treated as must-fix items before anyone trusts the document for fabrication.

### 1. PAM8403 pin mapping in the guide is wrong

Guide locations:

- `FM_Pocket_Radio_Phase3_Guide.tex:891`
- `FM_Pocket_Radio_Phase3_Guide.tex:913`
- `FM_Pocket_Radio_Phase3_Guide.tex:970`
- `FM_Pocket_Radio_Phase3_Guide.tex:1333`

What the guide currently assumes:

- `INL = pin 8`
- `INR = pin 7`
- `~SHDN = pin 5`
- `VCC = pin 12`
- `PVCC = pins 2, 15`
- `OUTL+/OUTL- = pins 16, 13`

What the PAM8403 datasheet says:

- `INL = pin 7`
- `VREF = pin 8`
- `INR = pin 10`
- `SHDN = pin 12`
- `MUTE = pin 5`
- `PVDD = pins 4, 13`
- `+OUT_L / -OUT_L = pins 1, 3`
- `+OUT_R / -OUT_R = pins 16, 14`

Why this is a blocker:

- If the schematic is drawn from the guide as written, the amp will be wired to the wrong pins.
- The board may end up driving outputs, grounds, and control pins incorrectly.
- This is not a tuning problem. It is a likely non-working or damaged-audio-stage problem.

Required action:

- Rebuild the PAM8403 symbol and pin map directly from the datasheet before schematic capture continues.
- Only after the pin map is fixed should the mono-speaker topology be re-entered.

Source:

- PAM8403 datasheet: https://www.diodes.com/datasheet/download/PAM8403.pdf

### 2. TP4056 BAT pin is referenced with the wrong pin number

Guide locations:

- `FM_Pocket_Radio_Phase3_Guide.tex:380`
- `FM_Pocket_Radio_Phase3_Guide.tex:772`
- `FM_Pocket_Radio_Phase3_Guide.tex:774`
- `FM_Pocket_Radio_Phase3_Guide.tex:1357`

What the guide currently says:

- `TP4056 BAT = pin 3`

What the TP4056 pinout says:

- `pin 3 = GND`
- `pin 4 = VCC`
- `pin 5 = BAT`

Why this is a blocker:

- A charger wired from the wrong pin numbering will not match the actual IC.
- If copied directly into a schematic or footprint symbol, the charger section will be electrically wrong.

Required action:

- Fix the TP4056 symbol and all checklist references so `BAT` is on `pin 5`.
- Re-check `CHRG`, `STDBY`, `CE`, `TEMP`, and `PROG` against the same symbol while correcting it.

Source:

- TP4056 pin summary: https://docs.cirkitdesigner.com/component/b6442fdc-0e5f-4e56-a7bb-c604eee5ae95/tp4056
- TP4056 datasheet reference used in planning: https://www.digikey.com/en/htmldatasheets/production/2049110/0/0/1/tp4056.html

### 3. DW01A + FS8205A protection circuit is described incorrectly

Guide locations:

- `FM_Pocket_Radio_Phase3_Guide.tex:392`
- `FM_Pocket_Radio_Phase3_Guide.tex:427`
- `FM_Pocket_Radio_Phase3_Guide.tex:458`
- `FM_Pocket_Radio_Phase3_Guide.tex:577`
- `FM_Pocket_Radio_Phase3_Guide.tex:1361`

What the guide currently says:

- `R14 = 100 ohm` sits between the FS8205A source node and circuit ground as a "current sense resistor".

What the DW01A typical application shows:

- `VCC` is fed through `R1 = 100 ohm`.
- `VM` uses a separate `R2 = 1 kohm`.
- The standard circuit does not place a `100 ohm` resistor in the main battery return path the way the guide describes.

Why this is a blocker:

- Putting `100 ohm` in the battery return path would create a huge voltage drop and break normal operation.
- This section is safety-critical and should not be improvised from memory.

Required action:

- Replace the guide's battery-protection drawing with one exact datasheet-derived schematic.
- If your team is not fully comfortable with this stage, use a protected LiPo or a proven charger/protection module instead of rolling this block onto the main PCB.

Source:

- DW01A datasheet: https://datasheet.lcsc.com/lcsc/1901091236_PUOLOP-DW01A_C351410.pdf

### 4. The guide cannot receive a final fabrication "YES" yet

Guide/repo locations:

- `FM_Pocket_Radio_Phase3_Guide.tex`
- `ESP32_FM_Radio_V2/ESP32_FM_Radio_V2.kicad_sch`
- `kicad radio/kicad radio.kicad_sch`

Why this is a blocker:

- A document can still contain pin-map, footprint, or connector mistakes.
- The repo currently has no finished schematic or PCB to inspect.
- Without the actual netlist or KiCad schematic, there is no way to certify every connection.

Required action:

- Treat the current output as a guide audit only.
- Only move to final validation after the teammate shares the actual `.kicad_sch`, netlist, or both.

## Part Freeze Items Before Footprinting

These are not optional if the team wants to avoid footprint mistakes:

- Exact RDA5807M module variant and physical pin order.
- Exact 3.5 mm jack part number and detect-switch behavior.
- Exact USB-C receptacle part number.
- Exact charger/protection implementation:
  - on-board TP4056 + discrete protection, or
  - protected battery, or
  - ready-made charger/protection module.
- Exact OLED module and pin order.

Until those are fixed, any footprinting work should be considered provisional.

## Submission Workflow for Today

The lowest-risk workflow is:

1. Draw the schematic in KiCad from corrected symbols only.
2. Run ERC and fix all critical issues.
3. Assign exact footprints to the exact parts in hand.
4. Update PCB from schematic.
5. Place parts.
6. Route only after the schematic is trustworthy.
7. Add the ground zone and re-run DRC.
8. Use KiCad 3D Viewer for screenshots if time is tight.
9. If you still want Fusion, export or import only after the KiCad PCB is stable.

### KiCad First, Fusion After

Recommended:

- KiCad is the electrical source of truth.
- Fusion is for 3D, enclosure, rendering, and mechanical fit.

Why:

- KiCad is built for the schematic, nets, ERC, PCB, and DRC work you need right now.
- Fusion import is useful, but it is still a translation step that must be checked afterward.

### Fastest Same-Day Submission Path

If the goal is only to submit a PCB-looking deliverable today:

1. Finish enough corrected schematic in KiCad to produce a board.
2. Push footprints to PCB.
3. Arrange the components cleanly.
4. Route the obvious power and signal nets you have time for.
5. Pour GND.
6. Open KiCad 3D Viewer and capture screenshots.

If Fusion becomes a distraction, stop there and submit the KiCad visuals.

## KiCad to Fusion Options

### Option A: Export STEP from KiCad

Use this if you only need 3D/mechanical work in Fusion.

Recommended settings:

- Export components: enabled
- Export tracks/vias: enabled if you want copper geometry visible
- Export pads/zones: enabled if helpful
- `Substitute similarly named models`: enabled

Why this option is good:

- Fastest path to a 3D assembly in Fusion.
- Good for enclosure fit and presentation work.

Source:

- KiCad PCB Editor docs: https://docs.kicad.org/9.0/en/pcbnew/pcbnew.html

### Option B: Import KiCad Project into Fusion Electronics

Use this if you want Fusion to ingest the KiCad project files directly.

Expected inputs:

- `.kicad_pro`
- `.kicad_sch`
- `.kicad_pcb`

Important caution:

- Fusion can import KiCad v7+ files, but you still need to verify the converted output because tool differences can change details.

Source:

- Fusion KiCad import docs: https://help.autodesk.com/cloudhelp/ENU/Fusion-ECAD/files/ECD-KICAD-IMPORTER.htm
- Fusion KiCad import steps: https://help.autodesk.com/cloudhelp/ENU/Fusion-ECAD/files/ECD-KICAD-IMPORTER-DO.htm
- Fusion STEP import: https://www.autodesk.com/products/fusion-360/blog/import-step-file-into-fusion-360/

## Videos Worth Using

These are the most useful references for the exact workflow you need:

- KiCad schematic cleanup and completion:
  - https://www.youtube.com/watch?v=KN3fHXhNkTU
- KiCad footprint assignment and custom footprints:
  - https://www.youtube.com/watch?v=4YSZwcUSgJo
- KiCad PCB placement, routing, pours, DRC, and 3D viewer:
  - https://www.youtube.com/watch?v=zdieFtrvjew

## What I Need Before a Final "YES"

I can only give the final fabrication go/no-go after I receive the real design files.

Minimum useful inputs:

- The actual `.kicad_sch`
- The actual netlist export
- The actual `.kicad_pcb` once routing starts
- Exact part numbers or clear photos for:
  - RDA5807M module
  - headphone jack
  - USB-C connector
  - OLED
  - battery/charger/protection parts

Once those exist, the next review should focus on:

- Power path
- Charger path
- Battery protection
- USB/UART/reset
- I2C and module pin order
- Audio path and headphone detect
- Grounding and layout risk

## Current Verdict

- `Submission-ready visual workflow`: yes, through KiCad first.
- `Fabrication-ready electrical sign-off`: no, not yet.
- `Final yes/no without the real schematic or netlist`: not possible.
