---
name: github-repository-builder
description: Create and bootstrap GitHub repositories using authorized GitHub connector, GitHub CLI, or GitHub App credentials. Use when asked to create a repo, initialize a project, set visibility, configure branches, or scaffold repository files.
---

# GitHub Repository Builder

## Goal
Carry out GitHub repository creation and setup end-to-end whenever the connected runtime has the required authorized capability.

## First: inspect available capabilities
1. Inspect available GitHub tools/actions for repository creation and repository lookup.
2. Prefer native GitHub connector actions over shell/API fallbacks.
3. If no creation action exists, inspect whether the execution environment has an authenticated GitHub CLI (gh auth status) or a configured GitHub App service.
4. Do not assume a generic GitHub connector can create repositories merely because it can read and write files.

## Create flow
- Resolve owner/name; if owner is omitted, use the authenticated user's login.
- Default visibility to private.
- Default initialize README and default branch main.
- Check exact repository existence before creating.
- Create via connector if available; otherwise use gh repo create OWNER/NAME --private --add-readme only when gh is already authenticated and authorized.
- For an API backend, use GitHub's official repository creation endpoint with a secret-managed credential. Do not embed tokens in code, prompts, URLs, command history, or logs.
- Verify by fetching repository metadata and confirming owner, name, visibility, default branch, and browser URL.
- If setup requested, create a feature branch and add files in that branch; use PR review for multi-file changes when supported.

## Safe defaults
- Private repository unless user explicitly asks for public.
- Never delete, transfer, archive, or publish an existing repository without explicit confirmation.
- Never change visibility without explicit confirmation.
- Never silently use a similarly named repository.
- Never expose secrets in tool output or completion summaries.

## If blocked
Explain the exact missing capability:
- connector has no repository-create action;
- no authenticated gh session / GitHub App credential is configured;
- account or organization lacks repository-creation permission.
Give the owner the shortest official setup path. Do not ask for a PAT in chat. Do not pretend a repository was created.

## User-facing completion
Report the exact URL, visibility, default branch, bootstrap actions, and any unfinished setup.
