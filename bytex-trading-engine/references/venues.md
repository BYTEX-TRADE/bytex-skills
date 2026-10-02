# Venues

Part of the `bytex-trading-engine` skill. Written against BYTEX 0.11.0; when the engine disagrees, the engine wins.

## Contents

- All eight venues at a glance
- OKX
- Kraken
- Bitget
- Gate
- Hyperliquid

`bytex venues` (or `bytex venues --json`) prints every venue's families, the setting that selects each, key variables, hosts, default fees, free datasets and capabilities, read from the adapters themselves. Use it before writing a client config. The venue guides are `docs/integrations/<venue>.md`.

## All eight venues at a glance

| Venue (factory) | Families and the setting that selects them | Key variables | Example ids | Orders | Amend |
|---|---|---|---|---|---|
| `BINANCE` | `accountType`: `spot`, `usdMFutures`, `coinMFutures` | `BINANCE_API_KEY`, `_SECRET` | `BTCUSDT`, `BTCUSDT-PERP`, `BTCUSD_PERP` | market, limit, stops, if-touched; trailing on futures | yes |
| `BYBIT` | `productType`: `spot`, `linear`, `inverse`, `option` | `BYBIT_API_KEY`, `_SECRET` | `BTCUSDT`, `BTCUSDT-PERP`, `BTCUSD-PERP` | market, limit, stops, if-touched | yes |
| `KUCOIN` | `productType`: `Spot`, `Futures` | `KUCOIN_API_KEY`, `_SECRET`, `_PASSPHRASE` (+ optional `_KEY_VERSION`) | `BTC-USDT`, `XBTUSDT-PERP` | market, limit, stops, if-touched | spot yes, futures **no** |
| `OKX` | `instrumentType`: `spot`, `swap`, `futures` | `OKX_API_KEY`, `_SECRET`, `_PASSPHRASE` | `BTC-USDT`, `BTC-USDT-SWAP`, `BTC-USD_UM-261030` | market, limit only | yes |
| `KRAKEN` | `productType`: `Spot`, `Futures` | `KRAKEN_API_KEY`, `_SECRET` (a different key per platform) | `BTC-USD`, `PF_XBTUSD`, `FF_XBTUSD_261225` | market, limit, stops, if-touched | yes |
| `BITGET` | `productType`: `Spot`, `UsdtFutures`, `UsdcFutures` | `BITGET_API_KEY`, `_SECRET`, `_PASSPHRASE` | `BTCUSDT`, `BTCUSDT-PERP`, `BTCUSDC-PERP` | market, limit only | yes |
| `GATE` | `productType`: `Spot`, `Futures`, `Delivery` | `GATE_API_KEY`, `_SECRET` | `BTC_USDT` (spot **and** perpetual), `BTC_USDT_20261009` | market, limit only | spot and perpetual yes, delivery **no** |
| `HYPERLIQUID` | perpetuals only (no setting) | `HYPERLIQUID_PRIVATE_KEY` (+ optional `HYPERLIQUID_ACCOUNT_ADDRESS`) | `BTC-PERP` | limit, stops, if-touched; market is an IOC limit 5 % through the touch | yes |

Ids are shown without their venue suffix: add `.<FACTORY>`, e.g. `bx-market:v2/OKX/BTC-USDT-SWAP`. Enum settings are read case-insensitively (`spot` and `Spot` both work).

Common to all of them:

- **No venue testnets** in BYTEX: rehearse with the SANDBOX execution client on the venue's live data. OKX (`demoTrading: true`) and Bitget futures (`tradingMode: "demo"`) have the venue's own demo accounts.
- **Order lists** (brackets, OTO): the entry is sent, the exits are held in the node until it fills.
- **Quote-quantity orders** are taken only where the venue can express them (Binance, Bybit and KuCoin spot market orders, Bitget spot market buys, OKX spot) and refused by name elsewhere.
- **Silent time-in-force downgrades**: several venues send FOK, GTD or DAY as GTC (Binance spot GTD, Bybit, OKX, Kraken spot FOK and DAY, Bitget GTD, Hyperliquid FOK/GTD/DAY; KuCoin futures FOK/GTD). A strategy that depends on an order expiring must not rely on the venue to expire it.
- **Triggered orders** (stops, if-touched) are refused on OKX, Bitget and Gate: stop-based exits cannot rest there.
- **History depth**: Kraken spot keeps only its newest 720 candles per bar length, Hyperliquid about the last 5,000; Bybit options have no bar history.
- `catalog fetch-instruments` covers Binance, Bybit (spot, linear), KuCoin and OKX; the program in `references/data.md` covers every venue.
- Leverage and broker id per venue: `references/adapters.md`.

