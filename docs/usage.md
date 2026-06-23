# Using Naira Shared Workflows

This guide explains how to integrate the shared CI/CD workflows into any repository under the `naira-project` GitHub organization.

---

## Quick Start

### Step 1 — Copy the right example for your repo type

| Repo type | Example location |
|---|---|
| Go microservice | `examples/go-service/` |
| Node / frontend | `examples/node-service/` |
| Helm chart repo | `examples/helm-chart/` |

Copy the `.yml` files from the relevant example directory into your repo's `.github/workflows/` directory.

### Step 2 — Update the image name

In `release.yml` (and `pr-validation.yml` if overriding), change:

```yaml
image-name: "my-go-service"   # ← your service name here
```

### Step 3 — Add REUSE headers to all source files

Every file must carry an SPDX header:

```go
// SPDX-FileCopyrightText: 2026 Naira Project Contributors
// SPDX-License-Identifier: Apache-2.0
```

For files that cannot carry inline headers (JSON, generated files, assets), use `.reuse/dep5`. Copy the template from `examples/go-service/.reuse/dep5`.

The `LICENSES/` directory must contain `Apache-2.0.txt`. Download it from https://spdx.org/licenses/Apache-2.0.html.

### Step 4 — Configure branch protection

Go to **Settings → Branches → Add rule** for your `main` branch and enable:

- ✅ Require a pull request before merging
- ✅ Require status checks to pass before merging
  - Add `All Checks Passed` as the **only** required check (the gate job handles the rest)
- ✅ Require branches to be up to date before merging
- ✅ Require linear history
- ✅ Do not allow bypassing the above settings

### Step 5 — Enable GITHUB_TOKEN permissions

Go to **Settings → Actions → General → Workflow permissions** and select:
- **Read and write permissions**
- ✅ Allow GitHub Actions to create and approve pull requests

This is required for `release-please` to open Release PRs and for `ghcr.io` image pushes.

---

## Workflow Reference

### `reusable-dco-reuse.yml`

Enforces Developer Certificate of Origin on every commit and REUSE/SPDX license compliance.

```yaml
uses: naira-project/shared-workflows/.github/workflows/reusable-dco-reuse.yml@main
with:
  check-dco: true     # default: true
  check-reuse: true   # default: true
```

**DCO requirements:** Every commit must have a `Signed-off-by:` trailer. Contributors can add it automatically with:

```bash
git config --global commit.gpgsign false
git config --global format.signOff true
# Or per-commit:
git commit -s -m "feat: add thing"
```

---

### `reusable-go-lint.yml`

Runs `golangci-lint` and checks that `go.mod` / `go.sum` are tidy.

```yaml
uses: naira-project/shared-workflows/.github/workflows/reusable-go-lint.yml@main
with:
  go-version: "1.22"            # default: "1.22"
  working-directory: "."        # default: "."
  golangci-lint-version: "v1.57.2"
  timeout: "5m"
```

Copy `examples/go-service/.golangci.yml` to your repo root to use the standard Naira lint rules.

---

### `reusable-go-test.yml`

Runs `go test` with race detector and generates a coverage report.

```yaml
uses: naira-project/shared-workflows/.github/workflows/reusable-go-test.yml@main
with:
  go-version: "1.22"
  run-race-detector: true      # default: true
  coverage-threshold: 70       # fail if coverage < 70%  (0 = disabled)
  test-tags: ""                # e.g. "integration" to include tagged tests
```

**Outputs:**

| Output | Description |
|---|---|
| `coverage` | Coverage percentage string, e.g. `"82.3"` |

---

### `reusable-node-lint.yml`

Runs ESLint, TypeScript typecheck, and Prettier format check.

```yaml
uses: naira-project/shared-workflows/.github/workflows/reusable-node-lint.yml@main
with:
  node-version: "20"
  package-manager: "npm"        # npm | yarn | pnpm
  lint-script: "lint"           # maps to `npm run lint`
  typecheck-script: "typecheck" # set to "" to skip
```

---

### `reusable-container-build.yml`

Builds and optionally pushes multi-arch container images to `ghcr.io`.

```yaml
uses: naira-project/shared-workflows/.github/workflows/reusable-container-build.yml@main
with:
  image-name: "naira-api"            # optional, defaults to repo name
  dockerfile: "Dockerfile"           # default
  context: "."                       # default
  platforms: "linux/amd64,linux/arm64"
  push: true                         # false = build-only (for PRs)
  registry: "ghcr.io"                # default
  build-args: |
    VERSION=1.2.3
    COMMIT=abc1234
secrets:
  registry-password: ${{ secrets.GITHUB_TOKEN }}
```

**Outputs:**

| Output | Description |
|---|---|
| `image-digest` | `sha256:...` digest of the pushed image |
| `image-tags` | Comma-separated list of applied tags |
| `image-ref` | Digest-pinned image reference, e.g. `ghcr.io/org/app@sha256:...` |
| `attestation-url` | GitHub attestation summary URL for the image provenance |

Images are automatically tagged with:
- `latest` (on default branch)
- Semver (`1.2.3`, `1.2`, `1`) on version tags
- Branch name on branch pushes
- `pr-42` on pull requests
- `sha-abc1234` always (immutable)

When `push: true`, the workflow signs the image with keyless Cosign, generates
SLSA build provenance, pushes the attestation to the image registry, stores it
in GitHub's attestation store, and uploads the attestation bundle as a workflow
artifact named `provenance-<image-name>`.

