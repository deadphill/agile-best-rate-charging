---
layout: post
title: "Mastering Octopus Agile: 7-Day Forecasting & Hardware Automation"
date: 13/01/2026
description: "A beginner-friendly technical walkthrough of UK energy optimisation using Octopus Agile price forecasting and hardware retrofits."
---

# Mastering Octopus Agile: 7-Day Forecasting & Hardware Automation

This guide outlines how to build a 7-day predicted pricing dashboard for Octopus Agile. This allows you to find the cheapest windows for EV charging and automate your hardware control.

---

## 1. Prerequisites
* **Home Assistant:** Installed and running on your home network.
* **Octopus Energy Integration:** This guide uses the [Octopus Energy Integration by BottlecapDave](https://github.com/BottlecapDave/HomeAssistant-OctopusEnergy).
* **Agile Predict API Sensor:** A "REST" sensor pulling data from the [AgilePredict API](https://agilepredict.com/).

---

## 2. The Control Challenge: A Specific Problem to Overcome
The primary hurdle in any energy automation project is determining how to physically trigger the "on/off" state of your appliances. For EV charging, there are only two paths:

### Option A: Vehicle-Side Control (The Software Route)
If your car has a supported Home Assistant integration, you can control the charging session directly via the car's API.

### Option B: Charger-Side Control (Hardware Route)
This involves controlling the power delivery at the wallbox. This is the most robust method for "dumb" chargers.

1.  **Modern Smart Chargers:** Use an integration if your charger supports it (e.g. Ohme, Zappi, or Wallbox).
2.  **The Shelly Retrofit (Legacy Hardware):** If you have an older unit, such as a **Chargemaster**, you can retrofit a **Shelly 1** relay inside the unit. 
    * **The Method:** By using the Shelly to interrupt the **Control Pilot (CP)** wire, you signal the car to pause or resume charging safely. 
    * **Reference Guide:** A detailed walkthrough of this specific hardware hack can be found in this [Chargemaster Shelly Retrofit Guide](https://github.com/p-shane/chargemaster-shelly-mod).
    * **Visual Guide:** For a visual demonstration, watch this video: [Building a Smart EV Charger with Shelly](https://www.youtube.com/watch?v=OSiaMJQIXbE).
3.  **Busbar Contactor:** You can install a heavy-duty contactor in your fuse board to cut power to the entire circuit.



---

## 3. Software Organisation: The `templates.yaml` File
To keep your configuration manageable, we avoid bloating the main `configuration.yaml` file by using a dedicated `templates.yaml` file.

### Why do we do this?
Think of `configuration.yaml` as the **"Main Control Room"** of your house. If you put every piece of code in there, it becomes a giant, messy "wall of text." By using the **`!include`** command, we are essentially creating **"Chapters"**. This keeps your system clean and organised.

### Setup Instructions

1. **In `configuration.yaml`:** Add this single line:
   `template: !include templates.yaml`

2. **In `templates.yaml`:** Create this new file and paste the code below:

```yaml
- sensor:
    - name: "Agile Forecast Summary"
      unique_id: agile_forecast_summary
      attributes:
        daily_data: >
          {% set prices = state_attr('sensor.agile_predict', 'prices') %}
          {% set ns = namespace(days={}) %}
          {% for rate in prices %}
            {% set d = as_datetime(rate.date_time).strftime('%d/%m/%Y') %}
            {% if d not in ns.days %}{% set ns.days = dict(ns.days, **{d: []}) %}{% endif %}
            {% set ns.days = dict(ns.days, **{d: ns.days[d] + [rate.agile_pred | float]}) %}
          {% endfor %}
          {% set output = [] %}
          {% for date, p in ns.days.items() | sort %}
            {% set output = output + [{'date': date, 'avg': (p|sum/p|length)|round(1), 'min': p|min, 'max': p|max}] %}
          {% endfor %}
          {{ output | to_json }}
```
---


## 4. Visualising the 7-Day Forecast
Once your sensor is running, use this Markdown card in your Home Assistant dashboard to view the weekly prices at a glance:


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

## 5. Creating the "Target Rate" Switch
We need a sensor that turns on only when the current price is the cheapest of the day. This logic compares your current Agile rate to the min value we extracted in the previous step.

Add this to your templates.yaml:

```yaml
- binary_sensor:
    - name: "EV Cheapest Charging Window"
      unique_id: ev_cheapest_charging_window
      icon: mdi:car-electric
      state: >
        {% set current_price = states('sensor.octopus_energy_electricity_current_rate') | float(0) %}
        {% set forecast = state_attr('sensor.agile_forecast_summary', 'daily_data') | from_json %}
        {% set today = now().strftime('%d/%m/%Y') %}
        
        {# Find today's minimum price from our forecast sensor #}
        {% set today_min = forecast | selectattr('date', 'eq', today) | map(attribute='min') | first | default(99) %}
        
        {# The sensor turns ON if current price is equal to or less than the daily minimum #}
        {{ current_price <= today_min }}
```      
## 6. The Automation
Now we tell Home Assistant to actually move the hardware. This automation watches the binary sensor we just made. When it turns on, it switches the Shelly relay; when it turns off, it stops the charge.

You can add this via the Settings > Automations UI or paste this into your automations.yaml:

YAML

alias: "EV Charging: Agile Price Optimizer"
description: "Automatically toggles the Shelly relay based on the cheapest predicted price of the day."
trigger:
  - platform: state
    entity_id: binary_sensor.ev_cheapest_charging_window
condition:
  - condition: state
    entity_id: binary_sensor.car_is_plugged_in # Optional: Add if you have a car integration
    state: "on"
action:
  - service: >
      {% if is_state('binary_sensor.ev_cheapest_charging_window', 'on') %}
        switch.turn_on
      {% else %}
        switch.turn_off
      {% endif %}
    target:
      entity_id: switch.shelly_ev_charger_relay
mode: restart
Why this approach works:
Safety: By using the binary sensor as the "brain," you can easily see on your dashboard if the car should be charging right now.

Predictability: Because it uses the min price from your forecast, it ensures you are always hitting that bottom trough, even if the "cheapest" time changes from 2:00 AM to 4:00 AM the next day.

Would you like me to help you create a Lovelace Dashboard card that shows a countdown until the next cheap charging window?

---

### About this Project
This project documents the journey of extending the life of "dumb" hardware through intelligent software. By prioritising charging during low-cost periods, we achieve significant savings while reducing peak demand on the UK energy grid.
