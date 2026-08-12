# ESPHome Samsung HVAC Integration

ESPHome external component for integrating Samsung HVAC systems with Home Assistant over the Samsung communication bus.

This repository is a fork of the original **ESPHome Samsung HVAC Bus** project and retains support for Samsung HVAC systems using the **NASA** and **NonNASA** protocols.

## amithalp Fork — v1.0.0

This fork adds and validates several features developed while integrating a real Samsung VRF installation with ESPHome, Home Assistant, and Hubitat.

### Added in v1.0.0

* Filter warning detection
* Filter reset control
* 0.5°C thermostat setpoint steps
* Outdoor unit temperature
* Outdoor instantaneous power
* Outdoor cumulative energy
* Outdoor current
* Outdoor voltage
* Outdoor ODU operating mode
* Outdoor Heat/Cool state

The filter functionality has been validated against an actual Samsung filter-cleaning warning and reset cycle.

After issuing the reset:

* The ESPHome filter warning cleared.
* Home Assistant reported the filter status as normal.
* Samsung SmartThings reported the filter as OK.
* SmartThings updated **Last Cleaned** to the date of the reset.

---

## Tested System

The additions in this fork have been tested with:

* **System:** Samsung VRF
* **Protocol:** NASA
* **Indoor units:** 6
* **ESP:** M5Stack Atom Lite
* **RS485 interface:** M5Stack Atomic RS485 Base
* **Connection:** Samsung F1/F2 bus
* **UART:** 9600 baud, 8E1
* **ESPHome:** 2026.5.1
* **Home Assistant:** 2026.5.1

Test installation addresses:

```text
Outdoor unit: 10.00.00

Indoor units:
20.00.00
20.00.01
20.00.02
20.00.03
20.00.04
20.00.05
```

**Important:** Samsung NASA addresses are installation-specific. Do not assume these addresses will match another system.

---

## Installation

Use this repository as an ESPHome external component.

For a stable configuration, reference the tested release:

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/amithalp/esphome_samsung_hvac_bus
      ref: v1.0.0
    components:
      - samsung_ac
    refresh: 0s
```

---

## Basic Hardware Configuration

The tested M5Stack Atom Lite configuration uses:

```yaml
esp32:
  board: m5stack-atom
  framework:
    type: esp-idf

uart:
  tx_pin: GPIO19
  rx_pin: GPIO22
  baud_rate: 9600
  parity: EVEN
  stop_bits: 1
```

The RS485 interface is connected to the Samsung **F1/F2 communication bus**.

Correct F1/F2 polarity is important. Incorrect polarity may interfere with Samsung communication and can produce communication errors on the existing Samsung controllers.

---

## Indoor Unit Example

A typical indoor unit can be configured as:

```yaml
samsung_ac:

  devices:
    - address: "20.00.00"

      climate:
        name: "Bedroom AC Unit"

      power:
        name: "Bedroom AC Power"

      filter_reset:
        name: "Bedroom Reset Filter"

      custom_sensor:
        - name: "Bedroom Filter Warning"
          message: 0x4027
          type: binary
          device_class: problem
          entity_category: diagnostic
```

This provides:

* Climate control
* Power control
* Filter warning
* Filter reset

---

## Filter Warning

Samsung NASA message:

```text
0x4027
```

was identified and validated as the indoor-unit filter warning in the tested VRF installation.

Recommended configuration:

```yaml
custom_sensor:
  - name: "Bedroom Filter Warning"
    message: 0x4027
    type: binary
    device_class: problem
    entity_category: diagnostic
```

Home Assistant then exposes the warning as a binary sensor.

Normal condition:

```text
OFF / No Problem
```

Filter cleaning required:

```text
ON / Problem
```

---

## Filter Reset

The fork adds:

```yaml
filter_reset:
  name: "Bedroom Reset Filter"
```

Activating this entity sends the Samsung filter-reset command.

### Real-world validation

The reset was tested while an indoor unit had an actual Samsung filter-cleaning warning.

The sequence was:

```text
Filter Warning = ON
        ↓
Filter Reset = ON
        ↓
Samsung reset command
        ↓
Filter Reset = OFF
        ↓
