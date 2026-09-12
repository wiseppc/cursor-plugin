---
name: propose-ad-changes
description: Propose Amazon Ads changes through WisePPC for human approval. Use when analysis points at pause/enable, bid, budget, negatives, or bid strategy. Nothing hits Amazon until approved in the webapp.
---

# Propose Ad Changes

## When to use

- User asks what to change, or analysis points at a concrete ads action
- Pause/enable campaigns, ad groups, keywords, or product targets
- Adjust bids (campaign, ad group, keyword, product target, placement)
- Change budgets (daily campaign budgets)
- Add negative keywords or negative product targets
- Update bid strategies or placement adjustments

## Critical Rules

1. **Call `list_action_types` first** for the live registry and params schema. This returns the available action types, required parameters, and validation rules.

2. **Use ONLY `propose_action`.** Do not call Amazon Ads write APIs directly. All changes go through the human-review queue.

3. **Required fields:**
   - `action_type` — from `list_action_types` (e.g., `pause_campaign`, `adjust_bid`, `add_negative_keyword`)
   - `rationale` — 1–3 sentences explaining **why** this change makes sense (reference specific metrics)
   - `contextSnapshot` — the numbers you used to make this decision (e.g., `{ "acos": 0.45, "target_acos": 0.30, "spend_last_30d": 1200 }`)
   - `params` — action-specific parameters (e.g., `campaign_id`, `new_bid`, `keyword_text`)

4. **Nothing hits Amazon until a human approves** in the WisePPC webapp. Proposals are queued for review.

5. **After proposing:**
   - Use `wait_for_action_updates` to poll for review status changes (approved, declined, revision_requested)
   - Or use `list_proposed_actions` to see all pending/recent proposals

6. **If a reviewer requests revision or declines:**
   - `cancel_proposed_action` the pending row (do not leave stale proposals)
   - Propose a replacement with updated rationale if appropriate
   - `comment_on_action` with the new action id to link the conversation
   - **Do not argue a decline.** Reviewers have context you may not.

7. **Never put secrets in proposals:**
   - Do not include API keys, tokens, or `wpp_ak_*` strings in `rationale`, `contextSnapshot`, or `comment_on_action`

## Proposal Workflow

### 1. Analyze First

Before proposing, ensure you have:

- Called `get_session_context` for the `profileId`
- Queried relevant datasets to understand current performance
- Identified a specific, measurable opportunity or issue

### 2. Check Action Types

```
list_action_types
```

This returns the live registry of available action types with params schemas. Example action types:

- `pause_campaign` / `enable_campaign`
- `adjust_campaign_budget`
- `adjust_keyword_bid`
- `add_negative_keyword`
- `adjust_placement_bid`
- `change_bid_strategy`

### 3. Build the Proposal

```
propose_action with:
  - action_type (from list_action_types)
  - rationale (1-3 sentences, reference specific metrics)
  - contextSnapshot (JSON object with the numbers you used)
  - params (action-specific, per the schema from list_action_types)
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

### 4. Wait for Review

After proposing:

```
wait_for_action_updates
```

This polls the review queue for status changes. Possible statuses:

- `pending` — awaiting review
- `approved` — human approved, will execute on next sync
- `declined` — human rejected, see reviewer comments
- `revision_requested` — human wants changes, see comments
- `executed` — change applied to Amazon
- `failed` — execution error, see error details

### 5. Handle Feedback

If a reviewer declines or requests revision:

1. **Cancel the pending proposal:**
   ```
   cancel_proposed_action with action_id
   ```

2. **Propose a replacement** (if appropriate):
   ```
   propose_action with updated rationale and params
   ```

3. **Link the conversation:**
   ```
   comment_on_action with new_action_id, referencing the old proposal
   ```

Do not argue with reviewers. They have business context, risk tolerance, and strategic goals you may not be aware of.

## User Preferences

`get_session_context` loads any saved preferences (target ACOS, settling period, bid caps, etc.). Apply these **silently** when building proposals. Do not announce preferences unless the user asks.

If a reviewer's feedback suggests a new preference (e.g., "Never propose changes to campaigns modified within the last 7 days"), save it:

```
set_preference with key, value, category
```

Example:
- `key: "settling_period_days"`
- `value: "7"`
- `category: "defaults"`

## Runbooks

`get_session_context` also loads runbooks (guided workflows for common scenarios). Reference these when proposing changes for well-known patterns (e.g., "wasted spend on auto campaigns", "underperforming exact keywords").

## Security Reminders

- **Never** include API keys, tokens, or `wpp_ak_*` secrets in `rationale`, `contextSnapshot`, or comments
- If a proposal involves sensitive data (customer emails, phone numbers), redact or summarize
- All proposals are logged and visible to the account owner in the WisePPC webapp

## Production Environment

This plugin connects to **production MCP:** `https://mcp.wiseppc.com/mcp`
