# Deployment Standard

This document defines the target deployment standard for Shinjeom services.

It is designed for a one-person team:

- manual and explicit
- low cognitive load
- no custom deploy wrappers
- minimal abstraction
- plain `docker`, `helm`, and `kubectl`
- AWS Secrets Manager as the secret source of truth

This standard should be applied to:

- `shinjeom-api`
- `shinjeom-agents`
- `long-term-memory`

Canonical deploy environment tokens are:

- `staging`
- `prod`

Use those tokens in scripts and operator commands. Do not mix `prod` and `production`.

## Core Rules

1. Terraform manages cloud infrastructure only.
2. Helm manages Kubernetes app releases only.
3. `kubectl` is used only for verification, logs, restart, and manual debugging.
4. Deployments are run manually by explicit commands, not by deploy wrapper scripts.
5. Narrow secret utility scripts are allowed and recommended.
6. Every deployment uses an explicit immutable release tag.
7. The Git tag and Docker image tag must be the same string.
8. `deployed_tag`, `deployed_commit`, `deployed_at`, and `updated_at` must be recorded in `shinjeom-deployments`.

## Release Tag Policy

Release tags are explicit service-prefixed semantic versions:

- `shinjeom-api-v2.0.0`
- `shinjeom-api-v2.0.1`
- `shinjeom-agents-sarang-v2.1.0`
- `long-term-memory-v3.0.0`

Optional prerelease tags are allowed when useful:

- `shinjeom-api-v2.0.0-rc1`
- `shinjeom-api-v2.0.0-rc2`

Rules:

- tags are created manually
- tags are immutable
- tags point to one exact commit
- tags use the form `<service-name>-vX.Y.Z` or `<service-name>-vX.Y.Z-rcN`
- Docker images use the exact same tag
- Helm deploys use the exact same tag
- never use `latest`
- never use `stable`
- never use branch names as image tags

Recommended meaning:

- patch: bug fix or small release
- minor: backward-compatible feature release
- major: breaking change or major version step

## Deployment State Repo

`shinjeom-deployments` is the deploy-state repo.

After each deploy:

1. update `staging.yaml` or `prod.yaml`
2. record the deployed tag
3. record the deployed commit SHA
4. record the deployed timestamp
5. record `updated_at`
6. commit that state change

This answers:

- what is live in staging?
- what is live in prod?
- what tag should be rolled back to?

## What Scripts Stay

Keep only narrow operator scripts such as:

- `put-secret.py`
- `sync-secrets.py`

These are allowed because they solve secret-handling work that is tedious and error-prone in raw CLI.

Delete deploy wrapper scripts such as:

- build + push + secret sync + helm deploy in one command

Those wrappers hide too much and create drift between services.

## Secret Management Standard

Secret flow:

1. local secret JSON file is prepared outside Git
2. `put-secret.py` uploads it to AWS Secrets Manager
3. `sync-secrets.py` copies it into the Kubernetes `Secret`
4. Helm deploy references the existing Kubernetes `Secret`

Rules:

- AWS Secrets Manager is the source of truth
- Kubernetes `Secret` is only a runtime copy
- Helm values files must not contain real secrets
- real secret JSON files must never be committed
- only example files may live in Git

## Kubernetes Targeting Standard

Every deploy command must target the cluster explicitly.

Use:

- `--kubeconfig`
- `--kube-context` when needed
- `--namespace`

Do not rely on:

- current kubectl context
- implicit default kubeconfig behavior
- namespace values hardcoded inside Helm templates

Rules:

- namespace is chosen by Helm CLI, not by chart values
- each app should have its own namespace
- avoid deploying app workloads into `default`

Recommended namespaces:

- `shinjeom-api`
- `shinjeom-agents`
- `long-term-memory`

If staging and prod are separate clusters, the namespace name can stay the same across environments.

## Helm Chart Standard

Target chart layout per service:

```text
<service>/
  helm/
    Chart.yaml
    values-staging.yaml
    values-production.yaml
    templates/
  DEPLOY.md
  SECRETS.md
```

Rules:

- one chart per deployable unit
- one values file per environment
- values files contain non-secret configuration only
- chart templates must not set `metadata.namespace`
- image tag is not edited in values files for routine deploys

The normal deploy command must pass the tag explicitly:

```bash
--set-string image.tag="$TAG"
```

## Standard Manual Deploy Flow

The manual flow should be the same shape for every app.

