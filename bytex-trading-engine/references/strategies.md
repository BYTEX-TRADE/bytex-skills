# Create a strategy

Part of the `bytex-trading-engine` skill. Written against BYTEX 0.11.0; when the engine disagrees, the engine wins.

## Contents

- Strategy documents (the main path)
- Validate, always, before any run
- C# strategies (briefly)

## Strategy documents (the main path)

Start from a working file, never from a blank page:

```bash
bytex documents examples --out ./documents
```

| Example | Pattern |
|---|---|
| `ema-cross.json` | two indicators, `cond.cross` both ways, `act.order` + `act.close`, `$param` parameters |
| `breakout-retest.json` | levels, `cond.all`, risk sizing, `risk.exit` with stop, target and trail, phases and transitions |
| `support-bounce.json` | `cond.holdFor`, a target at a level |

Anatomy (every field below exists in the format; see `docs/concepts/documents.md` and `docs/design/0008-strategy-documents.md`):

```json
{
  "schemaVersion": "2.0",
  "id": "rsi-dip-trend",
  "name": "RSI dip in an uptrend",
  "description": "Buys when RSI(14) crosses back above the oversold line while the close is above the trend SMA. Stop 2 ATR under the close at entry, target 2R, stop to breakeven after 1R. Risks 1% per trade, never more than 50% of the balance in one position.",
  "instruments": [ { "ref": "primary", "marketKey": "bx-market:v2/BINANCE/BTCUSDT" } ],
  "candleSeriesDefinitions": [ { "ref": "main", "instrument": "primary", "step": 1, "aggregation": "hour", "priceType": "last", "source": "provider" } ],
  "parameters": [
    { "name": "trendLen", "type": "int", "label": "Trend SMA", "value": "50", "min": "20", "max": "400", "step": "10" },
    { "name": "oversold", "type": "decimal", "label": "Oversold line", "value": "30", "min": "10", "max": "45", "step": "1" },
    { "name": "stopAtr", "type": "decimal", "label": "Stop distance (ATR)", "value": "2", "min": "0.5", "max": "5", "step": "0.25" },
    { "name": "riskPct", "type": "decimal", "label": "Risk per trade (%)", "value": "1", "min": "0.1", "max": "3", "step": "0.1" }
  ],
  "nodes": [
    { "id": "bars", "type": "data.bars", "params": { "candleSeries": "main" } },
    { "id": "trend", "type": "ind.sma", "params": { "period": { "$param": "trendLen" } } },
    { "id": "rsi", "type": "ind.rsi", "params": { "period": 14 } },
    { "id": "atr", "type": "ind.atr", "params": { "period": 14 } },
    { "id": "uptrend", "type": "cond.compare", "params": { "op": "gt" } },
    { "id": "oversoldLine", "type": "level.pinned", "params": { "price": { "$param": "oversold" } } },
    { "id": "rsiTurn", "type": "cond.cross", "params": { "direction": "above" } },
    { "id": "entry", "type": "cond.all" },
    { "id": "stopDist", "type": "ind.math", "params": { "op": "multiply", "value": { "$param": "stopAtr" } } },
    { "id": "stopPrice", "type": "ind.math", "params": { "op": "subtract" } },
    { "id": "buy", "type": "act.order", "params": { "side": "buy", "orderType": "market", "onlyWhenFlat": true,
        "sizing": { "mode": "riskPercent", "value": { "$param": "riskPct" }, "maxPercentOfBalance": "50" } } },
    { "id": "protect", "type": "risk.exit", "params": {
        "stop": { "anchor": "level", "offset": { "unit": "price", "value": "0" } },
        "target": { "unit": "r", "value": "2" },
        "trail": { "enabled": true, "afterR": "1", "to": "breakeven", "value": "0" } } }
  ],
  "edges": [
    { "from": "bars:bars", "to": "trend:bars" },
    { "from": "bars:bars", "to": "rsi:bars" },
    { "from": "bars:bars", "to": "atr:bars" },
    { "from": "bars:close", "to": "uptrend:a" },
    { "from": "trend:value", "to": "uptrend:b" },
    { "from": "rsi:value", "to": "rsiTurn:value" },
    { "from": "oversoldLine:price", "to": "rsiTurn:reference" },
    { "from": "uptrend:out", "to": "entry:a" },
    { "from": "rsiTurn:out", "to": "entry:b" },
    { "from": "entry:out", "to": "buy:trigger" },
    { "from": "atr:value", "to": "stopDist:a" },
    { "from": "bars:close", "to": "stopPrice:a" },
    { "from": "stopDist:value", "to": "stopPrice:b" },
    { "from": "stopPrice:value", "to": "buy:stop" },
    { "from": "stopPrice:value", "to": "protect:stopLevel" },
    { "from": "buy:position", "to": "protect:position" }
  ],
  "repeat": { "enabled": true }
}
```

