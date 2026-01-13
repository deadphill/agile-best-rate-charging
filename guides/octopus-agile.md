---
layout: post
title: "Mastering Octopus Agile: Cheapest EV Charging with Target Timeframes (and Optional 7-Day Forecasting)"
date: 13/01/2026
description: "A beginner-friendly technical walkthrough of UK energy optimisation using Octopus Agile, BottlecapDave Target Rate Timeframes, and optional 7-day forecasting."
---

# Mastering Octopus Agile: Cheapest EV Charging with Target Timeframes (and Optional 7-Day Forecasting)

This guide shows you how to **automate EV charging at the cheapest possible Octopus Agile times** using Home Assistant and the **Octopus Energy integration by BottlecapDave**.

It’s written as a “show what I did, make it generic” walkthrough — i.e. you can follow the same pattern to control **any large electrical load** (immersion heater, dishwasher, storage heaters, etc.) when electricity is cheapest (or even negative).

> **Core idea:** BottlecapDave’s **Target Rate / Target Timeframes** feature gives you a schedule (`target_times`) of the cheapest half-hours. Once you have that schedule, controlling hardware becomes trivial.

This guide includes:
- ✅ **Core setup**: Cheapest time slots + hardware on/off automation
- ✅ **Dynamic hours**: Calculate how long to charge based on **kWh needed** (works for Tesla & non-Tesla)
- ✅ **Dashboard widgets**: Cost estimate + scheduled slots table
- ⭐ **Optional forecasting layer** (AgilePredict): 7-day context, “deep discount” alerts, etc.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)  
2. [The Control Challenge: How do we physically switch charging?](#2-the-control-challenge-how-do-we-physically-switch-charging)  
3. [The Core Concept: Target Rate Timeframes](#3-the-core-concept-target-rate-timeframes)  
4. [Hardware Control Automation (Charger Switch)](#4-hardware-control-automation-charger-switch)  
5. [Dynamic Charging Hours (kWh → hours → cheapest slots)](#5-dynamic-charging-hours-kwh--hours--cheapest-slots)  
6. [Dashboard: Charging Summary & Slot Table](#6-dashboard-charging-summary--slot-table)  
7. [Optional: 7-Day Forecast Dashboard (AgilePredict)](#7-optional-7day-forecast-dashboard-agilepredict)  
8. [Optional: Deep Discount Alert](#8-optional-deep-discount-alert)  
9. [Making it Generic: Replace-my-entities table](#9-making-it-generic-replace-my-entities-table)  

---

## 1. Prerequisites

- **Home Assistant** installed and running
- **Octopus Agile tariff** (import)
- **BottlecapDave Octopus Energy integration**  
  GitHub: https://github.com/BottlecapDave/HomeAssistant-OctopusEnergy

Optional:
- **AgilePredict API sensor** (REST sensor pulling predicted future prices)  
  Website: https://agilepredict.com/

---

## 2. The Control Challenge: How do we physically switch charging?

The biggest challenge in any “cheapest electricity” automation is not the pricing — it’s **controlling the device**.

For EV charging you generally have two routes:

### Option A — Vehicle-side control (software)
If your EV has a supported Home Assistant integration, you can start/stop charging via the car API.

Pros:
- No hardware mod
- Often includes SoC, charge limit, etc.

Cons:
- API reliability
- Some cars/systems sleep aggressively
- Integrations vary widely

### Option B — Charger-side control (hardware) ✅ robust
Control power delivery to the EVSE/charger.

This is the most reliable approach for “dumb” chargers and legacy hardware.

1) **Modern Smart Chargers:**  
If your charger supports it (Ohme, Zappi, Wallbox, etc.), use its integration.

2) **Shelly Retrofit (legacy hardware):**  
If you have an older charger (e.g. **Chargemaster**), you can retrofit a **Shelly 1** inside the unit.

- **Method:** interrupt the **Control Pilot (CP)** wire so the car pauses/resumes safely  
- **Reference guide:** https://github.com/p-shane/chargemaster-shelly-mod  
- **Video demo:** https://www.youtube.com/watch?v=OSiaMJQIXbE  

3) **Busbar Contactor:**  
Install a contactor to switch the entire charger circuit (fuseboard).

> ⚠️ **Safety note:** Any mains wiring modifications should be carried out safely / by a competent person.

---

## 3. The Core Concept: Target Rate Timeframes

This is the magic.

BottlecapDave’s integration can create a **Target Rate sensor** that:
- looks at Agile half-hour rates
- chooses the cheapest `X` hours inside your chosen time window
- exposes the chosen schedule as `target_times`
- turns a binary sensor **ON during those chosen time slots**

### Example (what this produces)

Entity:
- `binary_sensor.octopus_energy_target_optimal_car_charging`

Key attributes:
- `hours: 4`
- `type: Intermittent` *(cheapest slots anywhere in the window)*
- `start_time: 20:00`
- `end_time: 08:00`
- `target_times:` *(list of selected half-hour windows)*
- `overall_average_cost:` *(£/kWh average over the chosen slots)*

This means:
> Home Assistant now knows **exactly** when electricity is cheapest *for your configured requirement*.

---

## 4. Hardware Control Automation (Charger Switch)

Once you have the target sensor, this is easy:

- when the target sensor turns **ON**, turn the charger ON
- when it turns **OFF**, turn the charger OFF

```yaml
alias: "EV Charging: Optimal Agile Charging (Charger Switch)"
description: Turns the physical charger switch on/off based on Octopus Agile rates
trigger:
  - platform: state
    entity_id: binary_sensor.octopus_energy_target_optimal_car_charging
    from: "off"
    to: "on"
    id: start
  - platform: state
    entity_id: binary_sensor.octopus_energy_target_optimal_car_charging
    from: "on"
    to: "off"
    id: stop
action:
  - choose:
      - conditions:
          - condition: trigger
            id: start
        sequence:
          - service: switch.turn_on
            target:
              entity_id: switch.untethered_car_charger
      - conditions:
          - condition: trigger
            id: stop
        sequence:
          - service: switch.turn_off
            target:
              entity_id: switch.untethered_car_charger
mode: single
```
✅ In my setup `switch.untethered_car_charger` is a Shelly relay I retrofitted into a legacy charger.  
➡️ You should replace this with whatever switch/service controls *your* charger.

---

## 5. Dynamic Charging Hours (kWh → hours → cheapest slots)

### Why this matters
A fixed “charge 4 hours” schedule is useful, but the real goal is:

> “Charge **only the amount of energy I need** (to 80% or 100%), and do it in the cheapest slots.”

This section makes the solution generic:
- Tesla
- Non-Tesla
- Smart charger
- Dumb charger
- Different battery chemistries and manufacturer guidance

### 5.1 Helpers (universal inputs)
Create these in Home Assistant:  
**Settings → Devices & Services → Helpers**

- `input_number.ev_target_soc` (0–100, default **80**)  
- `input_number.ev_charger_kw` (default **7.0**)  
- `input_number.ev_battery_capacity_kwh` (e.g. 60–100 depending on car)  
- `input_number.ev_kwh_override` (default **0**) *(manual fallback)*  
- `input_number.ev_hours_override` (default **0**) *(manual fallback)*  

> **Target SoC:** 80% is a common daily charging choice.  
> Some battery chemistries/manufacturers recommend charging to **100%** more regularly.  
> That’s why `ev_target_soc` is configurable.

### 5.2 Template sensors: compute kWh + hours needed
Add this to your `templates.yaml` (or similar template include).

> Replace `sensor.car_soc` with your car’s SoC entity if you have one.  
> If you don’t, use the override helpers (still works perfectly).

```yaml
- sensor:
    - name: "EV kWh Needed"
      unique_id: ev_kwh_needed
      unit_of_measurement: "kWh"
      state: >
        {% set override = states('input_number.ev_kwh_override') | float(0) %}
        {% if override > 0 %}
          {{ override | round(1) }}
        {% else %}
          {# Replace with your own car SoC entity if available #}
          {% set soc_now = states('sensor.car_soc') | float(0) %}
          {% set soc_target = states('input_number.ev_target_soc') | float(80) %}
          {% set capacity = states('input_number.ev_battery_capacity_kwh') | float(0) %}

          {% if capacity > 0 and soc_target > soc_now %}
            {{ (((soc_target - soc_now) / 100) * capacity) | round(1) }}
          {% else %}
            0
          {% endif %}
        {% endif %}

    - name: "EV Hours Needed"
      unique_id: ev_hours_needed
      unit_of_measurement: "h"
      state: >
        {% set h_override = states('input_number.ev_hours_override') | float(0) %}
        {% if h_override > 0 %}
          {{ ((h_override * 2) | round(0) / 2) }}
        {% else %}
          {% set kwh = states('sensor.ev_kwh_needed') | float(0) %}
          {% set kw = states('input_number.ev_charger_kw') | float(7.0) %}
          {% if kw > 0 %}
            {% set raw = kwh / kw %}
            {# round UP to nearest 0.5h because Agile is half-hour slots #}
            {{ ((raw * 2) | round(0, 'ceil') / 2) }}
          {% else %}
            0
          {% endif %}
        {% endif %}
```

### 5.3 Update the target sensor dynamically (BottlecapDave Action)

Home Assistant action/service (as shown in **Developer Tools → Actions**):

- **Action:** `octopus_energy.update_target_config`

This lets Home Assistant say:

> “Target sensor, tonight you need 2.5h of charging, not 4h.”

Create this automation:

```yaml
alias: "EV: Update Agile Target Hours"
description: "Sets the BottlecapDave target-rate sensor hours based on kWh needed."
trigger:
  - platform: time
    at: "18:35:00"   # after Agile prices are published
action:
  - service: octopus_energy.update_target_config
    target:
      entity_id: binary_sensor.octopus_energy_target_optimal_car_charging
    data:
      hours: "{{ states('sensor.ev_hours_needed') | float(0) }}"
      start_time: "20:00:00"
      end_time: "08:00:00"
      persist_changes: true
mode: single
```

## 6. Dashboard: Charging Summary & Slot Table

This Markdown card shows:
- status of schedule
- hours + kWh estimate
- average price
- estimated total cost
- list of target charging half-hours

Paste this into a Lovelace Markdown card:

```yaml
type: markdown
content: >-
  {% set sensor = 'binary_sensor.octopus_energy_target_optimal_car_charging' %}
  {% set target = state_attr(sensor, 'target_times') %}
  {% set avg_rate = state_attr(sensor, 'overall_average_cost') | float(0) %}
  {% set hours = state_attr(sensor, 'hours') | float(0) %}
  {% set charger_kw = states('input_number.ev_charger_kw') | float(7.0) %}
  {% set kwh_needed = hours * charger_kw %}
  {% set total_cost = kwh_needed * avg_rate %}

  <ha-alert alert-type="info" title="Charging Summary">
    Status: **{{ states(sensor) | upper }}** |
    Need: **{{ hours }}h** ({{ kwh_needed | round(1) }} kWh) |
    Avg: **{{ (avg_rate * 100) | round(2) }}p**
  </ha-alert>

  <ha-alert alert-type="success">
    **Estimated Cost: £{{ "{:.2f}".format(total_cost) }}**
  </ha-alert>

  ### 📅 Scheduled Slots
  | Start | End | Price |
  | :--- | :--- | :--- |
  {% if target %}
    {% for slot in target %}
    | {{ as_datetime(slot.start).strftime('%H:%M') }} | {{ as_datetime(slot.end).strftime('%H:%M') }} | {{ (slot.value_inc_vat * 100) | round(2) }}p |
    {% endfor %}
  {% else %}
    | No | Slots | Scheduled |
  {% endif %}
```

## 7. Optional: 7-Day Forecast Dashboard (AgilePredict)

The Target Rate sensor does not require forecasting — it works purely from published Agile prices.

But forecasting is still useful for:
- seeing which day is predicted to be best
- “skip tonight” logic if tomorrow looks significantly cheaper
- confidence and context

### Template sensor to summarise daily min/avg/max

Once you have:

- `sensor.agile_predict`
- with `prices[]` containing `date_time` and `agile_pred`

Add this to `templates.yaml`:
```yaml
- sensor:
    - name: "Agile Forecast Summary"
      unique_id: agile_forecast_summary
      state: >
        {% set prices = state_attr('sensor.agile_predict', 'prices') %}
        {{ 'Ready' if prices and prices | length > 0 else 'Unavailable' }}
      attributes:
        daily_data: >
          {% set agile_predict = state_attr('sensor.agile_predict', 'prices') %}
          {% if agile_predict %}
            {% set ns = namespace(days={}) %}
            {% for rate in agile_predict %}
              {% set time_obj = as_datetime(rate.date_time) %}
              {% if time_obj %}
                {% set date_key = time_obj.strftime('%Y-%m-%d') %}
                {% if date_key not in ns.days %}
                  {% set ns.days = dict(ns.days, **{date_key: []}) %}
                {% endif %}
                {% set ns.days = dict(ns.days, **{date_key: ns.days[date_key] + [rate.agile_pred | float]}) %}
              {% endif %}
            {% endfor %}
            {% set output = [] %}
            {% set sorted_keys = ns.days.keys() | sort %}
            {% for date in sorted_keys %}
              {% set day_prices = ns.days[date] %}
              {% set item = {
                'date': date,
                'min': day_prices | min | round(1),
                'max': day_prices | max | round(1),
                'avg': (day_prices | sum / day_prices | length) | round(1)
              } %}
              {% set output = output + [item] %}
            {% endfor %}
            {{ output | to_json }}
          {% else %}
            []
          {% endif %}
```

### Markdown display card for the forecast

Paste this into a Lovelace Markdown card:
```yaml
type: markdown
content: >-
  | Day | Min | Avg | Max |
  |:---|:---:|:---:|:---:|
  {% set data = state_attr('sensor.agile_forecast_summary', 'daily_data') | from_json %}
  {% for day in data[:7] %}
  | {{ day.date }} | {{ day.min }}p | **{{ day.avg }}p** | {{ day.max }}p |
  {% endfor %}
```

## 8. Optional: Deep Discount Alert

This alerts you if tonight’s scheduled average is much worse than the predicted weekly low.

It compares:
- `binary_sensor.octopus_energy_target_optimal_car_charging` → `overall_average_cost`
- your forecast “weekly minimum” sensor

```yaml
alias: "EV: Deep Discount Alert with Best Day"
description: ""
trigger:
  - platform: time
    at: "17:05:00"
condition:
  - condition: template
    value_template: >
      {% set tonight = state_attr('binary_sensor.octopus_energy_target_optimal_car_charging', 'overall_average_cost') | float(0) * 100 %}
      {% set weekly_low = states('sensor.agile_predict_weekly_minimum') | float(0) %}
      {{ (tonight - weekly_low) >= 5.0 }}
action:
  - service: notify.mobile_app_flip_phone
    data:
      title: "📉 Better Deal Predicted!"
      message: >
        Tonight's slot is {{
        (state_attr('binary_sensor.octopus_energy_target_optimal_car_charging',
        'overall_average_cost') | float(0) * 100) | round(1) }}p.
        A lower price of {{ states('sensor.agile_predict_weekly_minimum') }}p is predicted for
        {{ state_attr('sensor.agile_predict_weekly_minimum', 'best_day') }}!
      data:
        tag: ev_price_alert
        actions:
          - action: SKIP_EV_CHARGE
            title: Skip Tonight
mode: single
```
### About `notify.mobile_app_flip_phone` (what this actually means)

Home Assistant doesn’t “magically know” where to send a notification — it needs a **notification target**.

If you use the **Home Assistant Companion App** (Android or iOS), Home Assistant will create a notification service for *each device* that has the Companion App logged into your Home Assistant instance.

In Home Assistant terms, a **device** here usually means:

#### ✅ Most common: your mobile phone
- Android phone
- iPhone

This is the typical use case. You install the Companion App on your phone, log in, and Home Assistant creates a service like:
- `notify.mobile_app_my_phone`
- `notify.mobile_app_iphone`
- `notify.mobile_app_pixel_8`

In my setup, my phone happens to be called **“Flip Phone”**, so the notify service is:
- `notify.mobile_app_flip_phone`

> It is *not* referring to an actual old-school flip phone — it’s just the friendly name given to the device in Home Assistant.

#### ✅ Also common: a tablet / wall-mounted dashboard panel
Some people keep a tablet on the wall running Home Assistant dashboards.
If that tablet is running the Companion App and logged in, it can receive push notifications too.

Examples:
- `notify.mobile_app_kitchen_tablet`
- `notify.mobile_app_hallway_panel`

#### ✅ Less common: multiple phones (shared household setup)
If more than one person uses Home Assistant:
- your phone
- your partner’s phone
- a child’s phone

Home Assistant will create notification services for each one, e.g.:
- `notify.mobile_app_my_phone`
- `notify.mobile_app_partners_phone`
- `notify.mobile_app_family_tablet`

You can then choose who gets which alerts.

#### ✅ Even less common: work devices
If you’ve logged in on a work phone/tablet, it can also appear.
(Just be mindful of privacy and what notifications you want showing up.)

---

### What else could notifications be (beyond the Companion App)?

The `notify.mobile_app_*` services are specific to the **Mobile App integration**, but Home Assistant supports many other notification destinations too, such as:

- Email notifications
- Telegram / WhatsApp / Discord
- Alexa / Google Home announcements
- Persistent notifications inside Home Assistant
- Text-to-speech announcements over speakers

So if you’re not using the Companion App, this section still applies — you’d just change the notify service to something else.

---

### How to find your own notify service name

There are a couple of easy ways:

### How to find your own notify service name

There are a couple of easy ways:

#### Option 1 (recommended): Developer Tools → Actions
1) Go to **Developer Tools → Actions**
2) Search for: `notify`
3) You’ll see a list of available notification services
4) Pick the one that matches your phone/tablet

#### Option 2: Settings → Devices & services
1) Go to **Settings → Devices & services**
2) Find **Mobile App**
3) Click it and look at the devices listed
4) Each device usually corresponds to a `notify.mobile_app_<device_name>` service

---

### What to change in this guide

Anywhere you see:

```yaml
service: notify.mobile_app_flip_phone
```
Replace it with your notification target, for example:
```yaml
service: notify.mobile_app_my_phone
```
If you’d like the notification to go to multiple devices, you can:

-duplicate the action, or

-use a notify group (advanced), or

-use another integration like Telegram.

---
Why this matters in this project

This alert is designed as “human-in-the-loop” decision support.

Your charging automation can remain fully automatic — but the notification gives you a chance to intervene when the forecast suggests:

“Tonight’s slots aren’t great — a better night is coming soon.”

That’s why notifications are included as an optional step.

## 9. Making it Generic: Replace-my-entities table

Here are the key entities in my setup and what you should replace them with.
| My Entity | What it does | Replace with |
|---|---|---|
| `binary_sensor.octopus_energy_target_optimal_car_charging` | “brain” schedule sensor | your target sensor |
| `switch.untethered_car_charger` | Shelly relay controlling charger | your charger switch |
| `input_number.ev_charger_kw` | charger power in kW | your EVSE power |
| `sensor.car_soc` | car state-of-charge | your car’s SoC sensor |
| `sensor.agile_predict` | optional forecast source | your forecast entity |
