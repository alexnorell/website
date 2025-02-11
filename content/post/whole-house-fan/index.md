---
title: "Smart Whole House Fan"
date: 2024-07-16T12:00:00-07:00
categories:
  - house
tags:
  - home automation
  - zwave
  - home assistant
draft: false
description: Install a whole house fan from QuietCool and make it smart using Home Assistant and ZWave.
---

We added a whole house fan to our house to bring in fresh air and aid in cooling down the house in the evening. The fan is fully controllable with Home Assistant using a couple ZWave relays.

## System Design

We went with a slightly oversized unit than our square footage and climate zone need. The unit we chose was the Quiet Cool QC ES-4700 which provides two speeds, high at 4,195 CFM and low at 2,304 CFM. The fan has two control lines that determine speed. Normally, these lines are hooked up to Quiet Cool's own smart fan control system to control the speed or a timer switch and control switch.

{{< img src="images/official_wiring.webp" alt="Quiet Cool Wiring Diagram" >}}

I prefer to keep things controlled locally and with ZWave or Zigbee if possible. I already had a ZWave multi relay in the attic for controlling an exterior light and I had the necessary relays left to control the system. This also means I didn't need to run additional wires anywhere in my house. I only needed to run two 12-2 wires from the junction box housing the relay to the fan.

{{< img src="images/wiring.webp" alt="Wiring Diagram" >}}

### Parts list

