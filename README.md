# staged-releases-branch-per-env

A minimal reproduction of the "branch-per-environment" pattern for staging
Renovate updates across environments, per
https://github.com/renovatebot/renovate/discussions/39737.

This is the "immediate `dev`, gate everything else behind the Dependency
Dashboard" approach suggested in that discussion, applied to a repo that
models each environment as a long-lived branch instead of a directory, plus
the preset-based version-gating approach from
https://www.jvt.me/posts/2026/09/08/renovate-staged-branches/, which ensures
`staging` and `prod` can never be updated to a version that hasn't already
been promoted to the environment below them.

The sibling repo,
[`staged-releases`](../staged-releases), covers the same idea but using a
directory per environment, instead of a branch per environment (which is the
structure the original discussion describes).

## Layout

This `main` branch only holds the Renovate config (this repo's default
branch is just used for config discovery, and isn't itself an environment).
The exact same `renovate.json` is also present, byte-for-byte, on every
environment branch, since Renovate reads each base branch's own config file.

The actual environments are separate branches, each with a `docker-compose.yml`
pinning the same `nginx` image, at whatever version has been promoted to that
branch so far, plus a preset that publishes that version for the next branch
up to consume:

- [`dev`](../../tree/dev) — `docker-compose.yml` + `.github/renovate-dev.json`
- [`staging`](../../tree/staging) — `docker-compose.yml` + `.github/renovate-staging.json`
- [`prod`](../../tree/prod) — `docker-compose.yml` + `.github/renovate-prod.json`

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
- `extends` pulls in the `.github/renovate-dev.json` preset from `dev` and the
  `.github/renovate-staging.json` preset from `staging`. Each preset contains
  an `allowedVersions` constraint (`matchBaseBranches`-scoped to the next
  branch up) that caps `nginx` updates to whatever version is currently
  deployed on the branch the preset lives on — so `staging` can never be
  updated past what's live on `dev`, and `prod` can never be updated past
  what's live on `staging`.
- The `customManagers` entry keeps each `.github/renovate-*.json` preset's
  `allowedVersions` in sync with that branch's own `docker-compose.yml`. It
  extracts just the version number after the `<=` (not the whole
  `"<=1.27.3"` string — the Docker datasource needs a real tag to look up,
  and can't resolve one containing a range operator) as the dependency's
  `currentValue`, and lets the regular Docker datasource propose bumping it.
  Because that extracted value shares a `depName`/bucket with the branch's
  own `docker-compose.yml` `nginx` dependency, Renovate groups both changes
  into the same branch/PR — `docker-compose.yml` and the preset's
  `allowedVersions` get bumped together, atomically.
- A `pinDigests: false` `packageRule` stops Renovate from trying to
  digest-pin that same synthetic dependency — `allowedVersions` is a version
  constraint, not an image reference, so there's no digest to pin.
- `.github/renovate-prod.json` has no `matchBaseBranches`, since it's meant to
  be `extends`-ed by other repos that want to track only the `nginx` version
  that's actually live in production.

## Expected behaviour

1. Renovate runs and immediately opens a PR bumping `docker-compose.yml` (and
   the matching `.github/renovate-dev.json` `allowedVersions`) on `dev`.
2. The Dependency Dashboard issue lists checkboxes to approve the equivalent
   update for `staging` and `prod`, but no PRs are raised for them yet — and
   even once ticked, `staging`'s PR is capped to whatever version is on `dev`
   (and `prod`'s to whatever's on `staging`) by the preset's `allowedVersions`.
3. Once `dev` has been merged (and tested), tick the `staging` checkbox on the
   dashboard to raise that PR against the `staging` branch. Repeat for `prod`
   once `staging` is merged.

This still requires a human to do the promotion/gating between environments —
Renovate doesn't have a built-in concept of "only propose this update for
`staging` once `dev` is merged". That's a deliberate scope decision, per the
discussion. What the presets add on top is a guarantee that *if* a PR is
raised for `staging` or `prod`, it can't jump ahead to a version that hasn't
been validated in a lower environment yet, even if a new upstream release
lands while a promotion is in flight.
