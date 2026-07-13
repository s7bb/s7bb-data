# s7bb-data

Published punctuality data for the **Deutsche Bahn S7 S-Bahn line at Baierbrunn
station** (Munich, EVA `8000781`): arrivals, delays, cancellations, and
aggregated on-time statistics.

This repository holds **only the generated JSON data**. It is the machine-readable
backend for the site at <https://s7bb.github.io>. The code that produces it lives
in a separate repository, [`s7bb/s7bb.github.io`](https://github.com/s7bb/s7bb.github.io).

## Data source, license, and disclaimer

The underlying timetable and real-time data comes from the
**[DB Timetables API](https://developers.deutschebahn.com/db-api-marketplace/apis/product/timetables)**
(IRIS-TTS) operated by Deutsche Bahn (DB InfraGO AG / DB Station&Service AG).

- **License:** [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/).
- **Attribution:** Data source is the DB Timetables API of Deutsche Bahn,
  licensed under CC BY 4.0. The raw API data is processed and aggregated here
  (delay and punctuality analysis); it is not the original, unmodified dataset.
- **No warranty (as required by the source):** Deutsche Bahn assumes no
  liability for the completeness or accuracy of the data. Short outages of the
  fetcher or the DB API can produce gaps, so the data does not necessarily
  reflect every train.

This project is an independent, non-commercial community effort. It is **not
affiliated with, endorsed by, or operated by Deutsche Bahn**.

## Files

| File | Contents |
|------|----------|
| `latest.json` | Rolling window of the last 7 days plus live today/week aggregates. Rewritten every fetch cycle. |
| `archive/YYYY-MM.json` | One calendar month of arrivals with monthly and per-day aggregates. |
| `archive/index.json` | Lightweight index of all archived months (aggregates only, no per-arrival rows). |

Flat tree at the repository root:

```
latest.json
archive/index.json
archive/2026-05.json
archive/2026-06.json
...
```

## Usage

Consume the raw files directly over HTTPS - no API key, no build step:

```
https://raw.githubusercontent.com/s7bb/s7bb-data/main/latest.json
https://raw.githubusercontent.com/s7bb/s7bb-data/main/archive/index.json
https://raw.githubusercontent.com/s7bb/s7bb-data/main/archive/2026-06.json
```

Notes for consumers:

- **Update cadence:** `latest.json` is refreshed roughly hourly (each push from
  the fetcher VM). `generated_at` tells you when the file was produced.
- **Single writer:** this repository is written **only** by the fetcher bot, one
  commit per update. Do not commit data files here by hand.
- **Times are UTC.** All `*_time`, `generated_at`, and `updated_at` values are
  ISO 8601 with an explicit `+00:00` / `Z` offset. The S7 runs on local Munich
  time (`Europe/Berlin`); convert if you display clock times.
- **Reprocessed history:** delay-cause text and other decoded fields are derived
  at export time, so archived months can gain richer labels retroactively
  without the raw codes changing.

## Data schema

### `latest.json`

| Field | Type | Description |
|-------|------|-------------|
| `generated_at` | string (ISO 8601 UTC) | When this file was produced. |
| `station` | string | Always `"Baierbrunn"`. |
| `line` | string | Always `"S7"`. |
| `window_days` | number | Size of the rolling window in days (7). |
| `arrivals` | array\<Arrival\> | Every observed arrival in the window (see [Arrival](#arrival-object)). |
| `aggregates.today` | Aggregate + `by_direction` | Stats for the current `Europe/Berlin` date. |
| `aggregates.last_7_days` | Aggregate + `by_direction` | Stats for the whole window. |
| `expected_slots.today` | object | `{ "muenchen": [ISO...], "wolfratshausen": [ISO...] }` - the scheduled arrival slots expected today per direction. |
| `terminus_health` | array | Diagnostic health of terminus enrichment (see [terminus_health](#terminus_health)). |

### Arrival object

One record per train arrival observed at Baierbrunn.

| Field | Type | Description |
|-------|------|-------------|
| `train_id` | string | Stable per-train identifier from the DB API. Deduplication key together with `scheduled_time`. |
| `line` | string | Service line, `"S7"`. |
| `station` | string | `"Baierbrunn"`. |
| `direction` | string | Raw destination label from DB, e.g. `"München Hbf Gl.27-36"` or `"Wolfratshausen"`. |
| `direction_bucket` | string | Normalized direction: `"muenchen"` (towards Munich / Stammstrecke) or `"wolfratshausen"` (towards Wolfratshausen). |
| `scheduled_time` | string (ISO 8601 UTC) | Planned arrival time at Baierbrunn. |
| `actual_time` | string \| null | Actual (or currently estimated) arrival time. `null` if unknown. |
| `delay_minutes` | number \| null | Delay at Baierbrunn in minutes (`actual - scheduled`). Can be negative (early). `null` if unknown. |
| `cancelled` | boolean | `true` if this arrival is cancelled. |
| `train_number` | string \| null | DB train number (Zugnummer), e.g. `"6755"`. `null` if not resolved. |
| `terminus_status` | string \| null | Fate of the train at its final destination: `"arrived"`, `"short_turn"` (turned back early), `"cancelled"`, or `null` (unknown / not yet determined). |
| `terminus_delay_minutes` | number \| null | Delay at the final destination in minutes. Can be negative (early); display code typically floors to 0. `null` if unknown. |
| `terminus_short_turn_station` | string \| null | If `terminus_status` is `"short_turn"`, the station where the train actually terminated. Otherwise `null`. |
| `disruption` | object \| null | Disruption / delay-cause detail, or `null` when none is reported (see [disruption](#disruption-object)). |

### disruption object

Present on an arrival when the DB API attaches a delay cause or a HIM
(Hafas Information Manager) disruption message.

| Field | Type | Description |
|-------|------|-------------|
| `category` | string \| null | HIM message category (German), e.g. `"Störung"`, `"Bauarbeiten"`. `null` if only a numeric cause code is present. |
| `cause_code` | number \| null | DB Verspätungsursachen (delay-cause) code. |
| `cause_text` | string \| null | German text for `cause_code` if the code's meaning is confirmed against the DB reference; otherwise `null`. The raw `cause_code` is always kept so no information is lost. |
| `window` | object \| null | Validity window of the disruption message: `{ "from": ISO, "to": ISO }`. `null` if not specified. |

### Aggregate object

Used by `aggregates.*`, `daily[]`, and `archive/index.json` month entries.

| Field | Type | Description |
|-------|------|-------------|
| `total` | number | Number of arrivals counted. |
| `on_time` | number | Not cancelled and delay <= 0. |
| `late` | number | Not cancelled and delay > 0. |
| `cancelled` | number | Cancelled arrivals. |
| `avg_delay_min` | number | Mean delay in minutes over non-cancelled arrivals (one decimal). |

`aggregates.today` and `aggregates.last_7_days` (and each monthly archive's
`aggregates`) additionally carry `by_direction.muenchen` and
`by_direction.wolfratshausen`, each an Aggregate. In `latest.json` the
per-direction aggregates also include:

| Field | Type | Description |
|-------|------|-------------|
| `missing` | number | Expected arrival slots for the period minus observed arrivals in that direction - an indicator of data gaps. |

### terminus_health

Internal diagnostic emitted in `latest.json`. One entry per direction bucket.

| Field | Type | Description |
|-------|------|-------------|
| `bucket` | string | `"muenchen"` or `"wolfratshausen"`. |
| `zero_match_streak` | number | Consecutive export cycles in which terminus enrichment matched no arrivals for this bucket. A rising value signals the terminus data may be stale. |
| `updated_at` | string (ISO 8601 UTC) | When this health row was last updated. |

### `archive/YYYY-MM.json`

One calendar month. Same `arrivals[]` shape as `latest.json`.

| Field | Type | Description |
|-------|------|-------------|
| `generated_at` | string (ISO 8601 UTC) | When this file was produced. |
| `station` | string | `"Baierbrunn"`. |
| `line` | string | `"S7"`. |
| `period` | string | The month, `"YYYY-MM"`. |
| `finalized` | boolean | `true` once the month is complete and will not change; `false` while still being filled or refined. |
| `arrivals` | array\<Arrival\> | All arrivals in the month. |
| `aggregates` | Aggregate + `by_direction` | Whole-month totals, plus per-direction Aggregates. |
| `daily` | array | Per-day Aggregates: `{ "date": "YYYY-MM-DD", ...Aggregate }`. |
| `daily_by_direction` | object | `{ "muenchen": [daily...], "wolfratshausen": [daily...] }`. |

### `archive/index.json`

| Field | Type | Description |
|-------|------|-------------|
| `generated_at` | string (ISO 8601 UTC) | When the index was produced. |
| `station` | string | `"Baierbrunn"`. |
| `months` | array | One entry per archived month. |

Each `months[]` entry:

| Field | Type | Description |
|-------|------|-------------|
| `period` | string | `"YYYY-MM"`. |
| `finalized` | boolean | Whether the month's archive file is finalized. |
| `total`, `on_time`, `late`, `cancelled`, `avg_delay_min` | number | Month Aggregate (see [Aggregate](#aggregate-object)). |
| `by_direction` | object | `{ "muenchen": Aggregate, "wolfratshausen": Aggregate }`. |
