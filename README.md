# workflows

Reusable GitHub Actions workflows for Think-Cube repositories.

## Workflows

| Workflow | Purpose |
|---|---|
| [`container-image.yml`](.github/workflows/container-image.yml) | Lint, test, scan, sign and publish a multi-arch container image |
| [`helm-chart.yml`](.github/workflows/helm-chart.yml) | Lint, validate, install, sign and publish a Helm chart as an OCI artifact |
| [`terraform-module.yml`](.github/workflows/terraform-module.yml) | Lint, scan and test a Terraform module |
| [`version-check.yml`](.github/workflows/version-check.yml) | Open an issue when versions Dependabot does not track have newer releases |

## container-image

Every run lints the Dockerfile with Hadolint, then builds the image for each platform on a native runner, runs the repository's smoke test and fails on fixable vulnerabilities found by Trivy.

On `main` and on `v*` tags it also builds a multi-arch image with an SBOM and build provenance, pushes it to GitHub Container Registry, signs it with keyless Cosign and mirrors the same digest to Docker Hub.

### Usage

```yaml
name: Build

on:
  push:
    branches: [main]
    tags: ["v*.*.*"]
  pull_request:
    branches: [main]
  schedule:
    - cron: "17 4 * * 1"
  workflow_dispatch:

permissions: {}

jobs:
  image:
    permissions:
      contents: read
      id-token: write
      packages: write
    uses: Think-Cube/workflows/.github/workflows/container-image.yml@v1.0.0
    with:
      image-name: ansible
    secrets:
      DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
      DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
```

The calling job must grant all three permissions. A reusable workflow cannot use more than its caller allows.

### Inputs

| Input | Default | Description |
|---|---|---|
| `image-name` | (required) | Image name without registry or namespace |
| `context` | `.` | Build context |
| `dockerfile` | `Dockerfile` | Path to the Dockerfile |
| `platforms` | `["linux/amd64", "linux/arm64"]` | JSON array of platforms to build |
| `smoke-test` | `tests/smoke.sh` | Script run with the image reference as its only argument |
| `trivy-severity` | `CRITICAL` | Severities that fail the build |
| `dockerhub-namespace` | `thinkcubedev` | Docker Hub namespace of the mirror |

### Secrets

| Secret | Description |
|---|---|
| `DOCKERHUB_USERNAME` | Docker Hub user. The mirror is skipped when it is empty. |
| `DOCKERHUB_TOKEN` | Docker Hub access token with read and write scope |

GitHub Container Registry uses the job's `GITHUB_TOKEN`, so it needs no secret.

### Tags

| Event | Tags |
|---|---|
| Tag `v2.1.0` | `2.1.0`, `2.1`, `2`, `latest` |
| Push to `main` | `edge`, `sha-<commit>` |
| Schedule | `edge`, `edge-YYYYMMDD` |
| Pull request | Nothing is published |

A release fails instead of overwriting a version that already exists in GitHub Container Registry. Scheduled runs build without cache so the image picks up current OS patches.

### Verify a published image

```bash
cosign verify ghcr.io/think-cube/<image>:<version> \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity-regexp '^https://github\.com/Think-Cube/workflows/\.github/workflows/container-image\.yml@refs/tags/v.+$'
```

## helm-chart

Every run lints the chart with `helm lint --strict` and validates the rendered manifests with kubeconform: Kubernetes resources against the official schemas, custom resources such as KEDA's against the [CRDs catalog](https://github.com/datreeio/CRDs-catalog). It then installs the chart into a kind cluster, waits for it to become ready and runs `helm test`.

When the chart has `ci/*-values.yaml` files, each check runs once per file instead of with the default values. Use them to cover the configurations that matter, such as an optional component switched on.

When the chart has `ci/setup.sh`, it runs against the kind cluster before the installs. Use it to install what the chart depends on, such as an operator and its CRDs.

On `v*` tags it also packages the chart, pushes it to GitHub Container Registry as an OCI artifact and signs it with keyless Cosign. The tag must match `version` in `Chart.yaml`, and a release fails instead of overwriting a version that already exists. If the chart directory contains `artifacthub-repo.yml`, it is pushed as Artifact Hub repository metadata.

### Usage

```yaml
name: Chart

on:
  push:
    branches: [main]
    tags: ["v*.*.*"]
  pull_request:
    branches: [main]
  workflow_dispatch:

permissions: {}

jobs:
  chart:
    permissions:
      contents: read
      id-token: write
      packages: write
    uses: Think-Cube/workflows/.github/workflows/helm-chart.yml@v1.2.0
```

### Inputs

| Input | Default | Description |
|---|---|---|
| `chart-path` | `.` | Directory that contains `Chart.yaml` |
| `helm-version` | `v4.3.2` | Helm version to use |
| `kubernetes-version` | `1.34.12` | Kubernetes version the rendered manifests are validated against |
| `kind-node-image` | `kindest/node:v1.35.8@sha256:...` | kind node image the chart is installed into |
| `registry-path` | `charts` | Path under `ghcr.io/<owner>` the chart is pushed to |
| `setup-script` | `ci/setup.sh` | Script run against the kind cluster before installing; skipped when missing |
| `wait` | `watcher` | Helm wait strategy for the test install. Use `legacy` when the chart creates custom resources that only become ready with a real backend. |

