# Tardis.dev

Part of the `bytex-trading-engine` skill. Written against BYTEX 0.11.0; when the engine disagrees, the engine wins.

Package `Bytex.Adapters.Tardis`, plugin `TardisPlugin`, factory `TARDIS` (`docs/integrations/tardis.md`). It downloads Tardis daily dataset files (`trades`, `quotes`, `book_snapshot_25`, `incremental_book_L2`) from `https://datasets.tardis.dev/v1` and converts them to `TradeTick`, `QuoteTick`, `OrderBookDepth` and `OrderBookDelta`. **Historical only:** a subscription fails with "Tardis provides historical data only". It serves no bars; strategies build `computed` bars from the ticks.

**A key is optional.** Tardis serves the **first day of every month** with no authentication, and the client starts and loads those days without one (tested: trades and 25-level book snapshots with no key set). Any other day needs `TARDIS_API_KEY` (in the env file) or `apiKey` in config; without one the request fails with Tardis's own answer (`Tardis refused ... with 401 and this client has no API key. The first day of each month is free ...`).

Config (for a node or a program):

```json
{ "factory": "TARDIS", "clientId": "TARDIS", "config": {
    "apiKey": null,
    "baseUrl": "https://datasets.tardis.dev/v1",
    "cacheDirectory": "./tardis-cache"
} }
```

**Leave `exchangeMap` out**: the default maps every market this engine has to its Tardis dataset, keyed `VENUE:CLASS` or `VENUE:CLASS:INVERSE` (`BINANCE:Spot` `binance`, `BINANCE:Swap` and `BINANCE:Future` `binance-futures`, `BINANCE:Swap:INVERSE` `binance-delivery`, `BYBIT:Spot` `bybit-spot`, `BYBIT:Swap` `bybit`, `BYBIT:Option` `bybit-options`, `OKX:Spot` `okex`, `OKX:Swap` `okex-swap`, `OKX:Future` `okex-futures`, `KRAKEN:Spot` `kraken`, `KUCOIN:Spot` `kucoin`, `KUCOIN:Swap` `kucoin-futures`, `GATE:Spot` `gate-io`, `GATE:Swap` `gate-io-futures`, `BITGET:Spot` `bitget`, `BITGET:Swap` `bitget-futures`, `HYPERLIQUID:Swap` `hyperliquid`). Supplying one **replaces** the default, and a market with no entry now throws instead of guessing; the old `BINANCE-PERP` keys no longer work. Kraken futures has no Tardis dataset. `new TardisDataClientConfig().DatasetFor(instrument)` answers which dataset a market maps to (a method of the config, with the map it holds). The file requested is `<exchange>/<dataType>/<yyyy>/<mm>/<dd>/<rawSymbol>.csv.gz`, one per day; missing days are skipped. The instrument must already be in the cache, so fetch it into the catalog first. Rate limit: 30 requests per 10 seconds, 5-minute download timeout.

**There is no CLI command for Tardis.** To fill a catalog, build a small console program next to the clone (this one compiles against 0.10.0):

