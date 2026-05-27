# Hierarchical EV Battery Intelligence and Protection System

## Self-Calibrated Mixed-Signal Battery Diagnostics, Electro-Thermal Safety, Aging-Aware Current Policy, and Fast-Fault Protection

This project presents a simulation-validated EV battery diagnostics and protection architecture developed using **LTspice** and **Python**. The system integrates analog front-end sensing, adaptive calibration, internal resistance estimation, electro-thermal modeling, aging-aware current control, thermal derating, runaway precursor detection, SAR ADC sampled-monitoring support, and millisecond-scale fast-fault shutdown.

The project is structured as a layered battery protection system where raw battery measurements are first conditioned and corrected, then used for estimation, thermal-risk assessment, current policy generation, and protection decisions.

---

## System Overview

The overall flow of the system is:

```text
Battery / Cell Inputs
        ↓
Analog Front-End Sensing
        ↓
Adaptive Calibration
        ↓
RLS-Based Internal Resistance Estimation
        ↓
Electro-Thermal Modeling
        ↓
Health and Risk Assessment
        ↓
Aging-Aware Current Policy
        ↓
Hierarchical Protection Logic
        ↓
Warning / Derating / Shutdown / Fast-Fault Response
```

Add the system-level architecture image here:

```markdown
![System Architecture](images/system_architecture.png)
```

The architecture contains two main paths:

1. **Protection and diagnostics path**
   This path uses calibrated voltage, current, temperature, thermal stress, aging indicators, and fault flags to generate warning, derating, and shutdown decisions.

2. **Sampled monitoring path**
   The SAR ADC subsystem demonstrates how calibrated analog signals can be sampled and converted into digital form for future microcontroller, FPGA, or ASIC-style BMS integration.

The ADC is treated as a sampled-monitoring support block. The main protection behavior is validated using analog/electro-thermal supervisory logic, LTspice simulation, and Python-based post-processing.

---

## Project Objectives

The main objectives of this project are:

* To model an EV battery protection system beyond fixed threshold-based monitoring.
* To improve measurement reliability using adaptive calibration.
* To estimate effective internal resistance for aging and health tracking.
* To connect electrical stress with thermal behavior using an electro-thermal model.
* To implement staged protection through warning, derating, and shutdown.
* To adjust current policy according to battery aging condition.
* To provide a separate fast-fault path for electrical faults.
* To validate the complete system using case-wise LTspice simulation and Python analysis.

---

## Repository Structure

```text
EV-Battery-Intelligence-and-Protection-System/
│
├── README.md
│
├── images/
│   ├── system_architecture.png
│   ├── subsystem_1_afe_sensing.png
│   ├── subsystem_2_adaptive_calibration.png
│   ├── subsystem_3_rls_estimation.png
│   ├── subsystem_4_electro_thermal_model.png
│   ├── subsystem_5_thermal_protection.png
│   ├── subsystem_6_aging_policy.png
│   ├── subsystem_7_fast_fault.png
│   └── subsystem_8_sar_adc.png
│
├── results/
│   ├── calibration_result.png
│   ├── case2_thermal_derating.png
│   ├── case3_runaway_prevention.png
│   ├── case4_case5_aging_policy.png
│   ├── case6_fast_fault_latency.png
│   ├── adc_validation.png
│   └── rls_estimation.png
│
├── ltspice_overview/
│   ├── integrated_schematic.png
│   └── subsystem_schematic_views.png
│
└── docs/
    ├── project_report.pdf
    └── presentation.pdf
```

---

# Subsystem 1: Analog Front-End Sensing

<img width="1448" height="1086" alt="image" src="https://github.com/user-attachments/assets/42b20f74-cd7d-4c4f-8cd7-c9179591874b" />


## Function

The analog front-end acquires the three primary battery signals required for monitoring and protection:

* Cell / pack voltage
* Pack current
* Cell / pack temperature

These signals are conditioned before being passed to the calibration, estimation, ADC, and protection blocks.

## Signal Flow

