# whychusData

Daily data and forecasts for the Whychus Creek RM 8.5 temperature forecast.

This repository is the data store for [whychusModel](https://github.com/FlowWest/udwc_temperature) (the R package, scripts and models). A GitHub Action in that repository runs every evening, pulls the latest observations and weather forecasts, makes a 1–7 day forecast of the **7-day average daily maximum water temperature (7DADM)** at river mile 8.5, and commits the results here.

**The files are written by the Action. Don't edit them by hand** (see [Working with the data](#working-with-the-data)).

```
whychusData/
├── input/
│   ├── hydro_vu.csv        RM 8.5 logger readings (HydroVu)
│   ├── nws.csv             NWS weather forecasts
│   ├── camp_polk.csv       Camp Polk flow and water temperature (OWRD)
│   └── model_inputs.csv    the exact feature row given to the model, per lead
└── output/
    └── model_outputs.csv   the forecast, per lead
```

## Conventions

- **Dates are Pacific** (`America/Los_Angeles`) calendar dates. The one exception is `timestamp` in `hydro_vu.csv`, which is UTC.
- **Units:** water and air temperature in °C, flow in cfs, cloud cover in percent.
- **7DADM** is the mean of the last 7 daily maximum temperatures. The HCP threshold the forecast is checked against is 20 °C.
- **`run_date`** is the day a forecast was produced. **`lead_days`** is how many days ahead of the run the forecast applies to (`forecast_date − run_date`).
- Blank values are written as `NA`.
- Files are cumulative and sorted by date. Re-running a day **replaces** that day's rows rather than adding duplicates.

## How it is updated

The workflow lives in whychusModel (`.github/workflows/daily-forecast.yml`) and runs at about **8 pm Pacific** each day. Two cron times are set, one hour apart, so one lands at 8 pm on either side of daylight saving. It:

1. checks out this repository,
2. runs `scripts/fetch_hydrovu.R`, `scripts/fetch_nws.R` and `scripts/fetch_camp_polk.R`, which update the three source files in `input/`,
3. runs `scripts/run-daily-forecast.R`, which reads those files, writes `input/model_inputs.csv` and `output/model_outputs.csv`,
4. commits and pushes everything here.

Commit messages look like `chore: daily data and forecast 2026-10-26 [scheduled]`. The tag is `[scheduled]` for the evening run and `[manual]` for a run started by hand to test; only a `[scheduled]` commit counts as that day's forecast.

If a source is down, its step fails without stopping the rest. The forecast then skips any lead whose inputs are missing and says why in `note` (see `model_outputs.csv`).

---

## `input/hydro_vu.csv`

Readings from the RM 8.5 data logger ("Whychus Creek RM 8.5"), pulled from the HydroVu API. Roughly hourly, one row per reading per parameter, in time order. Revised readings are updated in place.

| Column | Meaning |
|---|---|
| `location_id` | HydroVu location (`4884916384366592` = Whychus Creek RM 8.5) |
| `parameter_id` | `1` water temperature (°C), `2` pressure (psi), `10` specific conductivity (µS/cm) |
| `unit_id` | HydroVu unit code for that parameter (`1`, `17`, `65` respectively) |
| `timestamp` | Reading time in **UTC**, ISO 8601 (`2026-10-15T21:00:00Z`) |
| `value` | The reading |

The logger keeps local clock time, which HydroVu returns mislabelled as UTC. The fetch script corrects this, so `timestamp` is true UTC.

The forecast uses parameter `1` only: it takes the maximum per Pacific calendar day, then the 7DADM from those maxima. Coverage starts 2026-10-15.

## `input/nws.csv`

The National Weather Service gridpoint forecast (`PDT/35,57`, near Sisters), one pull per day. One row per run × variable × forecast period, so one run holds several rows for the same day (sky cover is hourly). The hour itself isn't kept; the model takes the maximum, minimum or mean per day.

| Column | Meaning |
|---|---|
| `run_date` | Day the forecast was pulled |
| `forecast_date` | Day the value applies to |
| `lead_days` | `forecast_date − run_date`; runs from −1 to 8 |
| `variable` | `maxTemperature` (°C), `minTemperature` (°C), `skyCover` (%) |
| `value` | The forecast value |
| `source` | `api` for a normal pull; `fallback_run_<date>` if the API failed and an earlier run was re-logged under today's date; blank on the earliest rows, which predate this column |

History runs from 2026-05-04.

## `input/camp_polk.csv`

Camp Polk (OWRD station 14076100), the upstream gauge, summarised to one row per day.

| Column | Meaning |
|---|---|
| `date` | Day |
| `cp_flow_mean` | Mean flow (cfs) |
| `cp_tmax` | Daily maximum water temperature (°C) |
| `cp_7dadm` | 7DADM of water temperature (°C) |
| `flow_source`, `temp_source` | `camp_polk` normally. If Camp Polk was unavailable, `was_regression` (flow estimated from Whychus Creek at Sisters, 14076050) or `gage14075000_regression` (temperature estimated from Whychus Creek near Sisters, 14075000) |
| `pulled_on` | The run that last wrote this row |

Each run re-pulls the last 14 days and replaces those dates, so recent values are corrected as OWRD revises them. Only days with a complete 7DADM are kept. The record starts 2026-07-06, with a gap from 2026-08-29 to 2026-10-17 (and one missing day, 2026-07-15), from before this pipeline was collecting it.

## `input/model_inputs.csv`

The complete input to the model for each forecast: one row per lead per run, so it can be audited or replayed. Predictions can be reproduced from a row alone.

| Columns | Meaning |
|---|---|
| `run_date`, `forecast_date`, `lead_days` | Which forecast this row is |
| `cp_flow_mean`, `cp_7dadm`, `cp_tmax` | Camp Polk on the run date |
| `cp_flow_trend`, `cp_7dadm_trend` | Camp Polk's rate of change over the previous 5 days (cfs/day, °C/day) |
| `air_max_d1`…`d7`, `air_min_d1`…`d7`, `cloud_d1`…`d7` | NWS forecast max temperature, min temperature and cloud cover for each of the next 1–7 days |
| `julian_d1`…`d7` | Day of year of each of those days |
| `dadm_m0`, `tmax_m0` | RM 8.5's 7DADM and daily maximum on the run date (`m0` = the run date) |
| `dadm_trend`, `tmax_trend` | RM 8.5's rate of change over the previous 5 days (°C/day) |
| `traj_air_max_max`, `traj_air_max_mean`, `traj_air_min_mean`, `traj_warming` | Summaries of the forecast days: hottest day, mean maximum, mean minimum, and last-day minus first-day maximum |
| `cp_flow_source`, `cp_temp_source` | As in `camp_polk.csv` |
| `air_max_source`, `air_min_source`, `cloud_source` | `api`, `fallback_run_<date>` if taken from an earlier NWS run, or `missing`. Several are joined with `;` |
| `rm85_source` | `hydro_vu`, or `missing` |
| `rm85_readings_today` | HydroVu readings behind the run date's maximum. A low number means `dadm_m0` / `tmax_m0` rest on a partial day |

A lead of `k` days uses forecast days 1 to `k`, so the `_d` columns are blank beyond that: lead 3 has no `d4`–`d7`. Lead 1 uses 17 inputs, lead 7 uses 41.

Starts 2026-10-26.

## `output/model_outputs.csv`

The forecast: one row per lead per run.

| Column | Meaning |
|---|---|
| `run_date`, `forecast_date`, `lead_days` | Which forecast this row is |
| `dadm_m0` | RM 8.5's observed 7DADM on the run date |
| `pred_change_C` | The model's prediction of the **change** in 7DADM from the run date to `forecast_date` (°C) |
| `pred_7dadm_C` | The forecast 7DADM: `dadm_m0 + pred_change_C` |
| `note` | Blank normally. If a lead was skipped, `skipped, missing: ` followed by the inputs that were absent, and the two prediction columns are blank |
| `cp_flow_source`, `cp_temp_source`, `air_source` | Where those inputs came from (as above; `air_source` combines the three air sources) |

`pred_7dadm_C` is computed from unrounded values, so it can differ by 0.01 from the sum of the two rounded columns shown.

The model predicts the change from today rather than the temperature itself because a random forest can't predict outside the range of what it was trained on, while day-to-day changes stay in range even in an unusually warm year. Starts 2026-10-26.

---

## Working with the data

**Reading it** needs nothing special. In R:

```r
nws <- readr::read_csv("input/nws.csv")
fc  <- readr::read_csv("output/model_outputs.csv")
```

**Running the scripts locally.** Clone this repository next to whychusModel (or set `WHYCHUS_DATA_DIR` to this repository's `input/` folder) and run the scripts from whychusModel. They read and write the files here.

**Don't edit the files by hand.** The Action rewrites them and pushes with `git pull --rebase`, so a manual change to the same file can conflict and make that night's push fail. If you need to change one, pull first and push before the 8 pm run.

**Column layouts are fixed in the scripts.** If `model_inputs.csv` or `model_outputs.csv` needs a new column, change the layout in `scripts/run-daily-forecast.R` and update the file to match.
