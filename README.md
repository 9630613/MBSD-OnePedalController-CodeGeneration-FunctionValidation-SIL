# 🤖 Embedded Deployment — One-Pedal Controller on Arduino (Code Generation & HIL Testing)

> Continuation of the [One-Pedal Drive Controller](../one-pedal-controller) and [Functional Safety](../one-pedal-safety) projects. This phase covers **automatic C code generation** from the Simulink controller model targeting an **Arduino microcontroller**, full **electrical interface mapping**, and **Hardware-in-the-Loop (HIL) validation** using SimulIDE.

---

## 📌 Project Overview

This project bridges the gap between model-based design and physical embedded deployment. The complete one-pedal controller — including its functional safety layer — is compiled and flashed onto an **Arduino** board using the **Simulink Support Package for Arduino Hardware**. The electrical interfaces are designed and wired in **SimulIDE**, and the full controller functionality (including safe state) is validated through interactive hardware simulation.

Key contributions:
- Mapping of all controller signals to Arduino physical pins (digital, analog, PWM)
- ADC conversion formula derivation for all analog inputs
- PWM encoding of the torque output signal
- Translation layers for enum-based transmission selector states to/from digital pins
- SimulIDE harness design for interactive real-time testing
- Full functional test coverage including normal operation modes and safe state activation

---

## 🔌 I/O Interface Design

All controller signals from the Simulink model are mapped to physical Arduino electrical interfaces. The table below summarizes the complete I/O mapping with conversion formulas and signal ranges.

### Interface Table

| **Signal Name** | **Unit** | **Pin Type** | **Conversion Formula** | **Min** | **Max** |
|---|---|---|---|---|---|
| Brake Pedal Pressed | — | Digital Input (DI) | — | 0 | 1 |
| Throttle Pedal Position | — | Analog Input (AI) | `5 / (2¹⁰ − 1) × 1/5` | 0 | 1 |
| Vehicle Speed | km/h | Analog Input (AI) | `5 / (2¹⁰ − 1) × (275/5)` | −45 | 275 |
| Redundant Throttle Pedal Position | — | Analog Input (AI) | `5 / (2¹⁰ − 1) × 1/5` | 0 | 1 |
| CAN Disconnection | — | Digital Input (DI) | — | 0 | 1 |
| Transmission Selector (R, B, P, N, D) | — | Digital Input (DI) × 5 | — | 0 | 1 |
| Requested Torque | N·m | PWM Analog Output | `255 / (80 + 80)` | −80 | +80 |
| Transmission State Output (R, B, P, N, D) | — | Digital Output (DO) × 5 | — | 0 | 1 |
| Safety Warning Lamp | — | Digital Output (DO) | — | 0 | 1 |

> 📷 *See `/images/controller_model_codegen.png` for the annotated controller model ready for code generation.*

---

## ⚙️ Signal Conversion Details

### Analog Inputs — ADC Formula

The Arduino Uno uses a **10-bit ADC** with a **5 V reference**, giving:

```
V_in = (V_ref / (2^N − 1)) × ADC_output
     = (5 / 1023) × ADC_output
```

Each analog input is then linearly scaled to its physical range:

#### Throttle Pedal Position (normalized [0, 1])
```
PedalPosition = ADC_output × (5 / 1023) × (1 / 5)
              = ADC_output / 1023
```

#### Vehicle Speed (km/h, range [−45, 275])
```
Speed_km_h = ADC_output × (5 / 1023) × (275 / 5) + (−45)
           = ADC_output × (275 / 1023) − 45
```
The full input range (275 + 45 = 320 km/h) is mapped to the 0–5 V analog input. A potentiometer (volume knob) in SimulIDE drives this input.

#### Redundant Throttle Pedal Position
Identical formula to the primary throttle pedal:
```
RedundantPosition = ADC_output × (5 / 1023) × (1 / 5)
```

---

### PWM Output — Requested Torque

The torque output spans a **signed range [−80, +80] N·m**, which is encoded into the **unsigned 8-bit PWM range [0, 255]**:

```
PWM_value = (TorqueRequest_Nm + 80) × (255 / 160)
```

| Torque (N·m) | PWM Value | Duty Cycle | Meaning |
|---|---|---|---|
| −80 | 0 | 0% | Maximum regenerative braking |
| 0 | 127 | ~50% | Zero torque (neutral / stopped) |
| +80 | 255 | 100% | Maximum drive torque |

