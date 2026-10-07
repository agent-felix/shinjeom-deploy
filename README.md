# shinjeom-deployments

Shinjeom 서비스의 `staging`과 `prod`에 **실제로 배포된 상태**를 기록하는 리포입니다.
소스 코드는 각 서비스 리포에서 관리하고, 이 리포는 환경별 artifact, commit, 배포 시각과 확인 결과만 관리합니다.

- GitHub: [agent-felix/shinjeom-deploy](https://github.com/agent-felix/shinjeom-deploy)
- 기본 branch: `main` (`dev` branch 없음)
- [staging.yaml](staging.yaml): staging 배포 상태
- [prod.yaml](prod.yaml): production 배포 상태
- `README.md`: 기록 규칙과 운영 기준

별도의 `docs/`는 유지하지 않습니다. 이전 배포 기록은 Git history에서 확인하고, 서비스별 상세 배포 절차는 각 소스 리포의 문서를 따릅니다.

## 추적 서비스

| 서비스 | 소스 리포 경로 | 배포 방식 |
| --- | --- | --- |
| `shinjeom-agents-sarang` | `../shinjeom-agents` | Kubernetes / Helm |
| `shinjeom-agents-yeona` | `../shinjeom-agents` | Kubernetes / Helm |
| `yeona-agent` | `../yeona-agent` | Kubernetes / Helm |
| `darae-agent` | `../shinjeom-report-agents` | Kubernetes / Helm |
| `minjun-agent` | `../shinjeom-report-agents` | Kubernetes / Helm |
| `heejin-agent` | `../shinjeom-report-agents` | Kubernetes / Helm |
| `ken-agent` | `../shinjeom-report-agents` | Kubernetes / Helm |
| `mio-agent` | `../shinjeom-report-agents` | Kubernetes / Helm |
| `shinjeom-api` | `../shinjeom-api` | ECS on EC2 |
| `long-term-memory` | `../agent-station/apps/long-term-memory` | Kubernetes / Helm |
| `orrery-api` | `../orrery` | Kubernetes / Helm |
| `shinjeom-mobile` | `../shinjeom-mobile` | App Store Connect / TestFlight / App Store |
| `faceless` | `../shinjeom-faceless` | Kubernetes / Helm |
| `shinjeom-backstage` | `../shinjeom-backstage` | S3 / CloudFront |
| `shinjeom-web` | `../shinjeom-web` | GitHub Actions → S3 / CloudFront |

같은 리포에서 배포해도 배포 단위가 다르면 별도로 기록합니다. LTM 소스가 `agent-station`에 있는 것과 이 배포 기록 리포의 GitHub 소유자는 별개입니다.

## 최근 확인 상태

2026-10-07 확인 기준입니다. 이후의 현재 상태는 YAML의 `verified_at`과 서비스별 값을 확인하세요.

- Web: 양쪽 환경 모두 `1.1.0`, commit `6169d929902260bc36d8aa096483cf602bb2441d`.
- API: staging `shinjeom-api-staging-ec2:8`, prod `shinjeom-api-prod-ec2:1`. 같은 image digest를 실행하며 각 환경에 두 개의 RUNNING Task가 있습니다.
- Faceless: 양쪽 환경 모두 `faceless-v0.1.0-rc14`. 애플리케이션, lock Redis, history-cache Redis의 Ready 상태를 확인했습니다.
- 나머지 추적 Kubernetes 서비스: 기존 image tag와 일치하며, 현재 Helm revision과 release timestamp를 반영했습니다.
- Mobile prod: App Store `2.5.0 (8)`, `READY_FOR_SALE` / `READY_FOR_DISTRIBUTION`.
- Mobile staging: 최신 uploaded build `2.5.0 (8)`, `VALID`. TestFlight group 배포 여부는 확인하지 않았습니다.
- Backstage: 양쪽 환경에서 공개 HTML과 S3 artifact가 일치합니다. 이전 rc3 기록 이후 파일이 변경됐지만 현재 source commit/tag를 증명할 metadata가 없어 해당 값은 `null`입니다.

이는 확인 시점의 snapshot이며, 지속적인 health monitoring 결과는 아닙니다.

## 배포 상태 기록 규칙

### 환경 단위

| 필드 | 의미 |
| --- | --- |
| `environment` | `staging` 또는 `prod` |
| `updated_at` | 배포 상태 기록을 갱신한 시각 |
| `verified_at` | 실제 환경을 마지막으로 확인한 시각 |
| `services` | 서비스별 배포 상태 |

이 리포의 환경 이름은 `staging`, `prod`로 통일합니다. 개별 서비스 도구가 `production`을 요구하면 해당 도구의 계약을 따르되 YAML에는 `prod`로 기록합니다.

### 서비스 단위

| 필드 | 의미 |
| --- | --- |
| `repo_path` | 이 리포 기준 소스 리포의 상대 경로 |
| `tracked_branch` | 참고용 소스 branch. 실제 배포 version을 증명하지 않음 |
| `deployed_tag` | 확인된 release/image tag. 없거나 확인 불가하면 `null` |
| `deployed_commit` | 실제 artifact에 대응하는 source commit SHA. 확인 불가하면 `null` |
| `deployed_at` | 현재 배포 이벤트 시각. 확인 불가하면 `null` |
| `notes` | 확인 근거, 예외, 운영상 주의사항 |

필요에 따라 `helm_revision`, `task_definition`, `chart_commit`, `deployed_version`, `deployed_build`, `asc_build_id` 등을 추가합니다. `chart_commit`과 infrastructure configuration commit은 application source commit과 구분하고, 재확인하지 않은 기존 값은 새 배포의 근거로 단정하지 않습니다.

시각은 timezone을 포함한 ISO 8601 형식으로 기록합니다. 현재 `deployed_at`의 기준은 다음과 같습니다.

- Kubernetes: 현재 Helm release timestamp. image 변경 없는 configuration-only upgrade도 포함합니다.
- ECS: 현재 PRIMARY deployment 생성 시각. 같은 image의 redeployment도 포함하며 최초 image promotion 시각과 다를 수 있습니다.
- Web: GitHub Actions의 Deploy step 완료 시각.
- Backstage / Mobile: 실제 배포·출시 시각을 확인할 수 없으면 `null`.

S3 `LastModified`는 `artifact_last_modified_at`, Mobile upload 시각은 `build_uploaded_at`으로 별도 기록합니다. staging의 최신 upload는 `latest_uploaded_version` / `latest_uploaded_build`로 기록하며 실제 TestFlight 배포와 구분합니다.

### 확인할 수 없는 값

- branch HEAD나 최신 Git tag를 실제 배포 version으로 간주하지 않습니다.
- Git tag를 만들었어도 배포하지 않았다면 환경의 배포 값을 바꾸지 않습니다.
- source commit, release tag, 배포 시각을 추정해 채우지 않습니다. `null`과 사유를 남깁니다.
- artifact가 바뀌어 이전 source attribution을 증명할 수 없으면 잘못된 기존 값을 유지하지 않습니다.
- 현재 상태는 YAML에, 과거 이력은 Git history에 남깁니다.

## Release tag와 artifact

컨테이너 서비스의 기본 정책은 immutable release tag입니다.

```text
<service-name>-vX.Y.Z
<service-name>-vX.Y.Z-rcN
```

예: `faceless-v0.1.0-rc14`, `long-term-memory-v3.0.5-rc1`.

- tag는 하나의 정확한 commit을 가리키며 생성 후 이동하거나 덮어쓰지 않습니다.
- Git tag와 Docker image tag는 같은 문자열을 사용하는 것이 기본입니다.
- `latest`, `stable`, branch 이름을 배포 image tag로 사용하지 않습니다.
- `rc`는 release candidate입니다. staging 검증 전에는 필요에 따라 prerelease tag를 사용합니다.
- patch는 bug fix, minor는 backward-compatible 기능 추가, major는 breaking change에 사용합니다.
- squash merge나 rebase로 commit이 바뀔 예정이면 final release tag를 먼저 확정하지 않습니다.

현재 예외도 사실대로 기록합니다. API의 `ecs-ec2-body-limit-1fad2ed`는 확인된 ECR image tag이며 local Git release tag는 아닙니다. ECS는 image digest와 Task Definition을 함께 확인합니다. Web은 Actions의 commit SHA, Mobile은 ASC version/build, Backstage는 확인 가능한 정적 artifact를 기준으로 기록합니다. 없는 Git tag를 만들어 과거 배포를 소급 인증하지 않습니다.

## 서비스별 배포 절차

이 README의 Helm 예시는 Kubernetes 서비스에만 적용합니다.

- API: [DEPLOY.md](../shinjeom-api/DEPLOY.md). ECS 전용 경로이며 Kubernetes recovery path는 없습니다.
- API Secret: [SECRETS.md](../shinjeom-api/docs/SECRETS.md).
- Faceless: [DEPLOY.md](../shinjeom-faceless/DEPLOY.md).
- Backstage: [DEPLOY.md](../shinjeom-backstage/DEPLOY.md).
- Web: [배포 절차](../shinjeom-web/docs/ops/deploy.md). 일반 흐름은 `dev` → `staging` → `main`의 branch promotion과 GitHub Actions입니다. 이는 Web 소스 리포의 정책이지 이 리포의 branch 정책이 아닙니다.
- Mobile: `../shinjeom-mobile`의 Fastlane 및 release 절차를 따릅니다. build upload, TestFlight distribution, App Store release를 구분합니다.

상대 경로 링크는 소스 리포들이 같은 상위 디렉터리에 있는 local checkout을 기준으로 합니다.

## Kubernetes 운영 기준

- Terraform은 infrastructure, Helm은 application release를 관리합니다.
- `kubectl`은 확인, logs, restart, 수동 debugging에 사용합니다.
- 기본적으로 명시적인 CLI 명령으로 배포합니다. 서비스에서 문서화한 scoped release 도구나 CI 경로는 해당 서비스 절차를 따릅니다.
- build, push, Secret sync, deploy 전체를 불투명한 공통 wrapper로 묶지 않습니다.
- 서비스별 chart와 환경별 values 파일을 사용하며, routine 배포의 image tag는 CLI에서 전달합니다.
- chart template에 `metadata.namespace`를 고정하지 않습니다. namespace는 Helm CLI에서 지정합니다.
- 실제 Secret은 values 파일이나 Git에 넣지 않습니다.

항상 `--kubeconfig`와 `--namespace`를 명시합니다. 필요하면 Helm에는 `--kube-context`, kubectl에는 `--context`를 추가합니다. 현재 context나 기본 namespace에 의존하지 않습니다.

```bash
export KUBECONFIG="$HOME/.kube/config-staging"
export NAMESPACE=<app-namespace>
export TAG=<service-name>-vX.Y.Z-rcN
```

서비스 리포에서 immutable tag를 생성하고 해당 commit으로 image를 build/push한 뒤, 필요한 Secret sync를 별도로 수행합니다. 아래 placeholder는 해당 서비스의 실제 값으로 바꿔야 합니다.

```bash
helm upgrade --install <release-name> <chart-path> \
  --kubeconfig "$KUBECONFIG" \
  --namespace "$NAMESPACE" \
  --create-namespace \
  -f <values-file> \
  --set-string image.tag="$TAG" \
  --atomic --wait --timeout 10m --history-max 10 \
  --description "deploy $TAG"

helm status <release-name> \
  --kubeconfig "$KUBECONFIG" --namespace "$NAMESPACE"

kubectl --kubeconfig "$KUBECONFIG" --namespace "$NAMESPACE" get deploy,pods
kubectl --kubeconfig "$KUBECONFIG" --namespace "$NAMESPACE" \
  rollout status deploy/<deployment-name> --timeout=10m
```

각 환경에서 image tag/digest, Ready 상태, Helm revision과 timestamp를 확인한 뒤 YAML을 갱신합니다. source commit은 Git tag 또는 artifact metadata로 대조합니다.

## Secret 관리

AWS Secrets Manager를 Secret의 source of truth로 사용합니다.

- Kubernetes Secret은 runtime copy입니다. `put-secret.py`, `sync-secrets.py` 같은 좁은 범위의 도구는 각 서비스의 문서에 따라 사용합니다.
- local Secret JSON과 실제 credential은 Git 밖에서 관리하고 example만 commit합니다.
- ECS API는 환경별 runtime Secret을 직접 사용합니다. Kubernetes Secret sync는 ECS Secret 값이나 실행 중인 Task를 갱신하지 않습니다.
- Secret 변경 후 runtime 반영 방식과 restart/redeployment 필요 여부는 서비스별 절차를 확인합니다.

## Promotion과 rollback

컨테이너 서비스는 staging에서 검증한 동일 tag/digest를 prod로 promotion하고, 같은 source라는 이유만으로 prod image를 다시 build하지 않습니다. 환경별 build가 필요한 정적 Web이나 Mobile은 해당 서비스의 release 절차를 따릅니다.

Kubernetes rollback 예시:

```bash
helm history <release-name> \
  --kubeconfig "$KUBECONFIG" --namespace "$NAMESPACE"

helm rollback <release-name> <revision> \
  --kubeconfig "$KUBECONFIG" --namespace "$NAMESPACE" \
  --wait --timeout 10m
```

- API는 ECS release 절차의 이전 Task Definition/image digest를 사용합니다.
- Web, Backstage, Mobile은 각 서비스의 rollback 제약을 따릅니다.
- Faceless history cache 문제는 `historyCache` 비활성화 경로를 검토합니다. `rc12`로 rollback하지 않습니다.
- rollback 후에도 실제 runtime을 확인하고 YAML을 갱신합니다. Git record를 되돌리는 것만으로 서비스가 rollback되지는 않습니다.

## 기록 갱신과 Git history

1. 실제 환경에서 배포 artifact와 runtime 상태를 확인합니다.
2. `staging.yaml` 또는 `prod.yaml`의 서비스 정보를 수정합니다.
3. `updated_at`을 갱신하고 실제 확인을 수행했다면 `verified_at`도 갱신합니다.
4. 확인 불가 항목과 예외는 `notes`에 남깁니다.
5. 변경을 commit하고 `main`에 반영합니다.

```bash
git diff --check
git add staging.yaml prod.yaml
git commit -m "chore: update deployment state"
git push origin main
```

위 push 예시는 현재 branch가 `main`일 때의 명령입니다. 작업 branch에서 변경했다면 먼저 `main`에 merge합니다. 환경을 기억하기 위한 별도의 `staging` / `prod` branch는 만들지 않습니다.

이전 상태와 삭제된 문서는 Git history에서 확인할 수 있습니다.

```bash
git log --oneline -- staging.yaml prod.yaml
git show <commit>:prod.yaml
git log --oneline -- docs/
git show <commit>:docs/<filename>.md
```
