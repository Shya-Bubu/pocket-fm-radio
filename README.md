# FM Pocket Radio

This repository contains the complete design and documentation for an FM Pocket Radio project, developed as part of a university module (EE2050). The project demonstrates an iterative engineering approach, moving from a perfboard prototype to a fully featured ESP32 + PAM8403 system, and finally to a custom PCB design.

## Project Structure
- **report_radio/**: Contains the LaTeX source code and the final compiled main.pdf report.
- **ESP32_FM_Radio_V2/** & **kicad radio/**: KiCad 9.0 project files for the final Phase 3 PCB layout and schematic.

## Highlights
- **Architecture**: ESP32 microcontroller, RDA5807M FM Radio IC, PAM8403 Audio Amplifier, OLED Display, and Rotary Encoders.
- **Design Verification**: The project was heavily AI-assisted for code generation, debugging, and PCB design verification.
- **Documentation**: See eport_radio/main.pdf for a deep dive into the technical architecture, challenges, and iterative prototyping process.

## Prototypes & PCB Design

### Iteration 2 (Dot Board Prototype)
Fully working prototype with premium UX elements (OLED, encoders).
![Working Prototype](report_radio/images/v2_working_1.png)

### Iteration 3 (KiCad 2D/3D Design)
The target architecture designed as a custom 2-layer 70x50mm PCB.
![2D PCB](report_radio/images/radio2_top.png)
![3D PCB](report_radio/images/RADIO.png)

---
*Authored by Shya-Bubu*
