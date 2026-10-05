# Production API 전환 전 실물 기록

조회: 2026-10-02 17시 KST 전후, read-only. **배포 기록이 아니라 전환 전 확인값이다.**
실행 순서는 [Production 전환 계획](../../shinjeom-api/docs/ECS_PRODUCTION_MIGRATION_PLAN.md)을 따른다.
아래 값은 작업 직전 다시 확인한다.

## 기존 API와 image

- Kubernetes: `shinjeom-api-v2.5.0-rc6`, Pod 2개 Running/restart 0, 같은 node에 배치.
- CronJob 4개 활성화, 조회 시 active 0. Daily-push `30 17 * * *` UTC.
- ECR: `agsn-prod` / `us-west-2` / `prod/shinjeom-api`.
- rc6 tag digest: `sha256:6af890fd1f6335c156f1539221b458301e7a8ee45755ddffe40a4de253421211`.
- Build provenance, 배포 시각, 현재 DB revision은 이번에 확인하지 않음.

## DNS 복구 기준값

| 항목 | 값 |
| --- | --- |
| AWS profile | `agsn-prod` |
| Route53 hosted zone | `Z03295722P1D5HGR0VFNR` (`agsn.ai`) |
| Record | `shinjeom-api.agsn.ai.` / A alias |
| Alias target | `afb32414f31d34239be7493b0cbfe324-a869ef73d9aa192c.elb.us-west-2.amazonaws.com.` |
| Alias target zone ID | `Z18D5FSROUN65G` |
| EvaluateTargetHealth | `false` |

Record의 Terraform state 소유권은 미확인이다. Import/소유권 정리와 원래 alias 복구 plan을 먼저 확정한다.
`api_dns_cutover=false` 적용은 원래 alias 복구가 아니라 record 삭제가 될 수 있다.

## 기존 DB와 신규 계정

- RDS: `shinjeom-prod-db`, `db.t4g.small`, Single-AZ/public, backup 7일.
- VPC: `vpc-0ba1af08ffdfabffc`. RDS SG: `sg-0fa4d0837a7a14b2f`.
- 5432 inbound는 기존 K8s NAT `16.145.41.228/32`만 허용. 새 ECS Task SG 허용 필요.
- LatestRestorableTime: `2026-10-02T08:00:28Z`. 실제 복구 시험은 하지 않음.
- `shinjeom-prod` / `763879260205` / `us-west-2`: ECS cluster·ELBv2 Load Balancer·ECR repository·NAT Gateway 없음.
- 신규 SES는 sandbox, quota 200/day·1/sec. 기존 운영 SES credential의 계정 귀속/실제 발송은 미검증.

이전 `prod.yaml`의 rc3 기록은 실제 rc6와 달라 정정했다. 기존 rc3의 배포 정보는 해당 notes와 Git 이력에 남긴다.
