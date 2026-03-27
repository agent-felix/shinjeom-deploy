# shinjeom-deployments

This repo tracks what is currently deployed to `staging` and `prod` across the Shinjeom services.

It is intentionally simple:

- Code keeps living in each service repo.
- Each deployable commit gets an immutable Git tag in that service repo.
- This repo records which tag and commit are live in each environment.

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

1. Make changes in the service repo, normally on `dev`.
2. When a commit is ready to deploy, create an annotated tag on that exact commit.
3. Deploy that tagged commit.
4. Update this repo's environment file with the deployed tag, commit SHA, and timestamp.
5. Commit the environment file change in this repo.

## Tagging Example

Run this inside the service repo that you are deploying:

```bash
git checkout dev
git pull
git tag -a v2.0.0 -m "Release v2.0.0"
git push origin v2.0.0
```

Tags point to commits, not branches, so Git allows tagging any commit. In practice, official release tags should point to the exact commit you actually deployed.

## Branch and Tag Rules

- Keep day-to-day work on `dev` if that is your normal flow.
- Deploying from `dev` to `staging` is the default and simplest path.
- It is valid to tag a commit on a feature branch because tags point to commits, not branches.
- If a feature branch commit is only for testing, prefer a pre-release tag such as `v2.0.0-rc1` instead of the final release tag.
- Avoid putting the final release tag on a feature branch if that commit will later be rewritten by squash merge or rebase.

## Release Candidates

`rc` means `release candidate`.

Examples:

- `v2.0.0-rc1`: first candidate for release `v2.0.0`
- `v2.0.0-rc2`: second candidate
- `v2.0.0-rc3`: third candidate

Use `rc` tags when you want to deploy and test before calling a version final.

Typical flow:

1. Tag a testable build as `v2.0.0-rc1`
2. Deploy it to `staging`
3. Fix issues if needed
4. Tag the next candidate as `v2.0.0-rc2`
5. When it is accepted, create the final tag `v2.0.0`

## Release Tags vs Environment State

- A release tag identifies the code version, such as `v2.0.0`.
- The deployment repo identifies which environment is running that version.
- The same release tag can appear in both `staging.yaml` and `prod.yaml` if both environments run the same commit.

You do not need separate environment tags such as `staging-v2.0.0` and `prod-v2.0.0` unless you specifically want per-environment deployment-event tags in each service repo.

## Updating Deployment State

After deploying, update `staging.yaml` or `prod.yaml`.

Example:

```yaml
environment: staging
updated_at: "2026-03-26T16:40:00+09:00"
services:
  shinjeom-api:
    repo_path: ../shinjeom-api
    tracked_branch: dev
    deployed_tag: v2.0.0
    deployed_commit: a1b2c3d4
    deployed_at: "2026-03-26T16:38:00+09:00"
    notes: ""
```

Then commit the change in this repo:

```bash
git add staging.yaml
git commit -m "staging: shinjeom-api -> v2.0.0"
git push
```

Update rule:

- If an environment changed, update this repo.
- If you only created a tag but did not deploy it, do not update this repo.
- If `v2.0.0-rc1` is deployed to `staging`, record `v2.0.0-rc1` in `staging.yaml`.
- If the final `v2.0.0` is deployed to `prod`, record `v2.0.0` in `prod.yaml`.

## Why This Setup

- You can answer "what is in prod right now?" from one repo.
- You keep immutable release history in the service repos via tags.
- You avoid branch drift and merge noise from environment branches.
- Rollbacks are simple because you can redeploy an older tag and update this repo.

The two agents are tracked separately as `shinjeom-agents-sarang` and `shinjeom-agents-yeona` even though they deploy from the same repo.

## Notes

- `tracked_branch` is set to `dev` because that is your current working style. If a service changes later, update that field.
- `deployed_tag` and `deployed_commit` should match the exact deployed revision.
- Keep release tags immutable. If you need a new deploy, make a new tag.