This document validates clean against a catalog and runs. Use it as a template for "indicator filter + trigger + computed stop + risk sizing + managed exit".

**Format rules:**

- `instruments[]` and `candleSeriesDefinitions[]` are declared once under a `ref`. The first instrument is primary; **the first candle series drives evaluation** (the graph is evaluated once per closed bar of it).
- Candle series fields: `step`, `aggregation` (`second`, `minute`, `hour`, `day`, `week`, `month`, `tick`, `volume`, `value`, and the information-driven `tickImbalance`, `volumeImbalance`, `valueImbalance`, `tickRuns`, `volumeRuns`, `valueRuns`), `priceType` (`last`, `bid`, `ask`, `mid`), `source`: `provider` for backtests on imported bars, `computed` for sandbox and live (the node builds bars from the trade stream).
- Market keys use `bx-market:v2/VENUE/<escaped-symbol>`: `bx-market:v2/BINANCE/BTCUSDT`, `bx-market:v2/BINANCE/BTCUSDT-PERP`, `bx-market:v2/KUCOIN/BTC-USDT`, `bx-market:v2/BYBIT/BTCUSDT-PERP`, `bx-market:v2/OKX/BTC-USDT-SWAP`, `bx-market:v2/KRAKEN/PF_XBTUSD`, `bx-market:v2/HYPERLIQUID/BTC-PERP` (all venues: `references/venues.md`). Candle series strings (used in run configs) are `bx-candle:v2/VENUE/<escaped-symbol>/method/step/price/origin`, for example `bx-candle:v2/BINANCE/BTCUSDT/hour/1/last/provider`.
- Prices, sizes and money are strings (`"0.5"`); integers may be numbers. Any numeric node param may be `{"$param": "name"}` pointing at `parameters[]`. Put every number a user might tune there with `min`/`max`/`step`: that is what sweeps vary.
- Edges are `"node:port"` (the last colon splits). One edge per input. Port kinds: `series`, `price`, `quantity` connect to each other; `pulse` feeds `bool` but not the reverse; `bars` and `position` connect only to themselves. No cycles.
- Phases (optional): `"phases": [{ "id", "name", "nodes": [...], "initial": true }]`, `"transitions": [{ "from", "to", "on": "buy:filled" }]`. Nodes in no phase are always active: keep actions and `risk.exit` outside phases.
- `repeat.enabled: false` finishes the strategy when its first position closes.
- `account.leverage` (default 1): the leverage the strategy is written for. See "Leverage" below.
- `evaluation` (default `barClose`): when the graph runs. `quote`, `trade` or `either` also run it on every tick of the primary instrument. On a tick frame the bar is still the last closed one, indicators do not move and bar counts do not advance; `data.price` publishes the tick's own price, and `cond.cross` with `confirm: instant` fires inside the bar. It costs hundreds of frames a minute and needs quote or trade data in the run (`EVALUATION_ON_TICKS` warns). It does not buy an intra-bar stop: `risk.exit` already rests its stop at the venue.
- `requires` (optional): node types a plugin brings, `[ { "prefix": "acme", "package": "Acme.Nodes", "minVersion": "1.2" } ]`. Load the plugin with `--plugins`, which `documents validate`, `catalog` and `schema` also read.
- **Every member is read strictly**: a misspelled or unknown member refuses the document (`DOCUMENT_UNREADABLE`). `layout` is free-form and ignored by the engine. `metadata` is typed and takes only `template` (true/false), `tags` (strings), `createdWith`, `difficulty` (`simple`, `intermediate`, `advanced`), `runMode` (`constant`, `firedOnce`), `marketRegime`, `author`.

