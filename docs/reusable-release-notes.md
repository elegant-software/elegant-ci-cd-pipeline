# Reusable Release Notes Workflow

## Purpose

This repository exposes a reusable GitHub Actions workflow for release notes and semantic version selection. Calling repositories can use a single `workflow_call` interface to:

- collect merged pull requests after a release boundary
- group changes into release-note sections from pull request labels
- determine the highest required semantic version bump
- consume structured outputs in later tagging, publishing, or deployment steps

## What The Workflow Produces

The workflow emits:

- `release_required`
- `version_bump`
- `next_version`
- `target_sha`
- `release_notes_markdown`
- `release_notes_json`
- `included_pr_count`
- `excluded_prs_with_reasons`
- `decision_reason`

When the workflow runs directly from a merged `pull_request` event, it resolves the latest reachable semantic version tag as the release boundary, evaluates only the merged pull request that triggered the event, and creates the resulting GitHub Release as a draft prerelease with an `-alpha` tag suffix, such as `v1.2.4-alpha`. Reusable `workflow_call` runs do not publish; they only return release-note and version outputs for the caller.

If downstream CD must start from the generated alpha tag, configure `RELEASE_PUBLISH_TOKEN` as a GitHub App token or fine-grained personal access token. Events caused by the default `GITHUB_TOKEN` do not start another workflow run.

## Consumer Example

```yaml
jobs:
  release_notes:
    uses: your-org/your-repo/.github/workflows/release-notes.yml@main
    with:
      base_branch: main
      category_rules: >-
        {"Features":["type:feature"],"Fixes":["type:fix"],"Documentation":["type:docs"],"Maintenance":["type:maintenance"]}
      version_rules: >-
        {"major":["release:major"],"minor":["release:minor"],"patch":["release:patch","type:fix","type:docs","type:maintenance"]}

  publish:
    needs: release_notes
    if: needs.release_notes.outputs.release_required == 'true'
    runs-on: ubuntu-latest
    steps:
      - name: Show generated release notes
        run: |
          echo "${{ needs.release_notes.outputs.release_notes_markdown }}"
```

## Release Note Categories

The workflow uses label-driven categories. A default configuration is provided, but calling repositories can override it.

Suggested sections:

- `Features`
- `Fixes`
- `Documentation`
- `Maintenance`
- `Other Changes`

If no category label matches a pull request, the workflow places that pull request in `Other Changes`.

## Fallback Behavior

- If a pull request has no recognized category label, it still appears in the notes.
- If a pull request has no recognized version label, the default behavior is to treat it as `patch`.
- If `unlabeled_behavior` is set to `fail`, the workflow fails instead of defaulting.
- If conflicting version labels are present, the workflow fails with an actionable message.

## Consumer Guidance

Calling repositories should:

- maintain a consistent pull request label taxonomy
- ensure merged pull requests target the configured base branch
- use release tags or pass an explicit `release_boundary`
- consume `release_notes_markdown` or `release_notes_json` in downstream workflows

## Output Examples

Example markdown output:

```markdown
## Features
- Add reusable issue capture workflow ([#42](https://github.com/example/repo/pull/42), by @octocat)

## Fixes
- Handle conflicting version metadata ([#43](https://github.com/example/repo/pull/43), by @octocat)
```

Example no-release output:

```text
No releasable changes.
```