```bash
mkdir tardis-load && cd tardis-load
cat > tardis-load.csproj <<'EOF'
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup><OutputType>Exe</OutputType><TargetFramework>net10.0</TargetFramework><ImplicitUsings>enable</ImplicitUsings><Nullable>enable</Nullable></PropertyGroup>
  <ItemGroup>
    <ProjectReference Include="$(HOME)/.bytex/engine/src/Bytex.Adapters.Tardis/Bytex.Adapters.Tardis.csproj" />
    <ProjectReference Include="$(HOME)/.bytex/engine/src/Bytex.Data/Bytex.Data.csproj" />
  </ItemGroup>
</Project>
EOF
cat > Program.cs <<'EOF'
using Bytex.Adapters.Tardis;
using Bytex.Core.TradingRuntime;
using Bytex.Core.Model.Identifiers;
using Bytex.Core.Model.Instruments;
using Bytex.Core.Model.Primitives;
using Bytex.Core.Timing;
using Bytex.Data;

// dotnet run -- <catalog> <marketKey> <startIso> <endIso> [trades|quotes|book|depth] [depthLimit]
// depth = 25-level book snapshots, at most depthLimit of them (default 200000)
MarketArchive catalog = new(args[0]);
MarketKey id = MarketKey.Parse(args[1]);
UnixNanos start = UnixNanos.Parse(args[2]), end = UnixNanos.Parse(args[3]);
string kind = args.Length > 4 ? args[4] : "trades";
Instrument instrument = catalog.Instrument(id) ?? throw new InvalidOperationException($"{id} is not in the catalog");

using LiveClock clock = new();
using TradingRuntime tradingRuntime = new(new TradingRuntimeConfig(), clock);
tradingRuntime.Cache.AddInstrument(instrument);
TardisDataClient client = new(new ClientId("TARDIS"), new TardisDataClientConfig { CacheDirectory = "./tardis-cache" }, tradingRuntime.Services);
var data = kind switch
{
    "quotes" => await client.LoadQuotesAsync(id, start, end, null, CancellationToken.None),
    "book" => await client.LoadBookDeltasAsync(id, start, end, CancellationToken.None),
    "depth" => await client.LoadBookSnapshotsAsync(id, start, end, args.Length > 5 ? int.Parse(args[5]) : 200_000, CancellationToken.None),
    _ => await client.LoadTradesAsync(id, start, end, null, CancellationToken.None),
};
await catalog.WriteAsync(data);
Console.WriteLine($"Wrote {data.Count} {kind} for {id}");
client.Dispose();
EOF
```

On Windows replace `$(HOME)` with the clone's absolute path. Then:

```bash
bytex catalog fetch-instruments --path ../catalog --venue BINANCE --quote USDT
dotnet run -- ../catalog bx-market:v2/BINANCE/BTCUSDT 2025-01-01T00:00:00Z 2025-01-01T23:59:59Z trades     # a first-of-month day: no key needed
dotnet run -- ../catalog bx-market:v2/BINANCE/BTCUSDT-PERP 2025-01-01T00:00:00Z 2025-01-01T23:59:59Z depth 200000
```

For any other day, run it with `TARDIS_API_KEY` loaded from the env file for that one command (never typed into chat).

Backtest the ticks with `"dataKind": "trades", "marketKey": "bx-market:v2/BINANCE/BTCUSDT"` in `data[]` and a document whose candle series has `"source": "computed"`. A day of trades on a busy symbol is tens of megabytes; start with one day. Note that `documents validate --catalog` checks data coverage only for `provider` candle series, so it cannot warn that the tick range is too short for the warm-up.

**Order-book depth without a key.** `client.LoadBookSnapshotsAsync(id, start, end, limit, ct)` reads `book_snapshot_25`: one `OrderBookDepth` per row, 25 levels a side, flagged as a snapshot (it replaces the book rather than merging into it). **Always pass a limit**: a full day is about 1.5 million rows (an 89 MB download for BTCUSDT futures); `LoadBookDeltasAsync` takes no limit, and `incremental_book_L2` is about 700 MB a day. The program above writes them to the catalog (`depth`), where they are stored as `book_depth` (files of at most 100,000 snapshots each), and a backtest reads them with `{ "catalogPath": "./catalog", "dataKind": "book_depth", "marketKey": "bx-market:v2/BINANCE/BTCUSDT-PERP" }` next to the bars the document runs on; add `"chunkSize": 20000` at the top of the run config for a long period. The venue then walks the book: a taking order pays each level below the best bid or above the best ask. Tested on a first-of-month day (190,336 snapshots of `bx-market:v2/BINANCE/BTCDOMUSDT-PERP`): best bid below best ask, each side moving away from the touch, `applied` reporting `bookDepth`, fills stepping through consecutive levels, and the streamed run identical to the loaded one.

Alternative without C#: download a Tardis CSV yourself (whole files only: never send a ranged request, which Tardis answers with an HTML error page under HTTP 200), rename `amount` to `size` and `id` to `trade_id`, and import with `--kind trades --timestamp-format unix_us` (Tardis timestamps are microseconds).
