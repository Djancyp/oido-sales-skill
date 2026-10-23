# oido-sales-skill

An outbound sales skill for any organization. It onboards an org, defines its ideal customer profile (ICP), finds matching prospects, drafts outreach for approval, and tracks replies and pipeline in a shared database, with every row scoped to one org.

## What it does

1. **Onboards an org**: name, domain, and what it sells.
2. **Defines the ICP**: industries, company size, stages, geographies, personas (buyer, champion, blocker), buying triggers, and an anti-ICP.
3. **Finds prospects** from public sources, each with a fit reason and a source URL.
4. **Drafts first messages and follow-ups** that cite a real trigger.
5. **Waits for approval.** Nothing is sent until the user approves that exact message.
6. **Tracks the pipeline**: prospect status, sent messages, replies, and opt-outs.

## Layout

| File | Purpose |
|---|---|
| `SKILL.md` | Entry point: rules and the step-by-step flow |
| `referances/setup.md` | Checks the DB, creates tables if needed, onboards the org |
| `referances/db-schema.md` | Postgres DDL for the four tables |
| `referances/db-usage.md` | Which action writes to which table, plus query patterns |
| `referances/icp-research.md` | Defines and saves the org's ICP |
| `referances/prospecting.md` | Find, qualify, draft, approve, send, follow up |

## Data model

All tables are prefixed `oido_sales_`.

| Table | Holds |
|---|---|
| `oido_sales_orgs` | One row per organization |
| `oido_sales_org_icp` | The org's ICP (one row per org) |
| `oido_sales_prospects` | Companies and contacts, status, source, opt-out flag |
| `oido_sales_messages` | Outbound drafts and sends, plus inbound replies |

Every table except `oido_sales_orgs` carries `org_id`. Every query filters by it.

## Setup

The skill needs a Postgres database. It is not bundled; bring your own (any provider).

The first time the skill is used, it:

1. Checks whether the four tables exist.
2. Asks before creating anything.
3. Applies `referances/db-schema.md` (idempotent; safe to re-run).
4. Onboards the organization.

Connection details are given by the user at setup and are never written into these files or printed in chat.

## Rules

- **Scope by org.** Reads and writes filter on `org_id`.
- **No invented facts.** Every prospect fact needs a `source_url`.
- **Approval before sending.** Each outbound message is approved individually.
- **Respect opt-outs.** `do_not_contact` prospects are never contacted.
- **Archive, don't delete.** Rows are the pipeline record.

## Status

Schema verified against Postgres 16: applies cleanly twice, and the constraints for one ICP per org, prospect dedupe, and message direction/status hold.

Not yet included:

- Row-level security keyed on `org_id`. Isolation currently relies on query discipline.
- A sending integration. Until one is connected, the user sends messages and the skill records them.
