# JBL Bar 800 MK2 in Home Assistant, fully local

Control a **JBL Bar 800 MK2** from Home Assistant through its local HTTPS API, with no custom integration and no cloud:

- Night mode and Broadcast (FlexListening) as switches, plus an exclusive "normal / night / broadcast" mode control
- Detachable rear speakers: battery, charging, docked, and volume (Mute / Low / Mid / High)
- Bass, matching the app's 1 to 5 steps
- Reliable **standby detection**, so HA does not show a mode the bar has already dropped
- A compact Bubble Card section for the dashboard

Everything uses core `command_line` and `shell_command`. I went this way because the HACS JBL integration I tried first had open reports of blocking HA startup.

Tested on a Bar 800 MK2, firmware `26.33.11.82.00` (One OS 4.0), November 2026. Other JBL One bars (300 to 1300, MK2) share most of this API, but check the values yourself.

## Contents

1. [Requirements](#requirements)
2. [How the API works](#how-the-api-works)
3. [Command reference](#command-reference)
4. [Install](#install)
5. [Scripts](#scripts)
6. [Standby detection](#standby-detection)
7. [Dashboard](#dashboard)
8. [Mapping values yourself](#mapping-values-yourself)
9. [Gotchas](#gotchas)
10. [Credits](#credits)

## Requirements

- The bar on your LAN with a fixed IP (DHCP reservation). This guide uses `192.168.1.207`.
- The **JBL One app's client certificate** as a PEM file. The bar refuses any client without it. It is not included here; [MrBearPresident/JBL_Soundbar](https://github.com/MrBearPresident/JBL_Soundbar) documents where it comes from. Put it at `/config/jbl_client.pem`.
- `curl` inside HA (Home Assistant OS has it).
- For the dashboard: [Bubble Card](https://github.com/Clooos/Bubble-Card).

> **File editor gotcha:** in the File editor app, the `homeassistant/` folder **is** `/config`. A `homeassistant/config/` subfolder is a different place. If you upload the cert there, curl exits with code 58 and every switch reads off.

## How the API works

```
https://<bar-ip>/httpapi.asp?command=<name>                 # most getters (GET)
https://<bar-ip>/httpapi.asp  -d 'command=<name>&payload=<json>'   # setters and some getters (POST)
```

- TLS uses a self-signed cert, so `-k` (insecure) is needed. Authentication is the client cert (`-E file.pem`).
- Replies are JSON with `"error_code":"0"` on success. Unsupported commands answer with the plain text `unknown command`.
- POST bodies are sent raw (`-d`), not URL-encoded. `getProdSetting` in particular only works that way.
- The bar keeps answering while it is in standby.

Windows `curl` (Schannel) cannot load the PEM. Test from HA's terminal, Linux, macOS or Python.

## Command reference

Everything below was verified on the 800 MK2 unless marked otherwise.

| Purpose | Read | Write | Values |
|---|---|---|---|
| Power / standby | GET `getPowerState` | | `power_state`: `0` on, `1` standby or off |
| Night mode | GET `getPersonalListeningMode` | POST `setPersonalListeningMode` `{"status":"on"}` | `status`: `on` / `off` |
| Broadcast | POST `getProdSetting` `{"FlexListening":""}` | POST `setProdSetting` `{"FlexListening":"2"}` | `0` off, `1` mono, `2` stereo. Turning it on turns night mode off |
| Rear speakers | GET `getRearSpeakerStatus` | | `rears[0]` left, `rears[1]` right, each with `capicity` (sic, battery %), `charging`, `docked`, `status` (online/offline), `volume` |
| Rear volume | GET `getRearSpeakerVolume` | POST `setRearSpeakerVolume` `{"volume":"N"}` | `0` Mute, `1` Low, `2` Mid, `3` High. Ignored while the rears are docked |
| Bass | GET `getEQ` | POST `setActiveEQ` (see below) | App step 1 to 5 = Custom EQ 125 Hz gain of `-9`, `-4.5`, `0`, `+3`, `+6` dB |
| EQ presets | GET `getEQList` | | `active_eq_id`, plus the 7 band gains of each preset |
| Smart mode | GET `getSmartMode` | POST `setSmartMode` | Needs the full payload `{"auto_mode_switch":1,"manual_mode":N,"status":"on"}`. Not Broadcast |
| Voice clarity | GET `getPureVoiceState` | POST `setPureVoiceState` `{"purevoice_state":"1"}` (from SH1FT-W, untested here) | `1` on |
| Dolby / Atmos | GET `getAtmosLevel` | | Reads `atmos_level` and `status`. No setter found yet |
| Source | GET `getMediaSourceStatus` | POST `setMediaSource` (from SH1FT-W, untested here) | `TV`, `HDMI`, `BT` |
| Now playing | GET `getPlayerStatus`, `getStreamingStatus` | | play status, volume, `is_atmos` |

**Not** what they look like:

- `sendAppController` with `keyRear` steps the rear volume (3 wraps to 0). It is not Broadcast.
- `setSmartMode` `manual_mode` is the app's Smart mode card, not Broadcast and not night mode.

## Install

Add this to `configuration.yaml`. If you already have a `shell_command:` or `command_line:` block, merge these entries into it instead of adding a second heading. Then check the config and restart HA (a reload is not enough for new `command_line` entities).

```yaml
command_line:
  - switch:
      name: JBL Night Mode
      unique_id: jbl_bar_night_mode
      icon: mdi:weather-night
      scan_interval: 30
      command_timeout: 10
      command_on: >-
        curl -sk -m 8 -E /config/jbl_client.pem https://192.168.1.207/httpapi.asp
        -d 'command=setPersonalListeningMode&payload={"status":"on"}'
      command_off: >-
        curl -sk -m 8 -E /config/jbl_client.pem https://192.168.1.207/httpapi.asp
        -d 'command=setPersonalListeningMode&payload={"status":"off"}'
      command_state: >-
        curl -sk -m 8 -E /config/jbl_client.pem
        'https://192.168.1.207/httpapi.asp?command=getPersonalListeningMode'
      value_template: "{{ value_json.status == 'on' }}"

  - switch:
      name: JBL Broadcast Mode
      unique_id: jbl_bar_broadcast_mode
      icon: mdi:surround-sound
      scan_interval: 30
      command_timeout: 10
      command_on: >-
        curl -sk -m 8 -E /config/jbl_client.pem https://192.168.1.207/httpapi.asp
        -d 'command=setProdSetting&payload={"FlexListening":"2"}'
      command_off: >-
        curl -sk -m 8 -E /config/jbl_client.pem https://192.168.1.207/httpapi.asp
        -d 'command=setProdSetting&payload={"FlexListening":"0"}'
      command_state: >-
        curl -sk -m 8 -E /config/jbl_client.pem https://192.168.1.207/httpapi.asp
        -d 'command=getProdSetting&payload={"FlexListening":""}'
      value_template: "{{ value_json.FlexListening != '0' }}"

  - sensor:
      name: JBL Power State
      unique_id: jbl_power_state
      scan_interval: 30
      command_timeout: 10
      command: >-
        curl -sk -m 8 -E /config/jbl_client.pem
        'https://192.168.1.207/httpapi.asp?command=getPowerState'
      value_template: "{{ value_json.power_state }}"

  - sensor:
      name: JBL Rear Left Battery
      unique_id: jbl_rear_left_battery
      scan_interval: 60
      command_timeout: 10
      command: >-
        curl -sk -m 8 -E /config/jbl_client.pem
        'https://192.168.1.207/httpapi.asp?command=getRearSpeakerStatus'
      value_template: "{{ value_json.rears[0].capicity }}"
      unit_of_measurement: "%"
      device_class: battery
      json_attributes_path: "$.rears[0]"
      json_attributes: [channel, charging, docked, status, volume]

  - sensor:
      name: JBL Rear Right Battery
      unique_id: jbl_rear_right_battery
      scan_interval: 60
      command_timeout: 10
      command: >-
        curl -sk -m 8 -E /config/jbl_client.pem
        'https://192.168.1.207/httpapi.asp?command=getRearSpeakerStatus'
      value_template: "{{ value_json.rears[1].capicity }}"
      unit_of_measurement: "%"
      device_class: battery
      json_attributes_path: "$.rears[1]"
      json_attributes: [channel, charging, docked, status, volume]

  - sensor:
      name: JBL Bass
      unique_id: jbl_bass
      scan_interval: 60
      command_timeout: 10
      command: >-
        curl -sk -m 8 -E /config/jbl_client.pem
        'https://192.168.1.207/httpapi.asp?command=getEQ'
      value_template: >-
        {% set g = value_json.eq_setting.eq_payload.gain[0] | float(0) %}
        {{ {-9.0: 1, -4.5: 2, 0.0: 3, 3.0: 4, 6.0: 5}.get(g, 'unknown') }}

shell_command:
  # data: {level: 0-3}
  jbl_rear_volume: >-
    curl -sk -m 8 -E /config/jbl_client.pem https://192.168.1.207/httpapi.asp
    -d 'command=setRearSpeakerVolume&payload={"volume":"{{ level }}"}'
  # data: {level: 1-5}. Writes 0 dB to the other six bands, see Gotchas.
  jbl_bass: >-
    curl -sk -m 8 -E /config/jbl_client.pem https://192.168.1.207/httpapi.asp
    -d 'command=setActiveEQ&payload={"active_eq_id":"0","band":7,"eq_payload":{"fs":[125,250,500,1000,2000,4000,8000],"gain":[{{ {1: -9, 2: -4.5, 3: 0, 4: 3, 5: 6}[level | int] }},0,0,0,0,0,0]}}'
```

You get:

| Entity | State |
|---|---|
| `switch.jbl_night_mode` | on / off |
| `switch.jbl_broadcast_mode` | on / off |
| `sensor.jbl_power_state` | `0` awake, `1` standby/off |
| `sensor.jbl_rear_left_battery`, `sensor.jbl_rear_right_battery` | battery %, with `charging`, `docked`, `status`, `volume` attributes |
| `sensor.jbl_bass` | `1` to `5` |
| `shell_command.jbl_rear_volume`, `shell_command.jbl_bass` | actions |

## Scripts

Create these in the UI (Settings, Automations and scenes, Scripts, Edit in YAML).

### Set one listening mode

Night and Broadcast are separate settings on the bar, and enabling Broadcast drops night mode. This script makes them one exclusive control: asking for a mode turns the other off first, and asking for the mode that is already on goes back to normal.

```yaml
alias: JBL Set Mode
mode: restart
fields:
  mode:
    required: true
    selector:
      select:
        options: [normal, night, broadcast]
sequence:
  - choose:
      - conditions:
          - condition: template
            value_template: "{{ mode == 'night' }}"
          - condition: or
            conditions:
              - condition: state
                entity_id: switch.jbl_night_mode
                state: "off"
              - condition: state          # asleep = really normal, see Standby detection
                entity_id: sensor.jbl_power_state
                state: "1"
        sequence:
          - action: switch.turn_off
            target: { entity_id: switch.jbl_broadcast_mode }
          - action: switch.turn_on
            target: { entity_id: switch.jbl_night_mode }
      - conditions:
          - condition: template
            value_template: "{{ mode == 'broadcast' }}"
          - condition: or
            conditions:
              - condition: state
                entity_id: switch.jbl_broadcast_mode
                state: "off"
              - condition: state
                entity_id: sensor.jbl_power_state
                state: "1"
        sequence:
          - action: switch.turn_off
            target: { entity_id: switch.jbl_night_mode }
          - action: switch.turn_on
            target: { entity_id: switch.jbl_broadcast_mode }
    default:
      - action: switch.turn_off
        target: { entity_id: [switch.jbl_night_mode, switch.jbl_broadcast_mode] }
```

### Cycle modes from one remote button

Normal, then night, then broadcast, then normal. Map any spare TV remote button to it (I use Button Mapper on Android TV).

```yaml
alias: JBL Mode Cycle
mode: single
sequence:
  - choose:
      - alias: Night is on, so go to broadcast
        conditions:
          - condition: not
            conditions:
              - condition: state
                entity_id: sensor.jbl_power_state
                state: "1"
          - condition: state
            entity_id: switch.jbl_night_mode
            state: "on"
        sequence:
          - action: switch.turn_off
            target: { entity_id: switch.jbl_night_mode }
          - action: switch.turn_on
            target: { entity_id: switch.jbl_broadcast_mode }
      - alias: Broadcast is on, so go to normal
        conditions:
          - condition: not
            conditions:
              - condition: state
                entity_id: sensor.jbl_power_state
                state: "1"
          - condition: state
            entity_id: switch.jbl_broadcast_mode
            state: "on"
        sequence:
          - action: switch.turn_off
            target: { entity_id: switch.jbl_broadcast_mode }
    default:
      - alias: Normal (or asleep), so go to night
        action: switch.turn_on
        target: { entity_id: switch.jbl_night_mode }
```

### Rear volume and bass with instant feedback

Wrap each `shell_command` so the sensor refreshes right after the write. Otherwise the dashboard only catches up on the next poll, up to a minute later.

```yaml
alias: JBL Rear Volume Set
mode: restart
fields:
  level:
    required: true
    selector: { number: { min: 0, max: 3, step: 1 } }
sequence:
  - action: shell_command.jbl_rear_volume
    data: { level: "{{ level | int }}" }
  - delay: { seconds: 1 }
  - action: homeassistant.update_entity
    target:
      entity_id: [sensor.jbl_rear_left_battery, sensor.jbl_rear_right_battery]
```

`JBL Bass Set` is the same with `shell_command.jbl_bass`, `level` 1 to 5, and `sensor.jbl_bass`.

## Standby detection

This is the part that took the most digging.

The bar resets to normal (night and broadcast off) every time it goes to standby. The command_line switches keep polling, but while the bar sleeps they still report the old value and only catch up when it wakes. Without a fix, HA can show "night" for hours while the bar is really on normal.

What does **not** work as a standby signal:

| Candidate | Why not |
|---|---|
| The Cast `media_player` (auto-discovered) | `off` whenever nothing is being cast, including while the bar plays Spotify Connect or TV audio |
| The DLNA `media_player` (`dlna_dmr`) | Never goes `off` |
| "API unreachable" | The bar keeps answering while asleep |

What works: **`getPowerState`**. Measured on my bar:

| Event | `power_state` |
|---|---|
| Playing | `0` |
| Playback stopped | still `0` |
| Auto standby, about 10 minutes after going quiet | `1` |
| Woken from the app remote | back to `0` within seconds |
| Turned off from the app remote | `1` |

So treat `sensor.jbl_power_state == "1"` as "normal" everywhere: in the scripts above and in the dashboard highlight below.

## Dashboard

A Bubble Card section: a header with the mode icons and the bar's volume, then a rear speaker row and a bass row. Each row is one pill with sub-buttons, and the active step is highlighted through Bubble Card's JS style templates. Replace `media_player.jbl_bar_800mk2_2` with your bar's DLNA or other volume-capable entity.

```yaml
# Header: moon = night, surround = broadcast, slider = bar volume
type: custom:bubble-card
card_type: separator
name: JBL
icon: mdi:soundbar
sub_button:
  main:
    - entity: switch.jbl_night_mode
      icon: mdi:weather-night
      show_name: false
      show_background: true
      state_background: false
      tap_action:
        action: perform-action
        perform_action: script.jbl_set_mode
        data: { mode: night }
    - entity: switch.jbl_broadcast_mode
      icon: mdi:surround-sound
      show_name: false
      show_background: true
      state_background: false
      tap_action:
        action: perform-action
        perform_action: script.jbl_set_mode
        data: { mode: broadcast }
    - entity: media_player.jbl_bar_800mk2_2
      icon: mdi:volume-high
      sub_button_type: slider
styles: |
  .bubble-sub-button-1 { background-color: ${hass.states['sensor.jbl_power_state']?.state !== '1' && hass.states['switch.jbl_night_mode']?.state === 'on' ? 'var(--accent-color)' : ''} !important; }
  .bubble-sub-button-2 { background-color: ${hass.states['sensor.jbl_power_state']?.state !== '1' && hass.states['switch.jbl_broadcast_mode']?.state === 'on' ? 'var(--accent-color)' : ''} !important; }
```

```yaml
# Rear speakers: L / R battery (green while charging), then Mute / Low / Mid / High
type: custom:bubble-card
card_type: button
button_type: name
name: Rears
icon: mdi:speaker-wireless
show_name: false
tap_action: { action: none }
sub_button:
  main:
    - { entity: sensor.jbl_rear_left_battery, name: L, show_name: true, show_state: true, show_icon: false, show_background: false }
    - { entity: sensor.jbl_rear_right_battery, name: R, show_name: true, show_state: true, show_icon: false, show_background: false }
    - { icon: mdi:volume-off, show_background: true, tap_action: { action: perform-action, perform_action: script.jbl_rear_volume_set, data: { level: 0 } } }
    - { icon: mdi:volume-low, show_background: true, tap_action: { action: perform-action, perform_action: script.jbl_rear_volume_set, data: { level: 1 } } }
    - { icon: mdi:volume-medium, show_background: true, tap_action: { action: perform-action, perform_action: script.jbl_rear_volume_set, data: { level: 2 } } }
    - { icon: mdi:volume-high, show_background: true, tap_action: { action: perform-action, perform_action: script.jbl_rear_volume_set, data: { level: 3 } } }
styles: |
  .bubble-sub-button-1 { color: ${hass.states['sensor.jbl_rear_left_battery']?.attributes?.charging ? '#2fe08a' : ''} !important; }
  .bubble-sub-button-2 { color: ${hass.states['sensor.jbl_rear_right_battery']?.attributes?.charging ? '#2fe08a' : ''} !important; }
  .bubble-sub-button-3 { background-color: ${hass.states['sensor.jbl_rear_left_battery']?.attributes?.volume == 0 ? 'rgba(30, 110, 200, 0.35)' : ''} !important; }
  .bubble-sub-button-4 { background-color: ${hass.states['sensor.jbl_rear_left_battery']?.attributes?.volume == 1 ? 'rgba(30, 110, 200, 0.35)' : ''} !important; }
  .bubble-sub-button-5 { background-color: ${hass.states['sensor.jbl_rear_left_battery']?.attributes?.volume == 2 ? 'rgba(30, 110, 200, 0.35)' : ''} !important; }
  .bubble-sub-button-6 { background-color: ${hass.states['sensor.jbl_rear_left_battery']?.attributes?.volume == 3 ? 'rgba(30, 110, 200, 0.35)' : ''} !important; }
```

```yaml
# Bass 1 to 5
type: custom:bubble-card
card_type: button
button_type: name
name: Bass
icon: mdi:sine-wave
tap_action: { action: none }
sub_button:
  main:
    - { name: "1", show_name: true, show_icon: false, show_background: true, tap_action: { action: perform-action, perform_action: script.jbl_bass_set, data: { level: 1 } } }
    - { name: "2", show_name: true, show_icon: false, show_background: true, tap_action: { action: perform-action, perform_action: script.jbl_bass_set, data: { level: 2 } } }
    - { name: "3", show_name: true, show_icon: false, show_background: true, tap_action: { action: perform-action, perform_action: script.jbl_bass_set, data: { level: 3 } } }
    - { name: "4", show_name: true, show_icon: false, show_background: true, tap_action: { action: perform-action, perform_action: script.jbl_bass_set, data: { level: 4 } } }
    - { name: "5", show_name: true, show_icon: false, show_background: true, tap_action: { action: perform-action, perform_action: script.jbl_bass_set, data: { level: 5 } } }
styles: |
  .bubble-sub-button-1 { background-color: ${hass.states['sensor.jbl_bass']?.state === '1' ? 'rgba(30, 110, 200, 0.35)' : ''} !important; }
  .bubble-sub-button-2 { background-color: ${hass.states['sensor.jbl_bass']?.state === '2' ? 'rgba(30, 110, 200, 0.35)' : ''} !important; }
  .bubble-sub-button-3 { background-color: ${hass.states['sensor.jbl_bass']?.state === '3' ? 'rgba(30, 110, 200, 0.35)' : ''} !important; }
  .bubble-sub-button-4 { background-color: ${hass.states['sensor.jbl_bass']?.state === '4' ? 'rgba(30, 110, 200, 0.35)' : ''} !important; }
  .bubble-sub-button-5 { background-color: ${hass.states['sensor.jbl_bass']?.state === '5' ? 'rgba(30, 110, 200, 0.35)' : ''} !important; }
```

## Mapping values yourself

The app does not tell you what number each step sends. To find out, poll the getters fast while you move a control through every step in the JBL One app, then match the log against the order you tapped. Read-only, nothing is written to the bar:

```python
"""Log JBL setting changes once a second. Usage: python jbl_watch.py <client.pem> <seconds>"""
import json, ssl, sys, time, urllib.request
from datetime import datetime

ctx = ssl.create_default_context()
ctx.check_hostname = False
ctx.verify_mode = ssl.CERT_NONE
ctx.load_cert_chain(sys.argv[1])

def get(cmd):
    try:
        url = "https://192.168.1.207/httpapi.asp?command=" + cmd
        with urllib.request.urlopen(url, context=ctx, timeout=5) as r:
            return json.loads(r.read().decode())
    except Exception as e:
        return {"err": type(e).__name__}

last, end = None, time.time() + int(sys.argv[2])
while time.time() < end:
    snap = {
        "power": get("getPowerState").get("power_state"),
        "rear": get("getRearSpeakerVolume").get("volume"),
        "bass": get("getEQ").get("eq_setting", {}).get("eq_payload", {}).get("gain"),
        "atmos": get("getAtmosLevel"),
    }
    if snap != last:
        print(datetime.now().strftime("%H:%M:%S"), json.dumps(snap), flush=True)
        last = snap
    time.sleep(1)
```

Change one control at a time and wait a few seconds per step. A 2 second poll missed quick taps.

## Gotchas

- **Bass writes the whole EQ.** `setActiveEQ` sends all seven bands, so the bass command above resets the other six to 0 dB. Fine if you only use Bass. If you also tune mids or treble, read `getEQList` first and keep those gains.
- **Bass steps are not even:** -9, -4.5, 0, +3, +6 dB.
- **Rear volume does nothing while the rears are docked.**
- **Broadcast is not Smart mode.** `setSmartMode` looks like the sound mode setting but does not control Broadcast. Broadcast is `FlexListening`.
- **Night mode is independent of `manual_mode`.** It reads the same with night on and off.
- **Two auto-discovered media players.** The Cast one is mostly `off`, and the DLNA one reports volume and track. Use the DLNA one for volume sliders.
- **Spotify Connect lists the bar by its device name** (`JBL BAR 800MK2` by default). If you rename the bar in the JBL app, update any template that compares the Spotify `source`.
- **`shell_command` with templates runs without a shell.** Arguments are split shlex-style, so keep the quoting exactly as shown.
- **Restart, not reload,** after adding new `command_line` entities.

## Credits

- [SH1FT-W/jbl-bar](https://github.com/SH1FT-W/jbl-bar) (Apache-2.0): command names, the `setActiveEQ` payload shape, and the `getPowerState` meaning.
- [MrBearPresident/JBL_Soundbar](https://github.com/MrBearPresident/JBL_Soundbar) (MIT): the client certificate approach.
- [Bubble Card](https://github.com/Clooos/Bubble-Card) for the dashboard.

## License

MIT. Not affiliated with JBL or Harman. Use at your own risk.
