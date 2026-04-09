# misc-actions

Shared GitHub Actions composite actions and reusable workflows used across multiple projects.

## Prerequisites

Consuming repos/orgs must have **"Read and write permissions"** enabled for workflow tokens. Several workflows need write access to packages (GHCR image push/delete) and attestations.

**Organization-level** (applies to all repos in the org):
Settings → Actions → General → Workflow permissions → **Read and write permissions**

Or via the GitHub API:
```bash
gh api orgs/{ORG}/actions/permissions/workflow \
  --method PUT \
  --field default_workflow_permissions=write \
  --field can_approve_pull_request_reviews=false
```

Without this, workflows that require `packages: write`, `attestations: write`, or `id-token: write` will fail with a `startup_failure` before any jobs run.

## Image Tagging Flow

Every event produces a `sha-` tag so you can always trace an image back to the exact commit.

| Event | Image Tags | Notes |
|---|---|---|
| PR opened / new commit pushed | `pr-42`, `sha-abc1234` | Built in `ci.yml`, fork-protected |
| Merged to main | `main`, `sha-abc1234` | Built in `build-and-push.yml` |
| Git tag `v1.2.0` pushed | `v1.2.0`, `sha-abc1234` | Built in `build-and-push.yml` |
| GitHub release published | `latest`, `v1.2.0`, `sha-abc1234` | Built in `build-and-push.yml` |
| PR closed | `pr-42` **deleted** | Cleaned up by `cleanup-pr-image.yml` |

## Actions

### [`ruff-lint`](actions/ruff-lint/action.yml)

Run Ruff linter and formatter checks using uv.

```yaml
- uses: ljmerza/misc-actions/actions/ruff-lint@main
  with:
    python-version: "3.12"        # optional, default: 3.12
    working-directory: "."        # optional, default: .
```

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `python-version` | no | `3.12` | Python version to install |
| `working-directory` | no | `.` | Directory to lint |

> Requires `[dependency-groups] dev` in `pyproject.toml` with ruff as a dependency.

---

### [`eslint`](actions/eslint/action.yml)

Run ESLint via npm for frontend projects.

```yaml
- uses: ljmerza/misc-actions/actions/eslint@main
  with:
    working-directory: frontend
    node-version: "20"            # optional, default: 20
```

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `node-version` | no | `20` | Node.js version |
| `working-directory` | **yes** | — | Frontend directory to lint |

---

### [`python-test`](actions/python-test/action.yml)

Run pytest with coverage via uv and optionally upload to Codecov.

```yaml
- uses: ljmerza/misc-actions/actions/python-test@main
  with:
    coverage-source: backup
    codecov-token: ${{ secrets.CODECOV_TOKEN }}
    env-vars: |
      SECRET_KEY=test-secret-key
      DEBUG=True
```

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `python-version` | no | `3.12` | Python version |
| `working-directory` | no | `.` | Project root |
| `coverage-source` | **yes** | — | Comma-separated packages for `--cov=` |
| `codecov-token` | no | `""` | Codecov token (skip upload if empty) |
| `codecov-flags` | no | `""` | Codecov flags (e.g. `backend`) |
| `env-vars` | no | `""` | Newline-separated `KEY=VALUE` env vars for tests |

> Requires `[dependency-groups] dev` in `pyproject.toml` with pytest and pytest-cov as dependencies.

---

### [`frontend-test`](actions/frontend-test/action.yml)

Run frontend tests via npm and optionally upload coverage to Codecov.

```yaml
- uses: ljmerza/misc-actions/actions/frontend-test@main
  with:
    working-directory: frontend
    test-command: "npx vitest run --coverage"
    codecov-token: ${{ secrets.CODECOV_TOKEN }}
```

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `node-version` | no | `20` | Node.js version |
| `working-directory` | **yes** | — | Frontend directory |
| `test-command` | no | `npm test` | Test command to run |
| `codecov-token` | no | `""` | Codecov token (skip upload if empty) |
| `codecov-flags` | no | `frontend` | Codecov flags |
| `coverage-file` | no | `coverage/coverage-final.json` | Coverage output path relative to working dir |

