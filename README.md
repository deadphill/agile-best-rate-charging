---
layout: post
title: "Mastering Octopus Agile: Cheapest EV Charging with Target Timeframes"
date: 23/02/2026
description: "A beginner-friendly technical walkthrough of UK energy optimisation using Octopus Agile, Target Timeframes, and optional 7-day forecasting."
---

# Mastering Octopus Agile: Charge my EV to take full advantage of the cheap rates within the Octopus Agile rates (typically overnight) using Target Timeframes (and Optional 7-Day Forecasting)

## The problem I am trying to solve

Using Octopus Agile is great. By and large this is the cheapest tariff for me to use for my needs. While I could say your mileage may vary, its unlikely you are going to use something like this if you are not taking advantage of a time of use tariff! However at night I would look to charge my car I would turn on the charger and then just leave it. This means while it was cheap, it could be cheaper! Typically in general use the time period for cheapest charging using agile is overnight but it isn't always that.

This guide attempts to give the option of seeing the rates and then being able to choose the timeslots with which to automate the activation or turning on and off of your device thats providing the load to whatever you are trying to control. It's written as a "show what I did, make it generic" walkthrough — i.e. you can follow use same principle to control **any large electrical load** (immersion heater, storage heaters, etc.) when electricity is cheapest (or even negative).

### Why?

I consider myself to be a nothing more than a tinkerer and this is by far the biggest problem and yet the most rewarding project I have undertaken in home assistant. I am not a coder, and have basic built up knowledge of home assistant and how it all works together. I wanted to share this with everyone making it as foolproof as I could while explaining as much as I could. Many a time have I followed a guide or setup of an integration and not understood what I needed to do because what was needed was inferred, or I just didn't yet know what I needed to know. I hope this helps someone else who like me who sometimes ends up getting frustrated with the setup process and just ends up abandoning the idea.

> ⚠️ **Note on integration versions:** This guide was updated in February 2026 to reflect changes in the BottlecapDave Octopus Energy integration (v17+). Target rates are no longer part of the Octopus Energy integration and have moved to a separate **Target Timeframes** integration. If you followed an older version of this guide, see the migration notes throughout.

## This guide shows you how to **automate EV charging at the cheapest possible Octopus Agile times** using Home Assistant, the **Octopus Energy integration by BottlecapDave**, and the **Target Timeframes integration**.

> **Core idea:** The **Target Timeframes** integration gives you a schedule (`target_times`) of the cheapest half-hours within your chosen window. Once you have that schedule, controlling hardware becomes trivial.

This guide includes:

