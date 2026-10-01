# merge-safety-action

A composite GitHub Action that posts the
[`@rmartz/merge-safety`](https://github.com/rmartz/merge-safety) check-run — the
pre-auto-merge verdict on whether a PR is current with its base, carries a
breaking change, or conflicts — and re-holds open PRs when their base moves.
Packaged so that:

1. **Updates propagate automatically.** Consuming repos pin this Action by SHA;
   Dependabot's `github-actions` ecosystem bumps the pin.
2. **The CLI version is pinned here.** Each release locks a specific
   `@rmartz/merge-safety` version, and a Dependabot bump of that dependency cuts
   the next release — so a CLI fix reaches the fleet with no per-repo edits.

The verdict logic lives in `@rmartz/merge-safety`. This repo routes each event to
the right CLI operation and pins the CLI version.

## Using it in a consuming repo

```yaml
jobs:
  merge-safety:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: rmartz/merge-safety-action@<sha> # vX.Y.Z
        with:
          pr: ${{ inputs.pr }}
```

That step needs the full caller around it — `pull_request_target` / `push` /
`check_suite` / `workflow_dispatch` triggers, write scopes, a job `if:` filter and
a concurrency group. Copy it from the [consumer setup guide](docs/consuming.md),
then require the `merge-safety` check on the default branch.

### Inputs

| Input             | Default               | Meaning                                                                |
| ----------------- | --------------------- | ---------------------------------------------------------------------- |
| `pr`              | `''`                  | PR to evaluate on `workflow_dispatch`. PR events use their own number. |
| `caller-workflow` | `''`                  | Workflow file `invalidate` re-dispatches. Empty uses the running one.  |
| `token`           | `${{ github.token }}` | Token for the API and git fetches (needs the caller's write scopes).   |
| `node-version`    | `'22'`                | Node.js version the CLI runs under.                                    |

There is no `version` input: the CLI version is the one pinned in this Action's
lockfile.

## How it relates to `@rmartz/merge-safety`

This Action replaces that repo's reusable workflow
(`rmartz/merge-safety/.github/workflows/merge-safety.yml`). See
[how the action works](docs/design.md) and the
[migration steps](docs/consuming.md#migrating-from-the-reusable-workflow).

## Documentation

Full docs, written in [Open Knowledge Format](docs/okf-format.md), start at
[docs/index.md](docs/index.md).

## Releases

Versioned by [semantic-release](https://semantic-release.gitbook.io/): a merge to
`main` cuts the tag + GitHub Release. It publishes no package and commits nothing
back. A Dependabot `fix(deps)` bump of `@rmartz/merge-safety` cuts a patch
release.
