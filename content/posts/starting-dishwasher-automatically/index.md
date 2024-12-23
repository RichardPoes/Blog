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
1. Get a by-the-hour energy contract, also known as a dynamic contract;
1. Fill the place with lots of smart appliances;
1. Get to know what the electricity prices are;
1. Find the cheapest price;
1. Leverage Home Assistant to turn on those appliances when electricity is cheapest;
1. Profit of those low prices.

This would have the additional effect of doing something green and to help balancing the grid.
Sounds like a fairly simple straightforward plan, huh?

Well the reality turned out to be much different.
Home Assistant wasn't as intuitive as we thought, and also not as powerful on its own.
Moreover, our energy provider did not offer an out of the box way to get these energy prices.
In this post we will describe our journey towards automating our first smart device, our dishwasher.

# Fetching energy prices without an official API (1)
When buying property for the first time, a lot of things need fixing, so we did not put in too much effort into finding _the best_ energy provider, whatever that meant to us.
So we just looked for energy companies offering a dynamic contract, and picked a reliable looking one: [Vrijopnaam](https://vrijopnaam.nl/en/).

After the dust had settled from the moving, we looked around if we could find a (semi) public API somewhere. 
We even reached out to their customer service, but there was no API we could use.
Which, if you ask us, is a bit weird, because to actually leverage those dynamic prices, a lot of automation needs to be done.

We were however able to see the dynamic prices ourselves.
The prices did however come baked in an HTML page.
It looked something like this:

```html
<table class="pricing-table">
    <thead class="has-pricing">
        <tr>
            <th class="column-period">Period</th>
            <th class="column-tariff">
                Today €/kWh
                <div>average <b>0.23379</b></div>
            </th>
            <th class="column-tariff">
                Tomorrow €/kWh
                <div>average <b>0.27562</b></div>
            </th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td class="column-period pricing-past">
                <span class="monospace">00 - 01</span> hr
            </td>
            <td class="column-tariff pricing-past">
                <span class="monospace">0.17505</span>
                <span class="mark-cheaper"></span>
            </td>
            <td class="column-tariff">
                0.25571
                <span class="mark-cheaper"></span>
            </td>
        </tr>
        <tr>
            <td class="column-period pricing-past">
                <span class="monospace">01 - 02</span> hr
            </td>
            <td class="column-tariff pricing-past">
                <span class="monospace">0.16766</span>
                <span class="mark-cheaper"></span>
            </td>
            <td class="column-tariff">
                0.25185
                <span class="mark-cheaper"></span>
            </td>
        </tr>
        <!-- etc.. -->
    </tbody>
</table>
```

This would render in a table looking somewhat like **Table ??**, as shown below.

| Period     | Today €/kWh average 0.23379 | Tomorrow €/kWh average 0.27562 |
|------------|-----------------------------|--------------------------------|
| 00 - 01 hr | 0.17505                     | 0.25571                        |
| 01 - 02 hr | 0.16766                     | 0.25185                        |



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