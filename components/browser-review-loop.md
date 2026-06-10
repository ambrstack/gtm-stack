# Browser Review Loop

A human-in-the-loop pattern for approving AI-drafted content in a real browser UI,
with the verdict flowing back to Claude automatically -- no copy-pasting into chat.

Built because LinkedIn has no posting API, but the pattern generalizes to any list of
decisions that need human authorization before Claude acts.

---

## The Pattern in 5 Steps

1. **Claude writes a small JSON draft** -- post text, pillar, date, hashtags, engagement targets, sources.
2. **`linkedin draft -f <file>`** writes the draft to disk, auto-starts the local review server if it
   is not already running, and opens the review URL in your default browser.
3. **You review, edit, and approve in the browser** -- a live tone meter flags problems as you type,
   you can edit the post text directly, add a comment, and click Approve / Request changes / Reject.
4. **The page POSTs a verdict JSON to the server** (`POST /api/submit/<id>`), which writes it to
   `<linkedin_state_dir>/responses/<id>.json` on disk.
5. **`linkedin wait <id>`** (run in the background) polls for that file at `linkedin_poll_interval`
   (default 1 s) and exits the moment it lands, printing the full verdict JSON.

Nothing is pasted back into chat. The terminal output of `wait` is the signal.

---

## Commands

### `linkedin draft -f <json>`

Registers a draft, ensures the server is healthy, and opens the review page.

```
linkedin draft -f draft.json
linkedin draft -f draft.json --no-open   # skip auto-open, just print URL
```

Internally: writes `<linkedin_state_dir>/drafts/<id>.json`, deletes any stale
`responses/<id>.json` so the page resets to pending (re-drafting the same id is always
a fresh review), calls `_ensure_server()`, then `_open_url(url)`.

The draft ID comes from the JSON's `id` field if present; otherwise it is derived from
the `date` + `pillar` fields as a kebab slug (e.g. `2026-06-10-pipeline-health`), falling
back to `post-<8-char hex>`.

### `linkedin wait <id>`

Blocks until a verdict lands, then prints it and exits. Run this in the background
(as a backgrounded terminal process or via a Claude Code background task) so it does not
hold up the conversation.

```
linkedin wait 2026-06-10-pipeline-health
linkedin wait 2026-06-10-pipeline-health --timeout 600   # give up after 10 min
```

Exit codes: `0` = verdict received; `2` = timed out (prints `{"id": "...", "status": "timeout"}`).
Default timeout: `linkedin_wait_timeout` = 1800 s.

### Verdict JSON shape

```json
{
  "id": "2026-06-10-pipeline-health",
  "status": "approved",
  "edited_text": "The text as it stood when you clicked Approve.",
  "comment": "Punch up the hook next time.",
  "targets_done": [],
  "submitted_at": "2026-06-10T14:22:01Z"
}
```

`status` is one of `approved`, `changes`, `rejected`. `edited_text` carries whatever
was in the textarea when you submitted -- if you edited the post in the browser, this is
the final copy. `comment` is the optional free-text note.

### Other commands

| Command | What it does |
|---|---|
| `linkedin serve` | Start the server in the foreground (draft auto-starts it detached) |
| `linkedin status <id>` | Print the current verdict or `{"status":"pending"}` -- non-blocking |
| `linkedin open <id>` | Reopen an existing draft's review page (starts server if needed) |
| `linkedin list` | Print all drafts + their current status to the terminal |
| `linkedin research` | Aggregate fresh RSS material ranked for engagement (separate flow) |

---

## One Fixed Template

Each item ships only a few lines of JSON. The full page design lives once in
`gtm/gtm/tools/linkedin_render.py` (`_DOC` + `_STYLE`). No per-draft HTML is generated;
no tokens are spent re-designing for each post.

### `__DATA__` and `__LOGO__` injection

`linkedin_render.review_page()` builds the page with two string replacements:

```python
return (_DOC
        .replace("__DATA__", _embed({"id": draft_id, "draft": draft, "cfg": cfg}))
        .replace("__LOGO__", logo_svg or _FALLBACK_LOGO))
```

- `__DATA__` becomes a single JSON blob embedded in a `<script>` tag:
  `const DATA = <json>;`. All dynamic content -- post text, pillar, date, hashtags,
  engagement targets, sources, tone-meter thresholds, word lists -- flows through this
  blob. There is no string interpolation inside the CSS or JS, so nothing needs escaping
  by hand. `_embed()` neutralises `</script` sequences (`<\/`).
- `__LOGO__` is replaced with the inlined SVG markup returned by `_logo_svg()`. If the
  logo file is missing, the fallback is a plain HTML wordmark:
  `<span class="brand-fallback"><span class="amb">your</span>brand</span>`.

The dashboard page (`/`) uses the same `_DASH` template with a `__ROWS__` slot, sharing
the same `_STYLE` block via `.replace("__STYLE__", _STYLE)`.

---

## Brandability

Everything visual is config-driven or derived from a single CSS block in `_STYLE`.

### Logo

