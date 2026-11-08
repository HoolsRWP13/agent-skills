---
name: telegram-channel-operations
description: Plan, create, configure, and operate Telegram channels and linked discussion groups, including editorial calendars, publishing workflows, moderation, and channel administration. Use for new channels or ongoing channel management.
---

# Telegram Channel Operations

## Overview
Help the owner launch and operate legitimate Telegram channels with a clear audience, editorial purpose, safe administration, and measurable outcomes.

## Process
1. Define channel purpose, target audience, language, public/private visibility, name, username options, description, visual identity, posting cadence, and success metrics.
2. Prepare a launch pack: channel description, pinned welcome post, rules, initial posts, visual brief, and moderation policy.
3. Explain that channel creation and ownership are performed by the human owner in the Telegram client. Guide the owner through the UI; do not claim Bot API can create a channel.
4. If automation is required, have the owner add the designated bot as an administrator and grant only necessary rights (typically post/edit/delete only when required). Verify permissions before attempting actions.
5. Build an editorial calendar with draft, review, approved, scheduled, published, and failed states.
6. Require owner approval before public publishing unless the owner has explicitly configured a narrowly scoped auto-publishing policy.
7. Track post identifiers and timestamps; verify every publication result. Make retries idempotent and avoid duplicate posts.
8. Moderate comments and linked groups using transparent rules, rate limits, escalation paths, and human review for ambiguous cases.
9. Review performance weekly: views, reactions, forwards, subscriber change where available, link clicks via privacy-respecting analytics, and content themes.
10. Maintain an incident log, admin access review, backup/export plan, and owner handover notes.

## Guardrails
- Never buy followers, fabricate engagement, spam users, or scrape member lists.
- Do not promise reach or subscriber growth.
- Never remove admins, change ownership, delete a channel, or publish sensitive/high-impact material without explicit owner approval.
- Disclose API and hosting limitations; Bot API does not expose every channel statistic or administrative capability.

## Verification
Report what was drafted, what the owner must do in Telegram, what permissions were granted, and which posts were actually published. Separate measured metrics from estimates.
