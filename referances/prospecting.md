# prospecting — find, qualify, draft, review

Trigger: the gate in `SKILL.md` is passed (org onboarded, ICP saved) and the user asks for prospects, leads, outreach or pipeline help.

## Step 1 — find

1. Read the org's ICP from `oido_sales_org_icp` and its `offer` from `oido_sales_orgs`. Both filter by the active `org_id`.
2. Search public sources for companies and people that match the ICP triggers (WebSearch). Run related searches together.
3. Keep only results that are public, match an ICP rule, and have a source URL.
4. Ask how many to find if the user has not said. Default: 10 per batch.

## Step 2 — qualify

For each candidate:

- Check the anti-ICP. Drop anything that matches.
- Check for an existing row (`dedupe_key`) and for `do_not_contact`. Skip both.
- Write `fit_reason` in one or two sentences tied to the ICP, and `trigger_event` with the public event, if any.
- Every fact must map to `source_url`. If it does not, leave the fact out.

Save the batch in one transaction. Status: `new`, then `researched` once qualified.

## Step 3 — draft

For each prospect the user wants to reach:

- Write a short first message for the org's `offer`. Cite the trigger or a sourced detail. No invented claims, metrics or praise.
- Include a clear, single ask and an easy "no" (for example: "Happy to close the loop if this isn't a priority").
- Keep the body short enough to read on a phone. Plain text, no tracking links unless the user supplies them.
- Save as `oido_sales_messages` with `status = 'draft'`, `step = 1`, and `content_hash`. Set the prospect to `drafted`.

Show the drafts to the user with the prospect, the fit reason and the source.

## Step 4 — review and approve

- Edits update the same row. Set `pending_review` when the user sends a draft back for changes.
- Set `approved` only when the user approves that exact message in this conversation. Each message needs its own approval.
- Set the prospect to `approved` once its first message is approved.

## Step 5 — send

- Sending is outward-facing and hard to reverse. Send only approved messages, one at a time, after the user confirms.
- Use the channel the user has connected. If none is connected, the user sends the message themselves and tells you; then record it.
- On success: message `sent` with `sent_at` (and `external_ref` if the channel returns one); prospect `contacted`.
- On failure: message `failed` with `failure_reason`. Do not retry without asking.

## Step 6 — replies and follow-up

- A reply from the prospect is logged as an inbound message (`status = 'received'`) and sets the prospect to `replied`.
- If the reply says no or unsubscribe, set `do_not_contact = true` and `disqualified`. Stop all outreach to that prospect.
- Follow-ups (`step` 2+) are drafted only for prospects with no reply after the window in `db-usage.md`, and only after the user approves the timing and text.

## Pipeline report

On request, show counts per status for the active org (`db-usage.md`). Report what the numbers say; suggest changes, but let the user decide.
