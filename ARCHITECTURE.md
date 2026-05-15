# Praetor — Architecture

> Component-level view of the system and how the pieces communicate.
> For the full project brief, see [PROJECT.md](./PROJECT.md).

## Components and responsibilities

### praetor-agent

- **Goal:** stay as small and quiet as possible. ~15 MB static binary, no runtime dependencies.
- **What it does:**
  - On first start: enrollment (token → CSR → cert) → stores identity in `/var/lib/praetor/`
  - Maintains a single persistent gRPC stream to the control plane
  - Collects: metrics (CPU/RAM/disk/net/processes), logs (journald, allowlisted files), security events (auth.log, listening ports, sudo, user changes)
  - Executes Tier 0/1 commands (with a local validator); refuses Tier 2/3 unless explicitly enabled in `ExecutionPolicy`
- **What it does NOT do:**
  - No listening ports — all connections are outbound
  - No telemetry written to disk (only identity state) — stateless beyond the cert
  - Never spawns a shell with metacharacters (`bash -c`, pipes, redirects)

### praetor-server (control plane)

- **Goal:** accept agents, store data, apply business logic, expose a REST API.
- **Internal services:**
  - **Agent ingress** — gRPC :8443, mTLS, accept loop, per-agent goroutine
  - **Storage routers** — heartbeat → Postgres, metrics → VictoriaMetrics, logs → Loki, security events → Postgres
  - **Anomaly detector** — cron job, reads from VictoriaMetrics, writes candidates to Postgres
  - **LLM triage worker** — picks up candidates, enriches context, calls Anthropic API (or a local Qwen3-compatible endpoint)
  - **Command broker** — per-agent command queue, delivers over the existing stream
  - **REST API :8080** — consumed by the MCP server and any future UI
- **External dependencies:**
  - Postgres (state, audit log, queue, alerts)
  - VictoriaMetrics (TSDB)
  - Loki (log storage)
  - Anthropic API or a local OpenAI-compatible LLM (triage worker only, planned for M4)

### praetor-mcp

- **Goal:** thin facade. MCP tools map 1:1 to REST endpoints.
- No business logic. No state. Restartable at any time.
- Written in TypeScript because the MCP SDK ecosystem is best there; the control plane stays in Go for performance and deployment simplicity.

### praetor-install

- **Goal:** one `curl | bash` command that works on Debian/Ubuntu/RHEL.
- Downloads the binary, creates the system user, writes the systemd unit, triggers enrollment.
- Verifies the binary checksum (cosign keyless signature planned for a later milestone).

## Network model

```
Agent (anything behind NAT)
  │
  │ outbound TCP 8443, mTLS
  ▼
Control plane (publicly reachable endpoint)
  │
  ├── Postgres (internal network)
  ├── VictoriaMetrics (internal network)
  └── Loki (internal network)

LLM client (Claude Desktop on the user's machine)
  │
  │ MCP (HTTP/SSE or stdio)
  ▼
praetor-mcp (runs locally or alongside the server)
  │
  │ HTTP + bearer token
  ▼
Control plane REST :8080
```

**Key invariant:** no component opens an inbound connection toward the agent.
Everything flows agent → server.

## Identity and trust

```
┌─────────────────────────────┐
│ Praetor Root CA             │  (offline key, generated at server install time)
└─────────┬───────────────────┘
          │ issues
          ▼
┌─────────────────────────────┐
│ Server cert (long-lived)    │  (1 year+, rotation planned)
└─────────────────────────────┘
          │ issues
          ▼
┌─────────────────────────────┐
│ Agent client certs (24 h)   │  (auto-rotated via Enroll RPC)
└─────────────────────────────┘
```

**Enrollment flow:**

1. Operator issues an enrollment token on the server (CLI or UI). The token is an Ed25519-signed structure: `{agent_label, exp, nonce}`. TTL: 15 minutes.
2. Operator passes the token to the install command as `PRAETOR_TOKEN=...`.
3. The agent generates a keypair and a CSR.
4. The agent calls `Enroll(token, csr_pem, host_info)` — an unauthenticated gRPC call.
5. The server verifies the token signature, expiry, and nonce (single-use), then issues a client certificate.
6. The agent writes `cert.pem`, `key.pem`, and `agent_id` to `/var/lib/praetor/`.
7. The agent calls `Connect()` over mTLS — bidirectional stream begins.
8. Every ~20 hours the agent re-calls `Enroll()` using the same flow but authenticated with its current cert (certificate rotation).

## Data flow

### Metrics

```
agent collector → in-memory ring buffer
               → flush every 15 s
               → MetricBatch over gRPC stream
               → server ingestor
               → VictoriaMetrics (write API)
```

### Logs

```
journald reader → parse → filter (severity ≥ warning)
               → batch (1 s or 100 entries, whichever comes first)
               → LogBatch over gRPC stream
               → server ingestor
               → Loki (push API)
```

### Security events

```
auth.log tail → parse → SecurityEvent
             → send immediately (no batching — low volume, high urgency)
             → server ingestor
             → Postgres (security_events table)
             → trigger: alert engine evaluation
```

### Commands (server → agent)

```
LLM/user → REST POST /v1/commands {host, type, params, reason}
         → server validates against host's ExecutionPolicy
         → command_queue (Postgres)
         → command broker picks up
         → finds active stream for the target host
         → CommandRequest sent down the stream
         → agent validator (second layer of defense)
         → agent executor
         → CommandResult sent up the stream
         → server stores result in command_audit
         → REST GET /v1/commands/:id returns the result
```

## Self-observability

Praetor must monitor itself — otherwise we'd have a "monitoring the monitor" problem. Goals for v0.1:

- Server exports its own metrics on `/metrics` (Prometheus format), scrapable by VictoriaMetrics.
- Agent sends *its own* operational metrics over the stream (buffered metric count, dropped events, gRPC reconnects, command queue depth).
- Structured JSON logs (Go `slog`) from all components.

## Deliberate non-goals (early milestones)

- **eBPF.** Powerful, but increases footprint and complexity. Possible in v1.x.
- **Auto-discovery.** Operators install the agent explicitly. No network scanning.
- **Web UI in the server repo.** The REST API is sufficient. A UI is a separate project later, or Grafana suffices.
- **Plugins.** Fixed set of collectors baked into the binary. Plugins → ecosystem → security nightmare.
- **Distributed control plane.** Single-node Postgres + VictoriaMetrics + Loki + server. Vertical scaling handles 500+ hosts comfortably.
