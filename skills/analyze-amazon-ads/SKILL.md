---
name: analyze-amazon-ads
description: Analyze Amazon Ads and seller data through WisePPC. Use for campaign, search-term, target and product performance, wasted spend, impression share, sales and traffic, fees and inventory, brand analytics, Brand Store, A+ content, Amazon Attribution, catalog health checks, full listing or catalog records, Multi-Channel Fulfillment orders, guided runbooks (search-term negation and harvesting, bid right-sizing, budget caps, out-of-stock spend), or "where should I focus?". Prefer structured query over raw SQL.
---

# Analyze Amazon Ads

## When to use

- Campaign, search-term, target, or product performance questions
- Budget pacing, wasted spend, or efficiency analysis
- Seller analytics: sales & traffic, economics, transactions, inventory, fees, returns, reimbursements, brand analytics
- Health checks, listing issues, or "where should I focus?" questions
- Brand Store traffic, A+ content, Amazon Attribution links, or current campaign settings
- A whole Amazon listing or catalog record (see "Whole Listing and Catalog Records")

## Before You Start

Call `get_session_context` **once** at session start (after picking a `profileId`) if you have not already. It loads preferences, account guidance, benchmarks, pending changes, runbooks, data-model gotchas, and the credential's grants (`key_grants`) in one round-trip. Refresh preferences with `list_preferences`; do not repeat `get_session_context` mid-session.

## Query Best Practices

### Prefer Structured `query`

The structured `query` tool is preferred over raw SQL `run_query`. A rejected `query` returns the legal vocabulary so you can self-correct:

1. **`describe_dataset` with no `dataset`** — the index of datasets (id, domain, grain; a snapshot dataset has no time series). Dataset families:
   - **Ads performance:** `ads_performance`, `searchterm` (customer search terms), `product_performance`, `conversion_performance`
   - **Ads entities (current settings):** `ads_campaigns`, `ads_ad_groups`, `ads_targets`, `ads_ads`, `ads_negatives`, `ads_portfolios`, plus `ads_change_history`
   - **Ads reach & off-Amazon:** `top_of_search_impression_share`, `search_impression_share`, Amazon Attribution (`amazon_attribution*`)
   - **Ads products:** `product_selector`, `product_selector_history`
   - **Brand health & Brand Store:** `amazon_brand_metrics*`, `amazon_brand_store*` — where your account has them
   - **Seller:** `sales_traffic`, `economics`, `transaction_ledger`, `consolidated_inventory`, `listing_item`, `cogs`, `fba_returns`, `monthly_storage_fees`, `long_term_storage_fees`, `reimbursements`, `customer_review_topics`, `search_term_results`, and Brand Analytics (`brand_search_query`, `brand_search_catalog`, `brand_market_basket`, `brand_repeat_purchase`)

   The live index is the authority; trust it over this list.

2. **`describe_dataset` with the chosen `dataset`:**
   - Returns metrics, segments (dimensions), grain, time coverage, filter operators, and gotchas
   - `expand` returns full entries for named fields; `detail: "full"` returns everything
   - Segments are breakdowns that **fan rows out**; attributes only label rows at the existing grain

