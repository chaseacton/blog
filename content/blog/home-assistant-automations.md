+++
authors = ["Chase Acton"]
title = "How I Automated My Entire Home with Home Assistant (And Why You Should Too)"
date = "2026-04-30"
description = "Security, lighting, air quality, 3D printing safety, and lessons learned from a layered Home Assistant setup."
tags = [
    "home-assistant",
    "automation",
    "smart-home",
]
slug = "home-assistant-automations"
aliases = ["/posts/home-assistant-automations/"]
+++

I've been running Home Assistant for a while now. What started as a weekend project to control a few lights turned into a setup that handles security, air quality, and a few safety nets around the 3D printer. Below is what I actually run, how it works, and why I built it this way.

---

## The Philosophy: Automation Should Disappear

The best home automation is the kind you stop thinking about. I never wanted a wall of tiles to babysit. I wanted something that runs in the background and keeps the house a little safer and more comfortable without me thinking about it. After a lot of tinkering I landed on SimpliSafe for security, UniFi Protect for cameras, Philips Hue for lights, and a Dyson for air quality, all glued together in Home Assistant.

---

## Security: A Layered Defense

Security is where I spent the most time, and it shows in how many overlapping automations ended up in the config.

### Automatic Alarm Arming

Forgetting to arm the alarm is a small thing that still gets under your skin. I use several staggered time triggers in the evening, each gated on the panel still being disarmed. The first one that fires does the work; the rest are backups that usually no-op. Each successful arm sends a push to my phone.

```yaml
# Pattern: repeat trigger: time with different `at` values after dark
alias: Scheduled evening arm (example)
triggers:
  - trigger: time
    at: "HH:MM:SS" # your chosen evening times
conditions:
  - condition: device
    type: is_disarmed
actions:
  - type: arm_home
  - action: notify.mobile_app_your_phone
    data:
      title: Alarm armed
      message: Armed to Home
```

On weekday mornings I run another check before the household is usually gone for the day. If the system is still disarmed, I get a nudge in case someone forgot to arm on the way out.

### The Doorbell Fingerprint Chain Reaction

This is probably my favorite automation in the whole system. My UniFi doorbell can read a registered fingerprint and hit a webhook in Home Assistant. One automation then does the whole routine: phone notification, disarm SimpliSafe, unlock the front lock, turn on a nearby inside light. Walking up and having the door ready without digging for keys still feels good every time.

```yaml
alias: Doorbell fingerprint (example)
triggers:
  - trigger: webhook
    webhook_id: your_webhook_id
actions:
  - action: notify.mobile_app_your_phone
    data:
      message: Fingerprint scanned
  - action: alarm_control_panel.alarm_disarm
  - action: lock.unlock
    target:
      device_id: <front_door_lock>
  - action: light.turn_on
    target:
      entity_id: light.some_inside_light
```

### Lock the Front Door. Then Lock It Again.

Smart locks are only as good as their reliability, and I don't trust any single automation with something as important as a locked door. The front door re-locks on a handful of overnight checkpoints, not just once. Overkill? Probably. Does it help me sleep? Yes.

```yaml
alias: Overnight re-lock (illustrative)
mode: parallel
triggers:
  - trigger: time
    at: "HH:MM:SS"
  - trigger: time
    at: "HH:MM:SS"
  # one trigger: time per checkpoint you want overnight
actions:
  - action: lock.lock
    target:
      entity_id: lock.front_door
```

### Unusual Hour Unlock Alerts

If the front door unlocks during the hours I treat as "should be quiet," I get an immediate push with a timestamp. No extra conditions. If it unlocks in that window, I want to know.

```yaml
alias: Night unlock alert (illustrative)
triggers:
  - trigger: state
    entity_id: lock.front_door
    to: unlocked
conditions:
  - condition: template
    value_template: >
      {% set h = now().hour %}
      {{ h >= 23 or h < 6 }}
actions:
  - action: notify.mobile_app_your_phone
    data:
      title: Front door unlocked
      message: "Unlocked overnight (add a template for exact time if you want)"
```

In the real automation I keep the time window in a template so I can tweak it without touching four separate conditions.

### SimpliSafe RF Jamming Detection

I added this after reading about burglars using RF jammers against wireless alarms. SimpliSafe exposes jamming state in Home Assistant, so I have an automation that fires the moment it sees a jam. I hope that notification never fires for real.

```yaml
alias: RF jamming alert (illustrative)
triggers:
  - trigger: state
    entity_id: binary_sensor.simplisafe_jamming
    to: "on"
actions:
  - action: notify.mobile_app_your_phone
    data:
      title: Possible RF jamming
      message: Check the alarm panel and connectivity.
```

