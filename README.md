# Candlestick Bible Proofing

A research project testing whether candlestick patterns predict subsequent price movements and whether strategies using them remain profitable after spreads and other trading costs.

The aim is to measure the evidence, including negative or inconclusive findings, rather than assume that patterns work.

## Current status

The repository currently contains 16 historical CSV files, a Python-oriented `.gitignore`, and an MIT license. The Streamlit dashboard, ETL program, pattern detectors, backtest engine, dependency configuration, and tests have not yet been implemented.

The data inventory below was checked against the CSV contents on October 22, 2026. These files are useful for ingestion and interface development; one day is insufficient for reliable strategy conclusions. No backtest results are claimed.

## Instruments and available data

All files contain 5-minute candles, with separate BID and ASK files. All observed timestamps fall on **January 4, 2021**, from 00:00 to 23:55 UTC.

| Instrument | Meaning | File symbol | BID rows | ASK rows |
| --- | --- | --- | ---: | ---: |
| AUD/CHF | Australian dollar / Swiss franc | AUD-CHF | 288 | 288 |
| EUR/USD | Euro / US dollar | EUR-USD | 288 | 288 |
| GBP/USD | British pound / US dollar | GBP-USD | 288 | 288 |
| USD/CAD | US dollar / Canadian dollar | USD-CAD | 288 | 288 |
| USD/CHF | US dollar / Swiss franc | USD-CHF | 288 | 288 |
| USD/CNH | US dollar / offshore Chinese yuan | USD-CNH | 287 | 287 |
| USD/JPY | US dollar / Japanese yen | USD-JPY | 288 | 288 |
| XAU/USD | Gold priced in US dollars | XAU-USD | 276 | 276 |

CNH is offshore yuan; do not substitute onshore CNY. XAU/USD is a metal instrument, not an ordinary currency pair.

BID and ASK timestamps match within each instrument. Compared with a continuous 288-slot UTC day, USD/CNH has no 22:25 bar and XAU/USD has no bars from 22:00 through 22:55. These gaps need provider/calendar verification before being classified as missing data or expected closures. Do not fill them with invented candles.

The provider, volume definition, and instrument contract specifications are not established by the CSV contents. Record them when confirmed. The earlier ten-instrument wishlist is superseded by these eight instruments for the current scope.

## Getting started now

Clone the repository:

```bash
git clone https://github.com/Hallowman3000/candlestick-bible-proofing.git
cd candlestick-bible-proofing
```

1. Read [the ETL guide](docs/ETL_GUIDE.md).
2. Inspect one BID/ASK pair in `data/`; the first column is named `Etc/UTC`.
3. Preserve original exports and record their source and download settings.
4. Expand coverage only after confirming a small download contains valid rows.
5. Implement ingestion and validation before pattern research or backtesting.

There is currently no runnable application or ETL command. Do not expect `streamlit run app.py` to work until the application exists.

### Intended application setup

The intended stack is Python, pandas, NumPy, Plotly, Streamlit, and pytest, with Parquet support for processed datasets. The implementation must declare its supported Python version and compatible dependencies.

Once `requirements.txt` and `app.py` have been implemented, the intended workflow is:

```bash
python -m venv .venv
```

Activate the environment in Windows PowerShell:

```powershell
.\.venv\Scripts\Activate.ps1
```

Or in macOS/Linux:

```bash
source .venv/bin/activate
```

Then, **only once those files exist**:

```bash
python -m pip install -r requirements.txt
python -m streamlit run app.py
```

## Data format and storage

The existing source header is:

```csv
Etc/UTC,Open,High,Low,Close,Volume
```

Example filename:

```text
EUR-USD_5Minute_BID_2021-01-04_00_00-23_59_Etc_UTC.csv
```

Timestamps are ISO 8601 strings with UTC offsets, such as `2021-01-04T00:00:00+00:00`. Instrument and price side come from the filename and verified metadata, not from separate CSV columns. Actual coverage must be calculated from rows, not trusted from filenames.

