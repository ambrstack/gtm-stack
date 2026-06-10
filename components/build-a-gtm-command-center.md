# Build a GTM Command Center in Claude Code + Obsidian — a self-serve playbook

This is the exact pattern behind an Obsidian vault that doubles as a go-to-market command center:
a second brain that compounds **and** a set of tools Claude Code runs for you. Drop into Claude
Code, follow the phases, paste the prompts. You'll have a working version the same day.

It's **tool-agnostic**. The examples use one stack (HubSpot, Customer.io, Smartlead, Apollo,
Granola), but the architecture cares about *roles*, not vendors. You swap adapters, not the system.
Use Salesforce or Attio, Mailchimp or Resend, Instantly or Lemlist, Clay or ZoomInfo, Fathom or
Fireflies — the playbook doesn't change.

> **Going deeper:** this is the overview. Each part has a dedicated deep-dive doc — see the
> [documentation library](README.md): [obsidian setup](obsidian-setup.md),
> [the knowledge base](knowledge-base.md), [the tool layer](tool-layer.md),
> [adding a tool](adding-a-tool.md), [the research engine](research-engine.md),
> [the browser review loop](browser-review-loop.md), [voice & guardrails](voice-and-guardrails.md),
> [skills & commands](skills-and-commands.md), [hooks](hooks.md), and [security](security.md).

---

## 0. The mental model (read this first)

You're building three layers that sit in one folder:

| Layer | What it is | In this build |
|---|---|---|
| **The brain** | Plain markdown notes that hold knowledge and *compound* every session | an Obsidian `wiki/` + `memory/` |
| **The hands** | A small, config-driven CLI that talks to your real tools | a Python `typer` package + shims on `PATH` |
| **The operator** | Claude Code — reads the brain, runs the hands, drafts the work, asks for your sign-off | the agent in your terminal |

Two ideas make it work:

1. **The 80/20 of AI ops.** The AI is the easy 20%. The 80% is the plumbing underneath: clean data,
   real connections to your tools, and rules that encode how you actually work. An agent on top of a
   messy CRM and three disconnected tools doesn't save time — it fails faster. Build the plumbing.
2. **Plain files + config, never hardcode.** Everything is markdown or JSON on disk, and every value
   that could change between people/tools/environments lives in config. That's what makes it
   portable, Obsidian-browsable, and easy for the next person (or the next tool) to adopt.

---

## Prerequisites

- **Claude Code** (CLI, desktop, or IDE extension). This is the operator.
- **A folder** for the vault. Git-initialized (`git init`).
- **Obsidian** (free) — optional, for visually browsing/editing the notes. The system works without it.
- **A runtime for the tools.** This build uses **Python 3.10+**; any language works (Node, Go…). The
  pattern below is shown in Python.
- **API keys** for whatever tools you use, plus the official CLIs for tools that ship one.
- Comfort letting Claude Code run shell commands and write files.

---

## The drop-in quickstart (≈10 minutes)

Open Claude Code in your empty vault folder and paste this. It scaffolds the brain and the operating
manual so every later session starts smart.

> **Bootstrap prompt — paste into Claude Code:**
>
> "Set up this folder as an Obsidian-style second brain that I'll also use as a GTM command center.
> Create: `.raw/` for immutable source files I drop in; `wiki/` for your generated knowledge,
> organized domain-first; `memory/` for cross-session facts with a `MEMORY.md` index; `_attachments/`
> for images/PDFs; and `wiki/hot.md` (a ~500-word 'hot cache' of recent context), `wiki/index.md`,
> and `wiki/log.md`. Then write a `CLAUDE.md` at the root that explains this structure and tells you
> to read `wiki/hot.md` first each session. Keep everything as plain markdown. Don't invent facts —
> leave placeholders where you need input from me."

Then, for the tools:

> "I want a config-driven CLI to run my GTM tools. Scaffold a Python package using `typer` and
> `pydantic-settings`: a `config.py` where every secret comes from a git-ignored `.env` and every API
> endpoint is overridable; a shared HTTP client with retry/backoff; one module per tool; and a
> `doctor` command that reports which tools are connected. Add a `.env.example` documenting every
> variable. Put a shim on my PATH so I can run it from anywhere. Start with just a `doctor` command
> and one tool — I'll tell you which."

From here, work phase by phase below. Each phase ends with the prompt to paste.

---

## Phase 1 — The vault skeleton (the brain)

**Goal:** a place where knowledge lands once and is reused forever.

