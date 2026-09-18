# Plot translation placeholder production fix

Production API deployment requested by the user on 2026-09-18. Staging, Faceless and Backstage were not deployed.

## Release

- Source branch: `fix/plot-translation-placeholder-presence`, pushed; not merged into `dev`.
- Commit: `9778210330f2aacb434081da3fd5619f2783a968`.
- Tag: `shinjeom-api-v2.4.1`, pushed.
- ECR image: `743807085349.dkr.ecr.us-west-2.amazonaws.com/prod/shinjeom-api:shinjeom-api-v2.4.1`.
- Image index digest: `sha256:0447e5562e36b11efd43a372afdb8c499a1cc7abde7876add3993c7733dbcdbb`.
- Linux amd64 manifest: `sha256:72e8ee63b543b23cca5f3acb9262ce9e69144a7f96f65224f8f5bc7240b075b1`.
- Helm revision 39, deployed; rollout started 23:52:30 KST. Public health healthy at 23:54:31 KST. Both API pods ready with zero restarts.

## Change and validation

Translations may repeat or reduce occurrences of `{{user}}` if the placeholder is present in both the original and translated field. Removing it entirely or introducing it into a field where it was absent is still rejected. Existing blank-text and structural validation remains intact.

66 relevant unit/integration tests passed. `git diff --check` and Helm lint passed. The rendered non-hook resource comparison against the live release showed only the API container image changed. Deployment used captured production values and Helm atomic/wait. Production runtime code SHA256 matches the release source: `c8d9de7c14d1cd03319a1f5e24f7a54a6072aeeb8ac2746820ada76e30c058da`.

Runtime free limit remains 10, with limits and sales enabled. Database revision remains `f82d3a0b5c41`. No new API pod error/traceback logs were found in the post-rollout check. Staging remains Helm revision 109, image `shinjeom-api-v2.4.0-rc10`.

## Remaining publication blocker

Read-only DB retrieval followed by translation and publication validation was run against the actual `배우, 서도훈` draft. No save or publish was performed. Draft revision remains 4 and published revision is null.

The actual translation validation did **not** pass: a further attempt identified hashtag counts changing from 10 in the source to 9 in French/Russian/es-MX/pt-BR and 8 in German/pt-PT. The existing structure check rejects this. Other compared structure fields were unchanged in that attempt. These are reproduction results, not proof of the original failed request's exact response.

The requested placeholder frequency relaxation is deployed, but it is insufficient to guarantee publication. Hashtag count handling remains an additional issue; it was not silently relaxed in this release. No plots were published.

## Rollback

Previous production Helm revision: 38, image `shinjeom-api-v2.4.0`. Roll back only if required using the production kubeconfig and namespace `shinjeom-api`. Revision 38 retains the strict placeholder count check, so it restores the prior publication limitation.
