# Voice and Guardrails

How to keep every piece of generated content on-brand and honest: a voice spec the AI reads
before drafting, a live tone meter that scores drafts in the browser, honesty and positioning
rules the AI enforces, and a compounding content log that prevents stale repetition.

The system applies today to LinkedIn posts (via `/ops-linkedin-daily`). The architecture
generalises to any channel with its own spec and its own `*_*` config namespace.

---

## 1. The Voice Spec: How It Sounds vs. What It's About

Before writing a single word, the AI reads
`wiki/growth/community/LinkedIn Voice and Tone.md`. That file is the human-readable source of
truth. It separates two orthogonal concerns:

### HOW it sounds (tone)

The one-line target from the spec:

> "Write like one sharp operator talking to another over coffee. Confident, specific, a little
> dangerous. Professional, never corporate. A tad engagement-farmy, never cringe."

Tone do-list (from the spec):
- Talk to one person. First and second person ("I", "you"). Contractions.
- Short sentences. One idea per line. Heavy white space.
- Read it out loud. Kill "in today's evolving landscape."
- Be concrete. Real numbers. Real specifics.
- Have a spine. A real opinion. A little vulnerability.
- Vary the rhythm. Fragments are fine.

Tone don't-list:
- No em-dashes, no en-dashes. (Hard rule.)
- No banned words: `streamline`, `leverage`, `seamless`, `game-changer`, `unlock`.
- No corporate filler: `synergy`, `circle back`, `move the needle`, `at the end of the day`.
- Polished-by-committee underperforms conversational.

### WHAT it's about (topic)

The spec defines four topic lanes:

| Lane | What it means | Source |
|------|---------------|--------|
| **Relatable** | A moment the ICP actually lives (junk portal inheritance, agency markup, stalled demo) | Audit Journal, deal folders |
| **Fresh** | Rides a current conversation from this week | `linkedin research` (Reddit + HubSpot Community + RSS) |
| **A little scandalous** | A take 30% will push back on; a receipt; a sacred cow named | Contrarian Take, Receipts, Agency Math |
| **Extremely value-driven** | A teardown with specifics nobody shares: exact steps, real numbers, actual math | Steal This, Agency Math, Audit Journal |

Banned as topics: generic advice. If the post could have been written without ever opening a
HubSpot portal, it is not our post.

---

## 2. The Non-Negotiable Positioning and Honesty Rules

These rules exist in the spec and are enforced by feedback memory written back after every
approved draft.

### Positioning rule

The author is the **founder of the company** who audits portals and builds systems for
Solutions Partners. The stance is auditor-to-agency, never agency-owner.

From the spec:
> "You speak **to** agency owners, RevOps, and partners; you are **not** one of them."

The trap is a CTA that accidentally casts you as the audience. The spec documents a live
example from 2026-06-10:
> "A close like 'I'll tell you where *mine* would break' implies you run an agency retainer
> -- wrong, and it killed a draft on 2026-06-10."

The correct close: "I audit these for a living, so I have a guess. Tell me I'm wrong."
(The final posted copy in the Post Log shows exactly this fix applied.)

The memory file at `memory/gtm-copy-positioning.md` carries the same rule for cross-session
persistence: write AS the founder/auditor, TO agencies/RevOps; never pose as an
agency owner.

### Honesty rule

From the spec:
> "Spicy never means fake. If you frame a number or a story as a real event, it must be real.
> Use 'what I keep finding' / 'the pattern is' for recurring truths instead of inventing a
> single fake client."

