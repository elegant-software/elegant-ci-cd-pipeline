# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A **reusable GitHub Actions workflow library** designed to be called from other repositories via `workflow_call`. It contains CI/CD orchestration (release notes, Docker builds, Gradle tests), repo maintenance automation, and Spec Kit-driven planning infrastructure.

## Key Commands

```bash
# Create a new numbered Spec Kit feature branch
.specify/scripts/bash/create-new-feature.sh

# Initialize a feature plan from Spec Kit (outputs JSON)
.specify/scripts/bash/setup-plan.sh --json
```

There is no build, lint, or test runner — this repo contains GitHub Actions YAML and shell scripts, not application code.

## Architecture

### Workflow Design Pattern
All workflows expose a `workflow_call` interface with explicit inputs, outputs, and permissions. Many also support `workflow_dispatch` for manual testing. Workflows are stateless and pull logic via inline `actions/github-script@v7` Node.js or Python steps.

### Label-Driven Release Automation
`.github/labels.yml` is the single source of truth for semantic versioning and release note categories. Labels map to:
- **Category sections** (Features, Bug Fixes, Documentation, etc.) for release notes
- **Semver bump levels** (major/minor/patch) for version calculation

`release-notes.yml` reads these mappings (configurable via JSON inputs for cross-repo use), walks git history to find merged PRs since the last release tag, and emits structured `release_notes_markdown` and `release_notes_json` outputs for downstream jobs.

`pull-request-auto-label.yml` auto-applies labels by matching changed file paths, PR title, and PR body against patterns defined in the same `labels.yml`.

### Workflow Inventory

| Workflow | Purpose |
|---|---|
| `release-notes.yml` | Compute next semver + generate release notes from merged PRs |
| `pull-request-auto-label.yml` | Auto-label PRs from `labels.yml` file/title/body patterns |
| `backend-build.yml` | Build & push Docker image from Java/Gradle app with provenance attestation |
| `backend-gradle-test.yml` | Run Gradle unit + integration tests with PostgreSQL service container |
| `cleanup-github-workflows.yml` | Delete failed/old workflow runs |
| `cleanup-release-tags.yml` | Delete stale releases+paired tags, then orphaned tags (tags with no release) |
| `codex-spec-kit-agent.yml` | Run Codex CLI on the self-hosted runner with a Spec Kit prompt and open a PR |
| `issue-spec-automation.yml` | Validate new issues and hand them off to `codex-spec-kit-agent.yml` using `$speckit.specify <issue body>` |
| `ci-action-version-drift.yml` | Validate all workflow `uses:` versions match `.github/actions-versions.yml` on PRs touching `.github/**` |

### Spec Kit (`specs/` and `.specify/`)
Features are tracked in `specs/NNN-feature-name/` with spec, plan, tasks, and checklist files. `.specify/templates/` holds the Markdown templates for each artifact type. `.specify/scripts/bash/` contains helpers to scaffold new features. The `codex-spec-kit-agent.yml` workflow drives automated Spec Kit work via Codex CLI.

### Tests
- `tests/contract/` — Documents expected workflow interfaces (inputs, outputs, permissions)
- `tests/integration/` — Documents end-to-end scenario assertions; not executable, treated as living specs

## Conventions (from CONSTITUTION.md)

- **Workflow naming & reporting**: follow [`docs/github-actions-conventions.md`](docs/github-actions-conventions.md): category-prefixed files, `<Category> · <What>` names, `run-name` on dispatch/schedule, named jobs with parameters, results in the job summary
- **Secrets**: `UPPER_SNAKE_CASE` (e.g. `GHCR_TOKEN`, `AWS_ACCESS_KEY_ID`)
- **Environments**: lowercase only (`dev`, `test`, `staging`, `prod`)
- **Docker**: no wildcards in `COPY`; Gradle artifact must be named `app.jar`; use deterministic image tags, not filename versioning
- **Permissions**: always declare least-privilege per-workflow permissions. **Gotcha**: if a `jobs.<job>.permissions` block exists, it fully overrides the top-level `permissions` block — omitting a permission at the job level silently drops it (e.g. `id-token: write` required for attestations).
- **Commit messages**: semantic prefix — `feat:`, `fix:`, `docs:`, `refactor:`, `chore:`

## Workflow Documentation Standard
When writing or updating workflow docs in `docs/`, document inputs and outputs in tables:

| Input | Description | Required | Default |
|---|---|---|---|

| Output | Description |
|---|---|

Explicitly note when a workflow has no declared `workflow_call` outputs.

## Recent Changes
- 009-centralize-action-versions: Added [if applicable, e.g., PostgreSQL, CoreData, files or N/A]
