---
okf_version: 0.2
---

# Documentation

Documentation for `merge-safety-action`, the composite GitHub Action that posts the
[`@rmartz/merge-safety`](https://github.com/rmartz/merge-safety) check-run in a
consuming repo's CI. Written in [Open Knowledge Format](okf-format.md).

- [Using it in a consuming repo](consuming.md) — the caller workflow, the
  permissions and labels it needs, requiring the check, and migrating from the
  reusable workflow.
- [How the action works](design.md) — event routing, the scratch git fetch, how
  the CLI version is pinned, and how new versions reach consumers.
- [The OKF documentation format](okf-format.md) — how these pages are structured
  and validated in this repo.
