---
name: propose-ad-changes
description: Submit Amazon changes through WisePPC. Use when analysis points at pause/enable, bid, budget, negatives, bid strategy, creating or deleting campaigns, ad groups, ads, targets or portfolios, automation rules, Brand Store page ASINs, creative assets, seller listing edits (title, bullets, price, quantity, create, delete) or a Multi-Channel Fulfillment cancel, or when a submitted change failed and needs fixing. Whether a change waits for a person's approval or is sent directly is decided by the credential's grants.
---

# Propose Ad Changes

## When to use

- User asks what to change, or analysis points at a concrete ads action
- Pause/enable campaigns, ad groups, keywords, or product targets
- Adjust bids (campaign, ad group, keyword, product target, placement)
- Change budgets (daily campaign budgets)
- Add negative keywords or negative product targets
- Update bid strategies or placement adjustments
- Create, update, attach or detach Ads automation rules (`ads.rules.*`)
- Create or delete campaigns, ad groups, ads, targets, or create and update portfolios
- Edit the ASINs on a Brand Store page draft, or upload a creative asset, where your key is granted those
- Edit a seller listing (title, bullets, description, price, quantity), create or delete a listing
- Cancel a Multi-Channel Fulfillment order
- Fix a change that failed (see "Fix a Failed Change")

## How writes work

Each granted write operation is either:

- **gated** — a person approves it in the WisePPC webapp; nothing is sent to Amazon until then, or
- **direct** — queued and sent to Amazon **without review**.

**The credential's grant decides which.** `submit_mutation` cannot ask, and a request that names a `mode` is refused (`mode_not_accepted`). If operations in one call resolve to different modes, the call answers `split_request` — send them in separate calls. Approving is human-only; you cannot approve.

An **OAuth session is read-only**: `submit_mutation` is not available. Changes need an API key with write grants. In an OAuth session, describe the proposed change in words instead.

**Treat a direct grant as live:** double-check the numbers before submitting, because no one reviews it first.

## Critical Rules

1. **Read `key_grants` first** (from `get_session_context`) to see which write operations are granted for this profile or marketplace. Do not discover your limits by being refused. Ads writes are listed per profile (`write_for_this_profile`), Selling Partner writes per seller and marketplace (`write_for_this_marketplace`); an empty list for one says nothing about the other.

2. **Read the entity first** with `get_ads_entities` (`entity`: campaigns, ad_groups, ads, targets, portfolios; for a Brand Store page, `get_brand_store_page`; for creative assets, `entity: "assets"`). It returns Amazon Ads API objects from WisePPC's synced copy, the shape `submit_mutation` takes (drop the `wiseppc*` keys, such as `wiseppcRules`, before sending an entity back), so you change what exists today and not what you remember. All states are returned unless you pass `states`.

3. **Call `list_mutation_types`** for the live operations and request schemas. Pass `operation` (e.g., `ads.campaigns.update`) to inspect one operation's `request_schema` and examples. Without `operation` it also returns guides for rules, failures, Brand Store and assets.

4. **Ads changes go only through `submit_mutation`.** Seller listing writes go through the WisePPC REST listings route (see "Seller Listing Writes"), which queues them the same way. Never call Amazon's write APIs directly.

5. **Required fields:**
   - `operation` — from `list_mutation_types` (e.g., `ads.campaigns.update`, `ads.targets.update`)
   - `request` — the operation's request body per its schema (profile, entities, or entity ids)
   - `idempotencyKey` — a unique string per intended change (reuse it only to retry the same submission)
   - `metadata.rationale` — 1–3 sentences explaining **why** this change makes sense (reference specific metrics)
   - `metadata.contextSnapshot` — the numbers you used (e.g., `{ "acos": 0.45, "target_acos": 0.30, "spend_last_30d": 1200 }`)

   The older `actionType` / `entityType` / `entityId` / `params` form is still accepted; `list_mutation_types` returns its `action_types` schemas.

