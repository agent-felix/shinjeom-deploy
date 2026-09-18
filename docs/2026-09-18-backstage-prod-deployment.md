# Backstage production static deployment

Published at `2026-09-18T22:04:05+09:00`, before the production API rollout as
requested by the user. Faceless was already deployed and ready at Helm revision 2.

## Release and hosting

- Same release as staging: `shinjeom-backstage-v0.1.0-rc3`.
- Source: `696480077ebe3a9e9291996811c388eb28c2a441`, clean detached worktree.
- Production build: `./scripts/deploy.sh build prod` (TypeScript check and Vite build passed).
- API: `https://shinjeom-api.agsn.ai/api/v1`.
- Faceless: `https://faceless.manshin.ai`.
- URL: https://backstage.manshin.ai/faceless/
- AWS account: `shinjeom-prod` / `763879260205`.
- Private S3: `manshin-backstage-prod`, prefix `faceless/`, region `us-west-2`.
- CloudFront: `E326HEXH1LVR6L` / `d3km1pizpmsjgi.cloudfront.net`.
- Invalidation: `IAZSO6XO3Q1L8NRSUM4VKS3L09`, Completed.

Used the same S3 + CloudFront OAC pattern as staging. Uploaded immutable hashed
assets first and `no-cache` HTML last, retaining old assets. Hosting resources were
created in the preceding [infrastructure step](2026-09-18-text-consult-prod-preparation.md).
No API deployment, database migration, Faceless change or secret change was made here.

## Verification

- Built bundles contain the production API/Faceless addresses and no staging addresses.
- Public HTTPS returned 200; HTML content type is `text/html`, cache policy `no-cache`.
- HTML, JavaScript, CSS and font served by CloudFront match the build bytes.
- The actual production URL renders the administrator login screen in the browser,
  with no console errors or warnings.
- Command-line DNS still had a local negative cache entry, so byte comparisons
  used the CloudFront target with the original host/SNI and normal TLS validation.
  The browser opened the production domain directly.

## API dependency

Production API remains `shinjeom-api-v2.3.6`. An OPTIONS request for
`/api/v1/auth/admin/login` with Origin `https://backstage.manshin.ai` returns
400 `Disallowed CORS origin`. The login screen is deployed; authenticated login,
plot editing and preview conversations require the subsequent API/CORS/plot
database rollout and end-to-end verification.
