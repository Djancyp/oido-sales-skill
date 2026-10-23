# db usage

How the skill reads and writes the sales DB. Table definitions are in `db-schema.md`; this file says when each table is used and how.

Connection details come from the user at setup (`setup.md`). Never print the connection string or any token in chat.

## Rules

- **Scope by org.** Every read and write filters on `org_id`. Keep the active org's `id` from setup and use it everywhere.
- **Read before you write.** Check for a duplicate (`dedupe_key` for prospects, `content_hash` for messages) and for `do_not_contact` before any insert or draft.
- **Archive, never delete.** Disqualified prospects go to `disqualified`; unwanted drafts go to `archived`.
- **One message, one row.** Review and approval update the same row by `id`.
- **Parameterized queries only.** Pass values as parameters.
- **Batch multi-row writes** (a list of prospects, a set of drafts) in one transaction.
- **Use the DB role the user set up.** It needs insert and update on the `oido_sales_` tables; no DDL in day-to-day use.

## Running SQL through a tool

The DB is Postgres. Some SQL tools (for example the Supabase `sp_execute_sql` MCP tool) accept one statement per call and may reject any `;`, even inside a quoted value.

- **One statement per call**, no trailing `;`.
- **No `;` inside text values.** Write `Raised 15M EUR, hired VP Engineering`, not `Raised 15M EUR; hired VP Engineering`. Applies to `fit_reason`, `trigger_event`, `body` and every other text column.
- **Escape quotes** by doubling them (`O''Brien`).
- **After one failure, compare the failing statement with the last one that worked** and change only the difference. Do not retry the same statement, and do not fall back to a "Test" row; it pollutes the org's data. If a test row was inserted, archive it (`status = 'disqualified'`).
- **Write access may be off.** Such tools can be read-only by default (Supabase: `SUPABASE_ALLOW_*`). If an insert is refused for permissions, tell the user which setting to enable.
- **Schema setup cannot go through a one-statement tool.** `db-schema.md` has function bodies and `DO` blocks that contain `;`. Ask the user to run it in their SQL editor or `psql`, then verify with the check query in `setup.md`.

## Action → table

| Action | Table | Write |
|---|---|---|
| Onboard an org | `oido_sales_orgs` | insert, `ON CONFLICT (slug) DO NOTHING` |
| Save or update the ICP | `oido_sales_org_icp` | upsert on `org_id` |
| Save a prospect | `oido_sales_prospects` | insert, `status = 'new'`, after the dedupe check |
| Mark researched / drafted / approved | `oido_sales_prospects` | update `status` |
| Opt-out or "no" from a prospect | `oido_sales_prospects` | `do_not_contact = true`, `status = 'disqualified'` |
| Draft a message | `oido_sales_messages` | insert, `status = 'draft'`, `direction = 'outbound'` |
| Submit for review | `oido_sales_messages` | `status = 'pending_review'` |
| Approve | `oido_sales_messages` | `status = 'approved'` |
| Mark sent | `oido_sales_messages` | `status = 'sent'`, `sent_at`, `external_ref` if any; set prospect `contacted` |
| Send failed | `oido_sales_messages` | `status = 'failed'`, `failure_reason` |
| Log a reply | `oido_sales_messages` | insert, `direction = 'inbound'`, `status = 'received'`; set prospect `replied` |
| Archive a draft | `oido_sales_messages` | `status = 'archived'` |

### Saving a prospect

Fill every column the facts support. Leave a column NULL rather than guess.

- `org_id`: the active org.
- `company_name`, `company_domain`: from the source.
- `contact_name`, `contact_title`, `contact_url`: only if a source names them.
- `contact_email`, `contact_phone`: only if published on a page you opened. `contact_source_url` is then required (the database enforces it). Never guess or build an address from a name pattern.
- `persona_role`: `buyer`, `champion`, `blocker`, or `other`, per the ICP.
- `fit_reason`: which ICP rules match, in plain words.
- `trigger_event`: the public event that makes it timely, if any.
- `signal_type`: the ICP trigger's event name it matched.
- `signal_date`: the date shown on the source page, if any.
- `source_url`: the page that supports the facts. Required.
- `dedupe_key`: `lower(company_domain) || '|' || lower(coalesce(contact_name, ''))`.

Before insert, check:

```sql
SELECT id FROM oido_sales_prospects
WHERE org_id = $1 AND dedupe_key = $2;
```

### Saving a message

- `prospect_id`: must belong to the same `org_id`. Check it first.
- `body`: the text to send, no placeholders left unfilled.
- `content_hash`: sha256 hex of `subject || body`.
- `step`: 1 for the first touch, 2+ for follow-ups.

## Pipeline queries

Counts per status for the active org:

```sql
SELECT status, count(*) AS prospects
FROM oido_sales_prospects
WHERE org_id = $1
GROUP BY status
ORDER BY prospects DESC;
```

Prospects with no reply after their last sent message:

```sql
SELECT p.id, p.company_name, m.sent_at
FROM oido_sales_prospects p
JOIN oido_sales_messages m ON m.prospect_id = p.id AND m.org_id = p.org_id
WHERE p.org_id = $1
  AND p.status = 'contacted'
  AND m.direction = 'outbound' AND m.status = 'sent'
  AND m.sent_at < now() - interval '7 days'
ORDER BY m.sent_at;
```

The 7-day window is a default. Change it only if the user asks.
