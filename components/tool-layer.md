# The Tool Layer: Config-Driven CLI Architecture

*How Claude runs real tools without hardcoded secrets or brittle paths.*

---

## Overview

The GTM command center is a Python package (`gtm`) that gives Claude direct access to HubSpot, Apollo, Granola, and the LinkedIn/review browser loops. It is built on four principles:

1. **Every credential comes from `.env`** -- never from source code.
2. **Every endpoint is an overridable setting** -- a vendor path change is a one-line `.env` edit.
3. **Every tunable (timeout, retries, batch size) has a named config field** -- no magic numbers buried in code.
4. **One module per tool** -- adding a new integration means adding one file, not touching existing ones.

---

## Package Shape

```
gtm/                          # repo sub-directory
  .env                        # git-ignored; all secrets live here
  .env.example                # committed; documents every variable
  pyproject.toml              # entry point: gtm = "gtm.cli:main"
  gtm/                        # Python package
    __init__.py
    cli.py                    # typer root app; registers sub-apps
    config.py                 # pydantic-settings Config class
    http.py                   # shared HTTP client with retry/backoff
    tools/
      __init__.py
      apollo.py               # typer sub-app: lead search + enrichment
      deals.py                # typer sub-app: deal-folder brain sync
      deals_render.py         # HTML renderer for deal review pages
      granola.py              # typer sub-app: meeting notes / transcripts
      hubspot.py              # typer sub-app: HubSpot CRM reads/writes
      linkedin.py             # typer sub-app: LinkedIn post review loop
      linkedin_render.py      # HTML renderer for LinkedIn review pages
      research.py             # shared RSS research engine
      review.py               # typer sub-app: generic decision-list loop
      review_render.py        # HTML renderer for generic review pages
      assets/
        logo.svg     # brand asset inlined into review pages
```

`cli.py` wires each tool module as a named sub-app:

```python
app = typer.Typer(no_args_is_help=True, help="the GTM command center ...")
app.add_typer(hubspot.app, name="hubspot")
app.add_typer(apollo.app,  name="apollo")
app.add_typer(granola.app, name="granola")
app.add_typer(deals.app,   name="deals")
app.add_typer(linkedin.app, name="linkedin")
app.add_typer(review.app,  name="review")
```

One-off commands like `research` and `doctor` are registered directly with `@app.command()`.

---

## config.py Deep Dive

`config.py` defines a single `Config` class that inherits `pydantic_settings.BaseSettings`. All values load from `gtm/.env`; the class instance `config` is imported by every module.

```python
class Config(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=str(ENV_PATH), env_file_encoding="utf-8",
        extra="ignore", case_sensitive=False
    )
```

### Credentials

Typed as `SecretStr | None = None` so the CLI loads cleanly even when a tool is not yet configured. `None` means the tool is not wired; `doctor` surfaces this explicitly.

```python
hubspot_private_app_token: SecretStr | None = None
apollo_api_key:            SecretStr | None = None
cio_token:                 SecretStr | None = None   # sa_live_... service-account token
smartlead_api_key:         SecretStr | None = None
granola_api_key:           SecretStr | None = None
```

### Tunables

Named fields, not literals scattered through `httpx.get(...)` calls:

```python
gtm_http_timeout: int = 30
gtm_max_retries:  int = 3
gtm_log_level:    str = "info"
```

### Endpoints

Every API path is a config field with a sensible default. If a vendor changes a path, one line in `.env` fixes it without touching code:

```python
hubspot_base_url:           str = "https://api.hubapi.com"
hubspot_deals_path:         str = "/crm/v3/objects/deals"
hubspot_sequences_path:     str = "/automation/v4/sequences"
apollo_base_url:            str = "https://api.apollo.io/api/v1"
apollo_search_path:         str = "/mixed_people/search"
granola_base_url:           str = "https://public-api.granola.ai/v1"
```

The pattern for building a URL anywhere in the codebase:

```python
url = config.hubspot_base_url.rstrip("/") + config.hubspot_deals_path
```

### Other Configured Behaviours

Paths, ports, timeouts, feed registries, tone-meter word lists, OOO regex patterns -- everything that could differ between environments or brands. The full list is in `gtm/gtm/config.py`; `gtm/.env.example` documents every variable with inline comments.

---

## http.py: One Shared Client

All API calls go through a single `request()` function rather than inline `httpx.get()` calls. This centralises retry logic and ensures secrets are handled consistently.

```python
RETRY_STATUS = {429, 500, 502, 503, 504}

def request(method, url, *, headers=None, params=None, json=None) -> httpx.Response | None:
    for attempt in range(config.gtm_max_retries + 1):
        try:
            resp = httpx.request(method, url, headers=headers, params=params,
                                 json=json, timeout=config.gtm_http_timeout)
        except httpx.HTTPError as exc:
            if attempt >= config.gtm_max_retries:
                console.print(f"[red]network error:[/red] {exc}")
                return None
            time.sleep(min(2 ** attempt, 30))
            continue
        if resp.status_code not in RETRY_STATUS:
            return resp
        # 429/5xx: honour Retry-After if present, else exponential backoff, cap 30s
        delay = resp.headers.get("Retry-After")
        time.sleep(min(float(delay) if delay else 2 ** attempt, 30))
    return resp
```

