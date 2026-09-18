# Manshin Pass consumer web release

## Release source

- Web version: `1.1.0`.
- Source commit: `2de7e0acf9bf71bf75ceca6b65f9db661b7e56bf`.
- PR #96 supplied the text-consult and Manshin Pass implementation and had already been merged into dev and promoted to staging.
- PR #97 was squash-merged as `278ec4025d145d0237853c0390191bddfebe909c`; its final delta adds dev/prod Faceless SSM origin configuration and bumps the web version in five files.
- The first dev deploy failed on an existing asynchronous analytics assertion in the Mio profile-selection test. PR #98 wraps the existing exact-once assertion in `waitFor`, preserving its event name, payload and call-count requirements. It changes only the test file; the application code is unchanged.
- PR #98 passed both full CI jobs (lint/typecheck/unit tests and build/Playwright), then was squash-merged as the release source above. The focused local test file passed all 9 tests.

## Deployment sequence

The user explicitly requested immediate staging promotion without waiting for the dev deployment to finish. `make promote-staging` fast-forwarded staging from `420d21036d44b99502150e8966c6130db6dbd539` to the release source. The staging workflow selected its normal fallback to run its own checks because the dev deployment was still running. No CI gate or workflow was modified.

- PR #97: https://github.com/agent-station/shinjeom-web/pull/97
- Test fix #98: https://github.com/agent-station/shinjeom-web/pull/98
- Passing #98 CI: https://github.com/agent-station/shinjeom-web/actions/runs/35355429647
- Initial failed dev deploy: https://github.com/agent-station/shinjeom-web/actions/runs/35354549072
- Dev deployment: https://github.com/agent-station/shinjeom-web/actions/runs/35356122387
- Staging deployment: https://github.com/agent-station/shinjeom-web/actions/runs/35356302206
- Production deployment: https://github.com/agent-station/shinjeom-web/actions/runs/35357145623

Staging deployment and post-deploy smoke succeeded. At 23:34:58 Asia/Seoul, public home, settings, chat list and the dynamic chat shell returned HTML 200, root locale selection returned 302 to `/ko/`, and served JavaScript contained the release SHA and staging Faceless origin. Production promotion then completed with `make promote-main`. The production workflow reused the successful dev checks for this exact SHA and passed its build, S3/CloudFront deployment and post-deploy smoke. At 23:39:31 Asia/Seoul, the same public paths returned HTML 200, root locale selection returned 302 to `/ko/`, and served JavaScript contained the release SHA and `https://faceless.manshin.ai`. The concurrent dev deployment also completed successfully and its public artifacts were verified.

`make sync-branches` passed: origin/dev, origin/staging and origin/main all point to the release source. The local web checkout is clean on dev. The merged release and test-fix branches were deleted locally and remotely; unrelated branches/worktrees were retained. Web deployments use the repository's branch/Actions policy, so deployment records identify the immutable commit and workflow run, with `deployed_tag: null`.

## Prepared infrastructure and validation

Dev and production already have the CloudFront chat-shell route and `NEXT_PUBLIC_FACELESS_BASE_URL` SSM parameter. Staging matched the intended configuration. Dev/staging use `https://faceless.staging.manshin.ai`; production uses `https://faceless.manshin.ai`.

Before promotion, production media verification passed for all 51 manifest entries. Faceless preflight requests to `/v1/rooms/{uuid}/turns` returned 200 with the expected allowed origin for dev, staging and production.

## Related release work

Production API Helm revision 38 already enables 10 free answers per account and plot, and registered Apple day/monthly pass products. Faceless and Backstage are already deployed. See [API activation](2026-09-18-manshin-pass-api-activation.md).

An actual iPhone StoreKit purchase/restore has not been verified by this web deployment. At the earlier API verification, the two production plots were drafts and the Apple products still needed review metadata. These are separate from publishing the consumer web.
