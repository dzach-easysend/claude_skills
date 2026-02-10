---
name: hubspot-cohort-analysis
description: >
  Analyze HubSpot CRM data to generate monthly cohort funnel views showing progression
  from Contacts → Leads → Deal stages (S1-S7) → Closed Won/Lost, delivered as an Excel
  workbook. Sliceable by traffic source (Paid Social, Offline, Organic Search, etc.) and
  sub-sliceable by specific campaigns or drill-downs (e.g., a specific Facebook campaign,
  a specific conference name).

  Use this skill whenever the user asks about: cohort analysis from HubSpot, funnel
  conversion rates, lead-to-deal progression, source attribution analysis, campaign
  ROI tracking, monthly pipeline cohorts, or questions like "show me how contacts
  from [source] convert through the pipeline." Also trigger when the user mentions
  "cohort view", "funnel by source", "conversion by month", "pipeline analysis",
  "how are our leads converting", "source breakdown", "campaign performance", or
  any request to slice HubSpot contact/deal data by acquisition source and time period.
---

# HubSpot Cohort Funnel Analysis

Build a monthly cohort analysis tracking how contacts created in each month progress
through the full sales funnel, broken down by traffic source and campaign.

## Quick Start

1. **Clarify scope** — ask the user for time range and source focus (default: last 12 months, all sources)
2. **Read the reference file** at `references/hubspot_model.md` for property mappings and stage definitions
3. **Execute the counting logic** described below using HubSpot MCP tools
4. **Generate Excel** using `scripts/build_excel.py`
5. **Share the result** — save the .xlsx and provide a link

## The Funnel

This company's actual sales process uses only two lifecycle stages, then deals:

```
Contact → Lead → Deal Created → S1 → S2 → S3 → S4 → S5 → S6 → S7 → Closed Won
                                                                    ↘ Closed Lost
```

**The lifecycle stages MQL, SQL, Subscriber, and Opportunity are NOT used in this
company's workflow.** Some contacts may have these values due to legacy data or
HubSpot automation, but they are NOT meaningful funnel stages. Do not include them
as separate columns. When counting "Leads", include contacts at ANY lifecycle stage
beyond subscriber (lead, mql, sql, opportunity, customer) — they all represent
contacts that have been qualified as leads.

### Non-Linear Progression

The funnel is NOT strictly sequential. Contacts and deals can skip stages entirely:
- A contact can get a deal at S3 or S4 without ever passing through S1 or S2
- A contact can have a deal without being marked as a "Lead" in lifecycle stage
- The numbers in each column are independent counts, NOT cumulative subsets

This means Deals can sometimes be > Leads (if some deals were created for contacts
still at subscriber stage), and deal stage columns don't need to sum to anything
specific. Each column is a snapshot of "how many records are at THIS stage right now."

### Non-Sales Pipeline Exclusion

This cohort is a **Sales Pipeline funnel** — it should only include contacts relevant
to the sales process. Contacts whose ONLY associated deals are in non-Sales pipelines
(Renewal, CS, or Partnerships) must be **excluded from ALL funnel stages** — Contacts,
Leads, AND Deals — because they represent existing customer activity, not new sales.

The exclusion operates at TWO levels:

**Contact-level**: If a contact has `num_associated_deals >= 1` and ALL their deals
are in non-Sales pipelines → exclude from Contacts, Leads, and Deals.

**Company-level**: If a contact's associated COMPANY has deals in non-Sales
pipelines → exclude the contact even if they personally have no deals. This catches
bulk-imported contacts from existing customer companies (e.g., contacts imported from
an Israeli enterprise event where the company is already a CS/Renewal customer).

The combined rule:
- Contact has deals in non-Sales pipelines → **exclude**
- Contact has NO deals, but their COMPANY has deals in non-Sales → **exclude**
- Contact has only Sales Pipeline deal → **include**
- Contact's company has only Sales Pipeline deal → **include**
- Contact has no deals AND no company association → **include** (new prospect)

Implementation approach:
1. After fetching the initial contact count for a cohort, query contacts with
   `num_associated_deals >= 1` for that month+source — check their deal pipelines
2. Also identify companies with non-Sales pipeline deals (search CS and Renewal
   pipeline deals, cross-reference company domains)
