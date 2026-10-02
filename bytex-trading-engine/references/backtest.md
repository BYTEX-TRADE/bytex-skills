# Backtest a strategy

Part of the `bytex-trading-engine` skill. Written against BYTEX 0.11.0; when the engine disagrees, the engine wins.

## Contents

- Before the run: data and validation
- Run config
- Venue simulation settings
- Run
- Read the report
- Simulator behaviour and limits (from `docs/concepts/backtesting.md`)

## Before the run: data and validation

Put the instrument and its data in a catalog (`references/data.md`), then validate the document against that catalog:

```bash
bytex documents validate --document documents/strategy.json --catalog ./catalog --json
bytex catalog info --path ./catalog          # rows and range per data set: the sample you are about to test
```

## Run config

```json
{
  "engine": { "runId": "rsi-dip", "tradingRuntime": { "moduleHostId": "BACKTESTER-001" } },
  "venues": [
    { "venue": "BINANCE", "accountType": "cash", "startingBalances": ["10000 USDT"] }
  ],
  "data": [
    { "catalogPath": "./catalog", "dataKind": "bars", "candleSeries": "bx-candle:v2/BINANCE/BTCUSDT/hour/1/last/provider" }
  ],
  "strategies": [
    { "providerId": "bytex.document",
      "payload": { "documentPath": "./documents/strategy.json", "strategyId": "RsiDip-001", "parameterOverrides": { "oversold": 28 } } }
  ],
  "outputDirectory": "./reports"
}
```

- `venues[].venue` must equal the instrument's venue (`BINANCE` for `bx-market:v2/BINANCE/BTCUSDT`, `SIM` for the sample). `accountType`: `cash` for spot, `margin` for perpetuals and futures.
- `data[]`: `dataKind` is `bars` (with `candleSeries`), `quotes`, `trades`, `book_deltas`, `book_depth` or `funding` (with `marketKey`); optional `start`/`end`. The run itself also takes `start`/`end` (ISO text or Unix ns). `book_deltas` or `book_depth` (order-book snapshots, free from Tardis on the first day of each month: `references/tardis.md`) makes the venue walk the book and queue resting orders; `funding` makes perpetuals pay and receive funding (the loader in `references/data.md` stores it).
- **A perpetual backtest needs a `funding` entry**, or it is charged nothing for holding positions: `{ "catalogPath": "./catalog", "dataKind": "funding", "marketKey": "bx-market:v2/BINANCE/BTCUSDT-PERP" }`.
- `chunkSize` (top level, elements per chunk): the run streams the catalog instead of loading the whole period. Leave it out for bars; set it for quotes, trades or book deltas over long periods (a month of quotes does not fit in memory).
- The document payload takes `documentPath` or an inline `document` (never both: refused), plus `strategyId`, `parameterOverrides`, `environment`, and any setting of the document strategy: `warmupBars`, `useHyphensInClientOrderIds`, `orderIdTag`, `omsType`, `externalOrderClaims`, `manageContingentOrders`, `manageGtdExpiry`, `emitDecisionEvents`, `decisionHistory`, `annotationWindow`. A name the engine does not have is refused, so spell them exactly.
- A file holding a JSON array of run configs runs them in sequence.
- A document with a `computed` candle series can backtest on `trades` or `quotes` data: the engine builds the bars from the ticks.
- Unknown fields are refused, so a misspelled setting stops the run with its name instead of being ignored.

## Venue simulation settings

Every setting of a simulated venue can be written in `venues[]` (defaults in brackets; all tested on 0.10.0):

