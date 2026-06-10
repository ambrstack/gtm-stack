# Adding a Tool (Adapter) to the GTM CLI

A step-by-step guide for wiring any vendor into `gtm`. The entire pattern is config-driven: credentials, base URLs, and every endpoint live in `gtm/.env` and `config.py`; no values are hardcoded in the adapter.

---

## Step 1 — Choose the integration type

| Type | When to use | Examples |
|---|---|---|
| **Official CLI** | Vendor ships a real CLI and you want `gtm doctor` to report its status | Customer.io (`cio`), Smartlead (`smartlead`) |
| **API adapter** | No CLI, or you need programmatic control beyond what the CLI exposes | Granola, HubSpot, Apollo |

**Official CLI path:** skip steps 2-3 (no adapter needed), add only a `shutil.which` + `subprocess.run` probe to `doctor`, and document the install command.

**API adapter path:** follow all steps below.

---

## Step 2 — Add config

Edit `gtm/gtm/config.py`. Three things to add:

**1. Credential** -- `SecretStr | None` so the CLI loads even if the tool is not configured yet.

**2. Base URL + each endpoint** -- one `str` field per path so a vendor path change is a one-line `.env` edit, not a code change.

**3. `.env.example`** -- document the variable name; never commit the actual key.

```python
# gtm/gtm/config.py  (inside class Config)

# ── Credentials ──
acme_api_key: SecretStr | None = None

# ── Endpoints ──
acme_base_url: str = "https://api.acme.io/v2"
acme_widgets_path: str = "/widgets"
acme_widget_detail_path: str = "/widgets/{id}"   # format with .format(id=...) at call site
```

```bash
# gtm/.env.example  (one line per credential)
ACME_API_KEY=          # your Acme API key (acm_live_...)
```

Pydantic-settings maps `ACME_API_KEY` in `.env` to `acme_api_key` in `Config` automatically (case-insensitive, underscores match).

---

## Step 3 — Write the adapter module

Create `gtm/gtm/tools/acme.py`. Model it on `granola.py` -- the smallest complete adapter in the codebase.

### Full copy-paste template

```python
"""Acme (API) — <one-line description of what this tool does>.

Auth: API key (acm_live_...) passed as Bearer token.
Rate limit: TODO: confirm with vendor docs.
Pagination: cursor-based (next_cursor field in response).
"""
import typer

from ..config import config
from ..http import request, show

app = typer.Typer(no_args_is_help=True, help="Acme (API): <short description>")


# ── Internal helpers ────────────────────────────────────────────────────────

def _headers() -> dict:
    """Build auth headers. Exits with a clear error if the key is missing."""
    key = config.acme_api_key
    if not key:
        typer.secho("ACME_API_KEY not set in gtm/.env", fg=typer.colors.RED)
        raise typer.Exit(1)
    return {
        "Authorization": f"Bearer {key.get_secret_value()}",
        "Content-Type": "application/json",
    }


def _url(path: str) -> str:
    """Resolve a path against the configured base URL."""
    return config.acme_base_url.rstrip("/") + path


# ── READ commands (safe to run directly) ────────────────────────────────────

@app.command("list")
def list_widgets(
    limit: int = typer.Option(20, help="max items to return"),
    cursor: str = typer.Option(None, help="pagination cursor from a previous response"),
):
    """List Acme widgets."""
    params: dict = {"limit": limit}
    if cursor:
        params["cursor"] = cursor
    show(request("GET", _url(config.acme_widgets_path), headers=_headers(), params=params))


@app.command("get")
def get_widget(widget_id: str):
    """Get a single Acme widget by ID."""
    path = config.acme_widget_detail_path.format(id=widget_id)
    show(request("GET", _url(path), headers=_headers()))


@app.command("search")
def search_widgets(
    query: str = typer.Option(..., "--query", "-q", help="search string"),
    limit: int = typer.Option(20, help="max items to return"),
):
    """Search Acme widgets by name or tag."""
    show(request(
        "GET",
        _url(config.acme_widgets_path),
        headers=_headers(),
        params={"q": query, "limit": limit},
    ))


# ── WRITE commands (must go through the review gate) ────────────────────────
# Every write/send/delete action MUST be presented as a decision list via
# `gtm review new` and executed only after the user approves on the review page.
# See: docs/browser-review-loop.md and docs/security.md

@app.command("create")
def create_widget(
    name: str = typer.Option(..., "--name", help="widget name"),
    # TODO: add vendor-specific fields
):
    """Create an Acme widget.

    WRITE action -- render as a decision list and confirm via the review gate
    before calling this command. See docs/browser-review-loop.md.
    """
    body = {"name": name}
    show(request("POST", _url(config.acme_widgets_path), headers=_headers(), json=body))
```

