# Telegram Master Agent

This module extends the private Multi-AI Command Center with Telegram bot engineering and channel operations.

## Included components
- Agent persona: `agents/telegram-master.md`
- `telegram-bot-builder`: bot design, development, testing, and deployment
- `telegram-channel-operations`: owner-led channel setup and daily operations
- `telegram-content-growth`: editorial strategy and ethical audience growth
- `telegram-security-analytics`: security, permissions, reliability, and measurement

## Recommended integration
- Use Telegram's official Bot API over HTTPS.
- Keep the Command Center adapter in TypeScript and use a maintained Python or TypeScript bot framework for bot services.
- Use PostgreSQL for durable configuration/audit records and Redis queues for scheduled jobs.
- Use webhooks for production where a stable HTTPS endpoint is available.
- Keep channel and bot credentials encrypted and scoped per account/workspace.
- Separate draft creation from publication; require owner approval by default.
- Use per-bot rate limits, idempotency keys, delivery receipts, audit logs, and retry policies.

## Owner-controlled setup
1. Create/register each bot using Telegram's official BotFather flow and keep its token private.
2. Create channels in the Telegram client; channel creation/ownership is not a Bot API operation.
3. Add the chosen bot as a channel administrator and grant only the rights needed.
4. Store credentials through the Command Center's secure secret-management interface.
5. Test in a private staging channel before production.

## Important capability boundary
The Bot API can publish and manage channel posts only when the bot has the required rights. It cannot create a channel or grant itself administrator rights. Telegram's Bot Management Mode/managed-bot features may enable additional owner-mediated bot creation flows; evaluate against current official documentation before relying on them.

## Cost policy
Prefer free/open-source libraries and self-hosting. The Telegram Bot Platform is free for normal bot use, but hosting, domains, external AI providers, paid broadcasts, and Telegram Stars/payment features may incur costs. Disclose these before enabling them.

## Installation and updates
Treat external skills as untrusted code/instructions. Inspect repository source, license, maintainers, dependencies, release history, and security posture before installing. Pin versions or commit SHAs, record provenance, and run tests in isolation.
