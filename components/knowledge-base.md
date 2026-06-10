# Knowledge Base: How the Brain Works

The vault is a compounding knowledge base. Every session leaves it richer than the last. This document explains the mechanics: how sources become wiki pages, how questions get answered from existing knowledge, and how the system stays healthy over time.

---

## 1. The Compounding Loop

This is the whole point. Four moves, repeating:

```
Drop source into .raw/
  -> "ingest it"
    -> Claude creates wiki pages + cross-links
      -> future questions answered from those pages
        -> "save this" files new insights
          -> loop again
```

**Ingest** turns raw material into structured wiki pages. A single source typically touches 8-15 pages: a source summary, entity pages (people, orgs, products), concept pages (ideas, frameworks), and updates to domain indexes, the master index, and the hot cache.

**Query** retrieves from what was already filed. Claude reads hot.md first, then narrows to the relevant domain, then reads the specific page. No duplicate research for things already known.

**Save** captures insights generated during a conversation that are worth keeping. Non-obvious conclusions, decisions with rationale, analyses that took effort. The conversation was ephemeral; the wiki note is permanent.

**Lint** periodically catches decay: orphaned pages no longer linked from anywhere, dead wikilinks, stale claims.

After 20 ingests the vault answers questions it could not answer after 5. After 100 it answers questions you forgot you asked. That is what compounding means here.

---

## 2. Domain-First Structure

The wiki is organized by domain, not by source or date.

```
wiki/
  index.md              master page registry
  hot.md                hot cache (recent context, ~500 words)
  log.md                ingest + save audit trail
  overview.md           big-picture synthesis
  <domain>/
    _index.md           domain sub-index + brief descriptions
    <topic>.md          individual pages
  sources/              source summaries (one per ingested file)
  concepts/             ideas, frameworks, patterns
  entities/             people, orgs, products, repos
  questions/            saved synthesis + Q&A
  meta/                 decisions, session notes
```

Every wiki page carries YAML frontmatter:

```yaml
---
type: concept
title: "Compounding Vault Pattern"
created: 2026-05-01
updated: 2026-06-10
tags:
  - knowledge-management
  - llm-wiki
status: developing
related:
  - "[[wiki-ingest]]"
  - "[[hot cache]]"
---
```

Wikilinks (`[[Note Name]]`) connect pages. Cross-references are created during ingest, not retroactively. The more links, the faster query traversal works.

---

## 3. Reading Order Claude Uses

Context is not free. Claude follows a strict reading ladder to keep token costs low:

| Step | File | When to go deeper |
|------|------|-------------------|
| 1 | `wiki/hot.md` | Always first. If the answer is here, stop. |
| 2 | `wiki/index.md` | Locate the relevant domain and page names. |
| 3 | `wiki/<domain>/_index.md` | Narrow to the right sub-area. |
| 4 | Individual page | Only when steps 1-3 are insufficient. |

**Why this order:** hot.md is ~500 words covering the last few sessions. It catches the most common follow-up questions with minimal token spend. index.md is a directory, not content -- fast to scan. Individual pages average 100-300 lines; reading 10+ per query defeats the purpose.

Prompt you can paste into another project's CLAUDE.md to wire this up:

```markdown
## Wiki Knowledge Base
Path: /path/to/your-vault

When you need context not already in this project:
1. Read wiki/hot.md first (recent context, ~500 words)
2. If not enough, read wiki/index.md
3. If you need domain specifics, read wiki/<domain>/_index.md
4. Only then read individual wiki pages

Do NOT read the wiki for general coding questions or things already in this project.
```

---

## 4. Skills That Drive It

All skills live under `skills/`. See [skills-and-commands](skills-and-commands.md) for full trigger lists and options.

| Skill | One-line purpose |
|-------|-----------------|
| `ingest [source]` | Read a source, extract entities/concepts, create wiki pages, update index + hot. |
| `query: [question]` | Answer from existing wiki content using the reading ladder. |
| `/save` | File the current conversation as a structured note (synthesis, concept, decision, or session). |
| `lint the wiki` | Health check: orphans, dead wikilinks, stale claims, missing frontmatter. |
| `/autoresearch [topic]` | Fan out web searches, fetch sources, synthesize, file -- no manual source dropping needed. |
| `/canvas` | Add images, PDFs, and notes to an Obsidian canvas for visual synthesis. |

---

## 5. The MEMORY System

Memory is separate from the wiki. The wiki stores knowledge about the world; memory stores facts about you and your working patterns that Claude needs across every session.

**Location:** `~/.claude/projects/<project-hash>/memory/`

**Structure:**
- `MEMORY.md` -- one-line index, loaded automatically at session start
- One file per fact, named by topic

