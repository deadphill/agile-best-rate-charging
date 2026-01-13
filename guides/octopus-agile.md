---
layout: post
title: "Mastering Octopus Agile: 7-Day Forecasting & Hardware Automation"
date: 2026-01-13
author: "The Hornets"
description: "A complete guide to automating EV charging with Octopus Agile, predicted rates, and Shelly hardware retrofits."
---

# Mastering Octopus Agile: 7-Day Forecasting & Hardware Automation

This guide outlines how to build a 7-day predicted pricing dashboard for Octopus Agile to help you find the cheapest windows for EV charging.

---

## 1. Prerequisites
* **Home Assistant:** Installed and running.
* **Octopus Energy Integration:** The [BottlecapDave integration](https://github.com/BottlecapDave/HomeAssistant-OctopusEnergy) must be configured.
* **Agile Predict API Sensor:** A REST sensor (usually `sensor.agile_predict`) pulling data from the [AgilePredict API](https://agilepredict.com/).

---

## 2. The Software Problem: Data Overload
The prediction API provides 336 data points (48 slots $\times$ 7 days). Processing this directly in the UI causes lag and a "wall of text."

### The Solution: Backend Optimization
We moved the logic to a **Template Sensor** in the Home Assistant backend. This processes the data once an hour and stores a clean "summary" for each day.

#### Setup: `templates.yaml`
Add this to your `templates.yaml` file to create `sensor.agile_forecast_summary`:

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
