# Praetor

> Self-hosted observability + security platform with a native MCP interface
> for LLM agents.

**Status:** 🚧 Pre-alpha (M0 — scaffolding). Repos are private until v0.1.0.

## What is it

Praetor lets you install a small agent on every Linux server in your fleet
with one curl command. It streams metrics, logs, and security events back
to a self-hosted control plane, which exposes everything through a
[Model Context Protocol](https://modelcontextprotocol.io/) server. Any LLM
agent (Claude Desktop, Claude Code, custom) can then investigate problems,
read further context, and (carefully) take actions on your fleet.

Imagine: *"Why is the database server slow?"* — and Claude actually goes
and looks. Pulls metrics, tails the right journal units, correlates with
recent deploys, and tells you what it found.

## Why

Existing options:

- **Datadog / New Relic** — closed source, expensive, your data leaves.
- **Wazuh** — heavy, security-focused, no AI surface.
- **Prometheus + Loki + Grafana** — great storage, but you still grep
  and dashboard manually.

Praetor sits on top of the Prometheus/Loki stack and adds:

1. A unified agent (one binary, not five exporters + promtail + filebeat)
2. Security event collection out of the box
3. A native MCP layer so any LLM agent can use it as a tool
4. A safe execution model so the LLM can do more than read

## Architecture (high level)

```
LLM Agent (Claude / your own)
        │ MCP
        ▼
   praetor-mcp ──► praetor-server ◄── praetor-agent (× N hosts)
                       │       │
                  Postgres  VictoriaMetrics + Loki
```

Full architecture in [ARCHITECTURE.md](./ARCHITECTURE.md).

## Repos

| Repo | Purpose |
|------|---------|
| [`praetor`](https://github.com/ondrejsindelka/praetor) | This umbrella |
| [`praetor-proto`](https://github.com/ondrejsindelka/praetor-proto) | Protobuf wire definitions |
| [`praetor-agent`](https://github.com/ondrejsindelka/praetor-agent) | Go agent (single static binary) |
| [`praetor-server`](https://github.com/ondrejsindelka/praetor-server) | Go control plane |
| [`praetor-mcp`](https://github.com/ondrejsindelka/praetor-mcp) | TypeScript MCP server |
| [`praetor-install`](https://github.com/ondrejsindelka/praetor-install) | Install script + packaging |

## Roadmap

See [ROADMAP.md](./ROADMAP.md). Short version:

- **v0.1** — install agent, see metrics & logs, query via MCP
- **v0.5** — anomaly detection + LLM triage + safe Tier-1 commands
- **v1.0** — production hardening, multi-tenancy, container introspection

Realistic timeline: **Q3 2026** for v0.1, **Q1 2027** for v1.0.

## License

🚧 To be decided before public release. See [ADR-006](./docs/adr/006-license.md).

## Author

Ondřej Šindelka — [@ondrejsindelka](https://github.com/ondrejsindelka)
