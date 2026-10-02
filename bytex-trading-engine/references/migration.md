# BYTEX v2 Migration And Recovery

Canonical examples:

```text
bx-market:v2/SIM/BTCUSDT
bx-candle:v2/SIM/BTCUSDT/minute/1/last/provider
bx-sampling:v2/minute/1/last
```

Symbols are escaped once inside identities; identities are escaped again as
object-key components. Construct them with MarketKey, CandleSeries and
SamplingRule rather than concatenating strings. Origin is provider or computed.
Normal readers reject legacy strings and timestamp-filename archives.

```bash
bytex migrate archive --source ./legacy-catalog --destination ./v2-catalog
bytex migrate document --source ./legacy-document.json --destination ./v2-document.json
bytex catalog check --path ./v2-catalog
bytex catalog recovery --path ./v2-catalog
```

Archive destinations must be empty on first use and separate from the source,
not nested inside it. Keep the source unchanged during conversion and resume.
Resuming verifies the pinned inventory and committed source hashes; already
committed files are not imported twice. Changed inputs and unknown source objects
are refused for review, not silently dropped. Do not accept a destination until
conversion completes, catalog check passes, and replay parity is established.

Opaque segment writes become visible only when their immutable manifest commits.
Failed pre-commit writes remain uncommitted recovery evidence. Consolidation
publishes replacement segments before superseding originals, which remain stored.
Recovery listing is read-only; no automatic garbage collection deletes evidence.
Run only one migration/consolidation maintenance operation per stream at a time;
concurrent appends use unique objects, but conflicting maintenance requires
explicit operator review. Do not edit manifests or delete retained data casually.

Document conversion writes a separate file and refuses a differing existing
destination. Keep legacy originals and a verified independent backup. These
commands do not authorize publication, remote deletion, or real orders.
