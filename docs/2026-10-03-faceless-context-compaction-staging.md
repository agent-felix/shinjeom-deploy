# Faceless context compaction: staging rollout and validation

## Scope and release

User approved staging steps 1–3 only: drain-time correction, summary-compatible reader rollout, and isolated compaction/backfill validation followed by enabling staging defaults. Production was not deployed or backfilled. No existing user room was used as a test fixture.

- Image: `138649935536.dkr.ecr.ap-northeast-2.amazonaws.com/staging/faceless:faceless-v0.1.0-rc13`
- Image source commit: `4aefd368afe94ebdd466b2530b4b1155ed0ef57f`
- OCI image index digest: `sha256:0ef6f587df0da1e5d63cf592caceaa9cd9e949c3912928c3b80e91294d1dfbe4`
- linux/amd64 image manifest digest: `sha256:e2072f608c6f161cfe566c14c7b848fdb53c5376dc92f72375557d2f4163a666`
- Final chart commit: `7a4ad8e41288f46bc5ae6ff99e3216d29c187f0e`
- Namespace/release: `faceless`, kubeconfig: `~/.kube/config-staging`
- Helm revision 16: reader rollout, `CONTEXT_COMPACTION_ENABLED=False`, 2026-10-03 12:02 KST.
- Helm revision 17: same image, default compaction enabled, 2026-10-03 12:09 KST.
- Final application and Redis readiness: 1/1 each.

`terminationGracePeriodSeconds` increased from 180 to 360 to cover the default 300-second combined turn/compaction deadline. Public ingress read/send timeouts are both 360 seconds. The base chart still disables automatic summary writes; staging values now explicitly enable them. Production values were not changed.

## Pre-deployment checks

- `uv run python -m pytest -q`: 242 passed.
- Staging Helm lint: passed.
- Build: linux/amd64, locked dependencies, new ECR tag, source revision OCI label.
- Following revision 16, the running pod reported summary-reader support and compaction disabled before any summaries were written.

## Isolated test fixtures

Two new Faceless-only preview scopes were created under synthetic user ID `2147483647`, integration `shinjeom-api`. They have synthetic plot snapshots and conversation events, and do not correspond to Shinjeom API `plot_rooms` or real users. Preview mode bypasses commercial usage and room-list callbacks. They will not appear in the Web/Backstage room lists.

| Test | Room ID | Seed generations | Seed event revision |
| --- | --- | ---: | ---: |
| Reader/backfill/recovery | `ae160343-14ba-42da-9ba7-c307bf5d2bdd` | 16 | 31 |
| Automatic default-budget compaction | `af4d6171-6e46-44a7-8e93-bef905ec205c` | 48 | 95 |

Each fixture deliberately starts with an abandoned input generation without a candidate, reproducing the previously identified P1 condition. Only that early input contains the synthetic promise code `MAPLE47`. A later real Gemini answer had to recall it after summarization.

A local validation harness used temporary port-forwards to staging Faceless, LTM, and the dedicated Redis. Credentials were loaded into process memory from staging configuration; none were written into this document or test result files. Forwarders were closed after validation. Public generation/history/selection/suggestion requests went through `https://faceless.staging.manshin.ai`; internal access grants used the forwarded service endpoint. Fixture events were seeded through Faceless's event writer under its normal room lease, not by SQL writes.

## Reader and real backfill validation

Completed 2026-10-03 12:08 KST, while service-side automatic compaction remained disabled.

- Normal SSE generation succeeded without writing a summary.
- The actual `scripts/backfill_context.py` runner was exercised against staging LTM/Redis using a single-room manifest.
- Only the isolated runner used reduced budgets: trigger 24,000; recent 4,000; summary target 1,500; summary maximum 3,000; summary output 6,000; chunk 12,000. Cluster budgets were not lowered.
- Dry-run reported 92,248 input-budget tokens and performed no summary write.
- The runner's second summary call deliberately raised `GenerationFailed` after one actual Gemini summary. This was local fault injection, not a provider outage or a shared-service fault.
- The first checkpoint persisted through 3 generations; the runner returned `retry_needed` with safe original-context fallback (59,046 provider-envelope budget).
- Re-running with the real model resumed from the stored summary, completed through 16 generations, and returned `completed` with budget 18,484.
- Full generation-history digest and public event revision were unchanged by dry-run/backfill/recovery. The original source snapshot was unchanged.
- `--resume` skipped the successfully completed manifest entry.
- A subsequent live SSE answer recalled the synthetic fact from the abandoned input.
- Replaying the same request added no event.
- Regeneration, candidate selection, three suggestions, access reissue, and history reload passed.
- Those subsequent operations reused the same summary checkpoint.
- Final public revision: 38.

## Automatic compaction with staging defaults

Completed 2026-10-03 12:10 KST, after revision 17 enabled normal staging settings:

- Trigger: 200,000; recent history: 50,000; safe input: 900,000.
- The new fixture had an initial local input budget of 382,128.
- Its next public SSE request automatically triggered compaction at budget 382,263.
- Two actual Gemini summary chunks persisted checkpoints through 27 and then 42 generations.
- Compaction completed at budget 64,687, approximately 8 seconds after the start log.
- The answer recalled `MAPLE47`; the stored prefix digest matched reconstructed selected history.
- Same-request replay, regeneration, candidate selection, suggestions, access reissue, and history reload passed again with checkpoint reuse.
- Final public revision: 100.
- During this validation window, the new pod logged one compaction start/completion, two summary saves, zero compaction failures, and zero failed turns.

This validates live Faceless/LTM/Gemini behavior and the public HTTP/SSE path. It is not a browser UI or production billing end-to-end test. Large production-room summary quality and timing remain untested.

## Manual testing and rollback

Staging is left enabled at the normal 200,000-token trigger for user testing. Existing staging rooms were much smaller than this threshold at the preceding audit, so ordinary short conversations will not visibly trigger a new summary. To exercise compaction from the UI, choose an explicitly approved staging test room and run a scoped backfill with reduced runner-only budgets, or grow a test room past the normal threshold. Do not globally lower production budgets or import production conversation content for testing.

Summary-bearing state now exists in staging. Do not roll back to rc12 / Helm revision 15 or earlier: the old reader rejects `context_summary`.

To stop new automatic writes while preserving readable summaries, retain rc13 and explicitly override staging's enabled value:

```sh
helm upgrade faceless deploy/helm/faceless \
  --kubeconfig "$HOME/.kube/config-staging" --namespace faceless \
  -f deploy/helm/faceless/values-staging.yaml \
  --set-string image.tag=faceless-v0.1.0-rc13 \
  --set-string config.CONTEXT_COMPACTION_ENABLED=False \
  --atomic --wait --timeout 10m --history-max 10
```

The two synthetic preview fixtures are retained for audit/recovery checks; no existing room was modified or removed. Production remains on rc12 and requires its own explicit approval, reader-first rollout, and scoped long-room backfill.
