# Feyree EV Charger - Tasmota & Home Assistant Integration

### Home Assistant Dashboard, Tasmota Web Interface, Feyree charger

<p align="left">
  <img src="ha-dashboard.png" alt="Home Assistant Dashboard" height="400">

  <img src="tasmota-interface.png" alt="Tasmota Web Interface" height="400">

  <img src="Feyree-7.6kw-32A-single-phase.png" alt="Feyree EV Charger" height="400">
</p>

## This is work in progress

A complete guide to integrating the Feyree 1P EV charger with Tasmota firmware on an ESP8266 module and Home Assistant for smart charging control and monitoring.

---

## CRITICAL SAFETY WARNING

```
+-----------------------------------------------------------------------+
|                                                                       |
|   DANGER - HIGH VOLTAGE - RISK OF DEATH OR SERIOUS INJURY             |
|                                                                       |
|   NEVER OPEN THE CHARGER OR WORK ON INTERNAL COMPONENTS               |
|   WHILE CONNECTED TO MAINS POWER (230V/110V)                          |
|                                                                       |
|   - ALWAYS DISCONNECT FROM MAINS BEFORE OPENING                       |
|   - VERIFY POWER IS OFF WITH A VOLTAGE TESTER                         |
|   - WAIT FOR CAPACITORS TO DISCHARGE (5+ MINUTES)                     |
|   - ONLY QUALIFIED ELECTRICIANS SHOULD MODIFY THIS EQUIPMENT          |
|                                                                       |
|   Failure to follow these precautions can result in:                   |
|   - Electric shock leading to death                                    |
|   - Severe burns                                                       |
|   - Fire hazard                                                        |
|   - Property damage                                                    |
|   - Voiding of all warranties and insurance                            |
|                                                                       |
+-----------------------------------------------------------------------+
```

**By proceeding with this modification, you acknowledge that:**
- You understand the risks of working with high-voltage equipment
- You are qualified to perform electrical work in your jurisdiction
- You accept full responsibility for any damage, injury, or death
- The authors of this guide assume NO LIABILITY whatsoever

---

## Quick How-To

A condensed checklist for experienced users. See the detailed sections below for full instructions.