- [Quiet Cool QC ES-4700](https://norell.link/qcfan)
- [Zooz ZEN16 Multi Relay](https://norell.link/zen16)
- 12-2 wire

### Tools required

- Impact Driver
- Screwdriver
- Wire cutter and strippers
- Drywall saw

### Additional tools

These are tools that I needed to cut out the floor and add some framing to my attic to mount the fan.

- Reciprocating saw
- Jigsaw
- Drywall Rotary Cutter

## Installation

Installing a whole home fan is exactly as easy as the manufacturer claims in their marketing.

Here are the steps:

1. Identify a place to put the fan
2. Cut a hole in the ceiling
3. Install the damper in the ceiling
4. Hook the tube up to the damper
5. Hang the fan from a rafter
6. Hook the tube up to the fan
7. Wire the fan up

In total it took 4 hours to install from opening up the box to turning on the fan. My install was a bit longer than normal due to needing to cut through an unused attic floor and framing something to hang the fan from.

{{< image-gallery images="images/ceiling_hole.webp|Ceiling Hole|Alex looking through the hole cut in the ceiling and attic floor for the fan,images/damper_installed.webp|Damper Installed|Damper installed in the ceiling,images/junction_box.webp|Junction Box Wiring|Inside view of the junction box with the relay box and other wires,images/wiring_fan.webp|Wiring the fan|Alex wiring the fan in the attic. The fan can be seen in the background attached to lumber. The duct is already attached to the fan.,images/cover_installed.webp|Final Install|Looking up at the white cover hiding the damper in the ceiling." >}}

## Home Assistant

I modified posts from [this thread](https://norell.link/ItGwC) in the Home Assistant community. The main changes were around having two speeds and having a separate script to handle the speed changes. I also renamed the entities for each relay to reflect what they are controlling.

{{< labelled-highlight lang="yaml" filename="configuration.yaml" >}}
fan:
  - platform: template
    fans:
      house_fan:
        friendly_name: "House Fan"
        value_template: >
          {% if is_state('switch.house_fan_high', 'on') or is_state('switch.house_fan_low', 'on') %}
            on
          {% else %}
            off
          {% endif %}
        percentage_template: >
          {% if is_state('switch.house_fan_high', 'on') %}
            100
          {% elif is_state('switch.house_fan_low', 'on') %}
            50
          {% else %}
            0
          {% endif %}
        turn_on:
          service: switch.turn_on
          data:
            entity_id: switch.house_fan_low
        turn_off:
          service: switch.turn_off
          data:
            entity_id:
              - switch.house_fan_high
              - switch.house_fan_low
        set_percentage:
          service: script.set_house_fan_speed
          data:
            percentage: "{{ percentage }}"
        speed_count: 2
{{</ labelled-highlight >}}

{{< labelled-highlight lang="yaml" filename="scripts.yaml" >}}
set_house_fan_speed:
  alias: Set House Fan Speed
  sequence:
    - service: switch.turn_off
      data:
        entity_id:
          - switch.house_fan_high
          - switch.house_fan_low
    - choose:
        - conditions:
            - condition: template
              value_template: "{{ percentage == 100 }}"
          sequence:
            - service: switch.turn_on
              data:
                entity_id: switch.house_fan_high
        - conditions:
            - condition: template
              value_template: "{{ percentage == 50 }}"
          sequence:
            - service: switch.turn_on
              data:
                entity_id: switch.house_fan_low
{{</ labelled-highlight >}}

### Control

With the fan entity created, a tile can be added that has the fan speeds embedded.

```yaml
type: tile
entity: fan.house_fan
features:
  - type: fan-speed
```

{{< img src="images/house_fan.webp" alt="House fan control entity in Home Assistant" >}}

Additional automations can be set up with the fan entity, like automatically turning on when the temperature outside is less than the temperature inside and at least 1 window is open. It can also be automatically turned off if the windows are closed. The options are endless.

### Automations

The fan is completely controlled automatically based on the state of temperature inside, outside, and the state of the windows. The automations rely on some additional helper inputs and sensors, including determining if the temperature is rising and if the windows are open.

#### Turn On

{{< img src="images/turn_fan_on.svg" alt="Turn on fan logic" >}}

{{< labelled-highlight lang="yaml" filename="automations.yaml" >}}
alias: Turn on House Fan
description: Turn on the Whole House fan
trigger:
- platform: state
  entity_id:
  - binary_sensor.indoor_hotter_than_outside
  - binary_sensor.any_window_open
  to: 'on'
  alias: Window opened or the temperature dropped outside
condition:
- condition: state
  entity_id: binary_sensor.indoor_hotter_than_outside
  state: 'on'
  alias: Inside is hotter than outside
- condition: state
  entity_id: binary_sensor.any_window_open
  state: 'on'
  alias: Any window is open
- condition: state
  entity_id: binary_sensor.pirate_weather_temperature_positive_change
  state: 'off'
  alias: Outside temperature is dropping
- condition: numeric_state
  entity_id: sensor.indoor_temperature_min
  above: input_number.cold_temperature_set_point
  alias: It isn't cold in the house
action:
- alias: Turn the fan on
  choose:
  - conditions:
    - condition: state
      entity_id: binary_sensor.more_than_two_windows_open
      state: 'on'
    sequence:
    - service: fan.set_percentage
      metadata: {}
      data:
        percentage: 100
      target:
        entity_id: fan.house_fan
    alias: More than two windows are open
  - conditions:
    - condition: state
      entity_id: binary_sensor.more_than_two_windows_open
      state: 'off'
    sequence:
    - service: fan.set_percentage
      metadata: {}
      data:
        percentage: 50
      target:
        entity_id: fan.house_fan
    alias: 1 or 2 windows are open
mode: single
{{</ labelled-highlight >}}

#### Turn Off

{{< img src="images/turn_fan_off.svg" alt="Turn off fan logic" >}}

{{< labelled-highlight lang="yaml" filename="automations.yaml" >}}
alias: Turn off house fan
description: Turn off the Whole House Fan when it is getting hotter or all the windows
  are closed.
trigger:
- platform: state
  entity_id:
  - binary_sensor.indoor_temperature_positive_change
  - binary_sensor.pirate_weather_temperature_positive_change
  to: 'on'
  alias: Its getting hotter
- platform: state
  entity_id:
  - binary_sensor.any_window_open
  to: 'off'
  alias: All windows are closed
condition:
- condition: or
  conditions:
  - condition: and
    conditions:
    - condition: state
      entity_id: binary_sensor.indoor_temperature_positive_change
      state: 'on'
      for:
        hours: 0
        minutes: 0
        seconds: 0
      alias: Getting Hotter Inside
    - condition: state
      entity_id: binary_sensor.pirate_weather_temperature_positive_change
      state: 'on'
      for:
        hours: 0
        minutes: 0
        seconds: 0
      alias: Getting Hotter Outside
    alias: Getting hotter
  - condition: state
    entity_id: binary_sensor.any_window_open
    state: 'off'
    alias: All windows are closed
- condition: state
  entity_id: fan.house_fan
  state: 'on'
action:
- service: fan.turn_off
  metadata: {}
  data: {}
  target:
    entity_id: fan.house_fan
mode: single
{{</ labelled-highlight >}}

## Review

We have seen a dramatic improvement in the temperatures inside the house when turning it on. We can feel a breeze near the windows that are open and actively drawing in air, and the attic temperature drops to the same temperature as the house when running. I was expecting to see the temperature to drop even closer to the outside temperature, but I assume the thermal mass of the house is still holding on to a lot of heat energy.

{{< img src="images/temperature.webp" alt="Temperature results" >}}

The fan was turned on the first time at 10PM on July 14th, and you can quickly see the attic temperature drop. The previous night, July 13th, you can see the attic temperature stay well above the house temperature, keeping the house feeling warm. At night, a noticeable breeze is present in the house when the fan is running on low, and we both have been getting better quality sleep.