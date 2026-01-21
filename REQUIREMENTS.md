# Meshtastic-Controlled Distributed Audio Alert Station  
## System Requirements Specification (SRS)

**Version:** 1.1  
**Status:** Consolidated Draft  
**Intended Audience:** Engineering, Operations, Governance  
**Repository Fit:** GitHub-style Markdown

---

## 1. Purpose and Goals

This system provides **distributed, locally-stored audio alert capability** controlled via a **Meshtastic mesh network**, with strong authentication, human-in-the-loop safeguards, redundancy of command authority, transparent tamper and availability signaling, and safe testing procedures.

Primary goals:

- Offline, resilient operation
- Cryptographically authenticated command authority
- Explicit trust and governance model
- Safe degradation and recovery
- No reliance on Wi-Fi, Bluetooth, or IP networking
- Clear separation between **test audio** and **public alert audio**

---

## 2. Definitions and Roles

### 2.1 Roles

| Role | Description |
|---|---|
| **Station Owner** | Local administrator for a specific station |
| **Station Commander** | Authorized to issue commands to a specific station |
| **Region Commander** | Authorized to issue commands to multiple assigned stations |
| **Local Operator** | Limited local trigger authority |
| **System** | Autonomous station functions (tamper, heartbeat, etc.) |

Roles are **explicitly authorized**, **scoped**, and **independently auditable**.

---

## 3. Station Functional Requirements

### 3.1 Audio Capabilities

Each station **MUST**:

- Store all audio samples locally
- Support at least **100 `.wav` files**
- Support:
  - Single-sample playback at randomized intervals
  - Multi-sample mixing (1–100 concurrent voices)
  - Randomized timing and gain within bounded limits
- Output audio via:
  - Line-level output
  - Balanced XLR output (direct or via line driver)

Stations **MUST NOT** stream audio over RF or IP.

---

### 3.2 Control Interfaces

Stations **MUST** accept commands via:

- Meshtastic serial interface (USB or UART)
- Authenticated, signed command messages only

Stations **MUST NOT** expose:

- Wi-Fi services
- Bluetooth services
- IP-reachable control surfaces

---

## 4. Operating Modes

### 4.1 Modes

Stations **MUST** support:

- **AUTO (Autonomous)**  
  Authenticated commands execute immediately.

- **HITL (Human-in-the-Loop)**  
  Commands are staged and require physical acceptance.

- **TEST MODE**  
  Only test-only audio profiles may be executed.

Mode selection **MUST** be physical (switch, jumper, or equivalent).

---

### 4.2 Emergency Stop

Stations **MUST** have a hardware-level emergency stop that:

- Immediately mutes output
- Clears playback buffers
- Overrides all remote commands

---

## 5. Cryptographic Authentication

### 5.1 Command Authentication

All commands **MUST** be:

- Signed using public-key cryptography
- Verified before staging or execution
- Bound to a unique key identifier (`kid`)

**Recommended algorithm:** Ed25519

---

### 5.2 Anti-Replay Protection

Stations **MUST** enforce replay protection using:

- Per-key monotonic counters (preferred), or
- Nonce + bounded cache

Commands with stale or repeated counters **MUST** be rejected.

---

## 6. Authorization and Governance

### 6.1 Commander Redundancy Requirements

#### Station Level

Each station:

- **MUST** have **≥1 Station Commander**
- **SHOULD** have **3 Station Commanders**
- **MUST NOT** enter AUTO mode without meeting minimum

#### Region Level

Each region:

- **MUST** have **≥3 Region Commanders**

---

### 6.2 Operator Management

- Station Owners **MAY** add or remove Local Operators
- Operators:
  - Have explicitly limited capabilities
  - Cannot modify keys, roles, or policies
  - Cannot target other stations

Operators **DO NOT** satisfy commander minimums.

---

### 6.3 Authorization Enforcement

Every command **MUST** pass:

1. Signature verification  
2. Replay protection  
3. Role authorization  
4. Scope validation (station / region)  
5. Capability limits  
6. Mode gate (AUTO vs HITL vs TEST)

---

## 7. Targeting and Alarm Policy

### 7.1 Targeting

Commands **MUST** explicitly target:

- A single station, or
- A defined group of stations

Mass broadcast (`ALL`) **MUST** be disabled by default and policy-restricted.

---

### 7.2 Alarm Profiles