```
your-vault/
├── .raw/            # source docs you drop in — immutable; Claude reads, never edits
├── wiki/            # Claude-generated knowledge, organized by domain
│   ├── hot.md       # the "hot cache": ~500 words of the most current context
│   ├── index.md     # the map of the wiki
│   └── log.md       # append-only activity log
├── memory/          # cross-session facts (see Phase 2)
├── _attachments/    # images / PDFs referenced by notes
└── CLAUDE.md        # the operating manual Claude reads every session
```

**The compounding loop** (this is the whole point):

1. Drop any source into `.raw/` → tell Claude *"ingest this."* It extracts entities/concepts, writes
   structured wiki pages, and cross-links them.
2. Ask any question → Claude reads `hot.md` → `index.md` → the relevant page, and answers with
   citations.
3. After a good session → *"save this"* → it files the insight as a note and updates the index/log.

Every session leaves the brain richer. That's the flywheel.

> **Prompt:** "Here's a source file in `.raw/`. Ingest it: pull out the key entities and concepts,
> write them as linked wiki pages under the right domain, and update `index.md`, `log.md`, and
> `hot.md`. Tell me what you created."

---

## Phase 2 — Memory + the operating manual (so Claude knows *you*)

Two mechanisms keep Claude consistent across sessions:

**`CLAUDE.md`** — read automatically every session. Put in it: what the vault is, the folder map, how
to use the tools, and your hard rules (e.g. *"confirm before any outward-facing action"*). Keep it
tight; it's loaded as context every time.

**`memory/`** — one fact per file, with frontmatter, plus a `MEMORY.md` one-line index that's loaded
each session. Four types:

| Type | Holds |
|---|---|
| `user` | who you are — role, preferences, expertise |
| `feedback` | how you want Claude to work (corrections + confirmed approaches) — always with the *why* |
| `project` | ongoing work/goals not derivable from the files |
| `reference` | pointers to external resources (URLs, dashboards) |

Save what was **non-obvious**. Don't duplicate what the code or git history already says. Example
from this build: *"write AS the founder/auditor talking TO agencies — never pose as an agency
owner"* — a positioning rule that would otherwise be re-learned the hard way every time.

> **Prompt:** "Remember this as a feedback memory, with the why: [your rule]. Add a one-line pointer
> to `MEMORY.md`."

---

## Phase 3 — The hands: a config-driven, tool-agnostic CLI

**Goal:** Claude can actually *do* things — query your CRM, pull meeting notes, search leads — through
one small toolkit where **swapping a vendor is a config edit, not a rewrite.**

The shape (Python shown; translate freely):

```
gtm/
├── .env                 # secrets — git-ignored, chmod 600
├── .env.example         # documents every variable
├── pyproject.toml       # deps + the `gtm` entry point
└── gtm/
    ├── cli.py           # wires sub-commands together + a `doctor`
    ├── config.py        # pydantic-settings: secrets from .env, endpoints overridable, tunables
    ├── http.py          # one HTTP client with timeout + retry/backoff
    └── tools/
        ├── crm.py       # one module ("adapter") per tool
        ├── outbound.py
        └── ...
```

**The non-negotiables** (these are what make it last):

- **Secrets only from env.** `config.py` validates them at startup; nothing hardcoded.
- **Endpoints are config.** A vendor changing an API path is a one-line `.env` override, not a code
  change.
- **Tunables are config.** Timeouts, retries, page sizes, batch sizes — named config values, no magic
  numbers.
- **One module per tool.** Single responsibility. Files under ~300 lines; split when they grow.
- **A `doctor` command.** It tells you, at a glance, which tools have a key and authenticate. This is
  the first thing you run.
- **Shims on `PATH`.** A 2-line shell script per command (`gtm`, and aliases like `deals`,
  `linkedin`) so you run them from anywhere. They just `exec` the venv binary.

**The adapter cheat sheet — map roles to your stack:**

| Role | This build | Swap in any of |
|---|---|---|
| CRM / deals | HubSpot (API) | Salesforce, Pipedrive, Attio, Close |
| Newsletter / broadcast | Customer.io (CLI) | Mailchimp, Resend, Beehiiv, Loops |
| Cold outbound | Smartlead (CLI) | Instantly, Lemlist, Apollo sequences |
| Lead data / enrichment | Apollo (API) | Clay, ZoomInfo, Clearbit |
| Meeting notes / transcripts | Granola (API) | Fathom, Fireflies, Otter, tl;dv |

Each adapter exposes read verbs (list/get/search) and write verbs (create/update/send). **Read is
free to run; writes are gated** (see Phase 5/7).