Filter Warning = OFF
```

Samsung SmartThings subsequently showed the filter as **OK** and updated its **Last Cleaned** date.

---

## 0.5°C Temperature Setpoints

The climate component in this fork uses:

```text
0.5°C
```

target-temperature increments.

This allows Home Assistant to represent Samsung setpoints such as:

```text
24.0°C
24.5°C
25.0°C
25.5°C
```

without rounding them to whole degrees.

---

## Outdoor Unit Telemetry

The outdoor NASA unit can expose additional telemetry.

Example:

```yaml
- address: "10.00.00"

  outdoor_temperature:
    name: "VRF Outdoor Temperature"
    accuracy_decimals: 1

  outdoor_instantaneous_power:
    name: "VRF Outdoor Power"

  outdoor_cumulative_energy:
    name: "VRF Outdoor Energy"

  outdoor_current:
    name: "VRF Outdoor Current"

  outdoor_voltage:
    name: "VRF Outdoor Voltage"

  outdoor_operation_odu_mode:
    name: "VRF Outdoor ODU Mode"

  outdoor_operation_heatcool:
    name: "VRF Outdoor Heat Cool State"
```

### Available telemetry

| Entity              | Purpose                            |
| ------------------- | ---------------------------------- |
| Outdoor Temperature | Outdoor-unit temperature reading   |
| Outdoor Power       | Instantaneous VRF system power     |
| Outdoor Energy      | Cumulative energy                  |
| Outdoor Current     | Outdoor-unit current               |
| Outdoor Voltage     | Outdoor-unit voltage               |
| ODU Mode            | Outdoor-unit operating mode        |
| Heat/Cool State     | Current system Heat/Cool direction |

---

## Example Complete Configuration

```yaml
esphome:
  name: samsung-vrf
  friendly_name: Samsung VRF

esp32:
  board: m5stack-atom
  framework:
    type: esp-idf

logger:
  level: WARN

api:
  encryption:
    key: !secret api_key

ota:
  - platform: esphome
    password: !secret ota_password

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

uart:
  tx_pin: GPIO19
  rx_pin: GPIO22
  baud_rate: 9600
  parity: EVEN
  stop_bits: 1

external_components:
  - source:
      type: git
      url: https://github.com/amithalp/esphome_samsung_hvac_bus
      ref: v1.0.0
    components:
      - samsung_ac
    refresh: 0s

samsung_ac:
  debug_log_messages_on_change: false
  debug_log_messages: false
  debug_log_undefined_messages: false
  debug_log_messages_raw: false

  devices:

    - address: "20.00.00"
      climate:
        name: "Bedroom AC Unit"
      power:
        name: "Bedroom AC Power"
      filter_reset:
        name: "Bedroom Reset Filter"
      custom_sensor:
        - name: "Bedroom Filter Warning"
          message: 0x4027
          type: binary
          device_class: problem
          entity_category: diagnostic

    - address: "20.00.01"
      climate:
        name: "Living Room AC Unit"
      power:
        name: "Living Room AC Power"
      filter_reset:
        name: "Living Room Reset Filter"
      custom_sensor:
        - name: "Living Room Filter Warning"
          message: 0x4027
          type: binary
          device_class: problem
          entity_category: diagnostic

    - address: "10.00.00"
      outdoor_temperature:
        name: "VRF Outdoor Temperature"
        accuracy_decimals: 1

      outdoor_instantaneous_power:
        name: "VRF Outdoor Power"

      outdoor_cumulative_energy:
        name: "VRF Outdoor Energy"

      outdoor_current:
        name: "VRF Outdoor Current"

      outdoor_voltage:
        name: "VRF Outdoor Voltage"

      outdoor_operation_odu_mode:
        name: "VRF Outdoor ODU Mode"

      outdoor_operation_heatcool:
        name: "VRF Outdoor Heat Cool State"
