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

Call `get_session_context` **once** at session start (after picking a `profileId`) if you have not already. It loads preferences, account guidance, benchmarks, pending changes, runbooks, data-model gotchas, and the credential's grants (`key_grants`) in one round-trip. Refresh preferences with `list_preferences`; do not repeat `get_session_context` mid-session.

## Query Best Practices

### Prefer Structured `query`

The structured `query` tool is preferred over raw SQL `run_query`. A rejected `query` returns the legal vocabulary so you can self-correct:

1. **`describe_dataset` with no dataset** — lists the available ads vs seller datasets
   - Ads datasets: campaign performance, search terms, products, targeting, placements, etc.
   - Seller datasets: sales & traffic, economics, brand analytics, listing health (requires Seller Central connection)

2. **`describe_dataset` with the chosen dataset id:**
   - Returns available metrics, segments (dimensions), grain, time coverage, and notes
   - Check `report_status` and `data_coverage` to understand freshness

3. **`query`** with:
   - `time_range` — required `{start, end}` as ISO dates (`YYYY-MM-DD`), clamped to the account's available data
   - `metrics` — what to measure (e.g., `impressions`, `clicks`, `sales`, `acos`)
   - `dimensions` — how to group (e.g., `campaign_name`, `targeting_type`, `asin`)
   - `granularity` — `total` (default, one row per dimension over the window), `daily`, `weekly`, or `monthly`
   - `filters` — narrow the scope on dimensions, before aggregation (e.g., `{field: 'campaign_name', op: '=', value: 'Brand Defense'}`)
   - `having` — filter on metrics after aggregation (metrics are not filtered in `filters`)
   - `limit` — cap rows returned (default 100; clamped to the credential's maximum)

### When to Use Raw SQL (`run_query`)

`run_query` is a normal read (covered by Amazon Ads read access), not a special opt-in. Use it **only** when `query` cannot express the ask, and call `describe_table` first for the tables, columns, and gotchas:

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

Listing health checks are available via `get_health_check` (MCP-based, returns `listing_health` data). This is **not** full listing/catalog record retrieval (planned for a future release).

## Analysis Workflow

1. **Start with session context:**
   ```
   get_session_context with profileId
   ```
   (once per session; skip if already done)

2. **Explore available datasets:**
   ```
   describe_dataset with no dataset
   ```

3. **Understand a dataset:**
   ```
   describe_dataset with dataset_id
   ```

4. **Run structured queries:**
   ```
   query with time_range, metrics, dimensions, granularity, filters, having
   ```

5. **Interpret results:**
   - Explain trends, outliers, and actionable insights
   - Highlight opportunities for optimization
   - Flag issues requiring investigation

## When to Propose Changes

**Do not propose Amazon writes from this skill.** That is `propose-ad-changes`.

If analysis points at a concrete action (pause/enable, bid change, budget adjustment, negative keyword, etc.), switch to the **propose-ad-changes** skill and use `submit_mutation`.

## Security Reminders

- **Never** echo API keys, Bearer tokens, or `wpp_ak_*` secrets in query results, chat, or logs.
- If a query returns sensitive data (customer emails, phone numbers, etc.), redact or summarize instead of echoing verbatim.
