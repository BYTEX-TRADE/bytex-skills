# Sandbox (paper trading)

Part of the `bytex-trading-engine` skill. Written against BYTEX 0.11.0; when the engine disagrees, the engine wins.

The sandbox (factory `SANDBOX`, in `Bytex.Live`; guide `docs/integrations/sandbox.md`) pairs **live market data** with **simulated fills** from the same venue simulation the backtester uses. Nothing reaches the exchange; no keys are needed for public data. It works with the data client of any of the eight venues, and it is the only rehearsal BYTEX offers: there are no venue testnets (`references/venues.md`).

Paper node config (a document strategy on Binance spot data; verified to run):

```json
{
  "tradingRuntime": { "moduleHostId": "PAPER-001", "environment": "sandbox", "loadState": false, "saveState": false },
  "dataClients": [
    { "factory": "BINANCE", "clientId": "BINANCE",
      "config": { "accountType": "spot", "instrumentProvider": { "loadIds": ["bx-market:v2/BINANCE/BTCUSDT"] } } }
  ],
  "executionClients": [
    { "factory": "SANDBOX", "clientId": "BINANCE-SANDBOX",
      "config": { "venue": "BINANCE", "accountType": "cash", "startingBalances": ["10000 USDT"] } }
  ],
  "strategies": [
    { "providerId": "bytex.document",
      "payload": { "documentPath": "./documents/strategy.json", "strategyId": "Paper-001", "environment": "sandbox" } }
  ],
  "reconcileOnStart": false,
  "cancelOrdersOnStop": true,
  "closePositionsOnStop": false,
  "heartbeatInterval": "00:01:00",
  "store": { "directory": "./node-store" }
}
```

Rules:

- `executionClients[].config.venue` must equal the data client's venue. The sandbox account id is `<VENUE>-SANDBOX`.
- SANDBOX config keys: `venue` (required), `omsType`, `accountType` (`cash` for spot, `margin` for perpetuals), `baseCurrency`, `startingBalances`, `defaultLeverage` [1], `barExecution`, `matchAgainstBook` [true], `subscribeOrderBook` [true], `bookDepth` [0 = all the venue sends], `bookType` [`l2`], `probFillOnLimit`, `probSlippage`, `latency`, `restore`. Fees are always the instrument's maker/taker rates.
- The document's candle series must be `"source": "computed"` (bars built from live trades). Validate with `--environment sandbox`.
- The document runs in the node's environment (`tradingRuntime.environment`): sandbox rules, and a warm-up from venue history before the first live bar. `"environment"` in the payload overrides it; leave it out. After start, the `view` shows the warm-up per candle series (`"warmedUp": true`, `"fromHistory"`).
- For perpetuals: the data client's family setting (`"accountType": "usdMFutures"` on Binance, `"productType": "linear"` on Bybit, `"instrumentType": "swap"` on OKX, ...: `references/venues.md`), SANDBOX `"accountType": "margin"` and `defaultLeverage` at least the document's `account.leverage`.
- `store.directory` writes a journal (`journal/<date>.jsonl`) and strategy state (`state/<id>.json`); a node restarted on the same directory restores its strategies (`store.restoreState`, default true). `store.saveInterval` (default `00:00:30`; `00:00:00` saves only on stop) saves state while it runs, so a node that is killed comes back where it was, with its timers keeping their phase. `store.journalDays` (default 30) prunes old journals.

**Matching against the live book.** By default the paper venue subscribes to the order book its real venue streams and fills against it: a taking order walks the book, a resting order waits its turn in the queue at its price. That answers "would this have filled, and at what" far better than quotes or bars. Check what it really matches against with a `view` over the control channel (below): `venues[].instruments[].against` is `book`, `quotes`, `bars` or `nothing`, the best source the venue has right now, with `wantsBook` and `bookRequested`. Report it with every paper result; fills against `bars` are much weaker evidence than fills against `book`. Resting orders in the `view` carry `sizeAhead` and `queuePosition` (`sizeAhead: 0` is the front of the queue).
- A simulated venue lives in memory. To resume where a stopped paper node left off, put balances in `startingBalances` and open positions/resting orders in `restore` (see `docs/integrations/sandbox.md`).

Validate, then run it in the background and tell the user how it stops:

```bash
bytex catalog fetch-instruments --path ./catalog --venue BINANCE --quote USDT
bytex documents validate --document documents/strategy.json --catalog ./catalog --environment sandbox
bytex run --config paper.json --control my-node --max-loss "500 USDT" --max-exposure 50%
bytex run --config paper.json --duration 01:00:00        # alternative: stops by itself after one hour
```

