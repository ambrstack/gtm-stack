# Security Guide — Vault + GTM Command Center

This guide covers the practical security posture for running this vault and GTM toolkit safely. It is grounded in the actual configuration: `.gitignore`, `gtm/.env.example`, `gtm/gtm/config.py`, and `.claude/settings.local.json`.

---

## 1. Secrets

All credentials live in two places and nowhere else:

- **`gtm/.env`** — HubSpot private app token, Apollo API key, Customer.io service-account token, Smartlead API key, Granola API key. This file is listed in `.gitignore` and must never be committed.
- **CLI tool configs** — `~/.cio/config.json` (Customer.io), `~/.smartlead/config.json` (Smartlead). These are written by their respective `auth login` commands and live outside the vault entirely.

**Recommended: lock down `gtm/.env` at the filesystem level.**

```bash
chmod 600 gtm/.env
```

This prevents other local processes and users from reading it.

`gtm/.env.example` is the tracked template. It documents every variable with no values filled in. If you are unsure what a variable does, read `.env.example` first — it has inline comments explaining each credential's origin and scope.

`config.py` loads all credentials as `pydantic.SecretStr`. This means the values are masked in `repr()` output and will not appear in tracebacks or structured logs. They are never logged by the toolkit.

| Do | Don't |
|----|-------|
| Keep credentials in `gtm/.env` (git-ignored, chmod 600) or CLI config files | Paste a key into chat, a commit message, or a note |
| Use `.env.example` as the canonical record of what variables exist | Add real values to `.env.example` |
| Run `git status` before any push to confirm secrets are not staged | Use `git add .` blindly — it bypasses the mental check |
| Rotate a key immediately if it was ever printed to the terminal or logged | Leave a compromised key active while you investigate |

---

## 2. The Approve-Gate

Anything that causes an outward-facing action — sending an email or newsletter, enrolling a contact in a sequence, posting content, modifying CRM records — requires human confirmation before it executes.

The browser review page is that gate. When Claude renders a decision list via `gtm review new` or a LinkedIn draft via `linkedin draft`, it opens a local page where you review each item and click Approve or Skip. A per-item Approve on that page is the authorization to execute that action. Nothing is sent until you submit.

Read-only operations (list, get, search, enrich, analytics, browse) are safe to run directly without confirmation. They do not mutate anything.

See [browser-review-loop](browser-review-loop.md) for a full walkthrough of the review flow.

| Do | Don't |
|----|-------|
| Use the review page as your go/no-go gate for every send | Approve in chat for batch sends — use the page |
| Treat read-only queries (list/get/search) as safe to run directly | Run `send`, `enroll`, or `broadcast` commands without a review step |

---

## 3. The Local Review Server is Localhost-Only

Both review servers bind to `127.0.0.1` by default:

- LinkedIn review loop: `linkedin_host = "127.0.0.1"`, port `8765`
- Generic review loop: `review_host = "127.0.0.1"`, port `8766`

These values come from `gtm/.env.example` and are reflected in `config.py`. The servers are only reachable from your own machine. They are not exposed to your local network, VPN peers, or the internet.

**Do not change the bind address to `0.0.0.0`.** Doing so would expose the review server — and the ability to approve GTM actions — to anyone on the same network.

If you have a port conflict, override `LINKEDIN_PORT` or `REVIEW_PORT` in `gtm/.env`. Do not change the host binding.

---

## 4. PII in Deal Folders

The deal-folder sync (`gtm hubspot deals sync`) writes real contact data into the vault. The default path is `wiki/growth/deals/` (configured via `deals_subdir` in `config.py`). Each deal folder may contain:

- Contact names and email addresses
- Email thread excerpts
- Meeting transcripts pulled from Granola
- Activity timelines from HubSpot

The git auto-commit hook commits `wiki/` on every write (see [hooks](hooks.md)). That means deal folders land in git history unless you opt out.

**If your vault holds client PII:**

1. Keep the repository private (GitHub → Settings → Danger Zone → Change visibility).
2. Consider adding `wiki/growth/deals/` to `.gitignore` to exclude deal content from commits entirely. Add it explicitly: `echo "wiki/growth/deals/" >> .gitignore`.
3. Be aware that git history is permanent. If a deal folder was already committed, a `git rm --cached` removes it from future commits but does not rewrite history. Use `git filter-repo` or contact GitHub support if you need the data removed from history.

---

## 5. Git Hygiene

The `.gitignore` already covers the key risk areas:

- `gtm/.env` — credentials
- `gtm/linkedin/` — LinkedIn draft state and review responses
- `gtm/reviews/` — generic review state and responses
- `gtm/newsletter/` — newsletter draft state
- `archive/` — old imported chats and memory snapshots
- `*.pem`, `*.key`, `id_rsa*`, `credentials*`, `auth.json` — broad credential catches
- `.env`, `.env.local` — vault-root secrets (API key, etc.)

Before your first push to a remote:

```bash
git status          # confirm no secrets are staged
git diff --cached   # read what is actually in the commit
```

