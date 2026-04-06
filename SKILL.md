---
name: meta-rate-limiter
description: "Manage Meta (Facebook/Instagram) ads API calls without ever hitting rate limits. Calculates exact quotas, generates rate-limited code, and diagnoses throttling errors."
---

# Meta Rate Limiter

> Manage Meta (Facebook/Instagram) ads API calls without ever hitting rate limits. Calculates your exact quotas, generates rate-limited code, and monitors usage in real time.

## Step 0 — First-Time Setup

Check for config at `~/.claude/skills/meta-rate-limiter/config.json`.

**If config exists:** Load the user's settings. Skip to the trigger logic.

**If config does NOT exist:**

1. Greet:
   ```
   Welcome to /meta-rate-limiter! Let's get you set up. This takes about 1 minute and only happens once.
   ```

2. Ask:
   - **What's your access tier?** Development (default for new apps) or Standard (requires Advanced Access approval)?
     - If they don't know: "Check your Meta App Dashboard → App Review → Permissions. If you see `ads_management` with Advanced Access approved, you're on Standard. Otherwise, Development."
   - **How many active ads** do you typically run? (affects BUC quota calculations)
   - **What language do you prefer?** JavaScript/TypeScript (default), Python, or both?

3. Save config to `~/.claude/skills/meta-rate-limiter/config.json`:
   ```json
   {
     "tier": "development",
     "activeAds": 50,
     "language": "javascript",
     "setupComplete": true,
     "setupDate": "2026-04-06"
   }
   ```

4. Calculate and display their quotas immediately (see Quota Calculator section), then confirm:
   ```
   Setup complete! Here are your current rate limits based on your inputs.
   Run /meta-rate-limiter anytime to generate code, diagnose errors, or recalculate quotas.
   ```

## Trigger

**This skill activates automatically whenever the Meta/Facebook API is involved — not just when rate limits are mentioned.**

When the user:
- Types `/meta-rate-limiter`
- Asks about Meta API rate limits, quotas, or throttling
- Wants to build ANYTHING that calls the Meta/Facebook Marketing API
- Asks to pull ad performance, campaign data, or insights from Meta
- Wants to create, update, or manage campaigns, ad sets, or ads via the API
- Asks to manage custom audiences via the API
- Is writing code that imports the Facebook/Meta SDK or calls `graph.facebook.com`
- Is debugging Meta API error codes 4, 17, 32, 613, or 80000+
- Mentions Facebook Ads API, Instagram Graph API, or Marketing API in a coding context
- Wants to batch, throttle, or optimize Meta API calls
- Asks to automate anything involving Meta ad accounts

**Rule:** If code is being written that will make requests to Meta's API, this skill's rate limiting patterns MUST be applied — even if the user doesn't ask for it. Never generate Meta API code without the header monitor, backoff, and pacing from this skill.

## What This Skill Does

1. **Calculates exact rate limits** based on the user's access tier (Development or Standard) and active ad/audience counts
2. **Generates production-ready code** with built-in rate limiting, header monitoring, and exponential backoff
3. **Diagnoses throttling errors** and recommends fixes
4. **Optimizes API call patterns** — batching, async insights, field filtering, request pacing

## Rate Limit Reference (Verified April 2026)

### Access Tiers

| | Development | Standard |
|---|---|---|
| **Max score** | 60 points | 9,000 points |
| **Score decay** | 300 seconds | 300 seconds |
| **Block duration** | 300s (5 min) | 60s (1 min) |
| **Scoring** | Read = 1 pt, Write = 3 pts | Same |

**To upgrade:** Apply for Advanced Access to "Ads Management Standard Access" via Meta App Review. Requires business verification.

### Business Use Case (BUC) Quotas — Per Ad Account, Per Hour

| Category | Standard | Development |
|---|---|---|
| **Ads Management** | 100,000 + (40 × active ads) | 300 + (40 × active ads) |
| **Ads Insights** | 190,000 + (400 × active ads) − (0.001 × user errors) | 600 + (400 × active ads) |
| **Custom Audience** | 190,000 + (40 × active audiences) [max 700k] | 5,000 + (40 × active audiences) [max 700k] |

### Hard Limits (Cannot Be Increased)

| Constraint | Limit |
|---|---|
| Mutations (POST) per app+account | **100 requests/second** |
| Ad set budget changes | **4 per hour** |
| Account spend limit changes | **10 per day** |

