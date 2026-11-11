---
name: rapidapi
description: Google Flights Live and Booking.com Live search via RapidAPI. Secondary source for cash flight prices and hotel availability when SerpAPI needs a second opinion.
category: hotels
summary: Booking.com hotel prices.
api_key: RapidAPI
license: MIT
---

# RapidAPI Skill

Live Google Flights and Booking.com scrapers on RapidAPI, both published by `mtnrabi`
(flightpowers.com). Secondary source for cash flight prices and hotel pricing.

**Sources** (moved from the old `apiheya` listings, which now 404; endpoints changed too):
- [Google Flights Live API](https://rapidapi.com/mtnrabi/api/google-flights-live-api): host `google-flights-live-api.p.rapidapi.com`
- [Booking.com Live API](https://rapidapi.com/mtnrabi/api/booking-live-api): host `booking-live-api.p.rapidapi.com`

## Authentication

`RAPIDAPI_KEY` in `.env`. Every request sends `x-rapidapi-key`, `x-rapidapi-host` and
`Content-Type: application/json`. The key only works for APIs you have **subscribed** to on
RapidAPI (each one separately; BASIC is free). `{"message":"You are not subscribed to this API."}`
means subscribe first. `Endpoint ... does not exist` means a wrong path.

## Quotas (per API, per month, hard cap on BASIC: request N+1 is refused, never billed)

| API | BASIC (free) | PRO |
|---|---|---|
| Google Flights Live | **50** requests | $10/mo, 2,500 |
| Booking.com Live | **10** requests | $10/mo, 2,000 |

Every call counts, **including 4xx errors** (e.g. a past date). Spend deliberately: one query per
date/cabin, no sweeps on the free tier.

## Google Flights Live API

All endpoints are `POST` with a JSON body. A search takes ~15–30 s.

| Endpoint | Purpose |
|---|---|
| `/api/google_flights/oneway/v1` | One-way |
| `/api/google_flights/roundtrip/v1` | Round trip, both legs paired with one `total_price` |
| `/api/google_flights/oneway_multi_airport/v1` | One-way, 1–4 airports per side (list or `"SFO,OAK,SJC"`; city codes like `LON` OK) |
| `/api/google_flights/roundtrip_multi_airport/v1` | Round trip, 1–4 airports per side (airports only, no city codes) |

### One-way

```bash
curl -s -X POST "https://google-flights-live-api.p.rapidapi.com/api/google_flights/oneway/v1" \
  -H "x-rapidapi-key: $RAPIDAPI_KEY" \
  -H "x-rapidapi-host: google-flights-live-api.p.rapidapi.com" \
  -H "Content-Type: application/json" \
  -d '{
    "from_airport": "SFO",
    "to_airport": "NRT",
    "departure_date": "2027-03-10",
    "max_stops": 0,
    "seat_type": 1,
    "passengers": [1, 1],
    "limit": 10
  }' | jq '.[] | {price, airline, stops, duration, departure_description, arrival_description, price_range_in_relation_to_other_periods}'
```

**Required:** `from_airport`, `to_airport` (IATA), `departure_date` (`YYYY-MM-DD`).

| Optional param | Meaning |
|---|---|
| `max_stops` | 0 = nonstop |
| `seat_type` | `1` economy, `3` business (documented). `2` premium economy / `4` first follow Google's convention but are undocumented; verify before relying on them |
| `passengers` | Array, one entry per traveler: `1` adult, `2` child, `3` infant on lap, `4` infant in seat. `[1,1]` = 2 adults |
| `sort_type` | `Overall` (default), `Price`, `Duration` |
| `airline_codes` / `exclude_airline_codes` | Arrays of IATA codes |
| `departure_time_min/_max`, `arrival_time_min/_max` | Hour of day, 0–23 |
| `currency` | Default `USD` |
| `max_price` | Integer |
| `limit` | Default 10 |
| `use_ext_proxy` | Default `true` (residential proxy); `false` = faster on easy routes |

**Response:** array of `{price, price_as_number, airline, stops, stops_info[{stop_airport,
stop_duration_seconds}], duration, duration_seconds, departure_description, arrival_description,
buy_link, price_insights_low, price_insights_high, price_range_in_relation_to_other_periods}`.
The last three are Google's low/typical/high verdict. They are **omitted** with `sort_type: "Price"`,
with `max_price`, or for stays over 32 nights.

### Round trip

Same host, `/api/google_flights/roundtrip/v1`, plus `return_date`. Filters split per leg:
`max_departure_stops` / `max_return_stops`, `departure_airline_codes` / `return_airline_codes`,
`departure_departure_time_min` … `return_arrival_time_max`, etc. Each result row is one paired
itinerary: `total_price`, `total_price_as_number`, `total_duration_seconds`, `total_stops`,
`departure_flight_*` and `return_flight_*` fields, and one `buy_link`.

### Caveats

- **Prices are Google's cheapest fare, so usually the basic/lowest fare brand** (e.g. United
  Basic Economy). There is no fare-brand field. Treat as a lower bound and get standard-economy
  prices from the carrier site or Duffel.
- `airline` is Google's carrier string and may list several carriers (codeshares, mixed legs).
- `200 []` = Google positively had no flights. `503` with `search_incomplete` = didn't finish, safe to retry.

## Booking.com Live API

All endpoints are `POST` with a JSON body at `https://booking-live-api.p.rapidapi.com`.
Past dates fail with a 502 after ~30 s and still count against the quota.

| Endpoint | Purpose | Required |
|---|---|---|
| `/hotel_by_name` | Price for one named property (resolves the name for you) | `hotel_name`, `checkin_date`, `checkout_date` |
| `/search` | Destination search | `destination` (free text), `checkin_date`, `checkout_date` |
| `/hotel` | Every room in one property | `hotel_booking_id` (Booking path ID, e.g. `cy/four-seasons-limassol`), dates |

Common optional params: `adults` (default 2), `children`, `currency` (default USD),
`proxy_country` (price as a resident of that market), `free_cancellation`.
`/hotel_by_name` also takes `area` (disambiguates a generic name). `/search` also takes
`budget_per_night`, `sort` (`price_asc` | `price_desc` | `rating_desc`) and `filters` (array, e.g.
`free_cancellation`, `breakfast_included`, `review_score_8`, `stars_4`, `free_wifi`, `parking`).

```bash
curl -s -X POST "https://booking-live-api.p.rapidapi.com/hotel_by_name" \
  -H "x-rapidapi-key: $RAPIDAPI_KEY" \
  -H "x-rapidapi-host: booking-live-api.p.rapidapi.com" \
  -H "Content-Type: application/json" \
  -d '{"hotel_name": "Park Hyatt Tokyo", "area": "Tokyo", "checkin_date": "2027-03-10", "checkout_date": "2027-03-13", "adults": 2}' | jq '.'
```

`/hotel_by_name` returns `{matched_name, available, price, price_string, room_type, review_score,
review_count, link, nights}`. **Check `matched_name`** before trusting the price. `price` is
the total for the stay. `/search` returns `{properties: [...]}` with the same per-property fields.

## When to Use

- **Google Flights Live**: second opinion on cash fares, or the Google Flights source when SerpAPI
  is out of quota. Round trips cost one request (vs two on SerpAPI).
- **Booking.com Live**: Booking.com-specific pricing/availability. Only 10 free/month, so use it
  only for a specific shortlisted property.

Do not:
- Use as primary search (SerpAPI and Seats.aero are primary).
- Burn through the free tier on broad searches or date sweeps.
