# chiselWare CI Workflows

This repository defines the shared CI/CD workflow framework for the chiselWare organization. It provides reusable GitHub Actions workflows that are called by individual IP repositories, keeping CI behavior consistent, auditable, and maintainable across the organization.

---

## 1. Purpose

`ci-workflows` is a versioned CI platform, not a product repository. Individual IP repositories (e.g. `00-000-dff`) do not implement CI logic themselves — they call workflows defined here. This keeps IP repositories lightweight and independent while enforcing consistent regression behavior across all cores.

`ci-workflows` provides:

- Reusable GitHub Actions workflows (`workflow_call`)
- A stable, versioned CI interface used by all IP repositories
- Centralized enforcement of CI behavior and security policy
- A clean path from self-hosted execution to cloud execution (e.g. Azure)

---

## 2. Repository Structure

```
ci-workflows/
├── .github/
│   └── workflows/
│       └── make-all.yml        # Primary regression workflow (v1 API)
└── README.md
```

---

## 3. Versioning Model

### Philosophy

CI workflows are infrastructure APIs, not user-facing products. Changes are almost always either backward-compatible or breaking — the middle ground of minor semantic versioning rarely applies. For this reason, `ci-workflows` uses **major-only floating tags** as its primary interface, mirroring the convention used by GitHub's own actions (e.g. `actions/checkout@v4`).

Major version increments are intentionally rare and are aligned with major chiselWare Standard (CWS) releases. IP repositories can expect a stable CI contract for the lifetime of a CWS major version.

### Tag Structure

Each release carries two tags:

- **Immutable patch tag** (e.g. `v1.2.3`) — created once, never moved. Provides a precise audit trail and allows any IP repo to pin to a specific workflow version for debugging or isolation.
- **Floating major tag** (e.g. `v1`) — always points to the latest stable release within that major version. This is what IP repos reference in their caller workflows.

Example usage from an IP repo:

```yaml
uses: chiselWare/ci-workflows/.github/workflows/make-all.yml@v1
```

### Tag Maintenance Procedure

After merging a change to `main`, the maintainer creates both tags:

```bash
# Create the immutable patch tag
git tag v1.2.3
git push origin v1.2.3

# Move the floating major tag forward
git tag -f v1 v1.2.3
git push origin v1 --force
```

IP repos using `@v1` pick up the update automatically on their next CI run. The immutable `v1.2.3` tag remains available indefinitely for pinning or rollback.

### When to Tag

- **Patch release** (e.g. `v1.0.0` → `v1.0.1`): backward-compatible fixes or additions. IP repos pick up the change transparently via the floating major tag.
- **Major release** (e.g. `v1` → `v2`): breaking changes to the workflow API — new required inputs, removed steps, changed artifact names or paths, or significant behavioral changes. Major releases are coordinated with the corresponding CWS major release. IP repos remain on `@v1` until explicitly migrated by their maintainer.

### Version Alignment with CWS

| ci-workflows | CWS |
|---|---|
| v1 | v1.x |
| v2 | v2.x |

An IP repo on CWS v1.x should always reference `ci-workflows@v1`.

---

## 4. Branching and Promotion Model

The organization uses a **feature → staging → main** branching model to balance developer autonomy with strong quality gates.

### Feature Branches

Day-to-day work happens on short-lived feature branches created from `main`. Developers are free to iterate and run local regressions as needed. Feature branches are never merged directly to `main`.

### Staging

When a change is ready, the developer opens a pull request targeting `staging`. Merging into `staging` triggers the authoritative CI regression via `ci-workflows`. CI must complete successfully before the branch is eligible for promotion. `staging` is the integration and verification gate — toolchain-heavy synthesis, timing, and documentation checks run here consistently, independent of individual developer machines.

### Main

Once `staging` is green, promotion to `main` is done via a second pull request (`staging → main`). This PR is reviewed by admins or maintainers and serves as the final human gate. `main` always reflects the last known-good, maintainer-approved state. Direct pushes to `main` are not permitted.

