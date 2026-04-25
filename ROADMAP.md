# Praetor — Roadmap

> Realistický timeline pro part-time tempo (vedle školy + práce):
> **2–3 měsíce do v0.1.0**.

## M0 — Scaffolding (1–2 dny)

**Cíl:** všechna repa existují, jsou nastavená, prázdné runy v CI procházejí.

- [ ] Repo `praetor` (umbrella) — public-facing PROJECT.md, README, LICENSE placeholder
- [ ] Repo `praetor-proto` — `buf.yaml`, `buf.gen.yaml`, prázdný `praetor/v1/` adresář, CI: `buf lint` + `buf breaking`
- [ ] Repo `praetor-agent` — Go module, cmd/internal layout, Makefile, prázdný main, CI: build + test + lint
- [ ] Repo `praetor-server` — totéž
- [ ] Repo `praetor-mcp` — pnpm + tsconfig + eslint + prettier, prázdný entry, CI: typecheck + lint
- [ ] Repo `praetor-install` — install.sh kostra, shellcheck v CI
- [ ] V `praetor` repu: ARCHITECTURE.md (zrcadlo z vaultu), CONTRIBUTING.md placeholder, ADR index
- [ ] CLAUDE.md v `praetor` repu (instrukce pro Claude Code)

**Hotovo když:**
- `git push` na každé repo zelený CI
- `gh repo list ondrejsindelka --limit 20` ukazuje všech 6 repů

## M1 — Walking skeleton (1–2 týdny)

**Cíl:** end-to-end demo. Spustím install na 3 hosty, vidím je v API.

- [ ] **Spec:** [SPEC-001](./docs/specs/001-enrollment-flow.md)
- [ ] **proto:** definice `Enroll`, `Connect`, `Heartbeat`, `HostInfo`, základní `Metric`
- [ ] **server:** Postgres migrace (`hosts`, `agent_identities`, `enrollment_tokens`)
- [ ] **server:** `Enroll` RPC handler (token verify, CSR signing)
- [ ] **server:** `Connect` stream handler — přijímá heartbeats, ukládá do Postgresu
- [ ] **server:** REST `GET /v1/hosts`, `GET /v1/hosts/:id`
- [ ] **server:** CLI subkomanda `praetor-server token issue --label foo`
- [ ] **agent:** identity loader, enrollment caller
- [ ] **agent:** stream client s reconnect/backoff
- [ ] **agent:** heartbeat loop (každých 30s)
- [ ] **agent:** první 4 metriky (cpu_pct, mem_used_pct, disk_used_pct per mount, net_bytes per iface)
- [ ] **install:** kompletní script — download → user → systemd → enroll
- [ ] Demo: install na 3 LXC kontejnerů v homelabu, `curl /v1/hosts` ukáže všechny

## M2 — Observability backbone (1–2 týdny)

- [ ] **Spec:** [SPEC-002](./docs/specs/002-grpc-stream-lifecycle.md) (config push, reconnect, backpressure)
- [ ] **proto:** `LogBatch`, `LogEntry`, `ConfigUpdate`, `AgentConfig`, `ConfigAck`
- [ ] **server:** VictoriaMetrics integrace — write přes vmagent nebo přímo
- [ ] **server:** Loki integrace
- [ ] **server:** config push — operátor mění config v DB, server pushne přes stream
- [ ] **agent:** journald reader (přes `sd_journal` cgo, nebo `journalctl --output=json --follow`)
- [ ] **agent:** přijímá `ConfigUpdate`, aplikuje za běhu
- [ ] Grafana dashboard: "Fleet overview" (počet hostů, celkový CPU/RAM, top heavy hosts)
- [ ] Grafana dashboard: "Host detail" (CPU/RAM/disk/net + journal logs)

## M3 — MCP layer (1 týden)

