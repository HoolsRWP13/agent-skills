# Family Finance Bot — Architecture and Delivery Specification

Status: implementation blueprint; not a deployed bot.

## Product goal

A private Telegram-first family finance system for two adults. It records transactions, separates family/shared finances from each person's private finances, tracks business spending, maintains account balances, distinguishes received from expected income, and produces auditable reports.

## Non-negotiable invariants

1. Never invent a transaction, balance, payment, or receipt. Expected income is never included in cash balances.
2. Every monetary amount is stored as an integer in the currency's minor units (for RUB, kopecks) or as a fixed-precision decimal; never use binary floating point.
3. Every transaction has an explicit owner/scope: `PERSONAL` (visible only to its owner), `FAMILY` (visible to both linked family members), or `BUSINESS` (visible only to authorized members; default private to creator until explicitly shared).
4. Access is enforced server-side on every read, write, export, search, report, and callback. Hiding Telegram buttons is not authorization.
5. Transfers between accounts are paired ledger entries and do not count as income or expense.
6. Deleting or editing a posted financial record requires confirmation and creates an immutable audit event. Prefer reversal entries over destructive deletion.
7. Every balance is derived from confirmed ledger entries plus an explicit opening balance. No silent balance correction.
8. A transaction is not considered received/paid until the user confirms the actual event.
9. Family membership is explicit and revocable. Invitation tokens are single-use, expiring, random, stored hashed, and never logged.
10. Backups are encrypted, access-controlled, and periodically restore-tested.

## Roles and privacy

- `OWNER`: account creator; can manage family membership, settings, backups, and own private data.
- `FAMILY_MEMBER`: spouse; can access shared family ledger and their own private ledger only.
- No member can read the other member's private ledger, private accounts, or private reports.
- Shared family records are visible to both linked members.
- Business scope is a separate classification; visibility is controlled independently by explicit membership grants.
- Audit log exposes only events the viewer is authorized to know about. Do not leak private record names or amounts through audit metadata.

## MVP stack

- Python 3.12+
- aiogram 3.x Telegram bot
- PostgreSQL 16+ as the source of truth
- SQLAlchemy 2.x async + Alembic migrations
- Redis only for ephemeral FSM state, rate limiting, and job coordination (not ledger truth)
- APScheduler or a PostgreSQL-backed worker for reminders and reports; jobs must be idempotent
- pytest + pytest-asyncio; testcontainers for integration tests
- Docker Compose for local development; production deployment on a private always-on host
- LLM integration is optional and pluggable. Deterministic parsing and explicit confirmation must remain available without an LLM.

## Core domain model

- `users`: Telegram ID, display settings, status, created_at
- `families`: family_id, owner_user_id, currency, timezone, created_at
- `family_memberships`: family_id, user_id, role, status, joined_at, revoked_at
- `accounts`: account_id, family_id, owner_user_id nullable, scope, name, currency, opening_balance_minor, active
- `categories`: category_id, family_id, scope, name, parent_id, active
- `transactions`: transaction_id, family_id, created_by, scope, kind, amount_minor, currency, category_id, account_id, counterparty_account_id nullable, occurred_at, status, description, source, created_at, updated_at, version
- `expected_cashflows`: expected_id, family_id, owner_user_id, scope, direction, amount_minor, currency, due_at, status, linked_transaction_id nullable
- `budgets`: budget_id, family_id, scope, category_id, period, limit_minor
- `recurring_rules`: rule_id, family_id, owner_user_id, scope, cadence, next_due_at, template_json, enabled
- `audit_events`: event_id, family_id, actor_user_id, entity_type, entity_id, action, safe_metadata_json, created_at
- `invitations`: invitation_id, family_id, token_hash, expires_at, used_at, created_by
- `notification_preferences`: user_id, quiet_hours, report_schedule, enabled
- `schema_migrations`: managed by Alembic

All tables containing financial data must include family_id and appropriate ownership/scope fields. Add PostgreSQL constraints and indexes. Use row-level security as defense in depth where operationally practical, with application authorization as the primary enforcement layer.

## Transaction lifecycle

1. Parse natural-language input into a proposed transaction.
2. Show the user amount, currency, date/time, category, account, scope, and whether it is an expense/income/transfer.
3. Ask for confirmation when any material field is uncertain. Never silently guess the account or scope.
4. Persist in one database transaction; write audit event in the same transaction.
5. Recalculate balance from ledger entries; do not mutate a cached balance as source of truth.
6. Provide edit/reversal actions with confirmation and optimistic concurrency via version.
7. If a transfer is entered, create both sides atomically and validate currency/account compatibility.

