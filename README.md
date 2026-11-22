# WisePPC Cursor Plugin

Connect Cursor AI to **[WisePPC Amazon Ads and seller analytics](https://wiseppc.com/mcp/)** through the WisePPC MCP server (`https://mcp.wiseppc.com/mcp`).

---

## What it does

- **HTTP MCP** to production WisePPC, authenticated with a `wpp_ak_*` API key (sign-in from Cursor is coming soon)
- **Skills:** connect, analyze ads/seller data, submit ad and seller listing changes (each waits for a person's approval in WisePPC unless your connection's permissions send it directly)
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
   - Learn about the WisePPC MCP server at **[wiseppc.com/mcp](https://wiseppc.com/mcp/)**; the technical overview is at **[wiseppc.com/docs/mcp](https://wiseppc.com/docs/mcp/)**

2. **Connect Amazon Ads** (required):
   - Sign in to **[app.wiseppc.com](https://app.wiseppc.com)**
   - Connect your Amazon Ads account in settings
   - This enables ads analytics

3. **Connect Seller Central** (optional, for sellers only):
   - Also connect Amazon Seller Central in **[app.wiseppc.com](https://app.wiseppc.com)**
   - This enables seller analytics (sales, traffic, economics, brand analytics)
   - **KDP authors** and **Vendors** (ads-only) can skip this step

4. **Get your API key:**
   - In the WisePPC webapp, open **AI Integration → Connections & Permissions**
   - Create a new key (starts with `wpp_ak_*`)
   - **Sign-in (OAuth) from Cursor is coming soon;** use an API key for now

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
   - **Sign-in (OAuth) from Cursor is coming soon;** use an API key for now

To update later, run `git pull` in that folder and reload the window.

**Teams and Enterprise:** local plugins need the admin setting **Allow Local Plugin Imports**. An admin can instead add this repository to a team marketplace (**Dashboard → Plugins & MCPs → Import from Repo**) so developers install it from **Customize**.

### Getting an API Key

1. **Sign in to WisePPC:** [app.wiseppc.com](https://app.wiseppc.com)
2. **Open AI Integration → Connections & Permissions**
3. **Create a new key** (starts with `wpp_ak_*`)
4. **Store in Cursor config:** Plugins → Configure → set `WISEPPC_API_KEY`

**Security:** Never paste API keys in chat, commits, or logs. Keys belong in Cursor's plugin config only.

---

## First Steps

Once installed and authenticated:

1. **Get session context** (once, before any analysis):
   ```
   get_session_context
   ```
   With no arguments it returns your business, your connected accounts (names, marketplaces, currencies), saved preferences, and what your connection may read and change (`key_grants`). Each account comes with a `target`, such as `amazon_ads:<ads account>:<profile id>` for an Ads profile or `amazon_sp:<seller id>:<marketplace id>` for a seller marketplace. The other tools take that `target`.

2. **Open a profile's primer** (optional, before working on one Ads profile):
   ```
   get_session_context with target
   ```
   Pending changes, recent changes, benchmarks, runbooks and data notes for that profile. `section="accounts"` lists every account (paged), `section="preferences"` refreshes preferences, and `section="grants"` shows each account's write permissions.

3. **Explore datasets:**
   ```
   describe (kind="datasets") → describe (kind="datasets", id=<dataset>) → query
   ```
   Ads datasets take an `amazon_ads` target; seller datasets take an `amazon_sp` target. Customer search terms are in the `searchterm` dataset. For a quick "what should I focus on?", start with `get_report` (`report="health_check"`).

4. **Analyze & submit changes:**
   - Use the **analyze-amazon-ads** skill for performance questions
   - Use the **propose-ad-changes** skill when analysis suggests an action

---

## What's included

This covers what is live on **production MCP today** (`https://mcp.wiseppc.com/mcp`):

- ✅ **Ads analytics:** campaigns, search terms, products, targets, placements, conversions, impression share, change history
- ✅ **Seller analytics:** sales & traffic, economics, transactions, inventory, storage fees, returns, reimbursements, Brand Analytics, listing content (requires Seller Central connection)
- ✅ **Structured query:** `describe` (`kind="datasets"`: the index, then one dataset) and `query` are preferred. Raw SQL via `run_sql` is an escape hatch for what `query` cannot express
- ✅ **Current settings:** `get_entities` (campaigns, ad groups, ads, targets, portfolios, creative assets, campaign settings and details, Brand Store pages), the read that pairs with `submit_mutation`
- ✅ **Reports:** `get_report` for the health check, product performance, unadvertised products and Amazon Attribution links
- ✅ **Brand Store, A+ content, Attribution:** Brand Store pages through `get_entities`, A+ content through `get_marketplace_data`, Attribution links through `get_report`, plus Brand Store and Attribution datasets where your account has them
- ✅ **Catalog history and category schemas:** `get_history` (observed changes to a listing or catalog item) and `get_marketplace_data` (`resource="product_type_definitions"`)
- ✅ **Whole listing and catalog records:** the WisePPC REST API (`https://mcp.wiseppc.com/api/v1`) returns complete Amazon-shaped listing (`/sp/listings/2021-08-01/items/...`) and catalog item (`/sp/catalog/2022-04-01/items/...`) records from WisePPC's stored copy, one marketplace per request. Use your API key, or in a sign-in connection a short-lived REST token from `issue_rest_token` with the same permissions as the connection. MCP tool results are size-capped, so use REST when you need the whole record
- ✅ **Changes:** `submit_mutation` / `get_mutations` / `update_mutation`. See "Changes follow your connection's permissions" below
- ✅ **Create and delete in Ads:** campaigns, ad groups, ads and targets (`ads.*.create` / `ads.*.delete`, built in that order; deletes cannot be undone and always wait for a person's approval) and portfolios (create and update)
- ✅ **Seller listing writes:** edit, create or delete a listing (`sp.listings.put` / `patch` / `delete`) through the REST listings route. Permissions are per field group (title, bullet points, description, price, quantity, Item Highlight, other, create, delete); delete always waits for a person
- ✅ **Multi-Channel Fulfillment:** cancel an order (`sp.fulfillment.cancel`). It stops a real shipment and cannot be undone, so confirm with the user first. Reading MCF orders (buyer details) needs the personal-data opt-in and is over REST only
- ✅ **Account types:** Seller, Vendor, Agency and KDP accounts. Vendors have no Seller Central, so seller datasets, A+ content, listings and product type definitions do not apply to them
- ✅ **Personal data opt-in:** buyer and customer names, emails, phones and addresses are the one opt-in on a connection, visible in `key_grants.personal_data`
- ✅ **Fixing failed changes:** a change Amazon rejects stays on the WisePPC Queue with Amazon's response until a person dismisses it; the agent can submit a corrected revision linked to it
- ✅ **Preferences & runbooks:** persistent account settings (`set_bp_data`, `section="preferences"`) and guided workflows (`describe`, `kind="runbooks"`: search-term negation and harvesting, bid right-sizing, budget caps, out-of-stock spend, and more); read the runbook before writing its queries
- ✅ **Session context:** `get_session_context` once per session for your accounts, their `target` ids and your permissions (`key_grants`)

### Not included

- ❌ **Whole listing and catalog records over MCP:** MCP tool results are size-capped, so complete Amazon-shaped listing and catalog item payloads come from the REST API (above), not from a tool. Catalog **health** (`get_report`, `report="health_check"`), observed changes (`get_history`) and listing content in the `listing_item` dataset are available over MCP.
- ❌ **Publishing a Brand Store page:** only the page draft can be changed.

---

## Changes follow your connection's permissions

This is the same for an API key and a sign-in (OAuth) connection. By default, every change on an account your connection can read is **gated**: it waits on the WisePPC Queue until a person approves it, and nothing is sent to Amazon before then. A business owner or admin can set one operation to **direct** (queued and sent without review) or **off** in **AI Integration → Connections & Permissions**; a permission change applies on the next call. The permission decides, not the request, and only a person can approve.

A completed change means it was accepted for sending; confirm the resulting state with a read when it matters. When a change fails, the agent reads the reason and can submit a corrected revision. A refusal names the missing permission; widening it is up to a business owner or admin.

---

## Skills Reference

| Skill | Description | When to Use |
|-------|-------------|-------------|
| **connect-wiseppc** | Set up authentication, start the session, find account `target` ids; account types, permissions, REST API for whole records | First-time setup, session start, full listing or catalog records |
| **analyze-amazon-ads** | Query ads/seller data via structured datasets; reports, Brand Store, A+ content, Attribution; runbook list | Performance questions, seller analytics, health checks, guided runbooks |
| **propose-ad-changes** | Submit changes (approved by a person by default, or direct where permitted) and fix failed ones | Pause/enable, bids, budgets, negatives, create/delete campaigns and ads, rules, Brand Store page ASINs, listing edits, MCF cancel |

---

## Rules & Best Practices

The plugin enforces these best practices:

- **No secrets in chat:** Never echo API keys, tokens, or `wpp_ak_*` strings
- **Prefer structured query:** Use `describe` → `query` over raw `run_sql` SQL
- **Writes follow your permissions:** Ads changes go through `submit_mutation` and seller listing edits through the WisePPC REST listings route; each one waits for a person's approval unless your connection sends that operation directly. Read `key_grants` instead of probing, and read the entity with `get_entities` before changing it
- **Session context once:** `get_session_context` at session start, not repeated mid-session (`section="preferences"` refreshes preferences)
- **Currency is native:** Money values are in the profile's native currency (no FX conversion)
- **NOT_STARTED ≠ no data:** `report_status` of `NOT_STARTED` does not mean there is no data

---

## Production Environment

This plugin connects to **production MCP only:** `https://mcp.wiseppc.com/mcp`

---

## License

MIT. Copyright (c) 2026 Crystal Logistics Corp / WisePPC. See [LICENSE](LICENSE).
