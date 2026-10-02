# Repository map

Part of the `bytex-trading-engine` skill. Written against BYTEX 0.11.0; when the engine disagrees, the engine wins.

```
src/Bytex.Core                   domain model, message bus, clock, cache, portfolio, engines (data, execution, risk),
                                 Strategy SDK, adapter SDK (Bytex.Core.Adapters: VenueDescriptor, capabilities), plugin contracts
src/Bytex.Indicators             technical indicators
src/Bytex.Documents              strategy documents: format, node catalog, validator, runtime (provider "bytex.document")
src/Bytex.Data                   MarketArchive (local or object storage), CsvLoader, InstrumentJson
src/Bytex.Backtest               simulated venues and matching engine, BacktestEngine, BacktestNode, batches, search, reports
src/Bytex.Live                   TradingNode, live tradingRuntime loop, network helpers, sandbox execution, control channel, node store
src/Bytex.Adapters.Binance       spot, USDⓈ-M and COIN-M futures (factory BINANCE)
src/Bytex.Adapters.Bybit         spot, linear, inverse, options (factory BYBIT)
src/Bytex.Adapters.Kucoin        spot and perpetual futures (factory KUCOIN)
src/Bytex.Adapters.Okx           spot, swaps, dated futures (factory OKX)
src/Bytex.Adapters.Kraken        spot and futures (factory KRAKEN)
src/Bytex.Adapters.Bitget        spot, USDT and USDC perpetuals (factory BITGET)
src/Bytex.Adapters.Gate          spot, USDT perpetuals and delivery futures (factory GATE)
src/Bytex.Adapters.Hyperliquid   perpetuals (factory HYPERLIQUID)
src/Bytex.Adapters.Tardis        Tardis.dev historical data (factory TARDIS)
src/Bytex.Adapters.Databento     Databento historical data (factory DATABENTO)
src/Bytex.Persistence.Redis      Redis cache state and bus streams (library use only, see `references/connectors.md`)
src/Bytex.Persistence.Postgres   the same cache state in PostgreSQL (library use only)
src/Bytex.Persistence.Shared     the key layout both stores share
src/Bytex.Persistence.S3         s3:// catalog locations (the CLI registers it)
src/Bytex.Cli                    the bytex command-line tool
examples/Bytex.Examples          C# example strategy (EmaCross), sample-catalog writer
examples/Bytex.Examples.Nodes    a plugin that brings its own document node types
examples/configs                 backtest, search, sandbox and live configs
examples/notebooks               first-backtest.ipynb
examples/reference               reference backtests of the three example documents, checked by the test suite (fixtures, not CLI configs)
tests/                           eight test projects (Core, Indicators, Data, Backtest, Live, Adapters, Documents, Cli)
bench/Bytex.Benchmarks           published benchmarks (docs/benchmarks.md)
api/*.txt                        the public surface of each package, held by a test
docs/getting-started             installation, cli, first-backtest, first-live-node, notebooks
docs/concepts                    architecture, data, backtesting, documents, strategies, indicators, orders, risk, live, plugins, redis, postgres, object-storage
docs/integrations                binance, bybit, kucoin, okx, kraken, bitget, gate, hyperliquid, sandbox, tardis, databento
docs/design                      numbered design notes 0001-0012 (contracts)
docs/extending.md                every extension point; docs/versioning.md the versioned contracts; docs/roadmap.md
```

How the pieces fit:

```
venue / vendor / CSV
   -> catalog (Parquet: instruments + bars/quotes/trades/book deltas/funding)   bytex catalog ... and the loaders in references/data.md
   -> BacktestNode: simulated venue per instrument + tradingRuntime + strategies         bytex backtest
   -> reports/<runId>/ (summary, tearsheet.html, CSVs, result.json)

live data client (any of the eight venues)
   -> TradingNode tradingRuntime (same engines, same strategies, same risk engine)       bytex run
   -> execution client: SANDBOX (simulated fills) or the venue itself (real orders)
```

CLI commands (all checked with `--help` on 0.10.0):

| Command | Does |
|---|---|
| `bytex backtest --config <file> [--output <dir>]` | one run, an array of runs, a batch (`run` key) or a guided search (`run` and `space` keys) |
| `bytex run --config <file> [options]` | a sandbox or live node until Ctrl+C, SIGTERM or `--duration` (options: `references/sandbox.md`) |
| `bytex catalog list --path <dir>` | instruments and data ranges in a catalog |
| `bytex catalog info --path <dir>` | data sets with rows, files, size and range; whether anything would stop a streaming read |
| `bytex catalog check --path <dir>` | badly named or overlapping files; exit 1 when it finds any |
| `bytex catalog consolidate --path <dir> [--kind <k>] [--key <id>]` | rewrite data sets in order, duplicates dropped, in the fewest files the 100,000-row bound allows (`--kind` includes `book_depth`) |
| `bytex catalog fetch-instruments --path <dir> --venue BINANCE\|BYBIT\|KUCOIN\|OKX [--instrument-type <t>] [--quote USDT] [--futures] [--base-url]` | instrument definitions from a venue |
| `bytex catalog add-instrument --path <dir> --file <instrument.json>` | one instrument from JSON |
| `bytex catalog import-csv --path <dir> --file <csv> --kind bars\|quotes\|trades --instrument <id> [--candle-series <bt>] [--timestamp-format <f>] [--no-header] [--columns <map>] [--separator <c>]` | CSV into the catalog |
| `bytex documents validate --document <file> [--catalog <dir>] [--environment backtest\|sandbox\|live] [--json]` | check a document |
| `bytex documents catalog` | every node type as JSON (with `--plugins`, plugin types too) |
| `bytex documents schema` | JSON Schema for documents |
| `bytex documents examples --out <dir>` | writes `ema-cross`, `breakout-retest`, `support-bounce` |
| `bytex verify-keys --venue <V> --env-file <file> [--json] [--timeout] [--base-url]` | read-only API key check, all eight venues |
| `bytex venues [--json]` | every venue's families, settings, keys, hosts, fees, free datasets, capabilities |
| `bytex version` | engine version (`bytex 0.11.0`) |

Every `--path` and `--catalog` also takes `s3://bucket/prefix`. There is **no** CLI command that downloads historical bars, funding, Tardis or Databento data, and no control-channel client: `references/data.md`, `references/tardis.md`, `references/databento.md` and `references/sandbox.md` give the programs and snippets.