Entity id and attribute names differ by integration version; the idea is a binary or attribute flip you can key off of.

---

## Lighting: Circadian Rhythm at Scale

### The Wind-Down Sequence

In the evening, any bedroom lights that are already on get transitioned to a deep purple at low brightness (`hs_color: [278, 95]`, brightness scaled down). Then I use Home Assistant's `transition` for a long fade so they drift off over roughly an hour instead of snapping off.

This one took some reading. The sequence is `light.turn_on` with the color and a short transition so it snaps to purple, then `light.turn_off` with a long `transition`. Home Assistant treats that as a long fade from the current state, which works out like a slow sunset.

A later backup automation force-clears the bedroom group for anyone who fell asleep before the fade finished.

```yaml
alias: Bedroom wind-down (illustrative)
triggers:
  - trigger: time
    at: "HH:MM:SS"
conditions:
  - condition: state
    entity_id: group.bedroom_lights
    state: "on"
actions:
  - action: light.turn_on
    target:
      entity_id: group.bedroom_lights
    data:
      hs_color: [278, 95]
      brightness: 64
      transition: 1
  - action: light.turn_off
    target:
      entity_id: group.bedroom_lights
    data:
      transition: 3600
```

### Winter Wake-Up Lighting

In the cooler months, on the weekdays I care about, the living room light runs a gradual sunrise over a fixed window before the household is usually up. It starts barely on at a warm color temp and ramps to full over the same window. I still use an alarm sometimes, but the light change alone is a noticeable upgrade.

```yaml
alias: Seasonal wake lighting (example)
conditions:
  - condition: template
    value_template: "{{ now().month >= 10 or now().month <= 4 }}"
actions:
  - action: light.turn_on
    data:
      brightness_pct: 1
      color_temp_kelvin: 2700
      transition: 1
  - action: light.turn_on
    data:
      brightness_pct: 100
      transition: 1800 # half of total ramp; tune to taste
```

### UniFi Person Detection → Night Lights

When UniFi Protect sees a person at night (after sunset, before sunrise), my string lights turn on for a bounded period, then turn off. The automation uses `mode: restart`, so if another person shows up while the timer is running, the timer starts over. Good for security and also a clear "someone is outside" signal.

```yaml
alias: Night person path lights (illustrative)
mode: restart
triggers:
  - trigger: state
    entity_id: binary_sensor.doorbell_person
    to: "on"
conditions:
  - condition: sun
    after: sunset
  - condition: sun
    before: sunrise
actions:
  - action: light.turn_on
    target:
      entity_id: light.patio_string
  - delay:
      minutes: 15
  - action: light.turn_off
    target:
      entity_id: light.patio_string
```

### Daily Light Schedule

One automation handles the weekday rhythm for a main living space (morning on, evening off) and a separate late-evening pass for a couple of circuits I do not want on all night. Packing related schedules into one `choose` block is easier to maintain than a pile of separate time triggers.

```yaml
alias: Weekday lights schedule (illustrative)
triggers:
  - trigger: time
    at: "HH:MM:SS"
    id: morning_on
  - trigger: time
    at: "HH:MM:SS"
    id: evening_off
actions:
  - choose:
      - conditions:
          - condition: time
            weekday:
              - mon
              - tue
              - wed
              - thu
              - fri
          - condition: template
            value_template: "{{ trigger.id == 'morning_on' }}"
        sequence:
          - action: light.turn_on
            target:
              entity_id: light.den
      - conditions:
          - condition: template
            value_template: "{{ trigger.id == 'evening_off' }}"
        sequence:
          - action: light.turn_off
            target:
              entity_id: light.den
```

In a real file each `trigger: time` gets its own `id` (shown above) so the `choose` branch can key off `trigger.id` reliably.

---

## Air Quality: The Dyson Deep Integration

I live in Oakland, so smoke season is on my mind. I have a bedroom PM2.5 sensor in Home Assistant and I actually look at the numbers.

### Automatic AQI Calculation in Jinja2

The EPA PM2.5 to AQI math is annoying on paper, but Jinja2 in Home Assistant can handle it. I wrote a template that walks the official breakpoints from the raw PM2.5 reading, then feed that value into the Dyson-on alert and the morning summary. The pattern is the same for every concentration band: pick the AQI index range that matches the PM2.5 interval, then linearly interpolate between the low and high concentration endpoints for that band (the EPA publishes the exact µg/m³ cutpoints and formulas).

