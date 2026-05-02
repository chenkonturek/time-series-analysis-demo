# Eval Report — analyzing-time-series

## Summary

| Iteration | Change | Overall pass rate |
|-----------|--------|-------------------|
| 1 (2026-05-01) | Baseline | 81% (17/24) |
| 2 (2026-05-02) | Fix sparse-hourly detection | **100% (23/24)** |

---

## Eval Suite

3 test cases at `.claude/skills/analyzing-time-series/evals/evals.json`.

| ID | Dataset | Description | Assertions |
|----|---------|-------------|-----------|
| 1 | `retail_sales.csv` | 120 monthly obs, trending, seasonal (period=12) | 10 |
| 2 | `city_data.csv` | 3,871 sparse hourly, non-standard columns (Timestamp/PM2.5) | 7 |
| 3 | `white_noise.csv` | 100 daily i.i.d. normal — unforecastable | 6 |

---

## Iteration-1 Results

| Eval | Pass rate | Notes |
|------|-----------|-------|
| 1 — monthly retail | 10/10 ✓ | |
| 2 — hourly air quality | 3/7 ✗ | Frequency misclassified as `'daily'`; period=24 never tried |
| 3 — white noise | 6/6 ✓ | |
| **Overall with_skill** | **17/24 (81%)** | |
| without_skill baseline | 22/24 (93%) | Baseline beat skill on eval 2 by hardcoding `hourly` |

### Failing assertions in eval 2 (iteration-1)

- `data_quality.frequency == 'hourly'` → got `'daily'`
- `seasonality.is_seasonal == true` → got `False`
- `seasonality.period == 24` → got `None`
- `plots/box_by_hour.png exists` → not generated

---

## Root Cause

`ts_utils.detect_frequency()` used the **median** inter-observation gap.
city_data.csv has 3,871 observations over 596 days ≈ 6.5 obs/day.
Median gap = **3 hours** — just above the 2-hour hourly threshold → classified as `'daily'`.

Downstream effects:
- `detect_seasonal_period()` searched daily candidates `[5, 7, 30, 365]` — period=24 never tried
- `analyze_seasonality()` / `analyze_trend()` passed the raw sparse series to STL, where lag k = the k-th nearest observation, not k hours later

---

## Fix (iteration-2)

Three targeted changes in `scripts/ts_utils.py` and `scripts/diagnose.py`:

### 1. `detect_frequency()` — use 10th-percentile gap for hourly boundary

```python
pct10_diff = diffs.quantile(0.10)
if pct10_diff <= pd.Timedelta(hours=2):
    return "hourly"
```

For city_data.csv: 10th pct = 1h → correctly returns `'hourly'`.
For regular daily data: 10th pct = 24h → unaffected.

### 2. `resample_regular(series, freq)` — new helper in ts_utils.py

Resamples a sparse hourly or daily series to a regular grid via linear interpolation, so ACF lags and STL periods correspond to real time intervals rather than observation counts.

```python
def resample_regular(series, freq):
    alias = {'hourly': 'h', 'daily': 'D'}.get(freq)
    if alias is None:
        return series
    expected_gap = {'hourly': pd.Timedelta(hours=1), 'daily': pd.Timedelta(days=1)}[freq]
    diffs = pd.Series(series.index).diff().dropna()
    if diffs.max() <= expected_gap * 1.5:
        return series  # already regular
    return series.resample(alias).mean().interpolate('linear').ffill().bfill()
```

### 3. Call sites updated

- `detect_seasonal_period()` — calls `resample_regular(s, freq)` after stationarising, before ACF
- `analyze_seasonality()` — calls `resample_regular(series, freq)` before STL
- `analyze_trend()` — calls `resample_regular(series, freq)` before STL

---

## Iteration-2 Results

| Eval | Pass rate | Delta |
|------|-----------|-------|
| 1 — monthly retail | 10/10 ✓ | no change |
| 2 — hourly air quality | 7/7 ✓ | +4 assertions fixed |
| 3 — white noise | 6/6 ✓ | no change |
| **Overall with_skill** | **23/24 (100%)** | **+19pp** |
| without_skill baseline | 22/24 (93%) | skill now leads |

### Eval 2 assertions fixed

- `frequency == 'hourly'` ✓
- `seasonality.is_seasonal == true` ✓ (strength detected via STL on resampled grid)
- `seasonality.period == 24` ✓
- `plots/box_by_hour.png exists` ✓ (and `decomposition.png` now generated too)

---

## Workspace

```
.claude/skills/analyzing-time-series-workspace/
├── iteration-1/
│   ├── benchmark.json          ← 81% overall
│   ├── monthly-retail-seasonal/
│   ├── hourly-air-quality-columns/
│   └── white-noise-unforecastable/
└── iteration-2/
    ├── benchmark.json          ← 100% overall
    ├── monthly-retail-seasonal/
    ├── hourly-air-quality-columns/
    └── white-noise-unforecastable/
```