The caller must grant these permissions for published images:

```yaml
permissions:
  contents: read
  packages: write
  id-token: write
  attestations: write
```

---

### `reusable-container-verify.yml`

Verifies a published image signature and its SLSA provenance attestation.

```yaml
uses: naira-project/naira-github-workflows/.github/workflows/reusable-container-verify.yml@main
with:
  image-ref: "ghcr.io/org/app@sha256:..."
  source-digest: ${{ github.sha }}
```

Set `verify-signature: false` or `verify-provenance: false` only when debugging
a partial release. Release gates should leave both enabled.

---

### `reusable-artifact-provenance.yml`

Generates provenance for non-container release artifacts that were previously
uploaded with `actions/upload-artifact`. It downloads the artifact, computes
`SHA256SUMS`, generates a SLSA provenance attestation over those checksums, and
uploads a sibling artifact containing:

- `SHA256SUMS`
- `build-inputs.json`
- the signed attestation bundle

```yaml
artifact-provenance:
  name: Generate Artifact Provenance
  needs: build-release-archive
  uses: naira-project/naira-github-workflows/.github/workflows/reusable-artifact-provenance.yml@main
  with:
    artifact-name: "release-archive"
    subject-path: "."
    build-inputs: |
      version=${{ needs.release.outputs.version }}
      commit=${{ github.sha }}
```

The caller must grant:

```yaml
permissions:
  contents: read
  id-token: write
  attestations: write
```

Operators can verify a downloaded file with:

```bash
gh attestation verify ./dist/my-archive.tar.gz \
  --repo naira-project/my-repo \
  --signer-workflow naira-project/naira-github-workflows/.github/workflows/reusable-artifact-provenance.yml \
  --source-digest <release-commit-sha>
```

---

### `reusable-helm-publish.yml`

Lints Helm charts with `chart-testing` and publishes to OCI or GitHub Pages.

```yaml
uses: naira-project/shared-workflows/.github/workflows/reusable-helm-publish.yml@main
with:
  charts-directory: "charts"
  helm-version: "v3.14.4"
  publish-method: "oci"          # "oci" or "gh-pages"
  lint-only: false               # true = lint only, no push (for PRs)
secrets:
  registry-password: ${{ secrets.GITHUB_TOKEN }}
```

For `gh-pages` method, create a `gh-pages` branch and enable GitHub Pages in repo settings.

---

### `reusable-security-scan.yml`

Runs Trivy (filesystem and/or image) and `govulncheck` for Go projects.

```yaml
uses: naira-project/shared-workflows/.github/workflows/reusable-security-scan.yml@main
with:
  scan-filesystem: true
  scan-image: false              # set true + image-ref after container is built
  image-ref: "ghcr.io/org/app:v1.2.3"
  run-govulncheck: true          # Go repos only
  severity: "HIGH"               # CRITICAL | HIGH | MEDIUM | LOW
  upload-sarif: true             # uploads to GitHub Security tab
permissions:
  security-events: write
  contents: read
```

Results appear in the **Security → Code scanning** tab of your repository.

---

### `reusable-release.yml`

Uses `release-please` to maintain a rolling Release PR and cut GitHub Releases on merge.

```yaml
uses: naira-project/shared-workflows/.github/workflows/reusable-release.yml@main
with:
  release-type: go               # go | node | helm | simple
  release-branch: main
  config-file: release-please-config.json      # optional; enables changelog-sections
  manifest-file: .release-please-manifest.json # optional
  notify-slack: true
secrets:
  github-token: ${{ secrets.GITHUB_TOKEN }}
  slack-webhook: ${{ secrets.SLACK_WEBHOOK_URL }}
```

Customize changelog grouping in `release-please-config.json` with `changelog-sections`.
The previous `changelog-types` workflow input is deprecated and ignored because
`googleapis/release-please-action@v4` does not support it as an action input.

**How release-please works:**

1. Every merge to `main` that follows Conventional Commits bumps `CHANGELOG.md` and opens/updates a Release PR.
2. When you merge the Release PR, `release-please` creates a GitHub Release and git tag.
3. Downstream jobs (`container-publish`, `helm-publish`) trigger off `release-created == 'true'`.

**Commit message prefixes:**

| Prefix | Effect |
|---|---|
| `feat:` | Minor version bump |
| `fix:` | Patch version bump |
| `feat!:` or `BREAKING CHANGE:` | Major version bump |
| `chore:`, `docs:`, `ci:` | No version bump |

---

## Pinning to a Specific Version

For stability, pin consuming repos to a tag rather than `@main`:

```yaml
uses: naira-project/shared-workflows/.github/workflows/reusable-go-lint.yml@v1.2.0
```

The shared-workflows repo itself follows semver. Check [Releases](../../releases) for the latest tag.

---

## Org-Level Secrets

The following secrets must be configured at the **organization level** (Settings → Secrets → Actions):

| Secret | Required by | Notes |
|---|---|---|
| `SLACK_WEBHOOK_URL` | `reusable-release.yml` | Incoming Webhook URL from Slack App |
| `CODECOV_TOKEN` | `reusable-go-test.yml` | Optional — only if using Codecov |

`GITHUB_TOKEN` is auto-provisioned by GitHub and does not need manual configuration.
