---
name: google-flights
description: Browser-automated Google Flights search via agent-browser. All airlines including Southwest. Cash prices, schedules, economy/business comparison, market selection.
category: flights
summary: Browser-automated Google Flights. All airlines including Southwest.
api_key: None (requires agent-browser)
allowed-tools: Bash(agent-browser:*)
---

# Google Flights Search

Search Google Flights via agent-browser to find flight prices, schedules, and availability.

## Prerequisites

Requires `agent-browser` CLI:
```bash
npm install -g agent-browser && agent-browser install
```

## When to Use

- User asks to search/find/compare flights or airfare
- User wants to know flight prices between cities
- User asks about flight schedules or availability
- User wants to find the cheapest flight for specific dates
- **Southwest flights**: Google Flights is one of the only sources for SW cash prices. No GDS or API returns Southwest.

## When NOT to Use

- **Completing purchases**: This skill finds flights and extracts booking links, but do not attempt to complete a purchase on a booking site.
- **Hotels/rental cars**: Use other tools for non-flight travel searches.
- **Historical price data**: Google Flights shows current prices, not historical.

## Session Convention

- **Economy + Business comparison** (default): `--session econ` and `--session biz`
- **Single cabin search**: `--session flights`
- **Interactive fallback**: `--session flights`

## Fast Path: URL-Based Search (Preferred)

Construct a URL with a natural language `?q=` parameter. Loads results directly. 3 commands total.

### URL Template

```
https://www.google.com/travel/flights?q=Flights+from+{ORIGIN}+to+{DEST}+on+{DATE}[+returning+{DATE}][+one+way][+business+class][+N+passengers][&gl=XX]
```

### Default: Economy + Business Comparison

Run two parallel sessions (economy and business) to show the price delta:

```bash
# Launch both in parallel
agent-browser --session econ open "https://www.google.com/travel/flights?q=Flights+from+BKK+to+NRT+on+2026-03-20+returning+2026-03-27" &
agent-browser --session biz open "https://www.google.com/travel/flights?q=Flights+from+BKK+to+NRT+on+2026-03-20+returning+2026-03-27+business+class" &
wait

# Wait for both to load
agent-browser --session econ wait --load networkidle &
agent-browser --session biz wait --load networkidle &
wait

# Snapshot both
agent-browser --session econ snapshot -i
agent-browser --session biz snapshot -i

# Close biz (only needed for delta column); keep econ alive for booking links
agent-browser --session biz close
```

### One Way

Add `+one+way` to the URL:

```bash
agent-browser --session flights open "https://www.google.com/travel/flights?q=Flights+from+LAX+to+LHR+on+2026-04-15+one+way"
agent-browser --session flights wait --load networkidle
agent-browser --session flights snapshot -i
```

### What Works via URL

| Feature | URL syntax | Status |
|---------|-----------|--------|
| Round trip | `+returning+YYYY-MM-DD` | Works |
| One way | `+one+way` | Works |
| Business class | `+business+class` | Works |
| First class | `+first+class` | Works |
| N passengers | `+N+passengers` | Works |
| Adults + children | `+2+adults+1+child` | Works |
| IATA codes | `BKK`, `NRT`, `LAX` | Works |
| City names | `Bangkok`, `Tokyo` | Works |
| Dates as YYYY-MM-DD | `2026-03-20` | Works (best) |
| Market/locale | `&gl=TH` (country code) | Works |
| **Premium economy** | `+premium+economy` | **Fails** |
| **Multi-city** | N/A | **Fails** |

### Reading Results from Snapshot

**If the snapshot shows the "Search results" heading with only a "Reload" button and no flight links, the results pane failed to load (the "Oops, something went wrong" state — the form parsed fine; only results failed). Click "Reload" once and re-snapshot.** If results still don't appear, this is Google soft-blocking automated traffic on the results endpoint — not a bad route or date. Don't keep hammering it; see Troubleshooting.

Each flight appears as a `link` element with a full description:

```
link "From 20508 Thai baht round trip total. Nonstop flight with Air Japan.
     Leaves Suvarnabhumi Airport at 12:10 AM on Friday, March 20 and arrives
     at Narita International Airport at 8:15 AM on Friday, March 20.
     Total duration 6 hr 5 min. Select flight"
```

Parse into a markdown table (see Output Format below).

## Market Selection Strategy

Different country markets return different prices for the same route. Always try:

1. **Departure country market first** (`&gl=US` for flights from the US)
2. **Destination country market** (`&gl=JP` for flights to Japan)
3. **Ask the user** before trying additional markets

