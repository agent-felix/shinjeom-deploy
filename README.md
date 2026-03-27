# shinjeom-deployments

This repo tracks what is currently deployed to `staging` and `prod` across the Shinjeom services.

It is intentionally simple:

- Code keeps living in each service repo.
- Each deployable commit gets an immutable Git tag in that service repo.
- This repo records which tag is live in each environment.
- `deployed_commit`, `deployed_at`, and `updated_at` are required so deployment state stays auditable.

For the deployment operating standard, see `DEPLOYMENT_STANDARD.md`.

This avoids using long-lived `staging` and `prod` code branches just to remember deployment state.

## Tracked Services

- `shinjeom-agents-sarang`
- `shinjeom-agents-yeona`
- `shinjeom-api`
- `long-term-memory`
- `shinjeom-mobile`

## Files

- `staging.yaml`: current state of the staging environment
- `prod.yaml`: current state of the production environment

## Recommended Workflow

1. Make changes in the service repo on whatever branch contains the deployable commit.
2. When a commit is ready to deploy, create an annotated tag on that exact commit using `<service-name>-vX.Y.Z` or `<service-name>-vX.Y.Z-rcN`.
3. Deploy that tagged commit.
4. Update this repo's environment file with the deployed tag, commit SHA, and timestamps.
5. Commit the environment file change in this repo.

## Tagging Example

Run this inside the service repo that you are deploying:

```bash
git checkout <branch-with-the-deployable-commit>
git pull --ff-only
git tag -a shinjeom-api-v2.0.0-rc1 -m "shinjeom-api v2.0.0-rc1"
git push origin shinjeom-api-v2.0.0-rc1
```

Tags point to commits, not branches, so Git allows tagging any commit. In practice, the release tag should point to the exact commit you actually deployed.

## Branch and Tag Rules

- Tags point to commits, not branches.
- It is valid to tag a commit on a feature branch, `dev`, or any other branch.
- This repo does not require separate `staging` or `prod` branches.
- Use service-prefixed tags so each deployable unit is unambiguous.
- Example final tag: `shinjeom-api-v2.0.0`
- Example pre-release tag: `shinjeom-api-v2.0.0-rc1`
- If a feature branch commit is only for testing, prefer a pre-release tag instead of the final release tag.
- Avoid putting the final release tag on a feature branch if that commit will later be rewritten by squash merge or rebase.

## Release Candidates

`rc` means `release candidate`.

Examples:

- `shinjeom-api-v2.0.0-rc1`: first candidate for release `shinjeom-api-v2.0.0`
- `shinjeom-api-v2.0.0-rc2`: second candidate
- `shinjeom-api-v2.0.0-rc3`: third candidate

Use `rc` tags when you want to deploy and test before calling a version final.

Typical flow:

1. Tag a testable build as `shinjeom-api-v2.0.0-rc1`
2. Deploy it to `staging`
3. Fix issues if needed
4. Tag the next candidate as `shinjeom-api-v2.0.0-rc2`
5. When it is accepted, create the final tag `shinjeom-api-v2.0.0`

## Release Tags vs Environment State

- A release tag identifies the code version, such as `shinjeom-api-v2.0.0`.
- The deployment repo identifies which environment is currently running that version.
- The same release tag can appear in both `staging.yaml` and `prod.yaml` if both environments run the same commit.

You do not need separate environment tags such as `staging-v2.0.0` and `prod-v2.0.0` unless you specifically want per-environment deployment-event tags in each service repo.

A tag alone does not tell you whether that revision is only in `staging`, already in `prod`, or later got rolled back. That environment mapping is the reason this repo exists. The commit SHA and timestamps are also recorded so deployment state stays explicit and auditable.

## Updating Deployment State

After deploying, update `staging.yaml` or `prod.yaml`.

Example:

```yaml
environment: staging
updated_at: "2026-03-27T10:30:00+09:00"
services:
  shinjeom-api:
    repo_path: ../shinjeom-api
    tracked_branch: feature/release-api
    deployed_tag: shinjeom-api-v2.0.0-rc1
    deployed_commit: a1b2c3d4
    deployed_at: "2026-03-27T10:28:00+09:00"
    notes: ""
```

Then commit the change in this repo:

```bash
git add staging.yaml
git commit -m "staging: shinjeom-api -> shinjeom-api-v2.0.0-rc1"
git push
```

Update rule:

- If an environment changed, update this repo.
- If you only created a tag but did not deploy it, do not update this repo.
- Record `deployed_tag` as the canonical value.
- Record `deployed_commit`, `deployed_at`, and `updated_at` for every deployment change.
- If `shinjeom-api-v2.0.0-rc1` is deployed to `staging`, record `shinjeom-api-v2.0.0-rc1` in `staging.yaml`.
- If the final `shinjeom-api-v2.0.0` is deployed to `prod`, record `shinjeom-api-v2.0.0` in `prod.yaml`.

## Why This Setup

- You can answer "what is in prod right now?" from one repo.
- You keep immutable release history in the service repos via tags.
- You can see where each tag is currently deployed without relying on branch names.
- You avoid branch drift and merge noise from environment branches.
- Rollbacks are simple because you can redeploy an older tag and update this repo.

The two agents are tracked separately as `shinjeom-agents-sarang` and `shinjeom-agents-yeona` even though they deploy from the same repo.

## Notes

- `tracked_branch` is informational only. It can be a feature branch, `dev`, or anything else.
- `deployed_tag` is the source of truth for the deployed revision in this repo.
- `deployed_commit`, `deployed_at`, and `updated_at` are required deployment-state fields.
- Keep release tags immutable. If you need a new deploy, make a new tag.
