# Branching and Promotion Model

This organization uses a **feature → staging → main** branching model to balance developer autonomy with strong quality gates. Day-to-day work happens on short-lived **feature branches** created from `main` (or from the current development baseline). Developers are free to iterate and run local regressions as often as needed. Feature branches are not merged directly to `main`; instead, they are promoted via staging, which acts as the integration and verification gate for a repository.

When a developer believes a change is ready, they open a **pull request targeting** `staging`. Merging into staging triggers the **authoritative CI regression** (currently make all via the shared ci-workflows@v1 framework on approved runners). CI must complete successfully, including generation of the standardized `modules/<module_name>/generated/error.rpt`. Any failures are surfaced through PR check status, workflow logs, and uploaded artifacts (e.g., `error.rpt`, reports, PDFs) so the maintainer can diagnose failures quickly. `staging` is therefore the place where toolchain-heavy synthesis/timing/doc checks run consistently, independent of individual developer machines.

Once `staging` is green, promotion to `main` is done via a **second pull request:** `staging → main`, which is treated as the final gate. This PR is reviewed by admins/maintainers (and optionally additional automated checks such as AI design review). The intent is that `main` always reflects the last known-good, maintainers-approved state. If review raises issues, changes are made on a feature branch and re-promoted through `staging` again; `main` remains protected from direct pushes and unreviewed merges.

# Certified Release Procedure

A certified release is promoted from `main` only after `staging` has passed CI and the `staging → main` PR has been approved and merged. After merge, the maintainer creates a semantic version tag on the `main` commit (e.g., v1.0.12) and publishes a GitHub Release referencing that tag. The tag is the immutable marker that the code at that exact commit has passed the full regression gate and has been human-reviewed. Downstream publication steps (e.g., Maven publishing, documentation deployment, or release artifact packaging) should be driven off the tag event to ensure that only certified commits are released.

This separation of concerns keeps the workflow crisp: **feature branches** are for development, **staging** is for verification, **main** is for certified code, and **tags/releases** are for public distribution.

# CI Workflows Architecture

This repository defines the *shared CI/CD workflow framework* for the chiselWare organization.  It is designed to support many independent IP repositories, each with complex toolchains and regression harnesses, while keeping CI behavior consistent, reviewable, and secure.

## 1. Purpose of ci-workflows

`ci-workflows` acts as a versioned CI platform, not a product repository.

It provides:
- Reusable GitHub Actions workflows (workflow_call)
- A stable CI interface used by many repos
- Centralized enforcement of CI behavior and security policy

A clean path from local/self-hosted execution or to cloud execution (e.g. Azure)

Individual IP repositories (e.g. 00-000-dff) do not implement CI logic themselves.
They only call workflows defined here.

## 2. Repository Structure

```
ci-workflows/
├── .github/
│   └── workflows/
│       └── make-all.yml        # Primary regression workflow (v1 API)
├── docs/
│   ├── AdminSetup.md           # CI admin & org configuration notes
│   └── Developer.md            # Expectations for repo maintainers
└── README.md
```

## 3. Versioning Model (Intentional Simplicity)

### CI workflows are versioned as major-only APIs

- main → active development
- v1 → stable CI contract
- v2 → future breaking changes (when needed)

Example usage from an IP repo:

`uses: chiselWare/ci-workflows/.github/workflows/make-all.yml@v1`

### Why not semantic versioning?

- CI workflows are infrastructure APIs, not user-facing products
- Changes are usually either compatible or breaking
- Major-only versioning prevents mass breakage across many repos
- Avoids confusion with IP repo semantic versions (v1.0.12, etc.)

This mirrors how GitHub versions its own actions (e.g. actions/checkout@v4).

## 4. Core Workflow: `make-all.yml`

### What it does

`make-all.yml` runs the authoritative regression entrypoint for an IP repo:

- Checks out the caller repository
- Runs make all (or a specified target)
- Evaluates a standardized regression result file
- Uploads diagnostics artifacts
- Fails the job deterministically on regression errors

### Why `make all`

The Makefile is treated as the single source of truth for regression behavior:

- CI does not re-encode build logic
- Developers and CI execute the same entrypoint
- Toolchain complexity stays in Make, not YAML

## 5. Regression Success Criteria

The `Makefile` in each IP repository will generate a number of artifacts with any errors summarized in:

`modules/<module_name>/generated/error.rpt`

This file will be interrogated by the workflow to insure the regression passes.

Rules:

- The file must exist
- The file must be zero bytes (no errors)

If:
- the file is missing → regression failed
- the file is non-empty → regression failed, contents explain why

This allows:

- Simple pass/fail logic
- Rich diagnostics when failures occur
- Stable CI behavior across different toolchains

The file is always uploaded as a CI artifact for inspection.

## 6. Self-Hosted Runner Model

### Why self-hosted runners

- Toolchains are too large and specialized for GitHub-hosted runners
- Full control over EDA, synthesis, LaTeX, timing tools, etc.
- Identical environment between local regression and CI

### Current execution model

- Jobs run on organization-level self-hosted runners
- Runners are labeled (e.g. `self-hosted`, `local-ci`)
- CI workflows select runners by label, not by hostname

# 7. Security Model and Decisions

### 7.1 Public ci-workflows Repository

ci-workflows is intentionally public.

#### Reasons:
- GitHub has known resolution issues with private reusable workflow repos
- No secrets are stored in this repository
- Improves transparency and auditability
- Simplifies reuse across many repos

#### 
Secrets remain in:
- Organization secrets
- Repository secrets
- Runner environments

### 7.2 Action Usage Policy (Org-Owned Actions)

#### The organization enforces:

- Only actions owned by chiselWare may be used

#### Implications:

GitHub-owned actions (e.g. actions/checkout) are mirrored. Mirrored repos include:
- chiselWare/actions-checkout
- chiselWare/actions-upload-artifact

All referenced actions are therefore org-owned.  This satisfies strict security policies without requiring GitHub Enterprise.  Mirrored action repositories are public and contain no secrets.

### 7.3 Public Repos vs Self-Hosted Runners

By default, self-hosted GitHub runners do not run jobs from public repositories. This prevents untrusted code from executing on private infrastructure.

During development or certification, repos remain private. After certification, the repos can remain private or be made public.

This policy is enforced at the runner group level, not in YAML.

## 8. CI Feedback to Developers

Developers receive feedback through:

1. **PR check status** (pass/fail)
2. **Live logs** from `make all`
3. **Artifacts** (including error.rpt)

CI does not post comments or modify PRs automatically. Human review remains the final gate before merging to main.

## 9. Role of IP Repositories (e.g. 00-000-dff)

Each IP repository:
- Contains its own Makefile and regression harness
- Calls shared workflows from `ci-workflows@v1`
- Passes only minimal configuration (module name, targets)
- Does **not** duplicate CI logic

This keeps IP repos:
- Lightweight
- Independent
- Easy for external maintainers to understand