6. **Refusals lead with a reason code and a sentence.** None are retryable and none are your error: name the missing permission and let the user decide whether to widen the credential. Do not retry or work around it.

7. **After submitting**, track it with `get_mutations`:
   - `mutationId` — one change in full (status, phase, reason, reviewer thread)
   - no id — list yours (pass `mineOnly=false` for the whole queue)
   - `waitSeconds` (with `profileId`) — long-poll until one changes; `{event: 'timeout'}` means call again

8. **If a reviewer requests revision or declines** (gated changes):
   - `update_mutation` with `action=cancel` on the pending row (do not leave stale submissions)
   - Submit a replacement with an updated rationale if appropriate
   - `update_mutation` with `action=comment` on the new mutation id to link the conversation
   - **Do not argue a decline.** Reviewers have context you may not.

9. **Never put secrets in submissions:**
   - Do not include API keys, tokens, or `wpp_ak_*` strings in `rationale`, `contextSnapshot`, or comments

## Submission Workflow

### 1. Analyze First

Before submitting, ensure you have:

- Called `get_session_context` for the `profileId` (once per session) and read `key_grants`
- Queried relevant datasets to understand current performance
- Identified a specific, measurable opportunity or issue

### 2. Read the Entity, Then Check Operations

```
get_ads_entities with profileId, entity, ids (or campaign_ids / ad_group_ids)
list_mutation_types
```

`get_ads_entities` shows what exists today, as Amazon Ads API objects. `list_mutation_types` returns the live list of write operations.
Pass `operation` to `list_mutation_types` to get one operation's request schema and examples.

### 3. Build the Submission

```
submit_mutation with:
  - operation (from list_mutation_types)
  - request (per that operation's schema)
  - idempotencyKey (unique per change)
  - metadata: { rationale, contextSnapshot }
```

**Example `rationale`:**
> "Campaign 'Brand Defense' has ACOS 0.52 vs. target 0.30, with $1,200 spend in the last 30 days. Pausing to stop bleed while we investigate targeting issues."

**Example `contextSnapshot`:**
```json
{
  "campaign_name": "Brand Defense",
  "acos": 0.52,
  "target_acos": 0.30,
  "spend_last_30d": 1200,
  "clicks": 450,
  "orders": 12
}
```

### 4. Track the Result

```
get_mutations with mutationId (or waitSeconds to long-poll)
```

Possible statuses:

- `pending` — awaiting review (gated) or dispatch
- `approved` — a person approved it, queued to execute
- `declined` — a person rejected it, see reviewer comments
- `needs_revision` — shown on a row when a reviewer asks for changes; read their comment (it is a flag on the row, not a `status` filter value)
- `cancelled` — retracted before execution
- `executing` / `executed` — being applied / applied to Amazon
- `failed` — execution error, see the reason and details; revise it if the row says `can_revise` (see "Fix a Failed Change")

Entries are immutable; follow replacement links rather than expecting a row to change.

### 5. Handle Feedback

If a reviewer declines or requests revision:

1. **Cancel the pending submission:**
   ```
   update_mutation with mutationId, action=cancel, expectedVersion
   ```
   Cancel works on pending / needs-revision rows only. Once approved, declined, or executed, submit a compensating change instead.

2. **Submit a replacement** (if appropriate):
   ```
   submit_mutation with updated rationale and request
   ```

3. **Link the conversation:**
   ```
   update_mutation with the new mutationId, action=comment, body referencing the old submission
   ```

Do not argue with reviewers. They have business context, risk tolerance, and strategic goals you may not be aware of.

### 6. Fix a Failed Change

When a change failed (for example Amazon rejected it) and the row has `can_revise: true`, revise it instead of submitting an unrelated new change. The failure stays as history, linked to the fix, so the same mistake is not repeated.

