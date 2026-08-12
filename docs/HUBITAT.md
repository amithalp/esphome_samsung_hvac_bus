# Hubitat Integration

This document describes the tested integration of the Samsung VRF ESPHome entities into Hubitat Elevation.

## Architecture

The tested data/control path is:

```text
Samsung VRF
    ↓
ESPHome
    ↓
Home Assistant
    ↓
Home Assistant Device Bridge (HADB)
    ↓
Hubitat Elevation
```

## Climate Devices

The Home Assistant climate entities are imported into Hubitat using HADB.

A modified **HADB Generic Component Thermostat** driver is used to provide a logical `thermostatOperatingState`.

Because the Samsung ESPHome integration currently does not expose a true Home Assistant `hvac_action`, the Hubitat driver generates a logical operating state from the thermostat mode.

This provides useful Hubitat states such as:

- `idle`
- `cooling`
- `heating`
- `fan only`

This is primarily useful for Hubitat Dashboard thermostat tiles, which can display different states/colors for heating, cooling and idle.

> Note: This is a logical operating state and does not represent confirmed compressor operation.

## Filter Warning

Each Samsung indoor unit exposes a Home Assistant binary sensor:

```text
binary_sensor.samsung_vrf_<room>_filter_warning
```

The sensor originates from Samsung NASA message:

```text
0x4027
```

It is imported through HADB into Hubitat.

Typical states:

```text
OFF = Filter OK
ON  = Filter cleaning required
```

This was verified against a real Samsung filter-cleaning warning.

## Filter Reset

ESPHome exposes a reset switch for each indoor unit:

```text
switch.samsung_vrf_<room>_reset_filter
```

Rather than importing this directly into Hubitat as a switch, a Home Assistant Template Lock can be used.

The lock provides a confirmation step before a filter reset can accidentally be triggered from a Hubitat Dashboard.

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

The Template Lock is imported into Hubitat using the **HADB Generic Component Lock** driver.

The two-second ON period also creates a clear state transition for HADB:

```text
LOCKED
   ↓
UNLOCKED
   ↓
2 seconds
   ↓
LOCKED
```

## Reset Sequence

The resulting workflow is:

```text
Samsung Filter Warning
        ↓
ESPHome Filter Warning = ON
        ↓
Home Assistant Binary Sensor = ON
        ↓
HADB
        ↓
Hubitat Warning Sensor = ON
        ↓
User selects Filter Reset Lock
        ↓
Hubitat requests confirmation
        ↓
Unlock
        ↓
HA Template Lock
        ↓
ESPHome Reset Filter switch = ON
        ↓
2 second delay
        ↓
ESPHome Reset Filter switch = OFF
        ↓
HA Template Lock returns to LOCKED
        ↓
Samsung clears filter warning
        ↓
Filter Warning = OFF
```

## Verified Real-World Test

The complete workflow was tested with an actual Samsung indoor unit reporting a filter-cleaning warning.

### Before reset

- Samsung SmartThings reported that filter cleaning was required.
- Home Assistant filter-warning binary sensor reported a problem.
- Hubitat HADB warning sensor was ON.

The filter-reset lock was then activated from Hubitat.

Hubitat observed:

```text
Filter Reset status became unlocked
Filter Reset rawStatus became unlocked

2 seconds later:

Filter Reset status became locked
Filter Reset rawStatus became locked
```

ESPHome observed:

```text
Roni Reset Filter >> ON
Roni Reset Filter >> OFF
```

After Samsung processed the reset, Hubitat received:

```text
Filter Warning = OFF
```

Samsung SmartThings subsequently showed:

```text
Filter status: OK
Last cleaned: current date
```

This confirms the complete reset path:

```text
Hubitat
   ↓
HADB
   ↓
Home Assistant
   ↓
ESPHome
   ↓
Samsung HVAC
```

and the return path of the updated Samsung filter status back to Hubitat.

## Tested Indoor Units

The implementation was configured for six Samsung VRF indoor units:

- Living Room
- Kitchen
- Parents
- First Floor
- Maya
- Roni

Each unit exposes:

- Climate entity
- Power control
- Filter warning
- Filter reset

## HADB Filter Warning Device

The Home Assistant filter-warning binary sensor can be imported through HADB.

In the tested configuration, Hubitat represents it using the **HADB Generic Component Unknown Sensor**.

The important state is:

```text
ON  = Filter warning active
OFF = Filter OK
```

## HADB Filter Reset Device

The Home Assistant Template Lock is imported using:

```text
HADB Generic Component Lock
```

The normal/resting state is:

```text
LOCKED
```

Selecting **Unlock** from Hubitat initiates the filter-reset sequence.

After two seconds, the Home Assistant template returns the lock to:

```text
LOCKED
```

## Notes

The Hubitat integration is optional.

ESPHome and Home Assistant can operate the Samsung VRF system without Hubitat.

The Hubitat layer provides additional functionality such as:

- Hubitat Dashboard control
- Dashboard confirmation before filter reset
- Rule Machine automations
- Hubitat notifications
- Integration with other Hubitat devices and systems
