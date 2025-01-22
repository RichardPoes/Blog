+++
title =  "Fetching VijOpNaam energy prices without an official API"
date = 2024-12-21
tags = ["Home Automation", "Programming" ]
authors = ["Richard", "Marc"]
summary = """
When we bought an apartment together, we wanted to automate as much as possible in our apartment.
Some of those automations require the knowledge of the current hourly electricity price.
Here we explain how we fetched our energy prices, despite our energy company not having an official API.
"""
+++

Ever since we bought the apartment we had a simple :sparkles:*master plan*:sparkles::
1. Find an energyprovider offering a by-the-hour energy contract, also known as a dynamic contract;
1. Fill the place with lots of smart appliances;
1. Get to know what the electricity prices are;
1. Find the cheapest  \(n\)-hour average price;
1. Leverage Home Assistant to turn on those appliances when electricity is cheapest;
1. Profit of those low prices.

Sounds like a fairly simple straightforward plan, huh?
Well, the reality turned out to be much different.
We had just looked for energy companies offering a dynamic contract, and picked a reliable looking one: [Vrijopnaam](https://vrijopnaam.nl/en/).
However, we could not find a (semi) public API anywhere.
The customer service confirmed that they did not have an API, which, if you ask us is a bit weird.
Naturally, being both software developers, we started to look for a work around.

# Investigating the user flow
Despite there not being a public API, we realized however that *we* were able to see the dynamic prices.
Sadly the prices did not get served to the front-end as a JSON, they came baked in an HTML page.
It looked something like this:

```html {caption="Partial HTML returned when looking for currect prices"}
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

This would render as a table looking somewhat like {{< cref-tab id="tab:ex-pricing" >}}, as shown below.

{{< table caption="Example of pricing data" id="tab:ex-pricing">}}
| Period     | Today €/kWh average 0.23379 | Tomorrow €/kWh average 0.27562 |
|------------|-----------------------------|--------------------------------|
| 00 - 01 hr | 0.17505                     | 0.25571                        |
| 01 - 02 hr | 0.16766                     | 0.25185                        |
{{< /table >}}

We immediately realized we could scrape the data from that webpage.
Of course, VrijOpNaam would not be too keen on that, so we had to make sure we did not query the pricing website too much.
So the idea was to create a local webserver which would cache the pricing data for us.
It would only fetch new prices at VrijOpNaam once or twice a day and we could then query the data at will.

# Retrieving the table page using Hoppscotch
Our next goal was to investigate what steps our program needed to do, before it could actually scrape the table page.
This meant replaying the whole process from login to page navigation using Hoppscotch, which is a free open source alternative to Postman (for now).

For an initial login, these were the steps one had to take:
1. Navigate to https://vrijopnaam.app;
1. Enter your credentials and submit;
1. Answer whether you want to stay signed in;
1. Press the button for the current electricity tariff, which is equivalent to navigating to https://vrijopnaam.app/mc/99999/pricing-electricity/;
1. Scrape that data as shown in {{< cref-tab id="tab:ex-pricing" >}}.

## The initial login
The important parts login form look somewhat like this:
```html {caption="Abbreviated sign-in form for VrijOpNaam"}
<form method="post" action="/">
  <input type="hidden" name="csrfmiddlewaretoken"
   value="DHPnlATWt9VkeaH6rfYuYId2e6ou6zrgknPMh01oeRwbrFXVBdrzkKVKdIrFLu80">
  <input type="text" name="username">
  <input type="password" name="password">
  <button type="submit" class="btn-icon btn-primary">
      Start
  </button>
</form>
```
When actually pressing the start/login button, we observe the following request.

```http {caption="POST request made when signing in"}
POST / HTTP/2
Host: vrijopnaam.app
Content-Type: application/x-www-form-urlencoded
Cookie: csrftoken=RQaz6AiCVSL1nFqZk8DfwcSS9MdlP5RU; 
Cookie: sessionid=qojt5ig7bfcresy9zl7iex9ms0ort92h;

# Line breaks for readability 
csrfmiddlewaretoken=DHPnlATWt9VkeaH6rfYuYId2e6ou6zrgknPMh01oeRwbrFXVBdrzkKVKdIrFLu80&
username=USERNAME&
password=PASSWORD
```
Obviously, all three entries of the form get sent: the hidden [CSRF](https://en.wikipedia.org/wiki/Cross-site_request_forgery) token, the username and the password.
We were however a bit surprised to see the two cookies `csrftoken` and `sessionid`.
CSRF-tokens are usually only send as an hidden input in forms and a session ID is usually sent *after* signing in.
It took us a while to figure out what was happening, but upon making the initial `GET /` request, to access the login form, we retrieved
```http
HTTP/2 200 OK
Set-Cookie: csrftoken=RQaz6AiCVSL1nFqZk8DfwcSS9MdlP5RU; 
Set-Cookie: sessionid=qojt5ig7bfcresy9zl7iex9ms0ort92h;
```
It seems that either multiple measures are taken to prevent CSRF, or they might have opted for a good old security-by-obscurity.
A new `sessionid` and `csrftoken` are generated each time we send a request to VrijOpNaam, so it is important to make sure we send the newly retrieved tokens back when making the next request.

## Staying signed in
After successfully 

# Parsing the HTML table
Now that it was clear how the requests had to be formatted, we could start making them automatically, so that we could parse the HTML table to typed variables.
With the explanation given above, the sign-in process was now straightforward and a Python implementation can be found on [GitHub](https://github.com/Kaaserne/vrijopnaam-prices/blob/main/vrijopnaam_prices/_vrijopnaam_session.py).

Once signed in, we could start scraping the HTML table.
This was done using the Python library [Beautiful Soup](https://beautiful-soup-4.readthedocs.io/en/latest/).
Given the fact that we are logged in, are on the page where the electricity pricing table is present and use the following table: ![Vrijopnaam dynamic electricity price table](pricing-table-explained.png)
We can write some very compact Python code to parse the table.

```python {caption="Python code to parse the pricing table"}
from bs4 import BeautifulSoup
import asyncio


async def scrape_prices() -> str:
    # Gets the HTML page and returns it
    ...

async def parse_prices(html: str):
    soup = BeautifulSoup(html, 'html.parser')
    # Find the table which has a class named 'pricing-table'
    table = soup.find('table', class_='pricing-table')
    # For each row in this table
    for row in table.find_all('tr'):
        # We have three columns in one row. The first is the period,
        # the second is today's price and the third is tomorrow's price
        columns = row.find_all('td')
        # We now have one <td> element. 
        # But each <td> element has a <span> element
        period, today_price, tomorrow_price = columns
        # So, we need to extract the text from the <span> element
        # for each of the three columns
        period = period.find('span').text
        today_price = today_price.find('span').text
        tomorrow_price = tomorrow_price.find('span').text
        print(f'{period}: {today_price} - {tomorrow_price}')

async def main():
    html = await scrape_prices()
    await parse_prices(html)
    # Here the output will be printed. It wil be something like:
    # 00-01 uur: 0.17505 - 0.25571
    # 01-02 uur: 0.16766 - 0.25185
    # etc...

asyncio.run(main())
```
