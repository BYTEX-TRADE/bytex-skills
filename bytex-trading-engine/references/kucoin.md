# KuCoin

Part of the `bytex-trading-engine` skill. Written against BYTEX 0.11.0; when the engine disagrees, the engine wins.

Package `Bytex.Adapters.Kucoin`, plugin `KucoinPlugin`, factory `KUCOIN`. Guide: `docs/integrations/kucoin.md`.

| `productType` | Market | Instrument ids | Account |
|---|---|---|---|
| `Spot` (default) | spot, through the high-frequency (HF) endpoints | `bx-market:v2/KUCOIN/BTC-USDT` (the venue's `BASE-QUOTE`) | cash |
| `Futures` | USDT/USDC perpetuals | `bx-market:v2/KUCOIN/XBTUSDT-PERP` (the venue's `XBTUSDTM`; `XBT` is its name for bitcoin) | margin |

Inverse contracts and dated futures are not offered. The two markets are separate APIs on separate hosts that share a key; the key's spot and futures permissions are separate.

```json
{ "factory": "KUCOIN", "clientId": "KUCOIN", "config": {
    "productType": "Spot",
    "apiKey": null, "apiSecret": null, "apiPassphrase": null,
    "apiKeyVersion": null,
    "instrumentProvider": { "loadIds": ["bx-market:v2/KUCOIN/BTC-USDT"] }
} }
```

Data and execution clients take these fields; a data client may add `handleRevisedBars`, an execution client `leverage` and `brokerId` (each refused on the other).

**Credentials:** `KUCOIN_API_KEY`, `KUCOIN_API_SECRET`, `KUCOIN_API_PASSPHRASE`, and optionally the key version from `apiKeyVersion` or `KUCOIN_API_KEY_VERSION`: with none set, a client signs as version 3 and retries once as version 2. `bytex verify-keys --venue KUCOIN` finds the version and names it. **KuCoin has no testnet**: paper trade with the SANDBOX execution client on KuCoin data (`examples/configs/sandbox-kucoin-ema-cross.json`).

**Spot data:** the stream address and token come from `bullet-public` before every connection. Quotes `/spotMarket/level1`, trades `/market/match`, bars `/market/candles`, a 50-level book `/spotMarket/level2Depth50`. KuCoin never marks a candle closed and sends nothing for a quiet interval: the client publishes a bar when the next candle starts or two seconds after its interval ends, and builds flat bars (previous close, zero volume) for intervals without trades. History: bars (gaps filled with the same flat bars), the last 100 trades. Bar intervals: 1, 3, 5, 15, 30 minutes; 1, 2, 4, 6, 8, 12 hours; day, week, month.

**Spot execution:** market (`size`, or `funds` for quote quantity), limit (GTC/IOC/FOK, GTD via `GTT` + `cancelAfter`, post-only), stop orders via `stop-order`. **Quote quantity only on a MARKET order**; a quote-quantity limit is refused by name. A modified stop is cancel-then-replace with a new client id `<id>-r1`, `-r2`, ...: for one request the position has **no stop at the venue**. Plain orders modify through `hf/orders/alter`. `clientOid` is limited to 40 characters. Commissions are computed from the instrument's maker/taker rate.

**Futures:**

- Orders are whole numbers of **contracts** (one `XBTUSDTM` is 0.001 XBT); the instrument publishes its size increment in base currency and the adapter converts. A quantity that is not a whole number of contracts is refused, and so is a quote quantity.
- **Leverage is sent on every order and is 1 when `leverage` is not set.**
- **Orders cannot be amended at all** (`amendOrders: false`): a modification is refused, not turned into cancel-and-replace. Documents with `act.modify`, `act.moveStop`, `act.trail` or a trailing `risk.exit` cannot manage their orders here.
- Market and limit, GTC and IOC, post-only, reduce-only; stops against the **mark price**.
- History: bars (oldest 200 rows per page, walked forwards), funding every 8 hours. Bar lengths 1, 3, 5, 15, 30 minutes; 1, 2, 4, 8, 12 hours; day, week.

**Broker id:** ignored (`brokerId` is not refused; nothing is carried).

**Status:** the trading side was built from KuCoin's published API and tested against recorded payloads; `docs/integrations/kucoin.md` lists the points still to confirm against a real account. Tell the user this before any live KuCoin run.
