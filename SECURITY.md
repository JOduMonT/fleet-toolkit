# Security Policy

Reusable GitHub Actions workflows shared across the fleet: upstream release checks (Renovate), compose lint and smoke tests, Coolify deploy triggers and post-deploy backup verification.
Other repositories call them as `JOduMonT/fleet-toolkit/...@v1`.

## Supported versions

Only the current `main` branch is supported.
Fixes land on `main`; there are no release branches.

## Reporting a vulnerability

Please report privately.
Do not open a public issue or pull request.

- **Preferred:** [report a vulnerability](https://github.com/JOduMonT/fleet-toolkit/security/advisories/new) through GitHub private vulnerability reporting.
- **Email:** jodumont+security@gmail.com
- Include what you found, the affected file or service, steps to reproduce and the impact you see.
- Do not access, change or delete data that is not yours, and do not run denial-of-service or automated scanning against live systems.

You can expect an acknowledgement within 3 business days and a status update within 10.
Confirmed issues are fixed as quickly as severity allows, and you are credited in the fix unless you prefer not to be.

## Scope

In scope:

- Workflow injection through inputs, `pull_request_target`, or unquoted `${{ }}` expressions.
- Over-broad `permissions:` or secrets passed to callers.
- Integrity of the `v1` tag that every caller trusts: anything that lets someone move it or change what it points to.
- The Coolify deploy trigger and its token handling.

Out of scope:

- The calling repositories' own workflows, GitHub Actions, Renovate and Coolify.
- Social engineering and physical attacks.

## How this repository is kept safe

- Dependabot alerts and security updates are on, and `.github/dependabot.yml` opens weekly grouped version updates.
- Dependabot pull requests are merged automatically by `.github/workflows/dependabot-auto-merge.yml` once every other check passes.
  Major version bumps are left open for review.
- GitHub secret scanning with push protection and CodeQL code scanning are enabled.
- CodeQL analyses the workflow files for injection patterns.
- Third-party actions are version-pinned and updated by Dependabot.