```

---

## Samsung VRF Heat/Cool Behavior

On the tested Samsung VRF Heat Pump system, indoor units share the system operating direction.

For example, when one indoor unit is operating in **Cool**, Samsung may dynamically remove **Heat** from the supported modes of the other indoor units.

This behavior was also observed through Samsung SmartThings and the wired Samsung controllers and is not caused by ESPHome.

---

## Room Temperature Sensor Selection

The tested Samsung wired controllers allow selection of the temperature sensor used for room-temperature control.

The installation was configured to use the **wired wall controller temperature sensor** rather than the indoor-unit sensor.

This may be particularly useful where an indoor unit is concealed in cabinetry or another location where its local temperature sensor does not accurately represent occupied-room temperature.

This is a Samsung installation/controller setting and is not controlled by this ESPHome component.

---

## Home Assistant

The climate entities are automatically exposed to Home Assistant through the ESPHome API.

The tested installation provides:

* Power
* HVAC mode
* Target temperature
* Current temperature
* Fan mode
* Filter warning
* Filter reset
* Outdoor telemetry

### Climate operating state

At the time of this release, the Samsung component does **not** expose a true Home Assistant `hvac_action` representing states such as:

```text
idle
cooling
heating
```

HVAC mode itself is available normally.

---

## Hubitat Integration

📖 **Detailed Hubitat setup:** [Hubitat Integration Guide](docs/HUBITAT.md)
The tested installation also integrates the Home Assistant entities with Hubitat using **Home Assistant Device Bridge (HADB)**.

This is optional and is not required for the ESPHome component itself.

The following have been successfully integrated:

* Samsung climate devices
* Filter warning sensors
* Filter reset controls
* Outdoor telemetry

### Filter Reset Confirmation

For Hubitat installations where confirmation is desired before resetting a filter, a Home Assistant Template Lock can wrap the ESPHome reset switch.

Example:

```yaml
template:
  - lock:
      - name: "Samsung VRF Bedroom Filter Reset"
        unique_id: bedroom_filter_reset_lock

        state: >
          {{ is_state('switch.samsung_vrf_bedroom_reset_filter', 'off') }}

        unlock:
          - action: switch.turn_on
            target:
              entity_id: switch.samsung_vrf_bedroom_reset_filter

          - delay:
              seconds: 2

          - action: switch.turn_off
            target:
              entity_id: switch.samsung_vrf_bedroom_reset_filter

        lock:
          - action: switch.turn_off
            target:
              entity_id: switch.samsung_vrf_bedroom_reset_filter
```

This creates the following sequence:

```text
Hubitat Lock tile
      ↓
Unlock confirmation
      ↓
HA Template Lock
      ↓
ESPHome Filter Reset ON
      ↓
2 seconds
      ↓
ESPHome Filter Reset OFF
      ↓
Lock returns to Locked
```

This workflow was tested successfully with an actual Samsung filter warning.

---

# Original Project

This repository is based on the **ESPHome Samsung HVAC Integration** project.

The original project provides an ESPHome component for integrating Samsung HVAC units with Home Assistant.

Samsung HVAC systems generally use two communication protocols:

* **NASA Protocol** — used by many newer systems.
* **NonNASA Protocol** — used by older systems.

The original project provides support for features including:

* Multiple indoor units
* Temperature and humidity monitoring
* Energy monitoring
* HVAC mode control
* Target-temperature control
* Error-code monitoring

## Original Documentation

* NASA Protocol Notes
  https://docs.samsung-hvac.aran.net.tr/wiki/nasa/samsung_nasa_protocol/

* Features Overview
  https://docs.samsung-hvac.aran.net.tr/wiki/Features-Overview/

* Compatibility
  https://docs.samsung-hvac.aran.net.tr/wiki/Compatibility/

* Installation Guide
  https://docs.samsung-hvac.aran.net.tr/wiki/Installation-Guide/

* Troubleshooting
  https://docs.samsung-hvac.aran.net.tr/wiki/Troubleshooting/

* FAQ
  https://docs.samsung-hvac.aran.net.tr/wiki/Frequently-Asked-Questions-(FAQ)/

* NASA vs NonNASA Protocols
  https://docs.samsung-hvac.aran.net.tr/wiki/NASA-vs-NonNASA-Protocols/

* Development Notes
  https://docs.samsung-hvac.aran.net.tr/wiki/Development/

---

## Credits

This fork builds upon the work of the original ESPHome Samsung HVAC community.

Special recognition goes to **Steve Wagner (@lanwin)** for founding and shaping the original project and to the contributors who have continued its development.

Original upstream repository:

https://github.com/omerfaruk-aran/esphome_samsung_hvac_bus

Contribution guidelines:

https://github.com/omerfaruk-aran/esphome_samsung_hvac_bus/blob/main/CONTRIBUTING.md
