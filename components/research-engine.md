# RSS Research Engine

The research engine aggregates fresh, relevant material from public RSS sources before any content draft is written. No API keys required. All configuration lives in committed JSON files and `.env` tunables, not code.

## What and Why

Before drafting a LinkedIn post or newsletter, the engine pulls live signal from Reddit subreddits and search, the HubSpot Community forum, and Google News topic queries. The goal: ride what is actually being discussed right now, not a static prompt written three months ago.

Source choices are deliberate:

| Source type | Why |
|---|---|
| Reddit subreddits (`r/hubspot`, `r/RevOps`, `r/sales`, etc.) | Authentic audience pain, phrased the way buyers actually phrase it |
| Reddit search RSS | Cross-subreddit coverage for a keyword without needing the Reddit API |
| HubSpot Community boards | Product pain from real users; high signal for a HubSpot-adjacent audience |
| Google News queries | Competitor moves, product launches, industry news |

None of these require an API key. Every feed is an RSS URL, fetched with `httpx` and parsed with `feedparser`.

## Feed Registry

Feeds are declared in a committed, editable JSON file. The LinkedIn profile reads `linkedin_feeds.json`; the newsletter profile reads `newsletter_feeds.json`. Adding or removing a feed is a JSON edit with no code change.

### Registry shape (`linkedin_feeds.json`)

```json
{
  "providers": {
    "google_news": "https://news.google.com/rss/search?q={q}&hl=en-US&gl=US&ceid=US:en",
    "reddit_search": "https://www.reddit.com/search.rss?q={q}&sort=top&t=week",
    "hubspot_board": "https://community.hubspot.com/rss/board?board.id={q}"
  },
  "feeds": [
    { "name": "r/hubspot", "url": "https://www.reddit.com/r/hubspot/.rss", "type": "reddit" },
    { "name": "HubSpot Community (all)", "url": "https://community.hubspot.com/rss/Community", "type": "hubspot" }
  ],
  "queries": [
    { "provider": "google_news", "q": "HubSpot AI when:7d", "type": "news" },
    { "provider": "reddit_search", "q": "hubspot", "type": "reddit" },
    { "provider": "hubspot_board", "q": "sales", "type": "hubspot" }
  ],
  "rank_keywords": ["ai", "hubspot", "revops", "gtm", "workflow", "automation", "agent"]
}
```

**`providers`** - URL templates with a `{q}` token. The token is URL-encoded at runtime by `build_feeds()`.

**`feeds`** - Fixed URLs with no query expansion. Pulled as-is every run.

**`queries`** - Each entry names a provider and a query string. `build_feeds()` expands them into full feed objects at runtime. The `type` field (`reddit`, `hubspot`, `news`, `updates`) tags the source for ranking and display.

**`rank_keywords`** - Boost terms used in scoring. The full LinkedIn registry includes: `ai`, `claude`, `anthropic`, `openai`, `gpt`, `chatgpt`, `perplexity`, `gemini`, `hubspot`, `breeze`, `revops`, `gtm`, `go-to-market`, `crm`, `workflow`, `automation`, `agent`, `pipeline`, `migration`, `audit`, `onboarding`, `integration`, `outbound`.

## Profiles

The engine is channel-agnostic. A profile is a (registry path, output dir) pair resolved from config. Wired in `research.py`:

```python
_PROFILE_REGISTRY = {"linkedin": "linkedin_feeds_path", "newsletter": "newsletter_feeds_path"}
_PROFILE_DIR      = {"linkedin": "linkedin_research_dir", "newsletter": "newsletter_research_dir"}
```

The corresponding `config.py` settings:

| Setting | Default |
|---|---|
| `linkedin_feeds_path` | `gtm/linkedin_feeds.json` |
| `linkedin_research_dir` | `gtm/linkedin/research/` |
| `newsletter_feeds_path` | `gtm/newsletter_feeds.json` |
| `newsletter_research_dir` | `gtm/newsletter/research/` |

`profile_registry(profile)` and `profile_dir(profile)` resolve these at runtime; the CLI passes the profile name, not a hardcoded path.

## Ranking

Every item that survives the recency window gets a score from `_score()`:

```python
_W_KEYWORD = 2.0   # per distinct rank_keyword found in title + summary
_W_RECENCY = 3.0   # times the recency fraction (1.0 = brand new, 0.0 = at the window edge)
_SOURCE_BONUS = {"updates": 1.0, "reddit": 1.0, "hubspot": 1.0, "news": 0.5}
```

Full formula:

```
score = (2.0 * keyword_hits) + (3.0 * recency_fraction) + source_bonus
```

