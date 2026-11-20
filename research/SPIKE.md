# Milestone 0 — data spike result (2026-11-20, about 20:40 Pacific)

**Verdict: the gate passes for a per-car site, but pairings are probably not recoverable, and that weakens the availability stats too.**

511 calls used: 6 of 60 (4 vehicle positions, 1 trip updates, 1 archive download). Cached responses are in `research/cache/511/`. The archive (`raw/`, gitignored) is 624 MB zipped; the Muni subset is 135 MB of Parquet for all modes.

Everything below was measured on real data unless marked [I] (inferred).

## Gate criteria

| Criterion | Result |
|---|---|
| Muni rail `vehicle_id` on ≥ 90% of archive rows | **Pass: 100%** (840,018 Metro rows, October 2026) |
| Arrival times present | **Pass: 100%** |
| Trips join to routes | **Pass.** `route_id`, `agency_id`, `direction_id` and `service_date` are columns on every row; `trips.txt` also has `block_id` |
| Two-car trains: one car or two? | **One.** 35,323 of 35,356 Metro trip-days have a single vehicle |

## Archive (`historic=2026-10-so`)

Actual columns (they differ from the published write-up): `trip_id, trip_start_time, schedule_relationship, service_date, vehicle_id, stop_sequence, observed_arrival_time, observed_departure_time, uncertainty, dwell_time_secs, scheduled_dwell_time_secs, route_id, agency_id, direction_id, from_stop_id, to_stop_id, scheduled_arrival_time, scheduled_departure_time`.

| Mode | Rows | Vehicles | Trip ids | Scheduled time filled |
|---|---|---|---|---|
| Metro (J K L M N T) | 840,018 | 186 | 3,195 | 96.1% |
| Bus | 8,831,968 | 712 | 19,910 | 99.8% |
| F line | 123,880 | 28 | 426 | 100% |
| Cable car | 307,698 | 99 (includes stray bus ids) | 1,666 | 100% |

- All 31 days present. Metro runs about 1,220 trips on weekdays and 980 on weekends.
- **No dwell data.** Observed departure equals observed arrival on every row, and `dwell_time_secs` is always 0. Only stop-to-stop time is available.
- `uncertainty` and `schedule_relationship` are constant (0), so they cannot be used as quality filters.
- A trip id is only unique together with `service_date`.
- Duplicates (same trip, date, stop, vehicle): 1.0% of Metro rows, 1.1% of bus rows.
- Metro stop-to-stop times: median 97 s; 1.4% are zero or negative; 0.13% exceed 15 minutes.
- Metro vehicle ids run 2001–2211, plus 77 stray rows for streetcar 1058. Cable car routes include stray bus and LRV ids. A roster filter is required.
- Metro cars per trip-day: a car runs a median of 18 trips on a day it appears.
- **A car appears on a median of only 11 of 31 days, and about 70 distinct cars appear per weekday.**

Hand check for later milestones: car 2010 ran 26 trips on 2026-10-05 (L and M), 26 on 10-06, 14 on 10-07 (M).

## Live feed (`vehiclepositions?agency=SF`)

- 596 vehicles. `vehicle.id` equals the fleet number for every mode: LRVs 20xx–22xx, streetcars 10xx, cable cars 1–60.
- Historic streetcars and cable cars are present and assigned to routes F, PH, PM and CA.
- Every vehicle has position, speed, bearing and occupancy status. 360 of 596 have a trip id.
- `multi_carriage_details` is empty for every vehicle. No trip has more than one vehicle.
- Positions refresh on every poll (all vehicle timestamps moved together across four polls 75–100 s apart).
- 135 LRVs were reporting: 56 on a route, 79 without one. 77 of those 79 were sitting in the two rail yards.

## Pairings

- Co-location test across four snapshots: **zero pairs** of LRVs stayed within 60 m while moving.
- Almost no route-less LRVs were out on the line, so trailing cars were not visible as separate vehicles.
- [I] Either evening trains were all single cars, or trailing cars do not report in the 511 feed. The 2021 DataSF data did show both cars, so this is a change or a time-of-day effect. **A weekday peak-hour sample is needed to settle it.** About 10 more calls.

## Consequences for the plan

1. Trips, game log, routes run, stop-to-stop pace, consistency and on-schedule rate are all buildable from the archive. Buses, streetcars and cable cars are too.
2. "Coupled with" cannot come from the archive. Whether a poller can recover it depends on the peak-hour check.
3. **Days in service is biased if trailing cars are invisible:** a car that spent the day as the second car of a train looks like it did not run. Unless pairings are recovered, the stat must be named and described as "days leading a train" or dropped as a ranking.
4. The dwell stat is dropped.
5. Backfill cost is higher than guessed: about 135 MB per month for all Muni modes, roughly 12 MB for Metro alone [I].
