# Project Brief: Closed-Loop Sensorless SVPWM BLDC Motor Controller
*(Full context handoff document — covers design evolution, final architecture, complete BOM, and build roadmap)*

---

## 1. Project Overview

**Original title:** "Implementation of Closed-Loop Space Vector PWM (SVPWM) for BLDC Motor Control using STM32 and Hall-Effect Feedback"

**Note on title vs. current implementation:** the project's naming still references "Hall-Effect Feedback," but the design has since evolved to a **sensorless (back-EMF)** approach due to difficulty sourcing a Hall-sensor-equipped BLDC motor. The core SVPWM/closed-loop concept is unchanged; only the rotor-position-sensing method changed. This should be flagged/renamed in any final report.

**Core objective:** Build a closed-loop BLDC motor speed controller using an STM32F446RE Nucleo board that:
1. Reads desired speed from a potentiometer
2. Estimates actual rotor speed/position from the motor itself
3. Computes speed error (target − actual)
4. Runs the error through a PI controller
5. Generates SVPWM signals via STM32 TIM1
6. Drives a 3-phase inverter (6 MOSFETs) through a gate driver stage
7. Continuously corrects the motor command to hold the target RPM, even under changing mechanical load

---

## 2. Working Principle

**Why BLDC + SVPWM:** BLDC motors need their three phases energized in a rotating sequence. Basic "six-step commutation" is simple but causes torque ripple and acoustic noise. **SVPWM (Space Vector PWM)** instead calculates continuous PWM duty cycles for all three phases together, treating the desired output as a rotating voltage vector in a 2D plane divided into six sectors — producing smoother rotation and better DC bus utilization than six-step switching.

**Closed-loop control:**
```
Error = Target RPM − Actual RPM
PI controller: Error → Voltage magnitude command
Voltage magnitude command → SVPWM → PWM duty cycles → Inverter → Motor
```
If the motor slows under load, the speed estimator detects it, the PI controller increases the voltage command, and SVPWM adjusts the inverter output accordingly — no manual intervention needed.

**Important conceptual note carried from the original design:** SVPWM (how the inverter is switched) and rotor-position sensing (Hall / encoder / sensorless) are independent design choices. This project pairs SVPWM with sensorless back-EMF sensing, but the SVPWM math itself doesn't require any particular sensing method.

---

## 3. Design Evolution (why choices changed)

| Decision point | Original plan | Final choice | Reason for change |
|---|---|---|---|
| Rotor position sensing | 3x Hall-effect sensors (digital states: 001, 011, 010, 110, 100, 101) | **Sensorless back-EMF zero-crossing detection** | Hall-sensor-equipped BLDC motors were hard to source; generic sensorless motors are widely available (drone/RC market) |
| Motor | Generic 3-phase BLDC with integrated Hall sensors | **Electronic Spices A2212, 2200KV, sensorless outrunner** | Availability; standard hobby/drone motor |
| Gate driver | DRV8302 module (from original proposal) | **3x discrete IR2110 half-bridge driver ICs** | Cheaper, more widely available locally, and standard/well-documented in DIY BLDC/ESC builds |

**Sensorless back-EMF principle:** as the rotor spins, it induces a back-EMF voltage in the undriven phase. The STM32 samples this (via a scaled-down ADC signal) and detects zero-crossings to estimate rotor angle/speed. **Limitation:** back-EMF is ~zero at standstill, so the system needs a forced **open-loop alignment + ramp** sequence at startup, then hands off to closed-loop sensorless operation once the signal is strong/stable enough. This handoff is widely considered the hardest part of sensorless BLDC control.

---

## 4. Final System Architecture

```
Potentiometer (target RPM)
        ↓
Error = Target − Actual RPM
        ↓
PI Controller → voltage magnitude command
        ↓
SVPWM (STM32 TIM1: sector ID + duty cycle calc, center-aligned PWM, hardware dead time)
        ↓
3x IR2110 gate driver ICs (one per phase, with bootstrap diode+cap for high-side drive)
        ↓
6x MOSFETs (3-phase inverter bridge: high-side + low-side per phase)
        ↓
A2212 BLDC motor (2200KV, sensorless, 3-phase outrunner)
        ↓
Phase voltages tapped by 3x voltage-divider + filter networks
        ↓
Back-EMF zero-crossing detection (STM32 ADC) → estimated actual RPM
        ↓
(feeds back into the Error calculation, closing the loop)
```

