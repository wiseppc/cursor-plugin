---
name: connect-wiseppc
description: Connect Cursor to WisePPC MCP via API key or OAuth. Use when setting up the plugin, minting a wpp_ak_ key, listing ads profiles, or starting a session.
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
- **KDP:** Only an Amazon Ads account connected to WisePPC (no Seller Central — ads datasets only).

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
   - This enables ads analytics (campaigns, search terms, products, targeting, placements)

3. **Connect Seller Central** (required for Seller datasets; skip if KDP-only):
   - In **https://app.wiseppc.com**, also connect Amazon Seller Central
   - This enables seller analytics (sales & traffic, economics, brand analytics, listing health)
   - **KDP authors** who only run ads (no physical/book selling via Seller Central) can skip this step

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
   Then proceed to analysis and proposals (approvals happen in WisePPC webapp)

### When Auth Fails

If MCP tools return authentication errors or "no profiles found":
- **First-time users:** Walk them through the cold-start flow above
- **Existing users:** Verify their `WISEPPC_API_KEY` is set correctly in Cursor config, or re-authenticate via OAuth
- **Do not invent analytics.** Stop trying tools until authentication succeeds.

## Authentication

MCP accepts **API key** or **OAuth** to the WisePPC user.

### API Key Path

API keys (`wpp_ak_*`) can be created:

1. **WisePPC webapp:** Navigate to the **API keys** page and create a new key, or
2. **MCP auto-generation:** After the user is authenticated via OAuth, MCP can mint a key for you.

Store a key only as the plugin variable `WISEPPC_API_KEY` (Plugins → Configure). **Never** ask them to paste it into chat. **Never** echo it back to the user.

### OAuth Path

Alternatively, authenticate via OAuth to the WisePPC user account. This is handled by the MCP server and does not require storing an API key in Cursor config.

## Connection Steps

1. **Confirm account setup:**
   - User has a WisePPC account with active subscription
   - Amazon Ads is connected (required for all users)
   - Seller Central is connected (required for Seller datasets; KDP users skip this)

2. **Authenticate:**
   - **Option A:** Set `WISEPPC_API_KEY` in Cursor Plugins → Configure (from a webapp- or MCP-minted `wpp_ak_*` key)
   - **Option B:** Use OAuth to WisePPC (MCP handles the flow)

3. **List business profiles** (if user manages multiple):
   ```
   list_business_profiles
   ```
   Then `select_business_profile` to choose one for the session.

4. **List advertising profiles:**
   ```
   list_profiles
   ```
   Returns advertising `profileId`s (one per marketplace). Note `currency_code` and `account_info_id` (seller_id for seller datasets).

5. **Load session context** (before any analysis or proposals):
   ```
   get_session_context with profileId
   ```
   This loads preferences, benchmarks, pending actions, runbooks, and data-model gotchas in one round-trip. Apply preferences silently — do not announce them unless the user asks.

## Key Concepts

- **`profileId`:** Unique identifier for an advertising profile (one per marketplace: US, UK, CA, etc.). Required for ads datasets.
- **`account_info_id` (seller_id):** Required for seller datasets. Only present if Seller Central is connected.
- **`marketplace_string_id`:** Marketplace identifier (e.g., `ATVPDKIKX0DER` for US). Required for seller datasets.
- **`currency_code`:** Native currency for the profile (e.g., `USD`, `GBP`, `CAD`). All money values are in this currency — no FX conversion.

## Data Coverage

- **`report_status` of `NOT_STARTED`** does not mean there is no data. It means report generation has not been triggered yet. Check actual data coverage in `describe_dataset`.

## Production Environment

This plugin connects to **production MCP:** `https://mcp.wiseppc.com/mcp`

## Security Reminders

- **Never** echo API keys, Bearer tokens, or `wpp_ak_*` secrets in chat, commits, or logs.
- If a tool returns a secret, confirm receipt without revealing the value.