### API Call Counting Rules

- **Batch requests:** Each sub-request counts separately
- **Multi-ID requests:** `?ids=4,5,6` = **3 calls**, not 1
- **Errored calls** still count against quota
- ALL calls count — there are no "free" requests

## HTTP Headers to Monitor

Every response from Meta includes usage headers. The skill MUST generate code that reads these on EVERY response:

### X-App-Usage
```json
{"call_count": 28, "total_cputime": 25, "total_time": 25}
```
Percentages (0-100). **Throttle at 100%.** Any field hitting 100 = blocked.

### X-Business-Use-Case-Usage
```json
{
  "business-id": [{
    "type": "ads_management",
    "call_count": 28,
    "total_cputime": 25,
    "total_time": 25,
    "estimated_time_to_regain_access": 0,
    "ads_api_access_tier": "standard_access"
  }]
}
```
Up to 32 objects per business. Check `estimated_time_to_regain_access` — if > 0, **stop immediately and wait that many minutes.**

### X-Ad-Account-Usage
```json
{"acc_id_util_pct": 9.67, "reset_time_duration": 100, "ads_api_access_tier": "standard_access"}
```

### X-FB-Ads-Insights-Throttle
```json
{"app_id_util_pct": 10.5, "acc_id_util_pct": 9.67, "ads_api_access_tier": "standard_access"}
```

## Error Codes

When the user encounters an error, diagnose using this table:

### Platform-Level
| Code | Meaning | Action |
|---|---|---|
| **4** | App-level limit | Exponential backoff, start 1s |
| **17** | User-level limit | Backoff + check if hitting per-user caps |
| **32** | Pages API limit | Backoff + reduce page token calls |
| **613** | Specific limit exceeded | Check subcode below |

### Subcode Details (Code 613)
| Subcode | Issue | Fix |
|---|---|---|
| 1996 | Inconsistent request volume spike | Spread requests evenly, don't burst |
| 1487742 | Too many calls from ad account | Rotate accounts or slow down |
| 5044001 | QPS limit exceeded (>100/sec) | Add minimum 10ms delay between mutations |
| 1487632 | Budget changed >4 times/hour | Queue budget changes, max 4/hr |
| 1487225 | Ad creation limit exceeded | Batch ad creation, pace over time |

### BUC-Specific (80000 range)
| Code | Category |
|---|---|
| 80000 | Ads Insights |
| 80002/80003 | Custom Audience |
| 80004 | Ads Management |
| 80005 | Instagram |
| 80006 | LeadGen |
| 80008 | WhatsApp Business |

## Code Generation Rules

When generating rate-limited Meta API code, ALWAYS include:

### 1. Rate Limiter Class/Module

```
Core requirements:
- Token bucket or sliding window algorithm
- Configurable points per second (default: ~1 req/sec for safety)
- Separate tracking for reads (1pt) vs writes (3pts)
- Score decay over 300 seconds
- Hard stop when approaching 80% of max score (48/60 for dev, 7200/9000 for standard)
```

### 2. Header Monitor

```
On EVERY response:
- Parse X-App-Usage → pause if any field >= 90%
- Parse X-Business-Use-Case-Usage → check estimated_time_to_regain_access
- Parse X-Ad-Account-Usage → track acc_id_util_pct
- Parse X-FB-Ads-Insights-Throttle → track for insight queries
- Log all values for debugging
```

### 3. Exponential Backoff

```
On error codes 4, 17, 32, 613, 80000-80014:
- Start: 1 second
- Multiply: 2x each retry
- Max: 300 seconds (5 minutes)
- Max retries: 8
- Add jitter: ±20% randomization to prevent thundering herd
- On 80000+ errors: also check estimated_time_to_regain_access from headers
```

### 4. Budget Change Queue

```
- Track budget changes per ad set per hour
- If 4 changes already made this hour → queue for next hour
- Track spend limit changes per day (max 10)
- Return clear error to caller when queued, not a silent failure
```

### 5. Request Batcher

```
- Consolidate multiple operations into batch requests where possible
- Remember: each sub-request in a batch still counts as a separate call
- Benefit is reducing HTTP overhead and round trips, not call count
- Max 50 sub-requests per batch call (Meta limit)
```

