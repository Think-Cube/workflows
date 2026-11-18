# workflows

Reusable GitHub Actions workflows for Think-Cube repositories.

## Workflows

| Workflow | Purpose |
|---|---|
| [`container-image.yml`](.github/workflows/container-image.yml) | Lint, test, scan, sign and publish a multi-arch container image |
| [`helm-chart.yml`](.github/workflows/helm-chart.yml) | Lint, validate, install, sign and publish a Helm chart as an OCI artifact |

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

## Versioning

Releases are tagged `vMAJOR.MINOR.PATCH`. Callers pin an exact tag, and Dependabot in the calling repository proposes upgrades. A new major version means callers must change their inputs, secrets or permissions.

Actions inside the workflows are pinned to commit SHAs and updated by Dependabot.

## License

This project is licensed under the [MIT License](LICENSE).
