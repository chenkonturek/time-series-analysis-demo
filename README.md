# Time Series Analysis — Claude Code Skill

A Claude Code skill that runs comprehensive diagnostic analysis on time series data before forecasting. Give it a CSV, get back stationarity tests, seasonality detection, trend analysis, forecastability verdict, and diagnostic plots.

## Quick start

```bash
pip install pandas numpy matplotlib statsmodels scipy

python3 .claude/skills/analyzing-time-series/scripts/diagnose.py data/retail_sales.csv --output-dir results/
python3 .claude/skills/analyzing-time-series/scripts/visualize.py data/retail_sales.csv --output-dir results/
```

Or invoke via Claude Code by describing your data — the `analyzing-time-series` skill triggers automatically.

## What it produces

```
results/
├── diagnostics.json       # All test results (stationarity, seasonality, trend, forecastability)
├── summary.txt            # Human-readable findings and recommendations
├── diagnostics_state.json # Internal state for plot synchronization
└── plots/
    ├── timeseries.png
    ├── decomposition.png  # STL seasonal decomposition
    ├── acf_pacf.png        # Autocorrelation / partial autocorrelation
    ├── box_by_hour.png     # Hourly pattern (hourly data)
    ├── box_by_dayofweek.png
    ├── box_by_month.png    # Monthly pattern (daily/monthly data)
    ├── rolling_stats.png
    └── lag_scatter.png
```

## Analyses performed

| Analysis | Method | Output key |
|----------|--------|------------|
| Stationarity | ADF + KPSS | `stationarity.differencing_needed` |
| Seasonality | ACF peak detection + STL | `seasonality.is_seasonal`, `seasonality.period` |
| Trend | STL decomposition | `trend.direction`, `trend.strength` |
| Forecastability | Ljung-Box test | `forecastability.forecastable` |
| Transform | Box-Cox lambda | `transform.recommendation` |

## CLI options

Both `diagnose.py` and `visualize.py` accept:

| Flag | Default | Description |
|------|---------|-------------|
| `--date-col NAME` | auto-detect | Date/timestamp column |
| `--value-col NAME` | auto-detect | Numeric value column |
| `--seasonal-period N` | auto-detect | Override seasonal period |
| `--output-dir PATH` | `diagnostics/` | Output directory |

## Sample data

- `data/retail_sales.csv` — 120 monthly observations (Jan 2015–Dec 2024), upward trend, annual seasonality
- `data/city_data.csv` — 3,871 sparse hourly air quality readings, non-standard columns (`Timestamp`, `PM2.5`)

## Skill version

**v1.1.0** — Correctly handles sparse hourly data (e.g. sensor readings with gaps). Earlier versions misclassified sparse hourly series as daily, suppressing period=24 detection. Fixed by using the 10th-percentile gap for frequency detection and resampling to a regular grid before ACF/STL.

## Evals

3 test cases covering the main surface area:

| Case | Dataset | Key assertions |
|------|---------|----------------|
| 1 | Monthly retail | period=12, d=1, forecastable |
| 2 | Sparse hourly air quality | frequency=hourly, period=24, box_by_hour.png |
| 3 | White noise | forecastable=false, is_white_noise=true |

Run evals:
```bash
cd /Users/chen/.claude/plugins/cache/anthropic-agent-skills/example-skills/12ab35c2eb56/skills/skill-creator
python -m scripts.run_eval \
  --skill-path /path/to/time-series-analysis-demo/.claude/skills/analyzing-time-series
```

Benchmark results: v1.0 = 81%, v1.1 = 100%. Full report at `.claude/skills/analyzing-time-series-workspace/eval_report.md`.
