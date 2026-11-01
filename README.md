<p align="center">
  <a href="https://ruviodb.com/"><img src="assets/icon.png" width="96" alt="Ruvio"></a>
</p>

<h1 align="center">Ruvio skills</h1>

<p align="center">
  Agent skills that teach AI coding assistants how to use <a href="https://ruviodb.com/">Ruvio</a>,<br>
  a Rust key-value server that speaks the Redis protocol.
</p>

---

Ruvio has single commands for jobs that usually need a Lua script or a second system: locks with fencing tokens, rate limits, request de-duplication, delayed jobs, cache fills and balance transfers. These commands are new, so models tend to guess their arguments. The skills give the assistant the exact commands, the rules that prevent common bugs, and links to the plain-text docs at [ruviodb.com/llms.txt](https://ruviodb.com/llms.txt) for everything else.

| Skill | Used when |
| --- | --- |
| [`ruvio`](skills/ruvio/SKILL.md) | Writing application code against Ruvio, choosing a client, or solving rate limits, locks, idempotency, queues, cache fills, counters and balances |
| [`ruvio-deploy`](skills/ruvio-deploy/SKILL.md) | Installing, configuring and operating a server: Docker, persistence, auth and TLS, shards, replicas, automatic failover, backups |
| [`ruvio-from-redis`](skills/ruvio-from-redis/SKILL.md) | Moving code or data from Redis, checking compatibility, replacing Lua scripts with single commands |

The skills follow the open [Agent Skills](https://agentskills.io) format: each one is a folder with a `SKILL.md`.

## Install

**Claude Code**

```
/plugin marketplace add salihcantekin/ruvio-skills
/plugin install ruvio@ruvio
```

**Cursor**: install the `ruvio` plugin from the Cursor marketplace once it is listed, or copy the `skills/` folders into `.cursor/skills/` of your project.

**GitHub Copilot, Codex and other tools that read Agent Skills**: copy the folders under `skills/` into your project's `.github/skills/`, `.agents/skills/` or `.claude/skills/` folder, or into your personal skills folder (for example `~/.copilot/skills/`).

**Claude apps**: zip a skill folder and upload it in the Skills section of the settings.

## Versions

The skills describe the latest released Ruvio server. For anything not covered here, they point the assistant to the plain-text docs on ruviodb.com, which are rebuilt from the documentation on every site release.

## Contributing

Issues and pull requests are welcome. Every command example in a skill must have been run against a released server. Report a wrong command or reply as an issue with the server version.

## License

[MIT](LICENSE). The Ruvio server is AGPL-3.0; its client libraries are MIT. See the [main repository](https://github.com/salihcantekin/ruvio).
