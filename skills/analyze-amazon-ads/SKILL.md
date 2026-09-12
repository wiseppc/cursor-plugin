---
name: analyze-amazon-ads
description: Analyze Amazon Ads and seller analytics through WisePPC. Use for campaign performance, wasted spend, catalog health checks, or seller reports. Prefer structured query over raw SQL.
---

# Analyze Amazon Ads

## When to use

- Campaign, search-term, or product performance questions
- Seller analytics (sales & traffic, economics, brand analytics) via structured datasets
- Health checks, listing issues, or "where should I focus?" questions
- Budget pacing, wasted spend, or efficiency analysis

## Before You Start

**Always** call `get_session_context` for the `profileId` if you have not already this session. This loads preferences, benchmarks, pending actions, runbooks, and data-model gotchas in one round-trip.

## Query Best Practices

### Prefer Structured `query`

The structured `query` tool is preferred over raw SQL `run_query`:

1. **`list_datasets`** — see available ads vs seller datasets
   - Ads datasets: campaign performance, search terms, products, targeting, placements, etc.
   - Seller datasets: sales & traffic, economics, brand analytics, listing health (requires Seller Central connection)

2. **`describe_dataset`** for the chosen dataset id:
   - Returns available metrics, segments (dimensions), grain, time coverage, and notes
   - Check `report_status` and `data_coverage` to understand freshness

3. **`query`** with:
   - `time_range` — start/end dates or relative periods (e.g., `last_7_days`, `last_30_days`)
   - `metrics` — what to measure (e.g., `impressions`, `clicks`, `sales`, `acos`)
   - `dimensions` — how to group (e.g., `campaign_name`, `targeting_type`, `asin`)
   - `granularity` — time bucket (e.g., `day`, `week`, `month`) or `total` for aggregates
   - `filters` — narrow the scope (e.g., `campaign_name = 'Brand Defense'`)
   - `limit` — cap rows returned (default 1000, max 10000)

### When to Use Raw SQL (`run_query`)

Use `run_query` **only** when `query` cannot express the ask:

- Cross-table joins (e.g., combining ads and seller datasets)
- Window functions (e.g., `LAG`, `LEAD`, `ROW_NUMBER`)
- Arrays or advanced SQL features (e.g., `UNNEST`, CTEs)

Always include a comment explaining why `query` was insufficient.

## Dataset Requirements

- **Ads datasets** need `profileId` (from `list_profiles`)
- **Seller datasets** need `seller_id` (from `account_info_id` in `list_profiles`) + `marketplace_id` (from `marketplace_string_id`)
- **KDP accounts** have ads only — skip seller datasets if Seller Central is not connected

## Currency & Data Interpretation

- **Currency is native:** All money values are in the profile's native currency (e.g., `USD`, `GBP`, `CAD`). Do not convert or mix currencies across profiles.
- **`report_status` of `NOT_STARTED`** does not mean there is no data. Check actual data coverage in `describe_dataset`.
- **Seller vs KDP distinction:**
  - **Seller:** Ads + Seller Central datasets (full analytics, including listing health)
  - **KDP:** Ads only (no Seller Central datasets available)

## Catalog Health Checks

Listing health checks are available via `get_health_check` (MCP-based, returns `listing_health` data). This is **not** the same as direct REST API access to SP listings/catalog endpoints (those are v0.2, not in production yet).

## Analysis Workflow

1. **Start with session context:**
   ```
   get_session_context with profileId
   ```

2. **Explore available datasets:**
   ```
   list_datasets with profileId or seller_id
   ```

3. **Understand a dataset:**
   ```
   describe_dataset with dataset_id
   ```

4. **Run structured queries:**
   ```
   query with time_range, metrics, dimensions, granularity, filters
   ```

5. **Interpret results:**
   - Explain trends, outliers, and actionable insights
   - Highlight opportunities for optimization
   - Flag issues requiring investigation

## When to Propose Changes

**Do not propose Amazon writes from this skill.** That is `propose-ad-changes`.

If analysis points at a concrete action (pause/enable, bid change, budget adjustment, negative keyword, etc.), switch to the **propose-ad-changes** skill and use `propose_action`.

## Security Reminders

- **Never** echo API keys, Bearer tokens, or `wpp_ak_*` secrets in query results, chat, or logs.
- If a query returns sensitive data (customer emails, phone numbers, etc.), redact or summarize instead of echoing verbatim.
