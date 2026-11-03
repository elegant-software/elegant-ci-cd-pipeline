# Engineering Constitution

## 1. GitHub Actions Naming Convention

Full rules, examples, and a new-workflow checklist: [`docs/github-actions-conventions.md`](docs/github-actions-conventions.md).
They apply to this repository and to every repository that calls its workflows.

- **File names**: kebab-case with a category prefix: `ci-`, `cd-`, `release-`, `pr-`, `cleanup-`, `automation-`.
  Reusable workflows in this repository keep their existing file names, because callers reference them by path.
- **Workflow `name:`**: `<Category> · <What>` (e.g. `Cleanup · Releases & Tags`).
- **`run-name:`**: required for dispatch, schedule, release, and repository-dispatch triggers; states the subject and, for destructive runs, `Scheduled|Manual · Dry run|LIVE`.
- **Job `name:`**: always set. Caller jobs use a short noun (GitHub shows `<caller> / <called>`); reusable or plain jobs use action + key parameters, and destructive jobs say `Preview …` vs `Delete …`.
- **Step `name:`**: imperative, sentence case, e.g. `- name: Delete old workflow runs`.
- **Results**: write what changed to `$GITHUB_STEP_SUMMARY` and emit one headline `notice` per job.

---

## 2. Secrets Naming Convention

- Use UPPER_SNAKE_CASE
- Be explicit and descriptive

Examples:
- GHCR_TOKEN
- AWS_ACCESS_KEY_ID
- DB_PASSWORD

Usage:
```yaml
${{ secrets.GHCR_TOKEN }}
```

---

## 3. Environment Naming Convention

- Use lowercase
- Standard names only

Allowed:
- dev
- test
- staging
- prod

Example:
```yaml
environment: prod
```

---

## 4. Docker Rules

- Do NOT use wildcards in COPY
- Use deterministic naming

Gradle:
```kotlin
tasks.named<BootJar>("bootJar") {
    archiveFileName.set("app.jar")
}
```

Dockerfile:
```dockerfile
COPY build/libs/app.jar app.jar
```

---

## 5. Versioning Strategy

- Use version in:
  - Git tags (v1.0.0)
  - Docker image tags
- Do NOT rely on filename versioning

---

## 6. CI/CD Principles

- Keep workflows small and reusable
- Use reusable workflows via workflow_call
- Use least privilege permissions

Example:
```yaml
permissions:
  contents: read
  actions: write
```

---

## 7. General Rules

- Prefer clarity over brevity
- Avoid duplication
- Keep naming consistent across repo
