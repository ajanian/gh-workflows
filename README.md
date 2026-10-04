# gh-workflows

Reusable GitHub Actions workflows for Cloud Run projects: one PR gate with Dependabot
auto-merge behind it, keyless deploys through a no-traffic candidate, and failure-only
notifications. Built for my own projects: `main` is the only supported ref and it changes
without notice.

Each repo carries three small stubs that own what a reusable workflow cannot — triggers,
concurrency, permissions — and pass the project's values. The behaviour, and every pinned
third-party action, lives here.

| Workflow | What it does | Caller stub |
|---|---|---|
| `pr.yml` | Runs the caller's `gate` on every PR. For a Dependabot PR that passed it: squash-merge bumps that are positively patch or minor; one comment on anything else. | `ci.yml`: `pull_request` trigger, per-PR concurrency, the gate as `with:` |
| `deploy-cloud-run.yml` | Drift check → optional verify → build (Cloud Build or buildx) → no-traffic candidate → smoke check → optional candidate hook → promote → roll back to the traffic-holding revision → `production` tag | `deploy.yml`: push / 6-hourly cron / dispatch, concurrency, the project values as `with:` |
| `notify-failure.yml` | Slack webhook on any non-success run; healthchecks ping on every Deploy completion | `notify.yml`: the `workflow_run` trigger, a `workflow_dispatch` wiring test, `secrets: inherit` |

## Stubs

```yaml
# .github/workflows/ci.yml
name: CI
on: pull_request
concurrency:
  group: ci-${{ github.event.pull_request.number }}
  cancel-in-progress: true
permissions:
  contents: write        # the gate job drops to read; only auto-merge writes
  pull-requests: write
jobs:
  ci:
    uses: ajanian/gh-workflows/.github/workflows/pr.yml@main
    with:
      gate: |
        npm ci
        npm run build
        npm test
```

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]
  schedule:
    - cron: "17 */6 * * *"   # drift check: deploy if prod != main
  workflow_dispatch:
concurrency:
  group: deploy
  cancel-in-progress: false
permissions:
  contents: write   # moves the `production` tag
  id-token: write   # keyless auth (Workload Identity Federation)
jobs:
  deploy:
    uses: ajanian/gh-workflows/.github/workflows/deploy-cloud-run.yml@main
    with:
      project: my-project
      region: us-east1
      service: my-service
      image: us-east1-docker.pkg.dev/my-project/my-repo/app
      app_url: https://app.example.com
      workload_identity_provider: projects/123456789/locations/global/workloadIdentityPools/github/providers/github
      service_account: github-deployer@my-project.iam.gserviceaccount.com
```

`notify.yml` passes `secrets: inherit` and needs two repo secrets, `SLACK_WEBHOOK_URL` and
`HC_PING_KEY`; its header comment in `notify-failure.yml` explains both.

## What the app must provide

`GET /api/version` returning `{"data":{"commit":"<short-sha>"}}`, read from the `COMMIT_SHA`
env var the deploy stamps. The smoke check, the promote check and the drift check all compare
it to the commit being deployed.

## Working on this repo

Callers pin `@main`, so a change here reaches every caller on its next run — at the latest the
next 6-hourly drift check. The notifier is what makes that acceptable. `Lint` runs `actionlint`
on every push; keep it green. Dependabot proposes action bumps here weekly; they are merged by
hand, because one merge changes every caller.

Each workflow documents its inputs, secrets and reasoning at the top of its file. The one-time
GCP setup (Workload Identity pool, deployer service account) is not covered here.
