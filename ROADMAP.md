# Praetor — Roadmap

> Realistic timeline for a part-time pace (alongside university and work).

## Status

| Milestone | Status | Description |
|-----------|--------|-------------|
| M0 | ✅ Done | Scaffolding — all repos, CI green |
| M1.1 | ✅ Done | Proto v0.1.0 — Metric / MetricBatch / MetricType |
| M1.2 | ✅ Done | Server Postgres schema + goose migrations + docker-compose |
| M1.3 | 🚧 In progress | CA + Enroll RPC + token CLI + Connect stream + heartbeat persistence |
| M1.4 | ⏭ Next | Agent enrollment + stream client + first metrics end-to-end |
| M1.5 | ⏭ Planned | Install script — real binary download + enrollment trigger |
| M2 | ⏭ Planned | Observability backbone — VictoriaMetrics + Loki + Grafana dashboards |
| M3 | ⏭ Planned | MCP layer — full tool set for Claude Desktop / Claude Code |
| M4 | ⏭ Planned | Anomaly detection + LLM triage + safe Tier-1 commands |
| M5 | ⏭ Planned | Security collectors |
| M6 | ⏭ Planned | Polish + open-source launch |

---

## M0 — Scaffolding ✅

**Goal:** all repos exist, are configured, and CI passes on empty runs.

- [x] Repo `praetor` (umbrella) — public-facing README, ARCHITECTURE, LICENSE placeholder
- [x] Repo `praetor-proto` — `buf.yaml`, `buf.gen.yaml`, `praetor/v1/` layout, CI: `buf lint` + `buf breaking`
- [x] Repo `praetor-agent` — Go module, cmd/internal layout, Makefile, CI: build + test + lint
- [x] Repo `praetor-server` — same as agent
- [x] Repo `praetor-mcp` — pnpm + tsconfig + eslint + prettier, CI: typecheck + lint
- [x] Repo `praetor-install` — install.sh skeleton, shellcheck in CI

**Done when:** `gh repo list ondrejsindelka --limit 20` shows all 6 repos with green CI.

## M1 — Walking skeleton 🚧

**Goal:** end-to-end demo. Install on 3 hosts, see them in the API.

**Spec:** [SPEC-001 — Enrollment flow](./docs/specs/001-enrollment-flow.md)

### M1.1 ✅ — Proto v0.1.0
- [x] `Enroll`, `Connect`, `Heartbeat`, `HostInfo`, `Metric`, `MetricBatch`, `MetricType`

### M1.2 ✅ — Server data layer
- [x] Postgres migrations: `hosts`, `agent_identities`, `enrollment_tokens`
- [x] docker-compose dev stack (Postgres, VictoriaMetrics, Loki, Grafana)
- [x] goose migration runner wired into the binary

### M1.3 🚧 — Auth + streams
- [ ] Internal CA: root CA + server cert generated at first start
- [ ] `Enroll` RPC handler: token verify → CSR sign → client cert issued
- [ ] `token issue` / `token list` / `token revoke` CLI subcommands
- [ ] `Connect` stream handler: receives heartbeats, persists to Postgres

### M1.4 ⏭ — Agent end-to-end
- [ ] Agent: identity loader, enrollment caller
- [ ] Agent: stream client with reconnect/backoff
- [ ] Agent: heartbeat loop (every 30 s)
- [ ] Agent: first 4 metrics — `cpu_pct`, `mem_used_pct`, `disk_used_pct` (per mount), `net_bytes` (per interface)
- [ ] Server: REST `GET /v1/hosts`, `GET /v1/hosts/:id`

### M1.5 ⏭ — Install script
- [ ] Full script: download → SHA256 verify → user → systemd → enroll
- [ ] Demo: install on 3 LXC containers in homelab, `curl /v1/hosts` shows all three

---

## M2 — Observability backbone ⏭

**Spec:** [SPEC-002 — gRPC stream lifecycle](./docs/specs/002-grpc-stream-lifecycle.md)

- [ ] **proto:** `LogBatch`, `LogEntry`, `ConfigUpdate`, `AgentConfig`, `ConfigAck`
- [ ] **server:** VictoriaMetrics integration — metrics write via vmagent or directly
- [ ] **server:** Loki integration
- [ ] **server:** config push — operator changes config in DB, server pushes update over stream
- [ ] **agent:** journald reader (`sd_journal` via cgo, or `journalctl --output=json --follow`)
- [ ] **agent:** receives `ConfigUpdate`, applies at runtime without restart
- [ ] Grafana dashboard: "Fleet overview" (host count, aggregate CPU/RAM, top heavy hosts)
- [ ] Grafana dashboard: "Host detail" (CPU/RAM/disk/net + journal logs)

