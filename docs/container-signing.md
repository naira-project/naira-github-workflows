<!--
SPDX-FileCopyrightText: 2026 Naira Project Contributors
SPDX-License-Identifier: Apache-2.0
-->

# Container Image Signing, Provenance & Verification

Published container images are signed with keyless Cosign and receive a SLSA
build provenance attestation. The provenance binds the image digest to the
source repository, source commit, workflow identity, and build invocation. The
attestation is stored in GitHub's attestation store and pushed to the OCI
registry for the image.

## Prerequisites
```
brew install cosign   # macOS
# or: https://docs.sigstore.dev/cosign/system_config/installation/
gh auth login
```

## Verify a specific release by digest
```
cosign verify \
  --certificate-identity-regexp \
    "https://github.com/naira-project/naira-github-workflows/.github/workflows/reusable-container-build.yml@.*" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/naira-project/naira-catalog@sha256:<digest>
```

Digest-pinned verification is preferred because tags are mutable.

## Verify the SLSA provenance
```
gh attestation verify \
  oci://ghcr.io/naira-project/naira-catalog@sha256:<digest> \
  --repo naira-project/naira \
  --signer-workflow \
    naira-project/naira-github-workflows/.github/workflows/reusable-container-build.yml \
  --source-digest <release-commit-sha>
```

Use `--format json` when you need to inspect the full statement:
```
gh attestation verify \
  oci://ghcr.io/naira-project/naira-catalog@sha256:<digest> \
  --repo naira-project/naira \
  --signer-workflow \
    naira-project/naira-github-workflows/.github/workflows/reusable-container-build.yml \
  --source-digest <release-commit-sha> \
  --format json \
  --jq '.[].verificationResult.statement | {predicateType, subject}'
```

## Verify by tag
```
cosign verify \
  --certificate-identity-regexp \
    "https://github.com/naira-project/naira-github-workflows/.github/workflows/reusable-container-build.yml@.*" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/naira-project/naira-catalog:v1.2.3
```

## Fetch and inspect the certificate
```
cosign verify ... | jq -r '.[0].optional | {Issuer, Subject, "GitHub Workflow": .Workflow}'
```

## Workflow verification gate

Consumer release workflows should verify each pushed image before marking a
release successful:

```yaml
container-verify:
  name: Verify Container
  needs: container-publish
  uses: naira-project/shared-workflows/.github/workflows/reusable-container-verify.yml@main
  with:
    image-ref: "ghcr.io/${{ github.repository_owner }}/my-service@${{ needs.container-publish.outputs.image-digest }}"
    source-digest: ${{ github.sha }}
```
