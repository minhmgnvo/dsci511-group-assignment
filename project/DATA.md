# Data Documentation

## Research Question

**Which car performance and race strategy factors best predict a driver's final race finishing position?**

---

## Data Sources

### 1. OpenF1 API

- **URL:** https://api.openf1.org/v1
- **Type:** REST API (free, no authentication required)
- **Coverage:** Formula 1 telemetry and session data for the 2023 season
- **Access method:** HTTP GET requests with query parameters (`session_key`, `driver_number`, `year`, etc.)

OpenF1 provides real-time and historical F1 data across multiple endpoints. Each endpoint returns a JSON array of records. The API has uneven coverage — some sessions and endpoints (notably `stints` and `pit`) are missing for certain races.

#### Endpoints Used

| Endpoint | Raw rows (approx.) | Description |
|---|---|---|
| `sessions` | ~29 rows | One row per session — circuit name, country, date, session key |
| `drivers` | ~478 rows | One row per driver per session — name, team, nationality |
| `laps` | ~23,000 rows | One row per lap per driver — lap time, sector times |
| `stints` | ~265 rows | One row per stint per driver — tire compound, lap range |
| `pit` | ~359 rows | One row per pit stop — lap number, pit lane duration |
| `position` | ~millions (aggregated) | Sub-second track position — aggregated to starting position only |
| `session_result` | ~558 rows | One row per driver per session — final position, points, DNF/DNS/DSQ flags |
| `car_data` | ~4.7 million rows | Sub-second telemetry (speed, throttle, brake, RPM, gear, DRS) — limited to top 5 finishers per session |

---

### 2. StatsF1

- **URL:** https://www.statsf1.com
- **Type:** Web scraping (public website, no authentication required)
- **Coverage:** All 22 Grand Prix races of the 2023 season
- **Access method:** HTTP GET requests with the `requests` library; HTML parsed with `pandas.read_html()`

StatsF1 publishes a structured race classification table for every Grand Prix. These tables provide a complete finishing order for all 22 races, which supplements the OpenF1 `session_result` data where it is missing.

Each race's result page follows the URL pattern:
```
https://www.statsf1.com/en/2023/{race-slug}/classement.aspx
```

Race slugs are hardcoded in calendar order (e.g. `bahrein`, `monaco`, `grande-bretagne`).

---

## Data Retrieval

### OpenF1

All endpoints are queried in a loop over `df_sessions["session_key"]`. When `SESSION_KEY` is set to a specific integer, only that one session is fetched (for development and testing). When `SESSION_KEY = None`, all 2023 race sessions are fetched.

API responses that return an error dict instead of a list are silently skipped via an `isinstance(data, list)` guard applied to every endpoint. This handles sessions where an endpoint has no data without crashing the loop.

`car_data` is the highest-volume endpoint (~18,000 rows per driver per session). To keep retrieval time manageable, only the top `SAMPLE_DRIVERS = 5` finishers per session are fetched, reducing API calls by ~75%. Drivers finishing 6th or lower receive `NaN` for all `car_data`-derived columns.

`position` data is aggregated immediately inside the fetch loop (rather than storing millions of raw rows) by taking each driver's first recorded position per session as a proxy for their starting grid position.

### StatsF1

Each race's `/classement.aspx` page is fetched sequentially. The 22 race slugs are iterated in calendar order with `enumerate(..., start=1)` to assign round numbers 1–22. Pages are parsed with `pd.read_html()`. If a page returns no table or raises an exception, that race is skipped with a printed warning.

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

A small number of drivers could not be matched automatically (`Nyck de VRIES`, `Guanyu ZHOU`) due to capitalisation differences. These rows receive `NaN` for `driver_number` and are excluded from the StatsF1 join.

Round numbers 1–22 are assigned to OpenF1 race sessions by sorting all Race sessions chronologically and excluding sprint sessions. The same 1–22 numbering applies to the StatsF1 slugs, giving a shared `round` key used to map each StatsF1 row to an OpenF1 `session_key`.

