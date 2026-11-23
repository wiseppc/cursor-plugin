---
name: connect-wiseppc
description: Connect Cursor to WisePPC MCP by signing in with a WisePPC account (or with an API key). Use when setting up the plugin, signing in from Cursor's MCP settings, connecting with a wpp_ak_ key instead, checking what a connection may read or change (key_grants, opt-ins), listing ads profiles and their target ids, telling Seller, Vendor, Agency and KDP accounts apart, or when a task needs a whole Amazon listing or catalog record through the WisePPC REST API.
---

# Connect WisePPC

## When to use

- First-time plugin setup
- User needs to sign in to WisePPC from Cursor, or wants to use an API key instead
- Session start before ads or seller analysis
- Listing available profiles and their `target` ids

## Who can connect

- Requires a **WisePPC account** with an active subscription.
- **Seller:** Amazon Ads **and** Seller Central must be connected to WisePPC (full ads + seller datasets).
- **Vendor:** a first-party (1P) Amazon Ads account. Vendors have no Seller Central, so the seller datasets, A+ content, listings and product type definitions do not apply to them: stay with the ads datasets.
- **Agency:** an agency Amazon Ads account. Ads datasets apply to its profiles; use seller datasets only where Seller Central is connected.
- **KDP:** Only an Amazon Ads account connected to WisePPC (no Seller Central — ads datasets only).

`get_session_context(section="accounts")` returns each profile's `account_type`. Check it before you reach for a seller dataset.

## Cold Start: No WisePPC Account Yet

If the user does not have a WisePPC account or Amazon connections, **stop trying MCP tools** and guide them through this flow. Without a WisePPC account + Amazon connections, MCP tools cannot return their data.

### Step-by-Step for New Users

1. **Create a WisePPC account:**
   - Visit **https://wiseppc.com** to learn about WisePPC and sign up
   - Or learn about the WisePPC MCP server at **https://wiseppc.com/mcp/** (technical overview: **https://wiseppc.com/docs/mcp/**)
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

4. **Sign in from Cursor** (see "Sign In" below): open Cursor's MCP settings and sign in (connect) next to the `wiseppc` server, then choose the business on WisePPC's approval page and connect. No API key is needed. (Using a key instead is described under "API Key Instead".)

5. **Verify connection:**
   ```
   get_session_context
   ```
   Should return the business and the connected accounts, each with a `target` (for example `amazon_ads:<ads account>:<profile id>`, and `amazon_sp:<seller id>:<marketplace id>` if Seller Central is connected)

6. **Start analyzing:** pick a `target` and go on to analysis (`analyze-amazon-ads`) and changes (`propose-ad-changes`; approvals happen in the WisePPC webapp)

### When Auth Fails

If MCP tools return authentication errors or no accounts:
- **First-time users:** Walk them through the cold-start flow above
- **Existing users:** ask them to sign in again from Cursor's MCP settings (next to `wiseppc`). If they connected an API key in an earlier plugin version, that key is no longer read by the plugin: sign in once, or use the key as described under "API Key Instead"
- **Key users:** check that their own `~/.cursor/mcp.json` entry carries the key and that the plugin's `wiseppc` server is disabled, so only one connection is active
- **Do not invent analytics.** Stop trying tools until authentication succeeds.

## Authentication

The plugin connects by **signing in** with the user's WisePPC account (OAuth). The plugin itself holds no key. A WisePPC **API key** is the alternative, set up by the user outside the plugin.

### Sign In

1. In Cursor's MCP settings, sign in (connect) next to the `wiseppc` server.
2. The browser opens WisePPC's approval page, which shows the request comes from Cursor. The user signs in to WisePPC if asked, chooses the business and connects.
3. Call `get_session_context` to confirm the connection.

