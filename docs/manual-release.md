# Manual Release Workflow

## Purpose

`manual-release.yml` centralizes manual semantic-version release publication for application repositories. It evaluates conventional commits after the latest release tag (or an explicit boundary), prepares notes, and creates the version tag and GitHub Release in the calling repository.

The centralized implementation lives in `.github/scripts/release/`; consumers do not copy these modules into their own repositories.

## Consumer Workflow

The calling repository keeps only a manual trigger:

```yaml
name: Manual release

on:
  workflow_dispatch:
    inputs:
      release_boundary:
        description: 'Tag to diff from. Leave blank to use the latest tag.'
        required: false
        type: string
        default: ''
      base_branch:
        description: 'Branch to release'
        required: false
        type: string
        default: main

permissions: {}

jobs:
  manual-release:
    uses: elegant-software/elegant-ci-cd-pipeline/.github/workflows/manual-release.yml@main
    with:
      release_boundary: ${{ inputs.release_boundary }}
      base_branch: ${{ inputs.base_branch }}
      automation_ref: main
    secrets:
      RELEASE_PUBLISH_TOKEN: ${{ secrets.RELEASE_PUBLISH_TOKEN }}
      AUTOMATION_READ_TOKEN: ${{ secrets.AUTOMATION_READ_TOKEN }}
```

## Required Secret

Each caller provides `RELEASE_PUBLISH_TOKEN`, scoped to create tags and releases in that repository. A GitHub App token or fine-grained personal access token is required when publishing a release must start a separate `on: release` deployment workflow: events caused by the repository `GITHUB_TOKEN` do not start another workflow run.

If `elegant-ci-cd-pipeline` is private or internal, each caller also provides `AUTOMATION_READ_TOKEN` with read access to that shared repository. For a public shared repository this secret can be omitted and checkout uses the caller's standard token.

## Why There Are Two Checkouts

A reusable workflow runs in the caller's event context. `actions/checkout` without a `repository` parameter checks out the calling application repository, which is required to inspect its commits and tags.

The workflow makes a second checkout of `elegant-ci-cd-pipeline` at `.release-automation` so its centralized Node modules are available while the current directory remains the caller repository. This allows the same tested scripts to execute against `blitz-pay`, `blitzpay-admin-dashboard`, or another caller.

Credentials are not persisted on the shared-script checkout; those files are executed locally and do not need authenticated Git access after retrieval.

## Reference Pinning

The examples use `@main` while adopting the workflow. For stable production use, pin the reusable workflow to a release tag or commit SHA and set `automation_ref` to the same tag or SHA so the YAML workflow and its checked-out scripts cannot drift independently.

## Interface

| Item | Required | Purpose |
|---|---:|---|
| `base_branch` | No | Branch evaluated for commits; defaults to `main`. |
| `release_boundary` | No | Semantic tag used as the lower boundary; defaults to the latest reachable semantic tag. |
| `automation_ref` | No | Shared-pipeline ref containing script dependencies; defaults to `main`. |
| `RELEASE_PUBLISH_TOKEN` | Yes | Credential used only for tag and GitHub Release creation. |
| `AUTOMATION_READ_TOKEN` | Only for private/internal sharing | Read-only credential for fetching shared script dependencies. |

The workflow emits `release_required` and attaches the computed release plan as an internal artifact between its planning and publication jobs.
