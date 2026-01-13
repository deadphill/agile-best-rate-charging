---
layout: post
title: "Mastering Octopus Agile: 7-Day Forecasting & Hardware Automation"
date: 2026-01-13

description: "A technical walkthrough of UK energy optimisation using Octopus Agile price forecasting and hardware retrofitting."
---

# Mastering Octopus Agile: 7-Day Forecasting & Hardware Automation

This guide outlines how to build a 7-day predicted pricing dashboard for Octopus Agile to help you find the cheapest windows for EV charging and automate the hardware control.

---

## 1. Prerequisites
* **Home Assistant:** Installed and running.
* **Octopus Energy Integration:** This guide uses the excellent [Octopus Energy Integration by BottlecapDave](https://github.com/BottlecapDave/HomeAssistant-OctopusEnergy).
* **Agile Predict API Sensor:** A REST sensor pulling data from the [AgilePredict API](https://agilepredict.com/).

---

## 2. The Control Challenge: A Specific Problem to Overcome
The primary hurdle in any energy automation project is determining how to physically trigger the "on/off" state of your appliances. For EV charging, you must decide how to bypass or integrate with the car’s charging function. Generally, there are only two paths to achieve this:

### Option A: Vehicle-Side Control (Integration)
If your vehicle has a supported Home Assistant integration (such as Tesla, Kia Connect, or Hyundai Bluelink), you can send commands directly to the car to start or stop charging.
* **Pros:** No electrical modifications required.
* **Cons:** Reliant on manufacturer cloud APIs which can be unstable or have latency.

### Option B: Charger-Side Control (Hardware Retrofit)
This involves controlling the power delivery at the wallbox. This is the most robust method for "dumb" or older chargers.
1.  **Modern Smart Chargers:** Use an integration if your charger supports it (e.g. Ohme or Zappi).
2.  **The Shelly Retrofit (Legacy Hardware):** If you have an older unit, such as a **Chargemaster**, you can retrofit a **Shelly 1** relay inside the unit. 
    * **The Method:** By using the Shelly to interrupt the **Control Pilot (CP)** wire, you can signal the car to pause or resume charging safely without cutting the main 32A power.
    * **Reference:** A detailed walkthrough of this specific hardware hack can be found in this [Chargemaster Shelly Retrofit Guide](https://github.com/p-shane/chargemaster-shelly-mod).
3.  **Busbar Contactor:** For a completely non-invasive hardware approach, you can install a heavy-duty contactor on your consumer unit busbar to cut power to the entire circuit, though this is less "elegant" than the CP-wire method.

---

## 3. Software Organisation: The `templates.yaml` File
To keep your configuration manageable, we avoid bloating the `configuration.yaml` file by using a dedicated `templates.yaml` file for our logic.

### Setup Instructions
1.  **In `configuration.yaml`:** Ensure you have added the line: `template: !include templates.yaml`
2.  **In `templates.yaml`:** Add the code below. This sensor processes 336 data points from the Agile Predict API and provides a summarised 7-day outlook.

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
            {% set d = as_datetime(rate.date_time).strftime('%Y-%m-%d') %}
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
Once your backend sensor is running, use this Markdown card in your Home Assistant dashboard to view the weekly prices at a glance:

{% raw %}
```yaml
type: markdown
content: >-
  | Day | Min | Avg | Max |
  |:---|:---:|:---:|:---:|
  {% set data = state_attr('sensor.agile_forecast_summary', 'daily_data') | from_json %}
  {% for day in data[:7] %}
  | {{ as_datetime(day.date).strftime('%a %d') }} | {{ day.min }}p | **{{ day.avg }}p** | {{ day.max }}p |
  {% endfor %}
```
{% endraw %}

---

### Further Resources & Video Guide
For a visual demonstration of how the hardware retrofit looks in practice, I highly recommend watching this video: [Building a Smart EV Charger with Shelly](https://www.youtube.com/watch?v=OSiaMJQIXbE).

### About this Project
This project documents the journey of extending the life of "dumb" hardware through intelligent software. By prioritising charging during low-cost periods, we achieve significant savings while reducing peak grid demand.