From the command line (Cursor's CLI agent): `cursor-agent mcp enable wiseppc`, then `cursor-agent mcp login wiseppc`.

- Only a signed-in person can approve a connection; Cursor never sees the WisePPC password.
- The connection reads what its permissions allow, and its changes follow the same rules as an API key (below). Owners and admins manage both in **AI Integration → Connections & Permissions**; a permission change applies on the next call, with no reconnecting.

### API Key Instead

API keys (`wpp_ak_*`) are created in the **WisePPC webapp**: open **AI Integration → Connections & Permissions** and create a new key. Keys also work for the WisePPC REST API.

The plugin does not take a key. A key user adds their own server entry to `~/.cursor/mcp.json` (for example named `wiseppc-key`, with `"url": "https://mcp.wiseppc.com/mcp"` and the header `"Authorization": "Bearer wpp_ak_..."`), then disables the plugin's `wiseppc` server in Cursor's MCP settings so only one connection is active. The README has the full example.

**Never** ask the user to paste a key into chat. **Never** echo it back, and never write it into a file yourself.

### One business per connection

A sign-in connection or an API key belongs to **one business**. `get_session_context(section="business_profiles")` shows it; it cannot switch to another. A different business needs its own sign-in (sign in again and choose it) or its own key.

## Connection Steps

1. **Confirm account setup:**
   - User has a WisePPC account with active subscription
   - Amazon Ads is connected (required for all users)
   - Seller Central is connected (required for Seller datasets; KDP and Vendor users skip this)

2. **Authenticate:**
   - **Sign in** from Cursor's MCP settings next to `wiseppc` (see "Sign In")
   - **Or** use an API key in the user's own `~/.cursor/mcp.json` entry, with the plugin's `wiseppc` server disabled (see "API Key Instead")

3. **Load session context** (once, at session start):
   ```
   get_session_context
   ```
   With no arguments it returns the business, the connected accounts with their `target` ids, names, marketplaces and currencies, the connection's permissions (`key_grants`) and saved preferences, in one round-trip. Apply preferences silently — do not announce them unless the user asks. Do not repeat it mid-session; use a `section` for one part:
   - `section="accounts"` — every account, paged, with `profile_id`, `account_type`, `currency_code`, `account_info_id` (the seller id) and `marketplace_string_id`
   - `section="preferences"` — refresh preferences
   - `section="grants"` — each unit's write operations and modes

4. **Open a profile's primer** before working on one Ads profile:
   ```
   get_session_context with target (an amazon_ads target)
   ```
   It returns that profile's pending changes, recent changes, benchmarks, runbooks and data gotchas.

5. **Know what the connection may do:** read `key_grants` rather than discovering limits by being refused.
   - **Reads:** every read-only dataset is available for the accounts and marketplaces the connection selects. Personal data (a buyer's or customer's name, email, phone or address, including Multi-Channel Fulfillment orders) is the one opt-in: `key_grants.personal_data` says whether it is on.
   - **Writes:** by default every write on what the connection reads is **gated**: a person approves it in WisePPC before anything is sent. An owner or admin can set one operation to **direct** (sent without review) or **off**. `key_grants.write_default` is `gated` or `off`; the `write_for_*` lists hold the exceptions.
   - A tool being listed does not mean the connection may use it on every account. If something is refused, name the missing permission and let the user widen it in WisePPC (**AI Integration → Connections & Permissions**).

## Key Concepts

- **`target`:** the account a tool works on, as `kind:account[:unit]`, copied from `get_session_context`. `amazon_ads:<ads account>:<profile id>` selects one advertising profile (one per marketplace: US, UK, CA, etc.) for ads datasets and tools; `amazon_sp:<seller id>:<marketplace id>` selects one seller marketplace for seller datasets and tools.
- **`profile_id`:** an advertising profile's id; it is the unit of its `amazon_ads` target.
- **`account_info_id` (seller id) and `marketplace_string_id`** (e.g., `ATVPDKIKX0DER` for US): the seller and marketplace of an `amazon_sp` target. The seller id is only present if Seller Central is connected.
- **Business profile:** a WisePPC workspace. A key or connection belongs to one.
- **`currency_code`:** Native currency for the profile (e.g., `USD`, `GBP`, `CAD`). All money values are in this currency — no FX conversion.

## Data Coverage

- **`report_status` of `NOT_STARTED`** does not mean there is no data. It means report generation has not been triggered yet. Check actual data coverage with `describe(kind="datasets", id=<dataset>)`.

## Whole Records: the REST API

MCP tool results are size-capped, so a long listing or catalog record can come back trimmed. When you need the **whole record**, call the WisePPC REST API:

- **In a sign-in (OAuth) connection:** call `issue_rest_token` for a short-lived REST token with the same permissions as the connection, never wider (15 minutes by default, up to 60; call it again when it expires). Send it as `Authorization: Bearer <token>`. Keep it in an environment variable; never print it or write it to a file.
- **With an API key:** send the same key as `Authorization: Bearer <key>`; never print it.
- **Base:** `https://mcp.wiseppc.com/api/v1`. The OpenAPI document is at `https://mcp.wiseppc.com/api/v1/openapi.json`.
- **Full listing records:** `GET /sp/listings/2021-08-01/items/{sellerId}` (search) and `GET /sp/listings/2021-08-01/items/{sellerId}/{sku}` (one SKU). Pass `includedData` to pick sections (summaries, attributes, issues, offers, and so on).
- **Catalog items:** `GET /sp/catalog/2022-04-01/items` (pass `identifiers` with `identifiersType=ASIN`, up to 20) and `GET /sp/catalog/2022-04-01/items/{asin}`. Only ASINs your business profile has a listing for are readable.
- Responses use **Amazon's own shapes**, served from WisePPC's stored copy (no Amazon quota used; each response says when it was last synced).
- Send **one `marketplaceIds` value per request**. The connection must be able to read that seller and marketplace.
- The seller id and marketplace id are the two parts of an `amazon_sp` target (`account_info_id` and `marketplace_string_id` in `get_session_context(section="accounts")`).

Use MCP `query` and `get_entities` for analysis and for reading Ads entities; use REST only when you need an Amazon-shaped whole record.

## Production Environment

This plugin connects to **production MCP:** `https://mcp.wiseppc.com/mcp` (REST API: `https://mcp.wiseppc.com/api/v1`)

## Security Reminders

- **Never** echo API keys, REST tokens, Bearer tokens, or `wpp_ak_*` secrets in chat, commits, or logs.
- If a tool returns a secret, confirm receipt without revealing the value.
