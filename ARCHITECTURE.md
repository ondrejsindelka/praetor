# Praetor — Architecture

> Detailní pohled na komponenty a jak spolu mluví. PROJECT.md je 30k stop;
> tohle je 5k stop.

## Komponenty a jejich odpovědnosti

### praetor-agent

- **Cíl:** být co nejmenší a co nejtišší. ~15 MB binárka, žádné runtime deps.
- **Co dělá:**
  - Po instalaci: enrollment (token → CSR → cert) → uloží identitu do `/var/lib/praetor/`
  - Drží jeden persistentní gRPC stream do control plane
  - Sbírá: metriky (CPU/RAM/disk/net/proc), logy (journald, allowlistované soubory), security eventy (auth.log, listening porty, sudo, user changes)
  - Vykonává Tier 0/1 commandy (s lokálním validátorem); Tier 2/3 odmítá pokud nejsou explicitně povoleny v ExecutionPolicy
- **Co nedělá:**
  - Nemá žádný listening port
  - Nezapisuje telemetrii na disk (jen identitu) — je stateless beyond identity
  - Nikdy nespouští shell s metaznaky (žádné `bash -c`, žádné pipes)

### praetor-server (control plane)

- **Cíl:** přijímá agenty, ukládá data, dělá business logic, vystavuje REST.
- **Procesy/služby uvnitř:**
  - **Agent ingress** — gRPC :8443, mTLS, accept loop, per-agent goroutina
  - **Storage routers** — heartbeat → Postgres, metrics → VM, logs → Loki, security events → Postgres
  - **Anomaly detector** — cron, čte z VM, píše candidates do Postgresu
  - **LLM triage worker** — vyzvedává candidates, obohacuje, volá Anthropic API
  - **Command broker** — fronta commandů per agent, odesílá přes existující stream
  - **REST API :8080** — pro MCP server a UI
- **Externí závislosti:**
  - Postgres (state, audit, queue, alerts)
  - VictoriaMetrics (TSDB)
  - Loki (logs)
  - Anthropic API (LLM triage) — nebo lokální Qwen3 přes OpenAI-compat endpoint

### praetor-mcp

- **Cíl:** tenká fasáda. MCP nástroje 1:1 mapují REST endpointy.
- Žádná business logic. Žádný stav. Restartovatelná kdykoliv.
- Důvod proč to existuje: MCP klienti chtějí MCP rozhraní, REST API server
  potřebuje pro UI a integrace. Oddělení = MCP může být psané v TS
  (ekosystém SDK je tam nejlepší), server jede v Go.

### praetor-install

- **Cíl:** jeden curl příkaz, který funguje na Debian/Ubuntu/RHEL.
- Stáhne binárku, vytvoří uživatele, položí systemd unit, spustí enrollment.
- Verifikuje cosign podpis binárky před instalací.

## Síťový model

```
Agent (cokoliv za NATem)
  │
  │ outbound TCP 8443, mTLS
  ▼
Control plane (veřejně přístupný endpoint)
  │
  ├── Postgres (interní síť)
  ├── VictoriaMetrics (interní síť)
  └── Loki (interní síť)

LLM klient (Claude Desktop u uživatele)
  │
  │ MCP (HTTP/SSE)
  ▼
praetor-mcp (běží lokálně u uživatele NEBO vedle serveru)
  │
  │ HTTP (s API tokenem)
  ▼
Control plane REST :8080
```

**Důležité:** žádná komponenta neotevírá inbound port směrem k agentovi.
Vše jde od agenta nahoru.

## Identita a důvěra

```
┌─────────────────────────────┐
│ Praetor Root CA             │  (offline klíč, generovaný při instalaci serveru)
└─────────┬───────────────────┘
          │ vystavuje
          ▼
┌─────────────────────────────┐
│ Server cert (long-lived)    │  (rok+, rotace plánovaná)
└─────────────────────────────┘
          │ vystavuje
          ▼
┌─────────────────────────────┐
│ Agent client certs (24h)    │  (auto-rotace přes Enroll RPC)
└─────────────────────────────┘
```

**Enrollment flow:**

1. Operátor vyrobí enrollment token na serveru (CLI nebo UI). Token je
   Ed25519-signed JWT-like struktura: `{agent_label, exp, nonce}`. TTL 15 min.
2. Operátor vloží token do install commandu jako `WATCHTOWER_TOKEN=...`
3. Agent při bootstrapu vygeneruje keypair, vyrobí CSR
4. Agent zavolá `Enroll(token, csr_pem, host_info)` — neauth gRPC volání
5. Server ověří podpis tokenu, exp, nonce (single-use), vystaví client cert
6. Agent uloží `cert.pem`, `key.pem`, `agent_id` do `/var/lib/praetor/`
7. Agent zavolá `Connect()` (mTLS) — bidirectional stream
8. Každých ~20 hodin agent znovu zavolá `Enroll()` s tím samým flow ale
   autentizovaný současným cert (rotace)

## Datový tok

### Metriky

```
agent.collector → in-memory ring buffer
                → flush every 15s
                → MetricBatch over gRPC stream
                → server ingestor
                → VictoriaMetrics (write API)
```

### Logy

```
journald reader → parse → filter (severity ≥ warning)
                → batch (1s nebo 100 entries)
                → LogBatch over gRPC stream
                → server ingestor
                → Loki (push API)
```

### Security eventy

```
auth.log tail → parse → SecurityEvent
              → immediate send (no batching, low volume)
              → server ingestor
              → Postgres (security_events table)
              → trigger: alert engine eval
```

### Commandy (server → agent)

```
LLM/user → REST POST /v1/commands {host, type, params, reason}
         → server validates against host's ExecutionPolicy
         → command_queue (Postgres)
         → command broker picks up
         → finds active stream for host
         → CommandRequest down the stream
         → agent validator (second layer of defense)
         → agent executor
         → CommandResult up the stream
         → server stores in command_audit
         → REST GET /v1/commands/:id returns result
```

## Observability sebe sama

Praetor musí monitorovat sám sebe — jinak budeme mít "monitoring monitoringu"
problém. Cíl pro v0.1:

- Server exportuje vlastní metriky na `/metrics` (Prometheus format),
  scrapable VM
- Agent exportuje *do svého vlastního streamu* metriky o sobě (počet
  buffered metrics, dropped events, gRPC reconnects, command queue depth)
- Strukturované logy (slog JSON) ze všech komponent

## Co schválně NEděláme v rané fázi

- **eBPF.** Mocné, ale zvyšuje footprint a komplexitu. Možná v v1.x.
- **Auto-discovery hostů.** Operátor explicitně instaluje agent. Žádný
  scan sítě.
- **Web UI v server repu.** REST API je dost. UI buď samostatný projekt
  později, nebo se použije Grafana.
- **Pluginy.** Pevně daný set kolektorů v binárce. Pluginy → ekosystém →
  bezpečnostní noční můra.
- **Distributed control plane.** Single-node Postgres + VM + Loki + server.
  Vertical scale je 100% OK pro 500 hostů.
