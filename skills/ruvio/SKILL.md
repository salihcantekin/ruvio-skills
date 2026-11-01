---
name: ruvio
description: Use when writing application code against a Ruvio server, choosing a Ruvio client library, or solving rate limits, locks, idempotency, delayed jobs, cache stampedes, counters or balances with Ruvio commands.
---

# Using Ruvio from application code

Ruvio is an in-memory key-value server that speaks the Redis protocol (RESP2, RESP3 after `HELLO 3`). Existing Redis clients and `redis-cli` connect, and it has single commands for jobs that need a Lua script or a second system elsewhere.

Ruvio's own commands are recent and are probably not in your training data. **Do not guess their arguments or replies.** The examples below are exact; for anything else, fetch the page named in the last section before writing code.

## Pick the command for the job

| Job | Command | Example and reply |
| --- | --- | --- |
| Rate limit (fixed window) | `LIMIT key max window_ms` | `LIMIT api:key-7 100 60000` → remaining count, or nil when full (nil = refused, nothing counted) |
| Rate limit (sliding / token bucket) | `LIMIT ... SLIDING`, `LIMIT key capacity refill_ms TOKENS [COST n] [WITHRETRY]` | `LIMIT api:key-7 10 1000 TOKENS COST 2 WITHRETRY` → `[8, 0]` allowed, `[-1, 150]` refused, retry in 150 ms |
| Lock / leader election with fencing | `LEASE key ttl_ms`, `RENEW key token ttl_ms`, `RELEASE key token` | `LEASE job:nightly 30000` → token or nil. Send the token to the protected resource and reject older tokens. |
| At most N concurrent holders | `SEMAPHORE key max ttl_ms`, `RELEASE key token` | `SEMAPHORE upstream:billing 8 15000` → token or nil when full |
| Request de-duplication | `IDEM key BEGIN fingerprint attempt ttl_ms`, `IDEM key COMPLETE fingerprint attempt result` | `BEGIN` → `["acquired"]`, `pending`, `conflict`, or `["completed", result]` |
| Run once per id (webhooks) | `ONCE key ttl_ms` | 1 the first time, 0 until the deadline |
| Stock / seats without overselling | `TAKE key n`, `INCRBY key n MAX cap` | nil and nothing written when refused |
| Move a balance | `TRANSFER src dst amount [MAX cap]`, `HTRANSFER key from_field to_field amount [MAX cap]` | `[src_after, dst_after]` or nil. TRANSFER keys must share a slot: `{user:42}:wallet`, `{user:42}:escrow`. |
| Compare-and-set, watermarks | `SET key value IFEQ expected`, `SET key n IFGT` / `IFLT`, `DELEX key IFEQ expected` | nil (SET) or 0 (DELEX) when the condition fails |
| Optimistic versioned writes | `SETV key value [version]` | new version number, error on a stale version |
| Counter with a deadline set on create | `INCR key EX seconds` / `PX ms` | deadline set only when the call creates the key |
| Cache fill without a stampede | `FILLGET key fill_ms [WAIT wait_ms]`, `FILLSET key value ttl_ms token [SWR stale_ms]` | `[HIT, v]`, `[FILL, token]`, `[BUSY, nil]`, `[STALE, v, token|nil]` |
| Due-job queue on a sorted set | `ZPOPBYSCORE key max [MIN min] [COUNT n] [WITHSCORES]`, `ZMOVEDUE src list max [COUNT n]` | pops or moves members with score ≤ max, up to 10000 per call |
| Delayed stream entries | `XADD key DELAY ms * field value ...` | `XREADGROUP` skips the entry until the delay passes |
| Capped recent-items list | `LPUSHCAP key cap element ...` / `RPUSHCAP` | `LPUSHCAP recent 3 a b c d` → 3, list is `d c b` |
| Change feed | `CHANGES SUBSCRIBE [FROM cursor|NOW] [MATCH glob] [TYPES ...]` | needs a WAL durability profile; give it its own connection |
| Durable acknowledgement | `CLIENT DURABILITY REPLICAS n [TIMEOUT ms]` | holds this connection's replies until n replicas have the writes on disk; refused on a server without replication |

Prefer these over Lua scripts or `WATCH`/`MULTI` retry loops: each one is a single command on the shard that owns the key, and a refused call writes nothing.

## Rules that prevent bugs

- Multi-key commands (`TRANSFER`, `ZMOVEDUE`, `MULTI` blocks, `MGET` in cluster mode) need every key in one hash slot. Use a hash tag: `{order:42}:items`, `{order:42}:total`.
- Store money as integers in the smallest unit (`INCRBY`, `TAKE`, `TRANSFER`). `INCRBYFLOAT` rounds to 16 significant digits.
- `LEASE`/`SEMAPHORE` tokens are opaque 64-bit integers that only grow, also across restarts and failovers. Treat them as integers, never parse them.
- A plain `SET` replaces a `SETV` versioned string. `GET` of a lease key is `WRONGTYPE`.
- For a lock or idempotency record that must survive a failover, send `CLIENT DURABILITY REPLICAS 1` on that connection and keep the nodes' clocks synchronized.
- `FILLGET ... WAIT` holds the connection. On one shared connection, use the client's `GetOrCreate`-style helper, which polls without `WAIT`.
- `CHANGES SUBSCRIBE` takes over its connection; open a dedicated one.

## Client libraries

All six have zero runtime dependencies and methods or helpers for the Ruvio commands above (locks, rate limits, idempotency, `GetOrCreate` for cache fills).

| Language | Package | Docs (plain text) |
| --- | --- | --- |
| Rust | `ruvio-client` (crates.io) | https://ruviodb.com/llms/clients/rust.txt |
| C# / .NET | `Ruvio.Client` (NuGet), `Ruvio.Client.AspNetCore` | https://ruviodb.com/llms/clients/csharp.txt |
| Go | `github.com/salihcantekin/ruvio/clients/go` | https://ruviodb.com/llms/clients/go.txt |
| Java | `io.github.salihcantekin:ruvio-client` (Maven Central) | https://ruviodb.com/llms/clients/java.txt |
| Python | `ruvio-client` (PyPI), sync and asyncio | https://ruviodb.com/llms/clients/python.txt |
| Node.js | `ruvio-client` (npm) | https://ruviodb.com/llms/clients/node.txt |

Read the client's page for the method names before writing code; do not assume they match another Redis client. Cluster routing, retries and replica reads are described in https://ruviodb.com/llms/client-behavior.txt.

ASP.NET Core has ready packages for rate limiting, idempotency, distributed locks, `IDistributedCache`, output caching and Data Protection: https://ruviodb.com/llms/integrations/dotnet.txt.

## Where the exact details are

- Index of every page, with sizes: https://ruviodb.com/llms.txt
- Every command with its limits: https://ruviodb.com/llms/commands.txt
- Ruvio commands in full, with replies and edge cases: https://ruviodb.com/llms/compatibility.txt
- Idempotency (`IDEM`) in depth: https://ruviodb.com/llms/idem.txt

Fetch the single page you need, not `llms-full.txt`.
