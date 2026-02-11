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

### Step 4: Count Deals and deal stages, collect record details

- Search for deals associated with contacts in each cohort
- Filter to Sales Pipeline (pipeline=default)
- Count how many deals are currently at each stage (S1 through S7, Won, Lost)
- The "Deals" column = sum of all stage counts (S1+S2+...+S7+Won+Lost)
- **Collect record details** for the links sheet: for each deal, save its ID, name,
  stage, amount, and associated contact name/ID

### Step 5: Generate Excel

Use `scripts/build_excel.py` — pass it a JSON file with the cohort data structure.
Also follow the xlsx skill formatting guidelines.

Sheet structure:
- **"All Sources"**: Full cohort table with counts + inline conversion rates below
- **One sheet per active source**: Same structure, filtered
- **"Conversion Rates"**: Dedicated stage-to-stage conversion percentages
- **"Record Links"**: Every count cell should be backed by a detail sheet listing
  individual records with clickable HubSpot URLs. See "Record Links" section below.
- **Sub-slice sheets** (if user requests): Break down by drill-down 1 or 2

### Step 6: Save and deliver

Save to the workspace folder with a descriptive filename like
`hubspot_cohort_<start>_to_<end>.xlsx`

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

## Handling Sub-Slices

When the user asks to "break down by conference" or "show me Facebook campaigns":

1. Identify which source level they mean (drill-down 1 or drill-down 2)
2. First, discover the distinct values by fetching a sample (limit=100) with that
   source filter to see what drill-down values exist
3. Run the same counting logic with the additional drill-down filter
4. Add a new sheet per sub-slice value

## Record Links

Every count in the cohort table should let the user drill into the underlying records.
The Excel workbook includes a **"Record Links"** sheet with individual records grouped
by cohort month, source, and funnel stage.

### What to include

| Funnel stage | Record type | Properties to fetch | HubSpot URL pattern |
|-------------|-------------|--------------------|--------------------|
| Contacts | Contact | firstname, lastname, email, createdate | `https://app-eu1.hubspot.com/contacts/25666518/record/0-1/{id}` |
| Leads | Contact | firstname, lastname, email, lifecyclestage | `https://app-eu1.hubspot.com/contacts/25666518/record/0-1/{id}` |
| Deals, S1-S7, Won, Lost | Deal | dealname, dealstage, amount, closedate | `https://app-eu1.hubspot.com/contacts/25666518/record/0-3/{id}` |

### Token-aware approach for links

For large counts (>50 records), don't fetch every single record. Instead:
- Fetch the first 50 records with their details
- Add a row at the bottom: "... and {remaining} more"
- In the cohort summary cells, add an Excel hyperlink that jumps to the
  corresponding section in the Record Links sheet

For small counts (<50), fetch all records and list them individually.

### Record Links sheet structure

Each section in the sheet is organized as:

```
[Month] | [Source] | [Stage]
Name | Email/Deal Name | Amount | HubSpot Link
record 1...
record 2...
(blank row)
[next section]
```

### Linking summary cells to detail

In the cohort summary sheets, make each count cell a clickable hyperlink that
jumps to the corresponding section in the Record Links sheet. Use Excel internal
links: `=HYPERLINK("#'Record Links'!A{row}", count_value)`

This way the user can click any number in the cohort table and immediately see
which contacts or deals make up that count.

## Edge Cases

- **No data for a month/source**: Show 0, don't skip the row
- **Very old data**: If user asks for >24 months, suggest last 12 and confirm
- **Multiple pipelines**: Only use Sales Pipeline (pipeline=default). Contacts whose
  ONLY deals are in non-Sales pipelines (CS, Renewal & Upsell, Partnerships) must be
  excluded from ALL funnel stages — Contacts, Leads, and Deals — not just from Deals
- **Closed Lost**: Show as its own column, not part of the cumulative deal stage progression

## Reference

All property names, stage IDs, pipeline IDs, source values, and API constraints
are documented in `references/hubspot_model.md`. Read it before executing.
