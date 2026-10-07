# WisePPC Cursor Plugin

Connect Cursor AI to **[WisePPC Amazon Ads and seller analytics](https://wiseppc.com/ai-integration/)** through the WisePPC MCP server (`https://mcp.wiseppc.com/mcp`).

---

## What it does

- **HTTP MCP** to production WisePPC, authenticated with a `wpp_ak_*` API key **or** OAuth to the WisePPC user
- **Skills:** connect, analyze ads/seller data, submit ad and seller listing changes (approved by a person or sent directly, per your key's grants)
- **Rule:** no secrets in chat, prefer structured `query`, session context once at start, Ads writes via `submit_mutation` and listing writes via the REST listings route
- **Data:** Ads campaigns and performance, search terms, impression share, seller reports (sales & traffic, economics, inventory, fees, brand analytics), Brand Store, A+ content, Amazon Attribution links, catalog health checks

## Who it's for

- A **WisePPC account** (active subscription)
- **Seller:** Amazon Ads **and** Seller Central connected to WisePPC (full analytics)
- **Vendor:** Amazon Ads only (1P, no Seller Central — seller datasets, A+ content, listings and product type definitions do not apply)
- **Agency:** Amazon Ads profiles; seller datasets only where Seller Central is connected
- **KDP:** Amazon Ads only (no Seller Central — ads datasets only)

---

## New to WisePPC?

If you don't have a WisePPC account yet, follow this setup flow:

1. **Sign up for WisePPC:**
   - Visit **[wiseppc.com](https://wiseppc.com)** to create an account (subscription required)
   - Learn about AI integration at **[wiseppc.com/ai-integration](https://wiseppc.com/ai-integration/)**

2. **Connect Amazon Ads** (required):
   - Sign in to **[app.wiseppc.com](https://app.wiseppc.com)**
   - Connect your Amazon Ads account in settings
   - This enables ads analytics

3. **Connect Seller Central** (optional, for sellers only):
   - Also connect Amazon Seller Central in **[app.wiseppc.com](https://app.wiseppc.com)**
   - This enables seller analytics (sales, traffic, economics, brand analytics)
   - **KDP authors** and **Vendors** (ads-only) can skip this step

4. **Get your API key:**
   - In WisePPC webapp, go to **API keys** page
   - Create a new key (starts with `wpp_ak_*`)
   - Or use **OAuth to WisePPC** (alternative to API key)

5. **Install the plugin** (see below) and set `WISEPPC_API_KEY` in Cursor config

Without a WisePPC account and Amazon connections, the MCP tools cannot return your data.

---

## Installation

### From cursor.directory

Find **WisePPC** on [cursor.directory](https://cursor.directory/plugins) and follow its install steps.

### From GitHub (local plugin)

1. **Clone this repository** into Cursor's local plugins folder:
   ```bash
   git clone https://github.com/wiseppc/cursor-plugin ~/.cursor/plugins/local/wiseppc
   ```
2. **Restart Cursor**, or run **Developer: Reload Window**
3. **Open Customize** and check that the WisePPC skills, rule and MCP server appear
4. **Configure authentication** on the plugin (**Configure**):
   - Set **WisePPC API key** (`WISEPPC_API_KEY`), **or**
   - Use **OAuth to WisePPC** (alternative to API key)

To update later, run `git pull` in that folder and reload the window.

**Teams and Enterprise:** local plugins need the admin setting **Allow Local Plugin Imports**. An admin can instead add this repository to a team marketplace (**Dashboard → Plugins & MCPs → Import from Repo**) so developers install it from **Customize**.

### Getting an API Key

1. **Sign in to WisePPC:** [app.wiseppc.com](https://app.wiseppc.com)
2. **Navigate to API keys** page
3. **Create a new key** (starts with `wpp_ak_*`)
4. **Store in Cursor config:** Plugins → Configure → set `WISEPPC_API_KEY`

**Security:** Never paste API keys in chat, commits, or logs. Keys belong in Cursor's plugin config only.

---

## First Steps

Once installed and authenticated:

1. **List profiles:**
   ```
   list_profiles
   ```
   Returns advertising `profileId`s (one per marketplace) with `currency_code` and `account_info_id` (seller_id for seller datasets).

2. **Select a business profile** (if you manage multiple):
   ```
   select_business_profile
   ```

3. **Get session context** (once, before any analysis):
   ```
   get_session_context with profileId
   ```
   Call it once at session start. It loads preferences, account guidance, pending changes, runbooks, benchmarks, data-model notes, and your key's current grants (`key_grants`) in one call. Refresh preferences with `list_preferences`.

4. **Explore datasets:**
   ```
   describe_dataset (no dataset: list) → describe_dataset (one dataset) → query
   ```
   Ads datasets take `profileId`; seller datasets take `seller_id` + `marketplace_id` from `list_profiles`. Customer search terms are in the `searchterm` dataset. For a quick "what should I focus on?", start with `get_health_check`.

5. **Analyze & submit changes:**
   - Use the **analyze-amazon-ads** skill for performance questions
   - Use the **propose-ad-changes** skill when analysis suggests an action

---

## What's in v0.1.4

This release covers what is live on **production MCP today** (`https://mcp.wiseppc.com/mcp`):

- ✅ **Ads analytics:** campaigns, search terms, products, targets, placements, conversions, impression share, change history
- ✅ **Seller analytics:** sales & traffic, economics, transactions, inventory, storage fees, returns, reimbursements, Brand Analytics, listing content (requires Seller Central connection)
- ✅ **Structured query:** `describe_dataset` (index, then one dataset) and `query` are preferred. Raw SQL via `run_query` is an escape hatch and can be a separate permission on your key
- ✅ **Current settings:** `list_campaigns`, `get_campaign_details`, and `get_ads_entities` (campaigns, ad groups, ads, targets, portfolios, creative assets), the read that pairs with `submit_mutation`
- ✅ **Brand Store, A+ content, Attribution:** `get_brand_store_page`, `get_aplus_content`, `get_attribution_link`, plus Brand Store and Attribution datasets where your account has them
- ✅ **Catalog health and history:** `get_health_check`, `get_change_history`, `get_unadvertised_products`, `product_type_definitions`
- ✅ **Whole listing and catalog records:** the same API key works on the WisePPC REST API (`https://mcp.wiseppc.com/api/v1`), which returns complete Amazon-shaped listing (`/sp/listings/2021-08-01/items/...`) and catalog item (`/sp/catalog/2022-04-01/items/...`) records from WisePPC's stored copy. API key only (OAuth is refused), one marketplace per request. MCP tool results are size-capped, so use REST when you need the whole record
- ✅ **Changes:** `submit_mutation` / `get_mutations` / `update_mutation`. Each granted write is **gated** (a person approves it in WisePPC) or **direct** (sent without review), decided by your API key's grants. OAuth sessions are read-only
- ✅ **Create and delete in Ads:** campaigns, ad groups, ads and targets (`ads.*.create` / `ads.*.delete`, built in that order; deletes cannot be undone) and portfolios (create and update)
- ✅ **Seller listing writes:** edit, create or delete a listing (`sp.listings.put` / `patch` / `delete`) through the REST listings route. Grants are per field group (title, bullet points, description, price, quantity, Item Highlight, other, create, delete); delete always waits for a person
- ✅ **Multi-Channel Fulfillment:** cancel an order (`sp.fulfillment.cancel`). It is direct only, with no review, and stops a real shipment, so confirm with the user first. Reading MCF orders is a separate sensitive opt-in (buyer details) over REST only
- ✅ **Account types:** Seller, Vendor, Agency and KDP accounts. Vendors have no Seller Central, so seller datasets, A+ content, listings and product type definitions do not apply to them
- ✅ **Key opt-ins:** raw SQL, MCF orders and other principals' queue rows are separate opt-ins, visible in `key_grants`
- ✅ **Fixing failed changes:** a change Amazon rejects stays on the WisePPC Queue with Amazon's response until a person dismisses it; the agent can submit a corrected revision linked to it, which always waits for approval
- ✅ **Preferences & runbooks:** persistent account settings and guided workflows (search-term negation and harvesting, bid right-sizing, budget caps, out-of-stock spend, and more); read the runbook before writing its queries
- ✅ **Session context:** `get_session_context` once per session for the account snapshot and your grants (`key_grants`)

### Not included

- ❌ **Whole listing and catalog records over MCP:** MCP tool results are size-capped, so complete Amazon-shaped listing and catalog item payloads come from the REST API (above), not from a tool. Catalog **health** (`get_health_check`), observed changes (`get_change_history`) and listing content in the `listing_item` dataset are available over MCP.

---

## Skills Reference

| Skill | Description | When to Use |
|-------|-------------|-------------|
| **connect-wiseppc** | Set up authentication, list profiles, start session; account types, key opt-ins, REST API for whole records | First-time setup, session start, full listing or catalog records |
| **analyze-amazon-ads** | Query ads/seller data via structured datasets; Brand Store, A+ content, Attribution; runbook list | Performance questions, seller analytics, health checks, guided runbooks |
| **propose-ad-changes** | Submit changes (gated approval or direct, per key grants) and fix failed ones | Pause/enable, bids, budgets, negatives, create/delete campaigns and ads, rules, Brand Store page ASINs, listing edits, MCF cancel |

---

## Rules & Best Practices

The plugin enforces these best practices:

- **No secrets in chat:** Never echo API keys, tokens, or `wpp_ak_*` strings
- **Prefer structured query:** Use `describe_dataset` → `query` over raw `run_query` SQL, which may need its own permission on your key
- **Writes follow your grants:** Ads changes go through `submit_mutation` and seller listing edits through the WisePPC REST listings route; each granted op is gated (webapp approval) or direct, and the key decides. Read `key_grants` instead of probing, and read the entity with `get_ads_entities` before changing it
- **Session context once:** `get_session_context` at session start, not repeated mid-session (`list_preferences` refreshes preferences)
- **Currency is native:** Money values are in the profile's native currency (no FX conversion)
- **NOT_STARTED ≠ no data:** `report_status` of `NOT_STARTED` does not mean there is no data

---

## Production Environment

This plugin connects to **production MCP only:** `https://mcp.wiseppc.com/mcp`

---

## License

MIT. Copyright (c) 2026 Crystal Logistics Corp / WisePPC. See [LICENSE](LICENSE).
