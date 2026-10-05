# Staging LTM common-contract restoration

Completed: 2026-10-01 16:02 KST. Operator authorized staging downtime/errors during the transition.
Production was not changed. Test suites were not run; only build, deployment and read-only runtime checks were performed.

## Sources and releases

| Repository | Implementation commit | Release |
| --- | --- | --- |
| agent-station | `0ce924206fae59a6b3e1b6c0cfca1011f30d480b` | `long-term-memory-v3.0.5-rc1`, Helm 16 |
| secretctl | `a06f843667f9db2f1839e4c3df00eb736acb3b4d` | local CLI rebuilt, LTM scope applied |
| shinjeom-api | `0ddd638a3a0705664430bf4e8020ae69999faef4` | configuration only, ECS Task Definition `shinjeom-api-staging-ec2:6` |
| shinjeom-faceless | `19a4f2b36ff4e44c06d8eb6f8223238da635eeaf` | chart only, Helm 15 |
| shinjeom-report-agents | `23c1d0f5e8ea51dbbde83974fb3058d63ec4617b` | chart only: Darae 30, Minjun 21, Heejin 14, Ken 18, Mio 7 |
| yeona-agent | `9cb93362727bd5596a72f7feaa4da62de206f515` | chart only, Helm 7 |

LTM image: `138649935536.dkr.ecr.ap-northeast-2.amazonaws.com/staging/long-term-memory-api:long-term-memory-v3.0.5-rc1`.
Digest: `sha256:96c3d68e81bc0ddb850c873509f25a2958778e4a99d77188480c9e64f2cfe9d1`.
Built from the clean tagged source for linux/amd64 with provenance disabled.

API image/source unchanged: `ecs-ec2-body-limit-1fad2ed`, commit `1fad2ed358ba04717fbe4a917ce0b021f93c95e4`,
digest `sha256:c98d45449d418ad5279b33f1e33ee60ac0c5fc509c17bef929e784e49becae16`.
The release script applied only a new Task Definition and Service reference, then waited for two healthy tasks on distinct hosts.

All other listed workloads retained their existing immutable images. `staging.yaml` keeps image source commits separate
from `chart_commit` / `configuration_commit`; `deployed_at` records this deployment, not the image build time.

## Changes and execution

- LTM no longer has ECS-specific routes/authentication or a read-only credential gate. Normal `/api/v1` endpoints
  use the existing API key to app-name mapping. Existing agent writers and Faceless's separate tenant are unchanged.
- LTM staging Ingress now routes `/api/v1` with `Prefix`; TLS and NAT source allowlist `3.39.49.48/32` remain.
- Fixed IP-based RPM caps removed from LTM, Faceless internal, all five Report Agent internal and Yeona Ingresses.
  Existing Faceless/Report Agent public Ingresses retain 120 RPM.
- Existing Helm release values were preserved, avoiding unrelated configuration changes. LTM updated image and Ingress;
  image-version labels/config checksums changed accordingly. Other releases changed only their Ingress annotation.
- Restored ECS runtime `MEMORY_SERVICE_API_KEY`, deployed the LTM image/chart, and released the API's common URL
  through `scripts/deploy_staging.py --tag ecs-ec2-body-limit-1fad2ed`. Brief LTM lookup errors during transition were permitted.
- After both ECS tasks verified the common API, removed the central dedicated credential and applied only the LTM Secret
  with the rebuilt secretctl CLI. Both LTM Deployments restarted and became ready.

## Secret audit (identifiers only)

No secret values are recorded. Existing key/label values were not rotated or renamed. Writes used temporary owner-only
files removed immediately after each operation, with response read-back and equality checks.

- API runtime previous version: `ad46eab4-a878-4901-93f5-4f6b7c88dddd`.
- API runtime new version: `771d6cc6-3f11-4fa6-b0f7-1af0779df42a`.
- Only `MEMORY_SERVICE_API_KEY` changed, to the existing central `shared.API_KEYS.long-term-memory.key`.
- Central previous version: `463561e5-f18e-4be1-a77e-e09953f64f32`.
- Central new version: `81ab3804-9f1e-4f32-a8ce-f890e53454b0`.
- Only `shared.API_KEYS.long-term-memory-shinjeom-api` was removed; other values were compared and preserved.
- LTM Kubernetes Secret no longer contains `ECS_SHINJEOM_API_KEY`; `API_KEYS` retains the two original credentials.
- Old CLI validate/status succeeded before modification. Restored document validated with new CLI before upload;
  new CLI validate/status/apply/status completed with final `synced`.

## Verification

- API release completed at 15:57:55 KST; two tasks, zero pending, deployment `COMPLETED`.
- ALB targets `172.31.64.151` and `172.31.65.176` healthy in different AZs.
- Public API health and DB products: HTTP 200.
- Both API containers read existing LTM thread and session events: HTTP 200, including after credential cleanup.
  No event contents or user identifiers were printed; no events were written.
- Both callers observed removed `/api/v1/ecs/...` route: HTTP 404; invalid key on common route: HTTP 401.
- Final SSM verification command: `a44901bd-b487-49a1-8a92-fa30c4080d79`, success on both hosts.
- LTM API 2/2, worker 1/1; other affected Deployments available.
- All internal Ingress RPM annotations absent, NAT allowlist preserved; all existing public RPM annotations still 120.
- ECS deployment lock absent, schedules enabled/resumed; Terraform plan exit 0 (`No changes`).
- No manual scheduled jobs, push notification sends, or generated reports were invoked for verification.

## Recovery

Do not restore API revision `:5` unchanged: it points at the removed LTM `/ecs` URL. Keep the common `/api/v1` URL
and existing tenant key when rolling back application code. A full restoration of the old dedicated contract would require
coordinated LTM image/chart and both runtime/central credential changes; old versions are audit history, not an automatic rollback recipe.