### Final Merge

`df_session_result` is used as the base table (left table) for the merge chain. This ensures one row per driver per session, limited to the sessions where OpenF1 returned result data. All other sources are joined with left joins on `["session_key", "driver_number"]`, so missing data for a given endpoint results in `NaN` rather than dropped rows. StatsF1 data is joined last as a supplementary source.

---

## Final Dataset: `df_combined_full`

**Shape:** 558 rows × 40 columns  
**Grain:** One row per driver per session  
**Season:** 2023

### Data Dictionary

#### Identifiers

| Column | Type | Source | Description |
|---|---|---|---|
| `session_key` | int | OpenF1 | Unique identifier for the race session |
| `driver_number` | int | OpenF1 | Driver's car number (1–99) |
| `meeting_key` | int | OpenF1 | Identifier for the race weekend (links sessions within the same event) |
| `round` | float | StatsF1 | Race round number within the 2023 season (1–22); `NaN` if not matched |

#### Session / Race Context

| Column | Type | Source | Description |
|---|---|---|---|
| `session_name` | str | OpenF1 sessions | Session type (e.g. "Race", "Sprint") |
| `circuit_short_name` | str | OpenF1 sessions | Short circuit name (e.g. "Monaco", "Silverstone") |
| `country_name` | str | OpenF1 sessions | Host country of the Grand Prix |
| `date_start` | datetime | OpenF1 sessions | UTC start time of the session |

#### Driver Info

| Column | Type | Source | Description |
|---|---|---|---|
| `full_name` | str | OpenF1 drivers | Driver's full name in "FirstName LASTNAME" format |
| `team_name` | str | OpenF1 drivers | Constructor/team name |
| `country_code` | str | OpenF1 drivers | Driver's nationality (ISO 3-letter code, e.g. "NED", "GBR") |

#### Race Result (OpenF1) — primary target variable

| Column | Type | Source | Description |
|---|---|---|---|
| `position` | float | OpenF1 session_result | Official classified finishing position (1–20) |
| `points` | float | OpenF1 session_result | Championship points awarded for this race |
| `number_of_laps` | int | OpenF1 session_result | Total laps completed as recorded in the official result |
| `dnf` | bool | OpenF1 session_result | True if the driver did not finish |
| `dns` | bool | OpenF1 session_result | True if the driver did not start |
| `dsq` | bool | OpenF1 session_result | True if the driver was disqualified |
| `duration` | float | OpenF1 session_result | Race duration in seconds (winner); gap for others |
| `gap_to_leader` | str/float | OpenF1 session_result | Time or laps gap to the race winner |

#### Race Result (StatsF1) — supplementary, complete for all 22 races

| Column | Type | Source | Description |
|---|---|---|---|
| `finish_pos_statsf1` | int | StatsF1 | Classified finishing position (1–20) |
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
| `stint_count` | int | Number of tire stints completed |
| `compounds_used` | list[str] | List of unique tire compounds used (e.g. `["SOFT", "HARD"]`) |

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

- **Dataset includes sprint sessions:** OpenF1's `session_type="Race"` query returns both Grand Prix races and Sprint races (6 sprint events in 2023), so the dataset contains both race types mixed together. Sprint races are shorter (~100 km) with different strategy characteristics than full Grand Prix races.
- **`car_data` limited to top 5 finishers:** Drivers finishing 6th or lower have `NaN` for all telemetry columns. This introduces a selection bias if telemetry features are used in a model predicting finishing position.
- **Unmatched StatsF1 drivers:** `Nyck de VRIES` and `Guanyu ZHOU` could not be automatically matched due to capitalisation differences; their rows have `NaN` for all StatsF1 columns.
- **OpenF1 `stints` and `pit` coverage is incomplete:** Some sessions returned no data from these endpoints; affected rows have `NaN` for stint and pit stop columns.
- **`start_position` is approximate:** The `position` endpoint captures track order at the first telemetry sample, not the official FIA grid position. Formation lap incidents or late grid changes may not be reflected accurately.
