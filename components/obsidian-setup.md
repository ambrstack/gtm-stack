# Obsidian Setup Guide

This vault is a plain markdown folder that doubles as a Claude Code GTM command center. Obsidian is an optional UI layer on top of it. Claude reads and writes files directly, so the system works without Obsidian. Open it in Obsidian if you want a visual graph, backlink navigation, and template shortcuts.

---

## 1. What Obsidian Is (and Why It Fits)

Obsidian is a local markdown editor. It reads a folder on disk, renders `[[wikilinks]]`, and shows a graph of connections between notes. It stores no data outside the folder. There is no sync account required.

This vault is already a folder of `.md` files. Obsidian simply makes them easier to browse. All Claude skills (`/wiki`, `/save`, `/autoresearch`, etc.) write plain files that Obsidian can display immediately.

**Obsidian is optional.** If you only use Claude Code via terminal, nothing breaks. The hot-cache hook, auto-commit, and all GTM commands work without Obsidian open.

---

## 2. Install and Open

1. Download Obsidian (free) from [obsidian.md](https://obsidian.md).
2. Open Obsidian. Click **Open folder as vault**.
3. Navigate to this directory (`your-vault`) and confirm.
4. Obsidian indexes the folder. It may prompt you to trust the vault. Click **Trust author and enable plugins** if prompted.

No additional configuration is needed to start reading notes. Community plugin setup is covered in section 6.

---

## 3. Folder Map

```
your-vault/
├── .raw/                   # Source documents — immutable. Claude reads, never modifies.
├── wiki/                   # Claude-generated knowledge base (domain-first layout)
│   ├── hot.md              # Hot cache — recent context injected at session start
│   ├── index.md            # Master index of all wiki pages
│   ├── log.md              # Chronological change log
│   ├── core/               # Domain: Core / platform overview
│   ├── growth/             # Domain: Growth / GTM / outbound / ABM
│   ├── service/             # Domain: Service service
│   ├── projects/           # Domain: Projects (beta)
│   └── meta/               # Vault meta pages (transport, conventions, etc.)
├── _attachments/           # Images and PDFs referenced by wiki pages
├── _templates/             # Obsidian Templater templates (see section 4)
├── docs/                   # This guide and other reference docs
├── gtm/                    # GTM CLI source (Python package)
├── hooks/                  # Claude Code hook definitions (hooks.json)
└── skills/                 # Claude Code skill definitions
```

**Rule of thumb:** Drop sources into `.raw/`. Read and write only in `wiki/`. Everything else is infrastructure.

---

## 4. Conventions

### YAML Frontmatter

Every wiki page opens with a frontmatter block:

```yaml
---
title: Page Title
type: overview | reference | runbook | note | cache
domain: core | growth | service | projects | meta
tags: [tag1, tag2]
status: active | draft | archived
created: YYYY-MM-DD
---
```

`type` and `domain` drive Dataview queries. `status: archived` signals pages that are out of date but preserved.

### Wikilinks

Use `[[Page Name]]` to link between pages. Obsidian resolves these to files in the vault. Claude also reads them as references when traversing the graph.

Display aliases: `[[Page Name|Display Text]]`.

### Tags

Use `#tag` inline or the `tags:` frontmatter list. Common tags in this vault: `#gtm`, `#hubspot`, `#outbound`, `#abm`, `#audit`, `#onboarding`.

### Templates

The `_templates/` folder contains these Templater templates:

| File | Purpose |
|------|---------|
| `source.md` | Template for ingesting a new `.raw/` source document |
| `concept.md` | Atomic concept note |
| `entity.md` | Named entity (person, company, product) |
| `question.md` | Open question to resolve |
| `comparison.md` | Side-by-side comparison of two or more things |

To use in Obsidian: install the Templater plugin (section 6), then use the command palette (`Cmd+P`) and type "Templater: Insert template".

---

## 5. The Hot Cache

`wiki/hot.md` is a short (~500-word) rolling summary of recent vault activity. It is the first thing Claude reads at the start of every session.

### Format

```markdown
---
title: Hot Cache
type: cache
created: YYYY-MM-DD
---

# Hot Cache

**Last Updated:** YYYY-MM-DD

## Key Recent Facts
- Bullet list of standing facts Claude should always have in context.

## Recent Changes
- YYYY-MM-DD: What changed and why.

## Active Threads
- Open items, pending decisions, or follow-ups that carry across sessions.
```

### How the Hook Works

The `hooks/hooks.json` file registers a `SessionStart` hook. On startup, it runs:

```bash
[ -f wiki/hot.md ] && cat wiki/hot.md || true
```

This pipes `hot.md` into Claude's context before any user input. A companion `prompt` hook tells Claude to silently absorb the content without announcing it. A `PostCompact` hook re-injects `hot.md` after context compaction (which would otherwise flush injected context).

At the end of a session, the `Stop` hook checks whether any wiki pages changed. If they did, it prompts Claude to update `hot.md` with a brief summary using the format above. Hot cache is a cache, not a journal. Overwrite it completely each time.

---

## 6. Recommended Community Plugins

Open **Settings > Community plugins > Browse** to install.

| Plugin | Why |
|--------|-----|
| **Dataview** | Query frontmatter. Run `TABLE status, domain FROM "wiki"` to audit coverage. |
| **Templater** | Activate the `_templates/` folder. Required for template insertion shortcuts. |

**claude-obsidian plugin:** The skills in this vault (`/wiki`, `/save`, `/autoresearch`, etc.) are Claude Code skills, not Obsidian plugins. They live in the `skills/` folder and are invoked from the Claude Code terminal, not from Obsidian. No Obsidian plugin installation is needed for the Claude side to work.

**Optional transport plugins:** The transport layer (`scripts/detect-transport.sh`) can use `mcp-obsidian` or `mcpvault` if installed. The filesystem fallback always works without them. See `wiki/references/transport-fallback.md` for the full decision tree.

---

## 7. How Claude Uses the Vault

Claude follows a read-depth ladder to avoid unnecessary file I/O:

```
1. wiki/hot.md         — always injected at session start via hook
2. wiki/index.md       — master page list, read when hot.md is not enough
3. wiki/<domain>/_index.md  — domain overview, read when narrowing to a topic
4. individual wiki pages    — read only when the question requires specifics
```

**Writing new pages:** Claude uses the `/wiki-ingest` skill for source ingestion and `/save` for conversation filing. Before writing, the transport layer (`scripts/detect-transport.sh`) picks the best available write path: Obsidian CLI, mcp-obsidian, mcpvault, or direct filesystem.

**Auto-commit:** The `PostToolUse` hook in `hooks/hooks.json` fires after every `Write` or `Edit` tool call. It stages changes under `wiki/`, `.raw/`, and `.vault-meta/`, then commits with the message `wiki: auto-commit YYYY-MM-DD HH:MM`. The commit is skipped if a `wiki-lock.sh` lock is held (active multi-writer ingest in progress). This keeps the git history clean without manual commits. See [hooks](hooks.md) for the full hook reference.

**Copy-paste prompts to try:**

```
ingest quarterly-review.pdf
```

```
query: What is the current audit coverage score and where is the authoritative source?
```

```
lint the wiki
```

---

## Related Docs

- [knowledge-base](knowledge-base.md)
- [hooks](hooks.md)
- [README](README.md)
