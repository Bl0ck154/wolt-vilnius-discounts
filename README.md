# Wolt discount monitor

Universal Wolt discount dashboard and scheduled data updater. The project keeps a
public catalog of Wolt cities/countries, fetches discount snapshots per city, and
serves a static GitHub Pages dashboard from `docs/`.

Telegram notifications are intentionally scoped to the default Vilnius monitor;
other cities can be browsed and cached but do not send notifications.

## What it does

- Fetches the full public Wolt city catalog from:

  ```text
  https://restaurant-api.wolt.com/v1/cities
  ```

- Normalizes city ids as `country/city-slug`, for example:
  - `ltu/vilnius`
  - `deu/berlin`
  - `jpn/tokyo`

- Fetches city discount snapshots through public Wolt web endpoints using city
  coordinates.
- Caches snapshots in `docs/data/` and skips Wolt API calls while city data is
  fresh.
- Renders a static dashboard with country-grouped city selection.

## Wolt endpoints used

City catalog:

```text
GET https://restaurant-api.wolt.com/v1/cities
```

Promotion venues for any city coordinate:

```text
GET https://consumer-api.wolt.com/v1/pages/venue-list/promotions-near-you?lon=<lon>&lat=<lat>
```

Restaurant seed for any city coordinate:

```text
GET https://consumer-api.wolt.com/v1/pages/restaurants?lat=<lat>&lon=<lon>
```

Required header for the consumer API endpoints:

```text
Platform: Web
```

Useful offer paths in the Wolt response:

```text
sections[*].items[*].venue.promotions[*]
sections[*].items[*].venue.badges_v2[*]
sections[*].items[*].venue.promotions_for_telemetry[*]
```

## Run locally

No package install is required. The scripts use Node's built-in `fetch`.

```bash
# Refresh the full Wolt city/country catalog
npm run cities

# Update the default city (Vilnius)
npm run check

# Update one city
WOLT_CITY=deu/berlin node src/check-discounts.mjs

# Update several cities
WOLT_CITIES=ltu/vilnius,ltu/kaunas,lva/riga node src/check-discounts.mjs

# Update every Wolt city from the catalog (large run)
WOLT_ALL_CITIES=true node src/check-discounts.mjs

# Override cache TTL in hours; default is 2
WOLT_CACHE_TTL_HOURS=4 node src/check-discounts.mjs
```

PowerShell example:

```powershell
$env:WOLT_CITY="deu/berlin"; node src/check-discounts.mjs; Remove-Item Env:\WOLT_CITY
```

Open `docs/index.html` locally or use GitHub Pages after pushing.

## Data files

The updater writes static JSON files consumed by the dashboard:

```text
docs/data/city-catalog.json
docs/data/cities.json
docs/data/latest.json                         # default Vilnius snapshot
docs/data/changes.json                        # default Vilnius diff
docs/data/changes-log.json                    # default Vilnius change log
docs/data/notified-offers.json                # Vilnius notification state
docs/data/cities/<country-city-slug>/latest.json
docs/data/cities/<country-city-slug>/changes.json
docs/data/cities/<country-city-slug>/changes-log.json
```

`docs/data/cities.json` contains the full dashboard city list plus cache status
for cities that have already been fetched.

## Cache behavior

Each city has its own cache. If `latest.json` for a city is newer than
`WOLT_CACHE_TTL_HOURS` (default `2`), the updater reuses it and does not call
Wolt for that city.

Use `FORCE_WRITE=true` to bypass the freshness check.

## Notifications

Telegram notifications remain limited to Vilnius by design:

```text
TELEGRAM_BOT_TOKEN
TELEGRAM_CHAT_ID
```

Optional notification variables:

```text
MIN_DISCOUNT_EUR=3
MIN_DISCOUNT_PERCENT=20
INCLUDE_ZERO_DELIVERY=false
```

Non-Vilnius cities are cached and displayed but skipped by Telegram.

## GitHub Actions

Workflow: `.github/workflows/check-discounts.yml`

- Scheduled runs update the default monitor inside Vilnius notification windows.
- Manual runs accept:
  - `cities`: comma-separated city ids, e.g. `deu/berlin,jpn/tokyo`
  - `all_cities`: large run over the full catalog
- The job uses a self-hosted runner because Wolt currently returns `429 Too Many
  Requests` from GitHub-hosted runner IP ranges.

```yaml
runs-on: [self-hosted, Linux, X64, wolt]
```

Useful commands:

```bash
gh workflow run "Update Wolt discount monitor" --repo Bl0ck154/wolt-discount-monitor --ref main -f cities=deu/berlin
gh run list --repo Bl0ck154/wolt-discount-monitor --workflow "Update Wolt discount monitor" --limit 5
```

GitHub Pages is deployed by `.github/workflows/deploy-pages.yml` from the
`docs/` folder after pushes and successful updater runs.

## Research notes

Historical endpoint research is kept in `FINDINGS.md`. It started with Vilnius
as the first tested city, but the implementation now applies the same endpoint
patterns to any Wolt city from the catalog.