## OKX

Factory `OKX`, guide `docs/integrations/okx.md`. Families by `instrumentType`: `spot` (`bx-market:v2/OKX/BTC-USDT`), `swap` (linear perpetuals, `bx-market:v2/OKX/BTC-USDT-SWAP`), `futures` (linear dated, `bx-market:v2/OKX/BTC-USD_UM-261030`; OKX has no USDT-margined dated futures). Inverse contracts and options are not offered.

```json
{ "factory": "OKX", "clientId": "OKX", "config": {
    "instrumentType": "swap",
    "apiKey": null, "apiSecret": null, "apiPassphrase": null,
    "instrumentProvider": { "loadIds": ["bx-market:v2/OKX/BTC-USDT-SWAP"] }
} }
```

- Execution client adds `marginMode` (`cross` default, or `isolated`), `leverage`, `brokerId` (ignored). `demoTrading: true` selects the demo environment (same REST host with a header; set `baseUrlWs` to `wss://wspap.okx.com:8443` for its socket).
- **Client order ids must be letters and digits only, at most 32 characters.** The engine's default id has hyphens, so every order would be refused. For a document strategy, put `"useHyphensInClientOrderIds": false` in its payload (with `documentPath` or an inline `document`) or in the strategy entry's `config` (tested: ids come out as `O202609200203000010011`).
- Market and limit orders only; **triggered orders are refused**. Amend via `amend-order`. The client trades net mode (no `posSide`): an account in long/short mode has its orders refused, and `verify-keys` warns about it.
- Margin rates and the leverage ceiling are the venue's own position tiers (1 % / 0.4 % and 100x on `BTC-USDT-SWAP`); a configured leverage above the ceiling refuses the run.
- History: bars from `history-candles` (full depth), funding is the **realized** rate (about 95 days kept). Granularities include `1s`, `1m` ... `12H`, `1D`, `1W`, `1M`; `8H` is refused. Derivative volume is converted from contracts to base currency.
- Not verified without a key (from the guide): order placement, `tgtCcy` on spot market orders (the adapter sends `base_ccy`), the fee sign, private socket payloads, demo trading on private endpoints.

## Kraken