> A **50% duty cycle** represents zero torque. Values above 50% drive the motor forward; values below 50% command regenerative braking. This is visible in SimulIDE using an oscilloscope on the PWM output pin.

---

### Transmission Selector — Enum ↔ Digital Pins

Since Arduino digital pins handle only binary signals, the **5 transmission states (P, R, N, D, B)** are encoded as 5 separate digital lines — one per state. A Stateflow block inside the model performs the encoding/decoding:

#### Input side (5 DI pins → enum):
Each digital pin corresponds to one selector state. The active pin sets the enum value used by the controller FSM.

#### Output side (enum → 5 DO pins):
The controller's current `AutoTransmissionState` enum is decoded back to 5 individual digital output pins, driving 5 LEDs on the SimulIDE harness to indicate the active mode visually.

> 📷 *See `/images/selector_input_stateflow.png` and `/images/selector_output_stateflow.png` for the internal Stateflow encoding logic.*

---

## 🔧 Code Generation for Arduino

### Steps to Generate Firmware

1. **Replace Simulink I/O ports** with Arduino Support Package blocks:
   - `Digital Input` / `Digital Output` blocks for binary signals
   - `Analog Input` block for ADC readings (throttle, speed, redundant pedal)
   - `PWM` output block for torque command

2. **Insert ADC conversion gain blocks** immediately after each analog input, applying the formula `V_ref / (2^N − 1)` with the appropriate scaling factor per signal.

3. **Scale and offset** each analog signal to its physical unit using a Gain + Sum block pair.

4. **Insert PWM encoding** before the torque output: add the offset (+80), multiply by the scale factor (255/160), and feed into the PWM block.

5. **Configure target hardware** in Simulink → Model Settings → Hardware Implementation → Arduino Uno.

6. **Build and deploy** using the Simulink Support Package deploy button, which cross-compiles to AVR C and flashes via USB.

> 📷 *See `/images/arduino_instrumented_model.png` for the full controller model with Arduino I/O blocks.*

---

## 🧪 SimulIDE Harness

The hardware circuit was designed and simulated in **SimulIDE**, replicating the Arduino's physical connections:

### Components Used

| Component | Purpose |
|---|---|
| Push-button switches (with resistors) | Digital inputs: brake pedal, CAN disconnection, transmission selector (×5) |
| Potentiometers (5 V volume knobs) | Analog inputs: throttle pedal position, vehicle speed, redundant pedal |
| LEDs (with resistors) | Digital outputs: active transmission state (×5), safety warning lamp |
| Oscilloscope | Visualizing PWM duty cycle of the torque output signal |

> 📷 *See `/images/simulide_harness.png` for the complete wired harness.*

---

## ✅ Functional Test Scenarios

All test scenarios were executed interactively in SimulIDE, manually applying stimuli through the harness components. The following sequence was validated:

### Test Scenario 1 — Normal State Transitions

| Step | Action | Expected Behavior | Result |
|---|---|---|---|
| 1 | Power on | Controller initializes in **Park** state | ✅ |
| 2 | Press brake + select Neutral | Transitions to **Neutral** | ✅ |
| 3 | Select Drive (velocity < 5 km/h) | Remains in Neutral — speed guard active | ✅ |
| 4 | Increase velocity above 5 km/h, select Drive | Transitions to **Drive** | ✅ |
| 5 | Select Brake (pedal position < 1/3) | Remains in Drive | ✅ |
| 6 | Increase pedal position > 1/3, select Brake | Transitions to **Brake** mode | ✅ |
| 7 | Rotate pedal volume to vary position | PWM duty cycle varies on oscilloscope (torque changes) | ✅ |
| 8 | Confirm max/min torque | PWM at ~100% (full drive) and ~0% (full regen braking) | ✅ |
| 9 | Select Reverse (velocity > 5 km/h) | Remains in Neutral — speed guard active | ✅ |
| 10 | Decrease velocity < 5 km/h, select Reverse | Transitions to **Reverse** | ✅ |
| 11 | Select Park (velocity < 5 km/h) | Transitions to **Park** | ✅ |

> 📷 *See `/images/test_state_transitions.png` for simulation captures of each transition.*

---

### Test Scenario 2 — Safe State Activation (CAN Disconnection)

**Setup:** Controller in Drive mode, throttle pedal at high position → PWM duty cycle well above 50% (high positive torque visible on oscilloscope).

**Action:** Activate `CAN_Disconnection` digital input (switch ON).

