---
name: ruvio-deploy
description: Use when installing, configuring or operating a Ruvio server: Docker or binary setup, durability and persistence, passwords, ACL and TLS, memory limits, shards, replicas, ruvio coordinator failover, backup and restore.
---

# Running and configuring Ruvio

Settings come from, in order: built-in defaults, a TOML file (`--config file`, or `RUVIO_CONFIG`), `RUVIO_*` environment variables, then command-line flags. Unknown TOML fields are rejected, and every setting is validated before a port is opened. `ruvio --help` lists the flags; release archives include `ruvio.example.toml`.

Setting names change between versions. **Do not invent a setting.** Check the Configuration page (link at the end) for the exact name, environment variable and flag.

## Start a server

Docker (linux/amd64 and linux/arm64):

```bash
docker run --rm -p 6379:6379 \
  -e RUVIO_REQUIREPASS=change-me \
  -v ruvio-data:/data \
  salihcantekin/ruvio:linux
redis-cli -a change-me PING
```

The image's data directory is `/data`. Its port is not loopback, so it needs `RUVIO_REQUIREPASS` (or an ACL password); `RUVIO_ALLOW_UNAUTHENTICATED=true` is the explicit exception for an already private network.

Binary: download from https://github.com/salihcantekin/ruvio/releases/latest (Linux x64/arm64, macOS arm64, Windows x64). Each archive has `ruvio` (server) and `ruvio-ctl` (backup, restore, checks, import). Arch Linux: `ruvio-bin` in the AUR.

```bash
./ruvio --requirepass change-me --data-dir ./data
```

A default process listens on `127.0.0.1:6379`.

## Settings that matter first

| Need | Environment variable | Notes |
| --- | --- | --- |
| Listen address | `RUVIO_ADDR` (`--address`) | default `127.0.0.1:6379` |
| Persistence | `RUVIO_DURABILITY` (`--durability`) | `memory` (default, nothing on disk), `snapshot`, `everysec`, `always` |
| Data directory | `RUVIO_DATA_DIR` (`--data-dir`) | needed for any profile other than `memory` |
| Password | `RUVIO_REQUIREPASS` (`--requirepass`) | |
| ACL users | `RUVIO_ACL_FILE` | |
| TLS | `RUVIO_TLS_CERT`, `RUVIO_TLS_KEY`, `RUVIO_TLS_CLIENT_CA`, `RUVIO_TLS_CLIENT_AUTH` | clients verify certificates by default |
| Memory limit | `RUVIO_MAXMEMORY`, `RUVIO_MAXMEMORY_POLICY` | one server-wide policy, applied per shard |
| Use more cores | `RUVIO_SHARDS` (`--shards`) | shard `i` listens on port + i; a command sent to the wrong shard is forwarded |

A non-loopback bind without a password keeps protected mode on and rejects remote clients. Turning protected mode off on a public bind without a password refuses to start.

## Replicas, failover and clusters

- Replication: `RUVIO_REPLICATION_ROLE` (`off`, `primary`, `replica`), `RUVIO_REPLICATION_LISTEN` on the primary, `RUVIO_REPLICATION_PRIMARY` on a replica, `RUVIO_REPLICATION_PASSWORD` for the handshake. Replication is asynchronous; a write that must survive a failover uses `CLIENT DURABILITY REPLICAS n` on its connection.
- Without coordinators, promotion is manual: `REPLICAOF NO ONE` on a replica.
- Automatic failover: run three (or five) `ruvio coordinator --config coordinator.toml` processes; a Raft majority decides. They hold no data; the six Ruvio clients follow the new primary by default.
- A multi-machine cluster uses 16384 slots split over groups by a topology file; nodes answer `MOVED` for slots they do not own.
- Keep node clocks synchronized when `LIMIT`, `ONCE`, `IDEM` or leases must survive a failover.

## Backup and checks

```bash
ruvio-ctl check   --data-dir ./data
ruvio-ctl backup  --data-dir ./data --dest ./backup   # add --sign-key FILE for a signed backup
ruvio-ctl restore --source ./backup --data-dir ./data
```

Upgrades can change the on-disk format: read the release notes and the Upgrading section of the coordinator page before upgrading a replicated or coordinated setup.

## Where the exact details are

- Setup (all platforms, Docker, build from source): https://ruviodb.com/llms/setup.txt
- Every setting, variable and flag: https://ruviodb.com/llms/configuration.txt (large; fetch it when you need a setting name)
- Shards, replication, cluster: https://ruviodb.com/llms/cluster.txt
- Automatic failover and upgrades: https://ruviodb.com/llms/coordinator.txt
- Backup, restore, integrity: https://ruviodb.com/llms/recovery.txt
- Metrics and logs: https://ruviodb.com/llms/observability.txt
- Index of every page: https://ruviodb.com/llms.txt
