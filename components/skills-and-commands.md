# Skills and Slash Commands

How Claude Code's two extension mechanisms work, and how to add your own.

---

## The Two Mechanisms at a Glance

| | Slash Commands | Skills |
|---|---|---|
| **File** | `commands/<name>.md` | `skills/<name>/SKILL.md` |
| **Activated by** | Typing `/name` explicitly | Keywords, intent patterns, file paths, or content patterns — automatic |
| **Best for** | Fixed routines you invoke manually | Reusable domain knowledge, guardrails, auto-surfaced guidance |
| **Frontmatter** | `description` | `name`, `description` (all trigger keywords live here) |
| **Scope** | User (`~/.claude/commands/`) or plugin (`<vault>/skills/<name>/commands/`) | User (`~/.claude/skills/`) or plugin (`<vault>/skills/<name>/`) |

---

## Slash Commands

A slash command is a plain Markdown file whose body is a prompt template that Claude executes when you type `/name`.

### File location

- **Global (all projects):** `~/.claude/commands/<name>.md`
- **Plugin-scoped:** `<vault>/.claude/commands/<name>.md`

The filename becomes the command name. Nested folders are allowed: `commands/gtm/daily.md` → `/gtm/daily`.

### Anatomy: real `ops-linkedin-daily.md`

```markdown
---
description: Daily LinkedIn — one genuinely valuable post + engagement targets, reviewed in-browser
---

Help me run LinkedIn for the day, per the brain's LinkedIn engine ...
```

The `description` field shows up in the `/` picker. Everything after the frontmatter is the instruction body Claude receives verbatim. The file can reference other files, call CLI tools, use sub-agents — anything Claude can do in a turn.

### Namespacing

The daily GTM commands are all prefixed `ops-` (`ops-linkedin-daily`, `ops-pipeline-followup`, etc.). Typing `/ops` in the chat prompt filters to just those commands. Pick a short prefix for any cluster of related commands so they stay discoverable without polluting the top-level namespace.

### When to use a slash command vs. a skill

Use a slash command when:
- You invoke the routine deliberately (e.g. "run today's LinkedIn loop").
- The steps are fixed enough to write out once.
- You want it visible in the `/` picker.

Use a skill when the context should be injected automatically without you typing anything, or when you want guardrail enforcement.

---

## Skills

A skill is a folder `skills/<name>/` containing at minimum a `SKILL.md`. The skill system auto-suggests (or enforces) the skill whenever a user prompt or file edit matches the configured triggers.

### File layout

```
skills/
  wiki-lint/
    SKILL.md          # required — under 500 lines
    references/       # optional detail files (progressive disclosure)
      dragonscale.md
      tiling.md
```

### SKILL.md frontmatter

```yaml
---
name: wiki-lint
description: >
  Health check the Obsidian wiki vault. Finds orphan pages, dead wikilinks,
  stale claims, missing cross-references, frontmatter gaps, and empty sections.
  Triggers on: "lint", "health check", "clean up wiki", "find orphans", "wiki audit".
---
```

All trigger keywords **must** appear in `description`. The activation hook reads this field to match prompts. Max 1024 characters.

---

## skill-rules.json

`~/.claude/skills/skill-rules.json` is the master configuration that maps every skill to its triggers and enforcement level.

### Schema shape (top of the real file)

```json
{
  "version": "2.0",
  "description": "Skill activation triggers customized for your projects.",
  "skills": {
    "core-principles": {
      "type": "guardrail",
      "enforcement": "warn",
      "priority": "critical",
      "description": "Non-negotiable: config-driven, no hardcoded values, modular, type-safe. Enforced on ALL code.",
      "promptTriggers": {
        "keywords": ["create", "implement", "add", "build", "write", "fix"],
        "intentPatterns": [
          "(create|add|implement|build|write|make).*?(function|component|route|endpoint|service|module)"
        ]
      },
      "fileTriggers": {
        "pathPatterns": ["**/*.ts", "**/*.tsx", "**/*.js", "**/*.py", "**/*.go"],
        "pathExclusions": ["**/node_modules/**", "**/*.test.*", "**/*.md"]
      }
    }
  }
}
```

### Trigger types

| Type | Field | Matches on |
|---|---|---|
| Keyword | `promptTriggers.keywords` | Exact substring in prompt |
| Intent | `promptTriggers.intentPatterns` | Regex over the full prompt |
| File path | `fileTriggers.pathPatterns` | Glob matched against any file being read/edited |
| Content | `fileTriggers.contentPatterns` | Regex matched against file contents |

All triggers within a skill are OR'd. A single match is enough to activate.

### Enforcement levels

| Level | Mechanism | When to use |
|---|---|---|
| `block` | PreToolUse hook exits with code 2; Edit/Write tools are physically blocked until the skill is consulted | Critical guardrails: data integrity, security, must-not-miss patterns |
| `suggest` | UserPromptSubmit hook injects a "SKILL ACTIVATION CHECK" reminder into Claude's context before Claude sees your prompt | Domain guidance, best practices, how-to knowledge |
| `warn` | Advisory only, low friction | Nice-to-have reminders |

