---
name: hubspot-cohort-analysis
description: >
  Analyze HubSpot CRM data to generate monthly cohort funnel views showing progression
  from Contacts → Leads → Deal stages (S1-S7) → Closed Won/Lost.

  Triggers: cohort analysis, funnel conversion, lead-to-deal progression, source
  attribution, "cohort view", "funnel by source", "conversion by month", "pipeline
  analysis", campaign performance, or slicing contact/deal data by source and time.
---

# HubSpot Cohort Funnel Analysis

## Efficiency Rules (CRITICAL — read before doing anything)

1. **ALWAYS filter by date first.** Never scan the full contact database. Every query
   must include a `createdate` range filter scoping to the cohort month(s).
2. **Count, don't fetch.** Use `search_crm_objects` with `limit=1` and read the
   `total` field for counts. Do not paginate results just to count them.
3. **Fetch IDs only when needed.** When you need record-level data (e.g., for deals),
   request only `hs_object_id` — not full contact objects.
4. **Lazy details.** Do NOT fetch record details (names, emails, amounts) or build
   "Record Links" unless the user explicitly asks for an Excel export or record list.
5. **Short-circuit on zero.** If a count query returns `total: 0`, skip all downstream
   steps for that cohort/source combo.

### Caveat: `total` field reliability

HubSpot's `total` may return approximate counts for very large result sets (10,000+).
For typical monthly cohorts (hundreds to low thousands), it is exact. If a count looks
suspect or the user needs auditable precision, paginate with `limit=100` and count
records explicitly. Flag this to the user when doing so.

## Tool Discovery (CRITICAL)

The HubSpot MCP server exposes tools that wrap the CRM Search API. **Tool names vary
by implementation.** Before executing queries:

1. List available tools to identify the exact names (e.g., `search_crm_objects`,
   `hubspot_search_contacts`, or similar).
2. The key tool is the **CRM search** tool. It accepts:
   - `objectType`: `"contacts"` or `"deals"`
   - `filterGroups`: array of filter groups (OR logic between groups, AND within)
   - `properties`: array of property names to return
   - `limit`: max results per page (max 100)
3. For **associations**, look for a tool like `list_associations` or
   `get_associations`. You need this to link contacts → deals.

All parameters follow HubSpot's CRM Search API format. See
`references/hubspot_model.md` for property names, stage values, and API constraints.

## The Funnel

```
Contact → Lead → Deal → S1..S7 → Won / Lost
```

| Column | What it counts |
|--------|---------------|
| **Contacts** | Contacts created in the cohort month (Sales Pipeline only — see exclusion below) |
| **Leads** | Subset of Contacts with lifecyclestage in the include-list below |
| **Deals** | Contacts with ≥1 deal in the Sales Pipeline (`pipeline = "default"`) |
| **S1–S7, Won, Lost** | Deals currently at each stage (snapshot, not cumulative) |

### Leads: explicit include-list

Count contacts whose `lifecyclestage` matches ANY of these values:
`lead`, `marketingqualifiedlead`, `salesqualifiedlead`, `opportunity`, `customer`.

Contacts at `subscriber`, `3016596683` (Churned), `3015834863` (Disqualified),
`3015834864` (Do Not Contact), or `other` are NOT leads.

This requires **5 separate count queries** (one per stage) when a source filter is
active, or **1 query with 5 filterGroups** when counting all-sources (see API
constraints in `references/hubspot_model.md`). Budget accordingly.

### Sales Pipeline Filtering (COMPANY-LEVEL)

**CRITICAL:** Sales Pipeline analysis must filter at the COMPANY level, not just deal level.

**Sales Pipeline contacts = contacts whose associated companies are:**
1. **ONLY in Sales Pipeline** (all company deals have `pipeline = "default"`), OR
2. **In NO pipeline** (company has zero deals = new prospect = sales pipeline)

**EXCLUDE contacts if their company has:**
- Deals in MULTIPLE pipelines (e.g., Sales + Renewal, Sales + CS)
- Deals ONLY in non-Sales pipelines (Renewal `2193617094`, CS `2193544436`, Partnerships `2412810436`)

**Why:** Contacts from companies with existing renewal/CS activity represent customer expansion,
not new sales pipeline activity. Mixing these skews conversion metrics.

## Execution Steps

### Step 0: Compute timestamps dynamically

Convert the requested month range to Unix milliseconds. The account timezone is
**Asia/Jerusalem (UTC+2)** — compute start-of-month boundaries in this timezone.

Example (for verification only, do NOT hardcode):
- Jan 1 2026 00:00 Asia/Jerusalem = `1767222000000` (UTC+2)
- Feb 1 2026 00:00 Asia/Jerusalem = `1769900400000`

### Step 1: Count Contacts

For each month (+ source if specified):

```
search_crm_objects(
  objectType="contacts",
  filterGroups=[{"filters": [
    {"propertyName": "createdate", "operator": "GTE", "value": "<start_ms>"},
    {"propertyName": "createdate", "operator": "LT",  "value": "<end_ms>"},
    {"propertyName": "hs_analytics_source", "operator": "EQ", "value": "<SOURCE>"}  // omit if all-sources
  ]}],
  limit=1,
  properties=["hs_object_id"]
)
→ read `total`
```