3. **`query`** with:
   - `dataset`, plus `profileId` (Ads datasets) or `seller_id` + `marketplace_id` (seller datasets)
   - `time_range` — required `{start, end}` as ISO dates (`YYYY-MM-DD`), clamped to the account's available data
   - `metrics` — what to measure (e.g., `impressions`, `clicks`, `sales`, `acos`); ratios are computed after aggregation, never average them yourself
   - `dimensions` — how to group (e.g., `campaign_name`, `targeting_type`, `asin`)
   - `granularity` — `total` (default, one row per dimension over the window), `daily`, `weekly`, or `monthly`
   - `filters` — narrow the scope on dimensions, before aggregation (e.g., `{field: 'campaign_name', op: '=', value: 'Brand Defense'}`)
   - `having` — filter on metrics after aggregation (metrics are not filtered in `filters`)
   - `compare: "previous_period"` — adds `<metric>_prev` and `<metric>_delta` (total granularity only)
   - `sort`, `limit` (default 100, clamped to the credential's maximum) and `offset` for paging
   - `format` (`compact`, `json`, `csv`, `tsv`) and `round` shape the output

### Dataset Gotchas Worth Knowing

- **Search terms are in `searchterm`**, not in `ads_olap`. Search-term metrics are a split of the same clicks and are not additive with `ads_performance`: never sum across the two datasets.
- The newest few days are thinner than settled days (some click and view-through splits are not available yet), so be careful comparing them to older periods.
- Snapshot datasets (current settings, `listing_item`, `cogs`) have no time series; entity datasets hide archived rows by default and say so.
- Impression-share and Brand Analytics metrics are averages or latest values, never sums.

### Quick Pointers

- **"What should I focus on?"** → `get_health_check` (budget, waste, inventory, listing health, performance, anomalies), then drill in with `query`.
- **Per-ASIN ad efficiency** → `get_product_performance`; **products with no ad impressions** → `get_unadvertised_products`.
- **Current campaign settings** → `list_campaigns` / `get_campaign_details`, or `query` on `ads_campaigns` and friends. For Amazon-shaped objects (to read before a change), use `get_ads_entities` (`entity`: campaigns, ad_groups, ads, targets, portfolios, assets).
- **What changed and when** → `ads_change_history` via `query`, or `get_change_history` for one seller listing (`sku` + `sellerId`) or catalog item (`asin`); `marketplaceId` is required.
- **Methodology** → `get_runbook` (no `runbook_id` lists them; pass one, e.g. `search_term_analysis`, for the steps). **Read the runbook before writing its queries.** See "Runbooks" below.

### Seller-Side Analytics

- Needs Seller Central connected. Pass `seller_id` (= `account_info_id` from `list_profiles`) and `marketplace_id` (= `marketplace_string_id`) on `query`.
- `sales_traffic` for sessions, units and conversion; `economics` and `transaction_ledger` for profit and fees; `consolidated_inventory` for stock; storage fees, returns and reimbursements for FBA cost leaks; `brand_search_query` for what shoppers search.
- Join ads and seller views in a single raw SQL query only if `query` cannot, and say why.

### Brand Store, A+ Content, Attribution

- **Brand Store traffic:** `describe_dataset` for `amazon_brand_store*` (page views, visits, sources, ASINs). To read one page's content, find its id with `query` on `amazon_brand_store_page_snapshot`, then call `get_brand_store_page` with `profileId` and `page_id`. It reads a nightly snapshot of the draft, not the live page.
- **A+ Content:** `get_aplus_content` with `action` `search`, `get` or `asins` and a `marketplaceId`. Read-only, API key required. Some documents (for example Premium A+) have no stored body.
- **Amazon Attribution:** `get_attribution_link` with a `profileId` lists enrolment and publishers; add an `asin` to get tracked product URLs. It never enrols an advertiser.

### When to Use Raw SQL (`run_query`)

`run_query` is an escape hatch. Raw SQL can be a separate permission on the credential: check `key_grants` first, and if it is not granted, stay with `query` and tell the user what to widen. Call `describe_table` first for the tables, columns, and gotchas. Use it **only** when `query` cannot express the ask:

- Cross-table joins (e.g., combining ads and seller datasets)
- Window functions (e.g., `LAG`, `LEAD`, `ROW_NUMBER`)
- Arrays or advanced SQL features (e.g., CTEs)

Always include a comment explaining why `query` was insufficient. Every `ads_olap` scan needs a literal lower bound on `time_window_date`.

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

Listing health checks are available via `get_health_check` (returns `listing_health` among its alerts). `get_change_history` and the `listing_item` dataset add what WisePPC observed on listings.

## Whole Listing and Catalog Records

MCP tool results are size-capped. For a complete Amazon-shaped listing or catalog item, use the WisePPC REST API with the same API key: `GET /sp/listings/2021-08-01/items/{sellerId}/{sku}` and `GET /sp/catalog/2022-04-01/items/{asin}` on `https://mcp.wiseppc.com/api/v1` (API key only, one marketplace per request). The **connect-wiseppc** skill has the details. Read a listing this way before proposing a listing change.

## Multi-Channel Fulfillment

Reading Multi-Channel Fulfillment (MCF) orders is a separate sensitive opt-in because the orders carry buyer details. It is available over the REST API only (`GET /sp/fulfillment/outbound/2026-07-04/orders`, API key with the opt-in; pass the `x-wiseppc-seller-id` header if more than one seller is connected). Check `key_grants`; if it is not granted, tell the user what to widen. Cancelling an MCF order is a change: see **propose-ad-changes**.

## Runbooks

`get_runbook` with no `runbook_id` lists them live. The ids and the question each answers:

- `search_term_analysis`: which search terms to negate and which to harvest into exact match?
- `negation_gap_audit`: which terms are negated in some campaigns but still spend in others?
- `non_english_search_term_analysis`: which non-English search terms spend, and do they convert?
- `keyword_bid_rightsizing`: which keywords overbid, spend with no orders, or underbid?
- `match_type_efficiency`: how do exact, phrase, broad and auto compare?
- `auto_campaign_segmentation`: how do the four auto targeting types perform?
- `placement_bid_optimization`: how should placement bid adjustments change?
- `bid_strategy_comparison`: which bid strategy performs best?
- `asin_targeting_analysis` and `competitor_asin_targeting`: how efficient is ASIN targeting, own brand versus competitors?
- `budget_cap_detection`: which campaigns or portfolios keep hitting their budget cap?
- `portfolio_performance_audit`: how do portfolios perform, and which campaigns are orphaned?
- `parent_asin_analysis`: where do variations cannibalize or starve each other?
- `wow_anomaly_detection`: what moved between two periods?
- `day_of_week_performance`: which days convert best?
- `product_catalog_health`: how healthy is catalog coverage and eligibility?
- `product_change_impact`: how did stock, eligibility or price changes affect ad performance?
- `oos_spend_detection` and `stock_flip_waste`: what is being spent on out-of-stock or ineligible products, now and historically?
- `sb_new_to_brand_analysis`: how do Sponsored Brands new-to-brand campaigns perform?
- `sd_device_environment` and `sd_supply_os_analysis`: how does Sponsored Display perform by device, environment, supply source and operating system?

**Read the runbook before writing its queries**; it names the datasets, thresholds and traps.

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
   describe_dataset with dataset
   ```

4. **Run structured queries:**
   ```
   query with dataset, time_range, metrics, dimensions, granularity, filters, having
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
