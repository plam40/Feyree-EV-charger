# Feyree EV Charger - Complete TuyaMCU Integration Guide

> **Reverse-engineered documentation for Feyree F-OBZ2-AC-1P32 EV charger with Tasmota**

## Table of Contents
- [Overview](#overview)
- [Hardware Information](#hardware-information)
- [Complete dpID Reference](#complete-dpid-reference)
  - [Control dpIDs](#control-dpids)
  - [Monitoring dpIDs](#monitoring-dpids)
  - [Unknown/Conditional dpIDs](#unknownconditional-dpids)
  - [Missing dpIDs](#missing-dpids)
- [Critical Findings](#critical-findings)
- [Current Issues](#current-issues)
- [Tasmota Configuration](#tasmota-configuration)
- [Testing Protocol](#testing-protocol)
- [Contributing](#contributing)

---

## Overview

This document provides complete reverse-engineered documentation for integrating the **Feyree F-OBZ2-AC-1P32** single-phase EV charger with Tasmota firmware via TuyaMCU protocol.

### Key Features
- Single-phase 240V, 32A maximum
- Adjustable charging current: 6-32A
- Real-time power/voltage/current monitoring
- Temperature monitoring
- Session energy tracking
- Runtime timer

### Status
🚧 **Work in Progress** - Currently identifying and fixing integration issues

---

## Hardware Information

| Component | Details |
|-----------|---------|
| **Model** | F-OBZ2-AC-1P32 |
| **MCU** | GigaDevice GD32F303RCT6 |
| **Meter IC** | RN7302 |
| **Phases** | Single-phase (1P) |
| **Max Current** | 32A |
| **Voltage** | 240V AC |
| **WiFi Module** | WBR3 / CB3S (Tasmota compatible) |
| **Communication** | TuyaMCU Serial (9600 baud) |

### Physical Interface
- 8 screws (rubber sealed)
- Separate logic board and relay section
- External CT clamp (optional, not integrated with TuyaMCU)
- TFT display and buttons on logic board

---

## Complete dpID Reference

### Control dpIDs

#### dpID 115 - Current Limit (Command)

| Property | Value |
|----------|-------|
| **Direction** | Write (Command) |
| **Type** | value (0x02) |
| **Range** | 6-32 |
| **Units** | Amperes (A) |
| **Scale** | Direct value |
| **Tasmota** | fnId 21 (Dimmer) ⚠️ **PROBLEMATIC** |

**Description:** Sets the charging current limit.

**Issue:** Currently mapped to fnId 21 which triggers Power1 state changes, causing unwanted automatic charge start.

**Evidence:**
```
Command: TuyaSend2 115,10
Response: dpId 115 = 10 (immediate)
          dpId 125 = 10 (100-400ms delay)
```

---

#### dpID 124 - Charge Command

| Property | Value |
|----------|-------|
| **Direction** | Write (Command) |
| **Type** | enum (0x04) |
| **Values** | 0 = Charge<br>1 = Stop<br>2 = Ready<br>3 = Waiting<br>10 = Not Ready |
| **Units** | - |
| **Scale** | Direct value |
| **Tasmota** | Rule2 trigger |

**Description:** Starts or stops charging session.

**Behavior:** Command value does NOT persist. Reverts to `2` (Ready) approximately 1 second after command execution.

**Usage:**
```
Start: TuyaSend4 124,0
Stop:  TuyaSend4 124,1
```

**⚠️ Important:** Do NOT use dpID 124 value to determine charging state. Use dpID 101 instead.

---

### Monitoring dpIDs

#### dpID 101 - Device State

| Property | Value |
|----------|-------|
| **Direction** | Read (Status) |
| **Type** | enum (0x04) |
| **Values** | 0 = Charger Free<br>1 = Car Inserted ⚡<br>2 = Ready<br>3 = Charging<br>4 = Paused<br>5 = Charge Ended<br>6 = Fault |
| **Units** | - |
| **Scale** | Direct value |
| **Tasmota** | ✅ fnId 61 (mapped correctly) |

**Description:** **PRIMARY indicator** for charger and vehicle connection status.

**Use Cases:**
- Detect car connection: value = 1
- Confirm active charging: value = 3
- Detect faults: value = 6

---

#### dpID 102 - Phase A Voltage

| Property | Value |
|----------|-------|
| **Direction** | Read (Status) |
| **Type** | value (0x02) |
| **Range** | 0-3000 |
| **Units** | Volts (V) |
| **Scale** | ÷ 10 |
| **Tasmota** | ✅ fnId 33 (mapped correctly) |

**Description:** Main phase voltage measurement.

**Example:**
```
Raw: 0x0878 = 2168 decimal
Scaled: 2168 ÷ 10 = 216.8V
```

---

#### dpID 105 - Phase A Current

| Property | Value |
|----------|-------|
| **Direction** | Read (Status) |
| **Type** | value (0x02) |
| **Range** | 0-320 |
| **Units** | Amperes (A) |
| **Scale** | ÷ 10 |
| **Tasmota** | ⚠️ fnId 32 (÷100) **WRONG SCALE** |

**Description:** Actual current consumption (not limit).

**Issue:** Tasmota is using ÷100 scale, should be ÷10. Shows 10x too low.

**Example:**
```
Raw: 0x0043 = 67 decimal
Correct: 67 ÷ 10 = 6.7A
Tasmota: 67 ÷ 100 = 0.67A ❌
```

**Fix Required:** Change Tasmota function to use correct scale.

---

#### dpID 109 - Active Power

| Property | Value |
|----------|-------|
| **Direction** | Read (Status) |
| **Type** | value (0x02) |
| **Range** | 0-75000 |
| **Units** | Watts (W) |
| **Scale** | ÷ 10 |
| **Tasmota** | ✅ fnId 31 (mapped correctly) |

**Description:** Real-time power consumption.

**Example:**
```
Raw: 0x000E = 14 decimal
Scaled: 14 ÷ 10 = 1.4W
```

---

#### dpID 110 - Temperature

| Property | Value |
|----------|-------|
| **Direction** | Read (Status) |
| **Type** | value (0x02) |
| **Range** | 0-1000 |
| **Units** | Celsius (°C) |
| **Scale** | ÷ 10 |
| **Tasmota** | ❌ **NOT MAPPED** |

**Description:** Internal charger temperature.

**Example:**
```
Raw: 0x00E1 = 225 decimal
Scaled: 225 ÷ 10 = 22.5°C
```

**Required:** Map to Tasmota temperature sensor.

---

#### dpID 112 - Session Energy

| Property | Value |
|----------|-------|
| **Direction** | Read (Status) |
| **Type** | value (0x02) |
| **Range** | 0-999999 |
| **Units** | Kilowatt-hours (kWh) |
| **Scale** | ÷ 100 |
| **Tasmota** | ❌ **NOT MAPPED** |

**Description:** Energy consumed in current charging session.

**Example:**
```
Raw: 0x0006 = 6 decimal
Scaled: 6 ÷ 100 = 0.06 kWh
```

**Required:** Map to Tasmota energy counter.

---

#### dpID 120 - Runtime Timer

| Property | Value |
|----------|-------|
| **Direction** | Read (Status) |
| **Type** | string (0x03) |
| **Format** | "HH:MM:SS" |
| **Units** | Time |
| **Scale** | String |
| **Tasmota** | ❌ **NOT MAPPED** |

**Description:** Charging session duration timer. Updates every second.

**Example:**
```
Raw: "30303A32343A3533"
Decoded: "00:24:53"
Meaning: 24 minutes, 53 seconds
```

**Note:** Tasmota does not natively support string dpIDs well. Consider using MQTT rule to publish.

---

#### dpID 125 - Current Limit (Status)

| Property | Value |
|----------|-------|
| **Direction** | Read (Status) |
| **Type** | value (0x02) |
| **Range** | 6-32 |
| **Units** | Amperes (A) |
| **Scale** | Direct value |
| **Tasmota** | ❌ **NOT MAPPED** |

**Description:** Read-only feedback confirming the current limit actually set by MCU.

**Relationship:** Mirrors dpID 115 with 100-400ms delay.

**Evidence:**
```
Timeline:
16:47:11.972  dpId 115 = 10  (Command sent)
16:47:12.391  dpId 125 = 10  (Feedback received, 419ms later)
```

**Use Case:** Confirm current limit was accepted by hardware.

---

### Unknown/Conditional dpIDs

#### dpID 10 - Unknown (Heartbeat?)

| Property | Value |
|----------|-------|
| **Type** | raw (0x05) |
| **Value** | 0x00 (constant) |
| **Frequency** | ~500ms |
| **Status** | Low priority |

**Theory:** MCU heartbeat or keep-alive signal.

---

#### dpID 126 - Minimum Session Current

| Property | Value |
|----------|-------|
| **Type** | value (0x02) |
| **Value** | 6 |
| **Appearance** | **ONLY when charging starts** |
| **Status** | 🔍 Under investigation |

**Critical Finding:** This dpID only appears when a charging session is actively initiated, NOT when just adjusting current limit.

**Evidence:**
```
Test 1 (HA Slider → Starts Charging):
  - dpId 115 changed
  - dpId 126 = 6 appeared
  - Charging started

Test 2 (TuyaSend2 115 → No Charging):
  - dpId 115 changed
  - dpId 126 did NOT appear
  - Charging did NOT start
```

**Theory:** MCU safety mechanism confirming "I will charge at minimum 6A for this session."

---

#### dpID 127 - Unknown Counter

| Property | Value |
|----------|-------|
| **Type** | value (0x02) |
| **Values** | 5, 6, 8 (observed) |
| **Frequency** | Changes slowly (~2 sec) |
| **Status** | Low priority |

**Theory:** Unknown timer or state counter.

---

#### dpID 128 - Unknown

| Property | Value |
|----------|-------|
| **Type** | enum (0x04) |
| **Value** | 0 (constant) |
| **Status** | Low priority |

**Theory:** Disabled feature or placeholder.

---

### Missing dpIDs

These dpIDs are documented in forum sources but were **NOT observed** in testing:

| dpID | Function | Type | Expected Behavior | Status |
|------|----------|------|-------------------|--------|
| **14** | Work Mode | enum | 0=Now, 1=%, 2=Energy, 3=Schedule | Needs testing |
| **18** | Main Switch | bool | Global on/off | Needs testing |
| **103-104** | Phase B/C Voltage | value | Always 0 (single-phase) | Not needed |
| **106-107** | Phase B/C Current | value | Always 0 (single-phase) | Not needed |
| **113** | Device Max Current | enum | 16/32/60/80A (read-only) | Not broadcast |
| **114** | Alt Current Set | value | Alternative to dpId 115? | Unknown |
| **118-119** | Timer Strings | string | "HH:MM:SS" format | Not enabled |
| **121-122** | Timer Config | string | Complex timer data | Not enabled |
| **123** | RFID Status | bool | Always 0 (disabled) | Not relevant |

---

## Critical Findings

### 🔴 Issue 1: Current Adjustment Triggers Charging

**Problem:** Adjusting current via Home Assistant slider automatically starts charging.

**Root Cause:**
```
1. HA Slider moved
2. Tasmota fnId 21 (Dimmer) receives value
3. Dimmer change → Light state changes
4. Power1#state event fires
5. Rule2 detects: ON Power1#state=1
6. Rule2 executes: TuyaSend4 124,0
7. Charging starts (unwanted!)
```

**Impact:** Users cannot pre-set current limit without starting charge.

**Solution:** Decouple dpID 115 from fnId 21 (Dimmer/Light mechanism).

---

### 🔴 Issue 2: Current Reading 10x Wrong

**Problem:** Current sensor shows 0.67A when actually drawing 6.7A.

**Root Cause:** Tasmota fnId 32 using ÷100 scale, should be ÷10.

**Evidence:**
- Raw value: 0x43 = 67 decimal
- Forum documentation: "÷10 for current"
- Observed: Drawing ~6.7A (calculated from 216V × 6.7A ≈ 1.4kW)
- Tasmota shows: 0.67A ❌

**Solution:** Change scale from ÷100 to ÷10.

---

### 🟡 Finding 3: Command/Feedback Pair (115/125)

**Discovery:** dpID 115 and dpID 125 form a write/read pair.

**Behavior:**
- **dpID 115** = Command channel (user → MCU)
- **dpID 125** = Status channel (MCU → user)
- Delay: 95-400ms for acknowledgment

**Best Practice:** 
- Use dpID 115 to SET current
- Monitor dpID 125 for ACTUAL current limit

---

### 🟡 Finding 4: dpID 126 Conditional Appearance

**Discovery:** dpID 126 only appears when charging session starts, not during simple current adjustments.

**Implication:** Indicates active charging session vs. configuration change.

**Potential Use:** Could be used to detect when hardware actually begins charging.

---

## Current Issues

### Priority 1 - Broken Functionality

| # | Issue | Impact | Solution |
|---|-------|--------|----------|
| 1 | Current adjust starts charging | Cannot set current without charging | Remove fnId 21 mapping |
| 2 | Current reading 10x wrong | Shows 0.67A instead of 6.7A | Fix fnId 32 scale to ÷10 |
| 3 | Missing temperature | No thermal monitoring | Map dpId 110 → fnId 70 (temp) |
| 4 | Missing session energy | Cannot track kWh | Map dpId 112 → energy counter |
| 5 | No current status feedback | No confirmation of limit | Map dpId 125 → sensor |

### Priority 2 - Missing Features

| # | Feature | dpID | Benefit |
|---|---------|------|---------|
| 6 | Runtime timer | 120 | Show charging duration |
| 7 | Work mode control | 14 | Override schedules (if available) |

---

## Tasmota Configuration

### Current Configuration (BROKEN)
```
# Mappings (via TuyaMCU command)
TuyaMCU 11,99   # ❌ WRONG - dpId 99 doesn't exist
TuyaMCU 21,115  # ⚠️ PROBLEMATIC - triggers unwanted charging
TuyaMCU 32,105  # ⚠️ WRONG SCALE - using ÷100, should be ÷10
TuyaMCU 33,102  # ✅ Correct - voltage
TuyaMCU 31,109  # ✅ Correct - power
TuyaMCU 61,101  # ✅ Correct - device state

# Rules
Rule1 ON System#Boot DO Baudrate 9600 ENDON
Rule2 ON Power1#state=1 DO TuyaSend4 124,0 ENDON ON Power1#state=0 DO TuyaSend4 124,1 ENDON
Rule3 ON TuyaReceived#DpType4Id101 DO Status 10 ENDON

Rule1 1
Rule2 1
Rule3 1
```

### Proposed Fixed Configuration

**Coming in Step 2 - Configuration Implementation**

---

## Testing Protocol

### Prerequisites

1. **Enable verbose logging:**
```
   SetOption66 1
```

2. **Monitor MQTT messages:**
   Subscribe to: `tele/[DEVICE]/RESULT`

### Test Procedures

#### Test 1: Verify dpID Mappings
```bash
# Query all dpIDs
SetOption66 1
TuyaSend8
