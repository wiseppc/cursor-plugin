---
name: connect-wiseppc
description: Connect Cursor to WisePPC MCP via API key or OAuth. Use when setting up the plugin, creating a wpp_ak_ key in the WisePPC webapp, checking what a key may read or change (key_grants, opt-ins), listing ads profiles, telling Seller, Vendor, Agency and KDP accounts apart, or when a task needs a whole Amazon listing or catalog record through the WisePPC REST API.
---

# Connect WisePPC

## When to use

- First-time plugin setup
- User needs a WisePPC API key or OAuth session
- Session start before ads or seller analysis
- Listing available profiles or selecting a business profile

## Who can connect

- Requires a **WisePPC account** with an active subscription.
- **Seller:** Amazon Ads **and** Seller Central must be connected to WisePPC (full ads + seller datasets).
- **Vendor:** a first-party (1P) Amazon Ads account. Vendors have no Seller Central, so the seller datasets, A+ content, listings and product type definitions do not apply to them: stay with the ads datasets.
- **Agency:** an agency Amazon Ads account. Ads datasets apply to its profiles; use seller datasets only where Seller Central is connected.
- **KDP:** Only an Amazon Ads account connected to WisePPC (no Seller Central — ads datasets only).

`list_profiles` returns each profile's `account_type`. Check it before you reach for a seller dataset.

## Cold Start: No WisePPC Account Yet

If the user does not have a WisePPC account or Amazon connections, **stop trying MCP tools** and guide them through this flow. Without a WisePPC account + Amazon connections, MCP tools cannot return their data.

### Step-by-Step for New Users

1. **Create a WisePPC account:**
   - Visit **https://wiseppc.com** to learn about WisePPC and sign up
   - Or learn about AI integration at **https://wiseppc.com/ai-integration/**
   - A subscription is required to use WisePPC services

2. **Connect Amazon Ads** (required for all users):
   - Sign in to **https://app.wiseppc.com**
   - Navigate to account settings or integrations
   - Connect your Amazon Ads account
   - This enables ads analytics (campaigns, search terms, products, targeting, placements, impression share)

3. **Connect Seller Central** (required for Seller datasets; skip if KDP-only or Vendor):
   - In **https://app.wiseppc.com**, also connect Amazon Seller Central
   - This enables seller analytics (sales & traffic, economics, transactions, inventory, fees, brand analytics, listing health)
   - **KDP authors** who only run ads (no physical/book selling via Seller Central) can skip this step; so can **Vendors** (1P accounts have no Seller Central)

4. **Authenticate the plugin** (choose one):
   - **Option A (API key):** In WisePPC webapp, navigate to **API keys** page → create a new key → set `WISEPPC_API_KEY` in Cursor **Plugins → Configure**
   - **Option B (OAuth):** Use OAuth to WisePPC (MCP server handles the flow)
   - **Never** paste keys into chat or commit them to code

5. **Verify connection:**
   ```
   list_profiles
   ```
   Should return advertising profiles with `profileId`, `currency_code`, and `account_info_id` (if Seller Central connected)

6. **Start analyzing:**
   ```
   get_session_context with profileId
   ```
   Then proceed to analysis (`analyze-amazon-ads`) and changes (`propose-ad-changes`; approvals happen in the WisePPC webapp)

### When Auth Fails

If MCP tools return authentication errors or "no profiles found":
- **First-time users:** Walk them through the cold-start flow above
- **Existing users:** Verify their `WISEPPC_API_KEY` is set correctly in Cursor config, or re-authenticate via OAuth
- **Do not invent analytics.** Stop trying tools until authentication succeeds.

## Authentication

MCP accepts **API key** or **OAuth** to the WisePPC user.

### API Key Path

API keys (`wpp_ak_*`) are created in the **WisePPC webapp**: navigate to the **API keys** page and create a new key.

Store a key only as the plugin variable `WISEPPC_API_KEY` (Plugins → Configure). **Never** ask them to paste it into chat. **Never** echo it back to the user.

### OAuth Path

Alternatively, authenticate via OAuth to the WisePPC user account. This is handled by the MCP server and does not require storing an API key in Cursor config.

An OAuth session is **read-only**: every read and the change queue, but no Amazon writes (`submit_mutation` is not offered) and no approving. A few reads (`get_aplus_content`, `product_type_definitions`) need an API key, and so does the REST API (below). For changes, use an API key whose write grants an owner has set in WisePPC.

## Connection Steps

