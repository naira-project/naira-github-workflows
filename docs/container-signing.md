<!--
SPDX-FileCopyrightText: 2026 Naira Project Contributors
SPDX-License-Identifier: Apache-2.0
-->

# Container Image Signing & Verification

## Prerequisites:
```
brew install cosign   # macOS
# or: https://docs.sigstore.dev/cosign/system_config/installation/
```

## Verify a specific release (by digest — most secure):
```
cosign verify \
  --certificate-identity-regexp \
    "https://github.com/naira-project/naira-github-workflows/.github/workflows/reusable-container-build.yml@.*" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/naira-project/naira-catalog@sha256:<digest>
```

## Verify by tag (tag is mutable, prefer digest):
```
cosign verify \
  --certificate-identity-regexp \
    "https://github.com/naira-project/naira-github-workflows/.github/workflows/reusable-container-build.yml@.*" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/naira-project/naira-catalog:v1.2.3
```

## Fetch and inspect the certificate:
```
cosign verify ... | jq -r '.[0].optional | {Issuer, Subject, "GitHub Workflow": .Workflow}'
```