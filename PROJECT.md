# Praetor

> Self-hosted observability + security platform with a native MCP interface
> so LLM agents can investigate and (carefully) act on your fleet.

## What this is

Praetor is a monitoring agent + control plane + MCP server for Linux fleets.
You install the agent on a server with one curl command, it phones home over
mTLS, streams metrics/logs/security events, and exposes everything through
an MCP server that any LLM agent (Claude Desktop, Claude Code, custom) can
query and act on.

Think: Wazuh meets Datadog meets an AI SRE that actually understands what
it's looking at.

## Non-goals (v0.x)

- Windows or macOS targets (Linux only).
- APM / distributed tracing.
- Container runtime introspection (docker/podman/k8s) — phase 2.
- Replacement for Prometheus/Loki — we *use* them as storage.
- Multi-region HA control plane — single instance is fine for v0.

## Target scale

- **v0.1**: 10 hosts, 1 control plane, demo-ready.
- **v0.5**: 100 hosts, production-ready for homelab/SMB.
- **v1.0**: 500+ hosts, multi-tenant.

## High-level architecture

```
┌──────────────────────────────────────────────────────────────┐
│  LLM Agent (Claude Desktop / Claude Code / custom)           │
└──────────────────────┬───────────────────────────────────────┘
                       │ MCP (HTTP/SSE)
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  praetor-mcp (TypeScript)                                    │
│  Thin facade over praetor-server REST API.                   │
└──────────────────────┬───────────────────────────────────────┘
                       │ HTTP
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  praetor-server (Go) — control plane                         │
│  - gRPC ingress :8443 (agents, mTLS)                         │
│  - REST API   :8080 (mcp, ui)                                │
│  - Anomaly detector, command broker, alert engine            │
│  - Storage: VictoriaMetrics + Loki + Postgres                │
└──────────────────────┬───────────────────────────────────────┘
                       │ bidirectional gRPC over mTLS
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  praetor-agent (Go static binary)                            │
│  Collectors: metrics │ logs │ security │ command executor    │
└──────────────────────────────────────────────────────────────┘
```

## Repos

This is a multi-repo project. All under `github.com/ondrejsindelka/`:

| Repo | Language | Purpose |
|------|----------|---------|
| `praetor` | Markdown | Umbrella. Public-facing PROJECT.md, README, roadmap. |
| `praetor-proto` | Protobuf | Wire protocol between agent and server. Source of truth. |
| `praetor-agent` | Go | Single-binary agent installed on monitored hosts. |
| `praetor-server` | Go | Control plane: ingress, storage, REST API, business logic. |
| `praetor-mcp` | TypeScript | MCP server exposing praetor-server to LLM agents. |
| `praetor-install` | Bash | Install script + packaging (deb/rpm later). |

Each repo has its own README, CI, releases, and versioning.

Go module paths: `github.com/ondrejsindelka/praetor-agent`,
`github.com/ondrejsindelka/praetor-server`, etc.

## Key design decisions (locked in)

These are summarized here for quick reference. Full reasoning lives in ADRs
under `docs/adr/`.

1. **Agent in Go.** Single static binary, ~15 MB target, no runtime deps.
   ([ADR-001](./docs/adr/001-go-for-agent.md))
2. **Multi-repo, not monorepo.** Each component releases independently.
   ([ADR-002](./docs/adr/002-multi-repo.md))
3. **Bidirectional gRPC over mTLS** between agent and server. Agent always
   initiates the connection (works behind NAT). Server pushes commands and
   config back over the same stream. ([ADR-003](./docs/adr/003-grpc-bidirectional-mtls.md))
4. **Storage split**: VictoriaMetrics (metrics), Loki (logs), Postgres
   (state, audit, queue). ([ADR-004](./docs/adr/004-storage-split.md))
5. **Tiered command execution model.** Tier 0/1 auto-execute, Tier 2/3 need
   human approval. ([ADR-005](./docs/adr/005-tier-system-for-commands.md))
6. **`praetor-proto` is canonical.** Versioned with semver. `buf` toolchain.
   `buf breaking` enforced in CI.
7. **No agent listening ports.** Outbound only.
8. **mTLS with short-lived certs.** Enrollment is one-shot with a signed
   token (TTL ~15 min). Client cert auto-rotates daily.