### 1. Prepare a release

Inside the service repo:

```bash
git checkout <branch>
git pull --ff-only
git tag -a <service-name>-v2.0.0-rc1 -m "<service-name> v2.0.0-rc1"
git push origin <service-name>-v2.0.0-rc1
```

Set variables:

```bash
export TAG=<service-name>-v2.0.0-rc1
export KUBECONFIG=~/.kube/config-staging
export NAMESPACE=<app-namespace>
```

Add `KUBE_CONTEXT` only when the kubeconfig needs an explicit context override.

### 2. Build the image

```bash
docker buildx build \
  --load \
  --platform linux/amd64 \
  -f <Dockerfile> \
  -t <local-image-name>:"$TAG" \
  <build-context>
```

### 3. Push the image

```bash
aws ecr get-login-password --region <aws-region> --profile <aws-profile> | \
  docker login --username AWS --password-stdin <ecr-registry>

docker tag <local-image-name>:"$TAG" <ecr-repository>:"$TAG"
docker push <ecr-repository>:"$TAG"
```

### 4. Sync secrets

```bash
python3 scripts/sync-secrets.py <environment> \
  --kubeconfig "$KUBECONFIG" \
  --namespace "$NAMESPACE"
```

### 5. Deploy with Helm

```bash
helm upgrade --install <release-name> <chart-path> \
  --kubeconfig "$KUBECONFIG" \
  --namespace "$NAMESPACE" \
  --create-namespace \
  -f <values-file> \
  --set-string image.tag="$TAG" \
  --atomic --wait --timeout 10m --history-max 10 \
  --description "deploy $TAG"
```

Add `--kube-context "$KUBE_CONTEXT"` only when the kubeconfig needs an explicit context override.

### 6. Verify

```bash
helm status <release-name> \
  --kubeconfig "$KUBECONFIG" \
  --namespace "$NAMESPACE"

kubectl --kubeconfig "$KUBECONFIG" --namespace "$NAMESPACE" get deploy
kubectl --kubeconfig "$KUBECONFIG" --namespace "$NAMESPACE" get pods
kubectl --kubeconfig "$KUBECONFIG" --namespace "$NAMESPACE" rollout status deploy/<deployment-name> --timeout=10m
kubectl --kubeconfig "$KUBECONFIG" --namespace "$NAMESPACE" logs deploy/<deployment-name> --tail=100
```

Add `--context "$KUBE_CONTEXT"` only when the kubeconfig needs an explicit context override.

### 7. Record deployment state

Update `shinjeom-deployments/staging.yaml` or `shinjeom-deployments/prod.yaml`:

- `deployed_tag`
- `deployed_commit`
- `deployed_at`
- `updated_at`
- optional notes

Then commit the state repo change.

## Promotion Rule

Promotion from staging to prod should reuse the same release tag.

Example:

- deploy `<service-name>-v2.0.0` to staging
- verify staging
- deploy the same `<service-name>-v2.0.0` image to prod

Do not rebuild a new image for prod from the same branch state.

Prod should receive the same artifact that already passed staging.

## Rollback Rule

Rollback uses a previously deployed immutable tag.

Example:

```bash
helm history <release-name> \
  --kubeconfig "$KUBECONFIG" \
  --namespace "$NAMESPACE"

helm rollback <release-name> <revision> \
  --kubeconfig "$KUBECONFIG" \
  --namespace "$NAMESPACE" \
  --wait --timeout 10m
```

If needed, you can also redeploy an older image tag explicitly:

```bash
helm upgrade --install <release-name> <chart-path> \
  --kubeconfig "$KUBECONFIG" \
  --namespace "$NAMESPACE" \
  -f <values-file> \
  --set-string image.tag="<service-name>-v1.9.4"
```

Add `--kube-context "$KUBE_CONTEXT"` only when the kubeconfig needs an explicit context override.

After rollback, update `shinjeom-deployments` so recorded state matches real state.

## Final Standard

The target operating model is:

- explicit service-prefixed release tags like `shinjeom-api-v2.0.0` or `shinjeom-api-v2.0.0-rc1`
- one Git tag equals one Docker image tag
- one Docker image tag equals one Helm deploy version
- secrets handled by small utility scripts only
- deployments run by plain `docker`, `helm`, and `kubectl`
- actual live state tracked in `shinjeom-deployments`

That is simple enough for one operator, but still strong enough to keep releases understandable and repeatable.