1. **Read why it failed:** `get_mutations` with the `mutationId`. Look at `error_message`, `error_detail.amazon_response` and, for a partly applied change, which items already went through. `get_mutations` with `status: "failed"` lists failures; `open_failure` marks the ones still open.

2. **Submit the revision:**
   ```
   submit_mutation with:
     - operation and the corrected request
     - a new idempotencyKey
     - metadata: { rationale, revises: <failed mutationId>, revision_comment: <what changed and why it should work now> }
   ```
   The failed row's `revise_with` shows the shape. A revision holds one change for the same operation family, account and profile, and always waits for a person's approval.

3. **Follow the chain:** reading one mutation by id returns `revision_chain` (every earlier attempt with its error). A revision that was rejected, withdrawn or expired before it was sent can itself be revised. Cancelling a revision does not clear the failure; only a person can dismiss it. On `revision_limit_reached` or `already_revised`, stop and tell the user.

## Other Kinds of Change

All go through the same `submit_mutation` flow and the same grants. Ask `list_mutation_types` for the exact request shape; its guides explain each.

- **Automation rules** (`ads.rules.create`, `update`, `set_state`, `attach`, `detach`, `delete`): the request names a rule `family`, the `rule`, and `entity_ids` as strings. Creating most rule types, or editing a rule attached to several campaigns, always needs review even with a direct grant (`requires_review`); the response shows a `rule_impact` preview. A bid that a rule controls is refused with `bid_controlled_by_rule`: change the rule or detach it instead of the bid.
- **Brand Store page ASINs** (`ads.brand_store.update_page_asins`): read the page with `get_brand_store_page`, copy each edit's `path` and `widgetTag` from `wiseppc.asinLists`, and send the **full** new ASIN list for that field (replace, not patch), one page per change. It edits the page **draft**; nothing goes live to shoppers from here. If the page changed since you read it (`page_changed`, `widget_moved`), re-read and resubmit.
- **Creative assets** (`ads.assets.create`, `ads.assets.version`): read the library with `get_ads_entities` and `entity: "assets"` first; an existing asset is a `version`, not a new `create`. These always wait for a person's approval. An asset is visible to every profile of the advertiser and cannot be edited or deleted afterwards, so only upload what the user asked for. The source is a plain public https URL, or a small JPEG/PNG as base64 (never a presigned URL).
- **Create and delete** (`ads.campaigns`, `ads.ad_groups`, `ads.ads`, `ads.targets` with `.create` / `.delete`; `ads.portfolios.create` and `.update`): see "Creating and Deleting Ads Entities".
- **Seller listings** (`sp.listings.put`, `patch`, `delete`) and **Multi-Channel Fulfillment** (`sp.fulfillment.cancel`): see the two sections below. They need write grants per seller and marketplace.

## Creating and Deleting Ads Entities

Creates and deletes use the same `submit_mutation` flow and grants as updates: `ads.campaigns.create`, `ads.ad_groups.create`, `ads.ads.create`, `ads.targets.create`, the matching `.delete` operations, and `ads.portfolios.create` / `ads.portfolios.update` (portfolios have no delete). Call `list_mutation_types` with `operation` for the exact request schema.

- **Build order for a new campaign:** campaign, then ad group, then ads, then targets. Create the parent first and use the id Amazon returns for the next step; check the result with `get_mutations` between steps. Sponsored Brands refuses targets on an ad group that has no ad yet (`ad_group_has_no_ads`).
- **Deletes cannot be undone.** A delete archives the entity. To stop spend you can reverse, pause it instead (`.update` with `state: PAUSED`). Only delete when the user asked for it, and name what will be archived in the `rationale`.
- A create is undone by archiving what it made, so double-check names, budgets and bids before you submit.

## Seller Listing Writes

