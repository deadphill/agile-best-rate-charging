Mastering Octopus Agile: A 7-Day Forecasting & Hardware GuideThis guide explains how to move beyond 24-hour confirmed rates to a 7-day predicted outlook, and how to physically control "dumb" hardware to automate your EV charging.1. PrerequisitesHome Assistant (Installed).Octopus Energy Integration (by BottlecapDave). How to setup Octopus IntegrationAgile Predict API Sensor: A REST sensor (e.g., sensor.agile_predict) pulling data from the AgilePredict API.2. The Software Problem: Data OverloadThe prediction API provides 336 data points (48 slots $\times$ 7 days). Processing this in the UI causes lag and a "wall of text" (the "Long Box" issue).The Solution: Offload the math to a Template Sensor in your backend.Setup: The templates.yaml FileIf you don't have one, add template: !include templates.yaml to your configuration.yaml. Inside templates.yaml, add:YAML- sensor:
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
The UI: Clean Markdown TableInstead of a long list, use a Markdown card to display a scannable table:YAMLtype: markdown
content: >-
  | Day | Min | Avg | Max |
  |:---|:---:|:---:|:---:|
  {% set data = state_attr('sensor.agile_forecast_summary', 'daily_data') | from_json %}
  {% for day in data[:7] %}
  | {{ as_datetime(day.date).strftime('%a %d') }} | {{ day.min }}p | **{{ day.avg }}p** | {{ day.max }}p |
  {% endfor %}
3. The Hardware Problem: Triggering the ChargeHow do you automate a "dumb" charger? You have two main paths:Option 1: The Vehicle API. Use integrations like Tesla or Kia Connect. Easy setup, but cloud-reliant.Option 2: The Hardware Retrofit. For older units like a Chargemaster, you can install a Shelly 1 switch inside the unit.The Hack: Use the Shelly to interrupt the Control Pilot (CP) wire. This signals the car to pause/start charging without cutting the main 32A power directly, which is safer for the contactors.Busbar Alternative: For a "dumb" setup without opening the charger, use a heavy-duty DIN-rail contactor in your consumer unit triggered by a smart switch.
