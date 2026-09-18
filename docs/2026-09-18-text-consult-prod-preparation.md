# Text-consult production infrastructure preparation

This records step 1 of the production rollout on 2026-09-18. API application/DB
deployment, Apple product registration, consumer Web and mobile release are separate steps.

Source commits: Backstage `a5711c5`, Faceless deployment configuration `0babe98`,
central cluster ECR configuration `5c34c9b`. Runtime image remains the existing
`faceless-v0.1.0-rc10` tag at `f73240a`.

## Prepared

- Central production AWS account: `agsn-prod` / `743807085349`, `us-west-2`.
- Hosting AWS account: `shinjeom-prod` / `763879260205`, `us-west-2`.
- Terraform created immutable `prod/faceless` ECR and its lifecycle policy in the existing cluster state.
- Copied `faceless-v0.1.0-rc10` from staging without rebuilding. Both registries contain
  `sha256:c828b04d9333812337c027878c93569a3b0bb00086f3f0e537aad9a6c42a5730`.
- Helm release `faceless`, namespace `faceless`, revision 2 deployed at
  `2026-09-18T21:52:54+09:00`. Application and dedicated Redis are both ready (1/1).
  This prepares the runtime; the consumer text-consult rollout is still pending.
- Faceless production values use the stable node pool. Usage approval remains disabled
  until the new production API is deployed.
- Registered `faceless.manshin.ai` in Cloudflare. The cert-manager certificate is Ready;
  verified the domain's trusted TLS certificate through the production load balancer.
- Created `manshin-backstage-prod` S3 with public access blocked, AES256 encryption
  and bucket-owner-enforced ownership, plus CloudFront OAC, cache policy and edge function.
- Issued Backstage ACM certificate in `us-east-1`:
  `arn:aws:acm:us-east-1:763879260205:certificate/5a764f32-42a1-47aa-a51e-cf01d075404d`.
- Backstage production state: `s3://shinjeom-tfstate-prod/shinjeom-backstage/terraform.tfstate`.
- CloudFront distribution `E326HEXH1LVR6L` is Deployed, serving
  `d3km1pizpmsjgi.cloudfront.net`; its S3 origin policy is applied.
- No Backstage frontend files have been published in this step. The private bucket
  is empty, so `/faceless/` returns 403; the edge function redirects `/` to `/faceless/`.

## DNS registration

`manshin.ai` uses Cloudflare, not Route53. Backstage Terraform now supports
`manage_dns = false` for production; staging retains its existing Route53 resources.
The existing Web operations guide `shinjeom-web/docs/ops/dns-cloudflare.md` already
uses manual Cloudflare registration for production. Existing `*.agsn.ai` services
and the staging zone use Route53. Cloudflare access is needed only to add the new
records; ordinary application redeployments do not need a Cloudflare login.

Registered these CNAME records with Cloudflare proxy disabled (DNS only), and
verified them through authoritative DNS and public resolvers:

| Name | Target |
| --- | --- |
| `_c6fb1035c2803a12ec5a260f8e83d3d4.backstage.manshin.ai` | `_beab4a3c31408fb16f15cb97f24dac6d.wzccmgtwzk.acm-validations.aws` |
| `faceless.manshin.ai` | `afb32414f31d34239be7493b0cbfe324-a869ef73d9aa192c.elb.us-west-2.amazonaws.com` |
| `backstage.manshin.ai` | `d3km1pizpmsjgi.cloudfront.net` |

The full Backstage Terraform apply and subsequent no-change plan completed.
Existing apex/www records and staging delegation were preserved.
Faceless's cert-manager HTTP-01 certificate has finished issuing. Trusted HTTPS
was checked with the requested host/SNI through each registered origin; local
macOS DNS still had negative cache entries during initial verification.

## Runtime secrets applied

The latest `secretctl` source already supports Faceless (`4e4404d`, `7182c50`);
no Java feature change is necessary. Rebuilt CLI and all 31 tests passed.
The user saved a new dedicated production Gemini key through secretctl. AWS secret
`agent-station/prod/runtime` version `7bd32774-a674-46b0-be26-5704ef74d557` contains
the new Faceless fields. Compared with the previous version, only these paths changed:
`/apps/faceless`, `/shared/API_KEYS/faceless`, `/shared/API_KEYS/long-term-memory-faceless`.

Configured fields:

- `shared.API_KEYS.faceless`: new key, label `shinjeom-api`.
- `shared.API_KEYS.long-term-memory-faceless`: separate new key, label `faceless`.
- `apps.faceless.CHAT_TOKEN_SECRET`: new random signing secret (at least 32 bytes).
- `apps.faceless.GEMINI_API_KEY`: user-provided dedicated production key.

Existing application credentials, LTM's original key and all existing API service
keys were compared and preserved. Applied the LTM Secret and rolled its API and
worker; prepared the API and Faceless Secrets with `--no-restart`. All three scopes
report `synced` in secretctl. LTM API is 2/2 ready and its worker is 1/1 ready.

During the LTM rolling update, the existing stable node had insufficient CPU for
the surge pod. The existing cluster autoscaler increased the stable worker ASG
from 1 to 2 instances; the new node joined Ready and the rollout completed.
No manual capacity or rollout-strategy change was made.

The API application remains at `shinjeom-api-v2.3.6`; its next application rollout
will load the prepared key. No API DB migration or Apple configuration was changed.

Removed the initial replica-zero override by applying the same Faceless image
with production values (without `--reuse-values`):

```sh
helm upgrade --install faceless deploy/helm/faceless \
  --kubeconfig "$HOME/.kube/config-prod" --namespace faceless \
  -f deploy/helm/faceless/values-production.yaml \
  --set-string image.tag=faceless-v0.1.0-rc10 \
  --atomic --wait --timeout 5m --history-max 10
```

Keep usage approval disabled until production API supports it. API application/DB
rollout, Apple production purchase configuration, Backstage/Web publication and
mobile distribution still need their subsequent release steps.

## Verification performed

- ECR staging/prod image digest equality verified.
- Full production cluster Terraform plan after the ECR apply: no resource changes.
- Backstage Terraform validation passed; staging plan: no resource changes.
- Backstage bootstrap created 8 resources, then the full apply created the remaining
  3 resources. None changed/deleted. Final production plan: no changes.
- Production Faceless Helm lint passed; Redis 1/1, application 1/1, revision 2.
- Cloudflare login completed. All three CNAME records are registered as DNS only.
- Faceless HTTP-01 challenge returned the expected response; certificate is Ready and
  the trusted TLS handshake for `faceless.manshin.ai` succeeds (TLS 1.3).
- Faceless internal health: 200. Dedicated LTM key: 404 for a nonexistent probe
  state; deliberately invalid key: 401. No probe conversation/state was written.
- Gemini `gemini-3.8-flash` completed an actual Interactions API stream using the
  running application's client and new key (`store=false`, generic JSON probe).
- Public room history without a token: 401; internal access-grant path: 404.
- Backstage trusted HTTPS and root redirect verified. Frontend publication is pending.
- Existing production API remains `shinjeom-api-v2.3.6`, 2/2 ready; no API restart,
  DB migration or Apple configuration change in this step.
