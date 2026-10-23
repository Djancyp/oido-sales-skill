# ICP — define or update the org's ideal customer profile

Trigger: after org onboarding (`setup.md`), or when the user asks to change who they target.

The ICP is what every prospect is judged against. Do not search for prospects until it is saved.

## Step 1 — ask (one AskUserQuestion per section, or one form)

1. **Firmographics:** industries, company size (min and max employees), stages, geographies.
2. **Personas:** job titles, seniority, function, and whether each is the buyer, champion or blocker.
3. **Triggers:** public events that signal fit now (funding, hiring for a role, a product launch, a relevant post, leadership change), with a short "why now" for each.
4. **Anti-ICP:** who to actively exclude, and why.

Offer a suggestion drawn from the org's `offer` in `oido_sales_orgs`, and let the user confirm or change it. Do not assume a market the user did not state.

## Step 2 — save

Upsert the single row for the org:

```sql
INSERT INTO oido_sales_org_icp
  (org_id, industries, company_size_min, company_size_max, stages, geographies, personas, triggers, anti_icp)
VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
ON CONFLICT (org_id) DO UPDATE SET
  industries = EXCLUDED.industries,
  company_size_min = EXCLUDED.company_size_min,
  company_size_max = EXCLUDED.company_size_max,
  stages = EXCLUDED.stages,
  geographies = EXCLUDED.geographies,
  personas = EXCLUDED.personas,
  triggers = EXCLUDED.triggers,
  anti_icp = EXCLUDED.anti_icp;
```

Read the row back and show the user the saved ICP in plain language. Ask them to confirm it.

## Step 3 — check

- Each persona has a role (`buyer`, `champion`, `blocker`).
- Each trigger has a "why now".
- The anti-ICP is not empty. If the user has none, ask once; do not invent one.
