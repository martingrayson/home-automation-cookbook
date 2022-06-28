---
layout: post
title: "Sonos Doorbell"
tags: doorbell homeassistant sonos
permalink: /sonos-doorbell/
author: martin-grayson
---
* Do not remove this line (it will not be displayed)
{:toc}

## Background

Everyone has a smart doorbell these days, but not everyone has a chime box or actually wants one.

Last year I had my front door replaced and in the process, the door surround was also rebuilt. This gave me the opportunity to add wiring for a permanently wired smart doorbell, previously I had a battery operated dumb doorbell with no wiring available.

Being a Ubiquiti fan boy, I managed to import a [G4 Doorbell](https://store.ui.com/products/uvc-g4-doorbell) before they were released in the UK. To power the doorbell I needed a [hefty transformer](https://www.screwfix.com/p/british-general-fortress-8-24v-ac-8va-bell-transformer-module/8707p) that wouldn't fit in a chime box.

I ran wiring for the new doorbell and fitted a transformer on the wall in an enclosure near the front door. I opted to skip installing a chime box to keep the internal wall free of too many components. The omission of a chime box led me to use my smart speakers to announce visitors.

Over the years I've acquired a few Sonos speakers, in this post I'll document how I use these in place of a door bell chime.

## My Gear

Just to clarify my setup for those following, I'm using:

* Sonos Playbar, Play:1 and Play:3
* Unifi G4 Doorbell
* Home Assistant OS (previously I ran Supervised on Ubuntu) on an Intel NUC

## Requirements

I wanted to announce visitors on all speakers within my home using a generic chime sample. However, I have a Playbar so its highly likely that the TV will on when the doorbell is pressed, so I needed to consider how to resume the TV after the chime plays.

In addition to this, it would be nice to retain any current playlists and not have the doorbell ruin the mood.

The rough pseudocode of what I wanted to achieve:

1. Doorbell is pressed
1. Notification is pushed to all mobile devices
1. The state of all media players is stored
1. The volume of all media players is set to an appropriate level for the chime
1. All media players are grouped
1. The chime is broadcast
1. The camera image is streamed to my Google Home
1. Wait 5s
1. Resume all media players

## Implementation

The YAML for my automation can be found below. Notice the strange `binary_sensor` I'm using as a trigger - this comes from the Unifi integration for Home Assistant and is set to `on` then the doorbell is pressed.

```yaml
- id: doorbell_fired
  alias: Doorbell
  trigger:
  - platform: state
    entity_id: binary_sensor.doorbell_doorbell
    to: 'on'
  condition: []
  action:
  - service: notify.notify
    data:
      message: Someone is at the door 🔔
      title: Door Bell
  - data: {}
    entity_id: all
    service: sonos.snapshot
  - data:
      volume_level: 0.3
    service: media_player.volume_set
    target:
      entity_id: media_player.dining_room, media_player.kitchen, media_player.lounge
  - data:
      master: media_player.lounge
      entity_id: media_player.kitchen
    service: sonos.join
  - data:
      media_content_id: https://<my-domain>/local/sounds/doorbell-1.mp3
      media_content_type: music
    service: media_player.play_media
    target:
      entity_id: media_player.lounge
  - data:
      entity_id: camera.doorbell
      media_player: media_player.tv_kitchen
    entity_id: camera.doorbell
    service: camera.play_stream
  - delay: 00:00:05
  - data: {}
    entity_id: all
    service: sonos.restore
  - delay: 00:01:00
  - data:
      entity_id: media_player.tv_kitchen
    entity_id: media_player.tv_kitchen
    service: media_player.turn_off
  mode: restart
  max: 10
  ```

### Limitations

* Going through all those network hops, despite being a local doorbell and not having to hit the cloud, is a little slow. This means sometimes impatient delivery drivers start walking away before you respond to the chime.

* There is quite a lag on casting the doorbell's camera to the Google Home. We're talking 2-5s of latency.
