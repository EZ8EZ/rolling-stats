# MBTA Savant teardown (retrieved 2026-11-20)

All items verified from the rendered site, its JSON, or its public repo unless marked [I] (inferred) or [U] (unverified). Screenshots are in `research/screenshots/mbta-*.png` (phone 390 px and desktop 1440 px).

## Source

- Repo: `github.com/skaplan-dev/mbta-savant`, by Seth Kaplan. **No licence file was found**, so the code is not reusable. The full file tree could not be listed [U].
- No launch or methodology post was found [U].

## Pages

| Page | URL |
|---|---|
| Home (top 5 per line by TAR) | `/` |
| Line leaderboard | `/line/{red,orange,blue,green,mattapan,bus}?season={2026,2025,last30}` |
| Bus route leaderboard | `/route/{route}` (152 routes) |
| Car or bus page | `/car/{group}/{number}` (about 1,900 pages) |
| Glossary (the only methodology page) | `/glossary` |
| Search | header typeahead on every page |

HTML shells are prerendered per page with a title and description built from that night's numbers.

## Metrics (glossary wording)

- **TAR, Trips Above Replacement:** "how many more trips' worth of service this car gave than a replacement-level car on its line." Replacement = the line's 20th-percentile car. `TAR = On the field + Speed + Steadiness`. "TAR is mostly about showing up, by design."
- **Time Saved:** seconds of moving time saved per trip versus the line's typical car.
- **Dwell Time:** seconds saved per trip while stopped. "Crowds and operators affect dwell as much as the car does."
- **On-Schedule %:** how often the car makes the scheduled time between stops.
- **Consistency:** variation in stop-to-stop times, seconds; smaller is better.
- **Days in Service:** share of the line's service days the car ran (2 trips on rail, 1 on bus).
- **Iron Horse Streak:** most service days in a row; outages don't break it.
- **Miles Logged** (rail), **Routes Run** (bus), **Last Seen**, **Avg Crowding** (unranked), **Coupled With** (unranked).

From the code: expected travel time is the median per (route, direction, from stop, to stop, hour bucket, day type); per-segment delay capped at 300 s; hour buckets early / am peak 6–9 / midday 9–15 / pm peak 15–19 / evening 19–22 / night; each segment is credited to every car in the consist.

## Ranking

- Within line only. No ranking against vehicle model, none fleet-wide. Mattapan unranked.
- Qualifying: at least 50 trips and a quarter of what a typical car ran. Route boards: 20 trips on that route.
- Percentile = 100 × (qualified cars strictly beaten) ÷ (n − 1); ties don't count.
- Married pairs share identical stats.

## Pairings and game log

- Pairings come directly from a `vehicle_consist` field in MBTA data. **Nothing is inferred**, so there is no method to borrow for Muni.
- "Coupled With" table: car, trips together, share; top 5.
- Game log: date, trips, miles, time saved per trip, lines (rail) or routes (bus). 15 rows then "show all".
- Service calendar: one square per day, coloured slower → faster.

## Visual language

- Red is good (`#d62929`), blue is poor (`#3260b4`), grey middle.
- Percentile row: label, 8 px track with fill, white midline at 50, 32 px circle with the number, value at right. Grouped as "Performance" and "Availability".
- Card header: photo banner with credit, large car-number circle in the line colour, specs line.
- Leaderboard: sticky rank and car columns, cells tinted by percentile, default sort TAR. Wide table scrolls inside its container on phones.

## Plumbing

- Vite + React SPA; static JSON under `/data/`; no API. S3 + CloudFront. A car's season file is about 35 KB; a line's about 300 KB.
- Pipeline: Python 3.13 + DuckDB, nightly on GitHub Actions, per-day Parquet in S3. Author's cost estimate: "well under $1/month".
- Data: MBTA LAMP files (`performancedata.mbta.com`), which already contain travel time, dwell time, scheduled travel time and the consist. Bus history starts 2025-12-25.
- Attribution on site: "A fan site, not an official MBTA assessment."

## What this means for Rolling Stats

The MBTA publishes the hard parts pre-computed. Muni has no verified equivalent; see `research/muni-data.md`.
