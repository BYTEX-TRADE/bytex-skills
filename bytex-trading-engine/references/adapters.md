# Adapters

Part of the `bytex-trading-engine` skill. Written against BYTEX 0.11.0; when the engine disagrees, the engine wins.

An **adapter** (integration) is a plugin that registers up to three components through factories (`docs/design/0005-adapter-sdk.md`, `docs/concepts/plugins.md`, `docs/extending.md`):

| Component | Interface | Role |
|---|---|---|
| Instrument provider | `IInstrumentProvider` | loads instrument definitions (`loadAll`, `loadIds`, `filters`) |
| Data client | `IDataClient` | subscriptions (quotes, trades, bars, book, mark/index/funding) and historical requests |
| Execution client | `IExecutionClient` | orders out, execution events in, reports for reconciliation |

## Configuring a client

A node config names a factory and gives that factory's config as JSON:

```json
"dataClients":      [ { "factory": "<NAME>", "clientId": "<ID>", "config": { ... } } ],
"executionClients": [ { "factory": "<NAME>", "clientId": "<ID>", "config": { ... } } ]
```

Factories the CLI registers: `BINANCE`, `BYBIT`, `KUCOIN`, `OKX`, `KRAKEN`, `BITGET`, `GATE`, `HYPERLIQUID` (data + execution), `TARDIS`, `DATABENTO` (historical data only), `SANDBOX` (execution only). Data routes by explicit `clientId`, else by the instrument's venue, else to a default client. Key fields left `null` are read from the venue's environment variables (`bytex venues` names them).

**A data client and an execution client do not take the same fields**, and a field the client does not have is refused by name. What every client takes:

- Data client: `instrumentProvider` (`loadAll` [false], `loadIds` [], `filters` {}, `logWarnings` [true]) and `handleRevisedBars` [false].
- Execution client: `instrumentProvider`, `leverage` [null] and `brokerId` [null]. No `handleRevisedBars`.

Venue fields on top (from the config records in the 0.10.0 source):

| Factory | Data client | Execution client adds or differs |
|---|---|---|
| `BINANCE` | `accountType`, `apiKey`, `apiSecret`, `baseUrlHttp`, `baseUrlWs`, `useAggTrades` [true] | no `useAggTrades`; adds `defaultTriggerType` [`lastPrice`], `recvWindowMs` |
| `BYBIT` | `productType`, `apiKey`, `apiSecret`, `baseUrlHttp`, `baseUrlWs` | adds `defaultTriggerType`, `recvWindowMs` |
| `KUCOIN` | `productType`, `apiKey`, `apiSecret`, `apiPassphrase`, `apiKeyVersion`, `baseUrlHttp`, `baseUrlWs` | same |
| `OKX` | `instrumentType`, `demoTrading` [false], `apiKey`, `apiSecret`, `apiPassphrase`, `baseUrlHttp`, `baseUrlWs` | adds `marginMode` [`cross`] or `isolated` |
| `KRAKEN` | `productType`, `apiKey`, `apiSecret`, `baseUrlHttp`, `baseUrlWs` | same |
| `BITGET` | `productType`, `tradingMode` [`live`] or `demo`, `apiKey`, `apiSecret`, `apiPassphrase`, `baseUrlHttp`, `baseUrlWs` | adds `marginMode` [`crossed`] or `isolated` |
| `GATE` | `productType`, `apiKey`, `apiSecret`, `baseUrlHttp`, `baseUrlWs` | same |
| `HYPERLIQUID` | `privateKey`, `accountAddress`, `baseUrlHttp`, `baseUrlWs`, `testnet` | adds `crossMargin` [true] |

The family setting and its values per venue: `references/venues.md` or `bytex venues`. There is **no** `testnet` setting anywhere except Hyperliquid's, which only says which network to sign for when `baseUrlHttp` is a proxy the adapter cannot recognise.

**The node checks a client against its venue at start.** A client configured for a family that does not do what the client is for (market data, execution) is refused before anything connects, naming the family. `bytex venues --json` states per family: `loadOneInstrument`, `listInstruments`, `barHistory`, `fundingHistory`, `marketData`, `execution`, `amendOrders`.

## Leverage

`leverage` on an execution client is sent to the exchange before anything trades; left `null`, the account keeps whatever it holds. A figure above the ceiling a venue publishes for an instrument **refuses the run** instead of being lowered. Per venue:

| Venue | What a configured `leverage` does |
|---|---|
| Binance | futures only, set per symbol at connect; whole numbers only (a fraction is refused when the client is built). The ceiling is known only with a key (the venue's brackets are signed) |
| Bybit | linear and inverse, set per symbol at connect; spot and options have none (logged as unsent) |
| KuCoin | futures: sent on every order, **1 when not set**; spot has none |
| OKX | set per instrument at connect with the configured `marginMode` |
| Kraken | futures: sets a per-symbol preference that **switches the symbol to isolated margin** (the log says so); spot: ignored |
| Bitget | set on the instruments named in `instrumentProvider.loadIds` (or everything loaded when none are named), per side in isolated mode |
| Gate | set per contract; `0` (cross margin on Gate) is refused |
| Hyperliquid | set per asset together with `crossMargin`; whole numbers only |

Name the instruments with `loadIds` whenever `leverage` is set: it is set on what is loaded.

## Broker id

`brokerId` on an execution client tags orders for the venue's broker programme. Carried on Binance (a prefix on the client order id, stripped from everything the venue says back; the whole id must stay within 36 characters), Bybit (`X-Referer` header), Bitget (`X-CHANNEL-API-CODE` header) and Kraken (the order's `broker` field). Ignored on KuCoin, OKX, Gate and Hyperliquid. Set it only when the user has a broker code.

## Writing a new adapter

A new project with an instrument provider, a data client and an execution client (derive from `InstrumentProviderBase`, `DataClientBase`, `ExecutionClientBase` in `Bytex.Core.Adapters`), two config records deriving from `DataClientConfig` / `ExecutionClientConfig`, two factories (`IDataClientFactory`, `IExecutionClientFactory` with a `Name` and `ConfigType`), and an `IPlugin` that registers them. Implement `IVenuePlugin.Describe()` to return a `VenueDescriptor`: the venue, its broker tag and programme, and per family the instrument classes, funding, collateral, hosts, key variables, selecting config, default fees, free datasets and capabilities; that is what `bytex venues` prints and what the node checks clients against. Public static `<Venue>History.FetchBarsAsync` / `FetchFundingRatesAsync` helpers let data be stored without a node.

Network helpers are in `Bytex.Live.Network` (`WebSocketClient`, `HttpClientWrapper`, `RateLimiter`, `RetryPolicy`, `HmacSigner`). Follow the responsibilities checklist in design note 0005: symbol mapping both ways, correct increments and fees, reject unsupported order types before sending, venue timestamps into `EventTime`, errors translated into rejections, local rate limiting, resubscription on reconnect, reports from REST alone. Use `src/Bytex.Adapters.Bybit` as the model and `src/Bytex.Adapters.Tardis` or `src/Bytex.Adapters.Databento` for a data-only provider. Load it with `bytex --plugins <dir>` (one subfolder per plugin, containing only its own assembly and its non-BYTEX dependencies) or `pluginDirectory` in the node config.
