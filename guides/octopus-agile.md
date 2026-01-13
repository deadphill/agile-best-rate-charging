---
layout: post
title: "Mastering Octopus Agile: 7-Day Forecasting & Hardware Automation"
date: 13/01/2026
author: "Project Blog"
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
The primary hurdle in any energy automation project is determining how to physically trigger the "on/off" state of your appliances. For EV charging, you must decide how to bypass or integrate with the car’s charging function. Generally, there are only two paths to achieve this:

### Option A: Vehicle-Side Control (Integration)
If your vehicle has a supported Home Assistant integration (such as Tesla, Kia Connect, or Hyundai Bluelink), you can send commands directly to the car to start or stop charging.
* **Pros:** No electrical modifications required.
* **Cons:** Reliant on manufacturer cloud servers which can be unstable or have slow response times.

### Option B: Charger-Side Control (Hardware Retrofit)
This involves controlling the power delivery at the wallbox. This is the most robust method for "dumb" or older chargers.

1.  **Modern Smart Chargers:** Use an integration if your charger supports it (e.g. Ohme, Zappi, or Wallbox).
2.  **The Shelly Retrofit (Legacy Hardware):** If you have an older unit, such as a **Chargemaster**, you can retrofit a **Shelly 1** relay inside the unit. 
    * **The Method:** By using the Shelly to interrupt the **Control Pilot (CP)** wire, you signal the car to pause or resume charging safely. This is much safer than cutting the main 32A power directly.
    * **Reference Guide:** A detailed walkthrough of this specific hardware hack can be found in this [Chargemaster Shelly Retrofit Guide](https://github.com/p-shane/chargemaster-shelly-mod).
3.  **Busbar Contactor:** For a non-invasive approach, you can install a heavy-duty contactor in your fuse board to cut power to the entire circuit.

> **Visual Guide:** For a visual demonstration of how a hardware retrofit looks and functions in practice, watch this video: [Building a Smart EV Charger with Shelly](https://www.youtube.com/watch?v=OSiaMJQIXbE).

---

## 3. Software Organisation: The `templates.yaml` File
To keep your configuration manageable, we avoid bloating the main `configuration.yaml` file by using a dedicated `templates.yaml` file for our logic.

### Why do we do this?
Imagine your Home Assistant `configuration.yaml` is the "main switchboard" for your entire house. If you put every single piece of code in there, it becomes a giant, messy "wall of text" that is hard to read and easy to break. 

By using the **`!include`** command, we are essentially creating "chapters" in a book. We tell Home Assistant to keep the main settings in one place, but to look in `templates.yaml` for our custom sensor logic. This makes it much easier to find and fix mistakes later.

### Setup Instructions

1. **In `configuration.yaml`:** Find your main configuration file and add this single line. This tells Home Assistant to look for a separate file for all your templates:
   `template: !include templates.yaml`

2. **In `templates.yaml`:** Create this new file in the same folder and paste the code below. This sensor processes 336 data points from the API and provides a summarised 7-day outlook.

{% raw %}
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
{% endraw %}

---

## 4. Visualising the 7-Day Forecast
Once your sensor is running, use this Markdown card in your Home Assistant dashboard to view the weekly prices at a glance:

{% raw %}
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
{% endraw %}

---

### About this Project
This project documents the journey of extending the life of "dumb" hardware through intelligent software. By prioritising charging during low-cost periods, we achieve significant savings while reducing peak demand on the UK energy grid.
