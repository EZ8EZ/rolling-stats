# Rolling Stats — plan

## Context

Rolling Stats is a fan site giving every Muni vehicle a Baseball Savant-style "player page", modelled on mbtasavant.com. Research (three parallel agents, 2026-11-20) found the model site is a thin layer over a rich daily file the MBTA publishes (travel time, dwell, schedule, full consist). **Muni has no verified equivalent.** No 511 API key was available, so zero 511 calls were made, and the one keyless source (DataSF vehicle history) is frozen at 2021-12-30. The plan is therefore gated: no application code until a data spike passes.

Decisions made by the owner:
- MVP = Muni Metro LRVs only. Buses, heritage streetcars, cable cars later.
- Freshness = monthly rebuild from the 511 stop-observation archive. No poller unless the spike shows the archive cannot produce pairings.
- Hosting = public GitHub repo, GitHub Actions pipeline, Vercel (free URL for now).
- Stack = Python + DuckDB pipeline, Astro static site.
- If the spike fails outright: pivot to SFO aircraft tail numbers (after an equally hard spike on that data).
- Repo name stays `rolling-stats`; the owner has allowed renaming but there is no reason to.

Labels: **[V]** verified with real data, **[D]** documentation only, **[I]** inferred, **[U]** unverified.

## Feasibility matrix

| # | Feature | Source | Fields | Confidence | Caveat |
|---|---|---|---|---|---|
| F1 | Days and trips per car | 511 `historic=YYYY-MM-so` | `vehicle_id, trip_id, trip_start_date` | [D] columns exist; [U] Muni fill rate | Whole site depends on this |
| F2 | Game log (routes, trips per day) | same + RG GTFS `trips/routes` | above + `route_id` | [U] | Trip ids carry a hash suffix in historic feeds [D] |
| F3 | Pace vs peers | same | `observed_arrival_time, observed_departure_time, stop_sequence` | [U] | Departure populated only "when available" [D]; times derived from 15 s archive snapped to shapes |
| F4 | Pairings | archive if both cars appear; else own position poller | `vehicle_id` per trip, or lat/lon/time | Method [V] on 2021 DataSF (52 pairs, each car in one pair, no cross-model pairs); 2022+ [U] | Not published anywhere; archive likely holds one car per trip [I] |
| F5 | Percentiles | derived + roster | — | Roster [V]: Wikipedia table classifies 97% of 2021 ids | No machine-readable official roster |
| F6 | Top speed | none | — | **Not possible** from the archive (no speed field) | Honest substitute: fastest segments relative to baseline |
| F7 | Live view | 511 VehiclePositions | — | [U]; out of scope | 60 req/hour default |
| F8 | Backfill | archive Mar 2022 onward [D] | — | [D] | Jan–Feb 2022 has no source; no endpoint lists months |
| F9 | Heritage / cable cars | DataSF 2021 [V]; 511 [U] | — | later phase | Cable car id "1" collides with heritage car "1" |

## 1. Product spec

URL scheme (all pre-rendered):
- `/` — line leaders, fleet summary, "data through" date.
- `/car/{number}` — player page: header card (number, model, build year, status, last seen), percentile sliders, season table, service calendar heatmap, coupled-with table, game log.
- `/line/{J,K,L,M,N,T}` — route page: where route-level stats (on-schedule rate, segment factors) live.
- `/leaderboards?season=` — sortable table, qualified cars; unqualified listed below, unranked.
- `/glossary` — rendered from METRICS.md.
- `/about` — fan-site disclaimer, attribution, data-quality report for the current build.

MVP features → matrix rows: days/trips (F1), game log (F2), pace + consistency (F3), coupled-with (F4), percentiles + leaderboard (F5). Retired Breda cars appear with a "retired" badge where backfill covers them.

