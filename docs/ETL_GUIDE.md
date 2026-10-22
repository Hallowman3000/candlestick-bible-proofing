# ETL guide

ETL means **Extract, Transform, Load**: collect source exports, turn them into validated research tables, and save those tables for the dashboard.

This document specifies the intended pipeline for this repository. No automated ETL script exists yet. Paths, configuration names, and output schemas described as proposed are implementation requirements rather than working command-line interfaces.

## 1. Start with the data actually present

The repository has eight instruments and 16 CSVs under `data/`, one BID and one ASK file per instrument:

AUD/CHF, EUR/USD, GBP/USD, USD/CAD, USD/CHF, USD/CNH, USD/JPY, and XAU/USD.

The source header is exactly:

```csv
Etc/UTC,Open,High,Low,Close,Volume
```

One actual EUR/USD BID row is:

```csv
2021-01-04T00:00:00+00:00,1.22499,1.22499,1.22429,1.22431,636460000
```

The source interval is M5. All files currently cover January 4, 2021 only. The filenames do not represent five years of downloaded history.

## 2. Extract: collect and preserve exports

### Initial workflow

1. Retain the current files as small development inputs. Do not edit their price rows.
2. Confirm the provider and export settings from the account or service used to download them; do not infer the provider solely from naming.
3. For a larger dataset, export both BID and ASK at M5, using identical date boundaries and UTC timestamps.
4. Test one short completed period first. Check that it contains prices rather than only a header.
5. Expand toward January 1, 2021–December 31, 2025 where available, in monthly or yearly batches if needed.
6. Open the first and last rows of every batch and record actual coverage. Missing start/end portions must be reported.
7. Verify that the provider permits the intended storage, research, and redistribution.

Do not invent a downloader endpoint or assume that a website export is an API. Use the provider's documented export process.

### Proposed local layout

Choose a folder outside the Git checkout, for example `C:/MarketData/candlestick-bible-proofing/`, with:

- `raw/`: original CSV exports, retained unchanged.
- `processed/`: validated, versioned Parquet tables.
- `reports/`: file inventory, rejects, gaps, and run summaries.
- `metadata/`: source manifest, calendar information, and contract specifications.

The pipeline should accept this base path explicitly. A future `MARKET_DATA_DIR` setting is a suggested interface, not a currently supported environment variable.

Keep full downloads out of Git. If project-local equivalents are added, ignore `data/raw/`, `data/processed/`, `results/`, and `logs/`. Ignoring a path does not untrack files already committed; removing tracked data requires a deliberate, separately reviewed change.

### Source manifest

For each export, record:

| Field | Meaning |
| --- | --- |
| source_file | Original relative filename |
| sha256 | Hash of the original bytes |
| provider | Confirmed source, otherwise unknown |
| instrument | Canonical symbol, e.g. EURUSD |
| provider_symbol | Source identifier, e.g. EUR-USD |
| price_side | BID or ASK |
| source_interval | 5 minutes for current files |
| timestamp_timezone | UTC |
| timestamp_semantics | Bar start/end, confirmed or unverified |
| requested_start / requested_end | Download request boundaries |
| actual_start / actual_end | Earliest/latest parsed timestamps |
| downloaded_at | Actual download timestamp if known |
| volume_definition | Confirmed unit/meaning or unknown |
| contract_metadata | Verified tick size, contract size, quote currency, etc. |
| usage_terms | Source terms reference and relevant restrictions |

Do not backfill unknown metadata with guesses. Row counts and actual boundaries belong in the generated inventory even when source information is unknown.

## 3. Transform: normalize without hiding defects

### Filename mapping

Read instrument, interval, and side from the verified filename pattern, then cross-check against configuration:

| Source token | Canonical instrument | Type |
| --- | --- | --- |
| AUD-CHF | AUDCHF | FX |
| EUR-USD | EURUSD | FX |
| GBP-USD | GBPUSD | FX |
| USD-CAD | USDCAD | FX |
| USD-CHF | USDCHF | FX |
| USD-CNH | USDCNH | Offshore yuan FX |
| USD-JPY | USDJPY | FX |
| XAU-USD | XAUUSD | Gold |

Reject ambiguous or unrecognized names until explicitly mapped. Do not substitute CNH with CNY or add markets from the superseded ten-instrument list.

### Proposed canonical candle schema

Use long-form rows, retaining BID and ASK separately:

| Column | Type / rule |
| --- | --- |
| instrument | Canonical string |
| price_side | BID or ASK |
| timeframe | M5 initially |
| timestamp | Timezone-aware UTC; verified bar-start semantics |
| open / high / low / close | Finite numeric prices |
| volume | Numeric if supplied; preserve unknown semantics |
| source_file | Original file identifier |
| source_row | Original data-row number |
| run_id | Processing run identifier |

