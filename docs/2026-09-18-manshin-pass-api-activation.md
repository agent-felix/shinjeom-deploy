# Manshin Pass production API activation

Verified on 2026-09-18 at 23:06:14 Asia/Seoul. The user explicitly requested production free limits now, before App Review, and clarified that they will deploy the consumer web themselves.

## API deployment

- Configuration PR: https://github.com/agent-station/shinjeom-api/pull/277, merged into dev as `7630f76e2b90969c44813152017ab3dc2e7bd4c5`.
- Helm revision: **38**, status deployed, both API pods ready with zero restarts.
- Existing image: `shinjeom-api-v2.4.0`, source `ff0e666808494a58de5f92f9723db8d0ddb4a625`, unchanged digest `sha256:7964975bad9e54ab3a79baf0cdd2d51ea72042861f83953e79763ed9d0ea55c4`.
- `TEXT_CONSULT_LIMITS_ENABLED=True`, `TEXT_CONSULT_FREE_LIMIT=10`, `TEXT_CONSULT_APPLE_SALES_ENABLED=True`.
- Day product: `ai.agsn.apps.shinjeom.pass.day`; monthly product: `ai.agsn.apps.shinjeom.pass.monthly`.
- Bundle: `ai.agsn.apps.shinjeom`; Apple app ID `6759539908`; Xcode mode disabled.
- Existing team IAP credentials were validated for the production app in both Apple environments, then copied through secretctl into the three previously empty production Apple fields. Other secret fields were preserved. AWS runtime secret version: `0c6fcceb-c18f-451e-9dda-ddd857199492`.
- Faceless already has usage admission enabled; no Faceless redeployment was needed.

The free allowance counts successful saved AI answers per account and plot, including regeneration, within a 24-hour window beginning with first free use. It is not a midnight reset. The day pass is non-renewing for 24 hours; the monthly plan is an Apple one-month auto-renewable subscription.

## Verification

- Production Helm lint and diff whitespace validation passed.
- secretctl reports the production Kubernetes Secret synced with AWS.
- Runtime settings confirm both flags, limit 10, exact registered product IDs, all three Apple credentials configured, and Xcode mode disabled. No secret values were printed.
- DB migration head remains `f82d3a0b5c41`.
- Public API health returned healthy; both new pods had no ERROR or traceback entries in the checked logs.
- Apple Sandbox and Production TEST notifications were sent to the production app's configured webhook and both reported `SUCCESS`. The first immediate status query was not yet available; subsequent tests handled Apple's pending status and confirmed delivery.
- This verifies server notification delivery, not an actual StoreKit purchase or restore on an iPhone.

## Consumer release handoff

The consumer web was **not merged, promoted or deployed**. The user will handle it. Before that clarification, preparation had created https://github.com/agent-station/shinjeom-web/pull/97 from `feature/manshin-pass-release` at `0500594590a9dc6fbad1754fc51a82e1acdb2429`. It contains the existing text-consult implementation, web version 1.1.0, and dev/prod Faceless origin configuration. Local assets, lint (zero errors, existing warnings), typecheck and 1,241 unit tests passed. PR CI completion was not verified after the user stopped web work.

The following web infrastructure preparation had already been applied: dev and prod CloudFront chat-path shell routing and `NEXT_PUBLIC_FACELESS_BASE_URL` SSM parameters. Production uses `https://faceless.manshin.ai`; dev uses the staging Faceless origin. Staging already matched the intended state. Terraform was targeted to these resources after full plans exposed unrelated changes, preserving the existing Toss setting and other resources. No web S3 bundle was uploaded and no web branch promotion was performed.

App Store Connect read-only checks found production TestFlight **2.5.0 (4)** valid and in internal beta testing. Both pass products exist at the configured IDs but are `MISSING_METADATA`; review screenshots and review submission remain outstanding. The existing review account and notes are configured. No mobile build or App Review submission was performed in this task.

At verification, production contained two draft plots, `장군신, 다함` and `배우, 서도훈`, with **zero published plots**. Neither was published by this task. A published plot and the consumer web release are needed to exercise the full reviewer-facing chat-to-purchase flow.

## Branch cleanup and rollback

The API checkout is clean on dev and matches origin/dev. The merged API launch branch was deleted locally and remotely. The web preparation branch/PR and existing mobile working-tree changes were left in place for the user.

If the configuration must be reverted, use the explicit production cluster/namespace and Helm revision **37**. The application image and database schema are unchanged. Retaining the valid Apple verification credentials does not enable sales when the sales flag is disabled.

## Subsequent web deployment

After this API handoff, the user explicitly authorized PR #97 and staging-to-production web deployment, then requested staging promotion without waiting for dev deployment completion. Web 1.1.0 at `2de7e0acf9bf71bf75ceca6b65f9db661b7e56bf` was verified in production at 23:39:31 Asia/Seoul. See [the web release record](2026-09-18-manshin-pass-web-release.md) for PR #98's test-only follow-up, successful workflow runs and public artifact checks. The earlier handoff paragraphs above describe the state at the API activation time.