---

### [`docker-build-push`](actions/docker-build-push/action.yml)

Build and push Docker image to a container registry using Buildx with multi-arch support.

```yaml
- uses: ljmerza/misc-actions/actions/docker-build-push@main
  id: build
  with:
    image-name: ghcr.io/${{ github.repository_owner }}/my-app
    registry-password: ${{ secrets.GITHUB_TOKEN }}
    tags: |
      type=raw,value=pr-${{ github.event.pull_request.number }}
      type=sha,prefix=sha-
```

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `image-name` | **yes** | — | Full image name |
| `dockerfile` | no | `./Dockerfile` | Dockerfile path |
| `context` | no | `.` | Build context |
| `platforms` | no | `linux/amd64,linux/arm64` | Target platforms |
| `build-target` | no | `""` | Docker build target stage |
| `push` | no | `true` | Whether to push |
| `tags` | **yes** | — | Tag rules for `docker/metadata-action` |
| `registry` | no | `ghcr.io` | Container registry URL |
| `registry-username` | no | `""` | Registry username (defaults to `github.actor`) |
| `registry-password` | **yes** | — | Registry password |
| `cache-type` | no | `gha` | Build cache type |
| `build-args` | no | `""` | Docker build arguments |

**Auto-injected build args:** The following build args are automatically injected into every build. User-provided `build-args` with the same key override the auto-injected values.

| Build Arg | Source | Example |
|---|---|---|
| `GIT_COMMIT` | Full commit SHA | `abc123def456789...` |
| `GIT_COMMIT_SHORT` | 7-char short SHA | `abc123d` |
| `BUILD_DATE` | UTC ISO 8601 timestamp | `2026-04-09T16:30:00Z` |
| `BUILD_REF` | Branch or tag name | `main` or `v1.2.0` |
| `BUILD_NUMBER` | Workflow run number | `42` |
| `BUILD_REPO` | Repository full name | `owner/repo` |

> Dockerfiles that don't declare matching `ARG` declarations will silently ignore these.

**Outputs:** `digest`, `tags`, `metadata`

---

### [`provenance-attest`](actions/provenance-attest/action.yml)

Generate SLSA build provenance attestation for a container image.

> **Note:** The calling workflow must declare `id-token: write` and `attestations: write` permissions.

```yaml
- uses: ljmerza/misc-actions/actions/provenance-attest@main
  with:
    image-name: ghcr.io/${{ github.repository_owner }}/my-app
    digest: ${{ steps.build.outputs.digest }}
```

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `image-name` | **yes** | — | Full image name |
| `digest` | **yes** | — | Image digest from build step |
| `push-to-registry` | no | `true` | Push attestation to registry |

---

### [`cleanup-pr-image`](actions/cleanup-pr-image/action.yml)

Delete a PR-tagged container image from GHCR.

```yaml
- uses: ljmerza/misc-actions/actions/cleanup-pr-image@main
  with:
    package-name: my-app
    tag: pr-${{ github.event.pull_request.number }}
    token: ${{ secrets.GITHUB_TOKEN }}
```

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `package-name` | **yes** | — | GHCR package name |
| `tag` | **yes** | — | Image tag to delete |
| `owner` | no | `""` | Package owner (defaults to `github.repository_owner`) |
| `token` | **yes** | — | GitHub token with `packages:write` |

---

## Reusable Workflows

These workflows compose the actions above into complete CI/CD pipelines. Consuming repos only need thin wrapper workflows.

### [`python-tests.yml`](.github/workflows/python-tests.yml)

Reusable test workflow for Python-only projects (ruff lint + pytest with coverage).

