# Throttle and Drive Safety System

**Formula SAE Electric Vehicle — Anargya ITS EV Team**

Firmware and custom PCB implementing the accelerator pedal plausibility check and Ready-To-Drive logic for an FSAE electric race car. This repository documents the design reasoning and the two field-diagnosed failures that drove the redesign.

---

## 1. What This System Does

The accelerator pedal carries **two independent position sensors (APPS)**. Under normal operation they move together. If one sensor drifts, disconnects, or is damaged, the two readings diverge — and a divergent pedal signal on a high-torque electric drivetrain is a safety hazard.

This board continuously compares the two sensor signals. When the deviation exceeds the allowed threshold, it raises a fault and cuts torque delivery. It also handles the **Ready-To-Drive** sequence, which prevents the vehicle from becoming drivable without a deliberate driver action.

Redundant pedal sensing with an implausibility check is mandated by the FSAE ruleset precisely because a single-sensor failure on an electric powertrain can produce unintended acceleration.

---

## 2. Problem 1 — Fault Triggering During Wheel Rotation

### Symptom

The previous generation board produced **frequent spurious faults** whenever the tractive system was energised and the wheels were turning. The pedal was mechanically fine; the faults were false positives.

### Diagnosis

The pedal sensor signals were wired **directly** into the motor controller's analogue input while simultaneously being read by the microcontroller. With the inverter switching and the motor running, noise coupled back along this shared node. The microcontroller interpreted that noise as a genuine deviation between the two sensor channels and tripped the implausibility fault.

The root cause was therefore neither the sensors nor the firmware threshold — it was **electrical coupling on a shared, unbuffered signal node**.

### Measurement

Oscilloscope capture of the pedal signal at the motor controller input:

| Quantity | Value |
|---|---|
| Measured noise | **1.88 V peak-to-peak** |
| ADC reference | 3.3 V |
| ADC resolution | 12-bit (4096 counts) |

Fraction of full scale corrupted:

$$\frac{1.88\ \text{V}}{3.3\ \text{V}} = 56.97\%$$

Expressed in ADC counts:

$$0.5697 \times 4096 \approx 2333\ \text{counts}$$

More than half the usable input range was noise. No firmware filter or threshold adjustment can recover a signal degraded this badly — the problem had to be solved in hardware.

📷 *Oscilloscope capture of pedal signal noise:* `docs/noise_capture.png`

### Solution

A **buffer stage** was inserted between the pedal sensors and the motor controller input. The buffer presents a high input impedance to the sensor and a low output impedance to the controller, isolating the microcontroller's sensing node from noise fed back along the motor controller line.

In the schematic this stage is marked by the **yellow block**. After implementation, noise-triggered faults no longer occurred during vehicle operation.

📷 *System schematic (buffer stage highlighted):* `docs/schematic_overview.png`

---

## 3. Problem 2 — Microcontroller Freezing

### Symptom

The previous board, built around an **Arduino Nano (ATmega328P)**, would intermittently **freeze** during operation. On a safety-critical system an unresponsive controller is a worse failure mode than a false fault.

### Diagnosis

The freezes traced to **memory starvation**. The ATmega328P provides only 2 KB of SRAM. With the plausibility comparison, the Ready-To-Drive state machine, communication handling, and sensor acquisition all running concurrently, available RAM was exhausted and the controller stalled.

### Solution

Migration to the **ESP32**, which offers substantially more SRAM and a considerably higher clock frequency — enough headroom for all concurrent tasks with margin to spare.

---

## 4. Validation Workflow

Rather than jumping straight to a manufactured board, the change was validated in three stages. Each stage was allowed to fail cheaply before committing to the next.

### Stage 1 — Microcontroller swap test

The existing Arduino Nano pinout was remapped to an ESP32 development board and dropped into the existing circuit. The question at this stage was deliberately narrow: **can the ESP32 fulfil the same role at all?**

It was then taken to **vehicle testing** for validation under real operating conditions — because bench behaviour and on-car behaviour are not the same thing once inverter noise is involved. All functions performed correctly.

📷 *Microcontroller swap test:* `docs/mcu_swap_test.png`

### Stage 2 — Prototype PCB

Once every sub-circuit met its requirement individually, the design was committed to a **prototyping PCB**. This stage exists to allow physical rework: if a circuit misbehaves, it can be cut, patched, and retested on the same board.

Only after this revision cycle produced a design with no outstanding issues was the layout considered settled.

📷 *Prototype PCB:* `docs/prototype_pcb.png`

### Stage 3 — Final PCB

The validated design was manufactured through **JLCPCB** as the final production board.

---

## 5. Design Principles Applied

| Principle | Where it appears |
|---|---|
| Fix noise in hardware, not in software thresholds | Buffer stage, Problem 1 |
| Measure before redesigning | Oscilloscope capture preceded the buffer decision |
| Size the controller for the worst case, not the average | ESP32 migration, Problem 2 |
| Validate on the vehicle, not only on the bench | Stage 1 |
| Let the cheap board fail first | Prototype PCB before JLCPCB |

---

## 6. Hardware Summary

- **Microcontroller:** ESP32
- **Pedal sensing:** dual APPS with continuous plausibility comparison
- **Signal conditioning:** buffer stage isolating the sensor node from the motor controller input
- **Manufacturing:** JLCPCB
- **Application:** FSAE electric race vehicle, Anargya ITS EV Team

---

## 7. Availability of Design Files

Full schematics, board layout, and firmware source are **team-restricted** and not published in this repository. What is documented here is design reasoning and validation methodology.

For access to detailed design files, please get in touch.

---

## References

1. **SAE International — Formula SAE Rules** (APPS plausibility and EV safety requirements): https://www.fsaeonline.com/page.aspx?pageid=c4c5195a-60c0-46aa-acbf-2958ef545b72
2. **Formula Student Germany Rules** — accelerator pedal position sensor and plausibility device requirements: https://www.formulastudent.de/fsg/rules/
3. Espressif Systems — *ESP32 Technical Reference Manual*: https://www.espressif.com/sites/default/files/documentation/esp32_technical_reference_manual_en.pdf
4. Espressif — *ADC calibration and noise considerations* (ESP-IDF documentation): https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/peripherals/adc_calibration.html
5. Microchip — *ATmega328P Datasheet* (2 KB SRAM specification): https://ww1.microchip.com/downloads/en/DeviceDoc/Atmel-7810-Automotive-Microcontrollers-ATmega328P_Datasheet.pdf
6. Texas Instruments — *Op Amps for Everyone* (buffer and impedance-matching stages): https://www.ti.com/lit/an/slod006b/slod006b.pdf
7. Henry W. Ott — *Electromagnetic Compatibility Engineering*, Wiley, 2009 — standard reference for coupled noise in mixed-signal systems.