Listing edits are `sp.listings.put`, `sp.listings.patch` and `sp.listings.delete`. They appear in `list_mutation_types` and on queue rows, but `submit_mutation` refuses them (`use_the_listings_route`). Submit them through the WisePPC REST API with the same API key (base `https://mcp.wiseppc.com/api/v1`; see **connect-wiseppc**). The write is queued exactly like any other: the key's grants decide whether it is **gated** or **direct**; the route answers **202** with a mutation id; track it with `get_mutations`.

- **Read the listing first**: the `listing_item` dataset, or `GET /sp/listings/2021-08-01/items/{sellerId}/{sku}` for the whole record. Change what exists today.
- **Routes** (all on `/sp/listings/2021-08-01/items/{sellerId}/{sku}`, one `marketplaceIds` per request): `PATCH` changes named attributes (`{productType, patches: [{op, path, value}]}`); `PUT` creates a SKU or replaces a whole listing (`{productType, attributes, requirements?}`); `DELETE` removes it. `POST /sp/listings/2021-08-01/items/{sellerId}` queues up to 100 SKUs as one batch.
- **Grants are per field group**, on the seller and marketplace: title, features (bullet points), description, price, quantity, highlight (Item Highlight), other (every attribute not listed), create (new SKU) and delete. A PATCH needs the group of each attribute path it touches (`/attributes/item_name` is title, `/attributes/purchasable_offer` is price, anything unlisted is other).
- **A PUT on an existing SKU replaces everything**, so it needs *every* update group, not one. On a SKU that does not exist it is a create and needs the create group. Prefer PATCH for a targeted edit.
- **Delete is gated only** and cannot be undone: it always waits for a person, however the rest of the key is granted.
- Add `mode=VALIDATION_PREVIEW` to a PUT or PATCH to run Amazon's own dry run: nothing is queued, and it is authorized like the real write.
- Send an `Idempotency-Key` header (unique per intended change) and a `x-wiseppc-reason` header: 1-3 sentences on **why**, with the numbers, for the person reviewing it. No secrets in it.
- A mix of gated and direct items in one batch is refused (`split_request`).

## Multi-Channel Fulfillment Cancel

`sp.fulfillment.cancel` (with `submit_mutation`: `seller_id`, `region` of `na`, `eu` or `fe`, and `order_id`; or `PUT /sp/fulfillment/outbound/2026-07-04/orders/{orderId}/cancel` on the REST API) cancels a Multi-Channel Fulfillment order.

- **It is direct only: there is no review step**, it reaches Amazon immediately, and it stops a real shipment. It cannot be undone, and Amazon only accepts it before the order ships.
- **Confirm with the user first**, naming the order, before you submit.
- The key needs the `cancel` grant for that seller; reading orders does not imply it. Reading MCF orders is a separate sensitive opt-in (buyer details), REST only.
- On a timeout, do not retry blindly: the cancel may have gone through. Check the order first.

## User Preferences

`get_session_context` loads any saved preferences (target ACOS, settling period, bid caps, etc.). Apply these **silently** when building submissions; refresh them with `list_preferences`. Do not announce preferences unless the user asks.

If a reviewer's feedback suggests a new preference (e.g., "Never change campaigns modified within the last 7 days"), save it:

```
set_preference with key, value, category
```

Example:
- `key: "settling_period_days"`
- `value: "7"`
- `category: "defaults"`

To remove one, call `set_preference` with `value: null`.

## Runbooks

`get_session_context` also lists the runbook catalog (guided workflows for common scenarios); read one with `get_runbook`. Reference these for well-known patterns (e.g., "wasted spend on auto campaigns", "underperforming exact keywords").

## Security Reminders

- **Never** include API keys, tokens, or `wpp_ak_*` secrets in `rationale`, `contextSnapshot`, or comments
- If a submission involves sensitive data (customer emails, phone numbers), redact or summarize
- Submissions are logged and visible to the account owner in the WisePPC webapp

## Production Environment

This plugin connects to **production MCP:** `https://mcp.wiseppc.com/mcp`