---

## M3 — MCP layer ⏭

**Spec:** [SPEC-003 — MCP tools](./docs/specs/003-mcp-tools.md)

- [ ] `@modelcontextprotocol/sdk` setup, HTTP server
- [ ] Tools:
  - `list_hosts(filters?)` — list with metadata + status
  - `get_host(host_id)` — detail
  - `get_metrics(host_id, metric, range, step?)` — query VictoriaMetrics
  - `search_logs(host_id?, query, range, limit?)` — LogQL via Loki
  - `get_recent_security_events(host_id?, since?, severity?)` — from Postgres
- [ ] Bearer token auth flowing through to REST API
- [ ] Manual test with Claude Desktop: "What's happening on prod-db-01?"
- [ ] Doc: how to configure Claude Desktop (`claude_desktop_config.json`)

---

## M4 — Anomaly detection + LLM triage ⏭

- [ ] **proto:** `CommandRequest`, `CommandResult`, `DiagnosticCommand`, `CommandTier`, `ExecutionPolicy`
- [ ] **server:** classical anomaly detector — cron, EWMA + threshold rules
- [ ] **server:** alert storage (`alerts`, `alert_candidates` tables)
- [ ] **server:** triage worker — builds context bundle, calls LLM, stores result
- [ ] **server:** Discord webhook notifier
- [ ] **agent:** Tier 0 diagnostic checks:
  - `disk_usage` — gopsutil equivalent of `df -h`
  - `top_processes` — top 10 by CPU / RAM
  - `recent_auth_events` — last 50 lines of auth.log
  - `journalctl_for_unit` — `journalctl -u X --since=...`
  - `read_config_file` — read with allowlist (`/etc/nginx/*`, `/etc/systemd/*`, …)
- [ ] **agent:** Tier 1 shell executor — strict allowlist (`ss`, `journalctl`, `cat` on allowlisted paths), validator rejects metacharacters
- [ ] **server:** `POST /v1/commands`, `GET /v1/commands/:id`
- [ ] **mcp:** `run_diagnostic(host, check, params)`
- [ ] **mcp:** `run_shell(host, binary, args, reason)` (Tier 1 only)

---

## M5 — Security collectors ⏭

- [ ] **agent:** auth.log tail/parse — SSH login (success), SSH failed auth
- [ ] **agent:** sudo event parser
- [ ] **agent:** listening port watcher — diff vs. baseline
- [ ] **agent:** `/etc/passwd` integrity check
- [ ] **agent:** process anomaly — new process running from an unusual path
- [ ] **server:** rule engine for security alerts (e.g. ">5 failed auth attempts in 1 min from the same IP")
- [ ] **mcp:** extended `get_security_events`

---

## M6 — Polish + open-source launch ⏭

- [ ] License confirmed and applied to all repos
- [ ] CONTRIBUTING.md, CODE_OF_CONDUCT.md, SECURITY.md consistent across all repos
- [ ] Reproducible builds (Go: `-trimpath`, `-buildvcs=true`)
- [ ] cosign keyless signing in release CI
- [ ] SBOM attached to every release (CycloneDX)
- [ ] `praetor.dev` landing page
- [ ] Demo video (asciinema + voiceover, max 3 min)
- [ ] README badges, screenshots
- [ ] Show HN draft
- [ ] r/selfhosted draft
- [ ] **Repos set to public**

---

## Post-v0.1 backlog

- Multi-tenancy (`org_id` throughout)
- Container introspection (Docker, Podman, Kubernetes)
- eBPF collectors
- File integrity monitoring (FIM)
- Web UI (`praetor-ui` repo)
- Tier 2/3 commands with human-in-the-loop approval
- Autonomous agent runtime (LLM agent inside praetor-server that calls MCP tools itself)
- Helm chart + Docker Compose for the server stack
- packagecloud.io / Homebrew tap

---

## Principle

**Every milestone ends with a demo or screenshot in the daily note.** Without that, it's just a plan. M0 demo: `gh repo list`. M1 demo: `curl /v1/hosts`. M2 demo: Grafana screenshot. M3 demo: conversation with Claude Desktop. And so on.