**Frontmatter schema:**

```yaml
---
name: gtm-copy-positioning
description: "Brand voice/positioning for all the company GTM copy"
metadata:
  node_type: memory
  type: feedback          # user | feedback | project | reference
  originSessionId: <uuid>
---
```

**Real example** (`memory/gtm-copy-positioning.md`):

> In all the company GTM copy, the author speaks **as the founder/auditor of the company** -- the person who audits HubSpot portals and builds the system Solutions Partners use -- talking **to** agency owners, RevOps, and HubSpot partners. Never pose as an agency owner / one of the audience.
>
> **Why:** On 2026-06-10 a LinkedIn draft was rejected because its close ("I'll tell you where *mine* would break") cast the author as an agency owner with a retainer.

This is exactly the kind of fact that belongs in memory: non-obvious, applies to every GTM session, would cause a mistake if forgotten. It is not in the wiki because the wiki is knowledge about HubSpot and GTM strategy -- not about how you want to be written for.

**Rule of thumb:**
- Save to memory: preferences, voice/tone constraints, standing decisions, non-obvious personal context
- Save to wiki: facts about the world, research findings, domain knowledge, source summaries
- Do not duplicate what git or code already records (API keys live in `.env`; architecture decisions live in `docs/`)

`type` field guide:

| type | Use for |
|------|---------|
| `user` | Preferences, working style, personal context |
| `feedback` | Corrections Claude made and should not repeat |
| `project` | Standing decisions specific to this project |
| `reference` | Frequently-needed lookups (config values, IDs, etc.) |

---

## 6. The Hot Cache Lifecycle

`wiki/hot.md` is a rolling ~500-word cache of recent context. It is what makes returning to an active workstream instant instead of slow.

**SessionStart hook** (from `hooks/hooks.json`):

```json
{
  "type": "command",
  "command": "[ -f wiki/hot.md ] && cat wiki/hot.md || true"
},
{
  "type": "prompt",
  "prompt": "If a vault is configured for this session ... silently read wiki/hot.md to restore recent context. ... Do not announce this. Just have the context available."
}
```

Claude reads `wiki/hot.md` silently at session start. No prompt needed. No delay.

**PostCompact hook**: context compaction wipes injected context. The PostCompact hook re-prompts Claude to re-read hot.md immediately after compaction so the cache is always live.

```json
{
  "type": "prompt",
  "prompt": "Hook-injected context does not survive context compaction. If wiki/hot.md exists ... silently re-read it now to restore the hot cache. Do not announce this."
}
```

**Stop hook**: when wiki pages changed during the session, the Stop hook emits a reminder:

```
WIKI_CHANGED: Wiki pages were modified this session. Please update wiki/hot.md
with a brief summary of what changed (under 500 words). Use the hot cache format:
Last Updated, Key Recent Facts, Recent Changes, Active Threads.
Keep it factual. Overwrite the file completely. It is a cache, not a journal.
```

This fires only when `git diff --name-only HEAD` contains `wiki/` paths -- no false positives on read-only sessions.

**Hot cache format:**

```markdown
## Last Updated
2026-06-10

## Key Recent Facts
- [Most important current context]

## Recent Changes
- [Page created/updated and why]

## Active Threads
- [Open questions or work in progress]
```

Overwrite completely each time. It is a cache, not a journal. Old entries belong in `wiki/log.md`.

**PostToolUse auto-commit** (also in `hooks/hooks.json`): after any Write or Edit, the hook runs `git add wiki/ .raw/ .vault-meta/` and commits with `wiki: auto-commit <timestamp>` -- but only if no wiki-lock is held. This prevents torn commits during multi-page ingest.

---

## 7. Health: Linting the Wiki

Run "lint the wiki" every 10-15 ingests. What it checks:

| Check | What it finds |
|-------|--------------|
| Orphan pages | Pages not linked from any index or other page |
| Dead wikilinks | `[[Page]]` references that point to non-existent files |
| Missing frontmatter | Pages without required `type`, `title`, `created` fields |
| Stale claims | Pages with `status: stale` or `updated:` older than 90 days with `status: developing` |
| Index gaps | Pages in `wiki/` not listed in `wiki/index.md` |

Quick prompt:

```
lint the wiki
```

The skill reports findings by severity. Fix BLOCKERs (dead links, missing indexes) before the next ingest batch. LOW findings (minor gaps) can accumulate until the next lint pass.

---

## Related docs

- [obsidian-setup](obsidian-setup.md)
- [skills-and-commands](skills-and-commands.md)
- [hooks](hooks.md)
- [README](README.md)