```jinja2
{%- set pm25 = states('sensor.bedroom_pm_2_5') | float %}
{%- set ns = namespace(aqi=0) %}
{# Good: 0–12.0 µg/m³ → AQI 0–50 #}
{%- if pm25 <= 12.0 %}
  {%- set ns.aqi = ((50 - 0) / (12.0 - 0) * (pm25 - 0) + 0) | round(0) | int %}
{# Moderate: 12.1–35.4 → AQI 51–100 #}
{%- elif pm25 <= 35.4 %}
  {%- set ns.aqi = ((100 - 51) / (35.4 - 12.1) * (pm25 - 12.1) + 51) | round(0) | int %}
{# Repeat for USG, Unhealthy, Very unhealthy, Hazardous using current EPA cutpoints #}
{%- else %}
  {%- set ns.aqi = 500 %}
{%- endif %}
{{ ns.aqi }}
```

The last line is what you feed into a `sensor.template` or an automation `variables` block. Fill in the remaining `elif` branches from the published breakpoint table instead of the placeholder `else` above.

If the AQI crosses a threshold I picked and the Dyson is off, the automation turns it on at a moderate level, sends a push with the reading, then eases it down after a delay so it does not sit at full blast all night after a short spike.

```yaml
alias: AQI-driven bedroom fan (illustrative)
triggers:
  - trigger: numeric_state
    entity_id: sensor.calculated_pm25_aqi
    above: 100
conditions:
  - condition: state
    entity_id: fan.bedroom_purifier
    state: "off"
actions:
  - action: fan.set_percentage
    target:
      entity_id: fan.bedroom_purifier
    data:
      percentage: 60
  - action: notify.mobile_app_your_phone
    data:
      title: Air quality
      message: "AQI is {{ states('sensor.calculated_pm25_aqi') }}; fan on"
  - delay:
      minutes: 20
  - action: fan.set_percentage
    target:
      entity_id: fan.bedroom_purifier
    data:
      percentage: 30
```

### Air Purifier and Fan Schedule

During daytime hours the air purifier follows a schedule, and the bedroom fan sits at a fixed moderate level in Cool mode. Before bed I have a safety automation that turns the fan off if it is still running, so I do not leave a strong fan on all night by accident.

```yaml
alias: Fan off before bed (illustrative)
triggers:
  - trigger: time
    at: "HH:MM:SS"
conditions:
  - condition: state
    entity_id: fan.bedroom_purifier
    state: "on"
actions:
  - action: fan.turn_off
    target:
      entity_id: fan.bedroom_purifier
```

---

## 3D Printing: Safety First

I run OctoPrint, and long prints are a real fire and power concern. Three automations cover it:

**Emergency stop on smoke or CO:** If any of the smoke or CO detectors trip, OctoPrint cancels the current job and I get a notification. The automation watches every detector in parallel so one bad sensor does not block the rest.

**Idle printer alert:** If OctoPrint has been idle for a while but the bed is still over a temperature I consider "too hot to ignore," I get a reminder that the printer is on. That covers the classic "print finished, I walked away" case.

**Print finished:** When a job completes successfully (the automation checks `octoprint_current_state` so cancellations do not spam me), I get a push with the filename.

```yaml
alias: OctoPrint cancel on alarm (illustrative)
mode: parallel
triggers:
  - trigger: state
    entity_id:
      - binary_sensor.smoke_living
      - binary_sensor.smoke_bedroom
    to: "on"
actions:
  - action: octoprint.cancel_job
    target:
      entity_id: octoprint.octoprint
  - action: notify.mobile_app_your_phone
    data:
      title: Printer stopped
      message: Smoke or CO tripped; print canceled.
```

```yaml
alias: OctoPrint hot idle bed (illustrative)
triggers:
  - trigger: numeric_state
    entity_id: sensor.octoprint_actual_bed_temp
    above: 100
    for:
      minutes: 30
conditions:
  - condition: template
    value_template: "{{ states('sensor.octoprint_current_state') == 'Operational' }}"
actions:
  - action: notify.mobile_app_your_phone
    data:
      message: Bed still hot while printer looks idle; check power.
```

```yaml
alias: OctoPrint job finished notify (illustrative)
triggers:
  - trigger: state
    entity_id: sensor.octoprint_print_state
    to: Done
conditions:
  - condition: template
    value_template: "{{ trigger.from_state.state != 'Cancelled' }}"
actions:
  - action: notify.mobile_app_your_phone
    data:
      message: "Print finished: {{ state_attr('sensor.octoprint_print_state', 'filename') }}"
```

