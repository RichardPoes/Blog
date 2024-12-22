+++
title =  "Starting your dishwasher automatically in Home Assistant when electricity prices are low"
date = 2024-12-21
tags = ["Home Automation", "Programming" ]
authors = ["Richard", "Marc"]
summary = """
When we bought an apartment together, the immediate idea was to automate as much as possible.
In this post we explore how to start our dishwasher whenever the hourly electricity prices are low.
As it turns out, Home Assistant on its own is not powerful enough to do that.
There were many twists and turns on this road.
"""
+++

Ever since we bought the apartment our :sparkles:*master plan*:sparkles: was simple:
1. Get a by-the-hour energy tariff;
1. Fill the place with lots of smart appliances;
1. Get to know what the electricity prices are;
1. Find the cheapest price;
1. Leverage Home Assistant to turn on those appliances when electricity is cheapest;
1. Profit of those low prices.

This would have the additional effect of doing something green and helping balancing the grid.
Sounds like a fairly simple straightforward plan, huh?

Well the reality turned out to be much different.
Home Assistant wasn't as intuitive as we thought, and also not as powerful on its own.
Moreover, our energy provider did not offer an out of the box way to get these energy prices.

```yaml
sensor:
  - platform: rest
    unique_id: electricity_price_vrijopnaam
    resource: http://192.168.5.4/vrijopnaam-prices-today
    name: Huidige stroomprijs
    value_template: "{{ value_json.electricity.prices[now().hour].price }}"
    scan_interval: 60 # Every minute
    unit_of_measurement: "€/kWh"
 ```