- All alarm and tone sequences **MUST** be predefined profiles
- Free-form acoustic encoding is **NOT PERMITTED**
- Profiles are:
  - Named
  - Versioned
  - Reviewable during HITL

---

### 7.3 Alarm Attribution

All alarm activations **MUST** generate signed mesh events including:

- Station ID
- Alarm profile
- Initiator `kid`
- Role
- Counter / timestamp

---

## 8. Signal-Channel → Meshtastic Bridge

When available:

- Stations or regions **SHOULD** support a **Signal-channel → Meshtastic bridge**
- Bridge **MUST**:
  - Inject signed Meshtastic commands only
  - Hold its own authorized identity
  - Replay traffic from Mesh to signal channel
  - Replay traffic from signal channel to Mesh
  - Respect all role, scope, and mode rules

Bridge failure **MUST NOT** affect mesh operation. Bridge is for convienence for those without local meshtastic nodes and/or for system wide monitoring.

---

## 9. Tamper Detection

### 9.1 Tamper Events

Stations **MUST** detect and report:

- Case opened
- Processing unit ↔ Meshtastic node disconnection
- Critical power interruption inconsistent with graceful shutdown

Detection **MUST** use hardware signals.

---

### 9.2 Tamper States

- `CLEAR`
- `TAMPERED`
- `CLEARED_PENDING_OWNER`
- `UNKNOWN`

All transitions **MUST** be logged and broadcast.

---

### 9.3 Tamper Response

Upon tamper detection:

- Station **MUST** broadcast a tamper event
- Station **MUST** enter HITL or LOCKDOWN
- Remote configuration changes are suspended

---

## 10. Tamper Clearance

### 10.1 Region Commander Action

A Region Commander:

- **MAY** issue a remote **TAMPER_CLEAR_REQUEST**
- Transitions station to `CLEARED_PENDING_OWNER`
- **MUST NOT** restore full operation alone

---

### 10.2 Station Owner Confirmation (Out-of-Band)

Final clearance **MUST** require:

- Station Owner confirmation
- Out-of-band verification (physical presence or separate channel)

Only then may the station return to `CLEAR`.

All clearance stages **MUST** be mesh-visible.

---

## 11. Duress / Do-Not-Believe State

Stations **MUST** support a duress state that:

- Broadcasts a **DO-NOT-BELIEVE** event
- Suspends trust in remote commands
- Requires onsite recovery to clear

Duress **MUST NOT** erase logs or state.

---

## 12. Heartbeat and Availability Monitoring

### 12.1 Heartbeat Emission

Each station **MUST** emit signed heartbeats:

- Interval: configurable (recommended 30–90s)
- Includes:
  - Station ID
  - Mode
  - Tamper state
  - Commander quorum status
  - Uptime

---

### 12.2 Distributed Offline Detection

A station is **CONFIRMED_OFFLINE** when:

- Heartbeats are missing beyond threshold, **and**
- A quorum (≥3) of independent stations corroborate absence

Offline events **MUST** be broadcast and attributable.

---

### 12.3 Recovery

Upon reappearance:

- A **REAPPEARED** event **SHOULD** be broadcast
- Station **SHOULD** resume in HITL
- Tamper or duress states **MUST NOT** auto-clear

---

## 13. Test & Calibration Requirements (Test-Only Audio)

### 13.1 Purpose

The system **MUST** support testing and tuning using **test-only audio samples** that are **distinct from public alert signals**.

Real alert tones **MUST NOT** be used for routine testing.

---

### 13.2 Test Sample Set

Each station **MUST** include a **Test Sample Set** that is:

- Stored locally
- Segregated from operational samples
- Acoustically non-alarming

Allowed examples:
- Pink noise
- Sine sweeps
- Clicks / metronome ticks
- Neutral spoken phrases (e.g., “system test”)

Prohibited:
- Any operational alert sample
- Any sample reasonably interpreted as a live alert

---

### 13.3 TEST MODE Safeguards

When in **TEST MODE**:

- Only test profiles may execute
- Alert profiles are hard-blocked
- Output gain **MUST** be capped
- All mesh events **MUST** be labeled `TEST`

Entering TEST MODE:
- **MUST** require Station Owner authorization
- **MUST NOT** occur automatically on reboot

---

### 13.4 Test Profiles

Test profiles **MUST** be predefined and versioned.

Example:
```text
TEST_PROFILE CAL_LEVEL_CHECK
SAMPLES=TEST_PINK
VOICES=1
GAIN=-24..-12 dB
DURATION=120s