```yaml
jobs:
  tests:
    uses: ljmerza/misc-actions/.github/workflows/python-tests.yml@v2
    with:
      python-version: "3.13"
      coverage-source: backup
      env-vars: |
        SECRET_KEY=test-secret-key
        DEBUG=True
    secrets: inherit
```

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `python-version` | no | `3.12` | Python version |
| `coverage-source` | **yes** | — | Comma-separated packages for `--cov=` |
| `env-vars` | no | `""` | Newline-separated `KEY=VALUE` env vars for tests |

| Secret | Required | Description |
|--------|----------|-------------|
| `CODECOV_TOKEN` | no | Codecov upload token |

---

### [`fullstack-tests.yml`](.github/workflows/fullstack-tests.yml)

Reusable test workflow for full-stack projects (ruff + eslint + pytest + frontend tests).

```yaml
jobs:
  tests:
    uses: ljmerza/misc-actions/.github/workflows/fullstack-tests.yml@v2
    with:
      coverage-source: accounts,alarm,locks
      codecov-backend-flags: backend
      env-vars: |
        SECRET_KEY=ci
        DEBUG=True
      frontend-working-directory: frontend
      frontend-test-command: "npm test"
      codecov-frontend-flags: frontend
    secrets: inherit
```

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `python-version` | no | `3.12` | Python version |
| `coverage-source` | **yes** | — | Comma-separated packages for `--cov=` |
| `codecov-backend-flags` | no | `backend` | Codecov flags for backend coverage |
| `env-vars` | no | `""` | Newline-separated `KEY=VALUE` env vars for backend tests |
| `node-version` | no | `20` | Node.js version |
| `frontend-working-directory` | no | `frontend` | Frontend directory |
| `frontend-test-command` | no | `npm test` | Frontend test command |
| `codecov-frontend-flags` | no | `frontend` | Codecov flags for frontend coverage |
| `frontend-coverage-file` | no | `coverage/coverage-final.json` | Coverage output path |

| Secret | Required | Description |
|--------|----------|-------------|
| `CODECOV_TOKEN` | no | Codecov upload token |

---

### [`docker-ci.yml`](.github/workflows/docker-ci.yml)

Reusable workflow to build and push a PR Docker image. Fork-safe (skips builds from fork PRs).

Tags: `pr-{number}`, `sha-{hash}`

```yaml
jobs:
  build:
    needs: tests
    uses: ljmerza/misc-actions/.github/workflows/docker-ci.yml@v2
    with:
      image-name: ghcr.io/${{ github.repository_owner }}/my-app
    secrets: inherit
```

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `image-name` | **yes** | — | Full image name |
| `build-target` | no | `""` | Docker build target stage |
| `build-args` | no | `""` | Docker build arguments |

---

### [`docker-release.yml`](.github/workflows/docker-release.yml)

Reusable workflow to build, push, and attest a Docker image for main/release events.

Tags: `ref/tag`, `sha-{hash}`, `main` (if default branch), `latest` (if release)

```yaml
jobs:
  build:
    needs: tests
    permissions:
      contents: read
      packages: write
      attestations: write
      id-token: write
    uses: ljmerza/misc-actions/.github/workflows/docker-release.yml@v2
    with:
      image-name: ghcr.io/${{ github.repository_owner }}/my-app
    secrets: inherit
```

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `image-name` | **yes** | — | Full image name |
| `build-target` | no | `""` | Docker build target stage |
| `build-args` | no | `""` | Docker build arguments |

> **Note:** The calling job must declare `attestations: write` and `id-token: write` permissions.

---

### [`cleanup-pr-image.yml`](.github/workflows/cleanup-pr-image.yml)

Reusable workflow to delete a PR-tagged container image from GHCR when a PR is closed. Fork-safe.

```yaml
jobs:
  cleanup:
    uses: ljmerza/misc-actions/.github/workflows/cleanup-pr-image.yml@v2
    with:
      package-name: my-app
    secrets: inherit
```

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `package-name` | **yes** | — | GHCR package name |

---

## Full Workflow Examples

### Python/Django Project (backend only)

Only 3 files needed — no local `tests.yml` required.

