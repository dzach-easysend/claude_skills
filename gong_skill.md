---
name: gong-sales-intelligence
description: Analyze Gong sales call transcripts to extract pain points, objections, competitive mentions, and product gaps. Supports cross-MCP joins with HubSpot and FullStory for deal-aware and behavior-informed analysis. Use when the user asks about sales calls, lead pain points, objections, win/loss patterns, product feedback from calls, or cross-referencing Gong data with CRM or product analytics.
---

# Gong Sales Intelligence

Analyze Gong call transcripts to extract actionable sales and product insights. Works standalone or joined with HubSpot (CRM) and FullStory (product analytics) for deal-aware, behavior-informed analysis.

## Available Gong MCP Tools

| Tool | Purpose | Key params |
|------|---------|------------|
| `list_calls` | List calls in a date range | `from_date`, `to_date`, `limit` |
| `search_calls` | Find calls by text query, emails, or domains | `query`, `emails`, `domains`, `from_date`, `to_date` |
| `get_transcript` | Full transcript for one call | `call_id`, `format` (text/json) |
| `get_call_participants` | Lightweight participant lookup | `call_ids` |
| `analyze_calls` | Smart-routed analysis (direct or async) | `from_date`, `to_date`, `prompt`, `call_ids`, `emails`, `domains` |
| `get_job_status` | Poll async job progress | `job_id` |
| `get_job_results` | Retrieve completed async results | `job_id` |

### Smart Routing

`analyze_calls` automatically routes based on dataset size:
- **< 150K tokens** → returns transcripts directly for inline analysis
- **> 150K tokens** → starts async background job, returns `job_id` to poll

When async: poll `get_job_status` every 30-60s, then `get_job_results` when complete.

## Core Analysis Workflows

### 1. Extract Lead Pain Points

```
analyze_calls(
  from_date="2026-01-01",
  to_date="2026-02-10",
  prompt="Identify and categorize all customer pain points mentioned in these calls. For each pain point: (1) quote the exact words used, (2) name the speaker company, (3) rate severity as high/medium/low based on emotional intensity and business impact. Group by theme and sort by frequency."
)
```

### 2. Sales Objections (Prioritized by Count)

```
analyze_calls(
  from_date="2026-01-01",
  to_date="2026-02-10",
  prompt="Extract every sales objection raised by prospects. Categorize them (pricing, timing, competition, feature gap, internal buy-in, etc.). For each category: list exact quotes, count of unique calls mentioning it, and suggested response. Return a ranked table sorted by frequency descending."
)
```

### 3. Competitive Intelligence

```
analyze_calls(
  from_date="2026-01-01",
  to_date="2026-02-10",
  prompt="Find every mention of competitors. For each competitor: count mentions, summarize what prospects said (positive and negative), identify feature comparisons, and note any switching signals. Rank competitors by mention frequency."
)
```

### 4. Win/Loss Pattern Analysis

```
analyze_calls(
  prompt="Analyze these calls for signals that predict deal outcome. Identify: (1) phrases/topics correlated with closed-won, (2) phrases/topics correlated with closed-lost, (3) common turning points in conversations. Provide actionable coaching recommendations."
)
```

### 5. Feature Requests & Product Gaps

```
analyze_calls(
  prompt="Extract all feature requests, product complaints, and workflow gaps mentioned by prospects. For each: quote the request, note the company, estimate urgency, and categorize by product area. Rank by frequency across calls."
)
```

## Cross-MCP Workflows

### Gong + HubSpot: Deal-Aware Call Analysis

Use HubSpot to identify relevant contacts/deals, then pass emails to Gong for targeted analysis.

**Workflow:**
1. Query HubSpot for deals/contacts (e.g., open deals, specific pipeline stage, campaign leads)
2. Extract contact emails from HubSpot results
3. Pass emails to Gong `search_calls(emails=[...])` or `analyze_calls(emails=[...])`
4. Analyze the filtered call transcripts

**Example prompt to the agent:**
> "Look at all open deals in HubSpot worth over $50K. Find their Gong calls from the last 90 days and identify the top pain points, grouped by deal stage."

**Step-by-step for the agent:**
1. Use HubSpot MCP to list deals filtered by amount > 50K and stage != closed
2. Collect associated contact emails from each deal
3. Call `analyze_calls(emails=[...collected emails...], from_date="90 days ago", prompt="Identify pain points per prospect. Group by the deal stage context.")`
4. Join the Gong analysis back to HubSpot deal data (deal name, ARR, stage) for the final report

### Gong + FullStory: Behavior-Informed Call Analysis

Combine what prospects *say* in calls with what they *do* in the product.

**Workflow:**
1. Query FullStory for user behavior patterns (rage clicks, drop-off points, feature usage)
2. Cross-reference user emails/domains between FullStory and Gong
3. Analyze calls from those users with behavior context in the prompt