Key rules baked into the template:

- `_headers()` calls `config.acme_api_key` and fails with a clear message -- never silently returns an empty dict.
- `_url()` always strips a trailing slash from `base_url` before joining, matching every other adapter.
- `show(request(...))` is the shared response-printing pipeline from `http.py`; do not reinvent it.
- READ verbs (`list`, `get`, `search`) are safe to call directly.
- WRITE verbs (`create`, `update`, `delete`, `send`, `enroll`) must be routed through the review gate before execution (see step 6).

---

## Step 4 — Register in cli.py

Two edits to `gtm/gtm/cli.py`:

**1. Import and register the sub-app:**

```python
# top of cli.py -- add to the tools import line
from .tools import apollo, deals, granola, hubspot, linkedin, review, acme  # + acme

# after the existing app.add_typer lines
app.add_typer(acme.app, name="acme")
```

**2. Add a probe to the `doctor` command:**

```python
# inside the doctor() function, after the Granola block

# Acme (API -- widget data)
if config.acme_api_key:
    r = request(
        "GET",
        config.acme_base_url.rstrip("/") + config.acme_widgets_path,
        headers={"Authorization": f"Bearer {config.acme_api_key.get_secret_value()}"},
        params={"limit": 1},
    )
    code = r.status_code if r else "-"
    console.print(f"Acme      : key [green]set[/green] · widgets probe HTTP {code} ({interpret(r.status_code) if r else 'no response'})")
else:
    console.print("Acme      : [yellow]no key in .env[/yellow]")
```

The probe uses the smallest safe read endpoint (one item) to verify auth without side effects.

---

## Step 5 — Add a PATH shim (optional)

If you want `acme` as a top-level command (without typing `gtm acme ...`), copy the `deals` shim pattern:

```bash
# ~/.local/bin/acme  (chmod +x)
#!/bin/bash
exec "/path/to/your-vault/gtm/.venv/bin/gtm" acme "$@"
```

Make it executable and verify it is on PATH:

```bash
chmod +x ~/.local/bin/acme
which acme   # should print ~/.local/bin/acme
acme --help
```

---

## Step 6 — Test

```bash
# 1. Verify config loads and key is found
gtm doctor

# 2. Run a read command -- safe, no side effects
gtm acme list
gtm acme get <some-id>

# 3. Check the probe line in doctor output:
#    "Acme      : key set · widgets probe HTTP 200 (OK)"
#    200 = auth working
#    401 = key wrong or expired
#    403 = key valid but missing scope (check vendor dashboard)
#    404 = path changed (override acme_widgets_path in .env)
```

**Write/send verbs.** Any action that creates, modifies, sends, or deletes data in the vendor system MUST:

1. Be rendered as a decision list via `gtm review new -f <items.json>` (one item per action).
2. Wait for user approval on the review page (`gtm review wait <id>`).
3. Execute only against the items marked Approve.

See [browser-review-loop.md](browser-review-loop.md) for the full pattern and [security.md](security.md) for the confirm-before-outward rule.

---

## Step 7 — Vendor role table

Use this to identify which tools already cover a role and where a new vendor fits.

| Role | Existing tool | Typical new vendor |
|---|---|---|
| CRM (deals, contacts, pipelines) | HubSpot | Salesforce, Pipedrive |
| Newsletter / broadcast email | Customer.io (cio CLI) | Beehiiv, Mailchimp, Loops |
| Cold outbound (sequences) | Smartlead | Instantly, Lemlist, Outreach |
| Lead data / enrichment | Apollo | Clearbit, Clay, ZoomInfo |
| Meeting notes / transcripts | Granola | Fireflies, Otter.ai, Gong |
| Social publishing | LinkedIn (browser review loop) | Twitter/X, Bluesky |
| Ads / paid acquisition | (not yet wired) | Google Ads API, Meta Marketing API |

If a vendor fits an existing role, check whether the existing adapter can be extended with a new sub-command before adding a separate tool.

---

## Related docs

- [tool-layer.md](tool-layer.md) -- architecture of the tools directory
- [security.md](security.md) -- confirm-before-outward rule and credential hygiene
- [browser-review-loop.md](browser-review-loop.md) -- the review gate for write/send actions
- [README.md](README.md) -- full GTM command center overview