**`.github/workflows/ci.yml`** (pull requests):
```yaml
name: CI

on:
  pull_request: {}

concurrency:
  group: ci-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  tests:
    permissions:
      contents: read
    uses: ljmerza/misc-actions/.github/workflows/python-tests.yml@v2
    with:
      coverage-source: myapp
      env-vars: |
        SECRET_KEY=test-secret-key
        DEBUG=True
    secrets: inherit

  build:
    needs: tests
    permissions:
      contents: read
      packages: write
    uses: ljmerza/misc-actions/.github/workflows/docker-ci.yml@v2
    with:
      image-name: ghcr.io/${{ github.repository_owner }}/my-app
    secrets: inherit
```

**`.github/workflows/build-and-push.yml`** (production):
```yaml
name: Build and Push Image

on:
  push:
    branches: [main]
  release:
    types: [published]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  tests:
    permissions:
      contents: read
    uses: ljmerza/misc-actions/.github/workflows/python-tests.yml@v2
    with:
      coverage-source: myapp
      env-vars: |
        SECRET_KEY=test-secret-key
        DEBUG=True
    secrets: inherit

  build:
    needs: tests
    permissions:
      contents: read
      packages: write
      attestations: write
      id-token: write
    uses: ljmerza/misc-actions/.github/workflows/docker-release.yml@v2
    with:
      image-name: ghcr.io/${{ github.repository_owner }}/my-app
    secrets: inherit
```

**`.github/workflows/cleanup-pr-image.yml`**:
```yaml
name: Cleanup PR Image

on:
  pull_request:
    types: [closed]

jobs:
  cleanup:
    uses: ljmerza/misc-actions/.github/workflows/cleanup-pr-image.yml@v2
    with:
      package-name: my-app
    secrets: inherit
```

---

### Full-Stack Project (Python backend + React frontend)

Same 3-file structure but uses `fullstack-tests.yml` and passes `build-target`.

**`.github/workflows/ci.yml`** (pull requests):
```yaml
name: CI

on:
  pull_request: {}

concurrency:
  group: ci-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  tests:
    permissions:
      contents: read
    uses: ljmerza/misc-actions/.github/workflows/fullstack-tests.yml@v2
    with:
      coverage-source: accounts,alarm,locks
      codecov-backend-flags: backend
      env-vars: |
        SECRET_KEY=ci
        DEBUG=True
      frontend-working-directory: frontend
      frontend-test-command: "npm test"
      codecov-frontend-flags: frontend
    secrets: inherit

  build:
    needs: tests
    permissions:
      contents: read
      packages: write
    uses: ljmerza/misc-actions/.github/workflows/docker-ci.yml@v2
    with:
      image-name: ghcr.io/${{ github.repository_owner }}/my-app
      build-target: production
    secrets: inherit
```

**`.github/workflows/build-and-push.yml`** (production):
```yaml
name: Build and Push Image

on:
  push:
    branches: [main]
  release:
    types: [published]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  tests:
    permissions:
      contents: read
    uses: ljmerza/misc-actions/.github/workflows/fullstack-tests.yml@v2
    with:
      coverage-source: accounts,alarm,locks
      codecov-backend-flags: backend
      env-vars: |
        SECRET_KEY=ci
        DEBUG=True
      frontend-working-directory: frontend
      frontend-test-command: "npm test"
      codecov-frontend-flags: frontend
    secrets: inherit

  build:
    needs: tests
    permissions:
      contents: read
      packages: write
      attestations: write
      id-token: write
    uses: ljmerza/misc-actions/.github/workflows/docker-release.yml@v2
    with:
      image-name: ghcr.io/${{ github.repository_owner }}/my-app
      build-target: production
    secrets: inherit
```

**`.github/workflows/cleanup-pr-image.yml`**:
```yaml
name: Cleanup PR Image

on:
  pull_request:
    types: [closed]

jobs:
  cleanup:
    uses: ljmerza/misc-actions/.github/workflows/cleanup-pr-image.yml@v2
    with:
      package-name: my-app
    secrets: inherit
```
