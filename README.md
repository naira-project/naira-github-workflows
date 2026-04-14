## About this project

**Naira** is an open-source **AI Engineering Development Platform** for cloud-native teams building and operating AI-enabled products on Kubernetes.

AI engineering today is fragmented: models, inferencing, gateways, observability, policies, and application delivery are spread across many tools, teams, and workflows. Naira brings these worlds together into one coherent platform experience.

Naira helps teams:

- **Discover and manage AI assets** across models, datasets, inference endpoints, and integrations  
- **Orchestrate AI platform workflows** using an opinionated, extensible architecture  
- **Improve reliability and governance** with unified visibility, policy integration, and auditability  
- **Accelerate delivery** through reusable templates, golden paths, and ecosystem plugins  
- **Avoid lock-in** by integrating existing best-of-breed tools instead of replacing them  

Built with and for the cloud-native ecosystem, Naira is designed to integrate closely with technologies such as **PlatformMesh**, **OpenMFP/Luigi**, **KCP**, and other components of the NeoNephos stack.

# Naira Shared CI/CD Workflows

Central repository for reusable GitHub Actions workflows used across all Naira project repositories.

## Repository Structure

```
naira-github-workflows/
├── .github/
│   ├── workflows/                  # Reusable workflows (called by other repos)
│   │   ├── reusable-go-lint.yml
│   │   ├── reusable-node-lint.yml
│   │   ├── reusable-go-test.yml
│   │   ├── reusable-dco-reuse.yml
│   │   ├── reusable-container-build.yml
│   │   ├── reusable-helm-publish.yml
│   │   ├── reusable-security-scan.yml
│   │   └── reusable-release.yml
│   └── actions/                    # Composite actions (shared steps)
│       ├── setup-go/
│       ├── setup-node/
│       └── docker-meta/
├── examples/                       # Caller workflow examples per repo type
│   ├── go-service/
│   ├── node-service/
│   └── helm-chart/
└── docs/
    ├── usage.md
    └── troubleshooting.md
```

## How to Use in Your Repository

All reusable workflows are called using:

```yaml
uses: naira-project/naira-github-workflows/.github/workflows/<workflow-name>.yml@main
```

Refer to the `examples/` directory for complete caller workflow templates per service type.

## Permissions Required

Each consuming repository must have the following settings enabled:
- **Settings → Actions → General**: Allow GitHub Actions, allow reusable workflows
- **Settings → Secrets**: Org-level secrets are inherited automatically

## Org-Level Secrets Required

| Secret Name | Purpose |
|---|---|
| `REGISTRY_TOKEN` | ghcr.io push access (fallback if GITHUB_TOKEN insufficient) |
| `SLACK_WEBHOOK_URL` | Pipeline failure notifications |

## Contributing

All changes to shared workflows require:
1. A PR with at least one reviewer
2. Testing against a sample consumer repo
3. Version tag bump following semver (`v1.2.3`)

## Support, Feedback, Contributing

This project is open to feature requests/suggestions, bug reports etc. via [GitHub issues](https://github.com/naira-project/naira-github-workflows/issues). Contribution and feedback are encouraged and always welcome. For more information about how to contribute, the project structure, as well as additional contribution information, see our [Contribution Guidelines](CONTRIBUTING.md).

## Security / Disclosure
If you find any bug that may be a security problem, please follow our instructions at [in our security policy](https://github.com/naira-project/naira-github-workflows/security/policy) on how to report it. Please do not create GitHub issues for security-related doubts or problems.

## Code of Conduct

Please refer to our [Code of Conduct](https://github.com/naira-project/.github/blob/main/CODE_OF_CONDUCT.md) for information on the expected conduct for contributing to Platform Mesh.

<p align="center"><img alt="Bundesministerium für Wirtschaft und Energie (BMWE)-EU funding logo" src="https://apeirora.eu/assets/img/BMWK-EU.png" width="400"/></p>