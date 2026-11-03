# Server Room Fan

ESPHome controller for the server room's transfer fan. Firmware: [`serverroom-fan.yaml`](serverroom-fan.yaml).

ESPHome device `serverroom-fan`, friendly name "Server Room Fans". This yaml started as the original fork config (formerly `console-fan.yaml`).

Until the rename is flashed, the live device is still called `rack-fan` and its Home Assistant entities are `*.rack_fan_*` (listed below). After flashing, Home Assistant will see `serverroom-fan` as a new device. Expect new entity ids, and update any dashboards and automations that use the old ones.

## Role

The background below comes from `fan-hardware-handoff.md` (2026-10-16).

Two rooms are separated by a door:

- a **rack room**, holding the rack
- an **AC room**, cooled by a Faikin/Faikout-controlled split unit (`faikout-sottotetto`)

This is a **transfer fan**: it moves air from the rack room into the AC room. It is not a fan inside the rack and not the AC's fan.

The control split, as decided by the owner:
- the **fans** regulate the **rack room**
- the **AC** regulates the **AC room**
- rack temperature is only an **emergency escalation** input for the AC, never its normal input

The fan can never get the rack room below the AC room's temperature. If the AC is off or the AC room is warm, more fan does not help.

## Hardware

| | |
|---|---|
| Board | NodeMCU-32S (plain ESP32), WiFi |
| ESPHome | 2025.4.2 |
| Sensor | SHT21 (`htu21d` platform), I2C SDA GPIO21, SCL GPIO22, in the AC room |
| Fan PWM | GPIO13, 25 kHz, one output shared by fan 1 and fan 2 |
| Tach | fan 1 GPIO14, fan 2 GPIO26, 2 pulses/rev |
| Fans | 12 V 4-pin PWM PC fans (model not recorded here) |

Wiring follows the [upstream README](README.md): the 12 V and 3.3 V grounds must be joined.

## Entities (current live device, before the rename)

```
climate.rack_fan_thermostat     PID, modes off/cool, 20-50 °C
fan.manual_fan_speed            manual override
sensor.rack_fan_output_value    PID output
sensor.rack_fan_error_value     setpoint error, °C
sensor.rack_fan_p_term / _i_term / _d_term
sensor.rack_fan_fan_speed_1     tach 1, RPM
sensor.rack_fan_fan_speed_2     tach 2, RPM
sensor.rack_fan_is_in_deadband
switch.rack_fan_esp32_restart
```

The PID tuning (kp, ki, kd, deadband thresholds, deadband ki multiplier) is exposed as `number` entities.

Also exposed:
- temperature, humidity, dew point, VPD
- uptime, WiFi strength, IP address, ESPHome version
- a PID autotune button

The device restarts daily at 00:30.

## State on 2026-10-16

| reading | value |
|---|---|
| SHT21, AC room | 28.5 °C |
| PID setpoint | 25.9 °C |
| PID output | 164.9 / 255 ≈ 65 % |
| `i_term` | 100.0, pinned at its clamp |
| tach 1 / tach 2 | 1878 / **0** RPM |

- **The loop is saturated:** the integral is clamped and the error holds steady at about 2.5 °C. The likely cause is a warm sink (the AC room itself was 28.5 °C) rather than fan capacity.
- **Tach 2 reads 0:** the second fan is either missing, dead, or its tach is not connected.

## Ideas on file

- **Second temperature sensor:**
  - Put a second sensor in the rack room and run the fan only when the AC room is cooler than the rack room, since a transfer fan imports heat when the sign of ΔT is wrong.
  - Implement it as an interlock on the PID output, not by retargeting the PID.
- **ESPHome upgrade:** 2025.4.2 is old.
