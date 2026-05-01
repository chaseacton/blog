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
+++

I've been running Home Assistant for a while now, and what started as a weekend project to control a few lights has evolved into a comprehensive home automation system that manages everything from security to air quality to 3D printer safety. Here's a deep dive into the automations I've built, how they work, and the design philosophy behind them.

---

## The Philosophy: Automation Should Disappear

The best home automation is the kind you stop thinking about. My goal was never to build a dashboard I stare at — it was to build a system that runs quietly in the background, making my home safer and more comfortable without me having to think about it. After many iterations, I've settled on a stack centered around SimpliSafe for security, UniFi Protect for cameras, Philips Hue for lighting, and a Dyson fan for air quality — all orchestrated through Home Assistant.

---

## Security: A Layered Defense

Security is where I've put the most thought, and it shows in the number of overlapping automations I've built.

### Automatic Alarm Arming

Forgetting to arm the alarm is one of those small anxieties that adds up over time. I solved it with a cascade of scheduled arming automations — at 9 PM, 10 PM, and 11:55 PM — each with a condition that only fires if the alarm is currently disarmed. This means the earliest trigger that catches a disarmed alarm does the job; the later ones are safety nets that never fire unnecessarily. Each sends a push notification to my iPhone confirming the arm.

```yaml
alias: Arm at 9 PM
triggers:
  - trigger: time
    at: '21:00:00'
conditions:
  - condition: device
    type: is_disarmed
actions:
  - type: arm_home
  - action: notify.mobile_app_iphone
    data:
      title: SimpliSafe Armed
      message: Armed to Home
```

On weekday mornings, a separate automation checks at 8:15 AM whether the alarm is still disarmed — a useful reminder if I've left for work and forgotten.

### The Doorbell Fingerprint Chain Reaction

This is probably my favorite automation in the whole system. My UniFi doorbell supports fingerprint scanning, and when it reads a registered print, it fires a webhook to Home Assistant. From there, a single automation triggers a multi-step chain: notify my phone, disarm SimpliSafe, unlock the front door smart lock, and turn on the den light — all simultaneously. Walking up to my front door and having everything just *open* feels genuinely magical.

```yaml
alias: Doorbell Fingerprint Received
triggers:
  - trigger: webhook
    webhook_id: unifi_doorbell_fingerprint
actions:
  - action: notify.mobile_app_iphone
    data:
      message: Fingerprint scanned
  - action: alarm_control_panel.alarm_disarm
  - action: lock.unlock
    target:
      device_id: <front_door_lock>
  - action: light.turn_on
    target:
      entity_id: light.den
```

### Lock the Front Door. Then Lock It Again.

Smart locks are only as good as their reliability, and I don't trust any single automation with something as important as a locked door. The front door is automatically locked at 10 PM, 11 PM, midnight, 2 AM, 4 AM, and 6 AM. Yes, that's six separate lock commands across the night. Is it overkill? Probably. Does it give me peace of mind? Absolutely.

### Unusual Hour Unlock Alerts

If the front door unlocks between 11 PM and 6 AM, I get an immediate push notification with a timestamp. No conditions — if it's unlocked in that window, I want to know about it.

### SimpliSafe RF Jamming Detection

This one I built after reading about how some burglars use RF jammers to defeat wireless alarm systems. SimpliSafe exposes RF jamming state as an attribute in Home Assistant, so I built an automation that watches for it and fires an alert the moment it's detected. Hopefully I never see this notification in the wild.

---

## Lighting: Circadian Rhythm at Scale

### The Wind-Down Sequence

At 9 PM, any bedroom lights that are already on get transitioned to a deep purple at 25% brightness (`hs_color: [278, 95]`, brightness: 64/255). Then, using Home Assistant's built-in `transition` parameter set to 3600 seconds, the lights fade to off over the course of an hour — arriving at full darkness right around 10 PM.

This is one of those automations that required reading the docs carefully. The trick is to first call `light.turn_on` with the color and brightness (transition: 1 second, so it snaps to purple immediately), then immediately call `light.turn_off` with `transition: 3600`. Home Assistant interprets this as a slow fade-to-off from the current state — effectively a 60-minute sunset.

At 10 PM, a backup automation explicitly turns off all bedroom lights (Ami's lamp, Chase's lamp, bedroom overhead, headboard) for anyone who fell asleep before the fade completed.

### Winter Wake-Up Lighting

From October through April, Monday through Thursday, the living room light begins a 30-minute sunrise simulation at 6 AM. It starts at 1% brightness with a warm 2700K color temperature, then transitions to 100% over the next 30 minutes. Waking up to gradually increasing light rather than an alarm is a small quality-of-life improvement that's hard to quantify but genuinely noticeable.

```yaml
alias: Winter Wake-Up Lighting
conditions:
  - condition: template
    value_template: '{{ now().month >= 10 or now().month <= 4 }}'
actions:
  - action: light.turn_on
    data:
      brightness_pct: 1
      color_temp_kelvin: 2700
      transition: 1
  - action: light.turn_on
    data:
      brightness_pct: 100
      transition: 1800
```

### UniFi Person Detection → Night Lights

When UniFi Protect detects a person at night (after sunset, before sunrise), my string lights automatically turn on for 15 minutes, then turn off. The automation uses `mode: restart`, so if another person is detected while the lights are on, the 15-minute timer resets. This doubles as a security deterrent and a practical "someone's home" indicator.

