---
name: kb
description: |
  Atomic, verified knowledge base. Invoke when:
  (a) the user types `/kb ...`,
  (b) the user asks to read, update, create, verify, ingest into, or ask the KB,
  (c) you are ABOUT to edit files that appear in any entry's `code_anchors` — load the relevant KB first,
  (d) you just finished such a change — propose `/kb update` before reporting done.
  Do NOT use this skill for general documentation, CLAUDE.md edits, or anything outside `<kb_root>`.
version: 0.1.0
allowed-tools: [Read, Write, Edit, Glob, Grep, Bash]
---

# KB — Atomic Knowledge Base

Operates on the atomic knowledge base rooted at `<kb_root>`, resolved from `<kb_root>/kb-config.yaml`. The goal is **small, verified, updated-in-the-moment** entries that Claude and humans can trust.

**Before any verb except `init`, read `<kb_root>/kb-config.yaml` and `<kb_root>/INDEX.md` first.** If the config is missing or malformed, fail loudly and tell the user to run `/kb init`. Never fall back to built-in defaults.

## Config resolution

1. `Glob "**/kb-config.yaml"` from the repo root. Exactly one match expected.
2. The file's parent directory is `kb_root`. No other resolution path.
3. Zero or multiple matches → stop and ask the user.

## Commands

Invoked as `/kb <verb> [args]`.

### `/kb init [--force]`

Bootstrap a new KB.