If `total == 0`, skip this cohort/source entirely.

### Step 2: Count Leads

Run 5 queries (one per lifecycle stage) with the same date+source filters, adding
`lifecyclestage EQ <stage>`. Sum the totals.

For all-sources (no source filter), combine into 1 query with 5 filterGroups
(15 filters total, within the 18-filter limit).

### Step 3: Filter Contacts by Company Pipeline Membership

**CRITICAL:** Before counting deals, filter contacts to Sales Pipeline only.

**Procedure:**

1. **Fetch all cohort Contact IDs** with company associations:
   - Search contacts with date+source filters
   - `properties=["hs_object_id"]`, paginate with `limit=100`
   - Get associated company ID for each contact

2. **For each unique company, get ALL its deals across ALL pipelines:**
   - Fetch ALL deals for the company (no pipeline filter yet)
   - Get properties: `pipeline`, `dealstage`

3. **Categorize each company by pipeline membership:**
   - **Sales Only**: All deals have `pipeline == "default"` → INCLUDE
   - **No Pipeline**: Company has 0 deals → INCLUDE (new prospect)
   - **Mixed Pipeline**: Has deals in "default" AND other pipelines → EXCLUDE
   - **Other Pipeline Only**: All deals in non-sales pipelines → EXCLUDE

4. **Filter contacts:** Keep only those associated with "Sales Only" OR "No Pipeline" companies.

5. **Count deals for filtered contacts:**
   - Use only the filtered contact list
   - Fetch their deal associations
   - Count by `dealstage` (S1–S7, Won, Lost)

**Cost estimate for 300-contact cohort:**
- ~3 paginated calls to fetch contact IDs
- ~200–300 contact-company association calls
- ~150 unique companies × 1 deal query = ~150 calls
- ~10–20 deal property fetches for filtered contacts
- Total: ~365–470 API calls (higher due to company-level filtering)

### Step 4: Output Format (STANDARD)

**ALWAYS generate TWO files:**

**File 1: Excel Funnel Analysis** (`{cohort}_funnel_analysis.xlsx`)
- **Sheet 1 "Cohort Funnel"**: Table showing stage-by-stage progression with conversion rates
- **Sheet 2 "Pipeline Breakdown"**: Show excluded contacts by reason (Mixed Pipeline, Renewal Only, etc.)
- **Sheet 3 "Deal Details"**: List individual deals with company, contact, amount, stage, outcome
- Use professional formatting: headers, borders, currency formatting, conditional formatting

**File 2: Word Insights Document** (`{cohort}_insights.docx`)
- **Executive Summary**: Key findings and headline metrics
- **Sales Pipeline Definition**: Explain the company-level filtering logic
- **Root Cause Analysis**: Why contacts didn't convert (from Gong/qualitative data if available)
- **Strategic Recommendations**: Prioritized action items with expected impact
- Use professional formatting: styled headings, tables, bullet points

**Do NOT output markdown tables.** The Excel + Word format is required for all cohort analyses.

## Error Handling

- **Rate limits (429 responses):** Pause for the duration specified in the
  `Retry-After` header. If no header, wait 10 seconds and retry. Max 3 retries
  per call before aborting and reporting the failure to the user.
- **Empty cohort (`total: 0`):** Short-circuit. Do not proceed to Lead/Deal steps.
  Show 0 in all columns for that row.
- **Partial failures:** If association calls fail mid-batch, report which
  contact ID ranges were successfully processed and which failed. Present
  partial results clearly marked as incomplete.
- **Future months / no data:** Return zeros, don't error.

## Company-Level Pipeline Filtering (MANDATORY)

**Contact-level exclusion at the company level is REQUIRED for all analyses.**

Contacts are excluded from the cohort if their associated company has:
- Deals in multiple pipelines (mixed sales + renewal/CS/partnerships)
- Deals ONLY in non-Sales pipelines

This filtering is MORE expensive (~365–470 API calls per 300 contacts) but produces accurate
sales-only cohort metrics. Do NOT skip this step.

**Expected exclusion rate:** 30–50% of raw contacts will be filtered out as non-sales pipeline.

## Excel & Word Generation

Use the **xlsx skill** and **docx skill** to generate the output files:

**For Excel:**
- Invoke the xlsx skill with proper sheet structure
- Include formulas for conversion rates (not hardcoded values)
- Use professional formatting: borders, conditional formatting, currency formats

**For Word:**
- Invoke the docx skill with proper heading hierarchy
- Use tables for structured data (objections, recommendations)
- Use bullet points for lists (NOT manual Unicode bullets)
- Professional styling: Arial font, blue headers (#2E5090), proper spacing

## Reference

All property names, stage IDs, pipeline IDs, source values, and API constraints
are documented in `references/hubspot_model.md`. Read it before executing.
