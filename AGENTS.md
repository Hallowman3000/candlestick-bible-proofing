# Project instructions

## Purpose

Build a Python and Streamlit research dashboard that tests candlestick patterns against historical market data. The objective is to measure predictive value and performance after costs, not to prove that patterns work. Negative and inconclusive results are valid outcomes. Do not add live trading or broker order execution unless explicitly requested.

## Working approach

- Inspect the repository and real data schemas before changing code. Follow more specific instructions in nested AGENTS.md files within their scope.
- Implement the requested work and verify it; do not stop at a plan. Ask questions only when missing information prevents a sound implementation.
- Preserve unrelated work and existing data. Do not delete datasets, rewrite Git history, or force-push without explicit authorization.
- Prefer a small working implementation before optional features. Do not claim a feature or check is complete unless it actually works or ran.

## Instruments and data

- Support exactly eight instruments: EUR/USD, GBP/USD, USD/JPY, USD/CHF, USD/CAD, XAU/USD (gold), XAG/USD (silver), and USOIL (WTI light sweet crude oil).
- Map downloader identifiers EURUSD, GBPUSD, USDJPY, USDCHF, USDCAD, XAUUSD, and XAGUSD to their corresponding display symbols. Map LIGHT.CMD-USD to USOIL and preserve the provider identifier in metadata. Do not substitute Brent, oil ETFs, or energy stocks.
- Record provider symbols, timezone, price side, interval, coverage, and product specifications. For USOIL, identify the WTI product and known rollover treatment.
- Mark missing instruments or unknown metadata explicitly. Never silently substitute instruments or invent prices or backtest results.
- Accept a configurable local data directory and uploaded CSV files, including separate BID and ASK files and multiple files per instrument.
- Validate empty/header-only files, numeric values, OHLC relationships, timestamp parsing, duplicates, conflicting duplicates, intervals, and BID/ASK alignment.
- Normalize timestamps to timezone-aware UTC. Report excluded rows and missing intervals; distinguish expected closures where metadata permits. Never forward-fill OHLC to invent candles.
- Keep large raw and processed datasets outside Git. Track only small fixtures or samples permitted by the provider's terms. Keep credentials and secrets out of Git.
- Aggregate OHLC with explicit bar boundaries and incomplete-bar handling. Support only timeframes at least as coarse as the source data; never reconstruct finer bars.

## Current download workflow and ETL

- The user's Windows data base directory is `C:\Users\USER\Desktop\candestick proofs`. Original downloader storage is under `raw`; exported research CSVs are under `exports`. Quote Windows paths containing spaces in commands.
- Treat those as user-machine paths, not paths accessible from a remote agent or hosted Streamlit process. Make the data directory configurable; do not hardcode it into application logic. Never claim to inspect these files without actual access.
- The CLI is `duka`. The established workflow downloads M1 BID and ASK, then exports each side separately using `--resample 5min --tz UTC`. M1-to-M5 conversion is OHLC aggregation, not discretionary noise filtering.
- The coordinated download window is 2021-10-22 to 2026-10-22. Preserve it for cross-instrument comparison unless the user requests a change. Verify actual coverage and endpoint inclusivity from the data; filenames are not proof of coverage.
- Example exported names are `EURUSD_M5_BID_5years.csv`, `XAGUSD_M5_ASK_5years.csv`, and `USOIL_M5_BID_5years.csv`. Recognize these as well as legacy provider export filenames, with explicit symbol and side mapping.
- Inspect actual CSV headers before implementing the parser. Earlier samples used `Etc/UTC,Open,High,Low,Close,Volume`; CLI exports may use different headers or timestamp formats.
- Preserve raw files, produce validated normalized tables, and save processed data in compressed Parquet when practical. Record input hashes, source filenames, processing settings, rejected rows, gaps, and output counts. Publish completed runs without overwriting known-good outputs on failure.
- Validate M1 constituents where available before accepting M5 exports. Check zero-volume/flat candles, incomplete bins, missing slots, and session breaks. Do not silently delete them or assume CLI resampling already applied the research validation policy.
- Read `docs/ETL_GUIDE.md` and README.md if present. Update superseded instrument lists and distinguish small repository samples from externally downloaded history. Do not claim that external download completion establishes data quality.

## Patterns

Implement doji, dragonfly doji, gravestone doji, hammer, shooting star, bullish and bearish engulfing, inside bar, tweezer top and bottom, morning and evening star, bullish and bearish pin bars, and inside bar false breakout.

- Document mathematical definitions and configurable thresholds, including trend context and optional gap requirements.
- Handle zero-range candles safely. Distinguish inside bars from harami.
- Do not assign direction to a generic doji without an explicit rule.
- A signal becomes available only after all required candles and confirmations close. False breakout detection must not use a later confirmation retroactively.
- Permit overlapping pattern labels while preventing duplicate portfolio trades.

## Research validity

