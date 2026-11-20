# Rolling Stats

Fan stats site for San Francisco Muni vehicles. Not affiliated with SFMTA.

## Current state

Planning and the Milestone 0 data spike are done (`research/SPIKE.md`); no application code exists yet. One open check remains before M1: a weekday peak-hour live sample for pairings.

## Rules

- `docs/METRICS.md` is the source of truth for every stat. A new stat must be added there, with a formula and a "whose stat is this" label, before it ships.
- Never commit secrets. The 511 key lives in `.env` (gitignored) as `API_511_KEY`; `.env.example` shows the name only. Never print the key or put it in a cached filename or URL.
- 511 rate limit is 60 requests per hour. Cache every fetched response under `research/cache/` (or `raw/` for pipeline data) and never refetch what is cached.
- Never commit raw 511 zip files.
- Label claims in docs and research as verified (seen in real data), inferred, or assumed.
- Vehicle keys are mode-qualified (`lrv:2010`), because cable car and heritage car numbers collide.
- Do not use SFMTA or Muni logos. Every page carries the fan-site disclaimer, "Data provided by 511.org" with a link, and the SFMTA GTFS notice.
- Do not copy code from `skaplan-dev/mbta-savant`; it has no licence.

## Layout (planned)

- `pipeline/` — Python + DuckDB. CLI commands: `fetch`, `transform`, `stats`, `publish`, `quality`.
- `web/` — Astro static site.
- `data/roster.yaml` — hand-maintained fleet roster, each range with a cited source.
- `docs/` — plan and metric definitions.
- `research/` — research reports, screenshots, cached responses.

## Commands

None yet. Add build and test commands here when `pipeline/` and `web/` are created.
