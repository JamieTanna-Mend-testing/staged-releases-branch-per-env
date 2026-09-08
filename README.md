# staged-releases-branch-per-env

A minimal reproduction of the "branch-per-environment" pattern for staging
Renovate updates across environments, per
https://github.com/renovatebot/renovate/discussions/39737.

This is the "immediate `dev`, gate everything else behind the Dependency
Dashboard" approach suggested in that discussion, applied to a repo that
models each environment as a long-lived branch instead of a directory.

The sibling repo,
[`staged-releases`](../staged-releases), covers the same idea but using a
directory per environment, instead of a branch per environment (which is the
structure the original discussion describes).

## Layout

This `main` branch only holds the Renovate config (this repo's default
branch is just used for config discovery, and isn't itself an environment).

The actual environments are separate branches, each with a `docker-compose.yml`
pinning the same `nginx` image, at whatever version has been promoted to that
branch so far:

- [`dev`](../../tree/dev)
- [`staging`](../../tree/staging)
- [`prod`](../../tree/prod)

`dev` is intentionally ahead of `staging`, which is ahead of `prod`, to
simulate a rollout that's already in progress.

## Renovate config

See [`renovate.json`](renovate.json).

- `baseBranchPatterns` tells Renovate to process the `dev`, `staging` and
  `prod` branches (instead of just the default branch).
- The `packageRules` are split by `matchBaseBranches`, one block per branch:
  - `dev` — PRs are raised as soon as an update is found, no gating.
  - `staging` and `prod` — `dependencyDashboardApproval: true`, so the PR is
    only raised once it's manually ticked on the Dependency Dashboard issue.

## Expected behaviour

1. Renovate runs and immediately opens a PR bumping `docker-compose.yml` on
   the `dev` branch.
2. The Dependency Dashboard issue lists checkboxes to approve the equivalent
   update for `staging` and `prod`, but no PRs are raised for them yet.
3. Once `dev` has been merged (and tested), tick the `staging` checkbox on the
   dashboard to raise that PR against the `staging` branch. Repeat for `prod`
   once `staging` is merged.

This still requires a human to do the promotion/gating between environments —
Renovate doesn't have a built-in concept of "only propose this update for
`staging` once `dev` is merged". That's a deliberate scope decision, per the
discussion.
