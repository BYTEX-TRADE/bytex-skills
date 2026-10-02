# Get data into a catalog

Part of the `bytex-trading-engine` skill. Written against BYTEX 0.11.0; when the engine disagrees, the engine wins.

## Contents

- The catalog
- Instruments and bars from a venue (all eight venues, one program)
- Instruments from the CLI or from JSON
- Bars, quotes and trades from CSV
- Checking and repairing a catalog
- Other sources (Tardis, Databento, synthetic)

## The catalog

A backtest reads a market archive, which needs the **instrument definition** and the **data** (`bars`, `quotes`, `trades`, `book_deltas`, `book_depth`, `funding`). Layout: `instruments/<encoded-market-key>.json`, `<kind>/<encoded-key>/segments/<opaque-id>.parquet` and immutable `commits/<opaque-id>.json` manifests. Each segment holds at most 100,000 rows. Ranges, counts, hashes and ordering live in manifests, not filenames. Every `--path`, `catalogPath` and `--catalog` also takes object storage: `s3://bucket/prefix` (`references/connectors.md`, "Object storage"). Read [migration.md](migration.md) before converting legacy data.

**Every bar is stamped at its close.** A bar stamped at its open is seen by the strategy before it finished (look-ahead). Everything the engine fetches itself is stamped at the close; anything you import must be too.

## Instruments and bars from a venue (all eight venues, one program)

The CLI has **no download command for history**, but every venue adapter ships public history helpers (`BinanceHistory`, `BybitHistory`, `KucoinHistory`, `OkxHistory`, `KrakenHistory`, `BitgetHistory`, `GateHistory`, `HyperliquidHistory`: the code a node uses for its warm-up) and an instrument provider that loads one instrument without a key. This program uses them to put an instrument, its closed bars and, for a perpetual, the funding rates the venue charged into a catalog. Build it next to the clone (it compiles and ran against 0.10.0 on every family below):

