# Manshin Pass price confirmation: direct production web release

The user requested commits for the mobile/web change and a direct S3 deployment of the web only. The repository's normal branch-promotion path was not used for this explicitly requested direct deployment. No shared branch was pushed and no staging deployment or mobile build was performed.

## Sources

- Web: `4a434316848e44fce0635daa5dc80b2dfe389818`, local branch `fix/manshin-pass-price-confirmation`.
- Mobile: `9fbb09fc`, local branch `feature/introduce-text-consult`.
- Prior production web: `2de7e0acf9bf71bf75ceca6b65f9db661b7e56bf`.

The mobile change compares StoreKit product currency with the Korean storefront and retries once if it cannot confirm the price. Persistently inconsistent prices are returned as empty display prices. The web renders localized Apple checkout guidance for those empty prices and still permits choosing and purchasing the pass. Normal prices are unchanged. This web release is compatible with existing mobile clients, but existing TestFlight clients still need a new build to use the new currency validation.

## Build and checks

The web was built from a git archive of the committed source in `/tmp/manshin-price-prod-4a434316/source`, without local `.env` overrides. It uses Node 24.19.0, the existing installed dependencies, production SSM values from `/manshin-web/prod/`, and the exact commit as `NEXT_PUBLIC_BUILD_SHA`.

- Production environment assets check, lint, typecheck: passed.
- Full web unit/component suite: 165 files, 1,244 tests passed.
- Earlier targeted mobile suite: 27 tests passed; mobile type errors unchanged at 61 compared with the baseline, no new errors.
- Production media: 51 verified, none missing or mismatched.
- Compiled code contains the new fallback guidance, release SHA, and production Faceless origin.

## Deployment

Target: `s3://manshin-web-prod`, AWS profile `shinjeom-prod`, account `763879260205`, CloudFront distribution `E1BOSM43LEQ5UV`, public host `https://manshin.ai`.

A pre-deployment backup of all non-video website objects is retained at `/tmp/manshin-price-prod-4a434316/previous-site`. The existing `scripts/deploy.sh prod` performs the upload with existing cache headers and excludes separately managed MP4 objects. Deployment logs are in `/tmp/manshin-price-prod-4a434316/deploy.log`.

Deployment and verification completed at 2026-09-19T00:26:27+09:00.

CloudFront invalidation `IENQ74S9KLUFKOMHABI65BJ41H` completed. Public `/en/`, `/ko/settings/`, `/ko/services/chat/`, and the deployed plot route `/ko/services/chat/cb9cd26faafc48b98f880088f73454d8/` returned HTTP 200 with bodies matching the local release files exactly. The Korean message, settings release SHA and chat purchase JavaScript chunks also matched byte-for-byte. Root locale negotiation returned 302 to `/ko/`. Results are retained in `/tmp/manshin-price-prod-4a434316/verification.json`.

The initial smoke request mistakenly used a dashed database UUID in the public plot URL and returned 404; the public route uses a 32-character SKU. Repeating the check with the correct public route passed. No routing configuration was changed.

## Follow-up

The web source is committed on the local fix branch and the mobile source is committed locally. These commits are not merged into the shared release branches. A future deployment from the old shared branches can overwrite this direct release, so land the web fix before the next regular promotion. Publish a new mobile TestFlight build to enable native price validation. Actual iPhone verification of that new mobile code remains outstanding.
