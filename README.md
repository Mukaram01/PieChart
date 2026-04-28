# PieChart WinCC Global Script — Operator & Maintainer Guide

## Project Overview
This repository contains a **WinCC Global Script** that aggregates archived machine status signals into percentage outputs suitable for runtime dashboards and pie-chart widgets.

### What the script does
- Aggregates machine status activity over fixed historical windows.
- Uses a binary archived input model where each status tag is a `*_Count` tag sampled at high frequency.
- Produces normalized status percentages for both standard and pie-chart-specific output tags.

### Input model
- Source tags are binary **0/1** archive tags sampled every **500 ms**.
- The script reads the following base statuses per machine:
  - `Aborted`
  - `Halted`
  - `Running`
  - `Waiting`
  - `Interrupted`

### Output model
For each machine and period, the script writes:
- `Aborted`, `Halted`, and `Running` percentages.
- Standard percentage tags: `<Machine>_<Period>_Machine_<Status>_Percentage`
- Pie rendering tags: `<Machine>_<Period>_Machine_<Status>_Percentage_PieChart`

---

## Status Mapping
The script combines raw status fractions into three final outputs using fixed logic:

- **Aborted** = `Aborted + Waiting`
- **Halted** = `Halted + Interrupted`
- **Running** = `Running`

This mapping is intentionally stable so downstream WinCC screens and historians can rely on consistent semantics.

---

## Windows and Scheduling
The script computes aggregates for three period keys:

- `1D`
- `7D`
- `30D`

### Operational behavior for 1D / 7D / 30D
- `1D` reads machine status values directly from the WinCC archive query path.
- `7D` reads from `dbo.PieChartDailyRollup` and targets 7 daily rows.
- `30D` reads from `dbo.PieChartDailyRollup` and targets 30 daily rows.
- `7D` can still publish real percentages when at least 5 rows are available (partial window mode).
- `30D` can still publish real percentages when at least 21 rows are available (partial window mode).
- If the row count is below those minimums, the script writes zero/no-data outputs and logs `insufficient_rollup_rows`.
- Missing rollup rows are backfilled gradually by capped maintenance each invocation so runtime work stays bounded.
- Period selection is round-robin among due jobs, so `30D` is not permanently starved behind `1D`/`7D`.

### Rollup diagnostics query
Use this query to verify rollup completeness per machine:

```sql
USE [CC_SLSGLO_xxxxxxxx];
GO

DECLARE @Today date = CONVERT(date, GETDATE());

SELECT
    machine_id,
    SUM(CASE
            WHEN summary_date >= DATEADD(day, -7, @Today)
             AND summary_date < @Today
            THEN 1 ELSE 0
        END) AS rows_last_7_days,
    SUM(CASE
            WHEN summary_date >= DATEADD(day, -30, @Today)
             AND summary_date < @Today
            THEN 1 ELSE 0
        END) AS rows_last_30_days,
    MIN(summary_date) AS first_rollup_day,
    MAX(summary_date) AS last_rollup_day
FROM dbo.PieChartDailyRollup
GROUP BY machine_id
ORDER BY machine_id;
```

### Throttle constants
Execution throttling is configured by:

- `THROTTLE_1D_MINUTES` (default: 30 minutes)
- `THROTTLE_7D_MINUTES` (default: 360 minutes / 6 hours)
- `THROTTLE_30D_MINUTES` (default: 1440 minutes / 24 hours)

### Startup validation lookback and retry intent
At startup, validation probes check whether recent archived data exists before normal aggregation writes continue.

Configured lookbacks:
- `STARTUP_VALIDATE_LOOKBACK_1D_MINUTES = 1440`
- `STARTUP_VALIDATE_LOOKBACK_7D_MINUTES = 10080`
- `STARTUP_VALIDATE_LOOKBACK_30D_MINUTES = 43200`
- Probe cap: `STARTUP_VALIDATE_LOOKBACK_PROBE_CAP_MINUTES = 10080`

Validation/retry intent:
- Tracks validation state (`HARD_FAIL`, `NO_RECENT`, `OK`).
- Retries hard failures quickly and normal revalidation less frequently.
- Supports a startup behavior flag to optionally block writes when probe checks fail.
- Includes a compatibility gate (`STARTUP_NO_RECENT_SHORT_CIRCUIT`) for `NO_RECENT` startup state handling:
  - `False` (default): keep legacy behavior and continue normal SQL window aggregation (`AggregatePeriod` path).
  - `True`: early-exit each due window and write deterministic fallback outputs directly (skips per-window DB aggregation reads).

