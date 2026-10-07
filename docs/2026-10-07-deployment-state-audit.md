# Deployment state audit — 2026-10-07

Verified at: `2026-10-07T07:15:02Z`.

Read-only inspection of every service in `staging.yaml` and `prod.yaml`.
No deployment, restart, infrastructure change, App Store write, new tag, or source
repository update was performed.

## Sources and timestamp meanings

- Kubernetes: explicit `~/.kube/config-staging` and `~/.kube/config-prod`;
  live Deployment images/readiness and `helm list -A`.
- ECS: `shinjeom-staging/ap-northeast-2` and `shinjeom-prod/us-west-2`;
  DescribeServices, DescribeTaskDefinition, ListTasks and DescribeTasks.
- Web: successful GitHub Actions runs and publicly served settings JavaScript;
  S3 settings HTML LastModified cross-checked.
- Backstage: S3 HTML, its LastModified, and byte-for-byte matching public HTML.
- Mobile: read-only App Store Connect Apps, AppStoreVersions and Builds queries.
  Production version-to-build relationship was queried explicitly.

`verified_at` is the inspection time. `updated_at` is the record update time.
Neither is a deployment time. For Kubernetes, `deployed_at` now records the
current Helm release timestamp, including configuration-only upgrades.
For ECS it records the current PRIMARY deployment creation time, including
same-image redeployments. For Web it is the Actions Deploy step completion time.
Unknown source commits or deployment times are not inferred from branch HEAD,
build upload time, or S3 LastModified.

## Kubernetes services

All application Deployments inspected were at their desired Ready replica count.
Live image tags match the previous records; no application image promotion was
needed. Existing immutable Git tags resolve locally to the recorded commits.

| Service | staging image tag | prod image tag | staging / prod Helm revision |
| --- | --- | --- | --- |
| shinjeom-agents-sarang | shinjeom-agents-sarang-v3.0.10 | shinjeom-agents-sarang-v3.0.9 | 52 / 25 |
| shinjeom-agents-yeona | shinjeom-agents-yeona-v3.0.5 | shinjeom-agents-yeona-v3.0.5 | 37 / 18 |
| yeona-agent | yeona-agent-v0.1.3 | yeona-agent-v0.1.3 | 7 / 5 |
| darae-agent | darae-agent-v0.1.0-rc3 | darae-agent-v0.1.0-rc3 | 30 / 15 |
| minjun-agent | minjun-agent-v0.1.0-rc3 | minjun-agent-v0.1.0-rc3 | 21 / 15 |
| heejin-agent | heejin-agent-v0.2.0-rc5 | heejin-agent-v0.2.0-rc5 | 14 / 6 |
| ken-agent | ken-agent-v0.1.0-rc13 | ken-agent-v0.1.0-rc13 | 18 / 11 |
| mio-agent | mio-agent-v0.1.0-rc4 | mio-agent-v0.1.0-rc4 | 7 / 5 |
| long-term-memory | long-term-memory-v3.0.5-rc1 | long-term-memory-v3.0.2 | 16 / 8 |
| orrery-api | orrery-api-v0.1.5 | orrery-api-v0.1.5 | 8 / 6 |
| faceless | faceless-v0.1.0-rc14 | faceless-v0.1.0-rc14 | 18 / 8 |

Both LTM API and worker use the expected environment image tag.
Faceless application, lock Redis and history-cache Redis are Ready.
All listed staging application Deployments have one replica except LTM API (two).
Production Sarang, Yeona (shared agents repo) and LTM API have two replicas;
the other listed application Deployments have one, including report agents.
Historical rollout notes are not a current replica-count guarantee.

Helm configuration-only changes do not establish a new application source commit.
Previously recorded `chart_commit` values were not re-attested by this inspection.

## API / ECS

- Both environments: image tag `ecs-ec2-body-limit-1fad2ed`, source commit
  `1fad2ed358ba04717fbe4a917ce0b021f93c95e4`.