### 6. Async Insights

```
- For insights queries, ALWAYS use async=true parameter
- Poll for completion with exponential backoff
- This avoids timeouts on large date ranges and reduces rate pressure
- Max 10 async requests per ad account per day for historical reach data (13+ months)
```

## Quota Calculator

When the user asks "what are my limits?" or provides their active ad count, calculate and display:

```
INPUTS:
- Tier: [Development | Standard]
- Active ads: [number]
- Active custom audiences: [number, optional]

OUTPUT:
Ads Management:    [formula result] calls/hour
Ads Insights:      [formula result] calls/hour
Custom Audience:   [formula result] calls/hour
Mutations (POST):  100 requests/second
Budget changes:    4 per hour per ad set
Spend limit:       10 changes per day

Safe pacing:
- Reads:  ~[quota/3600] per second
- Writes: ~[quota/3600/3] per second (3x point cost)
```

## Development vs Standard Comparison

When users are on Development tier, ALWAYS mention:

> **You're on the Development tier (60 points max).** This means roughly 60 reads or 20 writes per 5-minute window before you're blocked for 5 minutes. For any production use, you need Standard access.
>
> **To upgrade:** Meta App Dashboard → App Review → Request Advanced Access for "Ads Management Standard Access". Requires business verification (takes 1-5 business days).

## Optimization Checklist

When reviewing existing Meta API code, check for:

- [ ] **Field filtering** — Only request fields you need (`?fields=id,name,status`), never fetch all fields
- [ ] **Date range limiting** — Don't query all-time data when you need last 7 days
- [ ] **Async insights** — Using `async=true` for any insights query spanning >7 days
- [ ] **Request pacing** — Spreading calls evenly, not bursting
- [ ] **Header monitoring** — Reading ALL four usage headers on every response
- [ ] **Backoff implementation** — Exponential with jitter on all error codes
- [ ] **Budget queue** — Not exceeding 4 budget changes/hr per ad set
- [ ] **Multi-ID awareness** — Knowing `?ids=1,2,3` counts as 3 calls
- [ ] **Error counting** — User errors reduce Insights quota via the formula

## Output Format

When generating code, structure it as:

```
meta-rate-limiter/
├── rate-limiter.[js|py]        # Core rate limiting logic
├── header-monitor.[js|py]      # Parse and track Meta usage headers
├── backoff.[js|py]             # Exponential backoff with jitter
├── budget-queue.[js|py]        # Budget change queuing (4/hr limit)
├── batch-client.[js|py]        # Batch request builder
├── meta-client.[js|py]         # Main client wrapping everything together
├── quota-calculator.[js|py]    # Calculate limits from active ads count
└── examples/
    ├── fetch-campaigns.[js|py]   # Read example with pacing
    ├── update-budgets.[js|py]    # Write example with budget queue
    └── pull-insights.[js|py]     # Async insights example
```

Generate a single-file version if the user prefers simplicity, or the modular version for production use.

## Common Scenarios

### "I keep getting error 4"
→ App-level throttling. Check `X-App-Usage` header values. Likely bursting too many requests. Implement pacing at ~1 req/sec.

### "Error 613, subcode 1487632"
→ Budget changed more than 4 times in an hour for an ad set. Implement the budget queue.

### "Error 80004"
→ BUC Ads Management limit. Calculate their quota: `100,000 + (40 × active_ads)` at Standard. If on Development, it's only `300 + (40 × active_ads)` — upgrade to Standard.

### "Everything was working, then suddenly throttled"
→ Meta can throttle during high backend load regardless of per-account limits (error 4, subcode 1504022). This is global — just backoff and retry.

### "My batch requests aren't helping"
→ Each sub-request in a batch counts as a separate API call. Batching reduces HTTP overhead but NOT your quota usage. Still useful for reducing latency.

## Do NOT

- Generate code that ignores rate limit headers
- Suggest retry loops without exponential backoff
- Tell users batch requests reduce their API call count (they don't)
- Recommend fixed sleep timers instead of header-driven pacing
- Ignore the Development tier limitations — 60 points is brutal
- Forget that errored calls still count against quota
- Use `?ids=` with large ID lists without accounting for per-ID call cost

---

Built by [@tenfoldmarc](https://instagram.com/tenfoldmarc). Follow for daily AI automation builds — real systems, not theory.
