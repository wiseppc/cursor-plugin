# Changelog

All notable changes to the WisePPC Cursor plugin will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [0.3.1] - 2026-11-24

### Changed

- The sign-in steps name Cursor's real buttons: open **Cursor Settings → Tools & MCPs**, select **Connect** next to `wiseppc` (listed under **Plugin MCP Servers**), then select **Connect and return** on WisePPC's approval page and let the browser open Cursor.
- To use an API key instead, disable the plugin's `wiseppc` server in **Cursor Settings → Tools & MCPs**.
- For current campaign settings, the analysis skill points to `ads_structure`, which holds every Ads level in one place.

## [0.3.0] - 2026-11-22

### Changed

- Sign in with your WisePPC account instead of an API key. If you set an API key in an earlier version, sign in once from Cursor's MCP settings.
- Setup is now: install the plugin, sign in next to `wiseppc` in Cursor's MCP settings, then choose the business on WisePPC's approval page and connect. From the command line: `cursor-agent mcp enable wiseppc`, then `cursor-agent mcp login wiseppc`.
- The plugin no longer asks for an API key. To keep using a key, add your own server entry with the key to `~/.cursor/mcp.json` and disable the plugin's `wiseppc` server; the README shows how. Keys still work for the WisePPC REST API.
- Corrected: seller listing edits and deletes go through `submit_mutation`, one SKU or a batch of up to 100. Only a whole-listing create or replace (`sp.listings.put`) needs the REST listings route.
- The dataset list names `ads_structure`, which holds every Ads level from portfolio to target in one place.

## [0.2.0] - 2026-11-22

### Changed