9. **Two-tier configuration.** Local YAML for boot/connect, control plane
   pushes runtime config (collection cadence, log sources, exec policy).
10. **License**: TBD ([ADR-006](./docs/adr/006-licence.md)).
    All repos private until decided.

## Command execution model

The LLM will be able to ask praetor to run things on monitored hosts.
This is risky and needs to be designed carefully from day one.

| Tier | What | Approval |
|------|------|----------|
| **0 — Safe** | Predefined diagnostic checks (`disk_usage`, `top_processes`, `recent_logins`...) | None, audit logged |
| **1 — Validated** | Shell command from allowlisted binaries, args validated by regex, no shell metacharacters | None, audit logged |
| **2 — Write** | Restart services, kill processes, edit configs | Human approval in chat |
| **3 — Destructive** | Delete files, reboot, firewall changes | 2FA approval (chat + Telegram/email) |

**Rules:**

- Tier 0 + 1 are the "let the LLM investigate" tier. Sufficient for ~95% of
  diagnostics.
- Tier 2 + 3 are out of scope for v0.1. Skeleton only.
- Every command is recorded in an immutable audit log with: who requested
  (user or LLM model name), why (free-text reason), what ran, exit code,
  full stdout/stderr (truncated to N MB).
- Per-host rate limit + circuit breaker on command execution.

## Anomaly detection model

Two stages, in this order:

1. **Classical first.** Statistical detectors (EWMA, MAD, simple thresholds)
   and Prometheus-style alert rules run on a schedule against
   VictoriaMetrics. They produce *candidates*.
2. **LLM second.** Each candidate is enriched with context (last N minutes
   of metrics, relevant logs, host info, similar past incidents from
   Postgres) and handed to an LLM for triage: explanation, severity,
   correlation, recommended action, and a yes/no on "wake a human".

**Do not** use the LLM as a primary anomaly detector on raw time series.

## Roadmap (summary — full version in [ROADMAP.md](./ROADMAP.md))

- **M0** — Scaffolding: all 5 repos created, scaffolded, CI bootstrap
- **M1** — Walking skeleton: enrollment + heartbeat + basic metrics end-to-end
- **M2** — Observability backbone: VM + Loki integration, Grafana dashboard
- **M3** — MCP layer: TS MCP server, tested with Claude Desktop
- **M4** — Anomaly + LLM triage: classical detector, triage pipeline, Tier 0 cmds
- **M5** — Security collectors: auth.log, port watcher, sudo audit
- **M6** — Polish + opensource launch: license, signing, landing page

Realistic timeline: **2–3 months to v0.1.0** at part-time pace.

## Coding standards

### Go (agent + server)

- Go 1.22+
- `slog` for logging, JSON handler
- `golangci-lint` with default + `gocritic`, `revive`, `gosec`
- No `init()` for non-trivial work
- Errors wrapped with `fmt.Errorf("context: %w", err)`
- Context propagation everywhere; no `context.TODO()` outside main
- Tests: `_test.go` colocated, table-driven, `testify` allowed but optional
- Public packages have package-level doc comments

### TypeScript (mcp)

- Node 20+, ESM, strict TS
- `pnpm` for package management
- `eslint` + `prettier`
- Zod for runtime validation of MCP tool args

### Bash (install)

- `set -euo pipefail`
- `shellcheck` clean
- All variables quoted, all paths absolute

## Security principles

- Agent runs as unprivileged `praetor` user, NOT root.
- Privileged reads via group membership (`systemd-journal`, `adm`) or
  narrow sudoers entries with explicit absolute commands.
- Enrollment tokens are Ed25519-signed, single-use, TTL 15 min.
- Client certs are short-lived (24h), auto-rotated.
- mTLS required on agent→server. Server cert pinned via CA bundle.
- Server validates EVERY agent message; no implicit trust.
- Command execution is allowlisted at multiple layers: server policy,
  agent-side validator, sudoers (if applicable).
- Audit log is append-only and includes the LLM's reasoning string.
- SBOM generated in CI, dependencies scanned with `govulncheck` and `osv`.
- Releases signed with cosign keyless (Sigstore).