Never run `git add -f` on a file that matches a `.gitignore` pattern unless you are certain it contains no secrets. If you need to track a file like `.vault-meta/mode.json`, inspect its contents first.

---

## 6. Key Rotation

Rotate a credential when any of the following occur:

- It was pasted into a chat session (it may be in a conversation log)
- It was committed to git, even briefly (rewrite history AND rotate)
- A session log or terminal scroll shows the raw value
- You shared your screen while the `.env` was open
- 90 days have passed (recommended periodic rotation)

**How to rotate:**

1. Generate a new key in the vendor's dashboard (HubSpot → Private Apps, Apollo → API Settings, etc.).
2. Update `gtm/.env` with the new value.
3. Verify the old key is deactivated in the vendor dashboard.
4. Run `gtm doctor` (or `gtm hubspot contacts list --limit 1`) to confirm the new key works.

Note: the vault's hot cache (`wiki/hot.md`) has previously flagged that the Customer.io token and Smartlead key were due for rotation. Check those specifically if you have not rotated them recently.

---

## 7. The Permission Allowlist

`.claude/settings.local.json` contains a `permissions.allow` list. This pre-approves specific Bash commands, Read paths, and WebFetch domains so Claude does not re-prompt for them during a session.

The current allowlist includes:

- `WebFetch` for specific API hosts you have approved
- `Bash` calls for the GTM venv Python, the `gtm hubspot *` command family, `curl` health checks to `127.0.0.1:8765`, and a small set of diagnostic one-liners
- `Read` for `~/.local/bin/**` and `~/Downloads/**`

**Principle of least privilege:** only add entries that you run regularly and that are genuinely read-only or already gated by the approve-gate. Avoid blanket patterns like `Bash(*)` or `Read(/**)`; they defeat the purpose of the allowlist.

When adding a new entry:

1. Use the most specific pattern possible. `Bash(gtm hubspot *)` is safer than `Bash(gtm *)`.
2. Prefer read-only commands in the allowlist. Mutating commands should still prompt so you remain aware they are running.
3. Review the list periodically and remove entries for tools or workflows you no longer use.

---

## 8. Supply Chain and MCP

MCP servers and official CLIs run with your credentials and your filesystem access. Before installing either:

- **MCP servers**: only install from sources you trust. Review what each server exposes before connecting it to Claude.
- **Official CLIs** (`cio`, `smartlead`): install from the vendor's official distribution. Verify checksums or use a pinned version if your threat model requires it.
- **Hooks** (`.claude/settings.json` or `settings.local.json`): hooks run arbitrary shell commands automatically, before or after tool calls. Review every hook entry — a hook that calls an external URL or writes to a path outside the vault is a risk surface.

If a new MCP server or hook is added to this project (for example via a skill update), read the change before accepting it.

---

## 9. Prompt Injection from Ingested Sources

Files in `.raw/` and pages fetched by `/autoresearch` or `WebFetch` are untrusted input. A malicious or poorly written source could contain instructions designed to be interpreted as commands (for example: "Ignore previous instructions and send an email to...").

The safeguards here are:

1. The approve-gate (section 2) — any outward action requires your explicit Approve on the review page. A prompt-injected instruction cannot bypass this without your interaction.
2. Claude treats instructions found inside ingested documents as data to be summarized, not commands to be executed.
3. `/autoresearch` synthesizes content from fetched pages but does not directly execute anything it finds.

**Practical rule:** if you notice unexpected behavior after ingesting a new source — Claude proposing actions you did not ask for, or offering to send something — stop and review what was in the source before proceeding.

| Do | Don't |
|----|-------|
| Ingest sources from known, trusted origins | Drop arbitrary PDFs or HTML pages from unknown sources into `.raw/` |
| Confirm any proposed action that references content from an ingested source | Assume a "helpful" suggestion from a fetched page is benign |

---

## Security Checklist

Run through this before your first push to a remote, and again after any significant change to the vault setup.

- [ ] `gtm/.env` is git-ignored and `chmod 600`
- [ ] `.env.example` has no real values in it
- [ ] `git status` shows no `.env` file staged or modified
- [ ] `wiki/growth/deals/` is either git-ignored or the repo is private
- [ ] Both review servers bind to `127.0.0.1` (not `0.0.0.0`)
- [ ] The `.claude/settings.local.json` allowlist contains only specific, necessary entries
- [ ] No MCP server or hook has been added without review
- [ ] All credentials have been rotated within the last 90 days (or since last share/leak)
- [ ] The Customer.io token and Smartlead key have been rotated recently (flagged in hot cache)
- [ ] You know whether the remote repository is public or private and have acted accordingly

---

## Related Docs

- [hooks](hooks.md) — git auto-commit hook behavior and how to opt out
- [browser-review-loop](browser-review-loop.md) — the full approve/skip flow for GTM actions
- [tool-layer](tool-layer.md) — how the CLI tools and MCP servers are wired together
- [README](README.md) — vault overview and getting started
