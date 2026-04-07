# misc-actions

Shared GitHub Actions composite actions used across multiple projects.

## Actions

### [`ruff-lint`](actions/ruff-lint/action.yml)

Run Ruff linter and formatter checks using uv.

```yaml
- uses: ljmerza/misc-actions/actions/ruff-lint@main
  with:
    python-version: "3.12"        # optional, default: 3.12
    working-directory: "."        # optional, default: .
    uv-install-args: "--all-extras" # optional, default: --all-extras
```

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `python-version` | no | `3.12` | Python version to install |
| `working-directory` | no | `.` | Directory to lint |
| `uv-install-args` | no | `--all-extras` | Arguments for `uv sync` (e.g. `--all-extras` or `--group dev`) |

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
| `uv-install-args` | no | `--all-extras` | Arguments for `uv sync` |
| `env-vars` | no | `""` | Newline-separated `KEY=VALUE` env vars for tests |

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
