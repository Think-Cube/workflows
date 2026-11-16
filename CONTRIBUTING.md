# Contributing to workflows

Thank you for your interest in contributing!

## How to Contribute

1. **Fork** this repository.
2. **Create a branch**: `git checkout -b fix/description` or `feat/description`
3. **Make your changes** — see guidelines below.
4. **Open a Pull Request** using the PR template.

## Workflow Guidelines

- Pin every action to a full commit SHA with the version in a comment.
- Declare `permissions: {}` at the top and grant each job only what it needs.
- Pass `${{ }}` values to `run` steps through `env`, never inline them into the script.
- Do not use `secrets: inherit`; declare each secret the workflow needs.
- Document new inputs and secrets in the README.
- Changes to inputs, secrets or required permissions that break callers need a new major version.

## Testing

Every pull request runs `actionlint`. Before releasing, point a calling repository at your branch and check a full run.

## Reporting Issues

Use the issue templates:
- **Bug report** — something is broken or behaves unexpectedly.
- **Feature request** — suggest an improvement.

## Code of Conduct

By contributing, you agree to abide by our [Code of Conduct](CODE_OF_CONDUCT.md).