**Example prompt to the agent:**
> "Find users who experienced friction in the onboarding flow (FullStory) and check if they mentioned onboarding issues in sales calls (Gong)."

**Step-by-step for the agent:**
1. Use FullStory MCP to identify users with high frustration signals (rage clicks, errors) in onboarding pages
2. Extract their emails/domains
3. Call `search_calls(emails=[...])` to find matching Gong calls
4. Call `analyze_calls(call_ids=[...matched IDs...], prompt="Focus on onboarding complaints, setup difficulties, and first-experience friction. Cross-reference with these known UX issues: [insert FullStory findings].")`

### Gong + HubSpot + FullStory: Prioritize Product Gaps by ARR

The most powerful pattern: combine CRM deal value, call transcript themes, and product behavior data to produce ARR-weighted product prioritization.

**Example prompt to the agent:**
> "Analyze current HubSpot deals, their Gong sales calls, and FullStory behavior patterns. Prioritize product gaps by aggregate lead ARR."

**Step-by-step for the agent:**

1. **HubSpot** — Get open deals with ARR and contact emails:
   - List deals in target pipeline/stages
   - For each deal, capture: deal name, ARR/amount, stage, associated contact emails

2. **Gong** — Analyze calls for those contacts:
   - `analyze_calls(emails=[...all contact emails...], prompt="Extract all product gaps, feature requests, and complaints. For each, quote the prospect and note their email/company.")`
   - If async: poll `get_job_status`, then `get_job_results`

3. **FullStory** — Get behavior signals for the same users:
   - Query sessions for the same emails/domains
   - Identify friction patterns, drop-offs, feature usage gaps

4. **Synthesize** — Build a prioritized table:

| Product Gap | Gong Mentions | FullStory Signal | Deals Affected | Aggregate ARR | Priority |
|-------------|--------------|------------------|----------------|---------------|----------|
| Missing SSO | 8 calls | N/A (not in product yet) | Acme, BigCo, ... | $420K | P0 |
| Slow reports | 5 calls | Rage clicks on /reports | DataCorp, ... | $310K | P0 |
| No bulk export | 3 calls | Drop-off at /export | SmallCo | $85K | P1 |

**Key:** The ARR weighting comes from HubSpot deal amounts. Gong provides the voice-of-customer. FullStory validates with behavioral evidence.

## Effective Prompts for analyze_calls

The `prompt` parameter drives the analysis. Write it like a brief to a research analyst.

**Do:**
- Be specific about what to extract (quotes, counts, categories)
- Request a structured output format (table, ranked list, grouped)
- Include context from other MCPs when available (e.g., "these users experienced X in FullStory")
- Ask for actionable recommendations, not just data

**Don't:**
- Use vague prompts like "analyze these calls" — be specific
- Forget to ask for frequency/counts when comparing across calls
- Skip requesting quotes — verbatim customer language is the most valuable output

### Prompt templates

**Pain points:**
> "Identify all customer pain points. For each: exact quote, speaker company, severity (high/medium/low), business impact. Group by theme, sort by frequency."

**Objections:**
> "Extract every sales objection. Categorize (pricing/timing/competition/feature/buy-in/other). For each category: count of calls, example quotes, suggested rebuttal. Rank by frequency."

**Product gaps with CRM context:**
> "Extract product gaps and feature requests. For each: quote, company name, email. I will join this with deal ARR data. Output as a structured list with one entry per unique gap."

## Output Template

When presenting cross-MCP analysis results, use this structure:

```
# [Analysis Title]

## Executive Summary
[2-3 sentence overview of key findings with top-line numbers]

## Key Findings

### [Category 1] (N mentions across M calls)
- **Finding**: [description]
- **Evidence**: "[exact quote]" — [Company/Speaker]
- **Impact**: [business impact + ARR if available]

### [Category 2] ...

## Prioritized Recommendations
| # | Recommendation | Supporting Evidence | ARR Impact | Effort |
|---|---------------|-------------------|------------|--------|
| 1 | ... | ... | ... | ... |

## Data Sources
- Gong: N calls analyzed (date range)
- HubSpot: N deals / N contacts referenced
- FullStory: N sessions reviewed
```

## Tips

- **Date ranges**: Always specify `from_date` and `to_date` to scope analysis. Defaults to last 7 days.
- **Domain filtering**: Use `domains=["acme.com"]` to quickly find all calls with a company without needing individual emails.
- **Async jobs**: For large analyses (10+ calls), expect async routing. Poll status every 30-60 seconds.
- **Joining on emails**: Emails are the universal join key across Gong, HubSpot, and FullStory. Always collect emails from the first MCP before querying the next.
- **Incremental analysis**: For very large datasets, break into segments (by month, by deal stage) and synthesize afterward rather than one massive analysis.