### Daily Light Schedule

A single automation handles the weekday rhythm of the den: lights on at 8:20 AM, off at 6:30 PM. The same automation handles bar lights and the garage light at 10 PM every night. Consolidating related schedules into one automation with a `choose` block makes maintenance much easier than managing a dozen separate time automations.

---

## Air Quality: The Dyson Deep Integration

Living in Oakland, air quality is something I think about — wildfire smoke season is real, and the bedroom PM2.5 sensor I have connected to Home Assistant gives me granular data.

### Automatic AQI Calculation in Jinja2

The EPA's PM2.5-to-AQI formula isn't trivial, but Home Assistant's Jinja2 templating is powerful enough to implement it directly in an automation. I wrote a custom template that computes the AQI from raw PM2.5 readings using the official EPA breakpoints, and use that value in two places: the air quality alert that turns on the Dyson automatically, and the morning summary notification.

```jinja2
{%- set pm25 = states('sensor.bedroom_pm_2_5') | float %}
{%- set ns = namespace(aqi=0) %}
{%- if pm25 <= 12.0 %}
  {%- set ns.aqi = ((50 - 0) / (12.0 - 0) * (pm25 - 0) + 0) | round(0) | int %}
{%- elif pm25 <= 35.4 %}
  {%- set ns.aqi = ((100 - 51) / (35.4 - 12.1) * (pm25 - 12.1) + 51) | round(0) | int %}
...
```

If the AQI crosses 100 and the Dyson is off, the automation turns it on at 60% power, sends a push notification with the exact AQI reading, then steps it down to 30% after 20 minutes. This prevents the fan from running full blast all night when the initial spike passes.

### Air Purifier and Fan Schedule

During the day (10 AM–7 PM), the air purifier runs on a schedule, and the Dyson bedroom fan runs at 40% in "Cool" mode. At 11:30 PM, a safety automation shuts off the Dyson if it's still running — a precaution against leaving a powerful fan running all night unintentionally.

---

## 3D Printing: Safety First

I run OctoPrint for 3D printing, and multi-hour prints create real safety concerns. I've built three automations around the printer:

**Emergency Stop on Smoke/CO**: If any of the eight smoke and CO detectors in the house triggers, OctoPrint immediately cancels the current print job and I get a notification. The automation watches all detectors in parallel — no single point of failure.

**Idle Printer Alert**: If OctoPrint has been idle for 30 minutes but the bed temperature is still above 100°C, I get a notification reminding me the printer is still on. This catches the "walked away after a print finished" scenario that can waste power or create a fire risk.

**Print Completion Notification**: When a print finishes successfully (the automation carefully excludes cancellations by checking `octoprint_current_state`), I get a push notification with the filename of the completed print.

---

## Infrastructure Monitoring: The Unglamorous Essentials

### Leak Detection Everywhere

I have leak sensors under the toilet, water heater, kitchen sink, dishwasher, fridge, laundry, coffee maker, and two bathroom sinks. A single automation watches all of them in `mode: parallel` and sends an immediate alert naming the specific sensor that triggered. Water damage is expensive — this one is pure insurance.

### Pi-hole Monitoring

My Pi-hole DNS ad-blocker runs on a local server, and if it goes down, network behavior degrades noticeably. Every 15 minutes, if the Pi-hole is unavailable, I get a notification with a formatted timestamp. Simple, but useful.

### Low Battery Notifications

Rather than write a custom automation for each battery-powered device, I use the excellent community blueprint from `sbyx` that monitors all battery sensors and sends a single notification listing every device below 25%. One automation to rule them all.

### Morning Summary

Every morning — at 6:30 AM on weekdays, 8:30 AM on weekends — I receive a rich push notification with: bedroom temperature, humidity, computed PM2.5 AQI with color-coded category, outdoor weather and temperature from OpenWeatherMap, and rainfall if applicable. It's the first thing I see on my phone each morning, and it takes zero effort to produce.

---

## Lessons Learned

A few things I'd tell myself if I were starting over:

**Use `mode: restart` for sensor-triggered lights.** If you're turning on lights when a sensor fires, you almost certainly want the timer to reset on subsequent triggers — not queue up a second execution.

**Condition your arming automations.** An alarm that tries to arm itself when it's already armed throws errors and creates noise. Always check current state first.

**One multi-trigger automation beats three single-trigger automations** for related schedules. The `choose` block in Home Assistant is underused and makes intent much clearer.

**Webhooks are the universal integration layer.** The UniFi fingerprint integration isn't natively supported in Home Assistant, but a webhook listener turns any system that can make an HTTP POST into a first-class automation trigger.

**Build notifications into everything while testing, remove them when stable.** I had temporary notifications on every automation when building them — invaluable for debugging, but noisy in production.

---

My Home Assistant config has grown organically, and not everything is perfectly organized. But it does exactly what I want: it runs quietly, handles edge cases, and mostly just disappears into the background of daily life. The best sign that an automation is working is that you forget it exists.

If you're just getting started with Home Assistant, pick one problem to solve — maybe the "did I lock the door?" anxiety — and build from there. The rabbit hole is deep, and the community blueprints at the [Home Assistant Community Store](https://hacs.xyz/) are a great way to get to good automations quickly.

Happy automating.
