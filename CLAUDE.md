# CLAUDE.md — fleet-toolkit

Reusable GitHub Actions workflows shared across a small self-hosted app fleet. Read
`README.md` first for the workflow list and usage — this file is the "don't repeat past
mistakes" layer for anyone modifying these workflows.

## Two audiences, same repo

- **App-facing** (`check-upstream-release.yml`, `compose-lint.yml`, `compose-smoke-test.yml`)
  — called directly by app/shared-infra repos. Must stay generic: no repo-specific
  hardcoding, parameterized entirely via `inputs:`.
- **Fleet-facing** (`trigger-coolify-deploy.yml`, `verify-post-deploy-backup.yml`) — called by
  the (private) fleet Hub's own orchestration workflow, not by individual app repos.

## Versioning: `@v1` is a floating tag, moved deliberately

Consumers pin `uses: <owner>/fleet-toolkit/.github/workflows/<name>.yml@v1`. `v1` gets moved
forward (delete + recreate + push) when a real bug is found and fixed in one of these
workflows — this has already happened three times in this repo's history (a bad third-party
action version pin, a missing env-file input, a false-positive health check). That's the
intended pattern for this repo at this stage (no external consumers beyond this fleet yet) —
not a mistake to avoid, but expect the harness's security classifier to flag a tag
force-push by default; it needs an explicit, narrowly-scoped permission grant, not a blanket
force-push allow.

## Gotchas specific to workflows in this repo

**`docker compose ps` without `-a` only lists running containers.**
`compose-smoke-test.yml`'s crash-detection step must use `ps -a` — the plain form silently
omits anything that already exited, so a stack where every container crashed still reports
zero problems. This shipped broken for a while before being caught by an app repo's real CI
run.

**A single-sample status check false-positives on a crash-looping container.**
`verify-post-deploy-backup.yml` requires 3 *consecutive* `running:*` reads (resetting the
streak on any non-running read) rather than exiting on the first success — a container
sampled mid-restart-cycle can read as `running` for exactly one poll and then die again.
Reproduced live: 14 consecutive `exited:unhealthy` reads, then one lucky `running:healthy`
sample that an unfixed single-sample check accepted immediately.

**Triggering a deploy only queues it.** `trigger-coolify-deploy.yml` doesn't just fire the
`GET /api/v1/deploy` call and declare success from the "queued" response — it captures the
returned `deployment_uuid` and polls `GET /api/v1/deployments/<uuid>` until `finished` or
`failed`, *before* any caller moves on to checking application-level health. Checking app
health alone is not enough: if a new deploy fails, Coolify leaves the previous container
running, so the app still reads `running:healthy` and a naive check goes green on a deploy
that never landed.

**A caller's `permissions:` block can only narrow what `check-upstream-release.yml`
declares, never widen it.** This workflow declares `contents: write, pull-requests: write`,
but if a calling repo's own workflow (or repo-level default) restricts the token further,
Renovate silently does nothing — exits 0, scans zero repos, looks like success. Callers also
need a `RENOVATE_REPOSITORIES` env var (Renovate doesn't infer the repo from Actions
context) and the *calling repo* needs `issues: write` at the repo/Actions-permissions level
for the Dependency Dashboard (a GitHub issue) to work — `contents`+`pull-requests` write
doesn't cover it. Don't trust "the Renovate run was green" as proof it did anything; check
for a non-zero file/dependency count in its logs or an actual PR/dashboard issue.