Later phases: buses and trolleybuses; heritage and cable cars (counting stats only, no percentiles); rolling windows; car comparison; custom leaderboard with CSV; segment "venue" pages; live view. Rejected: raw top speed, similarity scores, schedule as the "expected" value.

## 2. Metric spec (becomes `/docs/METRICS.md`)

Every entry carries: definition, formula, "whose stat is this" (Vehicle / Assignment), qualifier, measured reliability.

- **Trip**: one (`trip_id`, `trip_start_date`, `vehicle_id`) with ≥ 50% of scheduled stops observed, on routes J K L M N T (bus substitutions excluded).
- **Service day**: a service date with ≥ 2 trips.
- **Days in service %**: service days ÷ days the line ran since the car's first seen date. Vehicle stat.
- **Longest streak**: consecutive service days; days with a feed outage do not break it.
- **Segment time**: next stop observed arrival − this stop observed departure. If departure is unpopulated, arrival-to-arrival, renamed "stop-to-stop time" and stated as including dwell.
- **Baseline**: median segment time per (route, direction, from stop, to stop, hour bucket, day type), computed across all cars excluding the car being scored.
- **Time saved per trip (pace)**: mean of (baseline − actual) per segment, each capped at ±300 s, scaled to the line's typical segments per trip. Then shrunk toward the fleet mean: θ = (n·r̄ + k·μ)/(n + k), k estimated from within/between variance.
- **Consistency**: median absolute deviation of segment residuals, seconds; lower is better.
- **On-schedule rate**: share of segments at or under scheduled time. Shown on route pages; on car pages it is displayed unranked, labelled Assignment.
- **Miles (estimated)**: sum of shape distance over observed trips.
- **Coupled with**: partner cars by share of trips; method fixed after the spike.
- **Reliability gate**: a stat gets a percentile slider only if split-half reliability (odd vs even service days, across qualified cars) is ≥ 0.4. Otherwise it is shown as plain context. Expect pace to bunch near 50; that is the honest result.
- **Qualifying**: ≥ 50 trips and ≥ 25% of the median car's trips in the window. Unqualified cars get a hatched bar, no number.
- **Percentile**: 100 × (qualified cars strictly beaten) ÷ (n − 1). Peer group: all Metro LRVs in that season; Siemens and Breda ranked separately where both exist.
- **Two-car trains**: both cars receive identical trip stats. The glossary says so; individual pace is only partly separable through changing partners.
- **Bad data**: drop negative or zero segment times, implied speed > 65 mph, duplicate rows, non-scheduled `schedule_relationship`, and first/last segment of each trip (terminal snapping). Flag days with row counts under half the trailing median and exclude them from availability denominators.

## 3. Data architecture

The Parquet/DuckDB/static-build shape is right and is what the model site uses [V]. With monthly builds there is no database server and no poller.

Flow: 511 monthly zip → stream-filter to SFMTA rail → `raw/` Parquet (one file per month, stored as GitHub Release assets) → DuckDB transforms → `site-data/*.json` → Astro build → Vercel deploy from the Action.

Tables: `vehicles` (number, mode, model, status, first/last seen), `models`, `service_days`, `trips`, `stop_events`, `segment_baselines`, `pairings`, `daily_vehicle_stats`, `season_vehicle_stats`. Roster is `data/roster.yaml`, hand-maintained, each range citing its source.

Layout: `pipeline/` (Python package, CLI: `fetch`, `transform`, `stats`, `publish`, `quality`), `web/` (Astro), `data/roster.yaml`, `docs/`, `research/`, `.github/workflows/monthly.yml`.

## 4. Backfill

March 2022 to present is about 56 monthly files at roughly 500 MB each (~28 GB download, one API call per month, well inside the rate limit). Each month is processed alone on an Actions runner and reduced to a small rail-only Parquet (size estimate is a guess until the spike: single-digit MB per month). Start with the latest 12 months; extend backward only after M2 acceptance. Cost: $0 on a public repo.

## 5. Front-end