**Supporting elements (not in the control loop but essential):**
- OLED (SSD1306, I2C) displays target vs. actual RPM in real time
- Power supply: main motor/inverter bus (2S–3S LiPo, 7.4–11.1V) + a separate 12–15V rail specifically for the IR2110 gate-drive logic supply (since it needs more headroom than the motor bus)
- Oscilloscope used throughout bring-up to verify PWM, dead time, and especially the back-EMF zero-crossing waveform

---

## 5. Motor Details & Constraints (A2212, 2200KV)

- Operating voltage: 7.4–11.1V (2S–3S LiPo)
- Shaft diameter: 3.17mm
- Sensorless, 3-phase outrunner — no integrated Hall sensors
- **No-load speed estimate:** 2200KV × 11.1V ≈ 24,420 RPM (theoretical, no load) — plan to run at reduced voltage/duty cycle for a first working prototype rather than full speed
- **Pole count not specified by seller** — A2212 motors are typically 14-pole (7 pole-pairs); this must be confirmed (or measured by counting stator magnets) since electrical RPM = mechanical RPM × pole pairs, and getting this wrong will make the RPM display/setpoint wrong by that factor
- **Current rating not specified** — A2212 variants often draw 10A+ under load; MOSFETs and gate drivers must be sized with margin above this
- Motor will spin nearly unloaded by default; a small mechanical load (friction pad or fan blade) is recommended to actually demonstrate closed-loop load-rejection behavior, since an unloaded motor makes the PI controller's job trivially easy

---

## 6. Complete Bill of Materials with Ratings

### Microcontroller & UI
| Component | Rating / Spec | Qty |
|---|---|---|
| STM32F446RE Nucleo | ARM Cortex-M4, 180 MHz, 3.3V logic, TIM1 advanced timer with dead-time unit | 1 |
| Potentiometer | 10 kΩ, linear taper, ¼W | 1 |
| OLED display | SSD1306, 128×64, I²C, 3.3–5V supply | 1 |

### Motor
| Component | Rating / Spec | Qty |
|---|---|---|
| A2212 BLDC motor | 2200KV, 7.4–11.1V (2–3S), 3-phase sensorless outrunner, 3.17mm shaft | 1 |

### Gate driver stage (per phase ×3)
| Component | Rating / Spec | Qty |
|---|---|---|
| IR2110 gate driver IC | 500V offset voltage, VCC 10–20V, 2A source/sink gate drive current | 3 |
| Bootstrap diode | UF4007 (1000V, 1A, ~75ns trr) or UF4004 (400V, 1A, lower Vf ~1.0V) | 3 |
| Bootstrap capacitor | 1–2.2 µF, 25V, ceramic/tantalum | 3 |
| VCC bulk decoupling capacitor | 10 µF, 25V, electrolytic | 3 |
| VCC local decoupling capacitor | 100 nF, 50V, ceramic | 3 |
| Gate resistor | 15 Ω (10–33 Ω range), ¼–½W | 6 (2 per phase) |
| Gate-source pulldown resistor | 10 kΩ, ¼W | 6 |

### Power stage (inverter bridge)
| Component | Rating / Spec | Qty |
|---|---|---|
| Power MOSFET | N-channel, logic-level (Vgs(th) ≤ 4V), ≥30V Vds, ≥20A Id (e.g. IRFZ44N: 55V, 49A, Rds(on) ~17.5mΩ) | 6 |
| DC bus bulk capacitor | 470–1000 µF, 25V (≥1.5× supply voltage) | 1–2 |
| Per-leg snubber/decoupling capacitor | 100 nF, 50V, ceramic | 3 |