- [ ] **Spec:** [SPEC-003](./docs/specs/003-mcp-tools.md)
- [ ] **mcp:** `@modelcontextprotocol/sdk` setup, HTTP server
- [ ] **mcp:** MCP tools:
  - `list_hosts(filters?)` — seznam s metadaty + status
  - `get_host(host_id)` — detail
  - `get_metrics(host_id, metric, range, step?)` — dotaz na VM
  - `search_logs(host_id?, query, range, limit?)` — LogQL přes Loki
  - `get_recent_security_events(host_id?, since?, severity?)` — z Postgresu
- [ ] Auth — bearer token, MCP server pošle do REST API
- [ ] Test s Claude Desktop: "Co se děje na hostu prod-db-01?"
- [ ] Doc: jak nakonfigurovat Claude Desktop (claude_desktop_config.json)

## M4 — Anomaly + LLM triage (2 týdny)

- [ ] **proto:** `CommandRequest`, `CommandResult`, `DiagnosticCommand`, `CommandTier`, `ExecutionPolicy`
- [ ] **server:** classical anomaly detector — cron, EWMA + threshold rules
- [ ] **server:** alert storage (`alerts`, `alert_candidates` tabulky)
- [ ] **server:** triage worker — vyzvedne candidate, build context bundle, call LLM, store result
- [ ] **server:** Discord webhook notifier
- [ ] **agent:** Tier 0 diagnostic checks:
  - `disk_usage` — `df -h` ekvivalent přes gopsutil
  - `top_processes` — top 10 by CPU/RAM
  - `recent_auth_events` — posledních 50 řádek auth.log
  - `journalctl_for_unit` — `journalctl -u X --since=...`
  - `read_config_file` — read s allowlist (`/etc/nginx/*`, `/etc/systemd/*`, ...)
- [ ] **agent:** Tier 1 shell executor — strict allowlist (`ss`, `journalctl`, `cat` na allowlist paths), validátor proti metaznakům
- [ ] **server:** REST `POST /v1/commands`, `GET /v1/commands/:id`
- [ ] **mcp:** nástroj `run_diagnostic(host, check, params)`
- [ ] **mcp:** nástroj `run_shell(host, binary, args, reason)` (jen Tier 1)

## M5 — Security collectors (2 týdny)

- [ ] **agent:** auth.log tail/parse — SSH login (success), SSH failed auth
- [ ] **agent:** sudo events parser
- [ ] **agent:** listening port watcher — diff vs baseline
- [ ] **agent:** /etc/passwd integrita
- [ ] **agent:** process anomaly (nový proces běžící z neobvyklého místa)
- [ ] **server:** rule engine pro security alerts (např. ">5 failed auths in 1min from same IP")
- [ ] **mcp:** `get_security_events` rozšířený

## M6 — Polish + opensource launch (1–2 týdny)

- [ ] Licence vybraná a aplikovaná na všechna repa ([ADR-006](./docs/adr/006-licence.md))
- [ ] CONTRIBUTING.md, CODE_OF_CONDUCT.md, SECURITY.md ve všech repech
- [ ] Reproducible builds (Go: `-trimpath`, `-buildvcs=true`)
- [ ] cosign keyless signing v release CI
- [ ] SBOM v každém releasu (cyclonedx)
- [ ] `praetor.dev` landing page (Astro)
- [ ] Demo video (asciinema + voiceover, max 3 min)
- [ ] README badges, screenshots
- [ ] Show HN draft post
- [ ] r/selfhosted draft post
- [ ] **Repos public switch**

## Po v0.1 — backlog

- Multi-tenancy (org_id všude)
- Container introspection (docker, podman, k8s)
- eBPF collectors
- File integrity monitoring (FIM)
- Web UI (samostatný `praetor-ui` repo)
- Tier 2/3 commandy s human-in-the-loop schvalováním
- Vlastní agentní runtime (LLM agent v praetor-server, který volá MCP nástroje sám)
- Helm chart + Docker Compose pro server
- packagecloud.io / homebrew tap

---

## Princip

**Každý milník končí demem nebo screenshotem v daily note.** Bez toho je
to jen plán. M0 demo: `gh repo list`. M1 demo: `curl /v1/hosts`. M2 demo:
Grafana screenshot. M3 demo: konverzace s Claude Desktop. Atd.