> **Prompt:** "Add an adapter for [your tool]. Read its API docs at [url]. Put the base URL and every
> endpoint in `config.py` as overridable settings, the key in `.env`, and expose read commands first
> (list/get/search). Then add it to `doctor`. Don't hardcode anything."

**Secrets hygiene:** `.env` is git-ignored and `chmod 600`. Tools with their own config
(`~/.tool/config.json`) keep keys there. **Never print or commit a key.** Put a vault-wide rule in
`CLAUDE.md`: confirm before any outward-facing action; read-only queries run freely.

---

## Phase 4 — The research engine (fresh material, no API keys)

**Goal:** before you write anything (a post, a newsletter), pull what's *actually being discussed now*
so the content is fresh, not generic.

The approach: **aggregate RSS** from sources that don't need keys — Reddit (subreddit + search),
community forums, and Google News topic queries — then dedupe and rank by *relevance + freshness +
source*. The feed list is a **config-driven JSON registry** you edit without touching code.

**The freshness trap (learn from our scar):** an RSS item's `pubDate` is the *syndication* date, not
the original. PR wires and aggregators re-stamp old press releases with today's date — we shipped a
draft built on "fresh" news that was actually **seven weeks old**. Fixes that matter:

- **Trust self-posted timestamps** (Reddit, forum threads) — those are real.
- **Flag/penalize wire publishers** (a config-driven denylist) so a re-post can't rank as fresh.
- **Verify the original date** (fetch the article, read the dateline) before you build on any news item.

> **Prompt:** "Build a research command that pulls RSS from a config-driven JSON feed registry
> (Reddit + a community forum + Google News queries on my topics), dedupes, and ranks by relevance +
> freshness + source. Critical: news `pubDate` is the syndication date — add a config list of
> PR-wire publishers whose items get zero freshness credit and a 'verify' flag, and always trust
> Reddit/forum timestamps over news. Save each run to a dated JSON file."

---

## Phase 5 — The human-in-the-loop browser review (the killer pattern)

**Goal:** for anything that has no API (LinkedIn) or needs your judgment (sending a newsletter,
enrolling contacts), stop pasting drafts back and forth in chat. Review it in a real UI and let your
verdict flow back to Claude automatically.

**The pattern:**

1. Claude writes a small **JSON draft** (the content + metadata).
2. A tiny **local HTTP server** renders that JSON into a **single, fixed HTML template** and
   auto-opens it in your browser.
3. You **view, edit, approve, comment, and copy** in the page.
4. The page **POSTs your verdict to a file on disk**.
5. Claude runs a backgrounded **`wait`** that returns the moment the verdict lands — no pasting.

Why it's good:

- **One reusable template.** The design lives once, in code. Each item only ships a few lines of JSON.
  Zero per-item redesign cost (and zero tokens spent re-designing).
- **Brandable.** Inline your logo (SVG), set your colors, load your font, use real button styles. Ours
  ended up light-themed with a brand accent, an inlined logo, shadcn-style buttons, and Poppins.
- **Generalizes.** The same loop powers any approve-list: post drafts, newsletter sends, reply triage,
  per-contact follow-ups. **A per-item "Approve" on the page IS the authorization to execute** — the
  page is the go-ahead, not chat.

**Gotchas we hit (save yourself the debugging):**

- **Auto-open gets swallowed in sandboxes.** Launch the browser with the OS-native opener (`open` on
  macOS, `xdg-open` on Linux), not just a library call.
- **The server is long-running.** It reuses the template across the day (good), but you must **restart
  it after a code change** to pick up edits.
- **Form controls don't inherit fonts.** Set `font-family` on `button`/`textarea` explicitly or they
  fall back to the system font.
- **Re-drafting the same id should reset the page** — clear the stale verdict so the review starts
  fresh.

> **Prompt:** "Build a browser review loop: a command that takes a JSON draft, renders it into ONE
> fixed HTML template, and auto-opens it (use the native `open`/`xdg-open`). The page lets me edit,
> approve/request-changes/reject, comment, and copy; on submit it POSTs my verdict to a file. Add a
> backgrounded `wait <id>` command that blocks until the verdict file appears and prints it. Make the
> theme/logo/colors/font config-driven so I can brand it."

---

## Phase 6 — Voice & guardrails (so the output sounds like you)

**Goal:** drafts that are on-brand and honest, checkable before they go out.

- **A voice spec** the AI reads before drafting: how it should sound (tone) *and* what it should be
  about (topic), with do/don't lists and examples. Ours separates "how it sounds" from "what it's
  about" and bans generic advice.
