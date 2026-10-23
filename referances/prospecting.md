# prospecting — draft, review, send, follow up

Trigger: the gate in `SKILL.md` is passed (org onboarded, ICP saved) and the user asks for outreach, follow-ups or pipeline help.

Finding and verifying leads is `lead-finding.md`. Start there if there are no `new` prospects yet. Verified leads arrive here with status `new`; set `researched` once the user has seen them.

## Step 1 — draft

For each prospect the user wants to reach:

- Write a short first message for the org's `offer`. Cite the trigger or a sourced detail. No invented claims, metrics or praise.
- Include a clear, single ask and an easy "no" (for example: "Happy to close the loop if this isn't a priority").
- Keep the body short enough to read on a phone. Plain text, no tracking links unless the user supplies them.
- Save as `oido_sales_messages` with `status = 'draft'`, `step = 1`, and `content_hash`. Set the prospect to `drafted`.

Show the drafts to the user with the prospect, the fit reason and the source.

## Step 2 — review and approve

- Edits update the same row. Set `pending_review` when the user sends a draft back for changes.
- Set `approved` only when the user approves that exact message in this conversation. Each message needs its own approval.
- Set the prospect to `approved` once its first message is approved.

## Step 3 — send

- Sending is outward-facing and hard to reverse. Send only approved messages, one at a time, after the user confirms.
- Use the channel the user has connected. If none is connected, the user sends the message themselves and tells you; then record it.
- On success: message `sent` with `sent_at` (and `external_ref` if the channel returns one); prospect `contacted`.
- On failure: message `failed` with `failure_reason`. Do not retry without asking.

## Step 4 — replies and follow-up

- A reply from the prospect is logged as an inbound message (`status = 'received'`) and sets the prospect to `replied`.
- If the reply says no or unsubscribe, set `do_not_contact = true` and `disqualified`. Stop all outreach to that prospect.
- Follow-ups (`step` 2+) are drafted only for prospects with no reply after the window in `db-usage.md`, and only after the user approves the timing and text.

## Pipeline report

On request, show counts per status for the active org (`db-usage.md`). Report what the numbers say; suggest changes, but let the user decide.