Astro, pre-rendered, mobile-first (390 px baseline, tables scroll inside their container, sticky rank and car columns). Small client islands for sorting and the calendar heatmap; charts as inline SVG, no chart library unless needed. Visual language borrowed from Savant: percentile slider with number bubble, blue `#3661AD` → grey plateau 45–55 → red `#D82129`, hatched bars for unqualified, card header with car-number badge, per-stat popover linking to the glossary. No SFMTA or Muni logos. Footer on every page: "Fan site, not affiliated with SFMTA", "Data provided by 511.org" with link, and the SFMTA GTFS notice text.

## 6. Milestones

- **M0 — Data spike (gate).** Copy research into `/research`, capture the blocked screenshots, run the 10 budgeted 511 calls, download one month, cache everything. Pass criteria: Muni rail `vehicle_id` populated on ≥ 90% of rows; arrival times present; trips join to routes; a written answer on whether two-car trains show one car or two. Output: `research/SPIKE.md`. Fail → stop, run the SFO tail-number spike, re-plan.
- **M1 — Pipeline for one month.** Acceptance: car 2010's trips for 3 sampled days match a hand count from the raw rows; data-quality report generated.
- **M2 — Metrics.** Every METRICS.md stat implemented with fixture tests; reliability table produced; stats failing the gate demoted.
- **M3 — Pairings.** From the archive, or add a once-a-minute position poller if the spike requires it. Acceptance: ≥ 95% of paired cars have exactly one partner per trip; no pair spans two lines at once.
- **M4 — Site.** All MVP pages live on the Vercel URL; Playwright screenshots at phone and desktop sizes reviewed.
- **M5 — Backfill and launch.** 12 months loaded, monthly workflow green twice, MTC notified within 30 days of launch as the 511 agreement requires.

## 7. Verification

- Unit tests (pytest) for every metric function, using small real-data fixtures cut from the spike month.
- `quality` command writes a report each build: rows per day, vehicle-id fill, dropped-row counts by rule, missing days, cars absent from the roster. Build fails if fill drops below the M0 threshold.
- Playwright screenshots of home, a car page, a leaderboard and the glossary at 390 px and 1440 px on every build.
- Hand check: 3 cars × 3 days against raw rows before launch.

## 8. Risks, ranked

1. Muni vehicle ids missing or sparse in the archive → pivot to SFO tail numbers (owner's choice).
2. Archive has one car per train → add position poller for pairings; history starts that day.
3. Pace fails the reliability gate → site leads with availability and counting stats; pace shown as context.
4. No observed departures → stop-to-stop time, dwell stat dropped.
5. Monthly files late or missing → site shows "data through" date prominently; no fake freshness.
6. Roster drift → build-time report of unclassified ids.
7. Licence change or 511 charging (90 days' notice allowed) → raw months already archived in releases.

## 9. Repo conventions (becomes `CLAUDE.md`)

Build/test commands; METRICS.md is the source of truth and no stat ships without a formula there first; secrets only in `.env` (gitignored) with `.env.example`, key name `API_511_KEY`; never commit raw 511 zips; cache every fetched response under `research/cache`; every claim in docs labelled verified/inferred/assumed; mode-qualified vehicle keys (`lrv:2010`).

## Rate-limit email (only needed if a poller is added)

> Subject: 511 Open Data rate-limit increase request
> I run Rolling Stats, a non-commercial fan site showing per-vehicle statistics for SFMTA rail using 511 open data, with "data provided by 511.org" attribution. To archive VehiclePositions and TripUpdates for agency SF once a minute I need about 120 requests per hour. Could the limit on my key be raised to 180 per hour? Happy to share the project documentation.

## First implementation steps after approval

1. Commit `docs/PLAN.md`, `docs/METRICS.md`, `CLAUDE.md`, `.gitignore`, `.env.example`, `research/` reports.
2. Owner adds `API_511_KEY` to the environment; a new session runs M0.