- Running image digest in both environments:
  `sha256:c98d45449d418ad5279b33f1e33ee60ac0c5fc509c17bef929e784e49becae16`.
- Staging Task Definition: `shinjeom-api-staging-ec2:8` (record previously said 7).
  PRIMARY deployment created `2026-10-07T14:01:50+09:00`.
- Production Task Definition: `shinjeom-api-prod-ec2:1`.
  PRIMARY deployment created `2026-10-05T22:17:30+09:00`.
- Each environment has two RUNNING Tasks, desired count two, completed rollout.
- The image did not change. Its original production promotion time is still
  unknown; the new production timestamp describes the current ECS deployment.
- Staging ECR associates the observed digest with the recorded image tag.
  This tag is an ECR tag, not an existing local immutable Git tag.
- Existing staging Terraform configuration commit/hash fields were retained,
  but were not re-attested against Task Definition revision 8.

## Web

Both environments serve version `1.1.0`, commit
`6169d929902260bc36d8aa096483cf602bb2441d`.
Public `/ko/settings/` JavaScript contains the commit and version.
No immutable release tag is recorded.

| Environment | Successful Actions run | Deploy step completed |
| --- | --- | --- |
| staging | [37579694191](https://github.com/agent-station/shinjeom-web/actions/runs/37579694191) | 2026-10-07T06:10:36Z |
| prod | [37580172559](https://github.com/agent-station/shinjeom-web/actions/runs/37580172559) | 2026-10-07T06:15:11Z |

The production record no longer attributes the current site to the September
direct deployment.

## Backstage — source attribution unavailable

The current public HTML matches the corresponding S3 object. Both entrypoints
were modified after the recorded September 18 rc3 release. No source/build
metadata accompanies the current objects, so the previous rc3 tag and commit
cannot safely describe the live artifact. Their current-state fields are now
null; the previous release remains available in Git history.

| Environment | S3 index.html LastModified | JavaScript entrypoint | HTML SHA-256 |
| --- | --- | --- | --- |
| staging | 2026-09-22T02:47:07Z | index-DxMkwwSn.js | 2383f5d2aa2112039fb75ee73c116aa551eb3ffccacb5b2f04335801cf60fe1a |
| prod | 2026-09-23T02:57:47Z | index-BcOitXsh.js | af90dd3da6d03ac33ba4557b63291e87e46b494c1d73b1a70a5ae4a8f07c8e87 |

Both reference `index-bu6zSiF1.css`. LastModified is recorded separately as
`artifact_last_modified_at`, not as `deployed_at`.
No build or deployment was performed to reconstruct provenance.

## Mobile / App Store Connect

- Production bundle: `ai.agsn.apps.shinjeom`, ASC app `6759539908`.
  Version `2.5.0` is READY_FOR_SALE / READY_FOR_DISTRIBUTION and downloadable.
  Its linked build is `8` (`465b66a4-dc35-4e85-a718-33b212b272e9`),
  uploaded `2026-09-23T02:44:49Z`.
- Staging bundle: `ai.agsn.apps.shinjeom.staging`, ASC app `6760985275`.
  Latest uploaded build is `2.5.0 (8)`
  (`ad0d9a0c-3476-4f06-b4d2-3f3b6866b1ae`),
  uploaded `2026-09-23T01:49:52Z`, VALID and not expired.
  Staging App Store version `1.0` remains PREPARE_FOR_SUBMISSION.
- A staging upload is not proof that the build is assigned to TestFlight groups;
  it is recorded as `latest_uploaded_version` / `latest_uploaded_build`,
  not as a verified deployed release.
- Neither queried build exposes Git source attribution or a release/distribution
  timestamp. `deployed_tag`, `deployed_commit` and `deployed_at` remain null.
  Upload timestamps are separate `build_uploaded_at` fields.

## Scope limits

This audit covers all services already tracked in this deployment repository,
including the LTM application located in agent-station. Cluster infrastructure
and other applications (for example avatars and llm-gateway) were not added to
the tracked service set. Non-deployable tooling/source repositories are not
assigned invented deployment versions.
