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
- Helm release `faceless`, namespace `faceless`, revision 1 installed at
  `2026-09-18T21:32:07+09:00`. Dedicated Redis is ready (1/1). Application replicas
  are deliberately 0 pending its dedicated Gemini key. This is not a live chat release.
- Faceless production values use the stable node pool. Usage approval remains disabled
  until the new production API is deployed.
- Created `manshin-backstage-prod` S3 with public access blocked, AES256 encryption
  and bucket-owner-enforced ownership, plus CloudFront OAC, cache policy and edge function.
- Requested Backstage ACM certificate in `us-east-1`:
  `arn:aws:acm:us-east-1:763879260205:certificate/5a764f32-42a1-47aa-a51e-cf01d075404d`.
- Backstage production state: `s3://shinjeom-tfstate-prod/shinjeom-backstage/terraform.tfstate`.
- CloudFront distribution and S3 origin policy await external DNS certificate validation.
  No Backstage frontend files have been published in this step.

## DNS input required

`manshin.ai` uses Cloudflare, not Route53. Backstage Terraform now supports
`manage_dns = false` for production; staging retains its existing Route53 resources.
The existing Web operations guide `shinjeom-web/docs/ops/dns-cloudflare.md` already
uses manual Cloudflare registration for production. Existing `*.agsn.ai` services
and the staging zone use Route53. Cloudflare access is needed only to add the new
records; ordinary application redeployments do not need a Cloudflare login.

Add these CNAME records with Cloudflare proxy disabled (DNS only):

| Name | Target |
| --- | --- |
| `_c6fb1035c2803a12ec5a260f8e83d3d4.backstage.manshin.ai` | `_beab4a3c31408fb16f15cb97f24dac6d.wzccmgtwzk.acm-validations.aws` |
| `faceless.manshin.ai` | `afb32414f31d34239be7493b0cbfe324-a869ef73d9aa192c.elb.us-west-2.amazonaws.com` |

After ACM validation, run a fresh full Backstage Terraform plan/apply using the
production backend and vars. Add `backstage.manshin.ai` CNAME to the resulting
`backstage_dns_target` output, also DNS only. Do not change the existing apex/www records.
Faceless's cert-manager HTTP-01 certificate can then finish issuing.

## Runtime secrets input required

The latest `secretctl` source already supports Faceless (`4e4404d`, `7182c50`);
no Java feature change is necessary. Rebuilt CLI and all 31 tests passed.
The production central document currently lacks the Faceless entries. The user
chose a new dedicated production Gemini key rather than reusing another app's key.

Required fields:

- `shared.API_KEYS.faceless`: new key, label `shinjeom-api`.
- `shared.API_KEYS.long-term-memory-faceless`: separate new key, label `faceless`.
- `apps.faceless.CHAT_TOKEN_SECRET`: new random signing secret (at least 32 bytes).
- `apps.faceless.GEMINI_API_KEY`: user-provided production key.

An ephemeral editor helper `/tmp/shinjeom-faceless-prod-editor.py` was prepared to
generate only missing internal keys and open vi at the Faceless Gemini field.
It contains no credentials. Run from `secretctl`:

```sh
./secretctl edit --env prod --scope all --no-apply \
  --editor "python3 /tmp/shinjeom-faceless-prod-editor.py"
```

If the temporary helper is gone, use ordinary `secretctl edit --env prod --scope all
--no-apply` and fill these fields with fresh secrets. Never put secret values in this report.
The editor command only updates AWS after validation and operator confirmation.

After saving, validate and compare the selected runtime scopes before applying.
Preserve existing production credentials. Apply LTM first so the dedicated Faceless
scope is recognized, then prepare API/Faceless Secrets:

```sh
./secretctl validate --env prod
./secretctl apply --env prod --scope long-term-memory
./secretctl apply --env prod --scope shinjeom-api --no-restart
./secretctl apply --env prod --scope faceless --no-restart
```

The API application remains at `shinjeom-api-v2.3.6`; its next application rollout
will load the prepared key. No API DB migration or Apple configuration was changed.

From `shinjeom-faceless`, remove the temporary replica override by applying the
same image with production values (do not use `--reuse-values`):

```sh
helm upgrade --install faceless deploy/helm/faceless \
  --kubeconfig "$HOME/.kube/config-prod" --namespace faceless \
  -f deploy/helm/faceless/values-production.yaml \
  --set-string image.tag=faceless-v0.1.0-rc10 \
  --atomic --wait --timeout 5m --history-max 10
```

Verify runtime readiness, LTM authentication, Gemini access, HTTPS and public route
restrictions. Record the running replica count, final Helm revision and verification
in `prod.yaml`. Keep usage approval disabled until production API supports it.

## Verification performed

- ECR staging/prod image digest equality verified.
- Full production cluster Terraform plan after the ECR apply: no resource changes.
- Backstage Terraform validation passed; staging plan: no resource changes.
- Backstage bootstrap: 8 resources created, none changed/deleted. Remaining full
  plan contains certificate validation, CloudFront distribution and S3 origin policy.
- Production Faceless Helm lint passed; Redis 1/1, application 0/0 as intended.
- Cloudflare browser showed a login page; DNS changes have not been performed.
- Runtime secret update and authenticated AI conversation checks are pending user input.