The unique key is `instrument, price_side, timeframe, timestamp`.

Rename `Etc/UTC` to `timestamp` and the OHLCV headers to lowercase. Parse the explicit ISO 8601 format with offsets; fail or quarantine malformed values instead of silently converting them to missing timestamps. Preserve original timestamp text for rejected rows.

For these current instruments, non-positive prices require investigation and exclusion from research unless a documented product rule permits them. Keep that check instrument-specific rather than assuming every future market must be positive.

### Validation order

1. Check readable encoding, delimiter, required headers, empty files, and header-only files.
2. Parse timestamps and numeric columns; reject NaN/infinite OHLC values.
3. Require `high >= max(open, close)`, `low <= min(open, close)`, and `high >= low`.
4. Keep valid zero-range candles; detectors must avoid division by zero.
5. Sort by timestamp within instrument/side.
6. Collapse identical duplicates deterministically while preserving all contributing source references in a lineage report.
7. Quarantine conflicting rows with the same unique key. Do not let arbitrary file order decide the winning price.
8. Check the M5 grid, interval steps, and UTC boundaries.
9. Report gaps before deciding whether they are expected closures.
10. Align valid BID and ASK rows on exact timestamps, then validate comparable boundaries.

Return a reject table containing source file, original row, reason, and original values. Count exclusions in the run report. Do not silently repair OHLC, forward-fill prices, shift timestamps to force alignment, or replace rejected values with zero.

### Gaps and current acceptance fixture

A continuous UTC day has 288 five-minute slots, but that is a diagnostic grid, not a universal trading calendar.

| Instrument | Rows per side | Absent UTC slots on January 4, 2021 |
| --- | ---: | --- |
| AUDCHF | 288 | None |
| EURUSD | 288 | None |
| GBPUSD | 288 | None |
| USDCAD | 288 | None |
| USDCHF | 288 | None |
| USDCNH | 287 | 22:25 |
| USDJPY | 288 | None |
| XAUUSD | 276 | 22:00–22:55 inclusive |

Both sides share these timestamp sets. The inspected rows satisfy basic OHLC inequalities. This is a limited inspection, not certification of every research or execution assumption.

The gold gap may correspond to a product trading break, but that is unverified. Keep both gold and CNH gaps classified as unexplained until checked against provider hours. Report leading/trailing coverage gaps against the requested interval, not just holes between observed rows.

### BID/ASK alignment and spread information

Build a paired research view with `bid_open/high/low/close`, `ask_open/high/low/close`, side-specific volumes, and side-specific lineage.

Use an outer join for the audit so unmatched timestamps remain visible. Publish paired execution rows only where both validated sides exist. Preserve unpaired valid rows for explicitly one-sided research; never synthesize the missing side without a labeled assumption.

At comparable boundaries calculate:

```text
spread_open  = ask_open  - bid_open
spread_close = ask_close - bid_close
```

Flag negative boundary spreads for investigation. Provider OHLC opens/closes may be the first/last quote within a bucket rather than truly simultaneous ticks: label these as bar-boundary proxies unless synchronization is verified. Do not infer a continuous intrabar spread series from them.

Do not use `ask_high - bid_high` or `ask_low - bid_low` as contemporaneous spreads. Do not construct exact midpoint highs/lows by averaging independently occurring BID/ASK extrema.

For example, the first EUR/USD open difference is `1.22503 - 1.22499 = 0.00004` price units, equivalent to 0.4 conventional EUR/USD pips. This is a single boundary observation, not a typical-spread estimate.

Keep spread storage in price units. Conventional FX pip sizes are 0.0001 for the current non-JPY FX pairs and 0.01 for USD/JPY; broker point/tick sizes can differ. Do not assume a gold pip or cash value without verified specifications.

## 4. Aggregate timeframes

Validate M5 first, then aggregate each price side independently using UTC-aligned, left-closed, left-labeled bins anchored at midnight UTC.

| Target | Expected M5 bars in a fully open interval |
| --- | ---: |
| M5 | 1 |
| M15 | 3 |
| M30 | 6 |
| H1 | 12 |
| H4 | 48 |

Use first open, maximum high, minimum low, and last close. Sum volume only if the provider's definition supports that operation; otherwise retain raw volume and mark aggregate volume unavailable.

Count constituent bars and verify their exact expected timestamps. A count alone can hide off-grid observations. Exclude incomplete higher-timeframe candles from signals by default, recording the reason. Do not aggregate across an unexplained missing bar. If calendar-aware handling of scheduled breaks is later introduced, document its expected slots and how partially traded bins are treated.