**Expected behavior:**
- `AutoTransmissionState` → **Neutral**
- `TorqueRequest_Nm` → **0** (PWM duty cycle → **~50%**)
- `SafeStateAlarm` → **ON** (warning LED)

**Result:** ✅ Immediately upon CAN disconnection signal, the oscilloscope shows PWM dropping to 50% duty cycle and the transmission state LED switches to Neutral.

> 📷 *See `/images/safe_state_can_test.png` for oscilloscope capture before and after CAN disconnection.*

---

### Test Scenario 3 — Safe State Activation (Pedal Sensor Error)

**Note:** In SimulIDE it is not possible to change two potentiometers simultaneously to generate a realistic differential error between the primary and redundant pedal sensors. This scenario was verified at the model level (Simulink simulation) rather than in the physical harness. The CAN disconnection test above confirms the safe state logic path is correctly wired in hardware.

---

### Test Scenario 4 — PWM Torque Encoding Verification

The requested torque is encoded as PWM with 50% duty cycle = 0 N·m. This was validated visually using the SimulIDE oscilloscope:

| Controller State | Pedal Position | Expected Torque | PWM DC | Observed |
|---|---|---|---|---|
| Drive | 1.0 (max) | +80 N·m | ~100% | ✅ |
| Drive | 0.5 | +40 N·m | ~75% | ✅ |
| Brake / Decelerate | 0.0 (min) | −80 N·m | ~0% | ✅ |
| Safe State | any | 0 N·m | ~50% | ✅ |

---

## ⚠️ Implementation Notes & Limitations

- **Warning Lamp pin:** The safety alarm LED could not be implemented on the Arduino Uno in this configuration because all available digital output pins were already allocated to transmission state indicators and other outputs. The alarm signal is present in the model but not connected to a physical pin.

- **Redundant pedal test in hardware:** Simultaneous adjustment of two potentiometers to generate a controlled differential is not straightforward in SimulIDE. This is a known limitation of the simulation environment — in a real hardware setup, two separate physical sensors would be connected to two ADC pins.

- **Vehicle speed input:** In the absence of the full physical plant in SimulIDE, vehicle speed is manually simulated with a potentiometer. This means state transition guards based on speed (e.g., entering Drive or Reverse) must be set up manually by the tester.

---

## 🔗 Project Series

This is the **third and final project** in the one-pedal drive series:

| Project | Focus |
|---------|-------|
| [One-Pedal Controller](../one-pedal-controller) | FSM-based controller design in Simulink/Stateflow |
| [Functional Safety Concept](../one-pedal-safety) | ISO 26262 HARA, safety goals, Safety Function implementation |
| **This project** | Code generation for Arduino, electrical interface mapping, HIL testing in SimulIDE |

---

## 📁 Repository Structure

```
├── Controller_Arduino.slx         # ← Controller with Arduino I/O blocks (original)
├── init_fn.m                      # Model initialization parameters
├── images/
│   ├── controller_model_codegen.png      # Simulink model ready for code gen
│   ├── selector_input_stateflow.png      # DI → enum Stateflow logic
│   ├── selector_output_stateflow.png     # enum → DO Stateflow logic
│   ├── arduino_instrumented_model.png    # Full model with Arduino blocks
│   ├── simulide_harness.png              # SimulIDE wiring diagram
│   ├── test_state_transitions.png        # Normal mode transition captures
│   ├── safe_state_can_test.png           # Oscilloscope: safe state activation
│   └── pwm_torque_verification.png       # PWM duty cycle vs torque table
└── README.md
```

---

## 🛠️ Tools & Technologies

| Tool | Purpose |
|------|---------|
| **MATLAB / Simulink** | Controller model, code generation configuration |
| **Stateflow** | Enum ↔ digital pin encoding/decoding logic |
| **Simulink Support Package for Arduino Hardware** | AVR C code generation and deployment |
| **Arduino Uno** | Target microcontroller (10-bit ADC, PWM, DI/DO) |
| **SimulIDE** | Electronic circuit simulation for HIL testing |

---

## 👩‍💻 Author

**Zahra Sadeghi Jalalabadi** — s329869
Model-Based Software Design — A.Y. 2023/24

---

## 📄 License

Developed as part of academic coursework. Controller adaptation for Arduino, interface conversion design, SimulIDE harness, and all test scenarios are original work. The base controller model and plant were developed in the preceding projects of this series.