Risk flags on `bytex run` (each overrides `tradingRuntime.orderPolicy.limits` in the file): `--max-loss "<amount CCY>|<n>%"`, `--loss-period 1.00:00:00`, `--on-loss-limit stop-trading|deny-adds|flatten`, `--max-exposure "<amount CCY>|<n>%"`, `--max-open-positions`, `--max-open-positions-per-instrument`, `--max-working-orders`, `--max-working-orders-per-instrument`. Other flags: `--halted` (start placing nothing until `resume`), `--display-prices`, `--reconcile-interval`, `--env-file`, `--control`, `--control-heartbeat`.

The same limits in the file:

```json
"tradingRuntime": { "orderPolicy": { "maxOrderSubmitRate": 10, "limits": {
  "maxLossPerPeriod": "500 USDT", "lossPeriod": "1.00:00:00", "onLossLimit": "stopTrading",
  "maxExposure": "50%", "maxOpenPositions": 3, "maxWorkingOrders": 20 } } }
```

**Stopping.** Ctrl+C, SIGTERM or `--duration` stops the node; it then cancels orders and/or flattens per `cancelOrdersOnStop` / `closePositionsOnStop`. With `--control <name>` the node also serves the control channel: newline-delimited JSON, one client at a time, a named pipe `\\.\pipe\<name>` on Windows, a Unix socket elsewhere (`$TMPDIR/bytex-<name>.sock`; pass an absolute path such as `/tmp/bytex-my-node.sock` as the name to know it exactly). Messages are `{"type": "...", "payload": {...}}`; protocol version 6 (the `hello` says so). Commands: `status`, `view`, `cancel-all`, `flatten`, `halt` (`{cancelOrders, closePositions}`), `resume`, `limits` (a limits object), `stop` (`{cancelOrders, closePositions}`), and the strategy commands below. **`limits` replaces the whole set**: a field left out goes back to its default (a limit to off, `lossPeriod` to one day, `onLossLimit` to `stopTrading`), so always send every limit you want to keep. The node answers with a `status` carrying the limits it now holds. The node sends `hello`, `heartbeat`, `event`, `status`, `view`, `strategies`, `bye`. Protocol: `docs/design/0009-node-control-protocol.md`.

**Strategies while the node runs.** `strategies` lists them; `add-strategy` `{"path": "<absolute path>"}` loads one from a file holding a single strategy definition, the same object as an entry of the node config's `strategies` (for example `{"providerId": "bytex.document", "payload": {"documentPath": "./documents/b.json", "strategyId": "B-001"}}`); it is added **stopped**. `start-strategy` `{"id": "B-001"}`, `stop-strategy` `{"id", "cancelOrders", "closePositions"}`, `remove-strategy` `{"id"}` (refused while it runs or holds orders or positions). Each answers `strategies` with `done` or `refused` and every strategy's `state`, `openOrders` and `openPositions`. Tested on a `bytex run` paper node: add, start, stop and remove each answered `done: true`.

Send a command from a shell (the CLI has no client subcommand):

```bash
# Linux/macOS, node started with --control /tmp/bytex-my-node.sock
python3 - <<'EOF'
import json, socket
s = socket.socket(socket.AF_UNIX); s.connect("/tmp/bytex-my-node.sock")
s.sendall((json.dumps({"type": "stop", "payload": {"cancelOrders": True, "closePositions": False}}) + "\n").encode())
print(s.makefile().readline())   # hello
EOF
```

```powershell
# Windows, node started with --control my-node
$p = New-Object System.IO.Pipes.NamedPipeClientStream('.', 'my-node', [System.IO.Pipes.PipeDirection]::InOut)
$p.Connect(5000); $w = New-Object System.IO.StreamWriter($p); $w.AutoFlush = $true
$w.WriteLine('{"type":"stop","payload":{"cancelOrders":true,"closePositions":false}}'); Start-Sleep 3; $p.Dispose()
```

Read the `view` (what the node holds, what the venue matches against):

```powershell
# Windows, node started with --control my-node
$p = New-Object System.IO.Pipes.NamedPipeClientStream('.', 'my-node', [System.IO.Pipes.PipeDirection]::InOut)
$p.Connect(5000); $r = New-Object System.IO.StreamReader($p); $w = New-Object System.IO.StreamWriter($p); $w.AutoFlush = $true
$null = $r.ReadLine(); $w.WriteLine('{"type":"view"}')
do { $l = $r.ReadLine() } until ($l -match '"type":"view"'); $l; $p.Dispose()
```

On Linux/macOS use the Python snippet above with `{"type": "view"}` and read lines until one has `"type": "view"`.

What to report from a paper run: the heartbeat lines (`orders open`, `positions open`), fills from the log or the journal, what the venue matched against (`against` in the `view`), and how it compares with the backtest. Paper fills use the backtester's matching (`references/backtest.md`, "Simulator limitations"): against a book they walk it and queue; against bars they are bounded by bar volume.
