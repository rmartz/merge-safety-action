---
type: Design
title: How merge-safety-action works
description: How the action routes each event to evaluate or invalidate, fetches git data without touching the workspace, pins the CLI version in its lockfile, and versions and releases itself.
tags: [design, releases, versioning]
---

# How merge-safety-action works

The verdict logic lives in the
[`@rmartz/merge-safety`](https://github.com/rmartz/merge-safety) CLI. This action
only decides which CLI operation an event calls for, gives the CLI the git data it
needs, and pins which CLI version runs.

## Event routing

The first step reads the event and picks one of three modes:

| Event                                                      | Mode         | CLI call                                                |
| ---------------------------------------------------------- | ------------ | ------------------------------------------------------- |
| `pull_request_target`, `pull_request`, `workflow_dispatch` | `evaluate`   | `merge-safety evaluate --pr <n> --base origin/<base>`   |
| `push` to a branch (not a tag, not a deletion)             | `invalidate` | `merge-safety invalidate --base-branch <pushed branch>` |
| `check_suite` from `github-actions` on the default branch  | `invalidate` | `merge-safety invalidate --base-branch <default>`       |
| anything else                                              | `skip`       | none — no setup-node, no install                        |

`evaluate` takes the PR number from the event, or from the `pr` input on a
`workflow_dispatch` re-dispatch, and fails if neither is present. `invalidate`
re-dispatches the caller workflow per open PR; the target defaults to the running
workflow's own file (parsed from `GITHUB_WORKFLOW_REF`), so callers need no
`caller-workflow` input unless they want a different target.

The consumer's job-level `if:` repeats the skip rules only so irrelevant events
don't start a runner; the action is correct without it.

## Git data without a checkout

`evaluate` needs the base branch's full history (merge-base, base log, diffs) and
the PR head commit. The action `git init`s a scratch repo under `$RUNNER_TEMP`,
fetches `refs/heads/<base>` and `refs/pull/<n>/head` into it, and runs the CLI
with `--cwd` pointing there. The token rides in a repo-local `extraheader`, as
`actions/checkout` does, so private repos fetch. The consumer's workspace is never
touched, so the action composes with any other steps in the job.

## The CLI version is the lockfile

`package.json` pins `@rmartz/merge-safety` to an exact version and
`package-lock.json` locks it; the action runs `npm ci` in its own directory
(`$GITHUB_ACTION_PATH`) and invokes that copy's `merge-safety` bin. A given action
SHA therefore always runs the same CLI. There is no `version` input. This replaces
the reusable workflow's runtime lookup, which resolved the version by matching its
own pinned SHA against the merge-safety release tags.

## Releases and distribution

Releases are automatic via semantic-release ([`.releaserc.json`](../.releaserc.json)),
through the shared [semantic-release-ci](https://github.com/rmartz/semantic-release-ci)
workflows. A merge to `main` cuts the `vX.Y.Z` tag and GitHub Release; nothing is
published to a registry and nothing is committed back.

The chain that ships a CLI change to the fleet:

1. merge-safety releases a new CLI version to npmjs.
2. This repo's Dependabot (npm, daily, first-party cooldown exempt) opens
   `fix(deps): bump @rmartz/merge-safety …`; bot-automerge merges a patch/minor.
3. `fix:` maps to a patch, so semantic-release tags a new action release.
4. Consumers' Dependabot (`github-actions`) bumps their `@<sha>` pin.

The action stays on the **0.x line**, mirroring the CLI, until a deliberate
go-live: a breaking change (`!`) is capped at a minor bump. Leaving 0.x means
cutting `v1.0.0` by hand and removing that cap rule.
