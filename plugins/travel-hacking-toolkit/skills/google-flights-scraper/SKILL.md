---
name: google-flights-scraper
description: Google Flights cash prices via OmkarCloud's Google Flights Scraper API on RapidAPI (1,000 free calls/month). One-way, round trip, multi-city, plus per-seller FARE BRANDS (Basic vs Standard vs Refundable vs premium cabins) with bag fees, price insights/history, price calendar, date grid and explore. Use as the Google Flights source alongside or instead of SerpAPI, and whenever you need the bookable non-Basic fare. Triggers on "Google Flights price", "standard economy fare", "fare brands", "cheapest dates", "price calendar".
category: flights
summary: Google Flights via REST, with per-seller fare brands (Basic vs Standard) and bag fees. 1,000 free calls/month.
api_key: RapidAPI (1,000 free)
license: MIT
---

# Google Flights Scraper (OmkarCloud, RapidAPI)

REST access to live Google Flights data: [listing](https://rapidapi.com/OmkarCloud/api/best-google-flights-scraper-free-1000-calls).
Its main advantage over SerpAPI and other Google Flights scrapers is **Booking Options**, which lists
every seller's fare **brand** for a chosen itinerary. That gives the real Basic → Standard → Refundable
ladder (e.g. UA SFO–IAH: Basic $205 / Economy $265 / Fully Refundable $305 / Economy Plus $431).

## Setup

`RAPIDAPI_KEY` in `.env`, **subscribed** to this API on RapidAPI (Basic plan is free). Each request
sends both headers:

```bash
H=best-google-flights-scraper-free-1000-calls.p.rapidapi.com
gf() { curl -s -m 120 -G "https://$H/$1" -H "x-rapidapi-key: $RAPIDAPI_KEY" -H "x-rapidapi-host: $H" "${@:2}"; }
```

`403 {"message":"You are not subscribed to this API."}` = subscribe first.

**Quota:** Basic = 1,000 calls/month free. Pro $16 = 20,000. Every endpoint call counts, including
each step of the round-trip/booking flow. Typical latency: search ~7 s, booking options ~3 s.

## Endpoints (all GET, query params)

| Endpoint | Purpose |
|---|---|
| `/flights/search-one-way` | One-way results, each with a `booking_token` |
| `/flights/search-round-trip` | Outbounds priced as round trips, each with a `returning_token` |
| `/flights/returning-flights` | `returning_token=` → returns for that outbound, each with a `booking_token` |
| `/flights/booking-options` | `booking_token=` → **every seller + fare brand + price + bag fees + link** |
| `/flights/search-multi-city` | `legs=SFO,NRT,2027-03-10\|NRT,BNE,2027-03-20` (2–6 legs; `+` joins airports) → first-leg options with `next_leg_token` |
| `/flights/multi-city/next-leg` | `next_leg_token=` → next leg; the last leg returns `booking_token`s |
| `/flights/price-insights` | Google's low/typical/high verdict, typical range, price history |
| `/prices/calendar` | Cheapest fare per departure date, up to 92 days (`start_date`, `end_date`, optional `trip_length_days`) |
| `/prices/date-grid` | Round-trip fares for departure × return dates within ±3 days |
| `/explore/destinations` | Cheapest destinations (`month`, `trip_length`, `region`, `interest`) |
| `/routes/city-pair`, `/routes/from-city`, `/routes/to-city` | Route insights: cheapest by month, airlines, seasons |
| `/airports/search` | `query=` → place ids for cities/airports/regions |

## Search parameters (one-way / round-trip)

**Required:** `origin`, `destination` (airport code, city name, place id, or comma-separated list),
`departure_date` (`YYYY-MM-DD`); round trip adds `return_date`.

| Param | Values |
|---|---|
| `cabin_class` | `economy` (default), `premium_economy`, `business`, `first` |
| `stops` | `any` (default), `nonstop`, `one_stop_or_fewer`, `two_stops_or_fewer` |
| `sort_by` | `best` (default), `price`, `departure_time`, `arrival_time`, `duration`, `emissions` |
| `airlines` | Comma-separated codes and/or `oneworld`, `skyteam`, `star_alliance` |
| `departure_hours` / `arrival_hours` | Hour range, e.g. `6-12`; round trip also `return_departure_hours` / `return_arrival_hours` |
| `max_duration_minutes`, `max_layover_minutes`, `connecting_airports`, `max_price` | Filters |
| `adults` (1), `children`, `infants_in_seat`, `infants_on_lap` | Passengers (max 9) |
| `carry_on_bags` (0–1), `checked_bags` (0–2) | Per passenger; prices then include the bag fees |
| `currency` (`USD`), `country` (`US`), `language` (`en-US`) | `country` = the market Google prices from (point-of-sale arbitrage) |

An invalid value returns `400` listing the allowed values.

## Response shape

Search: `{search, price_insights, best_flights[], other_flights[], filters, airline_policies, airports}`.
Each flight: `price`, `airlines[{code,name}]`, `departure`/`arrival` `{airport{code,name,city}, date, time}`,
`duration_minutes`, `stops`, `layovers[]`, `segments[]` (each with `flight_number` like `"UA 686"`,
`operated_by`, `aircraft`, `seat.legroom`, `amenities`, `is_often_delayed`, `is_overnight`), and a
`booking_token` (one-way) or `returning_token` (round trip). The search `price` is the **lowest fare
brand** (usually Basic Economy).

Booking options: `{selected_flights, lowest_price, booking_options[]}`. Each option has `seller{name,
is_airline}`, `price`, `fare_name` (e.g. `Basic Economy`, `Economy`, `Economy Plus`, `Base Business`),
`fare_code`, `is_basic_economy`, `is_separate_tickets`, `features[]` (carry-on, seat selection,
changes, refunds), `checked_bag_fees[]`, and a `google_booking_link`.

## Getting the non-Basic fare (the common case)

```bash
gf flights/search-one-way --data-urlencode origin=SFO --data-urlencode destination=IAH \
  --data-urlencode departure_date=2026-12-10 --data-urlencode stops=nonstop > s.json
TOKEN=$(jq -r '[.best_flights[]?, .other_flights[]?] | map(select(.segments[0].flight_number=="UA 686"))[0].booking_token' s.json)
gf flights/booking-options --data-urlencode "booking_token=$TOKEN" \
  | jq -r '.booking_options[] | [.seller.name, .price, .fare_name, (if .is_basic_economy then "BASIC" else "" end)] | @tsv'
```

Round trip: `search-round-trip` → pick outbound → `returning-flights?returning_token=` → pick return →
`booking-options?booking_token=` (3 calls). The `price` on each step is the round-trip total.

## Caveats

- Booking options can include **fares from other cabins** (a business search returned an economy
  refundable option too). Filter by `fare_name`.
- **Coverage gaps are possible.** A SFO–SYD business nonstop search returned only United, so cross-check
  route coverage (SerpAPI, carrier site) before concluding a carrier doesn't fly it.
- It's a third-party scraper of Google Flights. Validate the finalist on the carrier site before ticketing.
- `{"error":"google-flights rejected the request: ... Google rejected the parameters ..."}` can be
  **transient**: the identical request succeeded on retry. Retry once before changing parameters.
- Tokens are short-lived. Use them right after the search that produced them.