3. Count contacts from excluded companies using email domain filters
   (e.g., `email CONTAINS_TOKEN *@harel-ins.co.il`)
4. Subtract both contact-level and company-level exclusions from Contacts/Leads counts
5. Remove excluded contacts from Record Links listings

### Funnel columns in the output

| Column | What it counts |
|--------|---------------|
| **Contacts** | Contacts created in the cohort month, excluding those tied to non-Sales pipelines |
| **Leads** | Contacts whose lifecycle stage progressed beyond initial entry, excluding those tied to non-Sales pipelines |
| **Deals** | Contacts with at least one associated deal only in the Sales Pipeline |
| **S1 - Discovery** | Deals that reached stage `appointmentscheduled` |
| **S2 - Deep Dive** | Deals that reached stage `qualifiedtobuy` |
| **S3 - Trial** | Deals that reached stage `3746014441` |
| **S4 - Consensus** | Deals that reached stage `presentationscheduled` |
| **S5 - Proposal** | Deals that reached stage `decisionmakerboughtin` |
| **S6 - Negotiation** | Deals that reached stage `2966701276` |
| **S7 - Contract** | Deals that reached stage `3087247588` |
| **Won** | Deals at `closedwon` |
| **Lost** | Deals at `closedlost` |

## Source Hierarchy

HubSpot tracks sources at three levels of granularity:

| Level | Property | Example |
|-------|----------|---------|
| **Source** | `hs_analytics_source` | PAID_SOCIAL |
| **Drill-down 1** | `hs_analytics_source_data_1` | Facebook |
| **Drill-down 2** | `hs_analytics_source_data_2` | july-testing-demo-thankyou |

The user can slice at any level. "Show me Paid Social" → filter by source. "Break
down Facebook campaigns" → filter by drill-down 1, group by drill-down 2.

## Token-Aware Data Strategy

The HubSpot account has **~127K contacts**. Pulling all contacts with full properties
would blow through token limits and likely fail. Instead, use a count-first approach.

### The Core Insight

The `search_crm_objects` tool returns a `total` count for any filtered query. By
running targeted searches with `limit=1` and reading only the `total` field, each
data point costs one API call but almost zero tokens.

### Counting Contacts and Leads

**Contacts** (column 1): filter by createdate range + source only → read `total`.

**Leads** (column 2): contacts at any lifecycle stage beyond initial entry. Since
the lifecycle property only shows CURRENT stage, you need to sum across all qualifying
stage values. Run individual queries per stage and sum the totals:

```
# Count Leads for a given month + source:
lead_total = 0
for stage in ["lead", "marketingqualifiedlead", "salesqualifiedlead", "opportunity", "customer"]:
    result = search_crm_objects(
        objectType="CONTACT",
        filterGroups=[{"filters": [
            {"propertyName": "createdate", "operator": "GTE", "value": "<start-ms>"},
            {"propertyName": "createdate", "operator": "LT", "value": "<end-ms>"},
            {"propertyName": "hs_analytics_source", "operator": "EQ", "value": "<SOURCE>"},
            {"propertyName": "lifecyclestage", "operator": "EQ", "value": stage}
        ]}],
        limit=1,
        properties=["hs_object_id"]
    )
    lead_total += result.total
```

Even though MQL/SQL/Opportunity aren't actively used, a handful of contacts sit at
those stages. Including them in the sum ensures an accurate "Leads" count.

For the ALL-sources total (no source filter), you can use 5 filterGroups in one query
since each group has only 3 filters (date GTE, date LT, stage) = 15 total filters,
which is within HubSpot's 18-filter limit:

```
search_crm_objects(
    objectType="CONTACT",
    filterGroups=[
        {"filters": [date_gte, date_lt, {"propertyName": "lifecyclestage", "operator": "EQ", "value": "lead"}]},
        {"filters": [date_gte, date_lt, {"propertyName": "lifecyclestage", "operator": "EQ", "value": "marketingqualifiedlead"}]},
        {"filters": [date_gte, date_lt, {"propertyName": "lifecyclestage", "operator": "EQ", "value": "salesqualifiedlead"}]},
        {"filters": [date_gte, date_lt, {"propertyName": "lifecyclestage", "operator": "EQ", "value": "opportunity"}]},
        {"filters": [date_gte, date_lt, {"propertyName": "lifecyclestage", "operator": "EQ", "value": "customer"}]}
    ],
    limit=1,
    properties=["hs_object_id"]
)
```

