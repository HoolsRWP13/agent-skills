---
name: family-finance-bot-builder
description: Design, implement, test, and secure a private Telegram family-finance bot with strict personal/family/business separation, ledger correctness, expected-vs-received cashflows, and auditable operations.
---

# Family Finance Bot Builder

## Mission

Build a private, reliable Telegram family-finance bot. Treat financial correctness and privacy as release blockers. The architecture specification is in `docs/family-finance-bot-architecture.md`.

## Mandatory rules

- Never invent transactions, balances, receipts, or payments.
- Never count expected cashflows as realized income/expense or include them in account balances.
- Use integer minor units or fixed-precision decimal; never binary floats for money.
- Model `PERSONAL`, `FAMILY`, and `BUSINESS` scopes explicitly.
- Enforce access on the server for every query, callback, report, and export.
- A spouse sees shared family data and their own private data, never the other person's private data.
- Transfers are paired ledger entries and are not income/expense.
- Deletion/editing requires confirmation; preserve immutable audit history and prefer reversals.
- Idempotency is mandatory for Telegram update handling.
- Secrets must never be requested in chat or committed to git.
- No expense notifications by default; respect per-user settings and quiet hours.
- Do not declare production readiness without backup-restore proof and authorization tests.

## Execution sequence

1. Inspect the existing repository and its conventions before editing.
2. Create a private implementation repository; if repository creation is unavailable, stop at a documented scaffold and report the exact owner-authorized action required. Do not put production finance data in a public skills repository.
3. Bootstrap Python, aiogram, PostgreSQL, SQLAlchemy async, Alembic, pytest, Docker Compose, and CI.
4. Implement schema and migrations before bot handlers.
5. Implement authentication/family membership and authorization tests before any report or export feature.
6. Implement ledger primitives (income, expense, transfer, reversal) with transactional writes and audit events.
7. Implement expected cashflows separately from realized transactions.
8. Implement reserve, recurring obligations, reports, exports, backup and restore.
9. Run unit, integration, authorization, idempotency, concurrency, and restore tests.
10. Open a pull request with test evidence. Never merge merely because CI is green; inspect business invariants and security tests.

## Required test cases

At minimum, test every release blocker listed in the architecture document. Add negative authorization tests for direct API calls and stale Telegram callbacks. Test exact monetary arithmetic, duplicate updates, paired transfers, expected-to-realized conversion, reversals, and database restore.

## User interaction

Ask the owner only for actions that require their account or secret, such as creating the private repository, creating a Telegram bot with BotFather, or provisioning deployment credentials. Do not ask for secrets in chat. Give concise, exact steps and continue all independent work.
