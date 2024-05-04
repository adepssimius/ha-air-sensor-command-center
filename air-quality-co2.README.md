# NOTE: All of the config contained here is for reference. The updated config is in the yaml files that are siblings to this file. This write up is to show my thought process for each of the items added to the dashboard.

The `HVAC Controllable CO2 Concentrations` chart at the top shows the status of my HVAC fan combined with the status of 2 sensor groups:
* The maximum reading from all CO2 sensors in my conditioned space
* The average reading from all CO2 sensors in my conditioned space

```
  - platform: group
    name: CO2 Average
    type: mean
    unit_of_measurement: ppm
    entities:
      - sensor.basement_air_1_co2
      - sensor.master_bedroom_air_1_co2
  - platform: group
    name: CO2 Max HVAC Connected
    type: max
    unit_of_measurement: ppm
    entities:
      - sensor.basement_air_1_co2
      - sensor.master_bedroom_air_1_co2
```

This allowed me to see how normalized the CO2 concentrations were compared to the rest of the house (I know it's just one other sensor I'm comparing against, but it's a room where there is no CO2 generation and gasses are only mixing). Here is my mini graph card YAML for the top card:

```
type: custom:mini-graph-card
name: HVAC Controllable CO2 Concentrations
entities:
  - entity: sensor.co2_max_hvac_connected
    name: Max
    show_state: true
    state_adaptive_color: true
  - entity: sensor.co2_average
    name: Average
    show_state: true
    state_adaptive_color: true
  - color: '#FFFFFF'
    entity: sensor.hvac_fan_integer
    name: HVAC Ventilation
    show_line: false
    show_points: false
    y_axis: secondary
    smoothing: false
color_thresholds:
  - value: 400
    color: '#5eff33'
  - value: 600
    color: '#33d4ff'
  - value: 1000
    color: '#f39c12'
  - value: 1500
    color: '#d35400'
  - value: 2000
    color: '#c0392b'
show:
  labels: true
decimals: 0
hours_to_show: 24
points_per_hour: 4
update_interval: 15
line_width: 2
animate: true
```

The `CO2 Max Difference from Mean` chart shows the fan status, the difference between the values of the max and average, and the trend of how those two values are changing relative to each other. This is the more interesting graph in my opinion, since this graph shows the fan status along with the physical measurements the fan can directly affect.

`CO2 Max Difference from Mean` is a template sensor that tells you if you have a high concentration of CO2 in one particular room. If you do, then turning on the ventilation can fix that. Sensor YAML:
```
  - platform: template
    sensors:
      co2_max_difference_from_mean:
        friendly_name: "CO2 Max Difference from Mean"
        value_template: "{{ states('sensor.co2_max_hvac_connected') | int - states('sensor.co2_average') | int }}"
        unit_of_measurement: ppm
        icon_template: mdi:molecule-co2
```

`CO2 Normalization Rate` is a bit trickier, since it is template sensor based on a derivative sensor. I wanted a sensor that would give me an indication of if the HVAC fan was effectively mixing the gasses in the house or not, with a positive value indicating that the CO2 concentrations in the house were converging and a negative value indicating that the concentrations were diverging. Since the derivative of the difference will be negative as the concentrations converge, I had to wrap the derivative in a template to change the sign of the sensor. Sensors YAML:
```
  - platform: derivative
    source: sensor.co2_max_difference_from_mean
    name: CO2 Denormalization Rate
    round: 0
    unit_time: h
    time_window: "00:05:00"
  - platform: template
    sensors:
      co2_normalization_rate:
        friendly_name: "CO2 Normalization Rate"
        value_template: "{{ int(states.sensor.co2_denormalization_rate.state) * -1  }}"
        unit_of_measurement: "ppm/h"
        icon_template: mdi:molecule-co2
```

`CO2 Max Difference from Mean` chart YAML:
```
type: custom:mini-graph-card
entities:
  - entity: sensor.co2_max_difference_from_mean
    state_adaptive_color: true
  - color: '#AAAAAA'
    entity: sensor.co2_normalization_rate
    show_state: true
    state_adaptive_color: true
  - color: '#FFFFFF'
    entity: sensor.hvac_fan_integer
    name: HVAC Ventilation
    show_line: false
    show_points: false
    y_axis: secondary
    smoothing: false
color_thresholds:
  - value: 50
    color: '#5eff33'
  - value: 100
    color: '#33d4ff'
  - value: 200
    color: '#f39c12'
  - value: 400
    color: '#d35400'
  - value: 1000
    color: '#c0392b'
show:
  labels: true
hours_to_show: 12
points_per_hour: 4
update_interval: 15
line_width: 2
animate: true
```

Now that I had a dashboard to see how well I was controlling CO2 concentrations in my house, I would be able to implement some automations and test how well those automations worked. What I landed on is to implement a cieling for the max CO2 PPM value which would trigger the HVAC fan on combined with several off triggers:
* Max CO2 reading reaches an acceptable level (I don't need it to run anymore)
* Max CO2 reading gets close to the average CO2 reading (Gasses are already well mixed and running the fan isn't helpful anymore)
* Normalization rate is too low (Running the fan is not effectively mixing the gasses for some reason)

The YAML for the inputs:
```
input_number:
  co2_ventilation_off_setpoint:
    name: CO2 Ventilation Off Setpoint
    initial: 750
    min: 400
    max: 2000
    step: 50
    mode: box
    unit_of_measurement: ppm
  co2_ventilation_on_setpoint:
    name: CO2 Ventilation On Setpoint
    initial: 1000
    min: 450
    max: 2000
    step: 50
    mode: box
    unit_of_measurement: ppm
  co2_ventilation_normalization_rate_minimum:
    name: CO2 Ventilation Minimum Rate
    initial: 150
    min: 10
    max: 20000
    step: 10
    mode: box
    unit_of_measurement: ppm/h
  co2_delta_ventilation_off_setpoint:
    name: CO2 Δ Ventilation Off Setpoint
    initial: 30
    max: 500
    min: 0
    step: 10
    mode: box
    unit_of_measurement: ppm
```

YAML for the input card:
```
type: entities
entities:
  - entity: input_number.co2_ventilation_on_setpoint
    icon: mdi:format-vertical-align-top
    name: 'On'
  - entity: input_number.co2_ventilation_off_setpoint
    icon: mdi:format-vertical-align-bottom
    name: 'Off'
  - entity: input_number.co2_delta_ventilation_off_setpoint
    name: Δ Off Setpoint
    icon: mdi:scale-balance
  - entity: input_number.co2_ventilation_normalization_rate_minimum
    icon: mdi:swap-horizontal
    name: Off Min Rate
view_layout:
  position: sidebar
title: Ventilation Control Setpoints
```

Since I did not want to turn the HVAC fan off if I had not turned it on, I added an input boolean to keep track of if the fan was running due to this automation:
```
input_boolean:
  co2_ventilation_running:
    name: CO2 Ventilation Running
    initial: false
    icon: mdi:molecule-co2
```

Now that all of the triggers and inputs are set up, we have an automation to turn the fan on:
```
automation:
  - alias: CO2 Circulate Start
    description: ""
    trigger:
      - platform: numeric_state
        entity_id:
          - sensor.co2_max_hvac_connected
        for:
          hours: 0
          minutes: 5
          seconds: 0
        above: input_number.co2_ventilation_on_setpoint
    condition:
      - condition: state
        entity_id: climate.downstairs
        attribute: fan_mode
        state: "off"
    action:
      - service: climate.set_fan_mode
        metadata: {}
        data:
          fan_mode: "on"
        target:
          device_id: 6156c6b208c71acd7d82cbd89f3276fd
      - service: input_boolean.turn_on
        target:
          entity_id: input_boolean.co2_ventilation_running
        data: {}
    mode: single
```

And an automation to turn the fan off:
```
automation:
  - alias: CO2 Circulate Stop
    description: ""
    trigger:
      - platform: numeric_state
        entity_id:
          - sensor.co2_max_hvac_connected
        for:
          hours: 0
          minutes: 5
          seconds: 0
        below: input_number.co2_ventilation_off_setpoint
      - platform: numeric_state
        entity_id:
          - sensor.co2_max_difference_from_mean
        for:
          hours: 0
          minutes: 5
          seconds: 0
        below: input_number.co2_delta_ventilation_off_setpoint
      - platform: numeric_state
        entity_id:
          - sensor.co2_normalization_rate
        for:
          hours: 0
          minutes: 5
          seconds: 0
        below: input_number.co2_ventilation_normalization_rate_minimum
    condition:
      - condition: state
        entity_id: input_boolean.co2_ventilation_running
        state: "on"
    action:
      - service: climate.set_fan_mode
        metadata: {}
        data:
          fan_mode: "off"
        target:
          device_id: 6156c6b208c71acd7d82cbd89f3276fd
      - service: input_boolean.turn_off
        target:
          entity_id: input_boolean.co2_ventilation_running
        data: {}
    mode: single
```

You will need to point the `climate.set_fan_mode` service to point at your device.

Lastly, I wanted to be able to see if the HVAC fan was on and if it was on because of this automation, so I added an additional card to the sidebar to show the status of the fan and the `co2_ventilation_running` boolean.
```
show_name: true
show_icon: true
show_state: true
type: glance
entities:
  - entity: input_boolean.co2_ventilation_running
    name: CO2 Ventilation
  - entity: sensor.hvac_fan
state_color: true
view_layout:
  position: sidebar
```