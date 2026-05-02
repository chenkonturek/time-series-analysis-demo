# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

**Run diagnostics:**
```bash
python3 scripts/diagnose.py data.csv --output-dir results/
```

**Generate plots** (run after diagnose.py — reads diagnostics_state.json for synchronization):
```bash
python3 scripts/visualize.py data.csv --output-dir results/
```

Both scripts accept:
- `--date-col NAME` — date column name (auto-detected if omitted)
- `--value-col NAME` — value column name (auto-detected if omitted)
- `--seasonal-period N` — seasonal period (auto-detected if omitted)
- `--output-dir PATH` — output directory (default: `diagnostics/`)

**Run evals:**
```bash
cd /Users/chen/.claude/plugins/cache/anthropic-agent-skills/example-skills/12ab35c2eb56/skills/skill-creator
python -m scripts.run_eval \
  --skill-path /Users/chen/PycharmProjects/claude-demo/time-series-analysis-demo/.claude/skills/analyzing-time-series
```

**Install dependencies:**
```bash
pip install pandas numpy matplotlib statsmodels scipy
```

## Architecture

This is a Claude Code skill repository. The skill lives at `.claude/skills/analyzing-time-series/` and is invoked via the `analyzing-time-series` skill name.

### Scripts

`scripts/ts_utils.py` is the shared analysis library. All statistical logic lives here:
- `load_data()` — reads CSV, auto-detects date/value columns
- `detect_frequency()` — infers data frequency; uses 10th-percentile gap (not median) to correctly classify sparse hourly data
- `resample_regular()` — resamples sparse hourly/daily series to a regular grid via linear interpolation, so ACF lags and STL periods correspond to real time intervals
- `test_stationarity()` — ADF + KPSS tests, determines differencing order d
- `detect_seasonal_period()` — resamples to regular grid, then searches frequency-appropriate candidate periods via ACF
- `analyze_seasonality()` — STL decomposition on regular grid, computes seasonal strength
- `analyze_trend()` — STL trend component on regular grid, direction and strength
- `analyze_autocorrelation()` — ACF/PACF at all lags
- `test_forecastability()` — Ljung-Box test, white noise classification
- `check_transform_recommendation()` — Box-Cox lambda, variance-stabilization advice

`scripts/diagnose.py` orchestrates all 8 analysis steps and writes:
- `diagnostics.json` — all test results and statistics
- `summary.txt` — human-readable findings
- `diagnostics_state.json` — internal state consumed by visualize.py

`scripts/visualize.py` reads `diagnostics_state.json` to synchronize ACF/PACF differencing with what diagnose.py computed, then writes `plots/` with frequency-appropriate box plots (box_by_hour, box_by_dayofweek, box_by_month, box_by_quarter), ACF/PACF, STL decomposition, and lag scatter.

### Key design constraint

`visualize.py` must be run **after** `diagnose.py` — it reads `diagnostics_state.json` to know which differencing order was chosen for the ACF/PACF plots. Running visualize.py first produces unsynchronized plots.

## Skill version

Current: **v1.1.0** (`.claude/skills/analyzing-time-series/SKILL.md`)
- v1.1.0 — fixed sparse-hourly frequency detection; added `resample_regular()`
- v1.0.0 — initial release

## Sample data and evals

- `data/retail_sales.csv` — 120 monthly observations (Jan 2015–Dec 2024), trending, seasonal
- `data/city_data.csv` — 3,871 hourly air quality readings with columns `Timestamp` and `PM2.5`

Eval test cases at `.claude/skills/analyzing-time-series/evals/evals.json` (3 cases).
Eval workspace and benchmark at `.claude/skills/analyzing-time-series-workspace/` (iter-1: 81%, iter-2: 100%).
Full report at `.claude/skills/analyzing-time-series-workspace/eval_report.md`.