```bash
mkdir venue-load && cd venue-load
cat > venue-load.csproj <<'EOF'
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup><OutputType>Exe</OutputType><TargetFramework>net10.0</TargetFramework><ImplicitUsings>enable</ImplicitUsings><Nullable>enable</Nullable></PropertyGroup>
  <ItemGroup>
    <ProjectReference Include="$(HOME)/.bytex/engine/src/Bytex.Adapters.Binance/Bytex.Adapters.Binance.csproj" />
    <ProjectReference Include="$(HOME)/.bytex/engine/src/Bytex.Adapters.Bybit/Bytex.Adapters.Bybit.csproj" />
    <ProjectReference Include="$(HOME)/.bytex/engine/src/Bytex.Adapters.Kucoin/Bytex.Adapters.Kucoin.csproj" />
    <ProjectReference Include="$(HOME)/.bytex/engine/src/Bytex.Adapters.Okx/Bytex.Adapters.Okx.csproj" />
    <ProjectReference Include="$(HOME)/.bytex/engine/src/Bytex.Adapters.Kraken/Bytex.Adapters.Kraken.csproj" />
    <ProjectReference Include="$(HOME)/.bytex/engine/src/Bytex.Adapters.Bitget/Bytex.Adapters.Bitget.csproj" />
    <ProjectReference Include="$(HOME)/.bytex/engine/src/Bytex.Adapters.Gate/Bytex.Adapters.Gate.csproj" />
    <ProjectReference Include="$(HOME)/.bytex/engine/src/Bytex.Adapters.Hyperliquid/Bytex.Adapters.Hyperliquid.csproj" />
    <ProjectReference Include="$(HOME)/.bytex/engine/src/Bytex.Data/Bytex.Data.csproj" />
  </ItemGroup>
</Project>
EOF
cat > Program.cs <<'EOF'
using Bytex.Adapters.Binance;
using Bytex.Adapters.Bitget;
using Bytex.Adapters.Bybit;
using Bytex.Adapters.Gate;
using Bytex.Adapters.Hyperliquid;
using Bytex.Adapters.Kraken;
using Bytex.Adapters.Kucoin;
using Bytex.Adapters.Okx;
using Bytex.Core.Adapters;
using Bytex.Core.Model;
using Bytex.Core.Model.Data;
using Bytex.Core.Model.Instruments;
using Bytex.Data;

// dotnet run -- <catalog> <family> <candleSeries> <startIso> <endIso>
// <family> is a family name from `bytex venues`, e.g. spot, usdm-futures, linear, swap, perpetuals.
// Loads the instrument from the venue (public data, no key) into the catalog, then the closed bars of the window,
// and for a perpetual also the funding rates the venue charged in it.
MarketArchive catalog = new(args[0]);
string family = args[1].ToLowerInvariant();
CandleSeries candleSeries = CandleSeries.Parse(args[2]);
DateTimeOffset start = DateTimeOffset.Parse(args[3]), end = DateTimeOffset.Parse(args[4]);
string venue = candleSeries.MarketKey.Venue.Value;

(IDisposable http, IInstrumentProvider provider, Func<Instrument, Task<IReadOnlyList<Bar>>> fetch, Func<Instrument, Task<IReadOnlyList<FundingRateUpdate>>> funding) = (venue, family) switch
{
    ("BINANCE", _) => Binance(family switch { "spot" => BinanceAccountType.Spot, "usdm-futures" => BinanceAccountType.UsdMFutures, "coinm-futures" => BinanceAccountType.CoinMFutures, _ => throw Unknown() }),
    ("BYBIT", _) => Bybit(family switch { "spot" => BybitProductType.Spot, "linear" => BybitProductType.Linear, "inverse" => BybitProductType.Inverse, _ => throw Unknown() }),
    ("KUCOIN", "spot" or "futures") => Kucoin(family == "futures" ? KucoinProductType.Futures : KucoinProductType.Spot),
    ("OKX", _) => Okx(family switch { "spot" => OkxInstrumentType.Spot, "swap" => OkxInstrumentType.Swap, "futures" => OkxInstrumentType.Futures, _ => throw Unknown() }),
    ("KRAKEN", "spot" or "futures") => Kraken(family == "futures" ? KrakenProductType.Futures : KrakenProductType.Spot),
    ("BITGET", _) => Bitget(family switch { "spot" => BitgetProductType.Spot, "usdt-futures" => BitgetProductType.UsdtFutures, "usdc-futures" => BitgetProductType.UsdcFutures, _ => throw Unknown() }),
    ("GATE", _) => Gate(family switch { "spot" => GateProductType.Spot, "futures" => GateProductType.Futures, "delivery" => GateProductType.Delivery, _ => throw Unknown() }),
    ("HYPERLIQUID", "perpetuals") => Hyperliquid(),
    _ => throw Unknown(),
};

using (http)
{
    await provider.LoadAsync(candleSeries.MarketKey, CancellationToken.None);
    Instrument instrument = provider.Find(candleSeries.MarketKey)
        ?? throw new InvalidOperationException($"{candleSeries.MarketKey} is not listed in {venue}'s {family} family");
    await catalog.WriteInstrumentsAsync([instrument]);
    IReadOnlyList<Bar> bars = await fetch(instrument);
    await catalog.WriteBarsAsync(bars);
    Console.WriteLine($"Wrote {instrument.Id} and {bars.Count} bars of {candleSeries}"
        + (bars.Count > 0 ? $" ({bars[0].CreatedTime} .. {bars[^1].CreatedTime}, stamped at close)" : ""));
    if (instrument.InstrumentClass == InstrumentClass.Swap)
    {
        IReadOnlyList<FundingRateUpdate> rates = await funding(instrument);
        await catalog.WriteFundingRatesAsync(rates);
        Console.WriteLine($"Wrote {rates.Count} funding rates of {instrument.Id}");
    }
}

static ArgumentException Unknown() => new("unknown venue/family pair: run `bytex venues` for the family names");

(IDisposable, IInstrumentProvider, Func<Instrument, Task<IReadOnlyList<Bar>>>, Func<Instrument, Task<IReadOnlyList<FundingRateUpdate>>>) Binance(BinanceAccountType t)
{
    BinanceHttp h = new(new BinanceDataClientConfig { AccountType = t });
    return (h, new BinanceInstrumentProvider(h, t), i => BinanceHistory.FetchBarsAsync(h, i, candleSeries, start, end), i => BinanceHistory.FetchFundingRatesAsync(h, i.Id, start, end));
}

(IDisposable, IInstrumentProvider, Func<Instrument, Task<IReadOnlyList<Bar>>>, Func<Instrument, Task<IReadOnlyList<FundingRateUpdate>>>) Bybit(BybitProductType t)
{
    BybitHttp h = new(new BybitDataClientConfig { ProductType = t });
    return (h, new BybitInstrumentProvider(h, t), i => BybitHistory.FetchBarsAsync(h, i, candleSeries, start, end), i => BybitHistory.FetchFundingRatesAsync(h, i.Id, start, end));
}

(IDisposable, IInstrumentProvider, Func<Instrument, Task<IReadOnlyList<Bar>>>, Func<Instrument, Task<IReadOnlyList<FundingRateUpdate>>>) Kucoin(KucoinProductType t)
{
    KucoinHttp h = new(new KucoinDataClientConfig { ProductType = t });
    IInstrumentProvider p = t == KucoinProductType.Futures ? new KucoinFuturesInstrumentProvider(h) : new KucoinInstrumentProvider(h);
    return (h, p, i => KucoinHistory.FetchBarsAsync(h, i, candleSeries, start, end), i => KucoinHistory.FetchFundingRatesAsync(h, i.Id, start, end));
}

(IDisposable, IInstrumentProvider, Func<Instrument, Task<IReadOnlyList<Bar>>>, Func<Instrument, Task<IReadOnlyList<FundingRateUpdate>>>) Okx(OkxInstrumentType t)
{
    OkxHttp h = new(new OkxDataClientConfig { InstrumentType = t });
    return (h, new OkxInstrumentProvider(h, t), i => OkxHistory.FetchBarsAsync(h, i, candleSeries, start, end), i => OkxHistory.FetchFundingRatesAsync(h, i.Id, start, end));
}

(IDisposable, IInstrumentProvider, Func<Instrument, Task<IReadOnlyList<Bar>>>, Func<Instrument, Task<IReadOnlyList<FundingRateUpdate>>>) Kraken(KrakenProductType t)
{
    KrakenHttp h = new(new KrakenDataClientConfig { ProductType = t });
    IInstrumentProvider p = t == KrakenProductType.Futures ? new KrakenFuturesInstrumentProvider(h) : new KrakenInstrumentProvider(h);
    return (h, p, i => KrakenHistory.FetchBarsAsync(h, i, candleSeries, start, end), i => KrakenHistory.FetchFundingRatesAsync(h, i.Id, start, end));
}

(IDisposable, IInstrumentProvider, Func<Instrument, Task<IReadOnlyList<Bar>>>, Func<Instrument, Task<IReadOnlyList<FundingRateUpdate>>>) Bitget(BitgetProductType t)
{
    BitgetHttp h = new(new BitgetDataClientConfig { ProductType = t });
    return (h, new BitgetInstrumentProvider(h), i => BitgetHistory.FetchBarsAsync(h, i, candleSeries, start, end), i => BitgetHistory.FetchFundingRatesAsync(h, i.Id, start, end));
}

(IDisposable, IInstrumentProvider, Func<Instrument, Task<IReadOnlyList<Bar>>>, Func<Instrument, Task<IReadOnlyList<FundingRateUpdate>>>) Gate(GateProductType t)
{
    GateHttp h = new(new GateDataClientConfig { ProductType = t });
    IInstrumentProvider p = t switch
    {
        GateProductType.Futures => new GateFuturesInstrumentProvider(h),
        GateProductType.Delivery => new GateDeliveryInstrumentProvider(h),
        _ => new GateInstrumentProvider(h),
    };
    return (h, p, i => GateHistory.FetchBarsAsync(h, i, candleSeries, start, end), i => GateHistory.FetchFundingRatesAsync(h, i.Id, start, end));
}

(IDisposable, IInstrumentProvider, Func<Instrument, Task<IReadOnlyList<Bar>>>, Func<Instrument, Task<IReadOnlyList<FundingRateUpdate>>>) Hyperliquid()
{
    HyperliquidHttp h = new(new HyperliquidDataClientConfig());
    return (h, new HyperliquidInstrumentProvider(h), i => HyperliquidHistory.FetchBarsAsync(h, i, candleSeries, start, end), i => HyperliquidHistory.FetchFundingRatesAsync(h, i.Id, start, end));
}
EOF
```

