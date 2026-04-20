---
title: TOPIC TITLE HERE
section: domain
status: active
last_verified: YYYY-MM-DD
verified_by: your-name
code_anchors:
  - path: path/relative/to/repo/root.ts
    symbol: ClassName
  # Each entry MUST be verified (Glob path, Grep symbol) before saving.
  # `symbol` is optional — omit when the whole file is the anchor.
related: []                        # other KB entry paths relative to kb_root
source_of_truth:
  some_field: "Where the canonical value lives (DB table, seed, config path, …)"
---

# Human Title Of The Topic

## What it is

One paragraph. What this is, who uses it, why it exists. No jargon without a gloss.

## Key invariants

The non-obvious rules a developer would break without this doc. This is the highest-signal section.

1. **Rule stated plainly.** Brief reason it matters / what breaks if ignored.
2. **Another rule.** Same shape.

Skip anything obvious from reading the code. Focus on constraints that are implicit, load-bearing, or historically bite people.

## Architecture

Compact prose or ASCII. Orientation, not duplication — link to `code_anchors` for implementation. Do not paste large code blocks.

```
┌─ Layer A ──────────────┐
│ does thing X           │
└────────┬───────────────┘
         ▼
┌─ Layer B ──────────────┐
│ does thing Y           │
└────────────────────────┘
```

## Mini recipe

The common "I need to change something here" task, distilled to numbered steps.

```
1. Do X in file A
2. Run command Y
3. Verify in file B
```

## Gotchas

Traps, counter-intuitive behavior, cache quirks, race conditions.

- **Trap name**: what it is, how to avoid it.
- **Another trap**: same.

## Open questions / known stale

Unverified claims or content that may have drifted. Better flagged than silently wrong. Date every note.

- **YYYY-MM-DD**: <note — what you're unsure about, what would confirm it>

## See also

- `section/related-topic` — 1-line reason it's related
