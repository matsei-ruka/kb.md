# kb

**A Claude Code skill for keeping a small, code-anchored knowledge base in sync with your repo.**

Atomic markdown entries. Frontmatter anchored to real files and symbols. Every anchor verified before it's written. Freshness tracked. One command: `/kb`.

Drop into any project. Survives refactors. Doesn't rot.

---

## Why

RAG pipelines re-derive knowledge on every query. Docs drift silently the moment the code changes. Neither scales with the pace at which an AI coding agent changes a codebase.

`kb` takes a different approach: a persistent, compounding knowledge base that is **written by the LLM, curated by you**, and **pinned to the code**. Every claim points to a real file and symbol; the skill verifies those anchors automatically and refuses to write hallucinated ones. When the code changes, the skill loads the relevant entries first and offers an update after.

Influences:
- [Andrej Karpathy — "LLM Wiki"](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) — incrementally-maintained wiki over ad-hoc RAG.
- The code-anchored KB pattern used internally at [Scovai](https://scovai.com), extracted and made generic.

---

## What you get

- **Atomic entries** — one topic per file, ~150 lines. Small enough to read in full.
- **Code anchors** — every claim in the KB points at a real `{path, symbol}`. The skill `Glob`s and `Grep`s before writing.
- **Freshness tracking** — `last_verified` dates; configurable staleness threshold; `/kb verify` audits the whole KB.
- **Source ingestion** — pull in external docs, PDFs, articles via `/kb ingest`; one source can update many entries.
- **Cited answers** — `/kb ask` synthesizes from the KB and cites every claim. Does not auto-file.
- **Linting** — orphans, contradictions, stale entries, missing cross-refs.
- **Proactive loading** — before you edit a file that a KB entry references, the skill loads that entry first.

---

## Install

```bash
# project-local (recommended)
git clone https://github.com/matsei-ruka/kb.md.git .claude/skills/kb

# OR user-level (available in every repo)
git clone https://github.com/matsei-ruka/kb.md.git ~/.claude/skills/kb
```

Claude Code picks it up automatically. The command is `/kb`.

---

## Quick start

```
/kb init
```

Creates `docs/kb/kb-config.yaml`, `docs/kb/INDEX.md`, and a starter section folder.

```
/kb new rate-limiter
```

Walks you through creating a new entry, verifies every anchor before writing, and appends a row to `INDEX.md`.

```
/kb verify
```

Audits every entry in the KB: anchors OK, last verified date, staleness.

```
File                        Anchors OK   Last verified   Status
subsystem/rate-limiter.md   3/3          2026-04-20      OK
domain/auth.md              4/5          2025-12-02      STALE + BROKEN ANCHOR
```

---

## Verbs

| Verb                   | Purpose                                                     |
|------------------------|-------------------------------------------------------------|
| `/kb init`             | Bootstrap config + KB skeleton                              |
| `/kb read <topic>`     | Load an entry, verify anchors, check freshness              |
| `/kb new <topic>`      | Create an entry; verify before write; update INDEX          |
| `/kb update <topic>`   | Surgical edit; bump `last_verified` only if re-read         |
| `/kb verify [topic]`   | Non-editing audit of anchors + staleness                    |
| `/kb ingest <source>`  | Summarize an external source into the KB (may touch many files) |
| `/kb ask <question>`   | Answer from the KB with citations; does not auto-file       |
| `/kb lint`             | Health check: orphans, contradictions, stale, xrefs         |
| `/kb log [N]`          | Tail `LOG.md`                                               |

---

## What a KB entry looks like

```markdown
---
title: Request Rate Limiter
section: subsystem
status: active
last_verified: 2026-04-20
verified_by: a-maintainer
code_anchors:
  - path: src/middleware/rateLimiter.ts
    symbol: rateLimiter
  - path: src/middleware/rateLimiter.ts
    symbol: RATE_LIMIT_TIERS
related:
  - subsystem/auth.md
source_of_truth:
  tier_definitions: "src/middleware/rateLimiter.ts :: RATE_LIMIT_TIERS"
---

# Request Rate Limiter

## Key invariants

1. **Redis outage fails open, not closed.** Losing Redis briefly should not take
   down the API.
2. **Tier changes require a rolling restart.** Buckets initialize from
   `RATE_LIMIT_TIERS` at boot; changing the const without restart silently keeps
   old limits.

## Gotchas

- **First-request spike**: a cold bucket allows a full burst immediately.
  Pre-warm with a scripted ping on deploy if it matters.

## Open questions / known stale

- **2026-03-11**: `x-forwarded-for` trust is configured at the LB; LB config
  isn't in this repo. Confirm with platform team before changing trust rules.
```

See [`examples/entry-example.md`](examples/entry-example.md) for the full version.

---

## Config

Lives at `<kb_root>/kb-config.yaml` (written by `/kb init`):

```yaml
kb_root: docs/kb
sections:
  - domain
  - subsystem
  - contract
  - decision
  - source
staleness_days: 60
anchor_verification: strict    # strict | warn | off
languages: [en]
log: true
lint:
  orphans: true
  contradictions: true
  missing_xrefs: true
  stale: true
verified_by: ""                # empty → falls back to `git config user.name`
```

The skill fails loudly if this file is missing or malformed. There is no fallback to built-in defaults.

---

## Frontmatter contract

**Required**

| Field           | Type                           | Notes                                         |
|-----------------|--------------------------------|-----------------------------------------------|
| `title`         | string                         |                                               |
| `section`       | string                         | must match one of `config.sections`           |
| `code_anchors`  | list of `{path, symbol?}`      | may be empty; every entry verified before write |
| `last_verified` | ISO date (`YYYY-MM-DD`)        |                                               |
| `verified_by`   | string                         |                                               |

**Optional**

| Field             | Type                           | Notes                                          |
|-------------------|--------------------------------|------------------------------------------------|
| `status`          | `draft \| active \| stale \| archived` | default `active`                       |
| `related`         | list of KB entry paths         | relative to `kb_root`                          |
| `sources`         | list of `{url \| path, title}` | for `/kb ingest` material                      |
| `source_of_truth` | map `field → canonical location` | DB table, seed, config path, …               |

`/kb verify` enforces this contract.

---

## Hard rules

1. **Verify every anchor before writing it.** Glob path, Grep symbol. Hallucinated anchors destroy trust in the system faster than anything else.
2. **Never delete KB content silently.** Wrong or drifted content moves to `## Open questions / known stale` with today's date.
3. **Only bump `last_verified` when you actually re-read the code.** It's not a formality.
4. **Prefer honest uncertainty over false confidence.** `## Open questions` is a first-class section.
5. **Config is authoritative.** Missing or malformed `kb-config.yaml` stops execution — no fallback to defaults.
6. **One `kb_root` per repo.**

---

## Proactive behavior

Before you edit a file that appears in any entry's `code_anchors`, the skill loads that entry first. After the change, it offers `/kb update` before reporting the task done.

This is the feature that makes the system pay for itself: you can't drift away from the KB without the skill noticing.

---

## Layout

```
kb/
├── kb.md                  # skill definition
├── README.md              # this file
├── templates/
│   ├── kb-config.yaml     # default config written by /kb init
│   ├── INDEX.md           # starter index
│   ├── entry.md           # template for code-anchored entries
│   └── source-entry.md    # template for ingested external sources
└── examples/
    └── entry-example.md   # filled-in example
```

---

## Non-goals

- No CI verifier — project-side concern; wire it around `/kb verify` if you want it.
- No auto-migration from existing KB formats.
- No web UI, graph view, or search index.
- One `kb_root` per repo (v1).

---

## License

MIT.
