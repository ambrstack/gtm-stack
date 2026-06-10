# Claude Code Hook System

Hooks let you run shell commands or inject instructions into Claude at specific lifecycle events. This document covers exactly what is wired in this setup and how to extend it.

## What a Hook Is

A hook is a `command` (shell script) or `prompt` (plain-text instruction injected into Claude's context) that the Claude Code harness executes automatically when a lifecycle event fires. Hooks are declared in one of two places:

| Config file | Scope |
|---|---|
| `~/.claude/settings.json` | User-level -- runs in every Claude Code session on this machine |
| `hooks/hooks.json` (this vault) | Plugin-level -- runs only when this vault is loaded as a Claude Code plugin |

Hooks cannot be fulfilled by Claude's memory. They are executed by the harness, not by Claude, which is why they are the correct mechanism for "always do X when Y happens".

---

## Hook Events

| Event | When it fires |
|---|---|
| `SessionStart` | Once when a session opens (variant: `startup`) or is resumed after interruption (variant: `resume`). Matcher is a regex on the variant string. |
| `UserPromptSubmit` | Every time the user submits a message, before Claude processes it. |
| `PreToolUse` | Before Claude calls a tool. Exit code `2` from a `command` hook blocks the tool call entirely. |
| `PostToolUse` | After a tool call completes. Matcher is a regex on the tool name (e.g. `Edit\|Write`). |
| `PostCompact` | After Claude Code compacts the context window. Hook-injected context does NOT survive compaction, so this is the right place to re-inject it. |
| `Stop` | When Claude finishes a turn and stops generating. |

---

## Hook Types

**`command`** -- runs a shell command. Claude Code passes a JSON payload on stdin. Stdout is captured and injected into Claude's context (as a tool result). Stderr is discarded. Exit code `2` on a `PreToolUse` hook blocks the tool.

**`prompt`** -- injects a plain-text string directly as an instruction to Claude. No shell is invoked; the string becomes part of the context window.

The `matcher` field is a regex. For `PostToolUse` it matches against the tool name. For `SessionStart` it matches against the session variant (`startup` or `resume`). An empty string `""` matches everything.

---

## Active Hooks in This Setup

### User-level hooks (`~/.claude/settings.json`)

#### UserPromptSubmit -- Skill Activation Check

```json
{
  "matcher": "",
  "hooks": [
    {
      "type": "command",
      "command": "~/.claude/hooks/skill-activation-prompt.sh"
    }
  ]
}
```

Fires on every user message. The script pipes stdin through `npx tsx skill-activation-prompt.ts`, which outputs a "SKILL ACTIVATION CHECK" reminder. This injects a prompt fragment that nudges Claude to check whether any installed skill matches the current request before responding. Matcher is empty, so it fires unconditionally.

#### PostToolUse (Edit | MultiEdit | Write) -- File Edit Tracker

```json
{
  "matcher": "Edit|MultiEdit|Write",
  "hooks": [
    {
      "type": "command",
      "command": "~/.claude/hooks/post-tool-use-tracker.sh"
    }
  ]
}
```

Fires after any `Edit`, `MultiEdit`, or `Write` tool call. The script:

1. Reads the tool payload from stdin (JSON) and extracts `tool_name`, `tool_input.file_path`, and `session_id`.
2. Skips `.md` and `.markdown` files.
3. Detects which sub-repo the edited file belongs to (frontend, backend, packages/*, etc.) based on the first path component under `$CLAUDE_PROJECT_DIR`.
4. Appends `timestamp:file_path:repo` to `~/.claude/tsc-cache/<session_id>/edited-files.log`.
5. Maintains `affected-repos.txt` (deduplicated list of touched repos) and `commands.txt` (inferred `tsc --noEmit` and `build` commands for each affected repo, sorted and deduplicated).

This cache is consumed by the `/build-check` skill, which reads it to know exactly which repos need a TypeScript check after a session.

#### Stop -- Error Handling Reminder

```json
{
  "matcher": "",
  "hooks": [
    {
      "type": "command",
      "command": "~/.claude/hooks/error-handling-reminder.sh"
    }
  ]
}
```

Fires at the end of every Claude turn. The script pipes stdin through `npx tsx error-handling-reminder.ts`, which outputs a reminder about error handling patterns. Skipped if the environment variable `SKIP_ERROR_REMINDER` is set.

---

### Plugin-level hooks (`hooks/hooks.json`)

#### SessionStart -- Hot Cache Injection

```json
{
  "matcher": "startup|resume",
  "hooks": [
    {
      "type": "command",
      "command": "[ -f wiki/hot.md ] && cat wiki/hot.md || true"
    },
    {
      "type": "command",
      "command": "[ -x scripts/wiki-lock.sh ] && bash scripts/wiki-lock.sh clear-stale --max-age 3600 >/dev/null 2>&1 || true"
    },
    {
      "type": "prompt",
      "prompt": "If a vault is configured for this session (check CLAUDE.md for VAULT_PATH or a wiki/ folder in the current directory), silently read wiki/hot.md to restore recent context. If wiki/hot.md does not exist, do nothing. This is a non-vault session. Do not announce this. Do not report what you read. Just have the context available."
    }
  ]
}
```

Fires on both `startup` and `resume`. Three hooks run in order:

1. **command** -- cats `wiki/hot.md` into stdout so the harness injects its contents as context. The `|| true` ensures exit 0 in non-vault sessions.
2. **command** -- runs `wiki-lock.sh clear-stale --max-age 3600` to remove any advisory locks older than one hour. Stale locks can block the PostToolUse auto-commit; this clears them at session open. Output is suppressed; exits 0 always.
3. **prompt** -- semantic fallback: instructs Claude to silently read `wiki/hot.md` if it was not already injected by the command hook. This is a workaround for the known plugin hook STDOUT bug (see below).

#### PostCompact -- Hot Cache Re-injection

```json
{
  "matcher": "",
  "hooks": [
    {
      "type": "prompt",
      "prompt": "Hook-injected context does not survive context compaction. If wiki/hot.md exists in the current directory, silently re-read it now to restore the hot cache. Do not announce this."
    }
  ]
}
```

Fires after any context compaction. Hook-injected context from `SessionStart` is lost when the window is compacted; only `CLAUDE.md` survives. This prompt tells Claude to re-read `wiki/hot.md` immediately after compaction, restoring the hot cache.

#### PostToolUse (Write | Edit) -- Wiki Auto-Commit

```json
{
  "matcher": "Write|Edit",
  "hooks": [
    {
      "type": "command",
      "command": "[ -d .git ] || exit 0; [ -f .vault-meta/auto-commit.disabled ] && exit 0; if [ -x scripts/wiki-lock.sh ]; then LOCK_LIST=$(bash scripts/wiki-lock.sh list 2>/dev/null); LOCK_RC=$?; if [ \"$LOCK_RC\" != \"0\" ]; then mkdir -p .vault-meta 2>/dev/null; printf '%s wiki-lock list failed rc=%s; deferred auto-commit\\n' \"$(date '+%Y-%m-%dT%H:%M:%SZ')\" \"$LOCK_RC\" >> .vault-meta/hook.log 2>/dev/null; exit 0; fi; if [ -n \"$LOCK_LIST\" ]; then exit 0; fi; fi; git add -- wiki/ .raw/ .vault-meta/ 2>/dev/null && (git diff --cached --quiet -- wiki/ .raw/ .vault-meta/ || git commit -m \"wiki: auto-commit $(date '+%Y-%m-%d %H:%M')\" -- wiki/ .raw/ .vault-meta/ 2>/dev/null) || true"
    }
  ]
}
```

Fires after every `Write` or `Edit` tool call. The command (a single inline shell pipeline) does the following:

1. Exits 0 immediately if `.git` does not exist -- safe in non-git directories.
2. Exits 0 if `.vault-meta/auto-commit.disabled` exists -- provides a kill-switch.
3. Calls `wiki-lock.sh list`. If the lock script fails (non-zero exit), logs to `.vault-meta/hook.log` and defers; does not commit. If any locks are held (non-empty output), skips the commit -- a multi-writer ingest is in progress.
4. Runs `git add -- wiki/ .raw/ .vault-meta/`.
5. Checks `git diff --cached --quiet` to skip empty commits.
6. Commits with message `wiki: auto-commit YYYY-MM-DD HH:MM`.

To disable auto-commits temporarily: `touch .vault-meta/auto-commit.disabled`. Remove the file to re-enable.

#### Stop -- WIKI_CHANGED Reminder

```json
{
  "matcher": "",
  "hooks": [
    {
      "type": "command",
      "command": "cd \"$PWD\" && [ -d wiki ] && [ -d .git ] && git diff --name-only HEAD 2>/dev/null | grep -q '^wiki/' && echo 'WIKI_CHANGED: Wiki pages were modified this session. Please update wiki/hot.md with a brief summary of what changed (under 500 words). Use the hot cache format: Last Updated, Key Recent Facts, Recent Changes, Active Threads. Keep it factual. Overwrite the file completely. It is a cache, not a journal.' || true"
    }
  ]
}
```

Fires at the end of every Claude turn. Checks whether any `wiki/` file changed in this session (`git diff --name-only HEAD | grep '^wiki/'`). If yes, prints a `WIKI_CHANGED:` message to stdout. The harness injects this as context, telling Claude to update `wiki/hot.md` with a brief summary. Exits 0 silently if nothing changed or if the directory is not a vault.

---

## Known Issue: Plugin Hook STDOUT Bug

`anthropics/claude-code#10875` -- plugin hook stdout may not be captured in some Claude Code versions. If hot cache restoration fails, test by opening a fresh session and asking "what is in the hot cache?". If Claude has no context, copy the relevant hooks from `hooks/hooks.json` into your user-level `~/.claude/settings.json` as a workaround.

---

## How to Add a Hook

### 1. Decide where to put it

- Affects all your Claude Code sessions: add to `~/.claude/settings.json` under `hooks`.
- Affects only this vault: add to `hooks/hooks.json`.

### 2. Pick the event and write your script

Create a script, make it executable, and test it standalone before wiring it as a hook. Hook scripts receive a JSON payload on stdin; print anything you want injected as context to stdout.

### 3. Add the hook entry

Minimal example -- a PostToolUse hook that runs a formatter after any Write:

```json
{
  "PostToolUse": [
    {
      "matcher": "Write",
      "hooks": [
        {
          "type": "command",
          "command": "/path/to/your/format-on-write.sh"
        }
      ]
    }
  ]
}
```

Minimal example -- a prompt hook that fires on every UserPromptSubmit:

```json
{
  "UserPromptSubmit": [
    {
      "matcher": "",
      "hooks": [
        {
          "type": "prompt",
          "prompt": "Always think step by step before answering."
        }
      ]
    }
  ]
}
```

### 4. Disable auto-commit if needed

If you are doing a large batch ingest and do not want a commit per file write, create the kill-switch file:

```bash
touch .vault-meta/auto-commit.disabled
# ... do your work ...
rm .vault-meta/auto-commit.disabled
git add wiki/ .raw/ .vault-meta/ && git commit -m "wiki: batch ingest"
```

---

## Security

Hooks run arbitrary shell commands on your machine automatically and silently. Before adding any hook:

- Only add scripts you wrote or have fully read.
- Never put credentials, API keys, or secrets in a hook command string -- use a sourced env file instead.
- Scripts run with your full user permissions; a malicious hook can exfiltrate data or modify files.
- Keep hook scripts in version control so changes are auditable.

See [security](security.md) for the broader threat model.

---

## Related Docs

- [skills-and-commands](skills-and-commands.md)
- [security](security.md)
- [knowledge-base](knowledge-base.md)
- [README](README.md)
