# Bybit

Part of the `bytex-trading-engine` skill. Written against BYTEX 0.11.0; when the engine disagrees, the engine wins.

Package `Bytex.Adapters.Bybit`, plugin `BybitPlugin`, factory `BYBIT`, V5 API, unified account. Guide: `docs/integrations/bybit.md`.

| `productType` | Market | Instrument ids | Account |
|---|---|---|---|
| `spot` | spot | `bx-market:v2/BYBIT/BTCUSDT` | cash |
| `linear` | USDT/USDC perpetuals and futures | `bx-market:v2/BYBIT/BTCUSDT-PERP`, `bx-market:v2/BYBIT/BTCUSDT-16OCT26` | margin |
| `inverse` | coin-margined perpetuals and dated futures | `bx-market:v2/BYBIT/BTCUSD-PERP`, `bx-market:v2/BYBIT/BTCUSDZ26` | margin |
| `option` | USDT-settled options | `bx-market:v2/BYBIT/BTC-25JUN27-106000-P-USDT` | margin |

The `-PERP` suffix matters: spot also lists `BTCUSD`, so without it one id would mean two instruments. Whether a contract is a perpetual or dated comes from the venue's `contractType`, never the name.

```json
{ "factory": "BYBIT", "clientId": "BYBIT", "config": {
    "productType": "linear",
    "apiKey": null, "apiSecret": null,
    "instrumentProvider": { "loadIds": ["bx-market:v2/BYBIT/BTCUSDT-PERP"] },
    "defaultTriggerType": "markPrice", "recvWindowMs": 5000
} }
```

That is an **execution** client (`defaultTriggerType`, `recvWindowMs`, and `leverage`, `brokerId` are execution-only). A data client takes `productType`, keys, URLs, `instrumentProvider` and `handleRevisedBars`. Credentials `BYBIT_API_KEY` / `BYBIT_API_SECRET`. There is no testnet setting: rehearse with the SANDBOX execution client on Bybit data (`references/sandbox.md`).

**Data:** quotes from `orderbook.1`, trades from `publicTrade`, bars from `kline.*` (confirmed only unless `handleRevisedBars`), book deltas from `orderbook.50`/`orderbook.200`, mark/index price and funding from `tickers`. History: bars (paged), recent trades, funding (linear and inverse). Bar intervals: 1, 3, 5, 15, 30, 60, 120, 240, 360, 720 minutes, day, week, month.

**Options differ on every point:** no funding, **no bar history by any route**, no published margin or leverage ceiling, book depths 25 and 100, trades per underlying, quotes from `tickers`. Listing without a filter returns one underlying: set `instrumentProvider.filters.baseCoin` to a comma list to list more. `catalog fetch-instruments` cannot fetch inverse or options; load them in a node or with the program in `references/data.md` (inverse only).

**Execution:**

| BYTEX order | Bybit |
|---|---|
| Market | `Market` (quote quantity only on spot, via `marketUnit`) |
| Limit | `Limit` with `GTC`/`IOC`/`FOK`/`PostOnly` |
| StopMarket / MarketIfTouched | conditional `Market` with `triggerPrice`, `triggerDirection`, `triggerBy` |
| StopLimit / LimitIfTouched | conditional `Limit` |

- No trailing stops (refused). **Quote quantity only on a spot MARKET order**; any other quote-quantity order is refused by name.
- Any time in force other than IOC and FOK is sent as GTC (or PostOnly), so `day` and GTD orders do not expire at the venue.
- `reduceOnly` on linear and inverse; left off option orders. Amend via `order/amend` for every family (whether the venue honours an option amend is unverified). Brackets: exits held in the node until the entry fills.
- `leverage`: set per symbol on linear and inverse at connect; spot and options have none. `brokerId`: `X-Referer` header, outside the signature.
- Only real trades become fills; funding, delivery and settlement rows on the execution topic are ignored.
