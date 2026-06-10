# Documentation library — build & operate the vault

This is the full documentation for the **Obsidian vault + Claude Code GTM command center**: a second
brain that compounds *and* a set of tools Claude Code runs for you. Everything here is tool-agnostic
where the concept allows — it talks in **roles** (CRM, newsletter, outbound, lead data, meeting
notes) and uses this repo's actual stack as the worked example.

## Start here

- **[build-a-gtm-command-center.md](build-a-gtm-command-center.md)** — the self-serve playbook. The
  whole system in phases, with copy-paste prompts. Read this first; the docs below go deep on each part.

## The build, step by step

| Area | Doc | What it covers |
|---|---|---|
| **Obsidian** | [obsidian-setup.md](obsidian-setup.md) | Install Obsidian, open the vault, the folder map, frontmatter + wikilinks, templates, recommended plugins, the claude-obsidian plugin |
| **The brain** | [knowledge-base.md](knowledge-base.md) | Ingest / save / query / lint loop, domain-first structure, the memory system, the hot-cache lifecycle, the flywheel |
| **The hands** | [tool-layer.md](tool-layer.md) | The config-driven CLI architecture: `config.py`, the HTTP client, `doctor`, sub-commands, shims, secrets |
| **Add a tool** | [adding-a-tool.md](adding-a-tool.md) | Step-by-step: wire a new tool (API or official CLI) as an adapter, with a template and tests |
| **Research** | [research-engine.md](research-engine.md) | RSS aggregation, the config-driven feed registry, ranking, and the freshness / wire-date verification fix |
| **Review UI** | [browser-review-loop.md](browser-review-loop.md) | The human-in-the-loop browser approval pattern: server, single template, verdict-to-disk, branding, gotchas |
| **Voice** | [voice-and-guardrails.md](voice-and-guardrails.md) | The voice spec, the live tone meter, honesty + positioning rules, the content log |

## Claude Code internals

| Doc | What it covers |
|---|---|
| [skills-and-commands.md](skills-and-commands.md) | Slash commands vs skills, `SKILL.md` structure, `skill-rules.json` triggers + enforcement, namespacing (`/ops-*`), progressive disclosure, the 500-line rule, how to add one |
| [hooks.md](hooks.md) | The hook events, the *actual* hooks in this setup (skill activation, hot-cache inject, auto-commit, update reminders), `settings.json` vs plugin `hooks.json`, how to add one |

## Security

- **[security.md](security.md)** — secrets handling, the approve-gate for outward actions, read-vs-write,
  the localhost-only review server, PII in deal folders, git auto-commit caution, key rotation, the
  permission allowlist, and prompt-injection from ingested sources.

## The principles behind all of it

1. **Config-driven, never hardcoded** — secrets, endpoints, ports, thresholds, word lists, feeds.
2. **Modular** — one module per tool/responsibility; files under ~300 lines.
3. **Verify before trust** — dates lie (syndication vs original), data lies (a messy CRM).
4. **Confirm before outward-facing actions** — the review page is the gate; read-only is free.
5. **Single reusable templates** — design once, feed data many times.
6. **Plain files** — markdown + JSON on disk; portable, diffable, Obsidian-browsable.
7. **Honesty** — spicy never means fake; never invent facts, numbers, or stories.
