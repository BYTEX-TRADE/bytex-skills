# Databento

Part of the `bytex-trading-engine` skill. Written against BYTEX 0.11.0; when the engine disagrees, the engine wins.

Package `Bytex.Adapters.Databento`, plugin `DatabentoPlugin`, factory `DATABENTO` (`docs/integrations/databento.md`). Historical trades, top-of-book quotes (`mbp-1`) and bars from Databento's historical service, for equities, futures and the other markets it covers. **Data only**: no execution, no live data, no instrument list, no book beyond the top.

- **Key:** `DATABENTO_API_KEY` (in the env file; the helper reads the environment itself).
- **Dataset is required**: `GLBX.MDP3` (CME futures), `XNAS.ITCH` (Nasdaq) and so on. Asking the wrong one is answered with an entitlement error rather than a hint.
- **Bars only as the vendor publishes them**: 1 second, 1 minute, 1 hour, 1 day (`ohlcv-1s`, `ohlcv-1m`, `ohlcv-1h`, `ohlcv-1d`). A five-minute bar is refused by name; build longer bars in the document (`computed` bars from ticks) or pick one of the four.
- Prices arrive in billionths and the adapter scales them; closed bars only, stamped at the close.
- **There is no CLI command** that fetches Databento data, and a node's history requests to `DATABENTO` return nothing: fetch into a catalog once, then backtest from the catalog.

## Into a catalog

1. **Add the instrument.** Databento lists none, so write it (`references/data.md`, "Instruments from the CLI or from JSON"). The venue code is yours to choose; `DATABENTO` keeps it obvious, and the run config's `venues[].venue` must use the same code:

```json
{
  "kind": "FuturesContract", "id": "bx-market:v2/DATABENTO/ESZ6", "rawSymbol": "ESZ6",
  "assetClass": "index", "instrumentClass": "future",
  "quoteCurrency": "USD", "settlementCurrency": "USD",
  "pricePrecision": 2, "sizePrecision": 0, "priceIncrement": "0.25", "sizeIncrement": "1",
  "multiplier": "50", "underlying": "ES", "activation": 0, "expiration": 1797984000000000000,
  "marginInit": "0.05", "marginMaint": "0.05", "makerFee": 0, "takerFee": 0
}
```

```bash
bytex catalog add-instrument --path ./catalog --file esz6.json
```

For an equity use `"kind": "Equity"`, `"assetClass": "equity"`, `"instrumentClass": "spot"`, a `cash` account, and the stock's own tick and lot.

2. **Fetch the bars** with a small program over the engine's `DatabentoHistory` helper (it compiles against 0.10.0 and reaches Databento; running it needs a key with access to the dataset):

```bash
mkdir databento-load && cd databento-load
cat > databento-load.csproj <<'EOF'
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup><OutputType>Exe</OutputType><TargetFramework>net10.0</TargetFramework><ImplicitUsings>enable</ImplicitUsings><Nullable>enable</Nullable></PropertyGroup>
  <ItemGroup>
    <ProjectReference Include="$(HOME)/.bytex/engine/src/Bytex.Adapters.Databento/Bytex.Adapters.Databento.csproj" />
    <ProjectReference Include="$(HOME)/.bytex/engine/src/Bytex.Data/Bytex.Data.csproj" />
  </ItemGroup>
</Project>
EOF
cat > Program.cs <<'EOF'
using Bytex.Adapters.Databento;
using Bytex.Core.Model.Data;
using Bytex.Core.Model.Instruments;
using Bytex.Core.Model.Primitives;
using Bytex.Data;

// dotnet run -- <catalog> <candleSeries> <symbol> <dataset> <startIso> <endIso>
// e.g. dotnet run -- ../catalog bx-candle:v2/DATABENTO/ESZ6/minute/1/last/provider ESZ6 GLBX.MDP3 2026-09-01T00:00:00Z 2026-09-02T00:00:00Z
// Reads DATABENTO_API_KEY from the environment. Bars only: 1 second, 1 minute, 1 hour or 1 day.
MarketArchive catalog = new(args[0]);
CandleSeries candleSeries = CandleSeries.Parse(args[1]);
Instrument instrument = catalog.Instrument(candleSeries.MarketKey)
    ?? throw new InvalidOperationException($"{candleSeries.MarketKey} is not in the catalog: add it with bytex catalog add-instrument");
IReadOnlyList<Bar> bars = await DatabentoHistory.FetchBarsAsync(
    instrument, candleSeries, symbol: args[2], dataset: args[3],
    start: UnixNanos.Parse(args[4]), end: UnixNanos.Parse(args[5]));
await catalog.WriteBarsAsync(bars);
Console.WriteLine($"Wrote {bars.Count} bars of {candleSeries}");
EOF
```

On Windows replace `$(HOME)` with the clone's absolute path. Run it with the key in the environment for that one command (from the env file, never typed into chat). The `symbol` is Databento's raw symbol (`ESZ6`), not the engine id.

3. **Validate and run**: `bytex documents validate --document documents/strategy.json --catalog ./catalog`, and in the run config `"venues": [{ "venue": "DATABENTO", "accountType": "margin", "startingBalances": ["100000 USD"] }]` (a `cash` account for equities). Set `sessionEndUtc` on the venue so `Day` orders expire at the market's close rather than at UTC midnight.

Trades and quotes: `DatabentoDataClient.LoadTradesAsync` / `LoadQuotesAsync(marketKey, symbol, start, end, ct)` in a program built like the one in `references/tardis.md`, then `catalog.WriteAsync(data)`. CSV is the other route: a Databento CSV export (with `pretty_px` and `pretty_ts`) imports with `bytex catalog import-csv --columns 'timestamp=ts_event,...'`, but `ts_event` is the bar's **open**, so shift it to the close first (`references/data.md`).

Limits of this vendor here: historical only, top of book only, and no exchange calendar beyond `sessionEndUtc`.
