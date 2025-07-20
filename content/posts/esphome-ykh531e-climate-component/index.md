---
title: "ESPHome YK-H/531E Climate Component: IR Control for AC Units"
date: 2025-07-11T12:00:00-07:00
draft: false
tags:
  - esphome
  - home-automation
  - climate
  - ir-control
  - iot
---

I recently modernized an existing [custom ESPHome component][2] first develped by [@iverasp][3] for controlling AC units that use the YK-H/531E IR remote control protocol. This component allows you to integrate compatible air conditioners into your home automation system using infrared signals. I have been able to fully test it on ESP32 dev board as well as ESP8266-based Athom Tasmoda IR Controller using latest **ESPHome v2025.7.0**.

## Features

In addition to what was already working I added a few features to my [custom YKH531E component][1] it includes the following capabilities:

- **Temperature Range**: 16-32°C (60-90°F)
- **Operating Modes**: Auto, Cool, Dry, Fan, Heat (heat is untested)
- **Fan Speeds**: Low, Medium, High, Auto
- **Swing Control**: Vertical swing support (Corrected)
- **Temperature Units**: Support for both Celsius and Fahrenheit (Fahrenheit added!)

## Installation

Adding the component to your ESPHome configuration is straightforward:

```yaml
external_components:
  - source: github://smazurov/esphome-ykh531e
    components: [ykh531e]
```

## Configuration

Here's a basic configuration example:

```yaml
remote_transmitter:
  pin: GPIO14
  carrier_duty_percent: 50%

climate:
  - platform: ykh531e
    name: "AC Unit"
    supports_heat: false
    supports_cool: true
    use_fahrenheit: true
```

- **IR Transmitter**: Connect to any GPIO pin (GPIO14 in the example)
- **IR Receiver** (optional).

> Note: You need enough [RMT capacity][4] on your esp32 or channels if using arduino framework.

{{< img "IMG_5055.JPG" "800x" "ESP32 hardware setup with IR transmitter on breadboard" "270" >}}

The IR transmitter allows the ESP device to send commands to the AC unit, while the optional receiver can capture commands from the original remote to keep the ESPHome state synchronized.

### Optional automatic thermostat

If you want your esphome to control when the unit turns on and off, modify the above config:

```yaml
- platform: ykh531e
  name: "ir remote"
  supports_cool: true
  supports_heat: false
  receiver_id: r
  use_fahrenheit: true
  id: ir_ac
  internal: true # Hide from HA
# Create a thermostat that controls the IR unit
- platform: thermostat
  name: "Frigidaire AC"
  id: smart_thermostat
  sensor: ha_office_temperature_sensor

  min_cooling_off_time: 180s
  min_cooling_run_time: 180s
  min_idle_time: 60s

  cool_action:
    - climate.control:
        id: ir_ac
        mode: COOL
        target_temperature: 24°C
        swing_mode: VERTICAL

  idle_action:
    - climate.control:
        id: ir_ac
        mode: "OFF"

  default_preset: Home
  preset:
    - name: Home
      default_target_temperature_high: 24
      mode: "COOL"
    - name: Sleep
      default_target_temperature_high: 30
      mode: "OFF"
    - name: Away_Cool
      default_target_temperature_high: 22
      mode: "COOL"
```

> Note if `use_fahrenheit: true` and `ha_office_temperature_sensor` is reporting in fahrenheit, remember to convert the sensor value to celsius

## Compatibility

Currently tested with the Frigidaire FHPC102AC1, but should work with other AC units using the YK-H/531E remote. Untested list includes:

_FHPC082AB1_, _FHPC102AB1_, _FHPC132AB1_, _FHPH132AB1_, _FHPC082AB10_, _FHPC102AB10_, _FHPC132AB10_, _FHPH132AB10_, _FFPA0822U1_, _FFPA0822U10_, _FFPA0822U100_, _FFPA1022U1_, _FFPA1022U10_, _FFPA1022U100_, _FFPA1222U1_, _FFPA1222U10_, _FFPA1222U100_, _FFPA1422U1_, _FFPA1422U10_, _FFPA1422U100_, _FHPC082AC1_

The [source code][1] is available on GitHub for those interested in contributing or adapting it for similar protocols. If you have any issues or have further enhancements, feel free to contribute or open an issue.

## Prior Art

- [Reverse engineering the YK-H/531E AC remote control IR protocol][5]
- [IRremoteESP8266 - Electra][6]
- [@iverasp's PR to ESPHome][2]

[1]: https://github.com/smazurov/esphome-ykh531e "ESPHome YK-H/531E"
[2]: https://github.com/esphome/esphome/pull/6848
[3]: https://github.com/iverasp
[4]: https://esphome.io/components/remote_receiver.html#esp32-configuration-variables
[5]: https://blog.spans.fi/2024/04/16/reverse-engineering-the-yk-h531e-ac-remote-control-ir-protocol.html
[6]: https://github.com/crankyoldgit/IRremoteESP8266/blob/master/src/ir_Electra.h
