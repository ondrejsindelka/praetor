# Project context for Claude Code

You are working on **Praetor**, a self-hosted observability + security
platform with native MCP for LLM agents.

## Current repo

This is the `praetor` umbrella repo. It contains:

- `PROJECT.md` — public-facing brief (synced from the vault)
- `ARCHITECTURE.md` — high-level architecture (synced from the vault)
- `ROADMAP.md` — milestones (synced from the vault)
- `README.md` — what people see on GitHub
- `docs/` — public design docs
- `.github/` — shared workflows

This repo does NOT contain code. Code lives in sibling repos.

## Sibling repos (all under `github.com/ondrejsindelka/`)

- `praetor-proto` — protobuf wire protocol (source of truth)
- `praetor-agent` — Go agent installed on monitored hosts
- `praetor-server` — Go control plane
- `praetor-mcp` — TypeScript MCP server
- `praetor-install` — bash install script + packaging

When asked to work in those, expect to `cd ../praetor-<name>`.

## Knowledge base — READ THIS FIRST

The project's working source of truth — design docs, ADRs, specs, meeting
notes — lives in my Obsidian vault at:

```
~/obsidian/Projects/Praetor/
```

**Before starting any non-trivial task**, read at minimum:

1. `~/obsidian/Projects/Praetor/00-MOC.md` — index
2. `~/obsidian/Projects/Praetor/PROJECT.md` — project brief
3. `~/obsidian/Projects/Praetor/Roadmap.md` — to know which milestone we're on
4. Any relevant ADR in `~/obsidian/Projects/Praetor/Decisions/`

The git-tracked `PROJECT.md` / `ARCHITECTURE.md` / `ROADMAP.md` in this
repo are *snapshots* for public consumption. The vault is the *working*
source. If they diverge, vault wins, and you should propose updating the
git-tracked file.

When you write a new ADR or spec, write it in the vault first, then mirror
the relevant portion to git when ready.

## Coding standards

See `PROJECT.md` § Coding standards. Briefly:

- **Go**: Go 1.22+, `slog` JSON, `golangci-lint` clean, errors wrapped,
  context propagated, table-driven tests, package doc comments.
- **TypeScript**: Node 20+, ESM, strict TS, pnpm, eslint + prettier, Zod
  for runtime validation.
- **Bash**: `set -euo pipefail`, shellcheck clean, all vars quoted, all
  paths absolute.

## Working principles

1. **Reference milestones explicitly.** "We're on M1, today we finish
   the enrollment flow."
2. **Small PRs.** One milestone item per PR. Skeleton → real impl →
   tests in separate PRs is fine.
3. **Update the vault as decisions evolve.** New ADR for any non-obvious
   decision. Update existing ADR's status to Superseded if changed.
4. **Audit log mindset.** This project handles security-sensitive
   operations. When in doubt, log more, default to deny, ask the human.
5. **Don't skip M0/M1 fundamentals to get to M3 cool stuff.** The boring
   foundation (enrollment, mTLS, identity, audit) is what makes the cool
   stuff possible.

## What NOT to do without asking

- Change `.proto` files in incompatible ways (use `buf breaking` to check).
- Add new dependencies without weighing supply chain risk.
- Add features outside the current milestone scope.
- Make any repo public (decided in M6, not now).
- Add LLM API calls outside the triage worker (we have one place for
  Anthropic/local LLM, don't sprinkle calls everywhere).

## Communication

I (Ondřej) often write in Czech. You can respond in Czech for chat, but
**all code, comments, commit messages, docs, PR titles, and ADR/spec
content stay in English** — this is open source.