**Leverage.** `"account": { "leverage": 5 }` states the leverage the strategy needs; it does not set any. The venue grants leverage: `defaultLeverage` on a `margin` venue in a backtest or SANDBOX config; live, the `leverage` of the execution client, which the adapter sets at the exchange before anything trades (`references/adapters.md`), or whatever the account already holds when it is left out. A `cash` account grants 1x.

- It is a floor. A venue granting less refuses the strategy: the log says `Strategy <id> is written for 5x leverage on <instrument> and this venue grants 1x ... it will not be traded`, nothing trades, and a backtest exits 1 with a `STOPPED` line. A venue granting more is fine.
- Below 1 is a validation BLOCK, `LEVERAGE_INVALID`. Omitted means 1.
- Leverage raises the ceiling on sizing: a position may be worth up to the free balance divided by `max(1 / leverage, marginInit)`, times 0.995. At 1x that is the free balance, at 5x five times it. It does not raise `percentOfBalance`, which stays a share (at most 100 %) of the free balance: use `notional`, `fixed` or `riskPercent` to size above the balance, and keep a `maxNotional` / `maxPercentOfBalance` ceiling.
- Use it only with a perpetual or future on a `margin` account. For spot, leave it out.

**Node families (88 types in 0.10.0):**

| Family | Types |
|---|---|
| data (8) | `data.bars` `data.price` `data.quotes` `data.trades` `data.book` `data.markPrice` `data.funding` `data.position` |
| ind (28) | `ind.sma` `ind.ema` `ind.dema` `ind.hma` `ind.wma` `ind.rsi` `ind.macd` `ind.stoch` `ind.adx` `ind.aroon` `ind.atr` `ind.bbands` `ind.keltner` `ind.donchian` `ind.vwap` `ind.obv` `ind.roc` `ind.slope` `ind.distance` `ind.zscore` `ind.math` `ind.ichimoku` `ind.supertrend` `ind.psar` `ind.linreg` `ind.cci` `ind.mfi` `ind.cmo` |
| level (9) | `level.range` `level.swing` `level.session` `level.vwapBands` `level.fib` `level.pinned` `level.prevBar` `level.channel` `level.round` |
| cond (14) | `cond.cross` `cond.compare` `cond.inBand` `cond.holdFor` `cond.all` `cond.any` `cond.not` `cond.once` `cond.pattern.pinBar` `cond.pattern.engulfing` `cond.pattern.divergence` `cond.time.window` `cond.time.session` `cond.time.afterBars` |
| act (13) | `act.order` `act.bracket` `act.ladder` `act.dca` `act.grid` `act.cancel` `act.close` `act.modify` `act.trail` `act.moveStop` `act.scaleOut` `act.signal` `act.note` |
| risk (8) | `risk.exit` `risk.sizing` `risk.maxPositions` `risk.maxOrders` `risk.dailyLoss` `risk.cooldown` `risk.exposure` `risk.noEntryNear` |
| flow (6) | `flow.timer` `flow.counter` `flow.latch` `flow.completeRun` `flow.gate` `flow.delay` |
| event (2) | `event.proximity` `event.active` |