Keep the existing small files unchanged during documentation work. Store future bulk downloads outside Git, with separate raw, processed, and report directories. The proposed pipeline should accept a configurable external directory and Streamlit uploads; neither interface exists yet. Details, schemas, and acceptance checks are in the [ETL guide](docs/ETL_GUIDE.md).

## Planned research

### Patterns and setups

| Group | Patterns |
| --- | --- |
| Single candle | Doji, dragonfly doji, gravestone doji, hammer, shooting star |
| Two candles | Bullish engulfing, bearish engulfing, inside bar, tweezer top, tweezer bottom |
| Three candles | Morning star, evening star |
| Derived setups | Bullish and bearish pin bars, inside bar false breakout |

Every detector needs documented mathematical rules, configurable thresholds, and tests. A generic doji has no default trading direction. Trend context and star gap requirements must be explicit. An inside bar is distinct from a harami.

### Two separate experiments

- **Event study:** measure forward returns over 1, 3, 5, and 10 bars after completed patterns, compare matched baselines, and report sample size, effect size, and dependence-aware uncertainty.
- **Strategy backtest:** apply explicit entry, stop, target, holding period, sizing, and cost rules. Planned defaults are next-bar-open entry, one open position per instrument, and conservative stop-first handling when a candle touches both stop and target.

Final pattern thresholds and execution defaults must be recorded in configuration before evaluation. They are not implemented or calibrated yet.

### Timeframes and sessions

Build M5, M15, M30, H1, and H4 from available M5 data. Never reconstruct M1 candles from M5. Document UTC aggregation boundaries and incomplete-bar exclusions.

Compare configurable session windows in Sydney, Tokyo, London, and New York using their IANA timezones. Handle DST and overlaps. Use entry-time membership for trades; count overlapping trades once in combined totals. Offer Africa/Nairobi as a display timezone. Session windows do not override instrument trading hours.

### Spreads and execution

Use aligned BID/ASK data for execution: long entry at ask and exit at bid; short entry at bid and exit at ask. Do not subtract spread twice.

Same-boundary BID/ASK opens and closes can supply spread proxies, subject to provider timestamp semantics. Candle extrema are not synchronous spread observations. If historical executable sides are unavailable, label assumed spreads and specify units, instrument, and session adjustments.

Show gross versus net performance, slippage, commissions, applicable financing, and low/base/high spread sensitivity. Cash P&L requires verified contract and conversion information; otherwise report normalized returns or R-multiples. Keep instrument results separate unless portfolio accounting is explicitly implemented.

## Planned dashboard

- Data inventory, coverage, validation issues, and exclusions.
- Candlestick explorer with pattern markers and trade entries/exits.
- Event-study effects and matched baseline comparisons.
- Strategy results, equity, drawdown, and trade log.
- Session and overlap comparisons.
- Spread and cost sensitivity.
- Methodology, definitions, assumptions, and limitations.
- CSV exports of results and trades.

## Validation standards

Reserve the final chronological 20% for held-out evaluation by default once sufficient history exists. Keep test data out of parameter selection and purge labels or trades crossing split boundaries. All features and signals must use information available at the decision time.

Tests should cover causal signals, pattern boundaries, zero ranges, aggregation, DST, overlaps, BID/ASK accounting, gap fills, stop/target ambiguity, end-of-data exits, and split leakage. Synthetic test fixtures must remain separate from market research data.

Report sample size and multiple-testing risk; a high win rate alone does not establish profitability. Repeatedly inspecting held-out results compromises their role as an untouched test.

## Development sequence

1. Implement and test ETL against the existing 16 files.
2. Expand and validate historical coverage across all eight instruments.
3. Implement documented pattern detectors and event studies.
4. Implement execution and cost accounting.
5. Build the Streamlit views and validate held-out reporting.

A target of 2021–2025 provides five complete calendar years, subject to availability. This is a collection target, not the coverage currently present.

## License

Project code is covered by the [MIT license](LICENSE). Downloaded market data remains subject to its provider's terms; the repository license does not establish redistribution rights for it.