```text
Battery Voltage / Current / Temperature
        ↓
Input Protection
        ↓
Scaling / Sensing Interface
        ↓
Amplification
        ↓
RC Low-Pass Filtering
        ↓
Buffered Analog Output
        ↓
Calibration and Diagnostic Blocks
```

## Voltage Sensing Path

The voltage sensing path scales the battery voltage into a safe measurable range. Protection elements and filtering are used before the signal is passed to the calibration stage.

The voltage path provides:

```text
V_sense_raw
```

This signal is later corrected by the adaptive calibration block.

## Current Sensing Path

The current sensing path is based on shunt measurement. The voltage across the shunt is amplified and filtered to obtain a current-related signal.

The current path provides:

```text
I_sense_raw
```

This signal is used for:

* Joule heating calculation
* current derating
* fast-fault detection
* RLS estimation
* aging-aware policy

## Temperature Sensing Path

The temperature path provides the thermal state of the battery. This signal is used by the electro-thermal model and protection logic.

The temperature-related signal contributes to:

```text
Tcell_abs
dT/dt
THERM_STRESS
RUNAWAY_PRECURSOR
```

## Design Note

The sensing stage is placed before all decision-making blocks so that the later algorithms receive properly conditioned physical signals rather than raw noisy measurements.

---

# Subsystem 2: Adaptive Calibration and Reference-Window Correction

<img width="1448" height="1086" alt="image" src="https://github.com/user-attachments/assets/af1033f2-2ffb-4c9a-ad48-10449209af49" />


## Function

The calibration subsystem reduces measurement offset and drift in the sensed voltage and current paths. It works by periodically switching between a known reference condition and normal measurement condition.

## Operating Modes

### Reference Mode

During reference mode, the system connects the sensing path to a known reference value. The measured output is compared against the expected reference level to estimate error.

```text
REF_MODE = 1
MEAS_MODE = 0
```

### Measurement Mode

During measurement mode, the system returns to the actual battery signal. The previously estimated error is applied as a correction.

```text
REF_MODE = 0
MEAS_MODE = 1
```

## Correction Logic

The basic correction idea is:

```text
Estimated Error = Measured Reference - Expected Reference

Corrected Signal = Raw Signal - Estimated Error
```

In the voltage path, this produces:

```text
V_cal_out
```

In the current path, this produces:

```text
I_cal_out
```

## Important Signals

```text
REF_MODE
MEAS_MODE
V_mux_in
V_cal_err_raw
V_CAL_HOLD
V_cal_corr
V_cal_out
I_CAL_HOLD
I_cal_corr
I_cal_out
CAL_VALID
```

## Validation Result

The calibration analysis showed:

```text
Raw RMS Error             ≈ 1.957 V
Settled Corrected RMS     ≈ 0.419 V
RMS Error Reduction       ≈ 78.6%
```

## Design Note

Calibration is kept before RLS, ADC, and protection logic because parameter estimation and fault decisions depend strongly on measurement quality.

---

# Subsystem 3: RLS-Based Internal Resistance Estimation

<img width="1448" height="1086" alt="image" src="https://github.com/user-attachments/assets/6b00bf50-5585-4431-b989-77f2fe6cf04d" />


## Function

This subsystem estimates the effective internal resistance of the battery:

```text
R0eff
```

R0eff is used as an aging and health-related parameter.

## Why R0eff Is Used

Battery internal resistance changes with:

* aging
* temperature
* operating condition
* degradation
* current stress

As R0eff increases, the same current produces more heat.

```text
Qirr = I²R0eff
```

So R0eff connects electrical behavior with thermal safety.

## Basic Pulse-Based Method

A simple estimate can be obtained from voltage and current change:

```text
R0 = ΔV / ΔI
```

This gives a snapshot of resistance.

## RLS-Based Method

Recursive Least Squares is used to estimate resistance dynamically from voltage-current behavior.

The general RLS update is:

```text
θ(k) = θ(k-1) + K(k)[y(k) - φ(k)^T θ(k-1)]
```

Where:

```text
θ(k)     = estimated parameter vector
K(k)     = RLS gain
y(k)     = measured output
φ(k)     = input feature vector
error    = y(k) - φ(k)^T θ(k-1)
```

