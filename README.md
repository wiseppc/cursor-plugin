# WisePPC Cursor Plugin

Connect Cursor AI to **WisePPC** (`https://mcp.wiseppc.com/mcp`) for Amazon Ads and seller analytics. Available on the **Cursor Marketplace** and compatible with **Grok Bot Plugins**.

---

## What it does

- **HTTP MCP** to production WisePPC, authenticated with a `wpp_ak_*` API key **or** OAuth to the WisePPC user
- **Skills:** connect, analyze ads/seller data, submit ad changes (approved by a person or sent directly, per your key's grants)
- **Rule:** no secrets in chat, prefer structured `query`, session context once at start, writes only via `submit_mutation`
- **Data:** Ads campaigns, seller reports, catalog health checks, performance analytics

## Who it's for

- A **WisePPC account** (active subscription)
- **Seller:** Amazon Ads **and** Seller Central connected to WisePPC (full analytics)
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
   - **KDP authors** (ads-only) can skip this step

4. **Get your API key:**
   - In WisePPC webapp, go to **API keys** page
   - Create a new key (starts with `wpp_ak_*`)
   - Or use **OAuth to WisePPC** (alternative to API key)

5. **Install the plugin** (see below) and set `WISEPPC_API_KEY` in Cursor config

Without a WisePPC account and Amazon connections, the MCP tools cannot return your data.

---

## Installation

### From Cursor Marketplace

1. **Open Cursor IDE** → **Plugins** → **Marketplace** (or **Customize** in Grok Bot)
2. **Search for "WisePPC"** and click **Install**
3. **Configure authentication:**
   - Open **Plugins → Configure**
   - Set **WisePPC API key** (`WISEPPC_API_KEY`), **or**
   - Use **OAuth to WisePPC** (alternative to API key)

### Getting an API Key

1. **Sign in to WisePPC:** [app.wiseppc.com](https://app.wiseppc.com)
2. **Navigate to API keys** page
3. **Create a new key** (starts with `wpp_ak_*`)
4. **Store in Cursor config:** Plugins → Configure → set `WISEPPC_API_KEY`

**Security:** Never paste API keys in chat, commits, or logs. Keys belong in Cursor's plugin config only.

### Developer Testing

To test plugin changes locally:

```bash
~/.cursor/plugins/local/wiseppc/
```

Copy or symlink the repo there. Folder name must match plugin `name` in `plugin.json`.

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

5. **Analyze & submit changes:**
   - Use the **analyze-amazon-ads** skill for performance questions
   - Use the **propose-ad-changes** skill when analysis suggests an action

---

## What's in v0.1.2

This release covers what is live on **production MCP today** (`https://mcp.wiseppc.com/mcp`):

- ✅ **Ads analytics:** campaigns, search terms, products, targeting, placements
- ✅ **Seller analytics:** sales & traffic, economics, brand analytics (requires Seller Central connection)
- ✅ **Catalog health:** listing health checks via `get_health_check` (MCP-based)
- ✅ **Structured query:** `describe_dataset`, `query` (preferred); raw SQL via `run_query` is a normal read
- ✅ **Changes:** `submit_mutation` / `get_mutations` / `update_mutation`. Each granted write is **gated** (a person approves it in WisePPC) or **direct** (sent without review), decided by your API key's grants. OAuth sessions are read-only today
- ✅ **Fixing failed changes:** a change Amazon rejects stays on the WisePPC Queue with Amazon's response until a person dismisses it; the agent can submit a corrected revision linked to it, which always waits for approval
- ✅ **Preferences & runbooks:** persistent account settings and guided workflows
- ✅ **Session context:** `get_session_context` once per session for the account snapshot and your grants

### What's NOT in v0.1.2

- ❌ **Full Amazon listing and catalog record retrieval:** Complete Amazon-shaped payloads for listing and catalog items are not available yet
  - v0.1.2 includes catalog **health** insights via `get_health_check` only
  - **Planned for a future release**

> **Note:** v0.1.2 provides catalog health data (issue detection and recommendations), not full listing/catalog records. Full record retrieval delivers complete Amazon-stored details for each item.

---

## Skills Reference

| Skill | Description | When to Use |
|-------|-------------|-------------|
| **connect-wiseppc** | Set up authentication, list profiles, start session | First-time setup, session start |
| **analyze-amazon-ads** | Query ads/seller data via structured datasets | Performance questions, health checks |
| **propose-ad-changes** | Submit changes (gated approval or direct, per key grants) | Pause/enable, bids, budgets, negatives |

---

## Rules & Best Practices

The plugin enforces these best practices:

- **No secrets in chat:** Never echo API keys, tokens, or `wpp_ak_*` strings
- **Prefer structured query:** Use `describe_dataset` → `query` over raw `run_query` SQL
- **Writes follow your grants:** All Amazon changes go through `submit_mutation`; each granted op is gated (webapp approval) or direct, and the key decides. Read `key_grants` instead of probing
- **Session context once:** `get_session_context` at session start, not repeated mid-session (`list_preferences` refreshes preferences)
- **Currency is native:** Money values are in the profile's native currency (no FX conversion)
- **NOT_STARTED ≠ no data:** `report_status` of `NOT_STARTED` does not mean there is no data

---

## Production Environment

This plugin connects to **production MCP only:** `https://mcp.wiseppc.com/mcp`

---

## Logo

`assets/logo.svg` is the official WisePPC wordmark copied from `wiseppc-webapp/public/logo.svg`.

---

## License

**Proprietary.** All Rights Reserved. Crystal Logistics Corp / WisePPC.
