# Connectors

Part of the `bytex-trading-engine` skill. Written against BYTEX 0.11.0; when the engine disagrees, the engine wins.

The repository does not use the word "connector". In this skill, **connectors** means the concrete clients an adapter registers, which a node config wires in, plus the stores a node or a backtest reads and writes. What exists in 0.10.0:

| Connector | Kind | Where it is configured | Credentials |
|---|---|---|---|
| `BINANCE`, `BYBIT`, `KUCOIN`, `OKX`, `KRAKEN`, `BITGET`, `GATE` data | live data + history | `dataClients[]` | none for public data |
| the same seven, execution | real orders | `executionClients[]` | per venue (`references/venues.md`) |
| `HYPERLIQUID` data / execution | perpetuals | both lists | `HYPERLIQUID_PRIVATE_KEY` (+ `HYPERLIQUID_ACCOUNT_ADDRESS`) for execution |
| `TARDIS` data | historical ticks, book snapshots and deltas | program or `dataClients[]` | `TARDIS_API_KEY`; none for the first day of each month |
| `DATABENTO` data | historical trades, quotes, bars | program (`references/databento.md`) | `DATABENTO_API_KEY` |
| `SANDBOX` execution | simulated fills on any venue's live data | `executionClients[]` | none |
| Parquet catalog | historical storage, on disk or `s3://` | `data[].catalogPath`, `bytex catalog` | none, or AWS variables for `s3://` |
| Node store | journal + strategy state + timers on disk | `store` in a node config | none |
| Redis | cache state and bus streams | **code only** | Redis connection string |
| PostgreSQL | cache state (the same keys as Redis) | **code only** | PostgreSQL connection string |

Patterns:

- **Paper:** venue data client + `SANDBOX` execution client with `config.venue` = that venue (`references/sandbox.md`).
- **Live:** venue data client + the same venue's execution client, same family setting, `reconcileOnStart: true`, `tradingRuntime.environment: "live"`. Credentials through `--env-file` (the Safety rules in `SKILL.md`).
- **Two markets on one venue:** two clients with distinct `clientId`s (for example `BINANCE-SPOT`, `BINANCE-FUT`), each with its own family setting.
- **Reconciliation:** `reconcileOnStart` (default true), `reconciliationLookback` (default one day), `reconciliationInterval` or `--reconcile-interval` for periodic checks; `externalOrderClaims` on a strategy claims orders the node finds at the venue. A C# strategy's `QueryOrder(order)` asks the venue about one order and applies the answer, with that order's fills, through the same reconciliation. An execution the venue reports for an order this node never placed is ignored.

**Object storage.** Every catalog path (`--path`, `catalogPath`, `--catalog`) also takes `s3://bucket/prefix`, with `?region=`, `?endpoint=` and `?path-style=true` for S3-compatible services (MinIO and the like). Region, endpoint and credentials otherwise come from the usual AWS variables: `AWS_REGION` / `AWS_DEFAULT_REGION`, `AWS_ENDPOINT_URL_S3` / `AWS_ENDPOINT_URL`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN` (put them in the env file, never in the command). The catalog only: a node store is a local directory. Guide: `docs/concepts/object-storage.md`.

**Redis** (`Bytex.Persistence.Redis`, guide `docs/concepts/redis.md`), two things, neither on by default:

- `RedisCacheDatabase(new RedisCacheConfig { ModuleHostId = "LIVE-001" })` (also `ConnectionString` [`localhost:6379`], `KeyPrefix` [`bytex`], `Database`), passed to `new TradingNode(config, registry, loggerFactory, database)` with `TradingRuntime.Cache = new CacheConfig { Persist = true }`: orders, positions, accounts, instruments, currencies, order lists and runtimeModule state are written as they change and reloaded at start.
- `RedisBusStream(new RedisBusStreamConfig { ModuleHostId = "LIVE-001", Topics = [...] }, bus)`: the node's bus messages published as Redis streams for other programs to read (the shape may change before 1.0).

**PostgreSQL** (`Bytex.Persistence.Postgres`, guide `docs/concepts/postgres.md`) holds the same cache state under the same keys, for a deployment that does not run Redis: `new PostgresCacheDatabase(new PostgresCacheConfig { ConnectionString = "Host=...;Database=bytex;Username=...;Password=...", ModuleHostId = "LIVE-001" })`, passed to `TradingNode` the same way. Four tables (`_kv`, `_set`, `_list`, `_hash`) are created on first use; two nodes can share one database. Put the password in the env file, never in code or chat.

**The CLI constructs none of them**, so `bytex run` cannot use Redis or PostgreSQL, and `loadState`/`saveState` in a CLI config have no Redis behind them. For restartable state from the CLI use `store.directory` (with `saveInterval`, `references/sandbox.md`).

Checking a connector before use: `bytex verify-keys --venue BINANCE|BYBIT|KUCOIN|OKX|KRAKEN|BITGET|GATE|HYPERLIQUID --env-file <file> [--base-url <url>] [--timeout <s>] --json`. It only reads. JSON: `ok`, `venue`, `facts` (`keyAccepted`, `canTrade`, `canWithdraw`, `ipRestricted`, `markets`, `keyExpiresAt`; `null` means the venue did not say), `failure` (`code`: `env_file_missing`, `no_key_in_file`, `venue_unknown`, `bad_key`, `bad_key_or_ip`, `bad_signature`, `key_expired`, `ip_not_allowed`, `clock_skew`, `permission_denied`, `geo_blocked`, `unreachable`, `rate_limited`, `venue_error`, `wrong_environment` (an OKX key made for the other of live and demo), `unknown`), `checks[]`. Exit code 1 when a check failed. There is no `--testnet`.