## Battery Model Used for Estimation

The estimation is based on an equivalent circuit view:

```text
Voc ─ R0 ─ (R1 || C1)
```

Where:

* R0 represents immediate ohmic resistance.
* R1 and C1 represent transient polarization behavior.
* Voc represents open-circuit voltage.

## Output of the RLS Block

The RLS block provides:

```text
Estimated R0eff
Predicted voltage
Prediction error
Aging trend
Health-related diagnostic information
```

## Design Note

RLS allows resistance estimation to update over time instead of depending only on one fixed value. This makes the health and current policy layers responsive to battery condition.

---

# Subsystem 4: Electro-Thermal Modeling

<img width="1448" height="1086" alt="image" src="https://github.com/user-attachments/assets/f073d2f8-474c-431a-8333-1d64b6bcf4d6" />


## Function

The electro-thermal model converts electrical loading into temperature behavior. It helps the protection logic understand how current and resistance affect heating.

## Thermal Energy Balance

```text
Cth · dT/dt = I²R + Qrev + Qrxn − (Tcell − Tamb)/Rth
```

## Meaning of Each Term

| Term  | Meaning                                     |
| ----- | ------------------------------------------- |
| Cth   | Thermal capacitance of the cell             |
| dT/dt | Rate of temperature rise                    |
| I²R   | Joule heating due to current and resistance |
| Qrev  | Reversible / entropic heat                  |
| Qrxn  | Reaction or exothermic heat                 |
| Tcell | Cell temperature                            |
| Tamb  | Ambient temperature                         |
| Rth   | Thermal resistance to ambient               |

## Thermal Interpretation

The model follows this physical idea:

```text
Heat generated inside the cell
        -
Heat removed to ambient
        =
Net heat stored in the cell
```

If heat generation is greater than cooling, temperature rises.
If cooling is greater than heat generation, temperature stabilizes or falls.

## Signals Derived

```text
Tcell_abs
TH_TDELTA
DTDT_f
THERM_STRESS
RUNAWAY_PRECURSOR
```

## Design Note

The model uses both temperature and temperature-rise rate. This allows the protection layer to observe not only where the temperature is, but also how quickly it is moving.

---

# Subsystem 5: Hierarchical Thermal Protection Logic

<img width="1448" height="1086" alt="image" src="https://github.com/user-attachments/assets/e3853b17-392f-4289-80d0-3cb478f14982" />


## Function

The thermal protection logic converts thermal state and risk indicators into controlled system actions.

## Protection Sequence

```text
Normal Operation
        ↓
Thermal Warning
        ↓
Charge Limit
        ↓
System Derating
        ↓
Thermal Shutdown
```

## Inputs

```text
Tcell_abs
DTDT_f
THERM_STRESS
RUNAWAY_PRECURSOR
RISK_SCORE
HEALTH_SCORE
R0eff
```

## Outputs

```text
SYS_WARN
CHG_LIMIT
SYS_DERATE
THERM_SHDN
FINAL_SHDN
ICMD_SCALE_FINAL
```

## Case 2: Thermal Derating

In the thermal derating case, the controller reduced current as thermal stress increased.

Measured results:

```text
Thermal drive reduction    ≈ 31.8%
dT/dt reduction            ≈ 67.1%
```

This shows that the derating action affected the thermal response, not only the logic state.

## Case 3: Runaway Prevention

In the runaway prevention case, the system detected precursor behavior before shutdown.

Measured timing:

```text
Warning window             ≈ 1126.8 s
Intervention window        ≈ 356.5 s
Total safety window        ≈ 1483 s
```

## Design Note

The protection sequence allows the system to reduce stress first and reserve shutdown for persistent or critical conditions.

---

# Subsystem 6: Aging-Aware Health and Current Policy

<img width="1448" height="1086" alt="image" src="https://github.com/user-attachments/assets/bdca2d41-cc9a-47bb-93a5-ec3936957fd4" />


## Function

This subsystem adjusts the allowed current according to battery health and aging condition.

## Input Features

```text
R0eff
HEALTH_SCORE
AGE_STRESS
RISK_SCORE
Tcell_abs
Current state
Voltage state
```