---

## Configuration Reference
Key constants maintainers should know:

### SQL connectivity and provider fallback
- `DEFAULT_PROVIDER = "MSOLEDBSQL"`
- `FALLBACK_PROVIDER = "SQLNCLI11"`
- `SQL_INSTANCE = ".\\WinCC"`
- `INITIAL_CATALOG = "master"`

### Timeout and retry controls
- `SQL_TIMEOUT_SECONDS`
- `SQL_TIMEOUT_SECONDS_PROBE`
- `MAX_RETRY_ATTEMPTS`
- `RETRY_BACKOFF_MS`
- `RETRY_BACKOFF_TIMEOUT_MS`
- `WINDOW_EXEC_WARN_THRESHOLD_MS`
- `STARTUP_NO_RECENT_SHORT_CIRCUIT` (startup `NO_RECENT` early-exit gate; default `False` for compatibility)

### Zero-total rendering behavior
- `RENDER_ZERO_TOTAL_AS_FALLBACK_PIE`
  - `True`: deterministic fallback pie split (34/33/33)
  - `False`: legacy all-zero pie output for idle windows

---

## Architecture / Flow
High-level execution lifecycle:

1. **`MainStatusAggregation`**
   - Entrypoint orchestrates machine/period processing.
2. **`ValidateMachineCountTags`**
   - Checks input tag readiness and recent archive availability.
3. **`GetStatusFractionsBatch`** (with fallback behavior)
   - Retrieves batched status fractions using preferred SQL aggregate expressions and fallbacks.
4. **`AggregatePeriod` + `WriteTag`**
   - Normalizes totals and writes standard + pie-chart output tags.

Operationally, this flow is designed to fail safely, log meaningful diagnostics, and continue where possible.

---

## Tag Naming Conventions

### Input tag naming
Core template and suffix:

- Core: `{MACHINE_ID}_ProgramStatus_{STATUS_NAME}`
- Suffix: `_Count`
- Full source pattern (with machine logging prefix):
  - `<MachinePrefix>\\{MACHINE_ID}_ProgramStatus_{STATUS_NAME}_Count`

### Output tag naming
- Standard output:
  - `<Machine>_<Period>_Machine_<Status>_Percentage`
- Pie-chart output:
  - `<Machine>_<Period>_Machine_<Status>_Percentage_PieChart`

### Concrete example
Using machine ID `MNA_0269` with configured machine prefix `DMG_Mori1_MNA_0269`:

- Example input tag:
  - `DMG_Mori1_MNA_0269\\MNA_0269_ProgramStatus_Running_Count`
- Example output tags:
  - `MNA_0269_1D_Machine_Running_Percentage`
  - `MNA_0269_1D_Machine_Running_Percentage_PieChart`

---

## Logging and Troubleshooting

### Log levels
The script defines:
- `LOG_LEVEL_ERROR = 1`
- `LOG_LEVEL_WARN = 2`
- `LOG_LEVEL_INFO = 3`
- `LOG_LEVEL_DEBUG = 4`

Active level is controlled by:
- `CURRENT_LOG_LEVEL`

### Typical failure points
- Archive probe failure during startup validation.
- SQL provider/connectivity issues leading to fallback provider usage.
- SQL timeout-like behavior in batch reads/probes.
- Missing or misnamed machine/status tags.

### What to check in WinCC runtime traces
- Entries prefixed with `PieChart/<source>`.
- Provider switch warnings (`MSOLEDBSQL` → `SQLNCLI11`).
- Validation state transitions and retry timing.
- Repeated write or aggregation warnings for a specific machine/period.
- Startup `NO_RECENT` branch markers:
  - `startup_validation_warning ... no_recent_short_circuit_applied=0` means legacy aggregation path remains active.
  - `startup_no_recent_rows_short_circuit ... no_recent_short_circuit_applied=1` means DB aggregation was bypassed and fallback outputs were written.
- Runtime/window coherence markers during short-circuit mode:
  - `RunNoRecentWindowShortCircuitWithGuard` logs `window_guard_short_circuit` and elapsed time per due period.
  - `runtime_guard` now includes `no_recent_short_circuit_eligible` and `no_recent_short_circuit_applied`.

---

## TODO (Maintainer Backlog)
- [ ] Document deployment/import steps into the WinCC project.
- [ ] Add machine onboarding checklist (required `G_MACHINES` entry + archive tag verification).
- [ ] Provide expected tag examples per machine for all five base statuses.
- [ ] Add a change log section for script revisions.
- [ ] Add known limitations and an operational runbook.