* ✅ **Core setup**: Cheapest time slots + hardware on/off automation
* ✅ **Dynamic hours**: Calculate how long to charge based on **kWh needed** (works for Tesla & non-Tesla)
* ✅ **Dashboard widgets**: Cost estimate + scheduled slots table
* ⭐ **Optional forecasting layer** (AgilePredict): 7-day context, "deep discount" alerts, etc.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [The Control Challenge: How do we physically switch charging?](#2-the-control-challenge-how-do-we-physically-switch-charging)
3. [The Core Concept: Target Timeframes](#3-the-core-concept-target-timeframes)
4. [Setting up Target Timeframes](#4-setting-up-target-timeframes)
5. [Hardware Control Automation (Charger Switch)](#5-hardware-control-automation-charger-switch)
6. [Dynamic Charging Hours (kWh → hours → cheapest slots)](#6-dynamic-charging-hours-kwh--hours--cheapest-slots)
7. [Dashboard: Charging Summary & Slot Table](#7-dashboard-charging-summary--slot-table)
8. [Optional: 7-Day Forecast Dashboard (AgilePredict)](#8-optional-7day-forecast-dashboard-agilepredict)
9. [Optional: Deep Discount Alert](#9-optional-deep-discount-alert)
10. [Making it Generic: Replace-my-entities table](#10-making-it-generic-replace-my-entities-table)

---

## 1. Prerequisites (and what's optional)

This project has a clean "core" and a few optional add-ons.
The core requirement is simply:

> **A way to control something (charger / appliance) + a way to know Agile prices.**

### 1.1 Core prerequisites (required for the automation to work)

#### ✅ Home Assistant

Installed and running.

#### ✅ Octopus Agile tariff (import)

You need to be on Octopus Agile (or another tariff supported by the integration) so Home Assistant can pull half-hour rates.

#### ✅ BottlecapDave Octopus Energy integration (v17+)

This guide uses the Octopus Energy integration by BottlecapDave.

* Integration: <https://github.com/BottlecapDave/HomeAssistant-OctopusEnergy>

Ensure this integration is set up and running using your account details.

> ⚠️ **Important:** From v17 onwards, target rates are **no longer part of this integration**. They have moved to the separate **Target Timeframes** integration. Do not look for target rate setup within the Octopus Energy integration — it isn't there anymore.

#### ✅ Target Timeframes integration

This is the new home for target rate sensors. Install it via HACS:

* Integration: <https://github.com/bottlecapdave/HomeAssistant-TargetTimeframes>

This integration creates the "brain" entity, e.g.:

* `binary_sensor.target_timeframes_octopus_energy_import_optimal_car_charging`

…and provides attributes like:

* `target_times`
* `hours`
* `overall_average_value`

---

### 1.2 Hardware control prerequisites (one of these is required)

To automate charging, Home Assistant must be able to **turn charging on/off** somehow.
There are several valid approaches — choose the one that matches your hardware.

#### Option A — Smart charger integration (recommended where available)

If you have a charger like Ohme / Zappi / Wallbox (etc.) and Home Assistant can control it using an integration, use that.

#### Option B — Shelly / relay switch control (for dumb/legacy chargers) ✅ common DIY route

If your charger is "dumb" (or an older charger with no integration), you can automate it by switching a relay.

In this guide, I used a **Shelly relay retrofit** inside a legacy charger so Home Assistant can control it like a simple switch.

That creates an entity like:

* `switch.untethered_car_charger`

✅ **Prerequisite if using this method:** Shelly integration in Home Assistant

* Shelly integration docs: <https://www.home-assistant.io/integrations/shelly/>

---

### 1.3 Optional prerequisites (nice-to-have, not essential)

#### ⭐ Car integration (Tesla / others)

If your EV has a Home Assistant integration that provides SoC (state of charge), you can fully automate "how much charge is needed".

> **Tesla tip:** Tesla aggressively sleeps its API. Consider using trigger-based cached template sensors (see Section 6.2) to hold the last known SoC value so your automations don't fail when the car is asleep.

If you don't have a car integration, that's fine — the guide includes manual fallback helpers.

#### ⭐ Companion App (mobile notifications)

If you want notification alerts, you will need the Home Assistant Companion App installed on your phone/tablet.

#### ⭐ Forecasting layer (AgilePredict)

Optional 7-day price prediction. Not required for the core charging automation.

---

### 1.4 Quick checklist

✅ **If you have a smart charger integration** — you can skip Shelly entirely.

✅ **If you have a dumb/legacy charger** — you need some method to switch it (Shelly relay, contactor, etc.).

✅ **If you don't have car telemetry** — use manual helpers (`ev_kwh_override` / `ev_hours_override`) — still works.

✅ **If you don't care about forecasts** — skip AgilePredict — still works.

---

## 2. The Control Challenge: How do we physically switch charging?

The biggest challenge in any "cheapest electricity" automation is not the pricing — it's **controlling the device**.

For EV charging you generally have two routes:

### Option A — Vehicle-side control (software)

If your EV has a supported Home Assistant integration, you can start/stop charging via the car API.

Pros: No hardware mod. Often includes SoC, charge limit, etc.

Cons: API reliability. Some cars/systems sleep aggressively. Integrations vary widely.

### Option B — Charger-side control (hardware) ✅ robust

Control power delivery to the EVSE/charger.

1. **Modern Smart Chargers:** If your charger supports it (Ohme, Zappi, Wallbox, etc.), use its integration.
2. **Shelly Retrofit (legacy hardware):** If you have an older charger, you can retrofit a **Shelly 1** inside the unit.
   * **Reference guide:** <https://www.mgevs.com/threads/adding-a-shelly-wifi-controller-to-a-chargemaster-homecharger.4518/>
   * **Video demo:** <https://www.youtube.com/watch?v=OSiaMJQIXbE>
3. **Busbar Contactor:** Install a contactor to switch the entire charger circuit.

> ⚠️ **Safety note:** Any mains wiring modifications should be carried out safely / by a competent person.

---

## 3. The Core Concept: Target Timeframes

This is the magic.

The **Target Timeframes** integration can create a **Target Timeframe sensor** that:

* looks at Agile half-hour rates
* chooses the cheapest `X` hours inside your chosen time window
* exposes the chosen schedule as `target_times`
* turns a binary sensor **ON during those chosen time slots**

### Example (what this produces)

Entity:

* `binary_sensor.octopus_energy_target_optimal_car_charging` *(after renaming — see Section 4)*

Key attributes:

* `hours: 4`
* `type: Intermittent` *(cheapest slots anywhere in the window)*
* `start_time: 20:00`
* `end_time: 08:00`
* `target_times:` *(list of selected half-hour windows)*
* `overall_average_value:` *(£/kWh average over the chosen slots)*

> ⚠️ **Attribute name change:** If you used the old Octopus Energy target rate integration, note that `overall_average_cost` is now called `overall_average_value`, and `value_inc_vat` on individual slots is now called `value`. Update any templates or dashboard cards accordingly.

---

## 4. Setting up Target Timeframes

### 4.1 Install Target Timeframes via HACS

1. Open HACS → search for **"Target Timeframes"**
2. Install it and restart Home Assistant

### 4.2 Add a Data Source

1. Go to **Settings → Devices & Services → Add Integration**
2. Search for **Target Timeframes**
3. Click **Add Entry** and choose to add a new data source
4. Name it `Octopus Energy Import`

### 4.3 Set up the Blueprint Automation

The data source needs to be fed rate data from the Octopus Energy integration. This is done via a blueprint automation.

1. Go to **Settings → Automations → Blueprints**
2. Find **"Target Timeframes - Octopus Energy source including Free Electricity sessions"**
3. Click **Create Automation** and fill in:
   * **Target timeframe data source sensor:** `Data source last updated (octopus_energy_import)`
   * **Previous day rates:** `event.octopus_energy_electricity_<YOUR_METER_ID>_previous_day_rates`
   * **Current day rates:** `event.octopus_energy_electricity_<YOUR_METER_ID>_current_day_rates`
   * **Next day rates:** `event.octopus_energy_electricity_<YOUR_METER_ID>_next_day_rates`
   * **Free Electricity events:** `event.octopus_energy_<YOUR_ACCOUNT_ID>_octoplus_free_electricity_session_events`
4. Save and run the automation manually once to populate the data source

> ⚠️ **Note:** The `previous_day_rates` entity may not exist until after midnight on your first day. If it doesn't appear, temporarily use `current_day_rates` as a placeholder and update it the next day.

### 4.4 Create Your Target Timeframe

1. Go to **Settings → Devices & Services → Target Timeframes**
2. Click **+ Add Target Timeframe**
3. Fill in:
   * **Name:** `optimal_car_charging`
   * **Hours:** `4` *(the automation in Section 6 will update this dynamically)*
   * **Type:** Intermittent
   * **Minimum time (start):** `20:00`
   * **Maximum time (stop):** `08:00`
   * **When should target times be selected:** `All existing target rates are in the past`
   * Everything else: leave as default

### 4.5 Rename the Entity ID

The new entity will have a long ID like `binary_sensor.target_timeframes_octopus_energy_import_optimal_car_charging`. Rename it to match the shorter ID used throughout this guide:

1. Go to **Settings → Devices & Services → Target Timeframes → 2 entities**
2. Click the entity → gear icon
3. Change the Entity ID to: `binary_sensor.octopus_energy_target_optimal_car_charging`

This means all the automations and dashboard code in this guide will work without modification.

---

## 5. Hardware Control Automation (Charger Switch)

Once you have the target sensor, this is easy:

* when the target sensor turns **ON**, turn the charger ON
* when it turns **OFF**, turn the charger OFF

```yaml
alias: "EV Charging: Optimal Agile Charging (Charger Switch)"
description: Turns the physical charger switch on/off based on Octopus Agile rates
triggers:
  - entity_id: binary_sensor.octopus_energy_target_optimal_car_charging
    from: "off"
    to: "on"
    id: start
    trigger: state
  - entity_id: binary_sensor.octopus_energy_target_optimal_car_charging
    from: "on"
    to: "off"
    id: stop
    trigger: state
actions:
  - choose:
      - conditions:
          - condition: trigger
            id: start
        sequence:
          - action: switch.turn_on
            target:
              entity_id: switch.untethered_car_charger
      - conditions:
          - condition: trigger
            id: stop
        sequence:
          - action: switch.turn_off
            target:
              entity_id: switch.untethered_car_charger
mode: single
```

✅ Replace `switch.untethered_car_charger` with whatever switch/service controls *your* charger.

---

## 6. Dynamic Charging Hours (kWh → hours → cheapest slots)

### Why this matters

A fixed "charge 4 hours" schedule is useful, but the real goal is:

> "Charge **only the amount of energy I need**, and do it in the cheapest slots."

### 6.1 Helpers (universal inputs)

> ⚠️ **Critical:** These helpers MUST be created before the template sensors or update automation will work. Do not skip this step.

Create these in Home Assistant: **Settings → Devices & Services → Helpers → Create Helper → Number**

| Name | Entity ID | Min | Max | Step | Unit |
|---|---|---|---|---|---|
| EV Target SoC | ev_target_soc | 0 | 100 | 1 | % |
| EV Charger kW | ev_charger_kw | 0 | 22 | 0.1 | kW |
| EV Battery Capacity kWh | ev_battery_capacity_kwh | 0 | 150 | 0.1 | kWh |
| EV kWh Override | ev_kwh_override | 0 | 150 | 0.5 | kWh |
| EV Hours Override | ev_hours_override | 0 | 12 | 0.5 | h |

After creating them, set their values:
* `ev_target_soc` → your usual target (e.g. `80`)
* `ev_charger_kw` → your charger speed (e.g. `7`)
* `ev_battery_capacity_kwh` → your car's battery size (e.g. `82` for Tesla Model 3 Long Range)
* `ev_kwh_override` and `ev_hours_override` → leave at `0`

> **Target SoC tip:** 80% is a common daily charging choice to preserve battery health. Set to 100% when you need a full charge. The dashboard in Section 7 lets you change this on the fly.

### 6.2 Template sensors: compute kWh + hours needed

Add this to your `templates.yaml`:

> Replace `sensor.your_car_soc` with your car's SoC entity if you have one.
> If you use Tesla and want sleep-safe cached sensors, use `sensor.phill_car_battery_cached` style trigger-based sensors (see note below).
> If you don't have a car integration, use the override helpers instead.

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
          {% set soc_now = states('sensor.your_car_soc') | float(0) %}
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

> **Tesla sleep tip:** Tesla aggressively sleeps its API, causing SoC sensors to show `unavailable`. Use trigger-based cached sensors in your templates.yaml to hold the last known value:
>
> ```yaml
> - trigger:
>     - platform: state
>       entity_id: sensor.your_tesla_battery_level
>       not_to:
>         - "unknown"
>         - "unavailable"
>   sensor:
>     - name: "Car Battery Cached"
>       unique_id: car_battery_cached
>       unit_of_measurement: "%"
>       device_class: battery
>       state: "{{ states('sensor.your_tesla_battery_level') }}"
> ```
>
> Then use `sensor.car_battery_cached` in your `EV kWh Needed` template instead of the live Tesla entity.

### 6.3 Update the target sensor dynamically

This automation runs at 18:35 (after Agile prices are published for the next day) and tells the Target Timeframes sensor how many hours to schedule tonight.

> ⚠️ **Service name change:** The old `octopus_energy.update_target_config` service no longer exists. The new service is `target_timeframes.update_target_timeframe_config` with different field names.

```yaml
alias: "EV: Update Agile Target Hours"
description: "Sets the Target Timeframes sensor hours based on kWh needed."
triggers:
  - platform: time
    at: "18:35:00"
actions:
  - action: target_timeframes.update_target_timeframe_config
    target:
      entity_id: binary_sensor.octopus_energy_target_optimal_car_charging
    data:
      target_hours: "{{ states('sensor.ev_hours_needed') | float(0) }}"
      target_start_time: "20:00"
      target_end_time: "08:00"
mode: single
```

> **Note:** Times must be in `HH:MM` format (not `HH:MM:SS`) for this service.

---

## 7. Dashboard: Charging Summary & Slot Table

This Markdown card shows:

* status of schedule
* hours + kWh estimate
* average price
* estimated total cost
* list of target charging half-hours
* configurable charging settings
* manual refresh button

Paste this into a Lovelace card (use the raw YAML editor):

```yaml
type: vertical-stack
cards:
  - type: markdown
    title: ⚡ EV Charging Plan
    content: >
      {% set sensor = 'binary_sensor.octopus_energy_target_optimal_car_charging'
      %} {% set target = state_attr(sensor, 'target_times') %} {% set avg_rate =
      state_attr(sensor, 'overall_average_value') | float(0) %} {% set hours =
      state_attr(sensor, 'hours') | float(0) %} {% set charger_kw =
      states('input_number.ev_charger_kw') | float(7.0) %} {% set kwh_needed =
      hours * charger_kw %} {% set total_cost = kwh_needed * avg_rate %}
      <ha-alert alert-type="info" title="Charging Summary">
        Status: **{{ states(sensor) | upper }}** | 
        Need: **{{ hours }}h** ({{ kwh_needed | round(1) }} kWh) | 
        Avg: **{{ (avg_rate * 100) | round(2) }}p**
      </ha-alert>
      <ha-alert alert-type="success">
        **Estimated Cost: £{{ "{:.2f}".format(total_cost) }}**
      </ha-alert>
      ### 📅 Scheduled Slots | Start | End | Price | | :--- | :--- | :--- | {%
      if target %}
        {% for slot in target %}
        | {{ as_datetime(slot.start).strftime('%H:%M') }} | {{ as_datetime(slot.end).strftime('%H:%M') }} | {{ (slot.value * 100) | round(2) }}p |
        {% endfor %}
      {% else %}
        | No | Slots | Scheduled |
      {% endif %}
  - type: entities
    title: ⚙️ Charging Settings
    entities:
      - entity: input_number.ev_target_soc
        name: Target SoC
        icon: mdi:battery-charging
      - entity: input_number.ev_charger_kw
        name: Charger Speed (kW)
        icon: mdi:lightning-bolt
      - entity: input_number.ev_battery_capacity_kwh
        name: Battery Capacity (kWh)
        icon: mdi:battery
      - entity: input_number.ev_kwh_override
        name: kWh Override
        icon: mdi:pencil
      - entity: input_number.ev_hours_override
        name: Hours Override
        icon: mdi:clock-edit
  - type: button
    name: Refresh Charging Plan
    icon: mdi:refresh
    show_icon: true
    show_name: true
    tap_action:
      action: perform-action
      perform_action: automation.trigger
      target:
        entity_id: automation.ev_update_agile_target_hours
    card_mod:
      style: |
        ha-card {
          height: 40px !important;
          --mdc-icon-size: 16px;
        }
        .card-content {
          padding: 0 16px !important;
          height: 40px !important;
          display: flex;
          align-items: center;
          justify-content: center;
          font-size: 12px !important;
        }
  - type: entities
    entities:
      - entity: input_boolean.skip_ev_charging_tonight
        name: Skip Next Session
        icon: mdi:cancel
      - entity: switch.untethered_car_charger
        name: Manual Charger Override
```

> **Note:** The Refresh button requires the **Card Mod** custom integration from HACS to style correctly. The button triggers the update automation on demand so you don't have to wait until 18:35.

> ⚠️ **Attribute name changes from old guide:**
> * `overall_average_cost` → `overall_average_value`
> * `slot.value_inc_vat` → `slot.value`

---

## 8. Optional: 7-Day Forecast Dashboard (AgilePredict)

The Target Timeframes sensor does not require forecasting — it works purely from published Agile prices.

But forecasting is still useful for:

* seeing which day is predicted to be best
* "skip tonight" logic if tomorrow looks significantly cheaper
* confidence and context

### Template sensor to summarise daily min/avg/max

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

---

## 9. Optional: Deep Discount Alert

This alerts you if tonight's scheduled average is much worse than the predicted weekly low.

```yaml
alias: "EV: Deep Discount Alert with Best Day"
description: ""
triggers:
  - platform: time
    at: "17:05:00"
conditions:
  - condition: template
    value_template: >
      {% set tonight = state_attr('binary_sensor.octopus_energy_target_optimal_car_charging',
      'overall_average_value') | float(0) * 100 %}
      {% set weekly_low = states('sensor.agile_weekly_minimum') | float(0) %}
      {{ (tonight - weekly_low) >= 5.0 }}
actions:
  - action: notify.mobile_app_your_phone
    data:
      title: "📉 Better Deal Predicted!"
      message: >
        Tonight's slot is {{ (state_attr('binary_sensor.octopus_energy_target_optimal_car_charging',
        'overall_average_value') | float(0) * 100) | round(1) }}p. A lower price
        of {{ states('sensor.agile_weekly_minimum') }}p is predicted for
        {{ state_attr('sensor.agile_weekly_minimum', 'best_day') }}!
      data:
        tag: ev_price_alert
        actions:
          - action: SKIP_EV_CHARGE
            title: Skip Tonight
mode: single
```

> ⚠️ **Attribute name change:** `overall_average_cost` has been replaced with `overall_average_value` in this updated version.

Replace `notify.mobile_app_your_phone` with your own notification target. Find yours in **Developer Tools → Actions** by searching for `notify`.

---

## 10. Making it Generic: Replace-my-entities table

Here are the key entities in my setup and what you should replace them with.

| My Entity | What it does | Replace with |
| --- | --- | --- |
| `binary_sensor.octopus_energy_target_optimal_car_charging` | "brain" schedule sensor | your renamed target timeframe sensor |
| `switch.untethered_car_charger` | Shelly relay controlling charger | your charger switch |
| `input_number.ev_charger_kw` | charger power in kW | your EVSE power |
| `sensor.your_car_soc` | car state-of-charge | your car's SoC sensor |
| `sensor.agile_predict` | optional forecast source | your forecast entity |
| `notify.mobile_app_your_phone` | push notifications | your notify service |

---

## Migration Notes (from old Octopus Energy target rates)

If you previously followed an older version of this guide that used the built-in Octopus Energy target rate sensors, here's what changed:

| Old | New |
|---|---|
| Target rates configured inside Octopus Energy integration | Separate **Target Timeframes** integration |
| `octopus_energy.update_target_config` service | `target_timeframes.update_target_timeframe_config` service |
| `hours`, `start_time`, `end_time` data fields | `target_hours`, `target_start_time`, `target_end_time` data fields |
| Times in `HH:MM:SS` format | Times in `HH:MM` format |
| `overall_average_cost` attribute | `overall_average_value` attribute |
| `slot.value_inc_vat` attribute | `slot.value` attribute |
