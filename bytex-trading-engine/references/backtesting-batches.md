# Backtesting beyond one run

Part of the `bytex-trading-engine` skill. Written against BYTEX 0.11.0; when the engine disagrees, the engine wins.

A backtest config whose top-level object has a `run` key is a **batch**: one run plus what to vary. Every combination of instruments x periods x sweeps is its own run with run id `<runId>-000`, `-001`, ... in a fixed order (instrument, then period, then sweeps as given).

```json
{
  "run": {
    "engine": { "runId": "rsi-dip", "tradingRuntime": { "moduleHostId": "BACKTESTER-001" } },
    "venues": [ { "venue": "BINANCE", "accountType": "cash", "startingBalances": ["10000 USDT"] } ],
    "data": [ { "catalogPath": "./catalog", "dataKind": "bars", "candleSeries": "bx-candle:v2/BINANCE/BTCUSDT/hour/1/last/provider" } ],
    "strategies": [ { "providerId": "bytex.document",
      "payload": { "documentPath": "./documents/strategy.json", "strategyId": "RsiDip-001" } } ],
    "outputDirectory": "./reports"
  },
  "periods": [
    { "start": "2025-01-01T00:00:00Z", "end": "2025-05-01T00:00:00Z", "label": "IS" },
    { "start": "2025-05-01T00:00:00Z", "end": "2025-09-01T00:00:00Z", "label": "OOS" }
  ],
  "sweeps": [
    { "path": "parameterOverrides.oversold", "values": ["25", "30", "35"] },
    { "path": "parameterOverrides.stopAtr", "values": ["1.5", "2", "3"] }
  ],
  "maxParallel": 4
}
```

- **Sweeps** set a value in the strategy payload at a dotted path (`parameterOverrides.<name>`; a numeric segment indexes an array). `parameterOverrides` is created if missing; any other missing parent is refused as a typo before anything runs. `strategy` (index, default 0) picks which strategy of the run a sweep targets.
- **Parameter sets** instead of sweeps: `"parameterSets": [ { "values": { "parameterOverrides.oversold": "30", "parameterOverrides.stopAtr": "2" } }, ... ]` runs exactly those points, one run each, in the order given (`strategy` index as for sweeps). Use it to re-run chosen points, such as a search's best few, or the in-sample winners on the out-of-sample period. A batch takes sweeps or sets, never both (refused).
- **Periods** set the run's start and end, with a label for the table.
- **Instruments:** `"instruments": ["bx-market:v2/BINANCE/BTCUSDT", "bx-market:v2/BINANCE/ETHUSDT"]` repoints every data stream (candle series keep step, price and source), and `instrumentPaths` / `candleSeriesPaths` write the instrument into the strategy payload. With documents, use an **inline** `document` in the payload (a `documentPath` payload has no `document.instruments` to write into) and `"instrumentPaths": ["document.instruments.0.marketKey"]`. Each instrument needs its data in the catalog.
- **maxParallel**: runs at once (default 1). Runs are independent engines; results do not depend on it.
- A run that fails is a row with its reason, not a hole. `stopOnFailure: true` stops the batch instead.
- `reportName` [`batch`] names the table file (`<reportName>_<CCY>.csv`), so two batches can share an `outputDirectory`.

Output: one table per currency on stdout (the Parameters column is truncated), each run's own report directory, and `reports/batch_<CCY>.csv` with columns `Index,RunId,Instrument,Period,Parameters,Trades,TotalPnl,ReturnPercent,MaxDrawdownPercent,SharpeRatio,ProfitFactor,Failure`. **Read the CSV, not the table.** The exit code is 1 only when no run produced a result.

## Guided search

A grid of six parameters at ten values each is a million runs. A **search** breeds generations of candidates over a space instead (a genetic search). A config whose top-level object has both `run` and `space` is a search:

```json
{
  "run": { "...": "the same run object as in a batch" },
  "space": [
    { "path": "parameterOverrides.oversold", "min": 20, "max": 40, "step": 1 },
    { "path": "parameterOverrides.stopAtr", "min": 1, "max": 4, "step": 0.25 },
    { "path": "parameterOverrides.riskPct", "choices": ["0.5", "1", "2"] }
  ],
  "population": 24, "generations": 5, "elites": 2,
  "crossoverRate": 0.5, "mutationRate": 0.2, "tournamentSize": 3,
  "seed": 7, "maxParallel": 4, "stopAfterGenerationsWithoutImprovement": 3,
  "objective": { "figure": "sharpeRatio", "currency": "USDT", "minimize": false }
}
```

- Each `space` entry is a numeric range (`min`, `max`, `step` [1]) or a list of `choices`, never both; `strategy` (index) as for sweeps. Only values the space allows are ever run.
- `objective` is required in a file: `figure` is one of `totalPnl`, `returnPercent`, `maxDrawdownPercent` (with `"minimize": true`), `sharpeRatio`, `profitFactor`, `trades`; `currency` is the table to read. A figure a C# host adds as a custom statistic can be named too, if it is also listed in `objective.custom`; from the CLI there are none.
- Defaults in brackets: `population` [24], `generations` [5], `elites` [2], `crossoverRate` [0.5], `mutationRate` [0.2], `tournamentSize` [3], `seed` [7], `maxParallel` [1], `stopOnFailure`, `stopAfterGenerationsWithoutImprovement`. The same seed gives the same candidates; a point already run is not run again; the elites carry forward, so the best never gets worse.
- A search varies parameters only. For several instruments, run one search per instrument; `instruments` and `periods` belong to batches.
- Output: the generation table and `Best: <point> (<figure> <value>) in <runId>` on stdout; exit 1 when no candidate produced a result. In `outputDirectory`: `search.txt` (the same summary), **`search_<CCY>.csv`** (one row per candidate in the order proposed: `Generation,Candidate,Reused,Fitness,Point,RunId,Trades,TotalPnl,ReturnPercent,MaxDrawdownPercent,SharpeRatio,ProfitFactor,Failure`), `generation-NN_<CCY>.csv` per generation, and each candidate's report directory `<runId>-gNN-NNN/`. Read `search_<CCY>.csv`; a `Reused: yes` row repeats a point already run, so count only `no` rows as runs.
- A search is tuning. Its best point is an in-sample result: check it out of sample (below) before reporting it as anything more.

**Walk-forward** is two stages the engine does not automate: sweep the grid over the in-sample period, choose parameters from the IS rows, then report the **out-of-sample row for those same parameters** (one batch with both periods gives both). With a search: search on the in-sample period only, then run the best few points as `parameterSets` over the out-of-sample period. Never present the best in-sample row as the expected result. If the grid wins in-sample and loses out-of-sample, say plainly that the edge did not survive. Prefer parameters whose neighbours also do well over a single sharp peak. Hold out data the user never tuned on.

Multi-instrument in one run (a portfolio, not a scan): list several `data[]` entries and several strategies, or a document with several `instruments`; one venue entry per venue.
