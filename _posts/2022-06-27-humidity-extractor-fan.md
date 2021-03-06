---
layout: post
title: "Humidity Triggered Extractor Fan"
tags: extractor fan bathroom automation sonoff 
permalink: /extractor-fan/
author: martin-grayson
---
<style>
  table.mbtablestyle td {
    border: none;
    text-align: center;
    background-color: white;
    font-style: italic;
    color: #828282;
  }
   table.mbtablestyle {
    border: none;
  }
</style>
* Do not remove this line (it will not be displayed)
{:toc}

## Background

When refurbishing my bathroom, I installed an [in-line extractor fan](https://www.screwfix.com/p/manrose-mf100t-100mm-axial-inline-extractor-fan-with-timer-240v/719gy) in the loft space. This extracts humid air from the shower cubicle via a discrete ceiling mounted vent.

The extractor is controlled via a [Sonoff Basic ZBR3](https://sonoff.tech/product/diy-smart-switch/basiczbr3/), a Zigbee relay that is made visible in [Home Assistant](https://www.home-assistant.io/). I also have a temperature and humidity sensor in the room, a [Sonoff SNZB-02](https://sonoff.tech/product/smart-home-security/snzb-02/).

This documents the how and why of my smart bathroom.

## Purpose

*Why did I make my bathroom extractor smart?*

My bathroom is fitted with [Hue downlights](https://www.philips-hue.com/en-gb/p/hue-white-and-colour-ambiance-3-pack-gu10/8719514342767) and a [Hue Dimmer](https://www.philips-hue.com/en-gb/p/hue-dimmer-switch--latest-model-/8719514274617) so that I can use various scenes depending on the situation. Lighting is motion controlled for the most part and scene selection depends on the time of day. At night a twilight scene is activated to make nighttime trips to the bathroom less abrupt.

As my lights are smart and I have no physical dumb switch (a pull cord switch is common in UK bathrooms), I was unable to simply activate the extractor using the switched live of a switch. I also don't think its very ergonomic to activate the extractor for every use of the bathroom since we're not always producing humidity.

This led me to search for humidity sensing extractors. I couldn't find many, and the ones that I did find were not inline (i.e. they couldn't be kept out of sight in the loft space and were not as powerful). Being the detail oriented person that I am, a bulky [ceiling mounted humidistat controlled extractor](https://www.screwfix.com/p/xpelair-dx100hts-100mm-axial-bathroom-extractor-fan-with-humidistat-timer-white-220-240v/8578h) wasn't going to cut it.

This led me to install a smart relay and figure out the automation at a later point.

## Implementation

Implementation of my bathroom extractor evolved over time and as I was actually tiling, painting and plumbing the room myself, I didn't have a whole lot of upfront experimentation time. The below documents how the system evolved over the course of a year.

### Manual Triggering

Version one of my smart extractor wasn't really smart. I simply used the long-press event of a [Hue Dimmer](https://www.philips-hue.com/en-gb/p/hue-dimmer-switch--latest-model-/8719514274617) to turn on the Sonoff and activate the extractor. The automation for this can be seen below

```yaml
- id: bathroom_switch
  alias: Bathroom - Switch
  description: Manual control of things via the bathroom switch
  trigger:
  - platform: event
    event_type: deconz_event
    event_data:
      event: 1001
      id: bathroom_remote
    id: bathroom_switch_on_long_hold
  - platform: event
    event_type: deconz_event
    event_data:
      event: 4001
      id: bathroom_remote
    id: bathroom_switch_off_long_hold
  condition: []
  action:
  - choose:
    - conditions:
      - condition: trigger
        id: bathroom_switch_on_long_hold
      sequence:
      - scene: scene.bathroom_day_mode
      - service: switch.turn_on
        data: {}
        target:
          entity_id: switch.bathroom_extractor
      - delay:
          hours: 0
          minutes: 45
          seconds: 0
          milliseconds: 0
      - service: switch.turn_off
        data: {}
        target:
          entity_id: switch.bathroom_extractor
    - conditions:
      - condition: trigger
        id: bathroom_switch_off_long_hold
      sequence:
      - service: switch.turn_off
        data: {}
        target:
          entity_id: switch.bathroom_extractor
      - service: scene.turn_on
        target:
          entity_id: scene.bathroom_off
        metadata: {}
  mode: restart
```

Note that I'm using `deconz`, the triggering events will look a little different if you're using another platform.

### Adjacent Room Humidity Difference Trigger

I wanted to automate the triggering of the extractor so that when the shower was turned on and the humidity rose, the extractor would automatically activate (and deactivate when the humidity dropped).

As the ambient relative humidity changes based on the [season among other factors](https://www.metoffice.gov.uk/weather/learn-about/weather/types-of-weather/humidity), using an absolute relative humidity trigger wasn't going to work. An ambient humidity of 60% may be normal in the summer but high in the winter. Due to this, I decided to compare the humidity of the bathroom to that of an adjacent room. If the humidity was `x%` above the adjacent room I would trigger the extractor.

I modelled this using a template sensor and automation (omitted as its super simple):

```yaml
- platform: template
  sensors:
    bathroom_humidity_difference:
      friendly_name: "Bathroom Humidity Difference"
      value_template: "{{ (states.sensor.bathroom_humidity.state | int ) - (states.sensor.joshuas_bedroom_humidity.state | int)}}"
      device_class: "humidity"
      unit_of_measurement: '%'
```

This worked fine, but had a few drawbacks:

* It was susceptible to sensors going offline (my Zigbee network has been temperamental). If the adjacent rooms sensor fell off the network, the template sensor wouldn't function correctly.
* It can also be influenced by high humidity activities in the adjacent room.
* The trigger value needed to be adequately loose to account for errors in both the bathroom sensor reading and the adjacent rooms reading and so the automation wasn't very responsive ([Propagation of Errors](https://www.geol.lsu.edu/jlorenzo/geophysics/uncertainties/Uncertaintiespart2.html#addsub)).

### Humidity Derivative Trigger

After experiencing the extractor either failing to trigger or triggering when it shouldn't, I decided to investigate an alternative trigger that was less reliant on external factors.

Logically, monitoring the rate of humidity change could be helpful here. If the humidity rapidly increased, we can assume the shower has been turned on and we should trigger the extractor to reduce the humidity. The advantages of this solution are:

* No external sensors in adjacent rooms are required and so we're not suspectable to hardware/network failures.
* Seasonal ambient humidity changes do not need to be considered (i.e. we wont hardcode activation at `x%` humidity).
* A trigger value could be much more precisely tuned and the extractor triggered sooner due to not combining the error of two sensors.

To accurately calculate the rate of humidity change (the derivative), I relocated my sensor to sit above the shower head. This worked in my instance as my shower head is fairly large and so the sensor could be kept dry and out of sight.

| ![Sensor and extractor fan]({{ site.baseurl }}/assets/sensor_above_shower_head.jpg) |
| Note the sensor hidden above the shower head (it needs a dust!). |
{:.mbtablestyle}

The logical way to calculate this humidity derivative was to use the [Home Assistant Derivative Helper](https://www.home-assistant.io/integrations/derivative/), as per the manual:
> The derivative integration creates a sensor that estimates the derivative of the values provided by another sensor (the source sensor). Derivative sensors are updated upon changes of the source sensor.

| ![Relative humidity in a typical day]({{ site.baseurl }}/assets/humidity_over_time.png) |
| The bathroom's relative humidity over a typical day, notice the two spikes when the shower was used. |
{:.mbtablestyle}
It's implementation as a sensor:

```yaml
- platform: derivative
  source: sensor.bathroom_humidity
  name: Bathroom Humidity Change Per Minute
  round: 2
  time_window: "00:05:00" 
  unit_time: min
```

Notice the use of the [`time_window`](https://www.home-assistant.io/integrations/derivative/#time-window) parameter to filter out any noise using a rolling 5 minute average.

| ![Humidity derivative sensor in a typical day]({{ site.baseurl }}/assets/humidity_change.png) |
| The humidity derivative sensor over the same period. |
{:.mbtablestyle}

I combined this with a binary sensor to translate this differential into a binary trigger. This will trigger the extractor when the derivative sensor detects a rate of change greater than 1.5% per minute.

```yaml
- platform: threshold
  entity_id: sensor.bathroom_humidity_change_per_minute
  upper: 1.5
  name: Bathroom Extractor Trigger
```

The automation for this looks very similar to my original solution since the threshold sensor above can be used in a state trigger:

```yaml
- id: bathroom_extractor
  alias: Bathroom - Extractor
  trigger:
  - platform: state
    to: 'on'
    from: 'off'
    entity_id: binary_sensor.bathroom_extractor_trigger
    id: bathroom_humidity_high
```

*n.b. The rest of the automation has been omitted as it follows the same pattern as the original automation.*
