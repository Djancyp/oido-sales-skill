# db schema — oido sales

Reference for `setup.md`. Every table is prefixed `oido_sales_` and carries `org_id`, so one shared cloud DB can serve any number of organizations.

Postgres SQL. If the provider differs, translate the types and keep names, constraints, statuses and `org_id` scoping the same.

Apply once, in file order. Re-running is safe: types are guarded, tables use `IF NOT EXISTS`, and no table is dropped.

## Tables

| Table | Purpose | Scoped by |
|---|---|---|
| `oido_sales_orgs` | One row per organization using the system | — (is the scope) |
| `oido_sales_org_icp` | The org's ideal customer profile, one row per org | `org_id` |
| `oido_sales_prospects` | Companies and contacts found, with status | `org_id` |
| `oido_sales_messages` | Outbound drafts and sent messages, plus received replies | `org_id` |

## 1. Enum types and shared trigger function

```sql
DO $$ BEGIN
  CREATE TYPE oido_sales_prospect_status AS ENUM (
    'new', 'researched', 'drafted', 'approved', 'contacted',
    'replied', 'meeting', 'won', 'disqualified'
  );
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

DO $$ BEGIN
  CREATE TYPE oido_sales_message_status AS ENUM (
    'draft', 'pending_review', 'approved', 'sent', 'received', 'failed', 'archived'
  );
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

DO $$ BEGIN
  CREATE TYPE oido_sales_message_direction AS ENUM ('outbound', 'inbound');
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

CREATE OR REPLACE FUNCTION oido_set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

## 2. Organizations

```sql
CREATE TABLE IF NOT EXISTS oido_sales_orgs (
  id          BIGSERIAL PRIMARY KEY,
  slug        TEXT        NOT NULL UNIQUE,            -- lowercase, dashes
  name        TEXT        NOT NULL,
  domain      TEXT,
  offer       TEXT        NOT NULL,                   -- what it sells, to whom
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

DROP TRIGGER IF EXISTS trg_sales_orgs_updated_at ON oido_sales_orgs;
CREATE TRIGGER trg_sales_orgs_updated_at
  BEFORE UPDATE ON oido_sales_orgs
  FOR EACH ROW EXECUTE FUNCTION oido_set_updated_at();
```

## 3. ICP

One row per org. Updated in place when the ICP changes; the old values are not kept.

```sql
CREATE TABLE IF NOT EXISTS oido_sales_org_icp (
  id                BIGSERIAL PRIMARY KEY,
  org_id            BIGINT      NOT NULL UNIQUE REFERENCES oido_sales_orgs (id) ON DELETE RESTRICT,
  industries        TEXT[]      NOT NULL DEFAULT '{}',
  company_size_min  INTEGER,                          -- employees
  company_size_max  INTEGER,
  stages            TEXT[]      NOT NULL DEFAULT '{}', -- e.g. seed, series-a, growth
  geographies       TEXT[]      NOT NULL DEFAULT '{}',
  personas          JSONB       NOT NULL DEFAULT '[]', -- [{title, seniority, function, role: buyer|champion|blocker}]
  triggers          JSONB       NOT NULL DEFAULT '[]', -- [{event, why_now}]
  anti_icp          JSONB       NOT NULL DEFAULT '[]', -- [{rule, reason}]
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  CHECK (company_size_min IS NULL OR company_size_max IS NULL OR company_size_min <= company_size_max)
);

DROP TRIGGER IF EXISTS trg_sales_org_icp_updated_at ON oido_sales_org_icp;
CREATE TRIGGER trg_sales_org_icp_updated_at
  BEFORE UPDATE ON oido_sales_org_icp
  FOR EACH ROW EXECUTE FUNCTION oido_set_updated_at();
```

## 4. Prospects

A prospect is a company, optionally with one contact. `dedupe` is the key used before insert.

```sql
CREATE TABLE IF NOT EXISTS oido_sales_prospects (
  id              BIGSERIAL PRIMARY KEY,
  org_id          BIGINT      NOT NULL REFERENCES oido_sales_orgs (id) ON DELETE RESTRICT,
  company_name   TEXT        NOT NULL,
  company_domain  TEXT,
  contact_name    TEXT,
  contact_title   TEXT,
  contact_url     TEXT,                                -- public profile, if sourced
  contact_email   TEXT,                                -- published only; never guessed
  contact_phone   TEXT,                                -- published only; may be a company main line
  contact_source_url TEXT,                             -- page where the email or phone is published
  persona_role    TEXT        CHECK (persona_role IN ('buyer', 'champion', 'blocker', 'other')),
  fit_reason      TEXT        NOT NULL,                -- why it matches the ICP, in one or two sentences
  trigger_event   TEXT,                                -- the "why now", if any
  signal_type     TEXT,                                -- the ICP trigger event it matched
  signal_date     DATE,                                -- date of the signal on the source page
  source_url      TEXT        NOT NULL,                -- where each fact came from
  dedupe_key      TEXT        NOT NULL,                -- lower(company_domain) || '|' || lower(coalesce(contact_name, ''))
  status          oido_sales_prospect_status NOT NULL DEFAULT 'new',
  do_not_contact  BOOLEAN     NOT NULL DEFAULT false,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (org_id, dedupe_key)
);

-- Upgrade path for tables created before these columns existed.
ALTER TABLE oido_sales_prospects ADD COLUMN IF NOT EXISTS signal_type TEXT;
ALTER TABLE oido_sales_prospects ADD COLUMN IF NOT EXISTS signal_date DATE;
ALTER TABLE oido_sales_prospects ADD COLUMN IF NOT EXISTS contact_email TEXT;
ALTER TABLE oido_sales_prospects ADD COLUMN IF NOT EXISTS contact_phone TEXT;
ALTER TABLE oido_sales_prospects ADD COLUMN IF NOT EXISTS contact_source_url TEXT;

-- An email or phone must say where it was published.
DO $$ BEGIN
  ALTER TABLE oido_sales_prospects ADD CONSTRAINT oido_sales_prospects_contact_source_chk
    CHECK ((contact_email IS NULL AND contact_phone IS NULL) OR contact_source_url IS NOT NULL);
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

CREATE INDEX IF NOT EXISTS oido_sales_prospects_org_status_idx
  ON oido_sales_prospects (org_id, status);

DROP TRIGGER IF EXISTS trg_sales_prospects_updated_at ON oido_sales_prospects;
CREATE TRIGGER trg_sales_prospects_updated_at
  BEFORE UPDATE ON oido_sales_prospects
  FOR EACH ROW EXECUTE FUNCTION oido_set_updated_at();
```

## 5. Messages

Outbound drafts and sends, and inbound replies. One row per message.

```sql
CREATE TABLE IF NOT EXISTS oido_sales_messages (
  id                BIGSERIAL PRIMARY KEY,
  org_id            BIGINT      NOT NULL REFERENCES oido_sales_orgs (id) ON DELETE RESTRICT,
  prospect_id       BIGINT      NOT NULL REFERENCES oido_sales_prospects (id) ON DELETE RESTRICT,
  direction         oido_sales_message_direction NOT NULL DEFAULT 'outbound',
  step              INTEGER     NOT NULL DEFAULT 1 CHECK (step >= 1),  -- 1 = first touch, 2+ = follow-up
  channel           TEXT        NOT NULL CHECK (channel IN ('email', 'linkedin', 'other')),
  subject           TEXT,
  body              TEXT        NOT NULL,
  content_hash      TEXT        NOT NULL,                              -- sha256 hex of subject || body
  status            oido_sales_message_status NOT NULL DEFAULT 'draft',
  sent_at           TIMESTAMPTZ,
  external_ref      TEXT,                                              -- message id from the channel, if any
  failure_reason    TEXT,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  CHECK ((direction = 'outbound') = (status <> 'received'))
);

CREATE INDEX IF NOT EXISTS oido_sales_messages_org_status_idx
  ON oido_sales_messages (org_id, status);
CREATE INDEX IF NOT EXISTS oido_sales_messages_prospect_idx
  ON oido_sales_messages (prospect_id);
CREATE INDEX IF NOT EXISTS oido_sales_messages_hash_idx
  ON oido_sales_messages (org_id, content_hash);

DROP TRIGGER IF EXISTS trg_sales_messages_updated_at ON oido_sales_messages;
CREATE TRIGGER trg_sales_messages_updated_at
  BEFORE UPDATE ON oido_sales_messages
  FOR EACH ROW EXECUTE FUNCTION oido_set_updated_at();
```

## Tenant isolation

These tables share one DB. `org_id` on every row is the boundary. Every query in `db-usage.md` filters by it. Database-level row security (RLS keyed on `org_id`) is the stronger guarantee; it is not in this schema yet and needs a decision on how the DB role is set per session.
