# claude-research-stack

The research setup I use in [Claude Code](https://claude.com/claude-code): one skill that makes
every research task pull **all** the sources, cheapest first, and say so out loud when one is down.

The skill is the whole point. It is not a search wrapper — it is the written-down doctrine of which
source to call in what order, what each one's known failure looks like, and how to tell real user
signal from marketing. Most of it was learned by getting it wrong once.

---

## What you get

`skills/research-sources/SKILL.md` — loads itself whenever you say "research", "validate this",
"what do people say about X", "check the competition". It then drives:

| Source | Cost | Reaches |
|---|---|---|
| **agent-reach** | free | whole-web search (Exa), any page (Jina), GitHub, RSS, YouTube transcripts, V2EX, Bilibili, podcasts |
| **OpenCLI** | free | Reddit, X/Twitter, XiaoHongShu, Bilibili subtitles — through your own logged-in Chrome |
| **Firecrawl MCP** | credits | web search with content inline, page scrape/crawl |
| **Apify MCP** | credits | Google Play reviews, LinkedIn, Instagram |

Free ones run first. Paid ones only reach what the free ones cannot.

---

## Setup

### 1. The skill

```bash
git clone https://github.com/aghori3004/claude-research-stack.git
cp -r claude-research-stack/skills/research-sources ~/.claude/skills/
```

Windows PowerShell:

```powershell
Copy-Item -Recurse claude-research-stack\skills\research-sources $env:USERPROFILE\.claude\skills\
```

Restart Claude Code. `/skills` should list `research-sources`.

### 2. agent-reach (free, do this one)

Not mine — install it from **https://github.com/Panniantong/Agent-Reach** and follow its README.
It is the free router the skill leans on hardest. Check it with:

```bash
agent-reach doctor --json
```

### 3. OpenCLI + Chrome extension (free — this is what gets you Reddit and X)

```bash
npm i -g @jackwener/opencli
opencli doctor
```

`doctor` tells you to install the Chrome extension and which profile it binds to. Put that profile
id where the skill says `<your-chrome-profile>`. Rules that matter:

- Chrome must be **open and logged in** to Reddit / X. Desktop only, never headless.
- If X returns `Navigation rejected`, open a tab first — `opencli browser t1 open "https://x.com/search?q=test"` — then retry. It is not a block.
- `doctor` under-reports. A real query settles it, not the health check.

### 4. Firecrawl MCP (needs a key)

Get a key at [firecrawl.dev](https://firecrawl.dev), then:

```bash
claude mcp add --transport http firecrawl "https://mcp.firecrawl.dev/fc-YOUR_KEY_HERE/v2/mcp"
```

The key sits in the URL, so keep that command out of anything you commit.

### 5. Apify MCP (free tier, then credits)

```bash
claude mcp add --transport http apify https://mcp.apify.com
```

First call opens an OAuth login. Verified Actor for Play Store reviews:
`neatrat/google-play-store-reviews-scraper` (`thewolves/...` is broken — returns `noResults` for
every valid input).

### 6. Tell Claude to load it first (optional but worth it)

Paste this into your `~/.claude/CLAUDE.md`:

```markdown
# Research Workflow — always pull all real-signal sources

Any research / deep-research / validation / find-signal task: load the `research-sources` skill
first, before the first search. It holds the source list, the call order, and the cost rules.
Pull the free sources (agent-reach, OpenCLI) before the metered ones (Firecrawl, Apify). Never
silently skip a source — if one is down, say so plainly in the deliverable.
```

---

## Using it

Just ask. "Research what people actually say about <product>." The skill loads, names which backend
it is using, runs free sources first, and flags any source it could not reach.

## Keeping it honest

When a source dies, comes back, or gets replaced, **edit `SKILL.md`** — not your `CLAUDE.md`. The
doctrine belongs next to the commands it describes. A dated note in there ("working as of
2026-09-02") is more useful than a vague claim.

## What is *not* in here

- No API keys. Every one of them is yours to get.
- Not the agent-reach skill itself — that is someone else's work, linked above.
- No wrapper scripts. The skill talks to the CLIs and MCPs directly, on purpose. Fewer moving parts to rot.