Your OctoPrint integration entity ids will differ; swap in the sensors and `octoprint.*` entity your install exposes.

---

## Infrastructure Monitoring: The Unglamorous Essentials

### Leak Detection Everywhere

Leak sensors sit in the usual high-risk spots (baths, kitchen, appliances, utility areas). One automation watches all of them in `mode: parallel` and names the sensor in the alert. Water damage is expensive; this is cheap insurance.

```yaml
alias: Leak sensor notify (illustrative)
mode: parallel
triggers:
  - trigger: state
    entity_id:
      - binary_sensor.leak_kitchen
      - binary_sensor.leak_bath
      - binary_sensor.leak_water_heater
    to: "on"
actions:
  - action: notify.mobile_app_your_phone
    data:
      title: Water leak
      message: "{{ trigger.to_state.name }} reported water"
```

### Pi-hole Monitoring

Pi-hole runs on a local box. If it stops answering DNS, the house feels broken fast. On a fixed interval, if Pi-hole is down, I get a notification with a timestamp.

```yaml
alias: Pi-hole deadman (illustrative)
triggers:
  - trigger: time_pattern
    minutes: "/15"
conditions:
  - condition: template
    value_template: "{{ states('binary_sensor.pi_hole_running') != 'on' }}"
actions:
  - action: notify.mobile_app_your_phone
    data:
      message: "Pi-hole probe failed at {{ now().isoformat() }}"
```

### Low Battery Notifications

Instead of one automation per device, I use the community blueprint from `sbyx` that rolls every battery sensor below a threshold into a single message.

### Morning Summary

On a different schedule for weekdays versus weekends, I get a push with indoor temp and humidity, the PM2.5 AQI with a color bucket, OpenWeatherMap for outside temp and rain when it matters. First glance at the phone, no manual assembly.

The automation is one `notify` with a **templated `message`** (and two `trigger: time` entries with different `id` values so a template condition can split weekday vs weekend). The body is mostly `states(...)` and `state_attr('weather.home', ...)` calls; the AQI line reuses the same template sensor the Dyson automation uses.

```yaml
alias: Morning summary push (illustrative)
triggers:
  - trigger: time
    at: "07:00:00"
    id: weekday
  - trigger: time
    at: "09:00:00"
    id: weekend
conditions:
  - condition: template
    value_template: >
      {% if trigger.id == 'weekday' %}
        {{ now().weekday() < 5 }}
      {% else %}
        {{ now().weekday() >= 5 }}
      {% endif %}
actions:
  - action: notify.mobile_app_your_phone
    data:
      title: Morning
      message: >-
        Inside {{ states('sensor.bedroom_temperature') }}°F ·
        RH {{ states('sensor.bedroom_humidity') }}% ·
        AQI {{ states('sensor.calculated_pm25_aqi') }}
        ({{ states('sensor.aqi_category') }}) ·
        Outside {{ state_attr('weather.home', 'temperature') }}°F
        {% set fc = state_attr('weather.home', 'forecast') %}
        {% if fc %}
        · Today {{ fc[0].condition }}
        {% endif %}
```

Swap entity ids for your indoor sensors, template sensor name for AQI, and `weather.*` entity for whatever forecast integration you use. If your mobile notify platform needs `data` structured differently (for example images or critical alerts), keep the same template string and move it behind a `variables` + `message: "{{ summary }}"` pattern once the template validates in **Developer tools → Template**.

---

## Lessons Learned

If I started again I would still do most of this, but a few choices saved headaches:

**Use `mode: restart` for sensor-driven lights.** If a motion event should extend a timer, you want the timer to reset, not stack duplicate runs.

**Gate your arm commands.** Arming when already armed throws errors and noise. Check state first.

**Prefer one automation with `choose` over three copies** when the schedules are related. Easier to read later.

**Webhooks fix weird gaps.** UniFi fingerprint is not a first-class Home Assistant integration for me, but anything that can POST can hit a webhook and become a trigger.

**Spam yourself while debugging, then strip notifications.** Temporary pushes on every new automation made tuning faster, then I peeled them back once things were stable.

---

The config is messy in places, but it does what I care about: it stays quiet, handles a few edge cases, and mostly stays out of the way. When I stop noticing an automation, that usually means it is doing its job.

If you are new to Home Assistant, pick one annoyance (for me it was "did I lock the door?") and solve that first. The [Home Assistant Community Store](https://hacs.xyz/) blueprints are a fast way to borrow patterns that already work.

That is the setup. Good luck with yours.
