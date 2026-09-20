---
name: research-sources
description: Use for ANY research, deep research, market scan, competitor check, validation, or demand/signal hunt — and whenever the user says "research", "deep research", "validate this", "find signal", "find demand", "what do users say", "what do people think of X", "check the competition", "is anyone building this", or asks what real users report about a tool, product, or company. Holds the required source list, the order to call them in, and the cost rules. Load it BEFORE the first search.
---

# Research sources — pull all of them, cheap ones first

Every research task uses **all** the sources below. If one is down, say so plainly in the
deliverable and fall back. Never silently skip a source — a missing source is a hole in the answer,
and the reader cannot see it unless you say it.

## Order — always cheapest first

### 1. Agent-Reach — free, broad, do this first

The `agent-reach` skill is a router, not a CLI. It picks a free backend per platform: Exa whole-web
search, any webpage via Jina, GitHub via `gh`, RSS, V2EX, YouTube (+ Groq transcription), Xiaoyuzhou
podcasts, Bilibili — all no-login. Plus **Twitter/X, Reddit, XiaoHongShu and Bilibili subtitles
through OpenCLI**, which reuses your logged-in Chrome.

Run `agent-reach doctor --json` to see which backend is live right now.

**`doctor` lies. Trust a live call instead.** On 2026-09-02 it reported Reddit and XiaoHongShu as
"OpenCLI 已安装，但 Chrome 扩展未安装" while `opencli doctor` showed the extension connected
(profile `<your-chrome-profile>`, v1.0.20) and Reddit searches were returning real threads. Read `doctor` as a
hint about *which backend would be tried*, never as proof a source is down. One real query costs a
few seconds and settles it.

Lean on this hard. It covers web reads, YouTube transcripts, GitHub and niche CN sites for free —
use it before spending any Firecrawl or Apify credit.

Note `agent-reach` really is a router with no content subcommands — `agent-reach twitter …` is an
error. Content comes from `opencli <site> …`.

### 2. Reddit — first-hand user signal, highest trust

**Working as of 2026-09-02.** (It was down 2026-08-15 → 2026-09-02: OpenCLI installed, Chrome
extension not. The extension is installed now.)

**Reddit is OpenCLI-only in this stack.** Jina gets 403 on `reddit.com` and `old.reddit.com`, and
Firecrawl does not support Reddit at all.

```
opencli reddit search "<query>" --subreddit <sub> --sort relevance --time all --limit 12 -f json
opencli reddit read <post-id> --sort top --limit 30 --depth 2 --replies 3 --max-length 700 -f json
opencli doctor
```

Needs Chrome open on profile `<your-chrome-profile>` and logged into Reddit — desktop only, never headless.

`read` returns a **flat list**, not a nested post object: one item with `"type": "POST"` carrying
the body in `text`, then sibling comment items. There is no `comments_list` key — parse by `type`.
Default `--depth 1` collapses replies into `[+N more replies]` stubs and loses the best material;
use `--depth 2 --replies 3`. Comment counts in search results are the true totals, so a thread
listed at 90 comments that returns 20 means the read was under-parameterised, not that Reddit lied.

Set `PYTHONIOENCODING=utf-8` before piping output through Python on Windows, or emoji in
titles crash the print with `UnicodeEncodeError` on cp1252.

Known limits, plan around them:
- OpenCLI **truncates post bodies at ~2000 chars**. A long listicle post loses its tail with no way
  to get it back.
- Reddit's own search relevance is poor. Bare brand names return unrelated viral threads.
  `--sort new` with a broad query ignores the query and returns the global feed. Only
  `--subreddit`-scoped searches or long distinctive phrases work. Budget for wasted queries.

**Dead — do not try these:** PRAW (401 on script-type apps), and the Smithery Reddit
MCP (its connected Reddit account returns `410 EXPIRED`).

Weight Reddit threads as honest operator signal against vendor blogs and "how I made $X" posts. Tag
sources by who benefits when it matters.

### 3. Firecrawl MCP — web search and page content (credit-metered)

`mcp__firecrawl__firecrawl_search` is the main web search — richer than the built-in one, and it
returns content inline, so prefer it over scrape-then-search loops. Then `firecrawl_scrape` /
`firecrawl_crawl` / `firecrawl_extract` for specific pages.

The 1-credit refund via `firecrawl_search_feedback` has been returning `400 INVALID_BODY` — try it,
but do not count on the refund.

### 3b. X / Twitter — works, but the adapter needs a warm tab

**Working as of 2026-09-02**, via `opencli twitter search "<query>" --limit 12 -f json`. Chrome is
logged in; `twitter-cli` is installed but unauthenticated, so OpenCLI is the live route.