Factory `KRAKEN`, guide `docs/integrations/kraken.md`. Families by `productType`: `Spot` (`bx-market:v2/KRAKEN/BTC-USD`: the venue's accepted `BASE/QUOTE` with a dash; bitcoin is `BTC` on spot) and `Futures` (perpetuals `bx-market:v2/KRAKEN/PF_XBTUSD`, dated `bx-market:v2/KRAKEN/FF_XBTUSD_261225`; bitcoin is `XBT` in futures symbols).

```json
{ "factory": "KRAKEN", "clientId": "KRAKEN", "config": {
    "productType": "Spot",
    "apiKey": null, "apiSecret": null,
    "instrumentProvider": { "loadIds": ["bx-market:v2/KRAKEN/BTC-USD"] }
} }
```

- **Spot and futures need different keys** in the same two variables (`KRAKEN_API_KEY`, `KRAKEN_API_SECRET`). A node running both families supplies one of them through the client config. `verify-keys --venue KRAKEN` tries both platforms and says which took the key; Kraken publishes no key permissions, so `canWithdraw` and IP facts come back `null`.
- No test environment on either platform: use the SANDBOX execution client.
- Spot orders: market, limit, stop-loss, stop-loss-limit, take-profit, take-profit-limit; post-only, IOC; real amend; quote quantity refused; cancel-all is done per order so other instruments are untouched. `leverage` is ignored on spot.
- Futures orders: `mkt`, `lmt`, `post`, `ioc`, `stp`, `take_profit`; stops trigger on the **mark price**; amend supported. Setting `leverage` puts each symbol into **isolated margin** (the log says so).
- Futures margin is a tier schedule; the instrument carries the first tier (1 % / 0.5 % on `PF_XBTUSD`, 100x).
- History: **spot only its newest 720 candles** per bar length (one-minute spot history older than 12 hours does not exist there); futures bars and funding (hourly) in full.

## Bitget

Factory `BITGET`, guide `docs/integrations/bitget.md`. Families by `productType`: `Spot` (`bx-market:v2/BITGET/BTCUSDT`), `UsdtFutures` (`bx-market:v2/BITGET/BTCUSDT-PERP`), `UsdcFutures` (`bx-market:v2/BITGET/BTCUSDC-PERP`, the venue's `BTCPERP`). Coin-margined futures are not offered.

```json
{ "factory": "BITGET", "clientId": "BITGET", "config": {
    "productType": "UsdtFutures",
    "apiKey": null, "apiSecret": null, "apiPassphrase": null,
    "instrumentProvider": { "loadIds": ["bx-market:v2/BITGET/BTCUSDT-PERP"] }
} }
```

- Execution client adds `marginMode` (`crossed` default, or `isolated`), `leverage`, `brokerId` (the `X-CHANNEL-API-CODE` header). `tradingMode: "demo"` selects the venue's demo futures; there is no demo spot (refused when the client is built).
- Market and limit orders only; **trigger orders are refused**. **A spot market buy is refused unless it carries a quote quantity** (the venue sizes a spot market buy in quote currency), so a document's market buy on Bitget spot is refused; use a limit order or a futures family. Futures amend in place; spot amends by cancel-replace in one request.
- Margin and ceiling from per-contract tiers (one request per contract: listing a whole futures family costs about 805 requests; name instruments with `loadIds`). Leverage is set on the named instruments.
- History: bars, funding; recent trades capped at 100.

## Gate

Factory `GATE`, guide `docs/integrations/gate.md`. Families by `productType`: `Spot`, `Futures` (USDT perpetuals), `Delivery` (USDT dated). Ids are the venue's own `BASE_QUOTE` with an underscore: **`bx-market:v2/GATE/BTC_USDT` is both the spot pair and the perpetual**, so keep the two families in separate catalogs and nodes. Dated: `bx-market:v2/GATE/BTC_USDT_20261009`.

```json
{ "factory": "GATE", "clientId": "GATE", "config": {
    "productType": "Spot",
    "apiKey": null, "apiSecret": null,
    "instrumentProvider": { "loadIds": ["bx-market:v2/GATE/BTC_USDT"] }
} }
```

- A Gate key has two parts only. Market and limit orders only (a derivative market order is an IOC limit at price 0); **triggered orders are refused**; post-only is `poc`; reduce-only on derivatives. Quote quantity refused.
- Derivative orders are whole contracts (0.0001 BTC on `BTC_USDT`); the adapter converts from base units and refuses a quantity that is not a whole number of contracts.
- **Delivery cannot amend** (refused). Spot and perpetual amend in place.
- `leverage` is set per contract; `0` (cross margin on Gate) is refused.
- The client order id travels in the order's `text`, prefixed `t-`, at most 28 characters after it; an id the venue would refuse is refused before sending.

## Hyperliquid

Factory `HYPERLIQUID`, guide `docs/integrations/hyperliquid.md`. Perpetuals only: `bx-market:v2/HYPERLIQUID/BTC-PERP`. Spot, builder-deployed markets and vaults are not covered.

```json
{ "factory": "HYPERLIQUID", "clientId": "HYPERLIQUID", "config": {
    "privateKey": null, "accountAddress": null,
    "leverage": null, "crossMargin": true,
    "instrumentProvider": { "loadIds": ["bx-market:v2/HYPERLIQUID/BTC-PERP"] }
} }
```

That is an execution client (`leverage`, `crossMargin`); a data client takes neither.

- **The credential is a wallet private key**, `HYPERLIQUID_PRIVATE_KEY`, not an API key. **Use an API wallet** the account has approved, never the account's own wallet key (it can move funds and nothing restricts it); set `HYPERLIQUID_ACCOUNT_ADDRESS` to the account when the API wallet signs for it, or every read returns an empty account. `verify-keys --venue HYPERLIQUID` warns when it holds a wallet key.
- **There is no market order**: a market order is an IOC limit placed 5 % through the opposite touch. TIF: GTC, IOC, post-only (`Alo`); FOK is not accepted. Amend replaces the order.
- `leverage` is a whole number, set per asset together with `crossMargin` (default true); an asset whose own maximum is below it is left alone with a warning.
- `testnet` exists here only to say which network to sign for when `baseUrlHttp` is a proxy the adapter cannot recognise; the network is otherwise taken from the host.
- History: bars only for about the **last 5,000 candles** of a bar length counted from now (daily bars back to 2020); funding is complete (hourly).