| Setting | Values |
|---|---|
| `omsType` | [`netting`] or `hedging` |
| `accountType` | [`cash`] or `margin` |
| `baseCurrency`, `startingBalances` | e.g. `["10000 USDT"]` |
| `defaultLeverage` | [1]; the leverage the venue grants on a `margin` account. Must be at least the document's `account.leverage`, or the strategy is refused. A `cash` account always grants 1x |
| `marginModel` | [`{ "kind": "rate" }`]: the instrument's own margin rates. `{ "kind": "tiered", "tiers": [ { "notionalFrom": 0, "initial": 0.02, "maintenance": 0.01 }, { "notionalFrom": 100000, "initial": 0.05, "maintenance": 0.025 } ] }` charges more as a position grows |
| `feeModel` | [`{ "kind": "makerTaker" }`]: the instrument's rates. `{ "kind": "percent", "rate": 0.0004 }`, `{ "kind": "fixed", "amount": "1 USDT" }` per fill, `{ "kind": "perContract", "amount": "0.5 USDT", "makerAmount": "0.2 USDT" }` |
| `barExecution` | [`ohlcPath`] open, nearer extreme, farther extreme, close; `closeOnly`; `highFirst`; `lowFirst`; `tickSizePath` (walks the range in price increments, bounded by `maxBarWalkSteps` [10000]) |
| `fillSizing` | [`availableSize`]: a fill is bounded by what is on offer; `wholeFills` fills orders whole |
| `barVolumeShare` | [0.10]: the share of a bar's volume one order may take; `null` for unbounded |
| `bookType` | [`l1`], `l2`, `l3`: the depth the venue keeps |
| `liquidate` | [true]: liquidate margin accounts at maintenance margin |
| `sessionEndUtc` | e.g. `"21:00:00"`: when a `Day` order expires. Left out, at the UTC date change (what a crypto venue does) |
| `refusesOrderAmends` | [false]; true makes the strategy cancel and replace, like KuCoin futures or Gate delivery |
| `supportContingentOrders` | [true]; false makes the engine manage bracket legs itself |
| `modules` | venue behaviours the simulator does not model by default. Built in: `[{ "name": "rolloverInterest", "parameters": { "rolloverTimeOfDay": "00:00:00", "longDailyRate": "0.0001", "shortDailyRate": "0.0001" } }]` |
| `probFillOnLimit` [1], `probSlippage` [0], `fillModelSeed` [42], `latency` [`00:00:00`], `rejectStopOrdersAtMarket` [true] | the fill model |

Use `highFirst` and `lowFirst` as a measurement: run the backtest both ways. The difference between the two results is how much of the result depends on an assumption the bars cannot settle (a stop and a target inside the same bar). Report it when it is large.

## Run

```bash
bytex --log-level Warning backtest --config run.json            # --output <dir> overrides outputDirectory
```

Output: a summary on stdout and `reports/<runId>/` containing `summary.txt`, `result.json`, **`tearsheet.html`** (the whole run as one page: give it to the user), `orders.csv`, `fills.csv`, `positions.csv`, `accounts.csv`, `equity_<CCY>.csv`, and when the run had any: `funding.csv`, `liquidations.csv`, `module_charges.csv`.

Exit code 1 when a strategy faulted and did not trade (for example a document refused for leverage): the summary then opens with `STOPPED: <strategyId> faulted; the log says why`, stderr lists `Strategies that faulted and did not trade`, and `result.json` lists them in `faultedStrategies`. Read the log for the reason before anything else.

## Read the report

Summary lines: `Period`, `Iterations`, `Orders`, `Positions (closed, open)`, `Win rate`, then per currency `start`, `end`, `pnl (%)`, `realized`, `unrealized`, `fees`, `max drawdown (%)`, `sharpe`, `sortino`, `profit factor`, `expectancy`.

`result.json` has the same under `currencies[]` (`startingBalance`, `endingBalance`, `totalPnl`, `returnPercent`, `maxDrawdownPercent`, `sharpeRatio`, `sortinoRatio`, `profitFactor`, `expectancy`, `totalCommissions`, `custom`) and `trades` (`closedPositions`, `winners`, `losers`, `breakeven`, `winRate` (a fraction, 0 to 1), `averageWinner`, `averageLoser`, `largestWinner`, `largestLoser`, `averageDurationSeconds`, `longPositions`, `shortPositions`), plus `faultedStrategies`, `funding`, `liquidations`, `moduleCharges`, and the fields that say what was simulated:

