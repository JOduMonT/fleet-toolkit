# fleet-toolkit

Reusable GitHub Actions workflows shared across this fleet's app and shared-infra repos. Two
audiences:

- **App-facing** — used by app/shared-infra repos directly:
  - `check-upstream-release.yml` — runs Renovate to open version-bump PRs. Caller needs a
    `renovate.json` (e.g. `{"extends": ["config:recommended"]}`).
  - `compose-lint.yml` — validates a docker-compose file's syntax.
  - `compose-smoke-test.yml` — starts the stack standalone, fails on any crashed container,
    optionally checks a health URL, tears down.
- **Fleet-facing** — used by the (private) fleet Hub, not by individual app repos:
  - `trigger-coolify-deploy.yml`
  - `verify-post-deploy-backup.yml`

## Usage

```yaml
jobs:
  lint:
    uses: JOduMonT/fleet-toolkit/.github/workflows/compose-lint.yml@v1
  smoke-test:
    uses: JOduMonT/fleet-toolkit/.github/workflows/compose-smoke-test.yml@v1
    with:
      health-url: http://localhost:PORT/
```

Pin to a tag (`@v1`), not `@main` — this repo's workflows can change behavior across
versions.
