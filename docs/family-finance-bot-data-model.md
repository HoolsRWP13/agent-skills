# Family Finance Bot — Data Model and Ledger Rules

This document supplements [the architecture](family-finance-bot-architecture.md). It is a schema-level implementation contract, not a production migration.

## Money representation

Use `BIGINT amount_minor` and ISO 4217 currency code. RUB values are kopecks (₽1 = 100). Amounts are positive magnitudes; direction is represented by transaction kind and ledger postings. Reject zero and negative magnitudes. Currency conversion is out of MVP scope; transfers require equal currencies.

## Suggested PostgreSQL schema

```sql
CREATE TYPE finance_scope AS ENUM ('PERSONAL', 'FAMILY', 'BUSINESS');
CREATE TYPE transaction_kind AS ENUM ('INCOME', 'EXPENSE', 'TRANSFER', 'REFUND', 'REVERSAL');
CREATE TYPE transaction_status AS ENUM ('CONFIRMED', 'VOIDED');
CREATE TYPE cashflow_direction AS ENUM ('INCOMING', 'OUTGOING');
CREATE TYPE expected_status AS ENUM ('EXPECTED', 'PARTIALLY_REALIZED', 'REALIZED', 'CANCELLED');

CREATE TABLE users (
  id UUID PRIMARY KEY,
  telegram_user_id BIGINT UNIQUE NOT NULL,
  display_name TEXT,
  timezone TEXT NOT NULL DEFAULT 'Europe/Warsaw',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  disabled_at TIMESTAMPTZ
);

CREATE TABLE families (
  id UUID PRIMARY KEY,
  owner_user_id UUID NOT NULL REFERENCES users(id),
  base_currency CHAR(3) NOT NULL DEFAULT 'RUB',
  timezone TEXT NOT NULL DEFAULT 'Europe/Warsaw',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE family_memberships (
  family_id UUID NOT NULL REFERENCES families(id),
  user_id UUID NOT NULL REFERENCES users(id),
  role TEXT NOT NULL CHECK (role IN ('OWNER','MEMBER')),
  joined_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  revoked_at TIMESTAMPTZ,
  PRIMARY KEY (family_id, user_id)
);

CREATE TABLE accounts (
  id UUID PRIMARY KEY,
  family_id UUID NOT NULL REFERENCES families(id),
  owner_user_id UUID REFERENCES users(id),
  scope finance_scope NOT NULL,
  name TEXT NOT NULL,
  currency CHAR(3) NOT NULL,
  opening_balance_minor BIGINT NOT NULL DEFAULT 0,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  CHECK ((scope = 'PERSONAL' AND owner_user_id IS NOT NULL)
      OR (scope IN ('FAMILY','BUSINESS')))
);

CREATE TABLE categories (
  id UUID PRIMARY KEY,
  family_id UUID NOT NULL REFERENCES families(id),
  scope finance_scope NOT NULL,
  name TEXT NOT NULL,
  parent_id UUID REFERENCES categories(id),
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  UNIQUE (family_id, scope, name)
);

CREATE TABLE transactions (
  id UUID PRIMARY KEY,
  family_id UUID NOT NULL REFERENCES families(id),
  created_by UUID NOT NULL REFERENCES users(id),
  scope finance_scope NOT NULL,
  kind transaction_kind NOT NULL,
  status transaction_status NOT NULL DEFAULT 'CONFIRMED',
  amount_minor BIGINT NOT NULL CHECK (amount_minor > 0),
  currency CHAR(3) NOT NULL,
  category_id UUID REFERENCES categories(id),
  account_id UUID NOT NULL REFERENCES accounts(id),
  destination_account_id UUID REFERENCES accounts(id),
  description TEXT,
  occurred_at TIMESTAMPTZ NOT NULL,
  idempotency_key TEXT NOT NULL,
  version INTEGER NOT NULL DEFAULT 1 CHECK (version > 0),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (created_by, idempotency_key),
  CHECK ((kind = 'TRANSFER' AND destination_account_id IS NOT NULL
          AND destination_account_id <> account_id)
      OR (kind <> 'TRANSFER' AND destination_account_id IS NULL))
);

CREATE TABLE expected_cashflows (
  id UUID PRIMARY KEY,
  family_id UUID NOT NULL REFERENCES families(id),
  owner_user_id UUID REFERENCES users(id),
  scope finance_scope NOT NULL,
  direction cashflow_direction NOT NULL,
  amount_minor BIGINT NOT NULL CHECK (amount_minor > 0),
  realized_minor BIGINT NOT NULL DEFAULT 0 CHECK (realized_minor >= 0),
  currency CHAR(3) NOT NULL,
  due_at TIMESTAMPTZ,
  status expected_status NOT NULL DEFAULT 'EXPECTED',
  description TEXT,
  created_by UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  CHECK (realized_minor <= amount_minor),
  CHECK ((scope = 'PERSONAL' AND owner_user_id IS NOT NULL)
      OR scope IN ('FAMILY','BUSINESS'))
);

CREATE TABLE expected_realizations (
  expected_id UUID NOT NULL REFERENCES expected_cashflows(id),
  transaction_id UUID NOT NULL UNIQUE REFERENCES transactions(id),
  amount_minor BIGINT NOT NULL CHECK (amount_minor > 0),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (expected_id, transaction_id)
);

CREATE TABLE audit_events (
  id UUID PRIMARY KEY,
  family_id UUID NOT NULL REFERENCES families(id),
  actor_user_id UUID NOT NULL REFERENCES users(id),
  entity_type TEXT NOT NULL,
  entity_id UUID NOT NULL,
  action TEXT NOT NULL,
  safe_metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE invitations (
  id UUID PRIMARY KEY,
  family_id UUID NOT NULL REFERENCES families(id),
  token_hash BYTEA NOT NULL UNIQUE,
  expires_at TIMESTAMPTZ NOT NULL,
  used_at TIMESTAMPTZ,
  revoked_at TIMESTAMPTZ,
  created_by UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Required constraints beyond basic DDL

- Composite foreign keys or deferred constraint triggers must ensure every referenced account/category belongs to the same family as the transaction.
- Personal accounts and transactions must match the owning user; family/business records must have explicit visibility policy.
- A transfer's source and destination account currencies must match, and both accounts must belong to the same family.
- A transfer must generate two balanced ledger postings in one database transaction. Do not infer balances from mutable cached totals.
- Prevent a transaction from referencing a category from another scope unless an explicit category policy permits it.
- Expected realization rows must sum to `realized_minor`; update both under row lock in one transaction.
- For every write, insert audit event in the same database transaction.
- Use database transactions and `SELECT ... FOR UPDATE` for expected-cashflow realization and concurrent account operations.
- Use a unique Telegram update/idempotency record so retries cannot duplicate writes.

## Recommended ledger extension

For production-grade accounting, use a postings table as the balance source of truth:

```sql
CREATE TABLE ledger_postings (
  id UUID PRIMARY KEY,
  transaction_id UUID NOT NULL REFERENCES transactions(id),
  account_id UUID NOT NULL REFERENCES accounts(id),
  amount_minor BIGINT NOT NULL CHECK (amount_minor <> 0),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ledger_postings_account_created_idx
  ON ledger_postings(account_id, created_at);
```

Convention: positive posting increases an account balance; negative posting decreases it. Income has one positive posting, expense one negative posting, transfer has equal-and-opposite postings. Refund/reversal postings explicitly counter prior postings. Enforce that postings for a transfer sum to zero per currency, and that non-transfer transaction posting shape matches kind. Implement through a deferred constraint trigger or a single trusted ledger service plus database constraints.

## Authorization query contract

Every repository method must receive an authenticated actor and family context. Query filters are mandatory:

- PERSONAL: `owner_user_id = actor_id`
- FAMILY: active membership in the same family
- BUSINESS: explicit business access grant; never assume family membership implies access
- revoked membership: deny immediately, even for stale inline callback IDs

Do not expose an unfiltered generic `get_by_id` to handlers. Provide scoped methods such as `get_visible_transaction(actor, family, transaction_id)`.

## Balance definition

`balance(account) = opening_balance_minor + SUM(ledger_postings.amount_minor WHERE account_id = ...)`

Expected cashflows, budgets, and reserve target gaps are not postings and never affect balance.

## Release-blocking tests

1. Cross-family account/category references fail.
2. Personal owner mismatch fails.
3. A transfer creates exactly two postings and net postings equal zero.
4. A failed second transfer posting rolls back the entire transaction.
5. Concurrent partial realizations cannot exceed expected amount.
6. Duplicate Telegram update is idempotent.
7. Revoked membership is denied for a previously valid callback.
8. Report aggregation uses authorized account set only.
9. Amounts preserve exact kopecks across parsing, persistence, and report output.
10. A reversal preserves original transaction and audit trail.