This returns the total lead count in a single call. Use this for the ALL row, and
fall back to individual queries when a source filter is needed (since adding a 4th
filter per group would push to 20 filters, exceeding the 18 limit).

### Counting Deals and Deal Stages

Deals require record-level data because they're a separate object type associated
with contacts. The approach:

1. For each cohort month + source combo where Leads > 0, search for deals associated
   with contacts from that cohort
2. Filter deals by `pipeline=default` (Sales Pipeline only)
3. Count deals at each stage

**Efficient deal counting strategy:**

Option A (small cohorts, <500 leads): Fetch contact IDs from the cohort, then search
deals by association filter in batches.

Option B (large cohorts): Search deals by `createdate` within the same month range,
fetch with `pipeline` and `dealstage` properties, then cross-reference with contact
source data. This avoids paginating through thousands of contacts.

For both options, use `limit=1` first to check the `total` — if zero deals, skip
that cohort/source combo entirely.

### Deal Stage Counting — Current State, Not Cumulative

Each deal sits at exactly ONE stage at any given time. The S1-S7 columns show how
many deals are **currently at** that stage — not how many "passed through" it.

For example, if there are 3 deals:
- Deal A is at S4
- Deal B is at S5
- Deal C is Closed Lost

Then the columns should read: S1:0, S2:0, S3:0, S4:1, S5:1, S6:0, S7:0, Won:0, Lost:1

This gives a clear snapshot of where the pipeline stands right now. The "Deals" column
is the total count of deals (sum of all stage columns including Won and Lost).

Deal stage values for filtering:
```
appointmentscheduled = S1    qualifiedtobuy = S2        3746014441 = S3
presentationscheduled = S4   decisionmakerboughtin = S5  2966701276 = S6
3087247588 = S7              closedwon = Won             closedlost = Lost
```

### Parallelization

Run independent counting queries in parallel (4-6 per batch). For example, count
contacts and leads for Jan/Feb/Mar/Apr all at once. This dramatically reduces
wall-clock time.

IMPORTANT: Date values in HubSpot search filters must be **Unix timestamps in
milliseconds** (e.g., 1735689600000 for Jan 1 2025).

## Step-by-Step Execution

**Step 1: Determine date range**
- Ask user or default to last 12 complete months
- Each month = one cohort row

**Step 2: Discover active sources**
- For the date range, run one query per source value to find which have contacts
- Only include sources with >0 contacts in the output
- Source values: ORGANIC_SEARCH, PAID_SEARCH, EMAIL_MARKETING, SOCIAL_MEDIA,
  REFERRALS, OTHER_CAMPAIGNS, DIRECT_TRAFFIC, OFFLINE, PAID_SOCIAL, AI_REFERRALS

**Step 3: Count Contacts and Leads per month × source**
- Run queries in parallel batches (4-6 at a time)
- For each combo: 1 query for contacts, 1 query (or 5 individual) for leads

**Step 4: Count Deals and deal stages, collect record details**
- Search for deals associated with contacts in each cohort
- Filter to Sales Pipeline (pipeline=default)
- Count how many deals are currently at each stage (S1 through S7, Won, Lost)
- The "Deals" column = sum of all stage counts (S1+S2+...+S7+Won+Lost)
- **Collect record details** for the links sheet: for each deal, save its ID, name,
  stage, amount, and associated contact name/ID

**Step 5: Generate Excel**
Use `scripts/build_excel.py` — pass it a JSON file with the cohort data structure.
Also follow the xlsx skill formatting guidelines.

Sheet structure:
- **"All Sources"**: Full cohort table with counts + inline conversion rates below
- **One sheet per active source**: Same structure, filtered
- **"Conversion Rates"**: Dedicated stage-to-stage conversion percentages
- **"Record Links"**: Every count cell should be backed by a detail sheet listing
  individual records with clickable HubSpot URLs. See "Record Links" section below.
- **Sub-slice sheets** (if user requests): Break down by drill-down 1 or 2

**Step 6: Save and deliver**
Save to the workspace folder with a descriptive filename like
`hubspot_cohort_<start>_to_<end>.xlsx`

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
