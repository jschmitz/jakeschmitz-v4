# Deployment Plan

This site is a static HTML/CSS site served from a single DigitalOcean droplet.
Deployment is automated via GitHub Actions on every push to `main`.

## Overview

| | |
|---|---|
| Domain | `jakeschmitz.com` |
| Hosting | DigitalOcean droplet, served from `/var/www/jakeschmitz/` |
| CI/CD | GitHub Actions ([deploy.yml](../.github/workflows/deploy.yml)) |
| Trigger | Push to `main` |
| Transport | `rsync` over SSH |

## Pipeline

Defined in [.github/workflows/deploy.yml](../.github/workflows/deploy.yml), the `Deploy to Production` workflow runs two jobs on every push to `main`:

1. **`deploy`**
   - Checks out the repo.
   - Replaces the `__CACHEBUST__` placeholder in `index.html` with the first 8 characters of the commit SHA, so browsers/CDNs fetch fresh assets after each release.
   - Syncs the repo root to `/var/www/jakeschmitz/` on the droplet via `rsync -avz --delete`, using `burnett01/rsync-deployments`. `--delete` means files removed from the repo are also removed from the server — there is no separate "untracked file" cleanup step.
2. **`lighthouse`** (runs after `deploy`, `continue-on-error: true`)
   - Runs Lighthouse CI against `https://jakeschmitz.com/` and uploads the report to temporary public storage.
   - Failures here do not block or roll back the deploy; treat it as a post-deploy signal to check, not a gate.

## Required secrets

Configured under the repo's GitHub Actions secrets:

- `DROPLET_HOST`
- `DROPLET_USER`
- `DROPLET_SSH_KEY`

These must grant the workflow SSH/rsync write access to `/var/www/jakeschmitz/` on the target droplet.

## Executing a deployment

Deployment is push-triggered, not manual:

1. Land changes on `main` (direct push or merged PR).
2. GitHub Actions picks up the push automatically and runs `deploy.yml`.
3. Watch the run in the repo's **Actions** tab. The `deploy` job should complete in well under a minute for a static site of this size; `lighthouse` runs after.
4. Verify the live site at `https://jakeschmitz.com/` once `deploy` succeeds.

No local build step is required — the repo is deployed as-is (aside from the cache-bust substitution), so what's committed is what ships.

## Rollback

There is no automated rollback. To roll back:

1. Revert or push a fix commit to `main` (preferred — keeps history linear and re-triggers the pipeline).
2. If urgent and Actions is unavailable, `rsync` the last known-good commit's contents to `/var/www/jakeschmitz/` directly over SSH using the same credentials as the workflow.

## Verifying a deployment

- Actions tab: confirm the `deploy` job succeeded.
- Load `https://jakeschmitz.com/` and confirm the change is visible; check the page source for the expected cache-bust hash in place of `__CACHEBUST__`.
- Check the `lighthouse` job's report for regressions (informational only — won't block a bad deploy).

## Out of scope / known gaps

- No staging environment — every push to `main` goes straight to production.
- No automated rollback or health check/auto-revert on failed Lighthouse runs.
- No build step, so no opportunity to run tests or linting before deploy; keep `main` deploy-ready.
