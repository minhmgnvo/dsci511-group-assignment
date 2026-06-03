# Data Documentation

## Research Question

**Which car performance and race strategy factors best predict a driver's final race finishing position?**

---

## Data Sources

### 1. OpenF1 API

- **URL:** https://api.openf1.org/v1
- **Type:** REST API (free, no authentication required)
- **Coverage:** Formula 1 telemetry and session data for the 2023 and 2024 seasons
- **Access method:** HTTP GET requests with query parameters (`session_key`, `driver_number`, `year`, etc.)

OpenF1 provides real-time and historical F1 data across multiple endpoints. Each endpoint returns a JSON array of records. The API has uneven coverage — some sessions and endpoints (notably `session_result` and `stints`) are missing for certain races.

#### Endpoints Used

| Endpoint | Raw rows (approx.) | Description |
|---|---|---|
| `sessions` | ~47 rows | One row per race session — circuit name, country, date, session key |
| `drivers` | ~478 rows | One row per driver per session — name, team, nationality |
| `laps` | ~23,000 rows | One row per lap per driver — lap time, sector times, compound |
| `stints` | ~265 rows | One row per stint per driver — tire compound, lap range |
| `pit` | ~359 rows | One row per pit stop — lap number, pit lane duration |
| `position` | ~millions (aggregated) | Sub-second track position — aggregated to starting position only |
| `session_result` | ~558 rows | One row per driver per session — final position, points, DNF/DNS/DSQ flags |
| `car_data` | ~4.7 million rows | Sub-second telemetry (speed, throttle, brake, RPM, gear, DRS) — limited to top 5 finishers per session |

---

### 2. StatsF1

- **URL:** https://www.statsf1.com
- **Type:** Web scraping (public website, no authentication required)
- **Coverage:** All Grand Prix races for 2023 and 2024 seasons (22 races in 2023, 24 races in 2024)
- **Access method:** HTTP GET requests with `requests` library; HTML parsed with `BeautifulSoup` and `pandas.read_html()`

StatsF1 is used as the **primary base** of the combined dataset because OpenF1's `session_result` endpoint has incomplete coverage (~9/22 races in 2023). StatsF1 provides a complete race classification table for every Grand Prix, ensuring every race and driver appears in the final dataset.

Each race's result page follows the URL pattern:
```
https://www.statsf1.com/en/{year}/{race-slug}/classement.aspx
```

Race slugs (e.g. `bahrein`, `monaco`, `grande-bretagne`) are discovered dynamically by scraping the season index page and filtering out non-race info pages (`bilan`, `stats-victoire`, `pilotes`, `modeles`).

---

## Data Retrieval

### OpenF1

All endpoints are queried in a loop over `df_sessions["session_key"]`. When `SESSION_KEY` is set to a specific value, only that one session is fetched (for development/testing). When `SESSION_KEY = None`, all race sessions for all years in `YEARS` are fetched.

API responses that return an error dict instead of a list are silently skipped via an `isinstance(data, list)` guard on every endpoint. This handles sessions where an endpoint has no data without crashing the loop.

`car_data` is the highest-volume endpoint (~18,000 rows per driver per session). To keep retrieval time manageable, only the top `SAMPLE_DRIVERS = 5` finishers per session are fetched, reducing API calls by ~75%. Drivers finishing 6th or lower receive `NaN` for all `car_data`-derived columns.

`position` data is aggregated immediately inside the fetch loop (rather than storing millions of raw rows) by taking each driver's first recorded position per session as their approximate starting grid position.

### StatsF1

Race slugs for each year are discovered by fetching the season index page and extracting all links matching the pattern `/en/{year}/[slug].aspx`, excluding known non-race section pages. This approach works for any future season without hardcoding slug lists.

Each `/classement.aspx` page is fetched and parsed with `pd.read_html()`. If a page returns no table or raises an exception, that race is skipped with a printed warning.

---

## Data Processing

### Aggregation

High-frequency raw data is reduced to one summary row per driver per session before merging:

| Source | Aggregation |
|---|---|
| `laps` | max lap number (total laps), mean/min/std of lap duration |
| `pit` | count/sum/mean of pit duration |
| `stints` | max stint number, list of unique compounds used |
| `car_data` | mean/max speed, mean throttle/brake/RPM, DRS usage rate |
| `position` | first recorded position per driver (starting position) |

### Name and Key Matching (StatsF1 → OpenF1)

Driver names in StatsF1 follow the `"FirstName LASTNAME"` format (e.g. `"Max VERSTAPPEN"`), which matches OpenF1's `full_name` field exactly. A lookup dictionary built from `df_drivers` maps each StatsF1 name to an OpenF1 `driver_number`.

A small number of drivers could not be matched automatically (e.g. `Nyck de VRIES`, `Guanyu ZHOU`, `Franco COLAPINTO`, `Jack DOOHAN`) due to capitalisation differences or drivers only present in one source. These rows receive `NaN` for `driver_number` and are excluded from OpenF1 feature joins.

Race round numbers are assigned per year by sorting OpenF1 race sessions chronologically (excluding sprint sessions) and counting from 1. StatsF1 slugs appear in calendar order on the season index page and are enumerated from 1 in the same way, giving a consistent `(year, round)` key used to map each StatsF1 row to an OpenF1 `session_key`.

### Final Merge

`df_statsf1` is used as the base table (left table) for all merges. All other sources are joined with left joins on `["session_key", "driver_number"]`. This guarantees one row per driver per race for every race in both seasons, with `NaN` where OpenF1 data is unavailable rather than dropping rows.

