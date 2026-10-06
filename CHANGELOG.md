# Changelog

All notable changes to the WisePPC Cursor plugin will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
