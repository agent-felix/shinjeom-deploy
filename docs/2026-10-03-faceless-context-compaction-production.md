# Faceless context compaction: production rollout and scoped backfill

## Approval and artifact

The user approved production deployment after staging validation and discussion of original-event preservation during failed backfills. Execution followed the reader-first rollout, one-room dry-run/backfill, verification, and automatic-compaction enablement sequence. No other room was explicitly backfilled, and no synthetic test conversation was added to production.

- Image: `743807085349.dkr.ecr.us-west-2.amazonaws.com/prod/faceless:faceless-v0.1.0-rc13`
- Image source commit: `4aefd368afe94ebdd466b2530b4b1155ed0ef57f`
- OCI image index digest: `sha256:0ef6f587df0da1e5d63cf592caceaa9cd9e949c3912928c3b80e91294d1dfbe4`
- linux/amd64 manifest digest: `sha256:e2072f608c6f161cfe566c14c7b848fdb53c5376dc92f72375557d2f4163a666`
- Final chart commit: `25a7ec2f3f74f90fb2cd61838bfe77d3c90013cf`
- Artifact copied from staging by digest using `docker buildx imagetools create`; no rebuild. Production ECR digest was verified equal to staging before rollout.
- Staging validation: [2026-10-03-faceless-context-compaction-staging.md](2026-10-03-faceless-context-compaction-staging.md).

## Deployment sequence

Release/namespace `faceless`, kubeconfig `~/.kube/config-prod`:

1. Helm revision 5, 2026-10-03 12:17:02 KST: rc13 reader with `CONTEXT_COMPACTION_ENABLED=False`.
2. Confirmed all application pods were rc13, no old terminating reader remained, summary schema support was present, and application/Redis were ready.
3. Scoped longest active room dry-run/backfill and original-event verification, detailed below.
4. Helm revision 6, 2026-10-03 12:27:19 KST: same rc13 image, `CONTEXT_COMPACTION_ENABLED=True`.
5. Repeated checkpoint, public history, source, and original-event verification after activation.

Final settings: trigger 200,000; recent 50,000; safe input 900,000; summary chunk 200,000; compaction timeout 180 seconds. Application termination grace and public ingress read/send timeouts are 360 seconds. Application and Redis were 1/1 ready with zero restarts after rollout. No API/LTM deployment, DB migration, or credential change was performed.

## Target selection and privacy

Only the longest eligible conversation room was selected from LTM. The API DB was queried through the existing tunnel to verify its owner/mode and that it was not soft-deleted. The latest generation was not pending. Integration was verified as `shinjeom-api`.

All SQL work used read-only transactions and bounded statement/lock timeouts. Backfill used the actual `scripts/backfill_context.py` runner against production LTM and the dedicated Faceless Redis via temporary service port-forwards, with the normal room lock. Credentials remained in process memory. Conversation text and summary text were not written into deployment records or result artifacts. The private manifest/results directory contains production scope identifiers and hashes, is outside Git, and uses restricted permissions; it must be treated as operational data.

The target is intentionally not identified in this committed document. At execution it had grown beyond the preceding audit:

- 1,096 generations, 2,191 original events/revision.
- One abandoned historical input generation without a candidate.
- No existing summary.
- Dry-run provider-envelope budget: 858,283 tokens.

## Backfill outcome

- Dry-run completed without a summary write.
- Apply completed on the first invocation in 81,332 ms, with six successful Gemini summary chunks and six state-version increments.
- Checkpoint covers 1,050 generations; 46 recent generations remained raw at completion.
- Returned compacted input-budget estimate: 78,697 tokens.
- `status=completed`; no retry or automatic application of other manifest entries.
- Every original event's ID, timestamp, and content remained unchanged, verified by a canonical SHA-256 digest of the complete original event set before and after.
- The immutable source snapshot hash was unchanged.
- Checkpoint prefix digest and reusable-summary validation passed against reconstructed current history.
- Access reissue and authenticated public history retrieval accepted the summary-bearing state.

The pre-backfill budget used the provider tokenizer because it exceeded the exact-check threshold; the post-backfill value is the normal local estimate. These are prompt-envelope budgets, not claims about exact Interactions usage.

## Concurrent user activity and verification correction

During the room lock, one user request received `409 room_busy`, an expected temporary consequence of synchronous backfill. After the lock was released, the real user's next request completed successfully in approximately 4.8 seconds while the service was still in reader-only mode.

The initial verification harness compared the public generation count for exact equality with a preceding history read. That assertion failed because the user's new turn appended a requested event at revision 2,192 and a completed event at revision 2,193 between reads. Automatic compaction was kept disabled while investigating.

Read-only inspection confirmed:

- All original 2,191 events still had exactly the baseline digest.
- Exactly two new legitimate user-turn events accounted for the difference.
- Current history had 1,097 generations and revision 2,193.
- No failed generation was logged for the observed subsequent turn.

Verification was rerun with monotonic public count/revision checks to allow concurrent appends, while still requiring exact identity for every baseline event and the immutable source. It passed before activation and again at 2026-10-03 12:28:32 KST after activation. Current compacted budget at that check was 79,762 tokens, the summary checkpoint was reusable, and public history reported total 1,097 / revision 2,193.

This was a verification-harness concurrency assumption, not a backfill failure or original-event mutation. Summary fact accuracy was not manually reviewed; the checks establish structured output validity, durable checkpoint reuse, original preservation, and continued user-turn processing.

## Rollback and remaining monitoring

Production now contains summary-bearing state. **Do not roll back to rc12 / Helm revision 4 or earlier.** To stop new automatic summaries, keep rc13 and disable writes explicitly (production values now enable them):

```sh
helm upgrade faceless deploy/helm/faceless \
  --kubeconfig "$HOME/.kube/config-prod" --namespace faceless \
  -f deploy/helm/faceless/values-production.yaml \
  --set-string image.tag=faceless-v0.1.0-rc13 \
  --set-string config.CONTEXT_COMPACTION_ENABLED=False \
  --atomic --wait --timeout 10m --history-max 10
```

Existing valid summaries remain readable when automatic writes are disabled. Observe `context.compaction_failed`, `turn.failed`, `turn_timeout`, lock contention, response latency, and long-term summary quality. Further backfill is not required by this deployment; newly growing rooms can compact on demand under the normal trigger.