On Windows replace `$(HOME)` with the clone's absolute path. Then, one call per instrument and bar length:

```bash
dotnet run -- ../catalog spot bx-candle:v2/BINANCE/BTCUSDT/hour/1/last/provider 2025-01-01T00:00:00Z 2025-04-01T00:00:00Z
dotnet run -- ../catalog usdm-futures bx-candle:v2/BINANCE/BTCUSDT-PERP/hour/1/last/provider 2025-01-01T00:00:00Z 2025-04-01T00:00:00Z
bytex catalog info --path ../catalog
```

Tested families and market keys (the venue path component is the factory name):

| Venue | Family argument | Example instrument id |
|---|---|---|
| Binance | `spot`, `usdm-futures`, `coinm-futures` | `bx-market:v2/BINANCE/BTCUSDT`, `bx-market:v2/BINANCE/BTCUSDT-PERP`, `bx-market:v2/BINANCE/BTCUSD_PERP` |
| Bybit | `spot`, `linear`, `inverse` (options have no bar history) | `bx-market:v2/BYBIT/BTCUSDT`, `bx-market:v2/BYBIT/BTCUSDT-PERP`, `bx-market:v2/BYBIT/BTCUSD-PERP` |
| KuCoin | `spot`, `futures` | `bx-market:v2/KUCOIN/BTC-USDT`, `bx-market:v2/KUCOIN/XBTUSDT-PERP` |
| OKX | `spot`, `swap`, `futures` | `bx-market:v2/OKX/BTC-USDT`, `bx-market:v2/OKX/BTC-USDT-SWAP`, `bx-market:v2/OKX/BTC-USD_UM-261030` |
| Kraken | `spot`, `futures` | `bx-market:v2/KRAKEN/BTC-USD`, `bx-market:v2/KRAKEN/PF_XBTUSD` |
| Bitget | `spot`, `usdt-futures`, `usdc-futures` | `bx-market:v2/BITGET/BTCUSDT`, `bx-market:v2/BITGET/BTCUSDT-PERP`, `bx-market:v2/BITGET/BTCUSDC-PERP` |
| Gate | `spot`, `futures`, `delivery` | `bx-market:v2/GATE/BTC_USDT` (spot **and** perpetual), `bx-market:v2/GATE/BTC_USDT_20261009` |
| Hyperliquid | `perpetuals` | `bx-market:v2/HYPERLIQUID/BTC-PERP` |