- The skills, the rule and the README use the current WisePPC tool names. The agent picks an account with the `target` that `get_session_context` returns, and uses `describe`, `get_entities`, `get_report`, `get_history`, `get_marketplace_data` and `run_sql` where it used older names that stop working after 2026.
- Changes are explained as they work today, for an API key and a sign-in (OAuth) connection alike: by default every change waits for a person's approval in WisePPC, and an owner or admin can set an operation to direct or off under **AI Integration → Connections & Permissions**. A sign-in connection is no longer described as read-only.
- A sign-in connection can use the WisePPC REST API (whole listing and catalog records, listing edits) with a short-lived token from `issue_rest_token`; the REST API is no longer described as API-key only.
- Updated facts: deletes always wait for a person's approval; creative asset uploads, Multi-Channel Fulfillment cancels and fixes of failed changes follow the connection's permissions like other changes; personal data (including Multi-Channel Fulfillment orders) is the one opt-in, and raw SQL is an ordinary read.
- A key or sign-in connection belongs to one business, so the setup steps no longer ask the agent to choose a business profile.
- The README's feature list is titled "What's included" instead of naming an old version, and a new section explains how changes follow your connection's permissions.
- The plugin description no longer assumes an API key.
- Sign-in from Cursor is not available yet; use an API key.
- Links point to the new WisePPC MCP pages: [wiseppc.com/mcp](https://wiseppc.com/mcp/) (product, also the plugin homepage) and [wiseppc.com/docs/mcp](https://wiseppc.com/docs/mcp/) (technical overview).
- API keys are created under **AI Integration → Connections & Permissions** in the WisePPC webapp; setup steps now use that name.
- Skills no longer tell the agent to redact customer emails and phone numbers from results: that data is the user's own, and the credential's opt-ins decide what arrives. It stays out of submissions, since approvers may not hold that opt-in.

## [0.1.6] - 2026-10-07

### Changed

- A shorter plugin description that leads with what the plugin does, and updated keywords.

---

## [0.1.5] - 2026-10-07

### Changed

- The plugin icon now has a white background, so it reads clearly on light and dark pages.

---

## [0.1.4] - 2026-10-07

### Changed

- The plugin icon is now square, so it is no longer cropped in the marketplace.
- **Installation:** install from cursor.directory, or from this repository as a local plugin (Teams and Enterprise admins can import it into a team marketplace).
- **Raw SQL:** `run_query` is described as an escape hatch that can be a separate permission on your API key (check `key_grants`); `query` stays the default. The previous wording called it a normal read.
- **Argument names corrected:** `describe_dataset` takes `dataset` (not `dataset_id`); the skills now spell out `query`'s `compare`, `sort`, `offset`, `format` and `round`, and `get_runbook`'s `runbook_id`.
- **Permissions:** the skills and rule now say that grants live on the key (reads per account or seller and marketplace, writes per operation on one profile or marketplace), that a listed tool can still refuse a target, and that an OAuth session is read-only.
- **Catalog health:** the old "not available yet" wording is replaced by what exists today: `get_health_check`, `get_change_history`, and the `listing_item` dataset.
- **Listing and catalog records:** the "full listing/catalog record retrieval is not available" wording is replaced. Whole Amazon-shaped records come from the WisePPC REST API with the same API key; MCP tool results stay size-capped.
- Skill descriptions and the rule were widened so the plugin is picked up for listing edits, create/delete, Multi-Channel Fulfillment and runbook questions.
- README: removed the local developer-testing section.

### Added

- **Dataset guide** in **analyze-amazon-ads**: Ads performance, search terms (`searchterm`, not `ads_olap`), entity settings, change history, impression share, Attribution, Brand Store and brand metrics, and the seller datasets (sales and traffic, economics, transactions, inventory, fees, returns, reimbursements, Brand Analytics), with the main gotchas (search-term metrics are not additive with performance metrics, the newest days are thinner).
- **Seller-side analytics**, **Brand Store**, **A+ content** (`get_aplus_content`) and **Amazon Attribution links** (`get_attribution_link`) coverage, plus `get_health_check`, `get_product_performance`, `get_unadvertised_products` and `get_change_history` pointers.
- **`get_ads_entities`** as the read that pairs with `submit_mutation`: read campaigns, ad groups, ads, targets, portfolios and creative assets as Amazon Ads objects before changing them.
- **Whole records over REST** in **connect-wiseppc** (pointers in **analyze-amazon-ads**): the same API key works on `https://mcp.wiseppc.com/api/v1` for complete Amazon-shaped listing (`/sp/listings/2021-08-01/items/...`) and catalog item (`/sp/catalog/2022-04-01/items/...`) records. API key only, one marketplace per request.
- **Seller listing writes** in **propose-ad-changes**: `sp.listings.put` / `patch` / `delete` through the REST listings route, grants per field group (title, bullet points, description, price, quantity, Item Highlight, other, create, delete), what a PUT requires, delete always reviewed, read the listing first, and a rationale with every change.
- **Ads create and delete** in **propose-ad-changes**: campaigns, ad groups, ads and targets (build order campaign, ad group, ads, targets; deletes cannot be undone) and portfolio create/update.
- **Multi-Channel Fulfillment** in **propose-ad-changes** and **analyze-amazon-ads**: `sp.fulfillment.cancel` is direct only with no review (confirm with the user first); reading MCF orders is a separate sensitive opt-in over REST only.
- **Account types** in **connect-wiseppc**: Seller, Vendor, Agency and KDP; Vendors have no Seller Central, so seller datasets, A+ content, listings and product type definitions do not apply.
- **Runbook list** in **analyze-amazon-ads**: each runbook id with the question it answers, and "read the runbook before writing its queries".
- **Key opt-ins** in **connect-wiseppc**: raw SQL, MCF orders and other principals' queue rows are separate opt-ins, visible in `key_grants`.
- **Other kinds of change** in **propose-ad-changes**: automation rules (`ads.rules.*`), Brand Store page ASIN edits (draft only) and creative asset uploads (always reviewed), where your key is granted them.

---

## [0.1.3] - 2026-10-07

### Changed

- **License:** the plugin is now released under the MIT license: you may use, copy, modify and redistribute it without restriction.

---

## [0.1.2] - 2026-10-07

### Added

- **Fixing a failed change:** a change that failed at Amazon now stays on the WisePPC Queue, with Amazon's own response, until a person dismisses it. The agent reads why it failed (`error_message`, `error_detail.amazon_response`, the `revision_chain`) and, when the row is `can_revise`, submits a corrected change with a new `idempotencyKey` and `metadata.revises` + `metadata.revision_comment` (what changed and why it should work now). A revision is one change of the same kind on the same account and profile, and always waits for a person's approval; it stops on `revision_limit_reached` or `already_revised`. Covered in the `wiseppc-mcp` rule and the **propose-ad-changes** skill (section 6).
- `get_mutations(status="failed")` also lists revisions that were rejected, withdrawn or expired before they were sent; `open_failure: true` marks a failure that still needs attention.

---

## [0.1.1] - 2026-10-06

### Changed

- **Tool names updated** to match the current WisePPC MCP server: `propose_action` is now `submit_mutation`; `list_action_types` is `list_mutation_types`; `list_proposed_actions` / `wait_for_action_updates` are `get_mutations`; `cancel_proposed_action` / `comment_on_action` are `update_mutation`; `list_datasets` is `describe_dataset` with no dataset; `list_business_profiles` is `select_business_profile` with no id.
- **Writes:** each granted write operation is either **gated** (a person approves it in WisePPC) or **direct** (queued and sent without review); the API key's grants decide. `submit_mutation` takes `operation` + `request` + `metadata` (`rationale`, `contextSnapshot`) + `idempotencyKey`. Refusals lead with a reason code; read `key_grants` instead of probing. OAuth sessions on production are read-only today.
- **Session start:** call `get_session_context` once per session (it now includes your grants); refresh preferences with `list_preferences`.
- **Raw SQL:** `run_query` is a normal read covered by Amazon Ads read access; `query` is still preferred.
- Removed references to MCP minting an API key; keys are created in the WisePPC webapp.
- Packaging: the plugin now ships a single rule (`wiseppc-mcp.mdc`); a maintainer-only packaging rule that was included by mistake has been removed.
- `query` guidance corrected: `time_range` is `{start, end}` dates, granularity is `total`/`daily`/`weekly`/`monthly`, `having` filters metrics, default `limit` 100.

---

## [0.1.0] - 2026-09-11

### Summary

Initial production release matching what is live on **production MCP today** (`https://mcp.wiseppc.com/mcp`). This release provides Amazon Ads and seller analytics via MCP, with human-reviewed change proposals.

### Added

- **Authentication:** API key (`wpp_ak_*`) or OAuth to WisePPC user
- **Ads Analytics:** Campaign performance, search terms, products, targeting, placements
- **Seller Analytics:** Sales & traffic, economics, brand analytics (requires Seller Central connection)
- **Catalog Health:** Listing health checks via `get_health_check` (MCP-based)
- **Structured Query:** `list_datasets`, `describe_dataset`, `query` (preferred over raw SQL)
- **Propose Actions:** Human-reviewed change queue via `propose_action`
- **Preferences & Runbooks:** Persistent account settings and guided workflows
- **Session Context:** `get_session_context` for full account snapshot
- **Skills:**
  - `connect-wiseppc` — Setup, authentication, profile listing
  - `analyze-amazon-ads` — Query ads/seller data via structured datasets
  - `propose-ad-changes` — Submit changes for human approval
- **Rules:** Enforcing security, structured query preference, and human review

### Production Scope

- **Audience:** WisePPC account holders
  - **Seller:** Ads + Seller Central datasets (full analytics)
  - **KDP:** Ads only (no Seller Central)
- **MCP Server:** `https://mcp.wiseppc.com/mcp` (production only)

### Explicitly NOT Included

- **Full Amazon listing/catalog record retrieval:** Complete Amazon-shaped payloads for listing and catalog items
  - v0.1.0 includes catalog **health** insights via `get_health_check` only
  - **Planned for a future release**

### Notes

- **Catalog health:** `get_health_check` provides MCP-based listing health data. This is **not** the same as full listing/catalog record management (planned for a future release).

---

## [Unreleased] - Future Releases

### Planned

- **Full Amazon listing/catalog record retrieval:** Complete Amazon-shaped payloads for listing and catalog items
  - Retrieve full listing details (title, description, attributes, images)
  - Access complete catalog item metadata and relationships
  - Navigate variation families and parent-child structures
- **New skill:** `retrieve-amazon-records` for full record access
- **Enhanced catalog workflows:** Deeper record inspection and cross-referencing

---

## Version History

- **v0.1.1** (2026-10-06) — Skills, rules and README aligned with the current MCP tools and write model
- **v0.1.0** (2026-09-11) — Initial production release (ads + seller analytics via MCP)
