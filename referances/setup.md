# setup — sales tracking database and org onboarding

Trigger: the user asks for sales help, prospecting, or any of the flow steps, and the gate in `SKILL.md` is not yet passed.

## Step 1 — check the DB

Ask the user for the provider and connection details if none are connected. Never write them into these files or print them in chat.

Check whether every table in `db-schema.md` exists:

```sql
SELECT t, to_regclass(t) IS NOT NULL AS present
FROM unnest(ARRAY[
  'oido_sales_orgs',
  'oido_sales_org_icp',
  'oido_sales_prospects',
  'oido_sales_messages'
]) AS t;
```

- All `present = true`: go to Step 3.
- Some missing, or no connection: go to Step 2.

## Step 2 — ask (do not skip)

Ask once with AskUserQuestion:

- Question: "Set up the sales tracking database so I can save prospects and outreach for your organization?"
- Options:
  - "Yes, set it up (Recommended)" — required for prospecting and pipeline tracking.
  - "No, keep it in conversation" — nothing is created; prospects are listed in chat only and not saved.

Create nothing on an answer you inferred. Only the user's answer counts.

## Step 3 — create tables (after yes, only if missing)

1. Apply `db-schema.md` in file order. Every statement is idempotent; re-running is safe.
2. Run the check query from Step 1. All four must show `present = true`.
3. Never drop a table that holds data without asking the user.

## Step 4 — onboard the org

One org per row in `oido_sales_orgs`. Several organizations can share the same DB; each is isolated by `org_id`.

1. Ask with AskUserQuestion (one question per field, or one form): organization name, website domain, and a short description of what it sells and to whom.
2. Pick a `slug`: lowercase, dashes, unique (for example `acme-robotics`).
3. Check for an existing org first:
   ```sql
   SELECT id, name FROM oido_sales_orgs WHERE slug = $1;
   ```
   If it exists, ask whether to use it. Do not create a duplicate.
4. Insert the org and keep its `id`. Every later write uses it.
   ```sql
   INSERT INTO oido_sales_orgs (slug, name, domain, offer)
   VALUES ($1, $2, $3, $4)
   ON CONFLICT (slug) DO NOTHING
   RETURNING id;
   ```
5. Read it back to confirm.

## Step 5 — define the ICP

Continue with `icp-research.md`. Prospecting is blocked until the ICP row exists.

## If the connection fails

Ask the user to confirm the provider and connection details. Do not create a second database to work around a failed connection.
