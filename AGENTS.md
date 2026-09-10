# AGENTS.md — fleet-toolkit

Reusable GitHub Actions workflows shared across a small self-hosted app fleet.
Read `README.md` first for the workflow list and usage; this file is the "don't repeat past mistakes" layer for anyone modifying these workflows.

`CLAUDE.md` and `QWEN.md` are symlinks to this file.
`AGENTS.md` is the instruction file for every coding agent whatever vendor it comes from, and a vendor-specific filename is only ever a symlink to it, never a second copy.
That rule exists because the copies had already diverged here: `CLAUDE.md` and `QWEN.md` were two different documents, each holding something the other lacked, and this file is their merge.

## Project overview

There is no application source code in this repo, only YAML workflow definitions invoked via `workflow_call` from app and shared-infra repos.

- Owner: `JOduMonT`
- License: GPL-3.0
- Deployment target: Coolify, a self-hosted PaaS
- Stack: Docker Compose, GitHub Actions, Renovate

## Two audiences, same repo

**App-facing** workflows are called directly by app and shared-infra repos.
They must stay generic: no repo-specific hardcoding, parameterized entirely via `inputs:`.

| Workflow | Purpose | Key inputs |
|---|---|---|
| `check-upstream-release.yml` | Runs Renovate against the calling repo to open version-bump PRs | None; the caller needs its own `renovate.json` |
| `compose-lint.yml` | Validates docker-compose file syntax | `compose-file`, `env-file` |
| `compose-smoke-test.yml` | Starts the stack standalone, detects crashed containers, optionally checks a health URL, tears down | `compose-file`, `env-file`, `health-url` |

**Fleet-facing** workflows are called by the private fleet Hub's own orchestration, not by individual app repos.

| Workflow | Purpose | Key inputs and secrets |
|---|---|---|
| `trigger-coolify-deploy.yml` | Triggers a Coolify deploy and polls until it reaches a terminal state | `app-uuid`, `coolify-url`, `coolify-api-token` |
| `verify-post-deploy-backup.yml` | Polls Coolify until the app is stably running after a deploy | `app-uuid`, `coolify-url`, `coolify-api-token` |

## Consumer usage pattern

```yaml
jobs:
  lint:
    uses: JOduMonT/fleet-toolkit/.github/workflows/compose-lint.yml@v1
  smoke-test:
    uses: JOduMonT/fleet-toolkit/.github/workflows/compose-smoke-test.yml@v1
    with:
      health-url: http://localhost:PORT/
```

Consumers pin `@v1`, never `@main`.

## Versioning: `@v1` is a floating tag, moved deliberately

`v1` gets moved forward — delete, recreate, push — when a real bug is found and fixed in one of these workflows.
That has already happened three times in this repo's history: a bad third-party action version pin, a missing env-file input, and a false-positive health check.
It is the intended pattern for this repo at this stage, with no external consumers beyond this fleet, rather than a mistake to avoid.
Expect the harness's security classifier to flag a tag force-push by default; it needs an explicit, narrowly-scoped permission grant, not a blanket force-push allow.

## Gotchas specific to workflows in this repo

**`docker compose ps` without `-a` only lists running containers.**
`compose-smoke-test.yml`'s crash-detection step must use `ps -a` — the plain form silently omits anything that already exited, so a stack where every container crashed still reports zero problems.
This shipped broken for a while before an app repo's real CI run caught it.

**A single-sample status check false-positives on a crash-looping container.**
`verify-post-deploy-backup.yml` requires 3 *consecutive* `running:*` reads, resetting the streak on any non-running read, rather than exiting on the first success.
A container sampled mid-restart-cycle can read as `running` for exactly one poll and then die again.
Reproduced live: 14 consecutive `exited:unhealthy` reads, then one lucky `running:healthy` sample that an unfixed single-sample check accepted immediately.

**Triggering a deploy only queues it.**
`trigger-coolify-deploy.yml` does not fire the `GET /api/v1/deploy` call and declare success from the "queued" response.
It captures the returned `deployment_uuid` and polls `GET /api/v1/deployments/<uuid>` until `finished` or `failed`, before any caller moves on to checking application-level health.
Checking app health alone is not enough: if a new deploy fails, Coolify leaves the previous container running, so the app still reads `running:healthy` and a naive check goes green on a deploy that never landed.

**A caller's `permissions:` block can only narrow what `check-upstream-release.yml` declares, never widen it.**
This workflow declares `contents: write`, `pull-requests: write` and `issues: write`, but if a calling repo's own workflow or repo-level default restricts the token further, Renovate silently does nothing — it exits 0, scans zero repos, and looks like success.
Callers also need a `RENOVATE_REPOSITORIES` env var, because Renovate does not infer the repo from Actions context, and the *calling repo* needs `issues: write` at the repo Actions-permissions level for the Dependency Dashboard, which is a GitHub issue that `contents` and `pull-requests` write do not cover.
Do not trust "the Renovate run was green" as proof it did anything; check for a non-zero file or dependency count in its logs, or an actual PR or dashboard issue.

**`compose-lint.yml` needs an env file.**
It copies the given `env-file` to `.env` before running `docker compose config -q`, because variable interpolation in a compose file needs real or dummy values present.
A caller that omits it lints a file whose `${VAR:?…}` references cannot resolve.

## Development conventions

- App-facing workflows stay generic, parameterized entirely via `inputs:`, with no repo-specific hardcoding.
- Inline comments explain the *why*.
  Each workflow carries detailed comments documenting the specific bug or failure mode that drove its current implementation.
  Do not strip these when editing.
- `set -euo pipefail` in every shell step.
- Tear down on `always()`.
  `compose-smoke-test.yml` runs `docker compose down -v` in an `if: always()` step so it cleans up even on failure.

## What this repo does not contain

- Application source code
- Dockerfiles or compose files, which live in app repos
- A Makefile, build system or test suite — these workflows are tested by being called from real CI pipelines in consumer repos
- Renovate configuration, since `renovate.json` lives in each consumer repo
