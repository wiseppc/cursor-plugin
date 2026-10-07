---
name: propose-ad-changes
description: Submit Amazon Ads changes through WisePPC. Use when analysis points at pause/enable, bid, budget, negatives, or bid strategy. Whether a change waits for a person's approval or is sent directly is decided by the credential's grants.
---

# Propose Ad Changes

## When to use

- User asks what to change, or analysis points at a concrete ads action
- Pause/enable campaigns, ad groups, keywords, or product targets
- Adjust bids (campaign, ad group, keyword, product target, placement)
- Change budgets (daily campaign budgets)
- Add negative keywords or negative product targets
- Update bid strategies or placement adjustments

## How writes work

Each granted write operation is either:

- **gated** — a person approves it in the WisePPC webapp; nothing is sent to Amazon until then, or
- **direct** — queued and sent to Amazon **without review**.

**The credential's grant decides which.** `submit_mutation` cannot ask, and a request that names a `mode` is refused (`mode_not_accepted`). If operations in one call resolve to different modes, the call answers `split_request` — send them in separate calls. Approving is human-only; you cannot approve.

An **OAuth session on the production server is read-only today**: `submit_mutation` is not available. Changes need an API key with write grants. In an OAuth session, describe the proposed change in words instead.

**Treat a direct grant as live:** double-check the numbers before submitting, because no one reviews it first.

## Critical Rules

1. **Read `key_grants` first** (from `get_session_context`) to see which write operations are granted for this profile or marketplace. Do not discover your limits by being refused. Ads writes are listed per profile (`write_for_this_profile`), Selling Partner writes per seller and marketplace (`write_for_this_marketplace`); an empty list for one says nothing about the other.

2. **Call `list_mutation_types` first** for the live operations and request schemas. Pass `operation` (e.g., `ads.campaigns.update`) to inspect one operation's `request_schema` and examples.

3. **Use ONLY `submit_mutation`.** Do not call Amazon Ads write APIs directly.

4. **Required fields:**
   - `operation` — from `list_mutation_types` (e.g., `ads.campaigns.update`)
   - `request` — the operation's request body per its schema (profile, entities, or entity ids)
   - `idempotencyKey` — a unique string per intended change (reuse it only to retry the same submission)
   - `metadata.rationale` — 1–3 sentences explaining **why** this change makes sense (reference specific metrics)
   - `metadata.contextSnapshot` — the numbers you used (e.g., `{ "acos": 0.45, "target_acos": 0.30, "spend_last_30d": 1200 }`)

   The older `actionType` / `entityType` / `entityId` / `params` form is still accepted; `list_mutation_types` returns its `action_types` schemas.

5. **Refusals lead with a reason code and a sentence.** None are retryable and none are your error: name the missing permission and let the user decide whether to widen the credential. Do not retry or work around it.

6. **After submitting**, track it with `get_mutations`:
   - `mutationId` — one change in full (status, phase, reason, reviewer thread)
   - no id — list yours (pass `mineOnly=false` for the whole queue)
   - `waitSeconds` (with `profileId`) — long-poll until one changes; `{event: 'timeout'}` means call again

7. **If a reviewer requests revision or declines** (gated changes):
   - `update_mutation` with `action=cancel` on the pending row (do not leave stale submissions)
   - Submit a replacement with an updated rationale if appropriate
   - `update_mutation` with `action=comment` on the new mutation id to link the conversation
   - **Do not argue a decline.** Reviewers have context you may not.

8. **Never put secrets in submissions:**
   - Do not include API keys, tokens, or `wpp_ak_*` strings in `rationale`, `contextSnapshot`, or comments

## Submission Workflow

### 1. Analyze First

Before submitting, ensure you have:

- Called `get_session_context` for the `profileId` (once per session) and read `key_grants`
- Queried relevant datasets to understand current performance
- Identified a specific, measurable opportunity or issue

### 2. Check Operations

```
list_mutation_types
```

This returns the live list of write operations. Pass `operation` to get one operation's request schema and examples.

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
- `needs_revision` — a reviewer wants changes, see comments
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
