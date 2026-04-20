# Knowledge Base — Index

> **Purpose**: a small, current map of how this project actually works.
> Load only the files you need for the task. The `code_anchors` in each file's
> frontmatter tell you what code the entry depends on; re-grep before acting
> on anything you read here.
>
> **Last rebuilt**: YYYY-MM-DD

## How to use

1. Find your topic in the tables below.
2. Read the matching file in full — they are small (~150 lines).
3. Re-grep the `code_anchors` before acting on what you read.
4. Stale or wrong? Use `/kb update`. Never silently delete — move to `## Open questions / known stale`.

## Topic map

<!-- One table per section defined in kb-config.yaml.
     `/kb new` appends a row to the matching table.
     Remove sections you don't use, and add any custom sections you define. -->

### domain

| Topic | File | What's inside |
|-------|------|---------------|

### subsystem

| Topic | File | What's inside |
|-------|------|---------------|

### contract

| Topic | File | What's inside |
|-------|------|---------------|

### decision

| Topic | File | What's inside |
|-------|------|---------------|

### source

| Topic | File | What's inside |
|-------|------|---------------|

## File contract

See the `kb` skill's README for the frontmatter contract and hard rules. In short:

- Required frontmatter: `title`, `section`, `code_anchors`, `last_verified`, `verified_by`.
- Every `code_anchors` entry is verified (Glob + Grep) before being written.
- Wrong content is never silently deleted — it moves to `## Open questions / known stale` with a dated note.
- `last_verified` is bumped only when the code was actually re-read.
