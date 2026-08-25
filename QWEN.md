# QWEN.md — fleet-toolkit

## Project Overview

A collection of reusable GitHub Actions workflows shared across a small self-hosted app fleet. The workflows are invoked via `workflow_call` from app/shared-infra repos — there is no application source code in this repo, only YAML workflow definitions.

- **Owner:** `JOduMonT` (GitHub)
- **License:** GPL-3.0
- **Deployment target:** Coolify (self-hosted PaaS)
- **Stack:** Docker Compose, GitHub Actions, Renovate

## Workflow Inventory

### App-facing (called by individual app/shared-infra repos)

| Workflow | Purpose | Key inputs |
|---|---|---|
| `check-upstream-release.yml` | Runs Renovate against the calling repo to open version-bump PRs | None (caller needs `renovate.json`) |
| `compose-lint.yml` | Validates docker-compose file syntax | `compose-file`, `env-file` |
| `compose-smoke-test.yml` | Starts the stack standalone, detects crashed containers, optionally checks a health URL, tears down | `compose-file`, `env-file`, `health-url` |

### Fleet-facing (called by the private fleet Hub, not by individual app repos)

| Workflow | Purpose | Key inputs/secrets |
|---|---|---|
| `trigger-coolify-deploy.yml` | Triggers a Coolify deploy and polls until it reaches a terminal state | `app-uuid`, `coolify-url`, `coolify-api-token` |
| `verify-post-deploy-backup.yml` | Polls Coolify until the app is stably running post-deploy | `app-uuid`, `coolify-url`, `coolify-api-token` |

## Consumer Usage Pattern

```yaml
jobs:
  lint:
    uses: JOduMonT/fleet-toolkit/.github/workflows/compose-lint.yml@v1
  smoke-test:
    uses: JOduMonT/fleet-toolkit/.github/workflows/compose-smoke-test.yml@v1
    with:
      health-url: http://localhost:PORT/
```

Pinned to `@v1` (floating tag), not `@main`. The `v1` tag is moved forward deliberately when bugs are fixed.

## Critical Gotchas (from CLAUDE.md — do not regress these)

1. **`docker compose ps` needs `-a`** — Without `-a`, only running containers appear. A stack where every container crashed reports zero problems. The smoke test uses `ps -a` intentionally.

2. **Single-sample health checks false-positive** — A crash-looping container can read as `running:healthy` for one poll then die again. `verify-post-deploy-backup.yml` requires 3 consecutive `running:*` reads (streak resets on any non-running read).

3. **Triggering a Coolify deploy only queues it** — The API returns immediately. The trigger workflow captures `deployment_uuid` and polls `GET /api/v1/deployments/<uuid>` until `finished` or `failed`. App health alone is insufficient because Coolify keeps the old container running on failed deploys.

4. **Caller `permissions:` can only narrow, never widen** — `check-upstream-release.yml` declares `contents: write`, `pull-requests: write`, `issues: write`. If a calling repo restricts further, Renovate silently does nothing (exits 0, scans zero repos). Callers need `RENOVATE_REPOSITORIES` env var and `issues: write` at the repo level for the Dependency Dashboard.

5. **Env-file is required for lint** — `compose-lint.yml` copies the env-file to `.env` before running `docker compose config -q` because variable interpolation in compose files needs real (or dummy) values present.

## Development Conventions

- **App-facing workflows must stay generic** — No repo-specific hardcoding; parameterized entirely via `inputs:`.
- **Extensive inline comments explain the *why*** — Each workflow has detailed comments documenting the specific bug or failure mode that drove the current implementation. Do not strip these when editing.
- **`set -euo pipefail` in all shell steps** — Consistent strict bash error handling throughout.
- **Tear down on `always()`** — `compose-smoke-test.yml` runs `docker compose down -v` in an `if: always()` step to clean up even on failure.
- **Versioning via tag force-push** — `@v1` is a floating tag. This is intentional for this repo's current scope (private fleet, no external consumers). Security classifiers may flag the force-push; that requires an explicit permission grant, not a blanket allow.

## What This Repo Does NOT Contain

- Application source code
- Dockerfiles or docker-compose files (those live in app repos)
- A Makefile, build system, or test suite — workflows are tested by being called from real CI pipelines in consumer repos
- Renovate configuration (`renovate.json` lives in each consumer repo)