Driver metadata (`full_name`, `team_name`, `country_code`) is joined on `driver_number` only (not `session_key`) to work around sparse OpenF1 driver endpoint coverage for 2024, using the most recent known entry per driver.

---

## Final Dataset: `df_combined_full`

**Shape:** ~880 rows × ~37 columns  
**Grain:** One row per driver per race  
**Seasons:** 2023 (22 races), 2024 (24 races)

### Data Dictionary

#### Identifiers

| Column | Type | Source | Description |
|---|---|---|---|
| `session_key` | int | OpenF1 | Unique identifier for the race session |
| `driver_number` | int | OpenF1 / StatsF1 | Driver's car number (1–99) |
| `round` | int | StatsF1 | Race round number within the season (1-indexed, per year) |
| `year` | int | StatsF1 | Season year (2023 or 2024) |

#### Session / Race Context

| Column | Type | Source | Description |
|---|---|---|---|
| `session_name` | str | OpenF1 sessions | Session type (always "Race" in this dataset) |
| `circuit_short_name` | str | OpenF1 sessions | Short circuit name (e.g. "Monaco", "Silverstone") |
| `country_name` | str | OpenF1 sessions | Host country of the Grand Prix |
| `date_start` | datetime | OpenF1 sessions | UTC start time of the session |

#### Driver Info

| Column | Type | Source | Description |
|---|---|---|---|
| `full_name` | str | OpenF1 drivers | Driver's full name in "FirstName LASTNAME" format |
| `team_name` | str | OpenF1 drivers | Constructor/team name |
| `country_code` | str | OpenF1 drivers | Driver's nationality (ISO 3-letter code, e.g. "NED", "GBR") |

#### Race Result (OpenF1) — partial coverage

| Column | Type | Source | Description |
|---|---|---|---|
| `position` | float | OpenF1 session_result | Official classified finishing position (1–20); `NaN` where OpenF1 has no coverage |
| `points` | float | OpenF1 session_result | Championship points awarded for this race |
| `dnf` | bool | OpenF1 session_result | True if the driver did not finish |
| `dns` | bool | OpenF1 session_result | True if the driver did not start |
| `dsq` | bool | OpenF1 session_result | True if the driver was disqualified |
| `duration` | float | OpenF1 session_result | Total race duration in seconds (winner only; others show gap) |
| `gap_to_leader` | str/float | OpenF1 session_result | Time or laps gap to the race winner |

#### Race Result (StatsF1) — complete coverage, preferred target variable

| Column | Type | Source | Description |
|---|---|---|---|
| `finish_pos_statsf1` | int | StatsF1 | Classified finishing position (1–20); use as the primary target variable |
| `laps_statsf1` | float | StatsF1 | Number of laps completed |
| `time_statsf1` | str | StatsF1 | Race time (winner) or gap to leader as a formatted string |
| `points_statsf1` | float | StatsF1 | Championship points awarded |
| `team_statsf1` | str | StatsF1 | Constructor name as listed by StatsF1 |

#### Lap Timing (aggregated from OpenF1 laps)

| Column | Type | Description |
|---|---|---|
| `total_laps` | int | Maximum lap number completed by the driver |
| `avg_lap_duration` | float | Mean lap time in seconds across all laps |
| `fastest_lap` | float | Minimum (best) lap time in seconds |
| `lap_time_std` | float | Standard deviation of lap times — a measure of consistency |

#### Pit Stops (aggregated from OpenF1 pit)

| Column | Type | Description |
|---|---|---|
| `pit_stop_count` | int | Total number of pit stops made |
| `total_pit_time` | float | Sum of all pit lane durations in seconds |
| `avg_pit_duration` | float | Mean pit stop duration in seconds |

#### Tire Strategy (aggregated from OpenF1 stints)

| Column | Type | Description |
|---|---|---|
| `stint_count` | int | Number of tire stints (pit stops + 1) |
| `compounds_used` | list[str] | Ordered list of unique tire compounds used (e.g. `["SOFT", "HARD"]`) |

#### Starting Position (aggregated from OpenF1 position)

| Column | Type | Description |
|---|---|---|
| `start_position` | float | Driver's first recorded track position at session start — proxy for grid position |

#### Telemetry (aggregated from OpenF1 car_data — top 5 finishers only)

| Column | Type | Description |
|---|---|---|
| `avg_speed` | float | Mean speed in km/h across all telemetry samples |
| `max_speed` | float | Maximum recorded speed in km/h |
| `avg_throttle` | float | Mean throttle application (0–100 scale) |
| `avg_brake` | float | Mean brake application (0–100 scale) |
| `avg_rpm` | float | Mean engine RPM |
| `drs_usage` | float | Proportion of telemetry samples where DRS was active (value ≥ 10); range 0–1 |

---

## Known Limitations

- **OpenF1 `session_result` coverage:** Only available for a subset of sessions (~9/22 in 2023, 0/24 in 2024 at time of collection). Use `finish_pos_statsf1` as the target variable, not `position`.
- **OpenF1 `stints` coverage:** Incomplete for 2024; all 2024 rows will have `NaN` for `stint_count` and `compounds_used`.
- **`car_data` limited to top 5:** Drivers finishing 6th or lower have `NaN` for all telemetry columns. This creates a selection bias if telemetry features are used in a model.
- **Unmatched StatsF1 drivers:** A small number of drivers (e.g. Guanyu ZHOU, Franco COLAPINTO) could not be automatically matched to an OpenF1 `driver_number` due to name format differences; these rows have `NaN` for all OpenF1-derived columns.
- **`team_name` reflects most recent known session:** Due to sparse 2024 driver endpoint coverage, team assignments are taken from the driver's most recently observed session in OpenF1, which may not reflect mid-season team changes.