`data.position` also reports the last exit while flat: `lastExitPrice`, `lastExitBarsAgo`, and `lastExitSecondsAgo` (elapsed time, so it stays right across data gaps and on a document evaluated on ticks; never negative after a restart). None of them has a value before the first exit, so a comparison against them is false until then: write a time-based cooldown as `cond.not` of `lastExitSecondsAgo < N`, not `>= N`, or the strategy never enters. Tested: 600 seconds on 1-minute bars gives the same trades as `risk.cooldown` with `bars: 10` and `afterAnyClose: true`. For a bar count, use `risk.cooldown`.

**Look up every node before you use it.** The catalog is the reference for ports, parameter names, defaults, ranges and what each enum choice means:

```bash
bytex documents catalog > catalog.json
# one type, readable:
node -e 'const c=require("./catalog.json");const t=c.types.find(x=>x.type===process.argv[1]);console.log(JSON.stringify(t,null,1))' act.order
# or with jq:
jq '.types[] | select(.type=="risk.exit")' catalog.json
```

The most used nodes, from the 0.10.0 catalog (`*` = required input):

| Node | Inputs | Outputs | Key params |
|---|---|---|---|
| `data.bars` | none | `bars`, `open`, `high`, `low`, `close`, `volume`, `typical` | `candleSeries` (a `ref`) |
| `ind.ema` / `ind.sma` / `ind.rsi` / `ind.atr` | `bars*` | `value`, `ready` | `period`, `priceType` |
| `ind.math` | `a*`, `b` | `value` | `op`: `add` `subtract` `multiply` `divide`; `value` used when `b` is unwired |
| `level.pinned` | none | `price` | `price` |
| `level.range` | `bars*` | `high`, `low`, `mid`, `width`, `ready` | `lookback`, `excludeCurrent` |
| `cond.compare` | `a*`, `b` | `out` | `op`: `gt` `gte` `lt` `lte` `eq` `neq`; `value` when `b` is unwired; `tolerance` |
| `cond.cross` | `value*`, `reference*` | `out`, `above` | `direction`: `above` `below` `either`; `confirm`: `barClose` `instant` (`instant` differs only when the document's `evaluation` runs on ticks) |
| `cond.all` / `cond.any` | `a*`, `b`, `c`, `d` | `out` | none (nest for more than four) |
| `cond.holdFor` | `in*` | `out`, `count` | `bars` |
| `act.order` | `trigger*`, `price`, `stop`, `quantity`, `atr` | `submitted`, `filled`, `rejected` (pulses), `position`, `fillPrice`, `working` | `side`, `orderType`, `sizing`, `offset`, `tif`, `postOnly`, `reduceOnly`, `onlyWhenFlat` (default true), `work`, `cancelAfterBars`, `tag` |
| `act.close` | `trigger*` | `done`, `filled`, `fillPrice` | none |
| `act.bracket` | `trigger*`, `stop*`, `target*`, `entry`, `quantity` | `submitted`, `filled`, `position`, `closed` | `side`, `sizing`, `tif`, `onlyWhenFlat` |
| `risk.exit` | `position*`, `stopLevel`, `targetLevel`, `atr` | `stop`, `target`, `rMultiple`, `managing`, `closed` | `stop.anchor` (`level` `entry` `percent`), `stop.offset.unit` (`atr` `percent` `ticks` `price`), `target.unit` (`r` `percent` `level` `none`), `trail` (`enabled`, `afterR`, `to`: `breakeven` `atr` `percent`), `anchor`, `timeStopBars` |

`act.order` sizing modes: `fixed` (base units), `notional` (quote value), `percentOfBalance` (% of free quote balance), `riskPercent` (% of balance lost if the stop is hit). Ceilings `maxNotional` and `maxPercentOfBalance` apply after the mode; 0 means none; the smaller wins. `work: { "algorithm": "twap", "horizonMinutes": "5", "intervalMinutes": "1" }` slices a market or limit order.

**Rules that prevent most failed validations and bad strategies:**

1. `riskPercent` sizing needs a price on `act.order:stop` (`SIZING_NEEDS_STOP`). Compute it (`ind.math`: close minus ATR x k) or take a level, and feed **the same price** to `risk.exit:stopLevel` so the size and the real stop agree.
2. Always cap risk sizing with `maxPercentOfBalance` or `maxNotional`. A near stop otherwise buys the whole account.
3. Protect every entry: `buy:position -> risk.exit:position`, or an `act.close` on a condition. Without one the validator warns `NO_EXIT`; treat that as a bug unless the user asked for it.
4. Spot cannot short (`SHORT_ON_SPOT`). Shorts need a perpetual instrument (`fetch-instruments --futures`) and a `margin` account.
5. For a constant line in `cond.cross`, wire `level.pinned` into `reference`; `cond.cross` needs both inputs wired.
6. Set `repeat.enabled: true` unless the strategy should stop after one trade.
7. Write `description` as one paragraph that states entry, exit, stop and sizing in plain words. It is what the user checks against their intent.
8. Check the document against the venue it will trade on. OKX, Bitget and Gate refuse triggered orders (stops and if-touched), so `risk.exit` stops and stop-type `act.order` cannot be placed there live; use market and limit orders or another venue. Nodes whose catalog entry has `amendsOrders: true` (`act.modify`, `act.moveStop`, `act.trail`, `risk.exit`) need a venue family that amends orders: KuCoin futures and Gate delivery refuse amendments (`bytex venues` shows `amendOrders` per family), so a trailing stop there stays where it was first placed.

Do not invent node types or parameters. If the catalog does not have it, the engine does not either: say what is missing and build the closest thing from existing nodes (`ind.math`, `cond.all/any/not`, `flow.*`).

## Validate, always, before any run

```bash
bytex documents validate --document documents/strategy.json --catalog ./catalog --json
```

Exit code `1` means at least one BLOCK, including `DOCUMENT_UNREADABLE` and `ENVIRONMENT_UNKNOWN` (an environment is never guessed). Match on the `code`, never the message. Fix every BLOCK; fix or explain every WARNING. Without `--catalog` the context layer (instrument, minimum size, tick, data range) is skipped and reported as `CONTEXT_SKIPPED`. Before paper or live add `--environment sandbox` or `--environment live`.

Before any layer the document has to be read: a file that is missing, empty, not JSON, or has a member the engine does not model answers `DOCUMENT_UNREADABLE` (`stoppedAt: "read"`), and an `--environment` that is not `backtest`, `sandbox` or `live` answers `ENVIRONMENT_UNKNOWN`. Validation then runs in three layers (structural, semantic, context) and stops at the first layer with a BLOCK, so fixing one BLOCK can reveal the next. Codes that commonly block, with fixes:

| Code | Level | Fix |
|---|---|---|
| `SCHEMA_VERSION`, `NAME_MISSING` | block | `"schemaVersion": "2.0"`, a `name` |
| `INSTRUMENT_MISSING`, `INSTRUMENT_ID_INVALID`, `INSTRUMENT_UNKNOWN` | block | declare `instruments[].marketKey` as `bx-market:v2/VENUE/<escaped-symbol>`; with `--catalog`, put the instrument in the catalog |
| `LEVERAGE_INVALID` | block | `account.leverage` of at least 1, or leave `account` out |
| `EVALUATION_UNKNOWN` | block | `evaluation` is `barClose`, `quote`, `trade` or `either` |
| `NODE_PROVIDER_MISSING`, `NODE_PROVIDER_UNDECLARED` | block | a plugin node type: load the plugin with `--plugins`, and declare it in `requires` |
| `BARTYPE_MISSING`, `BARTYPE_INVALID` | block | declare `candleSeriesDefinitions[]` and a `data.bars` node that uses one |
| `NODE_TYPE_UNKNOWN`, `NODE_ID_DUPLICATE`, `NODE_NOT_RUNNABLE`, `NODE_MODE_NOT_ALLOWED` | block | catalog types only, unique ids |
| `PARAM_MISSING`, `PARAM_INVALID`, `PARAM_OUT_OF_RANGE` | block | param name, type, range, enum value as in the catalog |
| `PARAM_UNKNOWN` | block | a param name the catalog does not define for that node; remove it or use the catalog's name (`bytex documents catalog`) |
| `PARAMETER_UNKNOWN`, `PARAMETER_DUPLICATE`, `PARAMETER_INVALID` | block | `$param` must name a `parameters[]` entry whose `value` is inside `min`/`max` |
| `PORT_UNKNOWN`, `EDGE_DANGLING`, `PORT_TYPE_MISMATCH` | block | real port names and node ids; compatible kinds |
| `INPUT_REQUIRED`, `INPUT_MULTIPLE` | block | wire every required input exactly once |
| `GRAPH_CYCLE` | block | break the loop; use `flow.latch` or `flow.delay` for state |
| `CAP_EXCEEDED`, `LOOKBACK_TOO_LONG` | block | fewer nodes, shorter lookbacks |
| `PHASE_DUPLICATE`, `PHASE_INITIAL`, `PHASE_NODE_UNKNOWN`, `TRANSITION_INVALID` | block | one initial phase, real node ids, transitions on firing ports |
| `UNREACHABLE_PHASE` | warning | add a transition into the phase or remove it |
| `EVALUATION_ON_TICKS` | warning | tick evaluation is expensive and needs quote or trade data in the run; keep it only if the document decides inside a bar |
| `NO_ENTRY` | block | nothing ever places an order |
| `SIZING_NEEDS_STOP` | block | wire a stop price into `act.order:stop` |
| `TARGET_INSIDE_STOP` | block | target on the wrong side of the stop |
| `CONTRADICTION` | block | two comparisons in one `cond.all` that can never both hold |
| `SHORT_ON_SPOT` | block | futures instrument and margin account |
| `MIN_QUANTITY` | block | size below the instrument minimum or off the size step |
| `ORDER_CANNOT_BE_WORKED` | block | TWAP only on market or limit orders, and not with `cancelAfterBars` |
| `NO_EXIT` | warning | add `risk.exit` or `act.close` |
| `MIN_NOTIONAL`, `PRECISION` | warning | raise the size; round the price to the tick |
| `DATA_RANGE_INSUFFICIENT` | warning | the catalog holds fewer bars than the warm-up needs |
| `LIVE_AI_EVENT_CONDITION` | block (live) | AI-classified event gating live trading needs `modes.live.allowAiAnnotationConditions` |
| `DISCONNECTED_NODE`, `CONTEXT_SKIPPED` | info | remove or wire it; add `--catalog` |

`--json` always answers in JSON on stdout, the unreadable cases included: `name`, `valid`, `stoppedAt` and `findings[]` with `level`, `code`, `nodeId`, `port`, `message`, `fix`. Never hand the user a document that has not passed validation against the catalog it will run on.

## C# strategies (briefly)

A C# strategy subclasses `Strategy<TConfig>` (config record derives from `StrategyConfig`), subscribes in `OnStart` and trades in `OnBar`/`OnQuoteTick`/`OnTradeTick`. The reference is `examples/Bytex.Examples/Strategies/EmaCross.cs`, walked through in `docs/getting-started/first-backtest.md`; the contract is `docs/design/0004-strategy-sdk.md` and `docs/concepts/strategies.md`.

The CLI loads C# strategies from plugin assemblies with provider `bytex.importable`, `name` = the full type name, `payload` = the config. Copy **only** the strategy assembly into the plugin folder. A full build output also carries the adapter DLLs, and the CLI then fails with `Plugin bytex.binance is already registered`:

```bash
dotnet build examples/Bytex.Examples -c Release
mkdir -p plugins/examples
cp examples/Bytex.Examples/bin/Release/net10.0/Bytex.Examples.dll plugins/examples/
bytex --plugins ./plugins backtest --config examples/configs/backtest-ema-cross.json
```

Prefer documents unless the user needs logic the node catalog cannot express.
