# Staging API ECS Task 자원 확대

## 적용 내용

2026-10-01 Staging API의 Task 자원 설정만 변경했다.

| 항목 | 이전 | 적용 후 |
| --- | --- | --- |
| Task CPU | 256 units (0.25 vCPU) | 512 units (0.5 vCPU) |
| Task / API container memory limit | 512 MiB | 1024 MiB |
| Task Definition | `shinjeom-api-staging-ec2:6` | `shinjeom-api-staging-ec2:7` |
| EC2 | `t3.small` 2대 | 변경 없음 |
| Desired API Task 수 | 2 | 변경 없음 |

Task-level CPU는 EC2에서 CPU 사용 한도로 적용된다. 이번 변경은 CPU와 memory 한도를 모두 확대한다.
EC2 사양, AMI, host 예약 메모리 512 MiB, 네트워크, RDS, Secret, API image는 변경하지 않았다.

## 배포 식별 정보

- Account / Region: `344414913820` / `ap-northeast-2`
- Cluster / Service: `shinjeom-api-staging` / `shinjeom-api-staging-ec2`
- Image tag: `ecs-ec2-body-limit-1fad2ed`
- Image source commit: `1fad2ed358ba04717fbe4a917ce0b021f93c95e4`
- Image digest: `sha256:c98d45449d418ad5279b33f1e33ee60ac0c5fc509c17bef929e784e49becae16`
- Configuration: `shinjeom-api`의 `6d52f4ed961b7070c3cd076da3b19bd169da6a31` 기준 working tree에서
  `terraform/ecs/terraform.staging.tfvars`의 `task_cpu=512`, `task_memory=1024`만 변경해 적용했다.
  적용 시점에 이 변경은 commit되지 않았으며 새 image를 build하지 않았다.
- 배포 후 configuration commit: `72c92e1aafe6eec5e0f41b779c5cae0b7d3148aa`.
  적용한 tfvars와 새 아키텍처 문서를 함께 기록했다.
- 적용한 tfvars SHA-256: `67dc094fc52be2224e94ff67100615e685fe1e5246beb8747fdee5b8475729f6`
- ECS rollout 완료: `2026-10-01T17:30:29.898+09:00`
- 최종 API/DB 확인: `2026-10-01T17:31:29+09:00`

## 적용 및 검증

1. Staging AWS account와 기존 Terraform backend를 확인했다.
2. 기존 Task 2개와 ALB Target 2개가 정상이며 배포 lock이 없음을 확인했다.
3. 두 host의 ECS 등록 메모리는 각각 1401 MiB로, 기존 Task 종료 후 1024 MiB Task 배치가 가능함을 확인했다.
4. Terraform validate 성공. 사전 plan의 변경 범위는 Task Definition 교체와 Service 갱신뿐이었다.
5. `scripts/deploy_staging.py --tag ecs-ec2-body-limit-1fad2ed`로 배포했다.
   Script는 작업 pause, Terraform 적용, Task 안정화, health 확인, resume, lock 해제를 수행했고 exit 0으로 종료했다.
6. 배포 중 주기적 ALB 관측에서 healthy Target은 최소 1개였다. 연속적인 무손실 요청 처리를 검증한 것은 아니다.
7. 최종 상태:
   - revision `:7`의 Task 2개, running 2 / pending 0, rollout `COMPLETED`
   - 각 Task의 CPU `512`, memory `1024`, API container memory limit `1024`
   - 서로 다른 host와 AZ에 배치, ALB Target 2개 healthy
   - 두 Task 모두 schema `a8c3e1d7b902` 검증 및 application startup 성공
   - `/api/v1/health`, `/api/v1/bok/products` 모두 HTTP 200
   - SSM resume 응답 `{"paused":false}`, deployment lock 부재, schedules enabled 유지
   - 최종 Terraform plan `No changes` (detailed exit code 0)

| AZ | EC2 host | 새 Task ID | Target IP |
| --- | --- | --- | --- |
| ap-northeast-2a | `i-006f801bcf1d15085` | `a3d2df845829443db496a661873904c3` | `172.31.64.195` |
| ap-northeast-2c | `i-026ea10e6d5f79af1` | `4ea3f4c17df642f0a2d9e443b00befa3` | `172.31.65.15` |

## 운영 참고

- Pause는 17:25 KST경 시작했고 17:30 KST경 resume했다. 08:30 UTC payment-expiry 예정 시각은
  배포 lock 구간에 포함된다. 현재 정책상 pause 중 예정 실행은 자동 catch-up하지 않으며,
  이번 배포에서 해당 업무를 수동 실행하지 않았다.
- 테스트 스위트와 부하 테스트는 실행하지 않았다. 자원 한도 확대와 배포 정상화 확인이지
  peak 처리량을 보장하는 검증은 아니다.
- EC2의 CPU credit 정책은 Standard로 유지된다. Task CPU 확대가 EC2의 CPU credit이나
  지속 가능한 기본 CPU 성능을 높이는 것은 아니다.
- 기존 운영·migration 문서는 변경하지 않았다. 새 아키텍처 문서에는 적용한 자원 설정과
  Task-level CPU 한도 설명을 반영했다.

## 복구 기준

문제가 발생하면 tfvars의 CPU·memory를 기존 값으로 되돌리고 같은 release 절차로 적용한다.
DB downgrade나 Secret 변경은 필요하지 않다. 자원 설정 복구와 무관하게 현재 LTM `/api/v1`
endpoint와 tenant key 계약, 동일 image digest를 유지한다.