The `&gl=XX` parameter sets the market. Use ISO 3166-1 alpha-2 country codes.

## Booking Options Handoff

After presenting results, offer booking links. When the user picks a flight:

```bash
agent-browser --session econ click @eN    # Click the flight's link
agent-browser --session econ wait 3000
agent-browser --session econ snapshot -i  # Shows booking providers with prices and URLs
```

## Output Format

**Always use markdown tables** for flight results.

### Economy + Business comparison (default)

| # | Airline | Stops | Duration | Depart | Arrive | Economy | Business | Delta |
|---|---------|-------|----------|--------|--------|---------|----------|-------|
| 1 | JAL | Nonstop | 5h 55m | 8:05 AM | 4:00 PM | $523 | $1,490 | +185% |
| 2 | THAI | Nonstop | 5h 50m | 10:30 PM | 6:20 AM+1 | $628 | $1,675 | +166% |

### Economy only

| # | Airline | Stops | Duration | Depart | Arrive | Price |
|---|---------|-------|----------|--------|--------|-------|
| 1 | JAL | Nonstop | 5h 55m | 8:05 AM | 4:00 PM | $523 |

### Format rules

- One row per flight
- For connections, show stop cities (e.g., "1 stop via ICN")
- No code blocks around the table
- After the table, highlight cheapest, fastest, and best value

## Interactive Workflow (Fallback)

Use for multi-city, premium economy, or when the URL fast path fails. Drive the form directly instead of relying on the `?q=` parameter:

```bash
# 1. Open the flights home and snapshot the form
agent-browser --session flights open "https://www.google.com/travel/flights"
agent-browser --session flights wait --load networkidle
agent-browser --session flights snapshot -i

# 2. Set trip type / cabin if needed (click the control, re-snapshot, click the option)
agent-browser --session flights click @eN      # e.g. the "Round trip" / "Economy" dropdown
agent-browser --session flights snapshot -i
agent-browser --session flights click @eN      # the desired option (One way, Multi-city, Business, etc.)

# 3. Fill origin and destination (use fill, not type — it clears existing text)
agent-browser --session flights fill @eN "BKK"
agent-browser --session flights wait 2000       # autocomplete needs an API roundtrip
agent-browser --session flights snapshot -i
agent-browser --session flights click @eN       # CLICK the suggestion, never press Enter
# repeat for the destination field

# 4. Open the date picker, snapshot, click the departure and return dates
agent-browser --session flights click @eN
agent-browser --session flights snapshot -i
agent-browser --session flights click @eN       # departure date cell
agent-browser --session flights click @eN       # return date cell (round trip)

# 5. Search and read results
agent-browser --session flights click @eN       # "Search" / "Done"
agent-browser --session flights wait --load networkidle
agent-browser --session flights snapshot -i
```

For **multi-city**, pick "Multi-city" in step 2, then repeat steps 3–4 for each leg (the form adds a new origin/destination/date row per leg). Re-snapshot after every interaction — refs (`@eN`) change as the DOM updates. The same primitives in the Key Rules table below apply throughout.

## Key Rules

| Rule | Why |
|------|-----|
| Prefer URL fast path | 3 commands vs 15+ interactive |
| `wait --load networkidle` | Smarter than fixed `wait 5000` |
| Use `fill` not `type` for airports | Clears existing text first |
| Wait 2s after typing airport codes | Autocomplete needs API roundtrip |
| Always CLICK suggestions, never Enter | Enter is unreliable for autocomplete |
| Re-snapshot after every interaction | DOM changes invalidate refs |
| Keep results session alive for booking links | Close biz session after capturing delta |

## Troubleshooting

- **"Oops, something went wrong" / empty results with a "Reload" button**: The form populates correctly (origin, destination, dates all right) but the results pane fails to load. This is Google rate-limiting / bot-detecting automated browser traffic on the results endpoint — it is **not** a date or route error, and no CAPTCHA is shown. Clicking the "Reload" button under "Search results" + re-snapshot *sometimes* clears it (mainly on a cold first request); under repeated automated use the state persists and Reload does not help. Mitigations: reduce parallel sessions (run one at a time, not econ+biz together), space out requests, and retry after a pause. If it stays blocked, fall back to the interactive workflow or — better — to another flight source (Duffel, Ignav, Skiplagged, Kiwi) that doesn't depend on scraping Google.
- **Consent popups**: Click "Accept all" or "Reject all" in the snapshot.
- **URL fast path didn't work**: Fall back to interactive.
- **Bot detection / CAPTCHA**: Inform user. Do NOT solve CAPTCHAs.