`_logo_svg()` reads the file at `config.linkedin_logo_path` (default:
`gtm/gtm/tools/assets/logo.svg`), strips any hardcoded `width="..."` and
`height="..."` attributes from the `<svg>` root so CSS can size it, and adds
`class="logo"`. The CSS rule `.logo { height:22px; width:auto; display:block; }`
controls the rendered size. To swap brands, point `LINKEDIN_LOGO_PATH` in `gtm/.env`
at a different SVG.

### Theme

Light, minimal, matching the company site. Key values in `_STYLE`:

| CSS variable / rule | Value |
|---|---|
| Background | `#f6f7f9` |
| Text | `#16181d` |
| Brand accent | `#E4572E` (links, focus ring, pillar chip, textarea focus) |
| Font | Poppins (400/500/600/700) via Google Fonts |
| Button style | shadcn/ui conventions: default = brand black `#18181b`, outline, destructive |

### Form controls and font

`textarea` elements need an explicit `font-family` declaration -- they do not inherit
from `body` in all browsers. `_STYLE` sets:

```css
textarea {
  font: 15px/1.6 'Poppins',-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif;
}
```

The same explicit stack is applied to `button` via `font-family:inherit` (buttons inherit
from their parent, which inherits from `body`).

---

## Live Tone Meter

The review page runs a client-side JavaScript tone meter that fires on every `input`
event. It checks the post text against thresholds and word lists from the `cfg` blob
(`_checks_cfg()` in `linkedin.py`), surfacing green/amber/red badges in real time:

| Check | Config key | Default |
|---|---|---|
| Char count (must be <= 3000) | -- | 3000 |
| No em-dashes | -- | -- |
| Banned marketing words | `linkedin_banned_words` | streamline, leverage, seamless, game-changer, unlock |
| Hook <= fold | `linkedin_hook_max_chars` | 120 chars |
| Avg words/sentence | `linkedin_sentence_max_words` | 20 words |
| Longest block (wall of text) | `linkedin_paragraph_max_chars` | 280 chars |
| Engagement-farm cringe | `linkedin_cringe_words` | see config.py |
| Generic-advice tells | `linkedin_generic_words` | see config.py |
| Question (comment bait) | -- | -- |
| First/2nd-person count | -- | >= 4 = good |

Banned words block submission (the check shows red); cringe and generic words warn but
do not block. Full voice-and-tone rationale: [voice-and-guardrails](voice-and-guardrails.md).

---

## Generalizing: the `gtm review` Loop

The same pattern powers any decision list. `gtm/gtm/tools/review.py` is the generic
version, backed by `review_render.py` and a separate config block:

| Config key | Default |
|---|---|
| `review_state_dir` | `gtm/reviews/` |
| `review_host` | `127.0.0.1` |
| `review_port` | `8766` (different from the LinkedIn loop so both run at once) |
| `review_wait_timeout` | 1800 s |
| `review_poll_interval` | 1.0 s |
| `review_open_browser` | `True` |
| `review_default_decision` | `approve` |

Usage:

```
gtm review new -f items.json   # render items into the fixed template, auto-open
gtm review wait <id>           # run in background; exits when the page is submitted
```

The rendered page shows each item with Approve / Skip / comment and a single "Approve &
send" button. A per-item **Approve is the authorization to execute that action**. Claude
waits for `wait` to return before acting. Read-only queries (list/get/search/enrich) run
directly without review.

---

## Gotchas

### (a) Auto-open gets swallowed in sandboxes

`_open_url()` tries the OS-native opener first:

```python
native = {"darwin": "open", "linux": "xdg-open"}.get(
    "linux" if sys.platform.startswith("linux") else sys.platform
)
if native and shutil.which(native):
    subprocess.Popen([native, url])
    return
```

It falls back to `webbrowser.open()` only if the native binary is missing. In CI, Docker,
or headless environments the native opener exists but has nothing to open. In that case,
copy the printed URL manually. The `--no-open` flag skips the open call entirely.

### (b) Restart the server after a template change

The server (`linkedin serve` / the detached process started by `_ensure_server()`) loads
`linkedin_render.py` at startup. It is a long-running process and does not hot-reload.
After editing `_DOC`, `_STYLE`, or any render function, kill the server process and let
the next `draft` restart it.

The server PID is not tracked; find it with `lsof -i :8765` (or the configured port).

### (c) Re-drafting resets the verdict

`linkedin draft` unconditionally deletes `responses/<id>.json` before writing the new
draft:

```python
stale = _response_path(did)
if stale.exists():
    stale.unlink()
```

Any previously approved verdict for that id is gone. `linkedin wait` will block again.
This is intentional -- re-drafting means a fresh review.

### (d) localhost only

The server binds `config.linkedin_host` = `127.0.0.1` by default. The review URL is
only reachable from the same machine. Do not expose it on `0.0.0.0`. See
[security](security.md).

---

## Related docs

- [voice-and-guardrails](voice-and-guardrails.md) -- tone rules behind the live meter
- [research-engine](research-engine.md) -- how `linkedin research` sources fresh material
- [security](security.md) -- localhost-only binding, credential handling
- [README](README.md) -- vault overview and GTM command center entry point