`recency_fraction` = `max(0.0, 1.0 - age_days / research_days)`. An item published today on a 7-day window scores `3.0` on recency alone. An item at day 6 scores `0.43`. An item whose date is not trusted (see below) scores `0.0` on recency regardless of the reported date.

Items are sorted descending by score. Deduplication is by normalized title (lowercased, collapsed whitespace) so the same story from two feeds appears once.

## The Freshness Trap

**This is the most operationally important section.**

### The problem

An RSS `pubDate` is the date the feed was updated, not the date the underlying content was originally published. PR wire services and news aggregators routinely re-syndicate old press releases with a fresh timestamp. A real example: an April 23 story appeared in the research output as "0.7d old" because EIN News re-posted it on June 9.

If the engine rewarded that freshness credit at face value, an outdated press release would outrank genuinely fresh Reddit threads and HubSpot Community posts.

### The fix

`research_wire_publishers` in `config.py` is a comma-separated denylist of known wire/aggregator publishers:

```
EIN News, EINPresswire, EIN Presswire, Business Wire, Businesswire,
PR Newswire, PRNewswire, GlobeNewswire, Globe Newswire, ACCESS Newswire,
Newsfile, PRWeb, openPR, Yahoo Finance, Yahoo, MSN, Morningstar,
Benzinga, StreetInsider, GuruFocus, Cyprus Shipping News, Markets Insider
```

For Google News items, the publisher is extracted from the title suffix (titles end in ` - Publisher Name`). If that publisher matches the denylist, `date_trusted` is set to `False`. `_score()` short-circuits to `recency = 0.0` for untrusted dates. The table renderer marks these items `[red]0.7d⚠[/red]` and the table caption reads: "⚠ = date is a wire/aggregator re-post, not the original. Verify before using."

Reddit and HubSpot Community items are self-posted; their timestamps are treated as trusted unconditionally (`ftype != "news"` means `date_trusted = True`).

### The rule for drafting

Never draft a post anchored on a news item marked with `⚠` without first opening the article, reading the original dateline, and confirming it is actually recent. The engine flags the risk; you confirm the fact.

## Browser Headers

HubSpot Community is behind Cloudflare. Reddit also blocks bare HTTP clients. Both return `403` if the `User-Agent` looks like a script. The engine sends a full browser-like header on every request, configured in `config.py`:

```
research_user_agent = "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/126.0.0.0 Safari/537.36"
```

Override `RESEARCH_USER_AGENT` in `gtm/.env` if needed. The `Accept` and `Accept-Language` headers are also set to match a browser fetch.

## Engine Tunables

All engine-level settings live in `gtm/.env` (or the `Config` defaults in `config.py`):

| Setting | Default | Purpose |
|---|---|---|
| `research_days` | `7` | Recency window; items older than this are dropped |
| `research_max_per_feed` | `12` | Max entries pulled from each individual feed |
| `research_workers` | `8` | Concurrent feed fetches (ThreadPoolExecutor) |
| `research_user_agent` | Chrome 126 UA string | User-Agent header sent to every feed |
| `research_wire_publishers` | 23-entry denylist | Publishers whose freshness credit is zeroed |
| `gtm_http_timeout` | `30` | Per-request timeout in seconds |

## Daily Flow

### Run research before drafting

```bash
# LinkedIn profile, default 7-day window
gtm linkedin research

# Narrow to reddit sources only
gtm linkedin research --type reddit

# Add ad-hoc Google News queries on top of the registry
gtm linkedin research -q "HubSpot Breeze copilot" -q "RevOps consolidation 2026"

# Extend the window
gtm linkedin research --days 14

# Dump raw JSON for programmatic use
gtm linkedin research --json
```

The `run()` function in `research.py` fetches all feeds concurrently, filters to the recency window, dedupes by title, scores every item, and returns the sorted list. `save_results()` persists the run to `<research_dir>/<UTC-date>.json` so a draft can cite which items it rode on.

### What Claude does with it

1. Run `gtm linkedin research` and read the ranked table.
2. Identify the highest-scoring items with trusted dates.
3. For any item with `⚠`, open the article and verify the original dateline.
4. Pick one verified-fresh angle, draft the post, and send it through the browser review loop.

Research runs are saved to disk (`linkedin/research/YYYY-MM-DD.json`) so past sessions are auditable. If a post gets called out for referencing stale news, the saved run is the paper trail.

## Related docs

- [voice-and-guardrails](voice-and-guardrails.md)
- [browser-review-loop](browser-review-loop.md)
- [README](README.md)
