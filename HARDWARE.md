# Hardware Architecture & Build Guide

Meshtastic-Controlled Distributed Audio Alert Station  
Document: `HARDWARE.md`  
Version: 1.2  
Status: Reference Hardware Specification  
Primary Target: Raspberry Pi Zero 2 W  
Secondary Target: Raspberry Pi Zero / Zero W (constrained)

---

## 0. Scope, Responsibility, and Cross-References

This document defines the hardware architecture, physical interfaces, electrical expectations, indicators, and build/test guidance for the station.

### Explicitly Out of Scope (Defined Elsewhere)

The following are intentionally not defined here and are normative in:

- `SRS.md` — semantic meaning of modes, alarm policy, trust rules
- `COMMANDS.md` — command grammar and authorization
- `PROVISIONING.md` — key/sample update workflows
- Application firmware — state machines and policy enforcement

Hardware readers MUST NOT infer semantic behavior (e.g., what AUTO vs HITL “means”) beyond providing reliable signaling hooks (GPIO, LEDs, switches, interfaces) to the application layer.

---

## 1. Design Objectives

- Deterministic, appliance-style behavior
- No Wi-Fi / Bluetooth / IP services during normal operation
- Meshtastic serial control (UART preferred)
- Locally stored audio only
- Battery-backed USB-C power with USB data passthrough
- Hardware-enforced tamper signaling
- Clear, non-sensitive local indicators only
- All sensitive states (tamper, duress) are mesh-visible only

---

## 2. Core Compute Platform

### 2.1 Compute Module

- Raspberry Pi Zero 2 W (Preferred)
  - Quad-core Cortex-A53
  - Target load: ≤70% CPU under worst-case audio mix
- Raspberry Pi Zero / Zero W (Constrained)
  - Reduced voice count
  - Lower sample rate recommended

### 2.2 Disabled Interfaces (Hardware Expectation)

Outside Configuration Mode, hardware MUST support firmware disabling of:

- Wi-Fi
- Bluetooth

Hardware SHOULD support disabling (or not populating):

- HDMI (power/EMI reduction)

---

## 3. Power Architecture (Battery-Backed USB-C, Passthrough)

### 3.1 Interface Contract — USB-C Power

- Input voltage: 5.0 V nominal
- Operating range: 4.75–5.25 V
- Max continuous draw: 2.0 A (design for 2.5 A peak)
- Connector: USB-C receptacle (power + USB 2.0 data)

The USB-C port is both:

- The primary power input, and
- The sole USB data interface for gadget modes.

### 3.2 UPS / Power-Path Requirements (Guardrails)

The power-path / UPS module MUST:

- Support seamless switchover with ≤20 ms interruption
- Prevent Pi brownout during switchover
- Allow operation with battery removed (USB-powered only)
- Expose battery state (I²C or ADC) or allow external sensing

Minimum run-time (recommended):

- ≥60 minutes at nominal idle load
- ≥15 minutes under sustained audio output

Allowed battery chemistries:

- Li-ion 18650 (preferred)
- Li-Po pack (acceptable)

### 3.3 Brownout & Inrush Expectations

- Hardware must tolerate USB hot-plug without reset
- Inrush limiting or soft-start RECOMMENDED
- Bulk capacitance near Pi 5V rail RECOMMENDED

### 3.4 ESD / EMI Notes (Power & USB)

- USB-C D+/D− lines: ESD diodes recommended
- 5V input: TVS diode recommended
- Design target: IEC 61000-4-2 contact ±8 kV (best-effort)

---

## 4. Meshtastic Interface

### 4.1 Interface Contract — Meshtastic UART

- Logic level: 3.3 V TTL
- Baud: defined in app/SRS (hardware agnostic)
- Ground: shared ground required
- UART preferred over USB for determinism.

### 4.2 Link Supervision (Hardware Hook Requirement)

Hardware MUST expose a measurable condition allowing the app to infer link health, such as one of:

- Dedicated GPIO from Meshtastic node (preferred)
- UART activity watchdog (RX idle timeout)
- USB device presence (if USB serial used)

Exact thresholds are app-defined; hardware must make failure observable.

---

## 5. Audio Subsystem

### 5.1 Output Targets (Guardrails)

- Nominal output level: +4 dBu (balanced XLR)
- Maximum clean output: +14 dBu recommended
- Headroom: ≥10 dB above nominal before clipping
- Load impedance: ≥600 Ω
- Cable length assumption: up to 30 m balanced

### 5.2 Balanced Output Expectations

- Balanced driver IC (THAT1646, DRV134, or equivalent)
- Galvanic isolation via transformer OPTIONAL, not required
- Shield tied to chassis ground at connector recommended

### 5.3 Limiter & Gain Staging

- Hardware must allow ≥10 dB digital headroom
- Software limiter is mandatory (defined in app layer)
- Hardware must not hard-clip below +14 dBu equivalent

---

## 6. Physical Controls (DIP, Buttons, GPIO)

### 6.1 DIP Switch Truth Table (Hardware Mapping)

| DIP1 | DIP2 | Hardware Mode Signal                 |
|------|------|--------------------------------------|
| 0    | 0    | AUTO                                 |
| 0    | 1    | HITL                                 |
| 1    | 0    | TEST                                 |
| 1    | 1    | RESERVED / LOCKDOWN (app-defined)    |

Note: semantic behavior defined in `SRS.md`.

### 6.2 Configuration Enable

- DIP3 ON → Configuration Mode
- Hardware must expose this as a distinct GPIO state

### 6.3 Button Electrical Expectations

- Inputs pulled HIGH with internal or external pull-ups
- Active-low switches preferred
- Debounce: hardware RC or software (≤50 ms)

---

## 7. Display Subsystem

### Enforcement Responsibility

- Hardware MUST provide the display.
- Application firmware MUST enforce content restrictions.
- Hardware designers MUST NOT assume the display may show tamper or duress state.

---

## 8. Status LEDs

### 8.1 Priority Rules (Hardware Signaling)

When multiple states are active, LEDs resolve as:

1. ALARM (highest priority)
2. PROGRAMMING
3. PRE-LOAD
4. NORMAL (lowest)

Lower-priority LEDs may be suppressed when higher-priority states are active.

---

## 9. Tamper Detection

### 9.1 Electrical Expectations

- Normally-Closed (NC) wiring preferred
- Open circuit = tamper (fail-safe)
- Inputs pulled to known state (no floating pins)
- Debounce ≤100 ms
- Cable break or switch failure must resolve to tamper asserted.

---

## 10. Enclosure & Connectors

### Interface Contracts Summary

- USB-C: receptacle, power + USB 2.0 data
- XLR: male or female defined by deployment standard (document locally)
- Aux: 3.5 mm TRS optional

All external connectors should have:

- Strain relief
- ESD protection
- Chassis grounding where appropriate

---

## 11. Hardware Assembly — Bring-Up Checkpoints

Recommended smoke tests during build:

- Verify 5V rail (USB and battery) before Pi insertion
- Verify battery switchover under load
- Verify Meshtastic UART voltage levels
- Verify audio output with dummy load
- Verify DIP GPIO reads correctly
- Verify tamper input fails safe (open = tamper)

---

## 12. First-Boot & Test Dependencies

Recommended test fixtures:

- Known-good Meshtastic node
- Audio dummy load or powered speaker
- USB host PC for gadget mode
- Optional loopback cable for audio test

No specialized factory jig is required, but controlled test gear is recommended.

---

## 13. Appendix A — Configuration & Provisioning Mode (Hardware-Specific)

Appendix A is structurally unchanged from the prior revision and is scoped as hardware-visible behavior and interfaces for Configuration / Provisioning Mode.  
Semantics, workflows, file formats, signatures, and approvals remain defined in `PROVISIONING.md`.

---