## Output

```text
ICMD_SCALE_FINAL
```

This is the final allowed current fraction.

```text
0 ≤ ICMD_SCALE_FINAL ≤ 1
```

The actual command can be interpreted as:

```text
Allowed Current = Requested Current × ICMD_SCALE_FINAL
```

## Case 4 and Case 5 Behavior

| Case   | Condition     | Current Scale |
| ------ | ------------- | ------------- |
| Case 4 | Aging warning | ≈ 0.70        |
| Case 5 | Severe aging  | ≈ 0.251       |

## Interpretation

When aging indicators become stronger, the allowed current is reduced. This links battery health to current control.

## Policy Model Validation

Python analysis compared observed current policy with health and resistance indicators. The policy model showed strong alignment with the health-aware current behavior.

Approximate result:

```text
Policy model R² ≈ 0.995
```

## Design Note

The current policy is not only based on instantaneous current or voltage. It also depends on the estimated condition of the battery.

---

# Subsystem 7: Fast-Fault Detection and Electrical Shutdown

<img width="1448" height="1086" alt="image" src="https://github.com/user-attachments/assets/3cf0aad0-cc70-403e-98f8-e92e64f2048d" />


## Function

This subsystem handles fast electrical faults such as overcurrent and short-circuit events.

Thermal faults evolve slowly, but electrical faults can become unsafe within milliseconds. For that reason, the fast-fault block is separated from the slower thermal protection path.

## Fast-Fault Signal Flow

```text
Current / Fault Sensing
        ↓
Fast Comparator
        ↓
FAST_FAULT_RAW
        ↓
Validation / Hold Logic
        ↓
FAST_TRIP_VALID
        ↓
FAST_SHDN
        ↓
SYS_FAULT
        ↓
FINAL_SHDN
```

## Important Signals

```text
OC_FAST_RAW
FAST_FAULT_RAW
FAST_TRIP_VALID
FAST_TRIP_HOLD
FAST_SHDN
SYS_FAULT
FINAL_SHDN
```

## Validation Result

The fast-fault case showed:

```text
Fault detection time       ≈ 5.139 ms
Shutdown time              ≈ 12.141 ms
Fault-to-shutdown latency  ≈ 7.0 ms
Target requirement         = 10 ms
Latency margin             ≈ 3 ms
```

## Design Note

The fast-fault path gives electrical faults a direct response route without waiting for slower thermal-state evolution.

---

# Subsystem 8: SAR ADC and Sampled Data Support

<img width="1448" height="1086" alt="image" src="https://github.com/user-attachments/assets/eb4435ba-8244-4958-8cb6-128f4ca59a50" />


## Function

The SAR ADC subsystem demonstrates sampled digital monitoring support for calibrated analog signals.

## Correct Role in This Project

The ADC block is included as a sampled-data support interface. It shows how analog signals from the front-end can be converted into digital values for future embedded or ASIC-style implementation.

The ADC is not the only decision-making path in the current protection system.

## SAR ADC Flow

```text
Calibrated Analog Input
        ↓
Sample-and-Hold
        ↓
Comparator
        ↓
DAC Trial
        ↓
SAR Bit Decision
        ↓
Digital Code
        ↓
ADC Valid Signal
```

## Important Signals

```text
V_SAMP_HOLD
DAC_OUT_SAR
VSAR_CODE
ADC_VALID_SAR
PH0_SAR ... PH7_SAR
```

## Validation Result

The voltage SAR validation showed:

```text
Sample-hold RMSE              ≈ 0.0211 V
Conversion residual RMSE      ≈ 0.0129 V
End-to-end RMSE               ≈ 0.1216 V
Source-to-code correlation    ≈ 0.970
Sample-to-valid delay         ≈ 3480 µs
```

## Design Note

The ADC block provides a bridge between the analog front-end and future digital BMS processing.

---

# LTspice Implementation

LTspice was used to build and validate the system behavior at circuit and behavioral level.

LTspice covered:

```text
Battery electrical model
Analog sensing paths
Adaptive calibration blocks
Thermal RC model
Thermal stress and dT/dt logic
Charge limit and derating logic
Aging-aware policy behavior
Fast-fault comparator and shutdown path
SAR ADC sampled-data support
```

Two important implementation versions were used:

| Draft   | Purpose                                                        |
| ------- | -------------------------------------------------------------- |
| Draft8  | Calibration and SAR ADC active testing                         |
| Draft11 | Long-run thermal, aging, protection, and fast-fault validation |

In the long-run protection simulations, ADC clocks were disabled for stability and simulation speed. The ADC was validated separately as a support subsystem.

---

# Python Analysis

Python was used as the validation and post-processing layer.

Python performed:

```text
LTspice data import
Signal extraction
Case-wise summary generation
Event timing extraction
Thermal derating effectiveness analysis
Runaway warning-window calculation
Aging policy comparison
Fast-fault latency measurement
Calibration error analysis
ADC validation
RLS internal resistance estimation
Report-ready plot generation
```

This allowed raw waveforms to be converted into measurable results such as timing windows, error reductions, current reductions, and latency values.

---

# Validation Cases

| Case   | Scenario           | Purpose                                            |
| ------ | ------------------ | -------------------------------------------------- |
| Case 1 | Baseline           | Stable operation without false protection          |
| Case 2 | Thermal derating   | Current reduction under rising thermal stress      |
| Case 3 | Runaway prevention | Precursor warning, derating, and shutdown sequence |
| Case 4 | Aging warning      | Moderate current restriction under aging           |
| Case 5 | Severe aging       | Stronger current restriction under degradation     |
| Case 6 | Fast fault         | Millisecond-level electrical shutdown              |

---

# Key Results

| Feature                         | Result     |
| ------------------------------- | ---------- |
| Calibration RMS error reduction | ≈ 78.6%    |
| Thermal drive reduction         | ≈ 31.8%    |
| dT/dt reduction                 | ≈ 67.1%    |
| Runaway warning window          | ≈ 1126.8 s |
| Runaway intervention window     | ≈ 356.5 s  |
| Total safety window             | ≈ 1483 s   |
| Aging warning current scale     | ≈ 0.70     |
| Severe aging current scale      | ≈ 0.251    |
| Fast-fault shutdown latency     | ≈ 7.0 ms   |
| Fast-fault target requirement   | 10 ms      |
| Fast-fault latency margin       | ≈ 3 ms     |
| SAR ADC source-code correlation | ≈ 0.970    |

---

# Result Images

Add result plots here after uploading them to the `results/` folder.

```markdown
![Calibration Result](results/calibration_result.png)

![Case 2 Thermal Derating](results/case2_thermal_derating.png)

![Case 3 Runaway Prevention](results/case3_runaway_prevention.png)

![Case 4 and Case 5 Aging Policy](results/case4_case5_aging_policy.png)

![Case 6 Fast Fault Latency](results/case6_fast_fault_latency.png)

![RLS Estimation](results/rls_estimation.png)

![ADC Validation](results/adc_validation.png)
```

---

# Tools Used

* LTspice
* Python
* NumPy
* Pandas
* Matplotlib
* Jupyter / Kaggle Notebook

---

# Final System Summary

```text
Measure battery signals
        ↓
Correct sensing errors
        ↓
Estimate internal resistance
        ↓
Calculate thermal and aging indicators
        ↓
Generate risk and health scores
        ↓
Apply health-aware current policy
        ↓
Derate under thermal stress
        ↓
Detect runaway precursor
        ↓
Shutdown under critical condition
        ↓
Use separate fast path for electrical faults
```

---

# Future Scope

* Hardware-in-loop validation
* Microcontroller integration
* Real battery dataset testing
* Multi-cell extension
* Cell balancing integration
* PCB implementation
* Real-time embedded RLS estimator
* Improved ADC integration
* ASIC-style analog front-end development
* Experimental validation using battery cycling data

---

# Note

This repository focuses on architecture, subsystem design, methodology, and validation results. Raw simulation/code files are not included in the main repository view to keep the project presentation clean and focused on system-level engineering.