1. Ask the user where `kb_root` should live (default: `docs/kb`).
2. Refuse if `<kb_root>/` already exists, unless `--force`. Never overwrite an existing config.
3. Create:
   - `<kb_root>/kb-config.yaml` from `templates/kb-config.yaml`
   - `<kb_root>/INDEX.md` from `templates/INDEX.md` (stamp today's date)
   - One empty placeholder section folder (first entry in `config.sections`)
4. Report every path created.

### `/kb read <topic-or-question>`

Load the relevant entry (or entries).

1. Read `<kb_root>/INDEX.md` and find matching file(s).
2. Read each in full.
3. **Verify anchors** per `config.anchor_verification`:
   - For each `code_anchors` entry: `Glob <path>` must match; if `symbol` is set, `Grep "<symbol>" <path>` must match at least once.
   - `strict` → any failure is an error.
   - `warn` → list failures, continue.
   - `off` → skip.
4. Compare each entry's `last_verified` to today. Older than `config.staleness_days` → warn inline.
5. Report: `Loaded <files>. Anchors: N/M verified. Last verified: YYYY-MM-DD.`

If the topic is not in INDEX.md, say so and offer `/kb new <topic>` or `/kb ask <question>`. Do not guess.

### `/kb new <topic>`

Create a new entry.

1. Ask the user which `section` (must equal one of `config.sections`).
2. Pick the template:
   - `templates/entry.md` for code-anchored sections.
   - `templates/source-entry.md` when the section is `source` or the entry has no code to anchor.
3. Ask which sources, files, or code paths to draw from. Read them.
4. Copy the template to `<kb_root>/<section>/<topic-slug>.md`.
5. Fill frontmatter:
   - `title`, `section`, `status: active`
   - `last_verified` → today (`Bash date +%F`)
   - `verified_by` → `config.verified_by` if non-empty, else `Bash git config user.name`
   - Every `code_anchors` entry MUST be verified before being written (Glob path, Grep symbol). If verification fails under `strict`, stop and ask.
6. Fill body sections. Keep the entry to ~150 lines; hard-cap around 15KB.
7. Append a row to `<kb_root>/INDEX.md` under the matching section table.
8. If `config.log`, append one line to `<kb_root>/LOG.md` (format below).

### `/kb update <topic>`

Surgical edit of an existing entry.

1. Run `/kb read <topic>` first (loads + verifies).
2. Ask the user what changed in the code or understanding. Do NOT invent changes.
3. Compute the minimal diff. Apply with `Edit` — never rewrite the whole file when a surgical change will do.
4. If any anchor was broken (detected in step 1): fix path/symbol, re-verifying first.
5. If you are uncertain whether content is wrong or just drifted: add to `## Open questions / known stale` with today's date — do not silently rewrite.
6. Frontmatter:
   - Bump `last_verified` to today **only if you actually re-read the code**. Don't bump it for cosmetic edits.
   - Update `verified_by`.
7. If `config.log`, append one line to `<kb_root>/LOG.md`.

### `/kb verify [topic]`

Audit without editing.

1. If `topic` given: verify that single file. Otherwise: `Glob <kb_root>/**/*.md` excluding `INDEX.md` and `LOG.md`.
2. For each entry: verify every `code_anchors` entry (same checks as `/kb read` step 3).
3. Check frontmatter contract (required fields present; `section` in `config.sections`; dates ISO).
4. Compare `last_verified` to today.
5. Print a table:

   ```
   File                        Anchors OK   Last verified   Status
   domain/rate-limiter.md      3/3          2026-04-20      OK
   domain/auth.md              4/5          2025-12-02      STALE + BROKEN ANCHOR
   ```

6. Do not modify anything. The user fixes via `/kb update`.

### `/kb ingest <source>`

Summarize an external source (URL, local file, PDF) into the KB. A single ingest often touches multiple entries — surface all of them.

1. Read the source.
2. Read INDEX.md. Decide whether the material belongs in:
   - A new `source/` entry (pure-source summary), AND/OR
   - Updates to existing entries whose topic the source informs.
3. For new source entries: use `templates/source-entry.md`. Populate `sources:` frontmatter with `{url|path, title}`.
4. For updates to existing entries: surgical Edit. Preserve existing claims. If the source contradicts an existing claim, add a dated note to `## Open questions / known stale` rather than overwriting.
5. Propose every affected entry and wait for approval before writing.
6. If `config.log`, append one `ingest` line to `<kb_root>/LOG.md`.

### `/kb ask <question>`

Answer a question from the KB with citations.

1. Read INDEX.md. Identify candidate entries.
2. Read them. Synthesize an answer. Cite every claim with the KB file path and (when useful) the `code_anchors` entry it rests on.
3. **Do NOT file the answer back into the KB by default.** Only create or update an entry if the user explicitly says "file this" / "save this" / similar. In that case, route through `/kb new` or `/kb update` (with full verification).
4. If `config.log`, append one `query` line to `<kb_root>/LOG.md`.

### `/kb lint`

Health-check. Non-editing; returns a punch list.

1. **Orphans** (`config.lint.orphans`): entries not linked from INDEX.md and not listed in any other entry's `related:`.
2. **Contradictions** (`config.lint.contradictions`): entries sharing a `code_anchors.path` whose bodies make conflicting claims.
3. **Stale** (`config.lint.stale`): entries past `config.staleness_days`.
4. **Missing cross-refs** (`config.lint.missing_xrefs`): entries sharing an anchor but not in each other's `related:` list — suggest the addition.
5. Propose, do not apply. User runs `/kb update` for each fix.

### `/kb log [N]`

Read-only. Print the last N entries from `<kb_root>/LOG.md` (default 20).

## Log format

`<kb_root>/LOG.md` is append-only. One line per operation, Unix-parseable:

```
## [YYYY-MM-DD] <verb> | <target> — <one-line note>
```

Example: `## [2026-04-20] update | domain/rate-limiter.md — added Redis-outage fail-open invariant`

Write with `Bash printf '...' >> LOG.md`. Never rewrite history; never edit past entries.

## Frontmatter contract

Required:
- `title` — string
- `section` — must equal one of `config.sections`
- `code_anchors` — list of `{path, symbol?}`; may be empty (e.g. for pure-source entries). Every non-empty entry must verify.
- `last_verified` — ISO date (`YYYY-MM-DD`)
- `verified_by` — string

Optional:
- `status` — `draft | active | stale | archived` (default `active`)
- `related` — list of KB entry paths relative to `kb_root`
- `sources` — list of `{url|path, title}` for ingested external material
- `source_of_truth` — map of `field → canonical location` (DB table, seed, config path, …)

`/kb verify` enforces this contract.

## Hard rules (never violate)

1. **Verify every anchor before writing it.** `Glob` path, `Grep` symbol. Hallucinated anchors are the fastest way to destroy trust in this system.
2. **Never delete KB content silently.** Wrong or drifted content moves to `## Open questions / known stale` with today's date. The KB prefers honest uncertainty over false confidence.
3. **`last_verified` is not a formality.** Bump it only when you actually re-read the code or source and confirmed the claim.
4. **Frontmatter is always English and machine-readable.** Body language follows `config.languages`; be consistent within a file.
5. **If a change affects multiple entries, propose edits to all of them.** Don't stop at the first match.
6. **Config is authoritative.** Missing or malformed `kb-config.yaml` → stop. Never fall back to defaults.
7. **One `kb_root` per repo.** If `Glob` finds multiple `kb-config.yaml`, stop and ask the user to pick one.

## Proactive triggers

Beyond explicit `/kb ...` invocations, run `/kb read` automatically at the **start** of any task that touches a file listed in any entry's `code_anchors`. Quick check:

```
Grep -rn "code_anchors" <kb_root>/   → list of KB entries that reference code
Grep -rn "^\s*- path:" <those files> → anchored file paths
```

If the current task touches any of those paths, load the relevant entry before editing. At the **end** of such a task, offer `/kb update` before reporting done.

## Templates

Live in this skill's directory under `templates/`:
- `kb-config.yaml` — written by `/kb init` into `<kb_root>/`
- `INDEX.md` — written by `/kb init` into `<kb_root>/`
- `entry.md` — used by `/kb new` for code-anchored entries
- `source-entry.md` — used by `/kb new` / `/kb ingest` for source-anchored entries

Resolve paths relative to this skill directory. Never write templates anywhere else.