If it returns `Pre-navigation to https://x.com failed: Navigation rejected`, **do not conclude the
site is blocked or the extension lacks permission** — that was guessed once and was wrong. Open a
tab first and retry:

```
opencli browser t1 open "https://x.com/search?q=test"
opencli twitter search "<query>" --limit 12 -f json
```

Root cause is still unproven. If it recurs, run `--trace retain-on-failure` and read the trace
rather than theorising again.

Use X to find that a thing exists and to measure how loudly, not to judge it — a 4,000-like thread
about protein prices is a real demand signal; a launch post is not an evaluation.

### 3c. Xiaohongshu — may be unreachable from your network

On the machine this was written on, `opencli browser t3 open "https://www.xiaohongshu.com/explore"` landed on
`chrome-error://chromewebdata/`. That is a **network-level** failure before any page loads — not a
login, not an extension permission, not fixable by a click here. It reports the same
`Navigation rejected` string as the X problem above, which is why the two get confused. Say it is
unreachable in the deliverable and move on.

### 4. Apify MCP — X, LinkedIn, Instagram only (credits + slow Actor runs)

`search-actors` to find the right Actor, then `call-actor`, then `get-actor-output`. Use this only
for what nothing else reaches. Verified Actors: `neatrat/google-play-store-reviews-scraper` works;
`thewolves/google-play-reviews-scraper` is broken, returns `noResults` for every valid input.

Play reviews, re-verified 2026-09-02 — this exact input returned 294 rows in 13s for ~0.002 CU:

```json
{"appIdOrUrl": "com.healthifyme.basic", "maxReviews": 300, "sort": "NEWEST", "lang": "en", "country": "in"}
```

Then `get-dataset-items` with `fields="rating,body,date"` — projecting fields is the difference
between a readable pull and one that floods the context. Page it: the run's rows are cheap, your
context is not. **Quote counts out of what you actually read**, never out of what the run scraped —
if you read 180 of 300, every number says 180.

## Cost rules

Full coverage with the fewest paid calls. Never drop a source to save credits — call it leaner.

- **Free before paid.** Agent-Reach and OpenCLI first, always. Firecrawl and Apify only for what
  those cannot reach.
- **Fewer, broader calls.** One well-formed query beats many narrow ones. Cap `limit` / `maxItems`.
  Do not fan out one call per keyword when a single search covers them.
- **Never re-fetch.** Reuse what you already pulled this session. For Apify, read more from the same
  run with `get-actor-output` / `get-dataset-items` using the `datasetId` — do not re-run the Actor.
- **On a 429 or credit cap:** say so plainly, fall back to a cheaper source, and note the gap in the
  deliverable.

## Other verified routes

- **Jina Reader** — `curl -s https://r.jina.ai/<URL>` reads any normal page, free. This is also how
  to read a LinkedIn profile URL; the full LinkedIn scraper is not set up.
- **Google Play listings** via Jina — good for rating, review count, downloads, last-updated.
- **Meta Ad Library** via Firecrawl scrape of
  `facebook.com/ads/library/?active_status=active&ad_type=all&country=IN&q=<kw>&search_type=keyword_unordered`
- **Trustpilot** — **use `firecrawl_search`, not Jina.** A plain Jina fetch returns page chrome
  only (the text is JS-gated), but a Firecrawl search naming the brand returns full review bodies
  inline in the result description — around twenty of them, free of a separate scrape call.
- **Xueqiu** (CN stocks) — down. Xueqiu API returns HTTP 400, no login cookie. Not set up.

## Judging what you find

Roughly 70% of what surfaces on a trending tool or product name is promotional. Three filters:

1. **Bot subreddits.** A ring of subs reposts every trending repo in an identical auto-generated
   template — always 1 point, 0 comments. Discard them.
2. **X is amplification, not evaluation.** All-caps launch posts with star counts carry no
   evaluative signal. Use X to find that a thing exists, not to judge it.
3. **Real signal is in comments and issues.** The launch post is written by the author; the 40th
   comment is written by someone who installed it. A GitHub issue that challenges a claim, plus the
   maintainer's reply underneath, is worth more than the whole README.
4. **Founders pitching in the replies are noise *and* data.** Discard them as opinion — they are
   selling. Then count them: 18 different people pitching their own AI food-photo tracker inside
   the threads you read is a measurement of how crowded that lane is, and it is a better one than
   any market report. Report the count, name the products.

When a tool publishes a benchmark, go straight to its methodology and look for the paragraph that
costs the author something. If there is no such paragraph, discount the number.