If review raises issues, changes are made on a new feature branch and re-promoted through `staging`. `main` is never patched directly.

### Certified Releases

A certified release is promoted from `main` only after the `staging → main` PR has been approved and merged. The maintainer then creates a semantic version tag on the `main` commit (e.g. `v1.0.12`) and publishes a GitHub Release referencing that tag. Downstream publication steps (Maven publishing, documentation deployment, release artifact packaging) are driven off the tag event to ensure only certified commits are released.

**Summary of branch roles:**

| Branch / Tag | Purpose |
|---|---|
| Feature branch | Development and local iteration |
| `staging` | Integration, verification, and CI gate |
| `main` | Certified, maintainer-approved code |
| Tag / Release | Public distribution |

---

## 5. Core Workflow: `make-all.yml`

### What It Does

`make-all.yml` runs the authoritative regression entrypoint for an IP repo:

1. Checks out the caller repository
2. Runs `make all` (or a specified target)
3. Evaluates a standardized regression result file (`error.rpt`)
4. Uploads diagnostic artifacts
5. Fails the job deterministically on regression errors

### Why `make all`

The Makefile is treated as the single source of truth for regression behavior. CI does not re-encode build logic — developers and CI execute the same entrypoint. Toolchain complexity stays in Make, not YAML.

### Regression Success Criteria

After `make all` completes, the workflow evaluates `error.rpt` in the repository root. This file is the standardized regression result across all IP repositories.

Rules:

- The file must exist
- The file must be zero bytes (no errors)

| Condition | Result |
|---|---|
| File exists and is empty | Regression passed |
| File is missing | Regression failed |
| File is non-empty | Regression failed; contents explain why |

The file is always uploaded as a CI artifact for inspection, regardless of pass/fail outcome.

---

## 6. Self-Hosted Runner Model

### Why Self-Hosted Runners

EDA toolchains are too large and specialized for GitHub-hosted runners. Self-hosted runners provide:

- Full control over EDA, synthesis, LaTeX, and timing tools
- An identical environment between local regression and CI
- No per-minute billing for long-running jobs

### Current Execution Model

Jobs run on organization-level self-hosted runners selected by label, not by hostname:

```yaml
runner_labels: '["self-hosted", "local-ci"]'
```

This decouples workflow YAML from specific machines, allowing runners to be added, replaced, or scaled without modifying caller workflows.

---

## 7. Security Model

### 7.1 Public Repository

`ci-workflows` is intentionally public.

- GitHub has known resolution issues with private reusable workflow repositories
- No secrets are stored in this repository
- Public visibility improves transparency and auditability
- Simplifies reuse across many repositories

Secrets remain in organization secrets, repository secrets, and runner environments — never in workflow YAML.

### 7.2 Action Usage Policy

The organization enforces that only actions owned by `chiselWare` may be used. GitHub-owned actions are mirrored under the org:

- `chiselWare/actions-checkout`
- `chiselWare/actions-upload-artifact`

All referenced actions are therefore org-owned. This satisfies strict security policies without requiring GitHub Enterprise. Mirrored repositories are public and contain no secrets.

### 7.3 Public Repositories and Self-Hosted Runners

By default, GitHub does not allow self-hosted runners to execute jobs from public repositories, preventing untrusted code from running on private infrastructure. During development and certification, IP repositories remain private. After certification they may be made public. This policy is enforced at the runner group level, not in workflow YAML.

---

## 8. CI Feedback to Developers

Developers receive feedback through:

1. **PR check status** — pass or fail, visible on the pull request
2. **Live workflow logs** — full output of `make all`
3. **Uploaded artifacts** — `error.rpt` and any additional reports

CI does not post comments or modify pull requests automatically. Human review remains the final gate before merging to `main`.

---

## 9. Role of IP Repositories

Each IP repository:

- Contains its own Makefile and regression harness
- Calls shared workflows from `ci-workflows@v1`
- Passes only minimal configuration (make target, runner labels)
- Does not duplicate CI logic

This keeps IP repositories lightweight, independent, and easy for external maintainers to understand.
