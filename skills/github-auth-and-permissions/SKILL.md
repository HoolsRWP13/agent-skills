---
name: github-auth-and-permissions
description: Safely configure and diagnose GitHub authentication and permissions for agents that create repositories, branches, files, and pull requests.
---

# GitHub Authentication and Permissions

## Principles
- Use official OAuth, GitHub App installation, or pre-authenticated GitHub CLI flows.
- Prefer a GitHub App scoped to the required owner/repositories and permissions.
- Never request secrets in chat, commit them, print them, or place them in source files.
- Store credentials only in an approved secret manager/environment secret store.
- Never attempt privilege escalation or bypass organization controls.

## Capability matrix
| Operation | Typical required authorization |
|---|---|
| Read repository metadata | Repository metadata read |
| Create personal repository | Authenticated user allowed to create repos; token/API authorization suitable for account repository creation |
| Create organization repository | Organization policy plus authorized org repository-creation permission |
| Create/update files and branches | Contents read/write and metadata access on target repo |
| Open pull requests | Pull requests read/write on target repo |
| Configure webhooks/secrets/settings | Separate elevated permissions; request only when needed |

Exact requirements depend on OAuth app, fine-grained PAT, classic PAT, or GitHub App flow. Confirm current GitHub documentation before changing permission scopes.

## Diagnostics
1. Identify the authenticated GitHub account without exposing tokens.
2. Verify the target owner and repository visibility.
3. Check the minimum required permissions for the requested operation.
4. If authorization is insufficient, stop and ask the owner to complete the official authorization/installation flow.
5. After authorization, re-check the effective permissions before retrying.

## Security rules
- Never ask the user to paste a PAT, OAuth token, app private key, or recovery code into the conversation.
- Never store credentials in repository files, .env.example, README, issue text, or logs.
- .env.example may contain variable names and placeholder values only.
- Rotate/revoke exposed credentials through GitHub's official settings if a leak is detected.