Most skills should be `suggest`. Use `block` only for genuinely critical guardrails where the cost of skipping is high.

### Priority

`"critical"` > `"high"` > `"medium"` > `"low"`. Controls ordering when multiple skills activate simultaneously.

---

## The Activation Hook

`~/.claude/hooks/skill-activation-prompt.ts` is registered as a **UserPromptSubmit** hook in `.claude/settings.json`. It runs before Claude processes any prompt:

1. Reads the prompt text.
2. Matches keywords and intent patterns from `skill-rules.json`.
3. If a `suggest` or `warn` skill matches, injects a formatted "SKILL ACTIVATION CHECK" block into Claude's context.
4. If a `block` skill matches, the corresponding PreToolUse hook (`skill-verification-guard.ts`) will exit with code 2 the moment Claude tries to edit a matching file.

You see the skill reminder as the grey system-reminder block at the top of Claude's response. That is the injection in action.

For a deeper look at hook internals see [hooks.md](hooks.md).

---

## How to Add a Slash Command

1. Create the file at `~/.claude/commands/<name>.md` (global) or `<vault>/.claude/commands/<name>.md` (vault-scoped).
2. Add YAML frontmatter with `description`.
3. Write the instruction body. Reference any files, CLI tools, or wiki pages you need.
4. Type `/<name>` in Claude Code to invoke it.

### Copy-paste template

```markdown
---
description: One-line description for the / picker
---

## What this command does

<Describe the goal in plain language so Claude has context.>

## Steps

1. <First thing Claude should do>
2. <Second thing>
3. <Etc.>

## Constraints

- Never do X.
- Always confirm before Y.
```

---

## How to Add a Skill (5 Steps)

### Step 1 — Create the skill folder and SKILL.md

```
~/.claude/skills/<your-skill-name>/SKILL.md
```

For a plugin-scoped skill (lives in this vault):

```
<vault>/skills/<your-skill-name>/SKILL.md
```

### Step 2 — Write SKILL.md

Keep it under 500 lines. Put the meaty details in `references/` sub-files.

```markdown
---
name: my-skill
description: >
  What this skill does. Include EVERY keyword or phrase that should activate it.
  Examples: "my topic", "related term", "another trigger phrase".
  Triggers on: "keyword one", "keyword two", "action phrase".
---

# My Skill

## Purpose

One paragraph on what this skill provides.

## Key guidance

...

## Reference files

- [details.md](references/details.md) — extended documentation
```

**Naming convention:** lowercase hyphens, gerund form preferred (`processing-leads`, `reviewing-deals`).

### Step 3 — Add an entry to skill-rules.json

Open `~/.claude/skills/skill-rules.json` and add inside `"skills": { ... }`:

```json
"my-skill": {
  "type": "domain",
  "enforcement": "suggest",
  "priority": "medium",
  "description": "One-line summary",
  "promptTriggers": {
    "keywords": ["my topic", "related term"],
    "intentPatterns": [
      "(create|add|build).*?my-topic",
      "how (do|does|to).*?my-topic"
    ]
  },
  "fileTriggers": {
    "pathPatterns": ["**/relevant-folder/**/*.ts"],
    "pathExclusions": ["**/node_modules/**"]
  }
}
```

Validate JSON after editing:

```bash
jq . ~/.claude/skills/skill-rules.json
```

### Step 4 — Test the triggers

```bash
# Test UserPromptSubmit (suggest/warn skills)
echo '{"session_id":"test","prompt":"your test prompt here"}' | \
  npx tsx ~/.claude/hooks/skill-activation-prompt.ts

# Test PreToolUse (block skills)
cat <<'EOF' | npx tsx ~/.claude/hooks/skill-verification-guard.ts
{"session_id":"test","tool_name":"Edit","tool_input":{"file_path":"relevant/file.ts"}}
EOF
```

Try at least 3 real prompts that should trigger the skill and 3 that should not.

### Step 5 — Refine and keep under 500 lines

- Add missing keywords that real prompts use.
- Tighten intent patterns to reduce false positives.
- Move detail into `references/` files if SKILL.md approaches 500 lines.
- Add a table of contents to any reference file over 100 lines.

---

## Anthropic Best Practices (Summary)

| Practice | Rule |
|---|---|
| 500-line rule | SKILL.md must stay under 500 lines; overflow goes to reference files |
| Progressive disclosure | Top-level file = summary; `references/` = depth |
| Rich description | Pack all trigger keywords into the frontmatter `description` (max 1024 chars) |
| Gerund naming | `processing-leads`, not `lead-processor` |
| Test before documenting | Run 3+ real prompts before calling the skill done |
| One level of nesting | Reference files should not chain to more reference files |
| Table of contents | Any reference file over 100 lines gets a TOC |

---

## Related docs

- [hooks.md](hooks.md)
- [knowledge-base.md](knowledge-base.md)
- [README.md](README.md)