### Verify a published chart

```bash
cosign verify ghcr.io/think-cube/charts/<chart>:<version> \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity-regexp '^https://github\.com/Think-Cube/workflows/\.github/workflows/helm-chart\.yml@refs/tags/v.+$'
```

## terraform-module

Checks a Terraform module that is consumed straight from GitHub, so nothing is published. The `lint` job checks `terraform fmt`, validates the module and every directory in `examples/`, runs TFLint with all rules of the Terraform ruleset and the azurerm ruleset, checks that the tables in the README match `terraform-docs` and scans for misconfigurations with Trivy. The `test` job runs `terraform test` once for every Terraform version, so the oldest supported version is tested too. It fails when the module has no tests.

Write the tests against mocked providers (`mock_provider`) so they need no cloud credentials.

### Usage

```yaml
name: Module

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: "41 4 * * 1"
  workflow_dispatch:

permissions: {}

jobs:
  module:
    permissions:
      contents: read
    uses: Think-Cube/workflows/.github/workflows/terraform-module.yml@v1.4.0
```

The scheduled run catches new provider releases that break the module before its users do. Require the `module / lint` and `module / test` checks in the repository's ruleset; `module / test` passes only when the tests pass on every Terraform version.

### Inputs

| Input | Default | Description |
|---|---|---|
| `terraform-versions` | `["1.14.9", "1.17.2"]` | JSON array of Terraform versions to test with; the first one also runs the lint job |
| `tflint-version` | `v0.66.0` | TFLint version |
| `tflint-azurerm-version` | `0.33.0` | azurerm ruleset version, used when the repository has no `.tflint.hcl` |
| `terraform-docs-version` | `v0.25.1` | terraform-docs version that checks the README |
| `trivy-severity` | `CRITICAL,HIGH` | Misconfiguration severities that fail the build |

A `.tflint.hcl` in the repository replaces the default configuration. The README needs `<!-- BEGIN_TF_DOCS -->` and `<!-- END_TF_DOCS -->` markers, and a `.terraform-docs.yml` sets how the tables look.

### Releases

Consumers pin a tag in the module source, for example `git::https://github.com/Think-Cube/terraform-azurerm-resource-group.git?ref=v1.0.0`, and Dependabot's `terraform` ecosystem proposes newer tags. A release is a `vX.Y.Z` tag with a GitHub release.

## version-check

Dependabot keeps base images, packages and actions current, but it cannot see versions pinned in build arguments, Helm values or Git commits, and it deliberately skips updates such as a new RabbitMQ minor. This workflow compares those pins with the latest releases and keeps a single issue, labelled `version-check`, up to date: it opens the issue when something is outdated, updates it as findings change and closes it once everything is current. It does not change any files, because these updates usually need a manual step.

### Usage

```yaml
name: Version check

on:
  schedule:
    - cron: "23 5 * * 1"
  workflow_dispatch:

permissions: {}

jobs:
  check:
    permissions:
      contents: read
      issues: write
    uses: Think-Cube/workflows/.github/workflows/version-check.yml@v1.3.0
```

### Configuration

List the checks in `.github/version-check.yml`:

```yaml
checks:
  - name: Azure Pipelines agent
    file: Dockerfile
    pattern: 'ARG AGENT_VERSION=(\S+)'
    source: github-release
    repository: microsoft/azure-pipelines-agent
    note: Update both checksums too.
  - name: curl used by helm test
    file: values.yaml
    path: .test.image.tag
    pattern: '^([^@]+)'
    source: docker-hub
    repository: curlimages/curl
    tag-pattern: '(\d+\.\d+\.\d+)'
```

| Key | Description |
|---|---|
| `name` | Shown in the issue |
| `file` | File that pins the version |
| `path` | Optional yq path to the value in a YAML file |
| `pattern` | Regular expression whose first group is the pinned version, applied to the `path` value or to the whole file |
| `source` | `github-release` (latest release tag), `github-commit` (head of the default branch) or `docker-hub` (highest matching tag) |
| `repository` | `owner/repo` on GitHub, or `namespace/name` on Docker Hub (`library/<name>` for official images) |
| `tag-pattern` | For `docker-hub`: tags to consider; its first group, if any, is the version compared |
| `note` | Optional instruction shown in the issue |

Versions are compared numerically, and commits by identity. A check that cannot run fails the workflow without affecting the others.

## Versioning

Releases are tagged `vMAJOR.MINOR.PATCH`. Callers pin an exact tag, and Dependabot in the calling repository proposes upgrades. A new major version means callers must change their inputs, secrets or permissions.

Actions inside the workflows are pinned to commit SHAs and updated by Dependabot.

## License

This project is licensed under the [MIT License](LICENSE).