Bar timestamps label the start; a completed H1 candle starting at 10:00 becomes usable at 11:00, not 10:00. Save or derive `available_at` explicitly. The source's timestamp semantics must be verified before relying on that timing.

Never generate M1 from M5. Do not infer intrabar price ordering from highs and lows.

## 5. Add session labels

Store candles in UTC. Derive configurable local session windows using:

| Window | IANA timezone |
| --- | --- |
| Sydney | Australia/Sydney |
| Tokyo | Asia/Tokyo |
| London | Europe/London |
| New York | America/New_York |

Use half-open intervals: start included, end excluded. Handle windows crossing midnight and DST using local dates and timezone rules. Store session membership as multiple flags or a separate membership table; overlaps are valid. No fixed UTC offset works for all seasons.

Candle membership must state whether it refers to bar start or another boundary. Strategy reporting must label membership at actual entry time. An optional Africa/Nairobi display conversion must not change stored timestamps or research ordering.

Session windows are analysis conventions, not confirmation that a product was trading.

## 6. Load: publish reproducible outputs

Proposed processed outputs:

- `candles/`: Parquet tables partitioned by instrument, side, timeframe, and year.
- `paired/`: aligned BID/ASK tables for execution.
- `inventory.csv`: files, hashes, schema status, row counts, coverage, and metadata.
- `rejects.csv`: invalid rows and explicit reasons.
- `duplicates.csv`: exact/conflicting duplicates and lineage.
- `gaps.csv`: missing slots, durations, calendar classification, and confidence.
- `alignment.csv`: matched/unmatched timestamp counts by instrument.
- `run.json`: run ID, input hashes, code revision, configuration hash, parser version, policies, and output counts.

Keep rejected/raw data separate from approved research inputs. A partial instrument failure may leave other instruments usable, but the dashboard must report the failure.

Write into a temporary run directory, validate outputs, and publish atomically within the same filesystem. Do not replace a known-good dataset with a failed run. Use input hashes plus parser/configuration versions to identify equivalent runs; identical inputs and policies should produce identical normalized tables.

For incremental data, validate new batches, detect overlap and conflicts with prior batches, recompute affected aggregates and reports, and publish a new run. Never append blindly.

The dashboard should load only a completed run and include its ID in exported results. Cache keys must include input/run identity, timeframe, and relevant settings.

## 7. Protect downstream research

- Keep raw data complete; apply chronological development/test splits in research with recorded timestamps.
- Default to the final 20% held out only after enough history is collected. One day does not become strong evidence merely by splitting it.
- Prevent forward labels and trades from crossing split boundaries; drop boundary observations as required.
- Fit any data-dependent normalization, matching thresholds, or regime thresholds on development data only.
- Compute causal features without future candles.
- Use completed patterns, then enter no earlier than the next eligible bar.
- Use ASK/BID execution sides and do not charge spread a second time.
- Disclose bar-level ambiguity, missing execution bars, slippage, commissions, financing, and contract/conversion limitations.

## 8. Acceptance checklist for the first implementation

1. All 16 current files are discovered and mapped to the eight instruments.
2. Current row counts and UTC coverage match the table above.
3. Matching BID/ASK keys are verified; neither side is lost silently.
4. Malformed inputs, conflicting duplicates, header-only files, and nonfinite prices produce actionable reports.
5. Gaps remain visible and no OHLC rows are manufactured.
6. Aggregation reproduces manually checked OHLC and excludes incomplete bins.
7. Session tests cover winter/summer dates, overnight windows, and overlaps; the current January sample cannot exercise all DST cases.
8. An unchanged rerun reproduces normalized results and provenance.
9. A failed run cannot overwrite the previous published dataset.
10. Research uses the selected run explicitly and records exclusions.

Use synthetic fixtures for edge cases; label them as test data and keep them out of real-data performance reports.

## Troubleshooting

| Symptom | Action |
| --- | --- |
| Only column headings | Re-export a nonzero completed date range; inspect actual rows before expanding |
| Files show only one date | Check export start/end controls; filename and actual timestamps must both be reviewed |
| Different BID/ASK coverage | Compare keys, report unmatched rows, and re-export matching ranges if needed |
| Duplicate timestamps | Separate exact overlap from conflicting prices; do not choose a winner silently |
| Wrong local session | Confirm UTC parsing, local timezone, DST, and configured session hours |
| Missing gold hour | Verify provider/product hours; retain an unexplained gap until confirmed |
| Negative spread proxy | Check mappings, source semantics, timestamp alignment, and parsing |
| Apparent huge volume | Verify provider units; do not assume centralized traded volume |
| Streamlit cannot start | The app is not implemented yet; complete application files and dependencies first |