## Income and expenses

- Confirmed income increases account balance.
- Confirmed expense decreases account balance.
- Expected income and expected expenses live in a separate table and never affect current balance, realized totals, or cashflow until marked received/paid.
- On receipt/payment, create a confirmed transaction and link it to the expected item atomically.
- Ozon/marketplace advances and remaining settlements must be tracked as separate expected/received amounts; only actual deposits count as received.
- Refunds and corrections use explicit reversal/refund records, preserving history.

## Family reserve (wife's NЗ)

Represent the reserve as a dedicated account owned by the wife, with privacy and family classification explicitly configured. A withdrawal from the reserve is one actual account expense and may be tagged as a family expense. Reports must show:
- reserve opening/current balance;
- target balance;
- withdrawals during the period;
- amount to replenish to restore withdrawn funds;
- additional amount needed to reach target.
Never double-count the same withdrawal as two cash outflows. A replenishment is a transfer into the reserve, not income.

## Reporting

- Daily: personal expenses (viewer only), family expenses, business expenses by authorized visibility, reserve withdrawals, confirmed income, expected inflows/outflows separately, per-account balances.
- Weekly/monthly: category totals, cashflow, budget variance, fixed obligations, reserve restoration gap, savings goals.
- Show source data and period boundaries. Reports must obey the same access checks as transaction queries.
- No notifications about expenses unless explicitly enabled by the user. Quiet hours and per-user preferences are required.

## Telegram UX

- Commands: /start, /help, /add, /income, /expense, /transfer, /accounts, /balance, /expected, /report, /family, /settings, /export.
- Natural-language input supported, but always return a review card before posting if ambiguity exists.
- Inline buttons use opaque IDs and re-check authorization at callback time.
- Invitation flow: owner requests invite; bot creates one-time expiring deep link; recipient authenticates via Telegram; owner can revoke membership.
- Never request bot tokens, database passwords, or API keys in chat. Secrets are environment variables or managed secrets.
- Rate-limit write operations and invitation attempts. Protect against duplicate Telegram updates with idempotency keys.

## Security and operations

- Private repository and private deployment by default.
- TLS in transit; encrypted database volume and encrypted backups at rest.
- Secrets never committed, printed, or included in exception traces.
- Structured logs must redact amounts/descriptions/Telegram IDs unless operationally necessary; never log invitation tokens.
- Database migrations are reviewed and reversible where possible.
- Backups daily with retention policy; restore drill at least monthly.
- Health/readiness checks, error reporting without financial payloads, dependency pinning, vulnerability scanning.
- Export requests require re-authentication/confirmation and are restricted to authorized scope.
- Data deletion must support export and legally appropriate deletion while retaining minimal non-sensitive audit evidence where required.

## MVP milestones

1. Repository bootstrap, configuration, Docker Compose, CI, migrations.
2. Telegram identity, family creation, secure invite/accept/revoke.
3. Accounts, categories, confirmed income/expense, transfers, audit log.
4. Expected cashflows, reserve account, fixed obligations and reminders.
5. Daily/monthly reports, CSV export, backup/restore.
6. Hardening: authorization matrix tests, duplicate update tests, concurrency tests, restore test, privacy review.

## Acceptance tests (release blockers)

- Spouse cannot read owner personal transactions by command, callback, export, search, or report.
- Both family members can read and add shared family transactions.
- Transfers leave total net worth unchanged and never appear as income/expense.
- Expected income never changes balance until confirmed received.
- Duplicate Telegram update does not duplicate a transaction.
- Editing/reversing a record preserves audit history.
- Unconfirmed deletion does not delete or reverse a record.
- Reserve withdrawal is counted once as an outflow and separately appears in reserve-restoration reporting.
- All amounts round-trip exactly; no float arithmetic.
- Backup can be restored into a clean database and ledger totals match.
- Revoked family member loses access immediately, including old callback buttons.

## Deployment gate

Do not call the bot production-ready until a real Telegram bot token is provisioned by the owner into the deployment secret store, database backups and restore are verified, and the privacy/authorization test suite passes. Never ask the user to paste credentials into chat.
