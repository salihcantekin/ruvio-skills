---
name: ruvio-from-redis
description: Use when moving an application or data from Redis to Ruvio, checking whether existing Redis code or Lua scripts will run on Ruvio, or replacing Lua scripts and WATCH loops with Ruvio's single commands.
---

# Moving from Redis to Ruvio

Ruvio speaks RESP2 and RESP3, so `redis-cli` and existing Redis clients connect without changes. It implements a tested subset of Redis commands and semantics, not every edge of them. Check the compatibility page for any command your code relies on before saying it works.

## 1. Check the code

- Commands: the list CI checks is https://ruviodb.com/llms/commands.txt; semantics and differences per command family are in https://ruviodb.com/llms/compatibility.txt.
- Multi-key commands, `MULTI`/`EXEC` and scripts are same-slot and shard-local: give related keys a hash tag (`{order:42}:items`). There is no cross-shard transaction.
- Cluster mode allows only database 0 (`SELECT 0`). Classic `PUBLISH` stays on the node that receives it in a multi-node cluster; use `SPUBLISH` for cluster-wide channels.
- Lua: a Lua 5.1 sandbox with `EVAL`, `EVALSHA`, `SCRIPT LOAD/EXISTS/FLUSH`, `FCALL`, `FUNCTION LOAD/LIST/DELETE`. `redis.call`, `redis.pcall`, `redis.error_reply` and `redis.status_reply` exist; `redis.log`, `cjson`, `redis.sha1hex` and `redis.set_repl` do not. Globals are read-only (use `local`), `math.random` is removed, and `SCRIPT KILL` returns `NOTBUSY` (an instruction quota stops long scripts). `EVALSHA` after a restart is `NOSCRIPT` until the script is loaded again.
- Modules: JSON documents (`JSON.*`, Ruvio's own JSON type) and Bloom filters (`BF.*`, RedisBloom wire shape) are built in; other modules are not available.
- `INCRBYFLOAT` computes in double precision (16 significant digits). Keep money in integers.

## 2. Replace scripts with single commands

Many Lua scripts exist only to make a read-compare-write atomic. Ruvio has one command for the common ones; each runs on the shard that owns the key and writes nothing when it refuses.

| Typical Redis script or pattern | Ruvio command |
| --- | --- |
| `INCR` + `EXPIRE` rate limiter, sliding-window or token-bucket script | `LIMIT key max window_ms [SLIDING]`, `LIMIT key capacity refill_ms TOKENS [COST n] [WITHRETRY]` |
| `SET NX PX` lock with a check-and-delete release script | `LEASE key ttl_ms` → fencing token, `RELEASE key token`, `RENEW key token ttl_ms` |
| Counting semaphore with a sorted set | `SEMAPHORE key max ttl_ms`, `RELEASE key token` |
| Decrement only if enough stock | `TAKE key n`, or `INCRBY key n MAX cap` for a capped counter |
| Move an amount between two balances | `TRANSFER src dst amount [MAX cap]` (same slot), `HTRANSFER key from to amount [MAX cap]` |
| Compare-and-set / compare-and-delete | `SET key value IFEQ expected`, `DELEX key IFEQ expected` |
| Keep the max / min value | `SET key n IFGT`, `SET key n IFLT` |
| `WATCH` + `MULTI` optimistic update | `SETV key value version` |
| `INCR` then `EXPIRE` only on the first call | `INCR key EX seconds` (deadline set only on create) |
| Pop due jobs from a sorted set | `ZPOPBYSCORE key max [COUNT n]`, or `ZMOVEDUE src list max [COUNT n]` |
| `LPUSH` + `LTRIM` | `LPUSHCAP key cap element ...` / `RPUSHCAP` |
| Request de-duplication record | `IDEM key BEGIN ...` / `COMPLETE ...`, or `ONCE key ttl_ms` |
| Cache stampede lock around a refill | `FILLGET key fill_ms [WAIT ms]` + `FILLSET key value ttl_ms token` |

Exact replies and edge cases are in the `ruvio` skill and on https://ruviodb.com/llms/compatibility.txt. A Ruvio client library has helpers for most of these; see the client pages under https://ruviodb.com/llms.txt.

## 3. Move the data

```bash
ruvio-ctl import --from redis://host:6379 --to ruvio://host:6379 --dry-run
ruvio-ctl import --from redis://host:6379 --to ruvio://host:6379 --verify
```

- Reads the source with `SCAN` and per-type reads, writes with ordinary commands, keeps TTLs. No RDB file needed; any RESP2 source works (streams need Redis 6.2+).
- Existing target keys are skipped, so a second run is safe; `--replace` rewrites them. `--match 'user:*'` limits the keys; the database is part of the URL (`redis://host/3`).
- Pass passwords with `RUVIO_IMPORT_SOURCE_PASSWORD` and `RUVIO_IMPORT_TARGET_PASSWORD`, not in the URL. Connections are plain TCP: run it on a trusted network or a tunnel.
- Not moved: stream consumer groups and pending entries, Lua scripts and functions, ACL users, module types, field TTLs of hashes.
- `SCAN` is not a snapshot: stop writes for a clean copy, or run again with `--replace`.

Full reference: https://ruviodb.com/llms/import.txt