- Separate pattern event studies from executable strategy backtests.
- Measure event returns over 1, 3, 5, and 10 bars. State the reference price and compare with baselines matched on instrument, sessions, and causally defined market conditions.
- Report sample counts, effect sizes, uncertainty, and the dependence caused by overlapping observations. Use dependence-aware uncertainty methods where implemented and disclose limitations.
- Reserve the final chronological 20% for held-out evaluation by default. Keep parameter selection on development data. Purge labels and trades crossing split boundaries.
- Calculate features, regime filters, and sizing using only information available at the decision time.
- Track tested combinations and disclose multiple-testing risk. Mark small samples as insufficient evidence. Win rate alone does not establish an edge.
- Clearly distinguish exploratory results from held-out results. Do not imply that viewing held-out results repeatedly preserves an untouched test set.

## Execution and costs

- Default to entry at the next available bar open after confirmation. Define stops, targets, holding limits, sizing, and signal priority explicitly.
- Default to one open position per instrument. Skip invalid or zero stop distances.
- For aligned historical BID/ASK data: enter longs at ask and exit at bid; enter shorts at bid and exit at ask. Never subtract spread a second time.
- Synchronized BID/ASK opens and closes may provide spread observations. Do not treat differences between candle highs or lows as simultaneous spread observations.
- With one-sided or midpoint data, document how executable sides are constructed and label spread assumptions by instrument and session. State units for spread, slippage, commissions, and financing.
- Handle gaps without impossible stop fills. If both stop and target are touched within a candle and finer data cannot resolve ordering, default to stop first and disclose the assumption.
- Document end-of-data exits, unfilled orders, missing execution bars, and ambiguous fills.
- Report gross and net performance and low/base/high cost sensitivity. Show break-even spread only when the calculation is meaningful.
- Use normalized returns or R-multiples when contract specifications or currency conversion data do not support reliable cash P&L. Do not combine instruments into a portfolio without explicit capital, sizing, conversion, and exposure rules.

## Sessions

- Compare Sydney, Tokyo, London, and New York using Australia/Sydney, Asia/Tokyo, Europe/London, and America/New_York.
- Make local session hours configurable; handle daylight saving, overnight windows, boundaries, and overlaps. Offer Africa/Nairobi for display.
- Label trading results by entry-time session membership. A trade can appear in several session comparisons but must count once in combined totals.
- Explain that session labels are analysis windows subject to each instrument's trading hours.

## Code and interface

- Prefer pandas, NumPy, Plotly, Streamlit, and pytest unless existing repository choices provide a good reason otherwise. Declare the supported Python version and compatible dependencies.
- Separate ingestion, validation, resampling, patterns, sessions, event studies, execution, costs, metrics, and UI. Keep research logic independent of Streamlit.
- Use descriptive names, small cohesive functions, and explicit configuration. Add abstractions only when they solve an existing problem.
- Follow the repository's formatting and naming conventions. Avoid unnecessary dependencies, unrelated refactors, and broad exception handlers that hide failures.
- Comments should explain a decision, assumption, or non-obvious calculation. Avoid repeating the code, promotional language, boilerplate docstrings, and conversation transcripts.
- Provide overview/data quality, candle explorer, event study, strategy results, sessions, cost sensitivity, and methodology views. Show missing/undefined metrics clearly and provide CSV result and trade-log downloads.
- Cache expensive work with keys covering data and relevant settings. Do not reuse results across incompatible configurations.
- Analyze all eligible data but render a bounded candle window, initially around 200–500 bars, with date navigation and trade selection. Do not send years of candlesticks to the browser at once or downsample research inputs just to speed rendering.
- Use effect-size plots with uncertainty intervals for event studies, gross/net equity and drawdown charts for strategies, monthly/session heatmaps with sample counts, and cost-sensitivity curves. Label units, exclusions, and exploratory versus held-out results clearly.

## Verification

Run focused tests appropriate to each change. Use clearly labeled synthetic fixtures for tests, never as real research results. Cover:

- Pattern thresholds, zero ranges, and causal signal availability.
- OHLC aggregation and incomplete bars.
- DST, overnight sessions, boundaries, and overlapping membership.
- BID/ASK alignment and long/short cost accounting.
- Stop/target ambiguity, gap fills, holding limits, and end-of-data exits.
- Chronological split boundaries and prevention of future-data leakage.

For substantive UI changes, launch the app when the environment permits and check the affected flow. Document commands actually run, their results, and any blockers. Do not invent successful checks.

## Commits and documentation

- Make focused commits that represent coherent changes. Review the diff and exclude datasets, credentials, generated clutter, and unrelated edits.
- Use short, specific commit subjects describing the behavior changed, for example: `Validate bid and ask timestamp alignment` or `Fix London session boundaries during DST`.
- Explain non-obvious reasoning and relevant validation in the body when useful. Avoid generic subjects such as `update files` and repetitive generated summaries.
- Keep code and documentation direct, consistent, and maintainable. Do not introduce deliberate errors, artificial style variation, fake development history, fabricated identities, or misleading authorship claims to disguise AI involvement.
- Preserve required attribution and accurate author metadata. Follow applicable disclosure requirements. Do not claim work was written or reviewed by a person who did not do so.
- Keep the README current with installation, `streamlit run app.py`, supported schemas, external data setup, definitions, costs, detected coverage, and limitations.

## Completion report

Briefly state what changed, which checks ran, and any remaining limitations. Distinguish implemented behavior from proposed future work. If data is unavailable, deliver actionable validation messages and explain what is needed to run real research.