### Back-EMF sensing network (per phase ×3)
| Component | Rating / Spec | Qty |
|---|---|---|
| Divider resistor (top) | 10 kΩ, ¼W, 1% tolerance preferred | 3 |
| Divider resistor (bottom) | 1 kΩ, ¼W, 1% tolerance preferred | 3 |
| Filter capacitor | 1–10 nF, 50V, ceramic | 3 |

### Power supply
| Component | Rating / Spec | Qty |
|---|---|---|
| Main motor/bus supply | 2S–3S LiPo, 7.4–11.1V, or bench DC supply ≥15A | 1 |
| Gate driver supply rail | 12–15V, ≥500mA (small buck/boost module, separate from motor bus) | 1 |
| Fuse/PTC resettable fuse | Rated ~1.5× expected max current (15–20A) | 1 |

### Mechanical & wiring
| Component | Spec |
|---|---|
| Motor mount/clamp | Sized for A2212 outrunner body diameter |
| Mechanical load | Friction pad or small fan blade, light/adjustable |
| Bullet connectors | 3.5mm or 4mm, gender matching motor leads |
| Perfboard | Preferred over breadboard for the power stage (breadboard contact resistance/current limits are unreliable above 1–2A) |
| Jumper wires | 22–24 AWG for signal, 18–20 AWG for power/motor leads |
| Heatsinks | TO-220 clip-on, sized for MOSFET package |

### Verification equipment
| Equipment | Spec |
|---|---|
| Oscilloscope | ≥2 channels, ≥20 MHz bandwidth |
| Multimeter | Standard DMM (continuity/voltage/current) |

---

## 7. Build & Firmware Roadmap (phased, hardware-safe order)

1. **Build and verify PWM generation** — Configure STM32 TIM1 for three center-aligned complementary PWM pairs with hardware dead time (~1–2 µs). No driver board or motor connected. Verify all six outputs on the oscilloscope, confirming correct dead time and no high/low overlap on the same leg.

2. **Bring up the gate driver and MOSFET bridge** — Wire the three IR2110s (with bootstrap diodes/capacitors) and six MOSFETs into the inverter bridge, powered from the separate 12V driver rail. Motor still disconnected. Feed in PWM and scope the gate-drive waveforms and switching-node voltages before risking the motor.

3. **Spin the motor open loop** — Connect the A2212's phase leads. Generate a software ramp angle and run SVPWM sector/duty calculations to spin the motor with no feedback yet. Confirms inverter, wiring, and SVPWM math together.

4. **Add back-EMF sensing** — Wire the three voltage-divider/filter networks from each motor phase into STM32 ADC pins. While the motor spins open loop (from Phase 3), log divided phase voltages and confirm clean, timeable zero-crossings; check estimated speed against the known open-loop reference speed.

5. **Implement the sensorless startup handoff** — Add a forced open-loop alignment/ramp routine for startup (where back-EMF is unusable), then switch to the zero-crossing-based angle/speed estimate once the signal is strong and consistent. This handoff is typically the hardest part to tune.

6. **Close the loop and validate** — Feed potentiometer target RPM and back-EMF-derived actual RPM into the PI controller, whose output becomes the SVPWM voltage magnitude command. Add a small mechanical load to demonstrate load-rejection, update the OLED with target/actual RPM, and capture final phase-voltage/PWM waveforms on the scope as validation evidence.

---

## 8. Open Items / Unconfirmed Details

- **Pole-pair count of the A2212 motor** — not yet confirmed; needed for correct electrical-to-mechanical RPM scaling in firmware
- **Actual current draw under the intended load** — not yet measured; affects final MOSFET/wiring margin sizing
- **Exact PI gains (Kp, Ki)** — not yet determined; will need tuning once the closed loop is running
- **Bootstrap capacitor/diode exact values and dead-time RC sizing** — general ranges given above; final values should be checked against the specific IR2110 datasheet application circuit before finalizing the PCB/perfboard layout
- **Boost/buck converter sizing for the 12–15V gate-driver rail** — not yet specified in detail
