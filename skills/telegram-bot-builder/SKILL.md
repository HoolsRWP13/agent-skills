---
name: telegram-bot-builder
description: Design, implement, test, deploy, and maintain Telegram bots using the official Bot API and maintained frameworks. Use for bot creation, commands, menus, inline keyboards, callbacks, webhooks, polling, Mini Apps, payments, integrations, and AI assistants.
---

# Telegram Bot Builder

## Overview
Build maintainable Telegram bots that solve a defined user problem. Prefer the official Bot API and documented framework behavior.

## Process
1. Define the bot's audience, jobs, commands, data collected, privacy needs, expected load, and success criteria.
2. Confirm owner setup: bot registration through the official @BotFather flow, token storage, bot username, and required group/channel permissions. Never request a token in ordinary chat.
3. Choose the smallest suitable stack:
   - Python: aiogram (async) for Python-first services.
   - TypeScript: grammY or Telegraf for TypeScript-first services.
   - Use direct HTTPS Bot API only when it materially reduces complexity.
4. Design conversation states, commands, callback payloads, idempotency, error handling, localization, and accessibility.
5. Choose webhook for production where a stable HTTPS endpoint exists; use long polling for local development or suitable small deployments. Never run polling and webhook simultaneously for the same bot.
6. Validate every incoming update, authorize users and admins server-side, and protect against replay, duplicate updates, malformed payloads, and abuse.
7. Store secrets outside source control. Redact tokens and personal data from logs. Use encrypted storage and rotation procedures.
8. Add unit tests for handlers and state transitions, integration tests against a test bot, and contract checks for API payloads.
9. Deploy with health checks, structured logs, metrics, alerting, graceful shutdown, and rollback.
10. Verify bot identity with getMe, webhook state with getWebhookInfo when applicable, and real test conversations before declaring success.

## Security and reliability
- Apply least privilege to channel/group admin rights.
- Respect Telegram limits and implement bounded retries with backoff for transient failures and 429 responses.
- Do not retry non-idempotent actions blindly.
- Do not initiate unsolicited private conversations; users must first start the bot.
- Never use bots for spam, scraping private conversations, or deceptive engagement.

## Verification
Provide test output, API verification evidence, deployment URL/health status where relevant, cost disclosure, and remaining owner actions. Clearly distinguish code written from a bot actually registered, configured, and running.
