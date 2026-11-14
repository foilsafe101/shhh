# Shhh — Project Status

**Last updated:** 2026-11-14

## What it is
Web map identifying locations with no audible anthropogenic sound (no flights, no roads, no rail). Leaflet frontend, flight data from OpenSky Trino, quiet-zone overlay computed from flight + road/rail proximity.

- **Repo:** https://github.com/foilsafe101/shhh
- **Live:** https://foilsafe101.github.io/shhh/ (GitHub Pages from `/docs`)
- **Audience:** Mark Tribe, SVA faculty NYC, field-recording scouting (Northeast US priority)

## Recovered Air dataset (2026-11-14)

The previously unpushed MacBook Air dataset has been recovered without changing
or deleting the original checkout. The October handoff understated the track
count by a factor of ten: the current file contains **3,339,319 tracks**, not
339,319, versus 238,339 in the previous published version.

- The canonical progress file contains exactly the 302,400 expected unique
  schedule keys, with no missing or unexpected keys.
- The main gzip file is 54,146,799 bytes, below GitHub's 100 MB limit.
- Both legacy gzip files are preserved byte-for-byte.
- All 238,339 previously published main-file records are retained.
- Tracks occur in every sampled month, from January 6 to December 7, 2025.
  All 8,247,999 coordinates are finite and within latitude/longitude bounds;
  no track has an empty path.
- The overlay has been regenerated using the current main-branch code and
  approved 0.145° flight threshold / 0.1° resolution, including roads and rails.
- Updated progress is 100% of the recorded sampling schedule, not proof that
  every query returned aircraft or that every location is acoustically quiet.
  Historical query logs are unavailable; collection completeness cannot be
  independently reconstructed for empty slots.

No new OpenSky queries were issued. The backfill workflow remains disabled.

## ⚠️ Open issue (2026-10-11): Trino refuses queries from GitHub Actions

The workflow was merged and test-run on 2026-10-11. The runner authenticates
with the `OPENSKY_USERNAME` / `OPENSKY_PASSWORD` secrets, but every query is
refused by Trino with `PERMISSION_DENIED: Access Denied: Cannot execute query`.
So the account behind the secrets is not authorised to run queries: either the
secrets hold a different OpenSky account than the one Martin Strohmeier
approved (`mtribe@sva.edu`), or that approval has lapsed since May.

The workflow is **disabled** in the Actions tab until this is resolved. To check
which it is, run a query on the Mac with the saved browser login:

```
cd /Users/Shared/shhh && .venv/bin/python -c "from pyopensky.trino import Trino; print(Trino().query('SELECT 1 AS ok'))"
```

If that works, re-enter the two secrets with that account's credentials and
re-enable the workflow. If it is refused too, access has to be re-requested
from OpenSky.

Before the fix below, a refused query was recorded as a completed slot with no
data; the cancelled first run pushed 150 such slots, which have been reverted.
`query_opensky_paths.py` now leaves a refused slot pending and aborts after
five consecutive errors, and the workflow preflight runs a test query so a
refused account fails in seconds.

## How the backfill runs now
`.github/workflows/opensky-backfill.yml` — scheduled GitHub Actions, no laptop involved.

- Runs every 3 hours; each run queries for up to 5 hours, commits what it finished, exits. (Actions caps a job at 6 hours.)
- A concurrency group keeps exactly one worker alive at a time, which also respects OpenSky's limit of 2 concurrent queries for this account.
- When no slots are pending, runs exit immediately at the pending check, so leaving it enabled costs nothing.
- Credentials come from the `OPENSKY_USERNAME` / `OPENSKY_PASSWORD` repo secrets. pyopensky uses them for a password-grant token exchange; **without both it falls back to browser OAuth and hangs**, which is what tied this to the Mac. A preflight step fails loudly rather than hanging.
- To run it by hand: Actions tab → "OpenSky backfill" → Run workflow.

`run_worker.sh` is the old Mac supervisor. It hardcodes `/Users/Shared/shhh` and `.venv`, and it carries the staging bug above — it is superseded by the workflow.

## Key files
- `docs/index.html` — Leaflet map, canvas layer (`FlightCanvas`); loads `flights.json.gz`, `flights_0.json.gz`, `flights_1.json.gz` in parallel
- `docs/quiet_overlay.png` — Web Mercator RGBA overlay (3600×2548, ~1.7 MB), dark blue = quiet
- `docs/flights.json.gz` — current single-worker output
- `docs/flights_0.json.gz`, `docs/flights_1.json.gz` — legacy 3-minute-era worker outputs, still loaded by the map
- `query_opensky_paths.py` — OpenSky Trino queries; `--worker 0|1` splits regions 0–14 / 15–29 into separate progress and output files (unused now)
- `generate_quiet_overlay.py` — cKDTree distance from flight + road/rail samples → Web Mercator PNG
- `gen_status.py` — writes `docs/status.json` from `query_progress_paths.json`

## Sampling configuration
30 uniform tiles (30°×60°, 5 lat bands × 6 lon columns, lat −60° to 90°). Replaced the older 11 hand-drawn regions on 2026-05-03.

- `SAMPLE_INTERVAL_MINUTES = 12` (was 3 until 2026-05-05)
- 120 slots/day × 7 days × 12 sample weeks of 2025 × 30 regions = **302,400 total slots**
- `PAUSE_SECONDS = 0`, `RDP_TOLERANCE = 0.1`, `MIN_ALT_M = 300`, `MAX_ROWS_PER_SLOT = 3000`

## Acoustic thresholds
- Flights audible: **0.145°** (~10 mi)
- Roads/rail audible: 0.015° (~1 mi)
- Fully quiet: 1.5° from any noise source
- Overlay resolution: **0.1°** (~11 km at equator)

Both of the changes that were gated on 80% coverage are now applied. At 0.1° the grid is 3600×2548 = 9.2M pixels — a 6.25× pixel increase that produces a 1.7 MB PNG in about 70 seconds using ~1.4 GB RAM. Well within a runner's budget, and small enough to serve.

The overlay now uses the recovered Air dataset. Sparse receiver coverage and the sampling design still limit what absence of recorded flights can establish.

## Data size headroom
The recovered main gzip file is 54.1 MB (3,339,319 tracks). This is below
GitHub's 100 MB file limit, but substantial for a browser download. The existing
workflow warns at 50 MB and fails at 90 MB. Regional or sample-week splitting
remains a possible future performance improvement.

## Architecture decisions / declined options
- **Cloud VM**: declined earlier in favour of Mark's Mac; superseded by GitHub Actions (free for public repos)
- **Web Workers for rendering**: declined — canvas layer is fast enough
- **PMTiles tile-based rendering**: deferred — revisit if rendering slows again

## Still open
- Historical empty-query outcomes are not independently auditable without logs
- `fetch_flights.py` writes `docs/flights.json`, which is in `.gitignore`, so the nightly workflow's `git add` cannot stage it. Worth confirming whether that job does anything today.
- Altitude-dependent audible radius — low-flying aircraft carry a much smaller radius than jets