What the window contains, and where a venue holds less:

- The window is every bar that closes at or after the start, opens at or before the end, and has closed by now: it includes the bar that closes exactly at the start and never the candle still forming. All eight venues return the same window.
- **Kraken spot serves only its newest 720 candles** of a bar length, whatever window is asked (one-minute history older than twelve hours does not exist there). Kraken futures has full history.
- **Hyperliquid serves only about the last 5,000 candles** of a bar length, counted from now.
- KuCoin fills intervals without a trade with flat bars (previous close, zero volume).
- **Gate uses the same id for spot and the perpetual** (`bx-market:v2/GATE/BTC_USDT`): never load both families into one catalog.
- Funding comes at the venue's own cadence: every 8 hours on most venues, hourly on Kraken futures and Hyperliquid. Put it in the backtest with a `funding` data entry (`references/backtest.md`).

## Instruments from the CLI or from JSON

Whole instrument lists (public endpoints, no key) for four venues:

```bash
bytex catalog fetch-instruments --path ./catalog --venue BINANCE --quote USDT                                  # spot
bytex catalog fetch-instruments --path ./catalog --venue BINANCE --instrument-type UsdMFutures                 # or CoinMFutures; --futures means UsdMFutures
bytex catalog fetch-instruments --path ./catalog --venue BYBIT --futures                                      # linear; spot without --futures
bytex catalog fetch-instruments --path ./catalog --venue KUCOIN --futures                                     # perpetuals; spot without --futures
bytex catalog fetch-instruments --path ./catalog --venue OKX --instrument-type Swap                            # Spot | Swap | Futures
```

`--venue` takes `BINANCE`, `BYBIT`, `KUCOIN` or `OKX` only; Bybit inverse and options cannot be fetched this way. For Kraken, Bitget, Gate and Hyperliquid, and for one instrument of any venue, use the program above.

One instrument from JSON (for venues without an adapter, stocks, a synthetic test):

```bash
bytex catalog add-instrument --path ./catalog --file instrument.json
```

