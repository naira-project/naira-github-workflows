# CI/CD Troubleshooting Runbook

Common failure modes and how to fix them. Keep this updated as new issues are encountered.

---

## DCO Check Failures

### Symptom
```
Error: Commit abc1234 is missing Signed-off-by trailer
```

### Cause
A commit in the PR does not have a `Signed-off-by: Name <email>` line.

### Fix

**Option A — Amend the last commit:**
```bash
git commit --amend --signoff
git push --force-with-lease
```

**Option B — Sign off all commits in a branch at once:**
```bash
# Rebase and add sign-off to every commit since branching from main
git rebase --signoff origin/main
git push --force-with-lease
```

**Option C — Prevent this in the future:**
```bash
# Auto sign-off every commit globally
git config --global format.signOff true
```

---

## REUSE Compliance Failures

### Symptom
```
reuse lint
✗ Missing license header: src/config/db.go
✗ Missing license in LICENSES/: MIT
```

### Fix — Missing header in a source file
Add the SPDX header to the top of the file:

```go
// SPDX-FileCopyrightText: 2026 Naira Project Contributors
// SPDX-License-Identifier: Apache-2.0
```

### Fix — Missing header in a file that cannot carry one (JSON, binary)
Add a bulk declaration to `.reuse/dep5`:

```
Files: path/to/file.json
Copyright: 2026 Naira Project Contributors
License: Apache-2.0
```

### Fix — Missing license text
Download the license text from https://spdx.org/licenses/ and save it as `LICENSES/<SPDX-ID>.txt`, e.g., `LICENSES/MIT.txt`.

### Running REUSE locally
```bash
pip install reuse
reuse lint            # Check compliance
reuse addheader --copyright "2026 Naira Project Contributors" \
                --license "Apache-2.0" src/myfile.go
```

---

## golangci-lint Failures

### Symptom
```
src/service.go:42:5: Error return value of `db.Close` is not checked (errcheck)
```

### Fix
Handle or explicitly ignore the error:
```go
// Handle it
if err := db.Close(); err != nil {
    log.Warn("failed to close db", "err", err)
}

// OR explicitly discard (add a comment explaining why)
_ = db.Close() // best-effort cleanup, already in error path
```

### Suppressing a false positive (use sparingly)
```go
//nolint:errcheck // reason: this function never returns a meaningful error
db.Close()
```

### go.mod / go.sum out of sync
```
Error: go.mod or go.sum is not tidy. Run 'go mod tidy' and commit.
```

```bash
go mod tidy
git add go.mod go.sum
git commit -s -m "chore: tidy go modules"
```

---

## Container Build Failures

### Symptom: QEMU / platform errors
```
error: failed to solve: failed to read dockerfile: ...
```

Ensure your `Dockerfile` uses a base image that supports the target platform:
```dockerfile
# Use multi-platform base images
FROM --platform=$BUILDPLATFORM golang:1.22-alpine AS builder
ARG TARGETOS TARGETARCH
RUN GOOS=$TARGETOS GOARCH=$TARGETARCH go build ...

FROM gcr.io/distroless/static:nonroot
```

### Symptom: ghcr.io push permission denied
```
denied: permission_denied: write_package
```

1. Check **Settings → Actions → General → Workflow permissions** is set to **Read and write**.
2. For the first push of a new package, ensure the package visibility is set to the org in `ghcr.io`.
3. Verify `secrets.GITHUB_TOKEN` is passed correctly — do not hardcode tokens.

### Symptom: Cache miss making builds slow
GHA cache is scoped to the branch. First run on a new branch will be a cache miss; subsequent runs will be fast. This is expected.

---

## Helm Chart Failures

### Symptom: ct lint fails
```
Error: chart [charts/my-chart] failed linting
```

Run locally to debug:
```bash
helm lint charts/my-chart --strict
ct lint --chart-dirs charts --target-branch main
```

Common fixes:
- Ensure `Chart.yaml` has `description`, `version`, and `appVersion` fields
- Add a `values.schema.json` to catch invalid value types
- Ensure chart version follows semver

### Symptom: OCI push fails on Helm chart
```
Error: failed to push: unexpected status code 403
```

Ensure you are logged in to the OCI registry:
```bash
echo $GITHUB_TOKEN | helm registry login ghcr.io \
  --username $GITHUB_ACTOR --password-stdin
```

---

## Security Scan Failures

### Symptom: Trivy finds a vulnerability
```
CRITICAL CVE-2026-XXXXX in golang.org/x/net v0.17.0
```

**Steps:**
1. Check if the CVE affects your code path: `govulncheck ./...`
2. Update the vulnerable dependency: `go get golang.org/x/net@latest && go mod tidy`
3. If no fix is available, add a `.trivyignore` with justification and expiry:

```
# CVE-2026-XXXXX: No fix available yet. Re-evaluate by YYYY-MM-DD.
CVE-2026-XXXXX
```

### Symptom: govulncheck reports a vulnerability but it's a false positive
If the vulnerable function is not in your call graph, `govulncheck` will report it as informational only and not fail. If it does fail, the vulnerable symbol is reachable and must be addressed.

---

## release-please Not Creating a Release PR

### Causes and checks:

1. **Commits do not follow Conventional Commits format.**
   Ensure commit messages start with `feat:`, `fix:`, etc. Commits like `"Updated stuff"` are ignored.

2. **Workflow is not triggered on `push` to `main`.**
   Verify the `release.yml` has:
   ```yaml
   on:
     push:
       branches: [main]
   ```

3. **Token lacks `pull-requests: write` permission.**
   Check workflow permissions in **Settings → Actions → General**.

4. **release-please-config.json is missing** (for multi-package repos).
   Add a config file for monorepos — see [release-please docs](https://github.com/googleapis/release-please).

---

## Pipeline Timing Out

### Long matrix builds
- Ensure `cache-from: type=gha` is set in `docker/build-push-action`
- Split test packages with `-run` filters across parallel jobs if test suite is large
- Use `timeout-minutes` per job:
  ```yaml
  jobs:
    go-test:
      timeout-minutes: 15
  ```

### Stuck workflow (no logs)
This is usually a GitHub Actions infrastructure issue. Re-run the job from the UI or push an empty commit:
```bash
git commit --allow-empty -s -m "ci: re-trigger pipeline"
git push
```

---

## Getting Help

1. Check the [GitHub Actions run logs](../../actions) for the exact error.
2. Search this runbook.
3. Open an issue in `naira-project/shared-workflows` with:
   - The repo and workflow name
   - A link to the failed run
   - The exact error message
