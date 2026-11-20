# Baseball Savant concept mapping (2026-11-20)

Savant facts below were read from Savant pages, its player-page JavaScript, or MLB glossary pages unless marked [I] (inferred) or [U] (unverified). Transit analogs are proposals. Screenshots: `research/screenshots/savant-*.png`.

## The main point

Most of what arrival data shows about a vehicle is decided by its assignment (route, segment, hour, operator, traffic), not by the vehicle.

| Metric family | Defensible as the vehicle's? |
|---|---|
| Service days, availability | Yes |
| Mid-trip dropouts or swaps | Yes, as a breakdown proxy |
| Miles, trips | Yes, as counting stats |
| Raw speed, on-time %, delay | No — route and yard pages |
| Run time vs segment × hour baseline | Partly — needs shrinkage and a reliability test |

Controls: leave-one-out segment × hour × day-type baselines; peer groups by mode/model; shrinkage toward the model mean; publish a percentile only if split-half reliability is about 0.4 or higher. Coupled cars get identical trip stats and cannot be fully separated.

## Features

| Savant feature | How it works | Transit analog | Verdict |
|---|---|---|---|
| Percentile sliders | Share of qualified players beaten, 1–100. Colour `d3.scaleLinear` range `#3661AD, #b4cfd1, #b4cfd1, #D82129` over domain 5, 45, 55, 95. Unqualified players get a hatched bar at 0.35 opacity, no bubble. Qualifier: 2.1 PA per team game (batters). | Same, within mode/model, on shrunk estimates | Adopt, including hatched bars |
| Sprint Speed | Feet per second in the fastest one-second window; best two-thirds of competitive runs; minimum 10 runs | Baseline-relative segment speed | Reject raw top speed (pings too coarse, mostly measures route) |
| Expected Stats | Hit probability from exit velocity and launch angle; column is actual minus expected | Run time vs segment baseline | Adopt with segment baseline; reject schedule as the expectation |
| Park Factors | 100 = average; same players at this park vs elsewhere; 3-year rolling | Segment factor = median time ÷ free-flow time | Adopt the idea; vehicles don't move between "parks" enough for Savant's method |
| Rolling Windows | Last N PA vs previous N (50/100/250) | Last N trips vs previous N | Later; a change usually means reassignment |
| Year to Year | Must qualify in both seasons | Month or year change, minus the fleet's change | Later |
| Custom Leaderboard | Column picker, minimum PA, beeswarm/scatter | Column picker with CSV | Later |
| Gamefeed | Live game list with event icons | "Today" board | Later; needs live data |
| Statcast Search | Query builder | Query over segment runs | Later |
| Player Comparison | Side-by-side sliders | Same-model cars or one car across periods | Later |
| Similarity Scores | Method not published [U] | Usage or performance twins | Gimmick — rediscovers "same line" |
| Glossary | Definition, then "why it's useful" | Add "whose stat is this" and measured reliability | Adopt |
| Game log, season table, header card | Per-game rows; mini season line | Per-service-day rows | Adopt |

## Other ideas worth keeping

- Calendar or "where it ran" map per car.
- Route and line pages as the home for assignment stats.
- A counting stat that rewards volume (MBTA Savant's Trips Above Replacement is this).
- Splits by line, direction and hour block.

Not verified: Savant's similarity method, the comparison tool's contents, the exact year-to-year formula, per-pitch Gamefeed content.