- **A live meter** in the review page that scores each draft against **config-driven** rules: banned
  words, sentence length, hook length, a real question present, cringe/generic tells. Green-before-you-send.
- **Honesty + positioning rules**, encoded once: never fabricate a number or a story; keep your real
  stance (don't pose as your own audience). These belong in both the spec *and* a feedback memory.

> **Prompt:** "Write a voice-and-tone spec for my [channel] as a wiki page — how it should sound and
> what it should be about, with a do/don't list and a pre-send checklist. Then add a live 'meter' to
> the review page that scores a draft against config-driven word lists and thresholds (banned words,
> hook length, sentence length, has-a-question, cringe/generic tells)."

---

## Phase 7 — Daily commands + the routine

**Goal:** turn the pieces into one-word daily moves.

- **Slash commands / skills**, one per job, **namespaced** so they're easy to find (ours are
  `/ops-*`: `ops-linkedin-daily`, `ops-smartlead-triage`, `ops-pipeline-followup`,
  `ops-hubspot-newsletter`, `ops-gtm-daily`…). Each command is a markdown file describing the steps
  Claude should take.
- **A standup command** that chains them (a daily GTM run).
- **The approve gate everywhere:** read-only summaries appear in chat; anything outward-facing routes
  through the Phase-5 review page, where a click is the authorization.

> **Prompt:** "Create a slash command `/[ns]-[job]` that runs my [job] end to end: [steps]. It should
> end any outward-facing action by opening the review page for my approval, never by sending
> directly. Namespace it `[ns]-` so it groups with my other daily commands."

---

## Phase 8 — Compounding (the flywheel that makes tomorrow easier)

The system gets better because every run writes back:

- **Logs.** A content/decision log (e.g. every posted item + what landed) so you rotate angles and
  never repeat yourself.
- **Memory.** New rules and corrections become feedback memories.
- **Health checks.** Periodically *"lint the wiki"* — find orphan notes, dead links, stale claims, gaps.
- **Hot cache.** Keep `hot.md` current so each session starts oriented.

> **Prompt:** "After I confirm this is done/posted, append it to the [log] with the key details, add a
> row to the tracker, and update `hot.md`. If you learned a new rule from my feedback, save it to
> memory."

---

## The principles that make all of it work

Carry these into every prompt you give Claude — they're the difference between a toy and a system:

1. **Config-driven, never hardcoded.** Secrets, endpoints, ports, thresholds, word lists, feeds — all
   config. Swapping a tool or tuning a rule is an edit, not a rewrite.
2. **Modular.** One module per tool/responsibility; files under ~300 lines.
3. **Verify before trust.** Dates lie (syndication vs. original); data lies (a messy CRM). Check before
   you build on it.
4. **Confirm before outward-facing actions.** Sending/posting/enrolling needs a human go-ahead; the
   review page is that gate. Read-only is free.
5. **Single reusable templates.** Design once, feed data many times. No per-item regeneration.
6. **Plain files.** Markdown + JSON on disk. Portable, diffable, Obsidian-browsable, and the next
   person can read it without your help.
7. **Honesty.** Spicy never means fake. Don't invent facts, numbers, or stories.

---

## Appendix A — The end-to-end prompt script

Paste these into Claude Code in order. Fill the brackets with your specifics.

1. *Scaffold the brain* — the Bootstrap prompt at the top.
2. *Scaffold the tools* — the second quickstart prompt.
3. `"Add an adapter for [CRM]. Read-only first. Everything config-driven."`
4. `"Add adapters for [outbound], [newsletter], [lead data], [meeting notes] the same way. Update doctor."`
5. `"Build the deal/contact sync: pull every [CRM] record into its own folder with the full comms, notes, and meeting transcripts beside it."`
6. *Build the research engine* — Phase 4 prompt.
7. *Build the browser review loop* — Phase 5 prompt.
8. *Write the voice spec + tone meter* — Phase 6 prompt.
9. *Create each daily command + a standup* — Phase 7 prompt.
10. *Wire the compounding writes* — Phase 8 prompt.
11. `"Run doctor. Walk me through what's connected and what still needs a key."`

---

## Appendix B — What a day looks like once it's built

```
$ gtm doctor                 # what's connected
$ /[ns]-gtm-daily            # standup: pipeline, reply triage, what needs attention
$ /[ns]-linkedin-daily       # research → draft → review in browser → approve → logged
$ /[ns]-hubspot-newsletter   # draft → review in browser → approve = send
```

Three layers, one folder, plain files. The AI is the easy part. You just built the 80% underneath.