Key properties:

| Property | Detail |
|---|---|
| Timeout | `config.gtm_http_timeout` (default 30s) -- named, not a literal |
| Max retries | `config.gtm_max_retries` (default 3) |
| Backoff | Exponential `2^attempt`, capped at 30s |
| Retry-After | Honoured when present in the response header |
| Retryable statuses | 429, 500, 502, 503, 504 only |
| Secrets | Passed as `headers=` by each caller; never printed, never in the URL |

The companion `interpret()` helper translates status codes to human-readable strings, including a `needs_master` flag for Apollo (which distinguishes 401 from 403 differently).

---

## The `doctor` Command

`gtm doctor` is the first thing to run when setting up or debugging the toolkit. It probes every wired tool in order: key configured? authentication succeeds?

```
────────── GTM toolkit — doctor ──────────
HubSpot   : key set · deals probe HTTP 200 (OK)
Apollo    : key set · search probe HTTP 200 (OK)
Granola   : key set · notes probe HTTP 200 (OK)
Customer.io: cio CLI found · auth authenticated
Smartlead : CLI found · auth configured

Codes: 200 OK · 401 invalid key · 403 valid key but missing scope/not master · 404 path changed (override in .env)
```

For API tools (HubSpot, Apollo, Granola) it fires a minimal live request -- one deal, one search, one note -- using `request()` from `http.py` and prints the HTTP status. For delegated CLIs (Customer.io via `cio`, Smartlead via `smartlead`) it uses `shutil.which()` to confirm the binary is on PATH, then runs the CLI's own auth-status subcommand:

```python
cio = shutil.which("cio")
if cio:
    out = subprocess.run([cio, "auth", "status"], capture_output=True, text=True, timeout=15)
    authed = out.returncode == 0 and '"error":true' not in out.stdout.replace(" ", "")
```

The footer line is printed unconditionally and explains the three most useful status codes plus the 404 self-fix hint, so operators know what to do without reading docs.

---

## PATH Shims

Each top-level command exposed to the shell is a two-line Bash script that `exec`s the venv binary directly. No activation step; no `python -m`.

`~/.local/bin/gtm`:
```bash
#!/bin/bash
# the GTM CLI shim — forwards to the toolkit's venv binary so `gtm` works from anywhere.
exec "/path/to/your-vault/gtm/.venv/bin/gtm" "$@"
```

`~/.local/bin/deals`:
```bash
#!/bin/bash
# Shim for the deal-folder sync (gtm deals ...)
exec "/path/to/your-vault/gtm/.venv/bin/gtm" deals "$@"
```

`~/.local/bin/linkedin`:
```bash
#!/bin/bash
# Shim for the LinkedIn review loop (gtm linkedin ...)
exec "/path/to/your-vault/gtm/.venv/bin/gtm" linkedin "$@"
```

Why shims instead of adding the venv `bin/` to `$PATH` permanently:

- The venv path is absolute and does not depend on `$PWD` or shell state.
- Adding a new sub-command (a new `tools/*.py` file + `app.add_typer(...)`) never requires updating the shim -- the single `gtm` binary covers all sub-commands.
- Alias names (`deals`, `linkedin`) let Claude call a sub-command without the parent verb, keeping skill invocations concise.
- `exec` replaces the shell process rather than forking a child -- no wrapper overhead.

---

## Editable Install

The package is installed in development mode:

```bash
cd /path/to/your-vault/gtm
pip install -e .
```

`pyproject.toml` wires the entry point:

```toml
[project.scripts]
gtm = "gtm.cli:main"
```

Because the install is editable (`-e`), `gtm/gtm/tools/` is on `sys.path` live. Adding a new tool module and registering it in `cli.py` is immediately active -- no reinstall, no shim change.

Dependencies are pinned to minimum-compatible versions:

| Package | Role |
|---|---|
| `typer>=0.12` | CLI framework; sub-apps, flags, help text |
| `httpx>=0.27` | Async-capable HTTP client used in sync mode |
| `pydantic-settings>=2.2` | `.env` loading + type coercion |
| `rich>=13` | Console output (tables, JSON, colour, rules) |
| `feedparser>=6.0` | RSS/Atom parsing for the research engine |

---

## Tool-Agnostic Note

The architecture is language-agnostic. Node.js, Go, or Rust toolkits built with the same principles look different in syntax but identical in structure:

- A config module that reads from environment / a git-ignored file; secrets typed as opaque values, not plain strings.
- A shared HTTP helper with timeout and retry pulled from config.
- One file (or module) per integration, registered in a root dispatcher.
- A `doctor` sub-command that probes each integration independently.
- PATH shims (or symlinks) that `exec` the real binary.

The language choice matters far less than these five constraints. A Python toolkit with hardcoded tokens and a 30-second literal in every `requests.get()` call is far harder to maintain than a Go binary that reads from a config struct and has a `doctor`.

---

## Related Docs

- [adding-a-tool](adding-a-tool.md) -- step-by-step: new module, register, shim, doctor entry, `.env.example` key
- [security](security.md) -- secret hygiene, `.gitignore` rules, `SecretStr`, what `doctor` must never log
- [README](README.md) -- quickstart, first-run checklist
