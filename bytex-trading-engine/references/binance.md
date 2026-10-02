# Binance

Part of the `bytex-trading-engine` skill. Written against BYTEX 0.11.0; when the engine disagrees, the engine wins.

Package `Bytex.Adapters.Binance`, plugin `BinancePlugin`, factory `BINANCE`. Guide: `docs/integrations/binance.md`.

| `accountType` | Market | Instrument ids | Account |
|---|---|---|---|
| `spot` | spot | `bx-market:v2/BINANCE/BTCUSDT` | cash |
| `usdMFutures` | USDⓈ-margined perpetuals and dated futures | `bx-market:v2/BINANCE/BTCUSDT-PERP`, `bx-market:v2/BINANCE/BTCUSDT_250926` | margin |
| `coinMFutures` | coin-margined (inverse) perpetuals and dated futures | `bx-market:v2/BINANCE/BTCUSD_PERP`, `bx-market:v2/BINANCE/BTCUSD_261225` | margin |

One client serves one account type. The venue is `BINANCE` for all three; a node using more than one market must route with explicit `clientId`s. Margin trading and options are not covered.

**Coin-margined contracts are inverse**: quoted in USD, sized in whole USD contracts (100 USD on BTCUSD, 10 USD on the others), margined and settled in the base coin, so profit, margin, fees and funding come out in the coin. 200 contracts of `BTCUSD_PERP` at 50,000 is 0.4 BTC.

**Data client config:**

```json
{ "factory": "BINANCE", "clientId": "BINANCE", "config": {
    "accountType": "spot",
    "apiKey": null, "apiSecret": null,
    "baseUrlHttp": null, "baseUrlWs": null,
    "instrumentProvider": { "loadAll": false, "loadIds": ["bx-market:v2/BINANCE/BTCUSDT"], "filters": { "quote": "USDT" } },
    "handleRevisedBars": false,
    "useAggTrades": true
} }
```

The data client needs no credentials. The **execution client** takes `accountType`, keys, URLs and `instrumentProvider`, plus `defaultTriggerType` (`lastPrice` or `markPrice`, futures), `recvWindowMs`, `leverage` and `brokerId`; it does **not** take `handleRevisedBars` or `useAggTrades` (refused by name).

**Credentials:** `apiKey`/`apiSecret` left `null` are read from `BINANCE_API_KEY` / `BINANCE_API_SECRET`, the same pair for all three account types. Keep them in an env file (the Safety rules in `SKILL.md`), never in the config. There is no testnet setting.

**Instrument provider:** `loadIds` loads exactly those instruments (use this in nodes), `loadAll` loads everything, `filters.quote` narrows by quote currency. Futures instruments carry the venue-wide margin default (5 % / 2.5 %) unless a key is held, because the real per-notional brackets are signed reads: `result.json` then reports `marginSource: venueWideDefault`, and the leverage ceiling is unknown, so nothing is refused on a guess.

**Hosts:** spot `api.binance.com` / `stream.binance.com:9443`; USDⓈ-M `fapi.binance.com` / `fstream.binance.com`; COIN-M `dapi.binance.com` / `dstream.binance.com`. `baseUrlHttp`/`baseUrlWs` override them.

**Data:** quotes from `bookTicker`, trades from `trade`, bars from `kline_*` (closed bars only unless `handleRevisedBars`), book deltas from `depth@100ms` with a REST snapshot, mark/index price and funding from `markPrice@1s` (futures). History: bars (1,000 per page on spot, 1,500 on futures), trades (aggregated by default), funding rates (futures); no quote history. Bar intervals: 1s, 1m, 3m, 5m, 15m, 30m, 1h, 2h, 4h, 6h, 8h, 12h, 1d, 3d, 1w, 1M.

**Execution:**

| BYTEX order | Spot | Futures (both families) |
|---|---|---|
| Market | `MARKET` (quote quantity supported) | `MARKET` |
| Limit | `LIMIT`, `LIMIT_MAKER` for post-only, iceberg via `icebergQty` | `LIMIT`, `GTX` for post-only |
| StopMarket | `STOP_LOSS` | `STOP_MARKET` |
| StopLimit | `STOP_LOSS_LIMIT` | `STOP` |
| MarketIfTouched | `TAKE_PROFIT` | `TAKE_PROFIT_MARKET` |
| LimitIfTouched | `TAKE_PROFIT_LIMIT` | `TAKE_PROFIT` |
| TrailingStopMarket | refused | `TRAILING_STOP_MARKET` |

- **An order sized in the quote currency is taken only as a spot MARKET order**; any other quote-quantity order is refused by name ("Binance takes an order sized in the quote currency only on a spot MARKET order ...").
- **A trailing offset must be in basis points** (`TrailingOffsetType.BasisPoints`); any other offset type is refused rather than sent in the wrong unit.
- TIF: GTC, IOC, FOK; GTD on futures. `day`, and GTD on spot, are sent as GTC, so they do not expire at the venue. `reduceOnly` on futures only; on spot, exits are plain sells.
- Brackets and OTO lists: the entry is sent, the exits are held in the node until it fills and dropped if it is rejected, cancelled or expires.
- Modify: `PUT` on each futures family's own order path; cancel-replace on spot (limit orders only).
- Events from the user data stream; the listen key is refreshed every 30 minutes. Client order ids must be at most 36 characters, broker prefix included, so keep moduleHost and strategy ids short.
- `leverage` (futures): set per symbol at connect, whole numbers only. `brokerId`: prefixes the client order id (`references/adapters.md`).

**Reconciliation:** open orders, recent trades per instrument and (futures) position risk; a strategy's `QueryOrder` asks after one order and applies the answer.

Check a key before any live use: `bytex verify-keys --venue BINANCE --env-file keys/binance.env --json`. It reads the key's restrictions, and when futures are enabled it also checks the coin-margined host (a warning, not a failure, when that fails).
