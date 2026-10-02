# Portable Single-Lead ECG Arduino Shield (AD8232)

A custom 2-layer PCB shield for the Arduino Uno R4 Minima, built around the Analog Devices AD8232 ECG analog front-end. Targets full ECG waveform fidelity (not just heart-rate blink detection), using the datasheet's "Cardiac Monitor Configuration" (Figure 66) topology.

## Status

🚧 **In progress — Fabrication/Assembly Phase.**
## Overview

| | |
|---|---|
| **Target MCU** | Arduino Uno R4 Minima (R3 shield form factor/pinout) |
| **Analog front end** | AD8232ACPZ-WP, 20-pin LFCSP (4×4mm) |
| **Topology** | 0.5 Hz two-pole high-pass filter → 40 Hz two-pole low-pass filter, gain ≈ 1100 |
| **Electrode interface** | 3-lead (RA/LA/RL) via Kycon STX-3000 3.5mm TRS jack |
| **Power** | Direct off Arduino 3.3V pin, no onboard LDO (AD8232 draws ~170µA) |
| **Output** | OUT → A0 (analog, ADC-sampled) |
| **Status flags** | LOD+ → D4, LOD− → D5 (D2/D3 kept free for future interrupt use) |
| **Assembly** | PCBA via JLCPCB (LFCSP not hand-solderable) |
| **Board** | 2-layer, bottom ground pour, top signal routing |

## Inspiration 

This was mainly inspired by the Sparkfun AD8232 board. Their board is marketed as a heart rate monitor and I wanted to see if I could build a board using the more complicated waveform fidelity circuit. Additionally Sparkfun's board doesn't use a solid ground plane and uses traces for all the routing. Since ECG signals are naturally really sensitive, I wanted to see if I could challenge myself to complete the routing while maintaining as solid of a ground plane as possible. 

## Design Rationale

### Why Figure 66, not the simpler heart-rate configuration

The AD8232 datasheet offers several reference circuits. This project uses the **Cardiac Monitor Configuration** (two-pole HPF + two-pole LPF, gain 1100) rather than the simpler single-pole heart-rate-detection circuit, because the goal is a clean, diagnostically-shaped waveform rather than just beat timing.

### Bias resistor topology (R1/R2)

R1/R2 (10MΩ) are tied to **V_S**, one of four datasheet-documented options (V_S / REFOUT / RLD / external) confirmed against the AD8232-EVALZ user guide. Tying to REFOUT instead would reject supply-rail noise better (the bias would track the reference rather than the raw rail), which matters more here since this board runs directly off the Arduino's unregulated 3.3V pin with no dedicated LDO. **Known trade-off, not a bug** — flagged for anyone reviewing this design.

### Electrode pin assignment

RA and LA are placed on the jack's easiest-to-route pads; RL takes the mechanically awkward pad. This is intentional: RA/LA feed the differential sensing pair (IN+/IN−) behind the 10MΩ bias resistors — high impedance, microvolt-scale signal, and sensitive to any length/routing asymmetry between the two legs (directly impacts CMRR). RL is the output of the RLD amplifier — actively driven, low impedance, tolerant of a less-clean route.

### High-impedance net handling

A recurring design constraint throughout layout: nets sitting behind the AD8232's 10MΩ-class resistors (IN+/IN−, the R1–R4 bias/protection network, the REFOUT↔HPSENSE feedback loop, HPDRIVE, RLDFB) are aggressively guarded — kept short, mirror-symmetric where paired, routed away from switching nets, and never casually dropped through plane gaps. The underlying mechanism: injected noise couples in as a roughly fixed current; the resulting noise *voltage* equals that current times the node's impedance (Ohm's law), so a stray nanoamp becomes millivolts across a 10MΩ node but is negligible across a low-impedance digital or driven output. Low-swing digital nets (LOD flags, control lines) and DC rails (V_S/GND) are comparatively tolerant and route freely.

The REFOUT (pin 8) ↔ HPSENSE (pin 20) feedback loop is a fixed exception: these pins sit on opposite sides of the LFCSP package by pinout, so the connection is routed around the package's less-congested side rather than through the high-density analog side — while still being treated as a guarded, high-Z net in its own right.

### Ground plane

Bottom layer carries a solid (near-unbroken) ground pour; V_S and digital signals route on top. The plane's job isn't primarily about the handful of components with an explicit GND pin — it provides a low-impedance return path for *every* signal's current loop (minimizing loop area and noise coupling) and shields the board's sensitive analog traces from 60Hz mains hum, the most common failure mode in DIY ECG builds. All Arduino header GND pins are tied into the plane via individual vias rather than a single tie-point, lowering overall return impedance and giving nearby signals (D4/D5/A0) a local return path.

## Known Limitations

- **No galvanic isolation** between the patient electrodes and the USB-connected host (Arduino). For a production medical device this would require an isolator (e.g., ADuM/ISO7xxx series) and isolated power per IEC 60601-1 patient-isolation requirements. Out of scope for this project's goals, but noted explicitly as a design limitation rather than an oversight.
- Bias resistors (R1/R2) tied to V_S rather than REFOUT — acceptable datasheet-documented option, but not the lowest-noise choice given the unregulated supply.

## Repository Structure

```
/Altium         — Schematic, PCB, project 
/Assembly       — Bill of materials, pick and place files
/Manufacturing  — Gerber files, NC Drill files
```

## References

- [AD8232 Datasheet](https://www.analog.com/media/en/technical-documentation/data-sheets/ad8232.pdf) — see Figure 66 (Cardiac Monitor Configuration) and Table 3 (Pin Function Descriptions)
- [AD8232-EVALZ User Guide (UG-514)](https://www.analog.com/media/en/technical-documentation/user-guides/ad8232-evalz_ug-514.pdf) — bias resistor configuration options

