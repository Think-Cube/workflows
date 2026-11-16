# workflows

Reusable GitHub Actions workflows for Think-Cube repositories.

## Workflows

| Workflow | Purpose |
|---|---|
| [`container-image.yml`](.github/workflows/container-image.yml) | Lint, test, scan, sign and publish a multi-arch container image |

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

## Versioning

Releases are tagged `vMAJOR.MINOR.PATCH`. Callers pin an exact tag, and Dependabot in the calling repository proposes upgrades. A new major version means callers must change their inputs, secrets or permissions.

Actions inside the workflows are pinned to commit SHAs and updated by Dependabot.

## License

This project is licensed under the [MIT License](LICENSE).
