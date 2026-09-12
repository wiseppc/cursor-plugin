# Changelog

All notable changes to the WisePPC Cursor plugin will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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

- **v0.1.0** (2026-09-11) — Initial production release (ads + seller analytics via MCP)
