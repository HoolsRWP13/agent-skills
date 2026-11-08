# GitHub Repository Builder — Setup

## What this agent can do
When properly authorized, it can create repositories, initialize README files, create branches, scaffold project files, and open pull requests.

## Current connector limitation
The currently connected GitHub toolset supports repository listing/search and repository file/branch/commit/PR operations, but does not expose a repository-creation action. File-writing access does not imply permission to create repositories.

## One-time authorization choices

### Option A — GitHub CLI on a trusted runner
1. Install GitHub CLI (gh) on the agent's trusted execution host.
2. Authenticate using GitHub's official device/browser flow (gh auth login).
3. Verify with gh auth status.
4. Ensure the account is permitted to create repositories.
5. Keep the runner private; never paste the CLI token into chat or commit it.

Then the agent can run:

    gh repo create OWNER/REPOSITORY --private --add-readme

### Option B — GitHub App backend (preferred for a hosted agent)
1. Create a GitHub App owned by the user.
2. Grant only the permissions required for the workflow.
3. Install it only on the intended account/organization.
4. Keep the App private key in a secret manager and mint short-lived installation tokens server-side.
5. Add an audited backend operation for repository creation. Do not expose the private key to the model.

A GitHub App's installation permissions and organization policies must support repository creation. If the App cannot create repositories, the owner must authorize that capability using GitHub's official UI.

## Important
An agent cannot grant itself GitHub permissions. A one-time owner authorization is unavoidable. After that, repository creation can be autonomous within the granted scope.

## Verification
After creating a repo, fetch repository metadata and confirm:
- owner and exact name;
- private/public visibility;
- default branch;
- HTML URL.
