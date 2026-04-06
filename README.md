# Meta Rate Limiter
### A Claude Code Skill by [@tenfoldmarc](https://www.instagram.com/tenfoldmarc)

Stop getting throttled by Meta's API. This skill knows every rate limit, error code, and quota formula for the Facebook/Instagram Marketing API — and generates code that stays safely under the limits every time.

---

## What It Does

1. **Asks your setup** — access tier (Development or Standard), active ad count, preferred language
2. **Calculates your exact quotas** — tells you precisely how many API calls per hour you can make
3. **Guards Claude's own API calls** — when Claude makes Meta API calls for you (via MCP tools), it automatically paces, batches, and minimizes calls to stay under limits
4. **Generates rate-limited code** — production-ready JavaScript or Python with built-in header monitoring, exponential backoff, and budget change queuing
5. **Diagnoses throttling errors** — paste any Meta error code and get the exact cause and fix
6. **Optimizes existing code** — reviews your Meta API code against a checklist of common rate limit mistakes
7. **Tracks all four Meta usage headers** — X-App-Usage, X-Business-Use-Case-Usage, X-Ad-Account-Usage, and X-FB-Ads-Insights-Throttle

---

## Requirements

- A Mac, Linux, or Windows computer
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed and working
- A Meta Developer App (free — create one at [developers.facebook.com](https://developers.facebook.com))

That's it. No MCP servers, no API keys in config, no external tools. The skill generates code — you bring your own Meta access token when you use it.

---

## Install

### Step 1 — Open your terminal

**Mac:** Press `Command + Space`, type **Terminal**, hit Enter.
**Windows:** Press `Win + R`, type **cmd**, hit Enter. (If you have Git Bash, use that instead.)
**Linux:** Open your terminal app.

### Step 2 — Run this command

Copy-paste this entire line and hit Enter:

```bash
git clone https://github.com/tenfoldmarc/meta-rate-limiter-skill ~/.claude/skills/meta-rate-limiter
```

You'll see some text scroll by. Wait until it finishes (takes a few seconds).

### Step 3 — Open Claude Code

In the same terminal window, type:

```bash
claude
```

Hit Enter. Claude Code will open.

### Step 4 — Run the skill

Type:

```
/meta-rate-limiter
```

Hit Enter. On your first run, the skill asks 3 quick questions (your tier, active ad count, preferred language) and then shows your exact rate limits.

---

## Usage

**Generate rate-limited code:**
```
/meta-rate-limiter
> Generate a rate-limited client for managing campaigns
```

**Diagnose an error:**
```
/meta-rate-limiter
> I'm getting error code 613, subcode 1487632
```

**Calculate your limits:**
```
/meta-rate-limiter
> What are my rate limits with 200 active ads on Standard tier?
```

**Review existing code:**
```
/meta-rate-limiter
> Review my meta-api.js file for rate limit issues
```

---

## What's Inside

The skill contains verified rate limit data (as of April 2026) covering:

- **Access tier scoring** — Development (60 pts) vs Standard (9,000 pts)
- **BUC quota formulas** — Ads Management, Ads Insights, Custom Audiences
- **Hard limits** — 100 QPS mutations, 4 budget changes/hr, 10 spend limit changes/day
- **All error codes** — Platform-level (4, 17, 32, 613) and BUC-specific (80000-80014)
- **All subcodes** — Budget limits, QPS, ad creation, volume spikes
- **Header parsing** — All four Meta usage headers with exact field definitions

---

## Updating

To get the latest version:

```bash
cd ~/.claude/skills/meta-rate-limiter && git pull
```

---

## Built By

[@tenfoldmarc](https://www.instagram.com/tenfoldmarc) — Follow for daily AI automation walkthroughs. Real systems, not theory.
