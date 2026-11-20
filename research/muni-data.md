# Muni data feasibility spike (2026-11-20)

Labels: **[V]** verified in real data pulled during the spike, **[D]** documentation only, **[I]** inferred, **[U]** not verified.

**No 511 API key was available. 511 calls against a key: 0.** Everything about the live feed and the stop-observation archive is unverified. Responses were analysed in memory and not cached (the session was in a read-only mode), so `research/cache/` is empty and the DataSF queries below must be rerun to be reproduced.

## A. 511 GTFS-RT — blocked

- [V] `https://api.511.org/transit/vehiclepositions?agency=SF` without a key returns HTTP 401.
- [V] No keyless mirror: Mobility Database lists only 511 for SFMTA realtime (mdb-1843, 1844, 1845, all needing `api_key`); Umo IQ's agency list does not include SF Muni.
- [U] id/label format per mode, `multi_carriage_details`, whether both cars of a train appear, heritage and cable car presence, update cadence.

Calls to run once a key exists (10 total, key redacted as `API_KEY`):
1. `https://api.511.org/transit/vehiclepositions?api_key=API_KEY&agency=SF` × 4, about 75 s apart.
2. `https://api.511.org/transit/tripupdates?api_key=API_KEY&agency=SF` × 1.
3. `https://api.511.org/transit/vehiclepositions?api_key=API_KEY&agency=RG` × 1.
4. `https://api.511.org/transit/VehicleMonitoring?api_key=API_KEY&agency=SF&format=json` × 1.
5. `https://api.511.org/transit/datafeeds?api_key=API_KEY&operator_id=RG&historic=2026-10-so` × 1 (about 500 MB).
6. Same with `historic=2022-03-so` × 1, to confirm backfill depth.
7. One spare.

There is no documented endpoint listing available months [D].

## B. 511 historic stop observations — documentation only

- [D] 511 transit spec, Appendix E: monthly historic feeds from December 2019; `stop_observations.txt` from March 2022; trip ids get a hash suffix; a missing day is filled from the previous day.
- [D] Columns, from Interline's write-up (https://www.interline.io/blog/mtc-regional-stop-observations/): `trip_id, trip_start_time, stop_id, stop_sequence, schedule_relationship, trip_start_date, vehicle_id, scheduled_arrival_time, observed_arrival_time, observed_departure_time, scheduled_dwell_time_secs, uncertainty, occupancy_status, occupancy_percentage`.
- [D] `vehicle_id` and `observed_departure_time` are populated only when the agency's feed provides them. Rows are derived from a 15-second archive by snapping positions to shapes; the write-up warns about loop routes and GPS error.
- [I] One `vehicle_id` per row, so a two-car train most likely records one car.
- [U] Whether `vehicle_id` is populated for Muni at all. **This is the fact the project depends on.**

## C. DataSF "Transit Vehicle Location History" — verified, and dead

- [V] Dataset `x344-v6h6` ("current year") covers 2021-01-01 to 2021-12-30 only; 470,470,157 rows. `9722-grnf` is 2021; `48aa-8sj9` is 2020. Nothing after 2021 was found.
- [V] Six columns only: `vehicle_position_date_time, vehicle_id, loc_x, loc_y, heading, average_speed`. No route, trip, direction or block.
- [V] About 1.2–1.6M rows per day; 1,038 distinct vehicles on 2021-12-01; 2021-12-24 and 12-31 missing.
- [V] Ping interval 60 s for LRVs, cable cars and PCCs; about 30 s for a sampled bus. Parked vehicles keep pinging (56% of rows have speed 0). Speed 255 is a sentinel.
- [V] Present: cable cars 1–28 and 49–60, PCCs, Breda 1400–1550, Siemens 2001–2070, trolleybuses, hybrids.
- [V] Licence: PDDL, attribution SFMTA.
- [V] **Two-car trains: both cars report separately.** Co-location test on 2021-12-01 07:30–09:30 (median nearest-in-time distance under 50 m, at least 10 samples with both cars moving) found 52 pairs; all 104 cars were in exactly one pair; zero Breda–Siemens pairs. The window hit a 50k-row query cap, so it undercounts.

Sample row: `{"vehicle_position_date_time":"2021-12-01T08:04:37.000","vehicle_id":"1062","loc_x":"-122.40107","loc_y":"37.789433","heading":"30","average_speed":"10"}`

## D. Static GTFS and roster

- [V] Official GTFS: `https://muni-gtfs.apps.sfmta.com/data/muni_gtfs-current.zip`. 68 routes: 58 bus, 7 tram (F, J, K, L, M, N, T), 3 cable car. No E route. `block_id` is populated on all 34,668 trips. `route_type` does not distinguish trolleybus from motor bus.
- [V] No machine-readable official roster found. Wikipedia's "San Francisco Municipal Railway fleet" table gives:

| Type | Numbers |
|---|---|
| Siemens S200 LRV4 | 2001–2219 (the page also says 2001–2249 in one row) |
| Breda LRV2 / LRV3 | 1400–1476 / 1477–1550 (retired 2021–2025) |
| PCC | 1006–1011, 1015, 1040, 1050–1053, 1055–1063, 1070–1080 |
| Peter Witt | 1807–1895 (11 cars) |
| Cable cars | Powell 1–28, California 49–60 |
| New Flyer XT40 / XT60 trolleybus | 5701–5885 / 7201–7293 |
| XDE60 hybrid | 6500–6554, 6560–6697, 6700–6730 |
| XDE40 hybrid | 8601–8662, 8701–8780, 8800–8969 |
| ENC 32-ft | 8531–8560 |
| Battery-electric test | 5001–5012 |

- [V] This classifies 1,007 of 1,038 ids observed on 2021-12-01 (97%). Unclassified ids look like non-revenue vehicles [I].
- [I] Heritage car "1" collides with cable car 1; keys must be mode-qualified.
- [U] "Metro is all Siemens now" was not checked against observed ids.

## E. Licences

- [V] 511 Data Disseminator Agreement (2026): may use, copy, store, distribute and create derivative works; may not resell as received; must show "data provided by 511.org" (or "powered by 511.org") with a link, in visual proximity; must send MTC documentation within 30 days of launch; no agency logos; MTC may start charging with 90 days' notice. Default limit 60 requests per 3600 seconds.
- [V] SFMTA GTFS licence: revocable right to use and redistribute; no SFMTA or Muni logos; derivatives must carry "Reproduced with permission granted by the City and County of San Francisco…" and an as-is disclaimer.

## Feasibility matrix

See `docs/PLAN.md`.