`instrument.json` fields (from `src/Bytex.Data/InstrumentJson.cs`; unknown fields are refused): `kind` (`CurrencyPair`, `CryptoPerpetual`, `CryptoFuture`, `Equity`, `FuturesContract`, `OptionContract`), `id`, `rawSymbol`, `assetClass` (`fx` `equity` `commodity` `debt` `index` `crypto` `alternative`), `instrumentClass` (`spot` `swap` `future` `forward` `cfd` `option` `warrant`), `quoteCurrency`, `baseCurrency`, `settlementCurrency`, `isInverse`, `pricePrecision`, `sizePrecision`, `priceIncrement`, `sizeIncrement`, `multiplier`, `lotSize`, `minQuantity`, `maxQuantity`, `minNotional`, `maxNotional`, `minPrice`, `maxPrice`, `marginInit`, `marginMaint`, `maxLeverage`, `marginSource`, `makerFee`, `takerFee` (fractions: `0.001` = 10 bps), `info`, and per kind `underlying`, `activation`, `expiration` (Unix ns), `exchange`, `isin`, `optionKind`, `strikePrice`. Instruments loaded from a venue carry that venue's own margin rates and leverage ceiling; one written by hand carries what you write.

## Bars, quotes and trades from CSV

```bash
bytex catalog import-csv --path ./catalog --file btc-1h.csv --kind bars \
  --instrument bx-market:v2/BINANCE/BTCUSDT --candle-series bx-candle:v2/BINANCE/BTCUSDT/hour/1/last/provider --timestamp-format unix_ms
```

- `--kind`: `bars`, `quotes` or `trades`. The instrument must already be in the catalog. One instrument per file: the loader does not filter by symbol.
- Default columns, matched by header name case-insensitively (extra columns ignored): bars `timestamp,open,high,low,close[,volume]`; quotes `timestamp,bid,ask[,bid_size,ask_size]`; trades `timestamp,price,size[,side,trade_id]` (`side`: `buy`/`sell`/`b`/`s`/`1`/`2`).
- **An exchange's own file can be read as it is**: `--columns` maps fields by header name or by position (`'close=last'`, or `'timestamp=0,open=1,high=2,low=3,close=4,volume=5'`; fields are `timestamp open high low close volume bid ask bidsize asksize price size side tradeid`), `--no-header` for a file whose first line is data (then `--columns` by position is required), `--separator` for one character or `tab`. Quoted fields are honoured.
- `--timestamp-format`: `iso` (default; also accepts a bare integer as Unix nanoseconds), `unix_s`, `unix_ms`, `unix_us`, `unix_ns`, or a .NET format string (read as UTC).
- **Stamp each bar with its close time** before import. Most exchange candle files carry the open time: shift it forward by one bar length first.
- A file whose first line is data, imported without `--no-header`, fails with `CSV is missing column 'timestamp'`: the first row was read as the header. Add `--no-header` with `--columns` by position (tested with a tab-separated exchange file).

## Checking and repairing a catalog

```bash
bytex catalog info --path ./catalog          # data sets with rows, files, size and time range; says whether anything would stop a streaming read
bytex catalog check --path ./catalog         # verifies manifests, hashes, counts and overlapping ranges; exit 1 when it finds problems
bytex catalog consolidate --path ./catalog --kind bars --key bx-candle:v2/BINANCE/BTCUSDT/hour/1/last/provider   # rewrite one data set in order, duplicates dropped, in the fewest files the 100,000-row bound allows
bytex catalog list --path ./catalog          # instruments and data ranges
```

Run `info` after every download and before reporting a backtest: its rows and range are the sample size. Run `consolidate` when `check` reports overlaps (for example after downloading overlapping windows); `--kind` (`quotes`, `trades`, `bars`, `book_deltas`, `book_depth`, `funding`) and `--key` narrow it, without them it rewrites every data set. It reduces the file count to the fewest the bound allows, never to one file for a set larger than 100,000 rows.

## Other sources

- **Tardis.dev** ticks, 25-level book snapshots and book deltas: `references/tardis.md`. The first day of every month is free, with no key: the only real order-book depth available without a subscription.
- **Databento** trades, quotes and bars (equities, futures): `references/databento.md`.
- **No network, quick experiment:** `dotnet run --project examples/Bytex.Examples -c Release -- write-sample-catalog ./catalog 5000` writes synthetic `bx-market:v2/SIM/BTCUSDT` 1-minute bars (candle series `bx-candle:v2/SIM/BTCUSDT/minute/1/last/provider`). Tell the user that results on it say nothing about any market.
