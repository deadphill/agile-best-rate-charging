layout: post title: "Mastering Octopus Agile: 7-Day Forecasting & Hardware Automation" date: 2026-01-13 author: "The Hornets" description: "A technical walkthrough of automating EV charging using Octopus Agile price forecasting and hardware retrofitting."
Mastering Octopus Agile: 7-Day Forecasting & Hardware Automation
This guide outlines how to build a 7-day predicted pricing dashboard for Octopus Agile to help you find the cheapest windows for EV charging and automate the hardware.

1. Prerequisites
Home Assistant: Installed and running.

Octopus Energy Integration: The BottlecapDave integration must be configured.

Agile Predict API Sensor: A REST sensor pulling data from the AgilePredict API.

2. The Control Challenge: How to Start/Stop the Charge
The biggest hurdle in EV automation is actually "flipping the switch." Depending on your hardware, there are two generic approaches to determine how to turn the car's charging function on and off.

Option A: Vehicle-Side Control (The Software Route)
If your car has a "connected" integration (e.g., Tesla, Hyundai Bluelink, Kia Connect), you can control the charging session directly via the car's API.

Pros: No electrical work; works with any "dumb" charger.

Cons: Cloud-dependent; some manufacturers limit API calls (causing "sleep" issues).

Option B: Charger-Side Control (The Hardware Route)
This involves controlling the power delivery at the wallbox itself. This is often more reliable than car APIs.

Smart Integration: If you have a modern charger (e.g., Ohme, Zappi, Wallbox), you can usually integrate it directly into Home Assistant via its official integration or OCPP.

The "Shelly Hack" (Retrofit): For older "dumb" units like a Chargemaster, you can open the unit and install a Shelly 1 relay.

How it works: You use the Shelly's "Dry Contacts" to interrupt the Control Pilot (CP) wire. When the relay is open, the car thinks the charger is disconnected. When closed, the handshake completes and charging begins. This is safer than switching the main 32A power.

Busbar Contactor Control: If you cannot modify the charger, you can install a high-load contactor on the EV's dedicated circuit in your consumer unit (fuse board), controlled by a smart switch. Note: This "hard cut" of power can occasionally cause errors in some vehicle onboard chargers.

3. Software Setup: Home Assistant Logic
To keep your Home Assistant organised, we separate the "brain" (logic) from the "body" (configuration).

The templates.yaml structure
Instead of cluttering your main configuration.yaml, you should use a dedicated templates.yaml file. This is where we process the 7-day forecast data into a readable summary.

In configuration.yaml: Ensure you have this line: template: !include templates.yaml

In templates.yaml: Add the following code to create a summary sensor. This processes the 336 data points from the Agile Predict API into 7 clean daily averages.

{% raw %}

YAML

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
{% endraw %}

4. Visualising the Forecast
To see your 7-day forecast on your dashboard, use a Markdown Card. This code pulls the data from your new template sensor and builds a clean table.

{% raw %}

YAML

type: markdown
content: >-
  | Day | Min | Avg | Max |
  |:---|:---:|:---:|:---:|
  {% set data = state_attr('sensor.agile_forecast_summary', 'daily_data') | from_json %}
  {% for day in data[:7] %}
  | {{ as_datetime(day.date).strftime('%a %d') }} | {{ day.min }}p | **{{ day.avg }}p** | {{ day.max }}p |
  {% endfor %}
{% endraw %}

About this Project
This project is part of a wider effort to bridge the gap between Hardware (IoT) and Data (APIs). By retrofitting legacy hardware like the Chargemaster with modern smart switches, we can extend the life of existing infrastructure while drastically reducing energy costs.
