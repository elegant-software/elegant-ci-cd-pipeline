# GitHub Actions Conventions

Naming and reporting rules for every workflow in the Elegant Software repositories
(`zack-pay`, `blitzpay-admin-dashboard`, `blitz-pay-mobile`, and this repository).
The goal: anyone can tell **what a run did, to what, and whether it changed anything**
from the Actions list alone, without opening logs.

## 1. File names

Prefix each workflow file with its category so related workflows sort together in the
Actions sidebar. Use kebab-case.

| Prefix | Use for | Examples |
|---|---|---|
| `ci-` | Build, test, quality checks | `ci-build.yml`, `ci-publish-stubs.yml`, `ci-mobile-quality.yml` |
| `cd-` | Image builds, deployments | `cd-build-image.yml`, `cd-deploy-k8s.yml`, `cd-deploy-aws.yml` |
| `release-` | Versioning, release notes, tagging | `release-notes.yml`, `release-manual.yml` |
| `pr-` | Pull-request housekeeping | `pr-auto-label.yml` |
| `cleanup-` | Scheduled deletion of runs, releases, images | `cleanup-workflow-runs.yml`, `cleanup-releases-tags.yml`, `cleanup-ghcr-images.yml` |
| `automation-` | Bots and agent integrations | `automation-issue-spec.yml` |

> **Exception:** reusable workflows in `elegant-ci-cd-pipeline` keep their current file
> names. Every caller references them by path (`uses: …/<file>.yml@main`), so renaming one
> breaks all consumers.

Renaming a file starts a fresh run history under the new name. Before renaming, update
every reference to it: `uses: ./.github/workflows/<file>`, `workflow_id: '<file>'` lookups,
README badges, and docs.

## 2. Workflow `name:`

Format: `<Category> · <What>` in Title Case, with the category matching the file prefix.

```yaml
name: CI · Build & Test
name: CD · Deploy to Kubernetes
name: Cleanup · Releases & Tags
```

## 3. `run-name:` (run title in the Actions list)

**Required** for any workflow triggered by `workflow_dispatch`, `schedule`, `release`, or
`repository_dispatch`. Without it every run shows the same static title. For `push` and
`pull_request` you may omit it, because GitHub already uses the commit message or PR title.

The title states the **subject** (tag, version, environment) and, for anything
destructive or dry-runnable, the **trigger and mode**:

```yaml
run-name: Deploy ${{ inputs.image-tag }} to ${{ inputs.environment }}

run-name: >-
  ${{ github.event_name == 'schedule' && 'Scheduled' || 'Manual' }} ·
  ${{ (github.event_name == 'schedule' || inputs.dry_run) && 'Dry run' || 'LIVE' }} ·
  older than ${{ inputs.older_than_days || 30 }}d
```

`run-name` can only use the `github`, `inputs`, and `vars` contexts. Scheduled runs have
no inputs, so always supply a fallback (`inputs.x || default`).

## 4. Job `name:`

Always set `name:`. A job without one is displayed by its id (`cleanup`, `build`, `test`).

### Callers of reusable workflows

GitHub displays a job that calls a reusable workflow as **`<caller job name> / <called job name>`**.
Keep the caller side a **short noun**, and let the reusable workflow describe the action:

```yaml
# caller
jobs:
  cleanup:
    name: Releases & Tags          # → "Releases & Tags / Preview releases & tags older than 30d"
    uses: elegant-software/elegant-ci-cd-pipeline/.github/workflows/cleanup-release-tags.yml@main
```

Don't repeat the workflow name or the called job's wording in the caller job name
(avoid `Delete Old Releases & Tags / Delete releases …`).

### Reusable workflows and plain jobs

Name the job **action + key parameters**. Job names can use `inputs`, `needs`, `matrix`,
`vars`, and `github`.

```yaml
name: ${{ inputs.dry_run && 'Preview' || 'Delete' }} releases & tags older than ${{ inputs.older_than_days }}d
name: Build ${{ inputs.image-name }}:${{ inputs.image-tag }}
name: Publish ${{ needs.plan.outputs.next_version }}
name: Test (${{ matrix.os }}, Node ${{ matrix.node }})
```

- **Destructive jobs must say whether they really act**: `Preview …` for dry runs,
  `Delete …` / `Deploy …` for live runs.
- Put matrix values in the name, so parallel jobs can be told apart.

## 5. Step `name:`

Imperative, sentence case, one action per step: `Build and push image`,
`Verify rollout`. Give `uses:` steps a name too (`- uses: actions/checkout@v6` on its
own shows as `Run actions/checkout@v6`).

## 6. Report results in the job summary

Logs are for debugging; the **job summary** is for humans. Every job that produces or
changes something writes to `$GITHUB_STEP_SUMMARY`:

| Job type | Summary content |
|---|---|
| Cleanup | Table of matched items (name, date, reason, result: ✅ deleted / ❌ error / would delete) and counts |
| Image build | Image reference(s), digest, platforms |
| Deploy | Environment, image tag, namespace or service, rollout result |
| Release | Version, decision (released / skipped and why), notes |

From `actions/github-script`:

```js
await core.summary
  .addHeading(dryRun ? '🔍 Dry run — would delete' : '🗑️ Deleted', 3)
  .addTable([['Tag', 'Created', 'Result'].map((data) => ({ data, header: true })), ...rows])
  .write();
```

From shell:

```bash
{
  echo "### 🚀 Deployed \`${IMAGE_TAG}\` to ${ENVIRONMENT}"
  echo "| Namespace | Image |"
  echo "|---|---|"
  echo "| ${NAMESPACE} | \`${IMAGE}\` |"
} >> "$GITHUB_STEP_SUMMARY"
```

Cap long tables (for example, the first 100 rows plus a "… and N more" line). A single
summary is limited to 1 MiB.

## 7. Annotations for the headline

Emit **one** `notice` per job with the outcome, shown at the top of the run page:

```js
core.notice(`Would delete 3 of 10 releases (cutoff 2026-10-01)`);
```

```bash
echo "::notice title=Deployed::${IMAGE_TAG} → ${ENVIRONMENT}"
```

Use `warning` for partial failures that don't fail the job. Don't use annotations as a
log; the summary holds the detail.

## Checklist for a new workflow

- [ ] File named `<category>-<what>.yml`
- [ ] `name: <Category> · <What>`
- [ ] `run-name:` if not triggered only by push or PR
- [ ] Every job has a `name:`; caller jobs short, reusable or plain jobs action + parameters
- [ ] Dry-run vs live is visible in the run title and job name
- [ ] Results written to the job summary, plus one headline `notice`
