# Agent guide — merge-safety-action

This repo is the **composite GitHub Action** that posts the
[`@rmartz/merge-safety`](https://github.com/rmartz/merge-safety) check-run in a
consuming repo's CI. It holds the CLI as a pinned `package.json` dependency, wraps
it in [`action.yml`](action.yml), and re-releases itself via semantic-release
whenever Dependabot bumps that pin. It **dogfoods itself**: the
[`merge-safety.yml`](.github/workflows/merge-safety.yml) caller runs `uses: ./`
from the base branch. See [README.md](README.md) and the
[documentation](docs/index.md).

## The check-run name is a fleet contract

The CLI posts a check-run and commit status named **`merge-safety`**; every
consumer requires it and the auto-merge gate keys off it. Nothing here sets that
name, and nothing here may rename it. See the
[check-run contract](https://github.com/rmartz/merge-safety/blob/main/docs/check-run-contract.md).

## Documentation — read it first, maintain it every task

- **Read first.** Before changing `action.yml`, a workflow, or a config, read the
  relevant [`docs/`](docs/index.md) page(s) and this file.
- **Extend, correct, and remove in the same PR.** If your change alters an input,
  the event routing, the release cadence, or a consumer step, fix the doc in the
  same PR. Delete docs that describe something that no longer exists.
- **Docs follow OKF.** Pages under `docs/` carry OKF frontmatter and stay
  reachable from [`docs/index.md`](docs/index.md); the `okf`, `okf-index`, and
  `docs-links` checks enforce this in the Repo Hygiene job. See
  [docs/okf-format.md](docs/okf-format.md).

## Repository conformance

This repo is held to the shared
[repository checklist](https://github.com/rmartz/ai/blob/main/docs/guidance/repository-checklist.md)
and **self-manages** its own config: fix conformance gaps directly here, in a PR.
Bootstrap (`ai-ensure-*`) is a one-time starter, not an ongoing manager.

## Common commands

```bash
npm ci                 # install deps (all from npmjs, no auth needed)
npm run format:check   # prettier --check .
npm run format         # prettier --write .
```

There is no build or test suite — the logic lives in `@rmartz/merge-safety`. CI
checks formatting and that the pinned CLI installs and its bin resolves. The full
evaluate / invalidate path is exercised by the self-caller, which runs the action
from the **base** branch (safe under `pull_request_target`): a change to
`action.yml` reaches this repo's own `merge-safety` check only after it merges.

## Releases

Automated via **semantic-release** ([`.releaserc.json`](.releaserc.json)) through
the shared [semantic-release-ci](https://github.com/rmartz/semantic-release-ci)
workflows: [`release.yml`](.github/workflows/release.yml) tags and creates the
GitHub Release on every push to `main`, and
[`release-check.yml`](.github/workflows/release-check.yml) (required check
`release-check / release-check`) proves the config works on every PR. Nothing is
published and nothing is committed back, so the built-in `GITHUB_TOKEN`
suffices. Never add the semantic-release toolchain to `package.json`.

- **0.x until a deliberate go-live**, mirroring the CLI: `feat:` → minor, `fix:` /
  `perf:` → patch, and a breaking change (`!`) is capped at a minor. Leaving 0.x
  means pushing a `v1.0.0` tag by hand and removing the cap rule.
- **A Dependabot `fix(deps)` bump of `@rmartz/merge-safety` cuts a patch release**,
  which is how CLI changes reach consumers. Dev-dependency and github-actions bumps
  are `chore(…)` and release nothing.
- PR titles are Conventional Commits and the repo squash-merges with the PR title;
  a non-conventional title makes semantic-release skip the release.

## Worktrees & PRs

- **Work in a dedicated worktree** under `.git-worktrees/` (`ai-new-worktree`),
  never on `main` in the root checkout. Run `npm ci` in a fresh worktree.
- **PR titles must be Conventional Commits.** Render PR/issue numbers as full
  Markdown links in chat and agent output, never a bare `#12`.

## Agent directive files

- **`AGENTS.md` is the single source of truth** for a directory's agent
  instructions — author directives here, never in `CLAUDE.md`.
- **Every `AGENTS.md` has a companion `CLAUDE.md`** (a bare `@AGENTS.md` wrapper),
  enforced by the `md-pairing` check.