1. **Replace the WiFi module** - Swap the stock WBR3 module for an ESP8266 per the [Elektroda.com guide](https://www.elektroda.com/rtvforum/topic4085036.html)
2. **Flash Tasmota** - Flash `tasmota-scripting.bin` (v14.x) to the ESP8266 via serial
3. **Connect to Tasmota AP** - Join the `tasmota-XXXXXX` WiFi network, configure your WiFi credentials
4. **Apply the device template** - In Tasmota go to *Configuration > Configure Other*, paste:
   ```json
   {"NAME":"Feyree Charger","GPIO":[0,0,0,0,0,0,0,0,0,0,0,0,0,0],"FLAG":0,"BASE":54}
   ```
5. **Disable power button** - Run `SetOption114 0` in the Tasmota console
6. **Install the Tasmota script** - Paste [`config/tasmota-script.txt`](config/tasmota-script.txt) into *Tools > Edit Script*, enable, and save
7. **Set up Home Assistant** - Add the config files to your HA `configuration.yaml`:
   - [`config/ha-light-entity.yaml`](config/ha-light-entity.yaml) - charger control via light entity
   - [`config/ha-mqtt-sensors.yaml`](config/ha-mqtt-sensors.yaml) - MQTT sensors
   - [`config/ha-template-sensor.yaml`](config/ha-template-sensor.yaml) - server time helper
   - [`config/ha-input-helpers.yaml`](config/ha-input-helpers.yaml) - schedule & keep-alive toggles
8. **Add automations** - Add [`config/ha-automations.yaml`](config/ha-automations.yaml) to your `automations.yaml`
9. **Add the dashboard card** - Paste [`config/ha-dashboard.yaml`](config/ha-dashboard.yaml) into a manual card (requires [custom:button-card](https://github.com/custom-cards/button-card))
10. **Restart Home Assistant** and verify entities appear

---

## Table of Contents

- [Overview](#overview)
- [Quick How-To](#quick-how-to)
- [Features](#features)
- [Hardware Requirements](#hardware-requirements)
- [Software Requirements](#software-requirements)
- [Installation](#installation)
  - [1. Flash Tasmota to ESP8266](#1-flash-tasmota-to-esp8266)
  - [2. Configure Tasmota](#2-configure-tasmota)
  - [3. Install Tasmota Script](#3-install-tasmota-script)
  - [4. Configure Home Assistant](#4-configure-home-assistant)
  - [5. Setup Automations](#5-setup-automations)
  - [6. HA Dashboard](#6-ha-dashboard)
- [Usage](#usage)
- [MQTT Topics](#mqtt-topics)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)

## Overview

This project enables full local control of the Feyree EV charger by replacing the cloud-dependent Tuya firmware with Tasmota on an ESP8266 module. It provides real-time monitoring, current adjustment (6-32A), and scheduled charging through Home Assistant.

**Prerequisites (Required Before Starting):**
- Completed ESP8266 module installation (replacing the stock WBR3) following the [Elektroda.com guide](https://www.elektroda.com/rtvforum/topic4085036.html)
- Knowledge of ESP8266 flashing procedures
- Working MQTT broker integrated with Home Assistant
- Basic Home Assistant YAML configuration skills
- Understanding of high-voltage electrical safety procedures
- Qualified to work with electrical equipment in your jurisdiction

**Safety First:** All configuration and flashing work is done with the charger **COMPLETELY DISCONNECTED** from mains power. See the safety warnings above before proceeding.

## Features

- **Real-time Monitoring**: Voltage, current, power, energy, and temperature
- **Current Control**: Adjust charging current from 6A to 32A
- **Charging States**: Free, Plugged in, Charging, Paused, Charged, Fault
- **Fault Detection**: Undervoltage, overvoltage, overcurrent, PE fault, temperature, CP signal, leakage, emergency stop
- **Scheduled Charging**: Start/stop charging at specific times
- **Keep-Alive Automation**: Prevents charger timeout during long charging sessions
- **Home Assistant Integration**: Full control through HA dashboard
- **MQTT Communication**: Local network control, no cloud dependency

## Hardware Requirements

- **Feyree EV Charger** (Tuya-based, 1-phase) with the **stock WBR3 WiFi module already replaced by an ESP8266**

> **PREREQUISITE**: This guide assumes you have already replaced the stock WBR3 module with an ESP8266 following the [Elektroda.com guide](https://www.elektroda.com/rtvforum/topic4085036.html). If you haven't done this yet, **STOP HERE** and complete that hardware swap first.

> **REMINDER**: ALL work on the charger must be done with it **COMPLETELY DISCONNECTED** from mains power. Verify disconnection with a voltage tester.

## Software Requirements

- **Tasmota v14.x** with scripting support from the [Tasmota Specials Repository](https://github.com/tasmota/install)
  - Standard Tasmota builds do not include full scripting support
  - Download the **tasmota-scripting.bin** variant
- **Home Assistant** (any recent version with MQTT support)
- **MQTT Broker** (Mosquitto or similar) - must be already configured and running
- **ESP8266 Flashing Tools** - ability to flash ESP modules via serial connection

**Assumed Prerequisites:**
- You know how to flash ESP8266 modules (many tutorials available online)
- You have a working MQTT broker integrated with Home Assistant
- You understand basic Home Assistant configuration (YAML editing)
- You've completed the [Elektroda.com ESP8266 replacement guide](https://www.elektroda.com/rtvforum/topic4085036.html)

## Installation

### 1. Flash Tasmota to ESP8266

**Flash Tasmota v14.x with scripting support** to the ESP8266 module using your preferred method:
- Use **tasmota-scripting.bin** from the Tasmota specials repository
- Standard ESP8266 flashing procedures apply (GPIO0 to GND for flash mode, 9600 or 115200 baud)
- Many tutorials are available online for ESP8266 flashing if needed

**After Flashing:**
1. Disconnect USB-to-Serial adapter
2. **VERIFY** mains power is still disconnected
3. Reassemble the charger (if module was removed)
4. **ONLY THEN** reconnect to mains power
5. The ESP8266 will boot into Tasmota and create a WiFi access point

### 2. Configure Tasmota

> **SAFETY CHECKPOINT BEFORE POWER-UP:**
> 1. Ensure the charger enclosure is **FULLY REASSEMBLED**
> 2. Verify all **connections are secure** and no tools/wires are left inside
> 3. **ONLY NOW** reconnect to mains power to configure Tasmota
> 4. If you need to access the module again, **DISCONNECT MAINS FIRST**

1. Connect to the Tasmota WiFi AP (e.g., `tasmota-XXXXXX`)
2. Configure your WiFi credentials
3. Access the Tasmota web interface (check your router for the IP)
4. Navigate to **Configuration** > **Configure Other**
5. Paste and check **Template**:
   ```json
   {"NAME":"Feyree Charger","GPIO":[0,0,0,0,0,0,0,0,0,0,0,0,0,0],"FLAG":0,"BASE":54}
   ```
6. Save and reboot

### 3. Install Tasmota Script

1. In the Tasmota console, disable option 114:
   ```
   SetOption114 0
   ```
   *(This hides the power button as it's not used)*

2. Navigate to **Tools** > **Edit Script**

3. Paste the complete script from [`config/tasmota-script.txt`](config/tasmota-script.txt)

4. Enable the script checkbox and click **Save**

#### Script Explanation

| Section | Description |
|---------|-------------|
| `>D` | Variable declarations (current, voltage, power, temp, etc.) |
| `>B` | Boot section - sets baudrate and initializes Tuya MCU |
| `>E` | Event section - processes Tuya datapoints and fault codes |
| `>S` | Sensor section - publishes MQTT data and updates web interface |
| `>W` | Web interface - displays current values and provides controls |

#### Tuya Datapoint Mapping

| DP ID | Type | Description | Range |
|-------|------|-------------|-------|
| 101 | Enum | Charging state | 0-6 |
| 102 | Integer | Voltage | 0-2500 (x0.1V) |
| 105 | Integer | Current | 0-320 (x0.1A) |
| 109 | Integer | Power | 0-7000 (x0.1kW) |
| 110 | Integer | Temperature | 0-100 (x0.1C) |
| 112 | Integer | Session Energy | 0-100000 (x0.1kWh) |
| 115 | Integer | Current Preset | 6-32 (A) |
| 124 | Boolean | Charge Enable | 0=On, 1=Off |
| 125 | Integer | Current Max | 6-32 (A) |
| 10 | Raw | Fault Code | Hex value |

### 4. Configure Home Assistant

> **Assumption**: You already have a working MQTT broker configured and integrated with Home Assistant.

Add the following configuration blocks to your `configuration.yaml`. Each file is provided separately in the `config/` directory for easy copy-paste.

#### Light Entity (Charging Control)

Add the contents of [`config/ha-light-entity.yaml`](config/ha-light-entity.yaml) to `configuration.yaml`.

> **Note**: The charger is exposed as a `light` entity to leverage HA's brightness slider for current control (6-32A).

#### MQTT Sensors

Add the contents of [`config/ha-mqtt-sensors.yaml`](config/ha-mqtt-sensors.yaml) to `configuration.yaml`.

#### Template Sensor (Helper)

Add the contents of [`config/ha-template-sensor.yaml`](config/ha-template-sensor.yaml) to `configuration.yaml`.

#### Input Helpers

Add the contents of [`config/ha-input-helpers.yaml`](config/ha-input-helpers.yaml) to `configuration.yaml`.

### 5. Setup Automations

Add the contents of [`config/ha-automations.yaml`](config/ha-automations.yaml) to your `automations.yaml`.

#### Automation Descriptions

| Automation | Purpose |
|------------|---------|
| **EV Charger Keep Alive** | Prevents charger timeout during long sessions by sending periodic pulses |

### 6. HA Dashboard

The dashboard card uses [custom:button-card](https://github.com/custom-cards/button-card) (install via HACS).

Add the contents of [`config/ha-dashboard.yaml`](config/ha-dashboard.yaml) as a manual card in your Home Assistant dashboard.

## Usage

### Basic Control

1. **Start Charging**: Turn on `light.108_ev_charger` in Home Assistant
2. **Stop Charging**: Turn off `light.108_ev_charger`
3. **Adjust Current**: Use the brightness slider (6A = dim, 32A = bright)

### Scheduled Charging

1. Enable `input_boolean.ev_charging_schedule_enabled`
2. Set `input_datetime.ev_charge_start` (e.g., 01:00 for cheap electricity)
3. Set `input_datetime.ev_charge_stop` (e.g., 06:00)
4. Charging will automatically start/stop at configured times

### Keep-Alive Feature

The keep-alive automation prevents the charger from timing out during long charging sessions:

- Runs every 20 minutes
- Briefly stops and restarts charging with a minimal current pulse
- Restores previous charging state and current setting
- Enable via `input_boolean.ev_charger_keep_alive`

## MQTT Topics

### Command Topics (cmnd/108/)

| Topic | Payload | Description |
|-------|---------|-------------|
| `cmnd/108/TuyaSend2` | `115,<6-32>` | Set charging current (A) |
| `cmnd/108/TuyaSend4` | `124,0` | Enable charging |
| `cmnd/108/TuyaSend4` | `124,1` | Disable charging |

### Status Topic (stat/108/data)

Published JSON format:
```json
{
  "state": "Charging",
  "voltage": 230.5,
  "current": 16.0,
  "power": 3.7,
  "energy": 12.5,
  "temp": 35.2,
  "preset": 16,
  "fault": "OK"
}
```

## Troubleshooting

### Charger Not Responding

1. Verify correct firmware (`tasmota-scripting.bin`) was flashed
2. Check Tasmota console for Tuya MCU communication:
   ```
   TuyaMCU 0,4
   TuyaSend0
   ```
   You should see Tuya datapoint responses
3. Verify baudrate is set to 9600 in boot section of script
4. Ensure MQTT is configured in Tasmota and topic is set to `108`
5. Check serial connection quality (try reflashing if needed)

### Home Assistant Entities Not Appearing

1. Check `configuration.yaml` for syntax errors
2. Validate YAML: **Developer Tools** > **YAML**
3. Restart Home Assistant
4. Check logs: **Settings** > **System** > **Logs**

### Charging Not Starting

1. Verify charger state: Check `sensor.108_ev_charger_state`
2. Check fault sensor: `sensor.108_ev_charger_fault`
3. Ensure vehicle is properly plugged in
4. Check Tasmota web interface to verify charger is responding
5. Verify topic `108` matches in both Tasmota and Home Assistant configs

### Script Not Loading

1. Ensure you flashed **tasmota-scripting.bin** (not standard tasmota.bin)
2. Verify Tasmota v14.x or newer is installed
3. Check script syntax in Tasmota console for errors
4. Verify `SetOption114 0` is set
5. Try clearing and re-pasting the script
6. Check Tasmota logs: **Console** > look for script errors

### Brightness/Current Mapping Issues

The brightness (0-255) maps to current (6-32A):
- Brightness 0 = 6A
- Brightness 128 = 19A
- Brightness 255 = 32A

Formula: `Current = (Brightness / 255 * 26) + 6`

## Config Files Reference

All configuration files are in the [`config/`](config/) directory:

| File | Description |
|------|-------------|
| [`tasmota-script.txt`](config/tasmota-script.txt) | Tasmota script for TuyaMCU communication |
| [`ha-light-entity.yaml`](config/ha-light-entity.yaml) | HA light template for charger control |
| [`ha-mqtt-sensors.yaml`](config/ha-mqtt-sensors.yaml) | HA MQTT sensor definitions |
| [`ha-template-sensor.yaml`](config/ha-template-sensor.yaml) | HA server time template sensor |
| [`ha-input-helpers.yaml`](config/ha-input-helpers.yaml) | HA input booleans and datetime helpers |
| [`ha-automations.yaml`](config/ha-automations.yaml) | HA keep-alive automation |
| [`ha-dashboard.yaml`](config/ha-dashboard.yaml) | HA dashboard card (custom:button-card) |

## Contributing

This is a personal hobby project shared as-is with **NO ACTIVE MAINTENANCE**.

- Pull requests may not be reviewed or merged
- Issues may not be responded to
- Feature requests will not be implemented

**However**, you are free to:
- Fork this repository and modify it for your needs
- Share your improvements with the community
- Use this as a starting point for your own projects

## License

This is a personal hobby project shared for educational purposes. Use at your own risk.

**No license is explicitly granted.** The code and documentation are provided "as-is" for reference only. Please ensure compliance with local electrical codes and safety regulations when working with EV charging equipment.

If you use or modify this project, you do so entirely at your own risk and responsibility.

## Disclaimer

**CRITICAL SAFETY REQUIREMENTS**

This project involves modifying high-voltage electrical equipment (EV charging station operating at 230V or higher). **This is EXTREMELY DANGEROUS** if proper precautions are not followed.

**MANDATORY SAFETY RULES:**

1. **NEVER** work on the charger internals while connected to mains power
2. **ALWAYS** disconnect from mains and verify with a voltage tester before opening
3. **ALWAYS** wait at least 5 minutes after disconnection for capacitors to discharge
4. **NEVER** touch internal components until verified safe with a voltage tester
5. **ONLY** qualified electricians should perform this modification
6. **ALWAYS** follow local electrical codes and safety regulations
7. **ENSURE** proper grounding and RCD/GFCI protection is installed

**RISKS INCLUDE BUT ARE NOT LIMITED TO:**
- Electric shock causing death
- Fire hazard from improper modifications
- Equipment damage and property loss
- Damage to your vehicle's battery and charging system
- Voiding of all warranties (charger, vehicle, home insurance)
- Legal liability for non-compliant electrical work

**PREREQUISITE KNOWLEDGE:**
- You must be familiar with the [Elektroda.com ESP8266 replacement guide](https://www.elektroda.com/rtvforum/topic4085036.html)
- You must understand ESP8266 module pinouts and flashing procedures
- You must be competent in electrical safety and high-voltage systems

**LEGAL DISCLAIMER:**

This guide is provided **AS-IS** for **EDUCATIONAL PURPOSES ONLY**.

**This is a personal hobby project created in free time:**
- **NO SUPPORT** of any kind is provided
- **NO WARRANTY** or guarantee of functionality
- **NO OBLIGATION** to respond to issues, questions, or requests
- **NO LIABILITY** for any consequences of using this information

The authors, contributors, and publishers:
- Make NO warranties or guarantees of any kind
- Accept NO liability for damage, injury, death, or loss
- Are NOT responsible for code compliance or safety violations
- Do NOT endorse circumventing safety features or manufacturer protections
- Provide this information purely for educational sharing

**By using this information, you agree that:**
- You are solely responsible for all risks and outcomes
- You have the necessary qualifications and legal right to perform electrical work
- You will comply with all local codes, regulations, and safety standards
- You release all authors and contributors from any and all liability
- You understand this is a hobby project with no support commitment

**WHEN IN DOUBT, HIRE A LICENSED ELECTRICIAN**

## Acknowledgments

- [Tasmota Project](https://github.com/arendst/Tasmota) for the excellent firmware
- [Elektroda.com Forum Community](https://www.elektroda.com/rtvforum/topic4085036.html) for the ESP8266 module replacement guide
- Home Assistant community for MQTT integration support
- Feyree for manufacturing accessible EV charging hardware

## Resources

This is a hobby project with **NO SUPPORT PROVIDED**. However, you may find these resources helpful:

- **Tasmota Documentation**: [https://tasmota.github.io/docs/](https://tasmota.github.io/docs/) - For general Tasmota questions
- **Home Assistant Docs**: [https://www.home-assistant.io/docs/](https://www.home-assistant.io/docs/) - For HA configuration help
- **Elektroda.com Forum**: [ESP8266 Replacement Thread](https://www.elektroda.com/rtvforum/topic4085036.html) - For hardware modification questions

**Note**: Do not expect responses to issues, pull requests, or questions. This code is shared as-is for educational purposes only.

---

**Made for smarter EV charging**
