# Data-Corrector

# Data-Corrector

## Overview

Data-Corrector is a financial data quality pipeline designed to distinguish data provider errors from legitimate market events across a universe of 6,081 tickers. It ingests raw daily market and quarterly fundamental data, applies a sequence of deterministic sanity checks and statistical outlier filters, produces corrected datasets, and generates structured audit logs suitable for forensic review by an LLM.

The full analysis report can be found in `Analysis/Report.pdf`.

## Pipeline Architecture

The pipeline processes data in four sequential phases:

### Phase 1 — Data Loading

Ticker CSV files and a universe metadata file are loaded as Polars LazyFrames via `read_csv_files_to_polars()`. Lazy evaluation defers computation until the final output stage, keeping memory usage low across all 6,081 tickers. The metadata (sector, industry, exchange, market cap, etc.) is loaded once and shared by reference.

### Phase 2 — Sanity Checks

Eight rule-based stages run in sequence. Each stage is parallelized across tickers using `ThreadPoolExecutor` (default 8 workers, batches of 512). Every stage mutates the shared data dictionary in-place and emits audit log entries recording the original and corrected values.

|
 Stage 
|
 Function 
|
 What it does 
|
|
-------
|
----------
|
--------------
|
|
 1 
|
`sort_dates`
|
 Enforces chronological order and removes duplicate dates 
|
|
 2 
|
`fill_negatives_fundamentals`
|
 Replaces negative fundamental values via quarterly forward-fill 
|
|
 3 
|
`fill_negatives_market`
|
 Corrects negative market prices using backward-looking cubic spline interpolation (no look-ahead bias) 
|
|
 4 
|
`zero_wipeout`
|
 Detects rows where shares outstanding = 0 but volume > 0 and forward-fills 
|
|
 5 
|
`mkt_cap_scale_error`
|
 Flags ≥10x jumps in shares outstanding (scale errors) and forward-fills 
|
|
 6 
|
`ohlc_integrity`
|
 Enforces High ≥ max(O,H,L,C), Low ≤ min(O,H,L,C), and recalculates VWAP 
|
|
 7 
|
`validate_financial_equivalencies`
|
**
Hard filters
**
: corrects Assets = Current + Noncurrent via proportional scaling. 
**
Soft filters
**
: flags equity identity, cash equivalence, and accounting equation violations without correction 
|
|
 8 
|
`validate_market_split_consistency`
|
 Validates that split/dividend-adjusted prices equal raw prices multiplied by the cumulative adjustment factor, and corrects mismatches 
|

### Phase 3 — Statistical Filters

Four model-based stages detect outliers that pass the sanity checks. Flagged values are nullified and forward-filled. Each filter targets a different statistical dimension:

|
 Stage 
|
 Filter 
|
 Approach 
|
|
-------
|
--------
|
----------
|
|
 1 
|
 Rolling Z-Score 
|
 Time-series: flags values beyond a dynamic threshold (≈3.8σ) over a 21-day rolling window 
|
|
 2 
|
 Mahalanobis Distance 
|
 Cross-sectional: groups tickers by sector, computes robust covariance (MinCovDet / Ledoit-Wolf), and flags multivariate outliers using a chi-squared threshold 
|
|
 3 
|
 MAD (Median Absolute Deviation) 
|
 Robust univariate: uses modified Z-scores that are resistant to masking by extreme values 
|
|
 4 
|
 GARCH Residuals 
|
 Volatility-adjusted: fits a GARCH(1,1) model with Student-t innovations (Numba-accelerated) and flags standardized residuals that exceed the threshold 
|

### Phase 4 — Output and Visualization

Corrected data is written to CSV or Parquet via a streaming strategy: each ticker is collected, written, and immediately released from memory. Two JSON audit logs are saved — one for the sanity checks and one for the statistical filters — preserving a full before/after trail for every correction made.

An interactive Dash dashboard (`launch_dashboard()`) lets users browse the audit logs, filter by ticker or error category, visualize before/after comparisons, and mark false positives for downstream LLM-assisted forensic analysis.

## Key Design Decisions

- **Polars LazyFrames** — All transformations are expressed as lazy query plans. Data is only materialized during the output stage, keeping peak memory well below what 6,081 concurrent DataFrames would require.
- **In-place mutation** — The ticker dictionary is mutated in-place at every stage rather than copied, further reducing memory pressure.
- **Batch parallelism with GC** — Tickers are processed in batches of 512 across 8 threads, with explicit garbage collection between batches to prevent memory accumulation.
- **Hard vs. soft filters** — Structural violations (e.g., Assets ≠ Current + Noncurrent) are corrected automatically. Logical inconsistencies that may reflect legitimate accounting (e.g., equity identity deviations) are flagged in a `data_warning` column but left uncorrected, avoiding false corrections.
- **No look-ahead bias** — Market price interpolation uses only backward-looking data, and rolling windows are strictly trailing, preserving the dataset's usability for time-series modeling.

## Tech Stack

|
 Category 
|
 Libraries 
|
|
----------
|
-----------
|
|
 Data processing 
|
 Polars, NumPy, PyArrow 
|
|
 Statistics 
|
 SciPy, scikit-learn (robust covariance), statsmodels, arch (GARCH) 
|
|
 Acceleration 
|
 Numba (JIT-compiled GARCH core), ThreadPoolExecutor 
|
|
 Visualization 
|
 Dash, Plotly, dash-bootstrap-components, dash-ag-grid 
|
|
 Runtime 
|
 Python 3.11, Conda 
|

To run the data cleaning pipeline:
- Create a new environment
- Run the environment.yml file
- Paste the input files into R_Data-Corrector\Input\Data
- Paste the universe information file to R_Data-Corrector\Input\Universe_Information
- Run the main

To replicate the forensic analysis:
- Select a language model with research capabilities (Gemini 3 Pro deep research mode was used for the report)
- Attach one of the the error logs found in R_Data-Corrector\Output\full_pipeline\error_logs
- Go to R_Data-Corrector\Analysis\prompts, copy the prompt which corresponds to the selected error log and paste it as an instruction

**Note:** As mentioned in the report, the soft filters from validate_financial_equivalencies are numerous and contain a very high number of false positives. Therefore, It is advised to remove the logs from this category so the LLM does not get distracted from genuine logs during analysis.