1. **Confirm account setup:**
   - User has a WisePPC account with active subscription
   - Amazon Ads is connected (required for all users)
   - Seller Central is connected (required for Seller datasets; KDP and Vendor users skip this)

2. **Authenticate:**
   - **Option A:** Set `WISEPPC_API_KEY` in Cursor Plugins → Configure (a `wpp_ak_*` key created in the WisePPC webapp)
   - **Option B:** Use OAuth to WisePPC (MCP handles the flow)

3. **Choose a business profile** (if user manages multiple):
   ```
   select_business_profile
   ```
   With no id it lists the business profiles; call it again with an id to choose one for the session.

4. **List advertising profiles:**
   ```
   list_profiles
   ```
   Returns advertising `profileId`s (one per marketplace). Note `currency_code` and `account_info_id` (seller_id for seller datasets).

5. **Load session context** (once, at session start, after picking a `profileId`):
   ```
   get_session_context with profileId
   ```
   This loads preferences, account guidance, benchmarks, pending changes, runbooks, data-model gotchas, and the credential's current grants (`key_grants`) in one round-trip. Apply preferences silently — do not announce them unless the user asks. Refresh preferences with `list_preferences`; do not repeat `get_session_context` mid-session.

6. **Know what the credential may do:** read `key_grants` rather than discovering limits by being refused. Permissions live on the key: reads are granted per Ads account/profile and per seller/marketplace; raw SQL (`run_query`) and some sensitive reads are separate opt-ins; writes are granted per operation on one profile or marketplace and are either **gated** (a person approves) or **direct** (sent without review). Raw SQL, Multi-Channel Fulfillment orders (buyer details) and other principals' queue rows are each a separate opt-in, visible in `key_grants`. A tool being listed does not mean the key may use it on every account. If something is refused, name the missing permission and let the user widen the key in WisePPC. Making changes needs an API key with write grants (OAuth is read-only).

## Key Concepts

- **`profileId`:** Unique identifier for an advertising profile (one per marketplace: US, UK, CA, etc.). Required for ads datasets.
- **`account_info_id` (seller_id):** Required for seller datasets. Only present if Seller Central is connected.
- **`marketplace_string_id`:** Marketplace identifier (e.g., `ATVPDKIKX0DER` for US). Required for seller datasets.
- **Business profile:** a WisePPC workspace. With several, `select_business_profile` (no id lists them, an id selects one) or pass `businessProfileId` on a call.
- **`currency_code`:** Native currency for the profile (e.g., `USD`, `GBP`, `CAD`). All money values are in this currency — no FX conversion.

## Data Coverage

- **`report_status` of `NOT_STARTED`** does not mean there is no data. It means report generation has not been triggered yet. Check actual data coverage in `describe_dataset`.

## Whole Records: the REST API

MCP tool results are size-capped, so a long listing or catalog record can come back trimmed. When you need the **whole record**, call the WisePPC REST API with the **same API key** (as `Authorization: Bearer <key>`; never print it):

- **Base:** `https://mcp.wiseppc.com/api/v1`. The OpenAPI document is at `https://mcp.wiseppc.com/api/v1/openapi.json`.
- **Full listing records:** `GET /sp/listings/2021-08-01/items/{sellerId}` (search) and `GET /sp/listings/2021-08-01/items/{sellerId}/{sku}` (one SKU). Pass `includedData` to pick sections (summaries, attributes, issues, offers, and so on).
- **Catalog items:** `GET /sp/catalog/2022-04-01/items` (pass `identifiers` with `identifiersType=ASIN`, up to 20) and `GET /sp/catalog/2022-04-01/items/{asin}`. Only ASINs your business profile has a listing for are readable.
- Responses use **Amazon's own shapes**, served from WisePPC's stored copy (no Amazon quota used; each response says when it was last synced).
- **API key only:** an OAuth session is refused. Send **one `marketplaceIds` value per request**. The key must be granted read access to that seller and marketplace.
- `seller_id` and `marketplace_id` come from `list_profiles` (`account_info_id`, `marketplace_string_id`).

Use MCP `query` and `get_ads_entities` for analysis and for reading Ads entities; use REST only when you need an Amazon-shaped whole record.

## Production Environment

This plugin connects to **production MCP:** `https://mcp.wiseppc.com/mcp` (REST API: `https://mcp.wiseppc.com/api/v1`)

## Security Reminders

- **Never** echo API keys, Bearer tokens, or `wpp_ak_*` secrets in chat, commits, or logs.
- If a tool returns a secret, confirm receipt without revealing the value.
