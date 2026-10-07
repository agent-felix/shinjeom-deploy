# Faceless history cache rollout

## Release

The user approved manual staging and production deployment after merging the history-cache change into `dev`, creating `staging`, and fast-forwarding `main`. Web remains a separate PR; the previous Web client is compatible with this Faceless release.

- Tag: `faceless-v0.1.0-rc14` (immutable annotated Git tag, pushed).
- Image and chart commit: `c8bb78e4168a8a89ffed2a5fb5b885e029dbcc84`.
- OCI index digest: `sha256:e09f66e64925eaa3923e87d841b6a76cd7581b4daeb4462298a81a1e51d5d321`.
- linux/amd64 manifest digest: `sha256:b4e83eb610b4eda5ca34c9499c472bd620c6608361d1b8174c7d2308a6e431c7`.
- Built once from a clean detached worktree and pushed to staging ECR. Copied to production ECR with `docker buildx imagetools create`; both registry index digests were verified equal. No production rebuild.
- Staging: Helm revision **18**, `2026-10-07T14:25:08+09:00`.
- Production: Helm revision **8**, `2026-10-07T14:27:31+09:00`.
- Both upgrades used `--atomic --wait --timeout 10m --history-max 10`.

Preflight found staging revision 17 and production revision 7 on rc13. Production's previous deployment record still said revision 6; live revision 7 already included the approved ECS internal ingress and HTTPS API callback. The rendered release diff preserved those settings.

## Change and preflight

The release adds a separate `faceless-cache-redis` Deployment, ClusterIP Service, and NetworkPolicy plus history-cache configuration. Existing lock/outbox Redis, public/internal ingress, model, commercial usage, context compaction, and runtime Secrets were unchanged. No API/LTM deployment, DB migration, summary backfill, or Web deployment was performed.

Cache defaults: 128MiB `maxmemory`, `allkeys-lru`, 64MiB aggregate client memory limit, 256MiB Pod memory limit, no RDB/AOF persistence, 900-second snapshot TTL, 8MiB snapshot limit, 250ms cache I/O timeout. The former 5% client memory limit could evict clients while storing 5.52MB snapshots and was fixed before this release.

- Clean release checkout: **277 tests and 7 subtests passed**.
- Both environment Helm charts passed lint.
- Rendered manifests were compared with live Helm manifests before rollout.
- Stable workers had sufficient scheduling capacity for the cache and rolling application replacement; Calico was healthy in both clusters.
- Runtime Secret key names were checked to ensure cache settings were not overridden. Credential values were not printed or persisted in verification artifacts.

## Staging validation

A new isolated Faceless preview scope was created under synthetic user `2147483647`, integration `shinjeom-api`. It is not an API plot room and does not appear in customer room lists. The fixture is retained for audit; no existing conversation was modified.

Synthetic preview room: `0bebbf46-5bd5-4ac4-add5-06f1570ba90c`.

- Local internal access grant, followed by authenticated public HTTPS requests with the staging Web Origin.
- Actual Gemini SSE completion, regeneration, first-candidate selection, and idempotent replay passed; final revision 5 with one generation and two candidates.
- Deleting only this fixture's cache produced a cold read; subsequent warm read returned exactly the same history. Cache revision was 5.
- Expiring only this fixture's snapshot caused a successful authoritative rebuild.
- An isolated MemoryStore client pointing at an unavailable cache verified LTM fallback. This did not interrupt the deployed cache service or any other room. Its expected `history.cache_unavailable` warning was confined to the probe process.
- Two synthetic 8MiB payloads were concurrently published and loaded through the real cluster cache using the default 250ms timeout. Both passed with zero client evictions; probe keys were deleted afterwards.
- Application logs contained 4 cache misses, 4 hits, 2 completed turns, and 1 replay. No application turn failure, cache-unavailable/invalid warning, or traceback was observed.

## Both-environment checks and limits

- Application, lock Redis, and cache Redis were all 1/1 Ready with zero restarts. Existing lock Redis Pods were not replaced (staging age 22 days, production age 18 days).
- `/healthz` succeeded; public unauthenticated history returned 401; expected Web Origin CORS preflight succeeded.
- Application Pod could connect to both Redis instances. Cache reported `allkeys-lru`, 128MiB maxmemory, 64MiB client limit, and disabled persistence. Lock Redis retained `noeviction`.
- A connection probe from the lock Redis Pod to cache Redis timed out in both environments, while the authorized application connection succeeded. BusyBox timeout returned 143, not GNU timeout's 124; the initial harness expectation was corrected without changing any cluster setting.
- Compaction and usage enforcement remained enabled.
- Both cache instances reported zero evicted clients and keys during verification.
- At `2026-10-07T05:29:13Z`, production had no observed conversation requests or cache entries since rollout, and no turn/cache errors or traceback. **Production live conversation/cache-hit validation remains pending organic traffic.** No synthetic conversation or model call was made in production, and no existing production conversation was inspected.

Staging smoke checks and local regression tests establish functionality, not production latency or performance guarantees. There was no disruptive live cache restart/outage test; expiry and isolated-client outage fallback were exercised instead.

## Rollback and monitoring

Watch `history.read` cache status/pages/timing, `history.cache_unavailable`, `history.cache_invalid`, `history.cache_skipped`, `turn.failed`, Redis client/key eviction, Pod restarts and memory. Sustained load may still reach the bounded cache/client memory limits; fallback preserves LTM as the authority.

To disable history caching while preserving the compatible application and summary reader, use the same rc14 chart and environment values with `--set historyCache.enabled=false`. This removes only the disposable cache resources and clears the cache URL. Do not roll back to rc12 or earlier: existing summary-bearing state requires a compatible reader.
