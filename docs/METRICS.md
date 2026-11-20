# Rolling Stats metric definitions

This file is the source of truth. No stat ships on the site unless it is defined here first, with a formula. The site glossary is rendered from this file.

**Status: draft.** Every definition below depends on fields in the 511 stop-observation archive that have not yet been checked against real Muni data (see `docs/PLAN.md`, Milestone 0). Definitions may change after the spike; thresholds marked *(provisional)* will be set from real distributions.

Each stat states whose it is:
- **Vehicle** — plausibly a property of the car itself.
- **Assignment** — mostly a property of the route, hour, operator and traffic the car was given. Shown as context, never as a ranking of the car.

## Building blocks

| Term | Definition |
|---|---|
| Trip | One (`trip_id`, `trip_start_date`, `vehicle_id`) on routes J, K, L, M, N or T with at least 50% of scheduled stops observed. Bus substitutions (KBUS, NBUS, etc.) are excluded. |
| Service day | A service date on which the car ran at least 2 trips. |
| Segment | A consecutive pair of stops within a trip. The first and last segment of every trip are dropped (terminal snapping errors). |
| Segment time | Next stop's observed arrival minus this stop's observed departure. If departures are not populated for Muni, arrival-to-arrival is used, and the stat is named "stop-to-stop time" and described as including dwell. |
| Hour bucket | early (before 6), am peak (6–9), midday (9–15), pm peak (15–19), evening (19–22), night (22 and later). |
| Day type | weekday, Saturday, Sunday/holiday. |
| Baseline | Median segment time per (route, direction, from stop, to stop, hour bucket, day type), computed over all cars except the one being scored. |
| Season | Calendar year. A "last 3 months" window is also published. |

## Stats

### Trips — Vehicle
Count of trips in the window.

### Days in service % — Vehicle
`service days ÷ eligible days`. Eligible days are dates the Metro ran, from the car's first seen date onward, excluding flagged outage days.

### Longest streak — Vehicle
Most consecutive eligible days that were service days. Outage days neither extend nor break a streak.

### Estimated miles — Vehicle
Sum over the car's trips of GTFS shape distance between first and last observed stop. Yard moves are not counted.

### Time saved per trip (pace) — Vehicle, only if it passes the reliability gate
1. Per segment: `residual = baseline − segment time`, capped to ±300 s.
2. Per car: `r̄ = mean(residual)` over `n` segments.
3. Shrink toward the fleet mean `μ`: `θ = (n·r̄ + k·μ) ÷ (n + k)`, where `k = within-car variance ÷ between-car variance`.
4. Publish `θ × typical segments per trip`, in seconds. Positive is faster than typical.

### Consistency — Vehicle, only if it passes the reliability gate
Median absolute deviation of the car's segment residuals, in seconds. Lower is better.

### On-schedule rate — Assignment
Share of segments with `segment time ≤ scheduled segment time`. Ranked on line pages. Shown unranked on car pages.

### Coupled with — Vehicle
Partner cars, by share of the car's trips run together. Method is fixed after Milestone 0: shared trip in the archive if both cars are recorded, otherwise co-location (median separation under 50 m with at least 10 moving samples) from archived positions.

### Last seen — Vehicle
Most recent service day.

## Not offered

- **Top speed.** The archive has no speed field. Not published.
- **Similarity scores.** They mostly rediscover "same line"; not published.
- **Performance against schedule as the expected value.** Schedules include padding; the segment baseline is used instead.

## Ranking rules

- **Qualifying:** at least 50 trips and at least 25% of the median car's trips in the window *(provisional)*.
- **Percentile:** `100 × (qualified cars this car strictly beats) ÷ (n − 1)`, rounded. Ties do not count as beating.
- **Peer group:** all Metro LRVs in the season. Where Siemens and Breda cars overlap in a season they are ranked separately.
- **Unqualified cars** are listed with a hatched bar and no percentile number.
- **Reliability gate:** split the season into odd and even service days and correlate each stat across qualified cars. A stat is ranked only if that correlation is at least 0.4 *(provisional)*. The measured value is printed in the glossary. Stats that fail are shown as plain numbers.

## Two-car trains

Both cars in a train receive identical trip stats, because they made the same trip. Individual pace is only partly separable, through cars changing partners. The glossary states this next to every pace stat.

## Bad data rules

Rows are dropped, and counted in the build's data-quality report, when:
- segment time is zero or negative;
- implied segment speed exceeds 65 mph;
- the row duplicates another (same trip, date, stop sequence, vehicle);
- `schedule_relationship` is not scheduled;
- the vehicle id is missing or not in `data/roster.yaml` as a Metro LRV.

A service date is flagged as an outage day when its row count is under half the trailing 28-day median for that day type. Outage days are excluded from availability denominators.