- `simulation`: what the venue could do (`partialFills`, `funding`, `bookDepth`, `liquidation`, `rolloverInterest`, `tickSizePath`, `statedBarOrder`).
- `applied`: what actually happened in this run. `bookDepth` here means orders really walked a book; `barWalkBounded` means some bars were too wide to walk and were jumped.
- `participation`: `barVolumeShare`, `boundedFills`, `boundedQuantity`. A non-zero `boundedFills` means orders were larger than the bar-volume share and filled in parts over several bars; say so when reporting.
- `leverages[]`: per instrument, the leverage `requested` and `applied`, `marginInit`, `marginSource` and `marginModel`. `marginSources`: where the margin figures came from. `venuePerContract` is the venue's own figure; `venueWideDefault` means the venue published only a default (Binance futures without a key); say which when reporting a leveraged result.
- `custom` (per currency): figures a C# host added as statistics; empty from the CLI.

Report back in this order: period and bar count; closed positions and win rate; return %, max drawdown %, profit factor, Sharpe; **fees and funding as a share of gross P&L**. Then one sentence on sample size: fewer than about 30 closed positions is an anecdote, say so.

Traps when reading:

- On a cash account that starts with or ends holding the base asset, the quote-currency `end` balance moves with those holdings. Read `pnl`/`returnPercent`, not `end`.
- `profitFactor` is `null` in `result.json` (`inf` in the summary) when there was no losing trade, and `0` when there was no winner.
- Sharpe and Sortino are annualised from daily equity returns with `sqrt(252)`; a run of a few days makes them meaningless.
- The equity curve marks open positions, so max drawdown includes unrealised falls.
- An empty `funding.csv` on a perpetual means no funding data was given, not that nothing was owed.

## Simulator behaviour and limits (from `docs/concepts/backtesting.md`)

What the simulator does:

- **Fills are bounded by what is on offer.** On bars, one fill takes at most `barVolumeShare` of the bar's volume and the rest waits for the next bar; on quotes, the quoted size. So partial fills happen, and IOC and FOK are real.
- **With `book_deltas` or `book_depth` data, orders walk the book** (`applied` lists `bookDepth`): a taking order pays each level up to its limit, and FOK is decided over the whole reachable book. While the latest data is a bar, the bar bound applies instead.
- **Queue position** with book data: a resting limit stands behind the size already quoted at its price and fills only after that size trades. Without book data every resting order is at the front, filling on touch by `probFillOnLimit`.
- **Funding** is paid and received on perpetuals, but only when `funding` data is in `data[]`.
- **Liquidation** on `margin` accounts, at maintenance margin: the venue closes the position with orders tagged `LIQUIDATION`, listed in `liquidations.csv`.
- **Margin**: opening a position posts `notional x max(1 / leverage, marginInit)`, and keeping it needs `notional x marginMaint`, whatever the leverage; a tiered `marginModel` raises both with position size. Instruments loaded from a venue carry that venue's own margin rates. Working orders on a margin account hold their initial margin, and an order the free margin cannot carry is rejected with `insufficient margin`. A cash account holds the quote a buy will spend, or the base a sell will deliver. Reduce-only exits hold nothing.

Limits that remain; tell the user the ones that matter for their strategy:

- No market impact beyond the book: a book that was walked does not refill, and size does not move later prices.
- With bar data the path inside a bar is an assumption (`barExecution`). A stop and a target inside one bar resolve by that assumption: measure it with `highFirst` / `lowFirst`, or use quote, trade or book data.
- The unfilled part of a part-filled order holds no balance or margin until it fills, so a second order in that window can be judged against money the first still needs.
- Liquidation is at the touch, with no insurance fund, liquidation fee or partial liquidation. Fees have no volume tiers.
- Results are not comparable across the 0.5 (fills bounded by what is on offer), 0.6 (margin model) and 0.7 (margin rates read from each venue) boundaries.