Numbers, pricing, outcomes cited in a post must be verifiable. If sourced from research, the
sources block in the draft JSON records them. First-comment attribution is the standard (as
in the logged post: "Hashtag: #HubSpot · Sources (first comment): HubSpot company news,
MarTech, Fortune.").

---

## 3. The Live Tone Meter

The review page rendered by `gtm/gtm/tools/linkedin_render.py` runs a second row of checks
(labelled "Tone meter -- authentic · conversational · a tad farmy (not generic)") beneath the
basic checks row. All thresholds and word lists are injected from the config object at render
time; the JS never hardcodes a number.

### How it works

The `tone()` function in `linkedin_render.py` receives the live textarea value and the `cfg`
object (which carries `hookMax`, `sentenceMax`, `paragraphMax`, `cringe[]`, and `generic[]`
arrays). It runs seven checks and renders each as a badge: green (`good`), amber (`warn`), or
red (`bad`).

```js
// Source: gtm/gtm/tools/linkedin_render.py -- tone() function (abridged)
function tone(v){
  var hookMax = cfg.hookMax || 120, sentMax = cfg.sentenceMax || 20, paraMax = cfg.paragraphMax || 280;
  ...
  t.appendChild(tbadge('hook ' + hook.length + (hook.length <= hookMax ? ' (fits the fold)' : ' (trim it)'),
    hook.length > 0 && hook.length <= hookMax ? 'good' : 'warn'));
  ...
  t.appendChild(tbadge('~' + avg + ' words/sentence', avg > 0 && avg <= sentMax ? 'good' : 'warn'));
  ...
  t.appendChild(tbadge(longest <= paraMax ? 'tight blocks' : 'wall of text (add line breaks)', ...));
  ...
}
```

### The seven tone checks

| Check | What it measures | Config field | Pass condition |
|-------|-----------------|--------------|----------------|
| Hook length | Chars in the first non-empty line | `linkedin_hook_max_chars` (default 120) | `<= hookMax` |
| Avg sentence length | Total words / sentence count (split on `.!?`) | `linkedin_sentence_max_words` (default 20) | `<= sentMax` |
| Wall of text | Longest paragraph block, bullet-list aware | `linkedin_paragraph_max_chars` (default 280) | `<= paraMax` |
| Comment bait | Presence of at least one `?` in the post | (none -- binary check) | `true` |
| First/second-person count | Regex match on `i`, `i'm`, `i've`, `you`, `your`, `we`, `us`, `let's` | (none -- threshold 4) | `>= 4` |
| Farm cringe | Any word from `linkedin_cringe_words` found (case-insensitive) | `linkedin_cringe_words` | zero matches |
| Generic-advice tells | Any phrase from `linkedin_generic_words` found | `linkedin_generic_words` | zero matches |

The paragraph/wall-of-text check is bullet-aware: a block whose majority of lines begin with
`-`, `*`, a bullet character, or a numbered list marker is measured by its longest individual
line rather than its total character count. This prevents a clean bullet list from triggering
the wall-of-text warning.

The basic `checks()` function (the first row) adds four additional hard checks: total
character count (pass <= 3000), em/en-dash count (pass = 0), banned words (from
`linkedin_banned_words`), and hashtag count (pass <= 1).

All badges are live -- they update on every keystroke. The post is ready to approve when the
tone row is all green.

---

## 4. Tuning: Config-Driven Thresholds and Word Lists

Every threshold and every word list lives in `gtm/gtm/config.py` under `linkedin_*` keys.
Override in `gtm/.env` without touching code.

```python
# gtm/gtm/config.py (linkedin tone meter block)
linkedin_hook_max_chars: int = 120       # line 1 fits before "...see more" fold
linkedin_sentence_max_words: int = 20    # avg words/sentence above this reads less conversational
linkedin_paragraph_max_chars: int = 280  # longest block; bigger = wall of text, add line breaks
linkedin_banned_words: str = "streamline,leverage,seamless,game-changer,unlock"
linkedin_cringe_words: str = (
    "thoughts?,agree?,👇,comment below,drop a,dm me,smash that,follow for more,"
    "let that sink in,read that again,who's with me,link in comments,comment 'yes'"
)
linkedin_generic_words: str = (
    "in today's,rapidly evolving,top 5 tips,top 10 ways,consistency is key,move the needle,"
    "at the end of the day,circle back,in this post i,here are some tips,without further ado,"
    "needless to say,it's no secret"
)
```

To tighten the hook threshold (e.g., enforce 100 chars): add `LINKEDIN_HOOK_MAX_CHARS=100` to
`gtm/.env`. The review page picks it up on the next `linkedin draft` run without any code
change.

To add a new cringe phrase: append it to `LINKEDIN_CRINGE_WORDS` in `.env` (comma-separated,
matching the default format). The JS receives the updated list in the rendered `cfg` object.

---

## 5. The Content Log: Rotating Pillars and Never Repeating a Hook

`wiki/growth/community/LinkedIn Post Log.md` is a compounding append-only record. On every
approved-and-posted draft, `/ops-linkedin-daily` appends an entry with:

- Date, pillar, hook type, topic/angle
- Why it worked (brief retrospective)
- Full final copy (as posted)
- Hashtag and sources attribution
- Lessons captured (positioning fixes, tone issues)

The pillar rotation tracker table at the top of the log gives a quick view across all posted
dates, preventing pillar repetition in back-to-back posts.

Current tracker from the log:

| Date | Pillar | Hook | Topic | Status |
|------|--------|------|-------|--------|
| 2026-06-10 | Agency Math (contrarian) | Number Drop | HubSpot outcome-based Breeze pricing vs. the retainer | Posted |

Before drafting, the AI reads the log to check the last 2-3 pillars and the last 5 hooks.
This prevents the system from recycling a hook it already used or hitting the same pillar two
sessions in a row. Over time the log also becomes a mine of angles: if a post performed well,
the lessons block records why, and future sessions can build adjacent takes from the same vein.

---

## 6. How This Generalises to Other Channels

The same four-component pattern applies to any channel:

1. **Spec page** in `wiki/` -- human-readable, AI reads it before drafting. Covers HOW it
   sounds and WHAT it's about for that channel's audience.
2. **Config namespace** in `gtm/gtm/config.py` -- e.g., `newsletter_*` for the newsletter.
   Thresholds and word lists that the render function receives.
3. **Render + tone meter** -- a channel-specific render module (like `linkedin_render.py`)
   with a `tone()` equivalent scoring the channel's rules.
4. **Post log** in `wiki/` -- append-only record of every sent item.

The newsletter already has a config namespace (`newsletter_feeds_path`,
`newsletter_research_dir`) and a feeds registry (`newsletter_feeds.json`). A newsletter tone
meter and post log follow the same pattern, pointed at a newsletter voice spec (e.g. a
`Newsletter Editorial Principles` page) instead of the LinkedIn one.

---

## Related docs

- [browser-review-loop](browser-review-loop.md) -- how the HTML review page works end to end
- [research-engine](research-engine.md) -- RSS + web search stack that feeds fresh topics
- [knowledge-base](knowledge-base.md) -- the Obsidian wiki structure the AI reads for context
- [README](README.md) -- vault overview and skill index
