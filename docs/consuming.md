---
type: Guidance
title: Using merge-safety-action in a consuming repo
description: The caller workflow to add, why it carries triggers and write scopes, the labels it needs, how to require the check, and how Dependabot keeps the pin current.
tags: [consumer, setup, auto-merge]
---

# Using merge-safety-action in a consuming repo

Add one workflow and require one check. The action is a normal step, so the
consumer owns the job — its triggers, its permissions, its `if:` filter and its
concurrency. The action owns everything else: picking `evaluate` or
`invalidate` from the event, fetching git data, and running the pinned CLI.

## 1. Create the labels first

`evaluate` reconciles two labels on each PR, **`update required`** and
**`merge conflict`**, and honors **`hotfix`** as the escape hatch when the base
branch's own CI is red. Label writes soft-fail silently when a label is missing,
so create all three before adopting — `ai-ensure-labels` seeds the standard
roster, which includes them.

A PR's breaking status is read from the title's `!` alone, so the repo relies on
pr-policy's title check to keep that `!` in step with the `breaking change` label.

## 2. Add the caller workflow

```yaml
# .github/workflows/merge-safety.yml
name: merge-safety

on:
  pull_request_target:
    types: [opened, synchronize, reopened, edited, labeled, unlabeled]
  push:
    branches: [main]
  check_suite:
    types: [completed]
  workflow_dispatch:
    inputs:
      pr:
        description: PR number to evaluate
        required: true

permissions:
  checks: write # post/flip the merge-safety check-run
  statuses: write # mirror the verdict to the merge-safety commit status
  pull-requests: write # reconcile update-required / merge-conflict labels
  contents: read
  actions: write # re-dispatch this workflow per open PR when the base moves

jobs:
  merge-safety:
    if: >-
      (github.event_name != 'push' && github.event_name != 'check_suite') ||
      (github.event_name == 'push' && startsWith(github.ref, 'refs/heads/') && !github.event.deleted) ||
      (github.event_name == 'check_suite' &&
       github.event.check_suite.app.slug == 'github-actions' &&
       github.event.check_suite.head_branch == github.event.repository.default_branch)
    concurrency:
      group: >-
        ${{ (github.event_name == 'push' || github.event_name == 'check_suite')
        && format('merge-safety-invalidate-{0}', github.event_name == 'push' && github.ref_name || github.event.check_suite.head_branch)
        || format('merge-safety-evaluate-{0}', github.run_id) }}
      cancel-in-progress: false
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: rmartz/merge-safety-action@<sha> # vX.Y.Z
        with:
          pr: ${{ inputs.pr }}
```

No `actions/checkout` step is needed: the action fetches the base branch and the
PR head into its own scratch repo under `$RUNNER_TEMP` and leaves the workspace
alone. Why each piece is there:

- **`pull_request_target`, not `pull_request`.** GitHub runs a `pull_request`
  workflow against the synthetic merge commit, which it cannot build for a PR that
  conflicts with its base — so no run is dispatched and the required check never
  posts, exactly when the verdict matters most. `pull_request_target` runs in the
  base context and fires regardless. It is safe here because the action fetches
  the PR head only as git _data_ and runs the published CLI; it never executes
  PR-authored code.
- **`push` drives `invalidate`.** When a branch moves, the action re-dispatches
  this workflow (via `workflow_dispatch`) once per open PR based on that branch.
  Subscribe to the default branch, or to `'**'` if the repo stacks PRs, so a push
  to a stacked parent re-checks its children. The re-dispatch targets the workflow
  the action is running in, whatever its filename.
- **`check_suite: [completed]` re-holds PRs when the base's CI flips.** A push-time
  evaluation sees base CI still pending, so the fan-out also runs when the default
  branch's GitHub Actions suite completes: PRs are held while a required base
  check is red (except `hotfix` PRs) and released when it turns green. It does not
  loop — the fan-out's own runs use `GITHUB_TOKEN`, which GitHub does not let
  trigger a further `check_suite` run.
- **The `if:` filter** skips, without starting a runner, the events the action
  would no-op on anyway: other apps' check suites, other branches' suites, tag
  pushes and branch deletions. Without it the job still behaves correctly, just
  slower and noisier.
- **The `concurrency` group** serializes the base-moved fan-outs per branch and
  never cancels one, so no base move is dropped. Each per-PR `evaluate` run gets a
  group of its own (keyed on `run_id`): a burst of PR events (opening a PR and
  labelling it fires several within a second) must not queue or cancel runs, which
  GitHub reports as `cancelled` — a conclusion automation reads as failing.
  `evaluate` is idempotent (it reads every fact live from the API), so overlapping
  runs post the same verdict.
- **Write scopes.** `checks` and `statuses` carry the verdict, `pull-requests` the
  labels, and `actions` the per-PR re-dispatch. The install needs no token or
  `packages: read`: `@rmartz/merge-safety` is public on npmjs.

## 3. Require the check

Add `merge-safety` to the default branch's required status checks. The name must
be exactly `merge-safety` — see the
[check-run contract](https://github.com/rmartz/merge-safety/blob/main/docs/check-run-contract.md).
GitHub only lists a check in the picker after it has posted once, so open a PR or
dispatch the workflow (`gh workflow run merge-safety.yml -f pr=<n>`) first, or set
it by name through the rulesets API.

## 4. Keep the pin current

Pin the action by commit SHA with a plain `# vX.Y.Z` comment and run Dependabot's
`github-actions` ecosystem:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
```

Each action release pins a specific `@rmartz/merge-safety` version in its
lockfile, so the SHA you pin fully determines the behavior.

## Migrating from the reusable workflow

Repos that call `rmartz/merge-safety/.github/workflows/merge-safety.yml` switch by
replacing their caller with the one above:

- Replace the job body (`uses:` / `with:` / `secrets: inherit`) with the `if:`,
  `concurrency`, `runs-on`, `timeout-minutes` and `steps` shown.
- Drop `packages: read` and any `caller-workflow:` input — the re-dispatch target
  is now detected from the running workflow.
- Keep the triggers, the other permissions, and the required `merge-safety` check
  unchanged; the check-run name is the same.

The caller runs from the base branch under `pull_request_target`, so the migration
PR's own `merge-safety` run still uses the old caller, and the new one takes over
once it merges.
