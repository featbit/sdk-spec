# Data Storage

[Specification index](../README.md) | [General principles](general.md)

The SDK stores evaluated flag results for local reads. It does not need a local copy of targeting rules or segments.

## Stored information

Retain the flag key, selected string value, declared value type, server reason, version, and analytics metadata, including variation options and experiment eligibility when supplied. Distinguish remote results from application-supplied bootstrap values. Preserve server metadata without recalculating targeting or experiment membership.

Flag keys are case-sensitive. Keep data and synchronization cursors scoped to both the environment and the complete user context, including attributes. An SDK may choose its own storage representation; disk formats and cache-key algorithms are not shared contracts.

## Updates

| Update | Required effect |
| --- | --- |
| Full response | Replace all remote results, regardless of old versions. Missing remote flags disappear. |
| Empty full response | Clear remote results. Applicable local-only bootstrap values may remain as defined below. |
| New flag in a patch | Insert it. |
| Newer or equal version in a patch | Accept the incoming result in synchronization order. |
| Older version in a patch | Ignore it. |
| Archived result | Hide it from normal evaluation and enumeration, retaining its version against older updates. |
| Accepted active result for an archived flag | Restore it under the same version rules. |

**Equal versions must be accepted.** A segment change can change a user's evaluated value while the flag's own timestamp remains unchanged. Applying the server-side SDK rule of rejecting equal versions would miss that change.

The cursor is the maximum committed remote flag timestamp, including archive records. A full replacement recalculates it; a patch never decreases it. Empty remote data uses zero. Local bootstrap timestamps and local receive times MUST NOT advance the remote cursor.

Commit each full response or patch batch as one coherent view. Evaluation and bulk reads must not observe part of a batch or combine one user's data with another user's context. Returned objects and caller-owned inputs must not allow external mutation of committed data.

## Bootstrap and local source precedence

Bootstrap support is optional. If offered, it MUST accept local flag keys, values, and declared types and identify these values as local. Document whether values apply to a particular context or are explicit application defaults applicable to all users. Never reuse personalized bootstrap data for another context.

At startup, an explicitly supplied, applicable bootstrap set takes precedence over a persistent cache: initialize the local view from that set and request a full online refresh with cursor zero. Without bootstrap, a matching cache may supply the initial view and remote cursor. Never combine a bootstrap-replaced data set with a cursor from the displaced cache, as unchanged remote flags could then be missed.

An accepted remote result takes precedence over a local value with the same key, including when the remote result is archived. Preserve a local-only bootstrap value omitted by a full response only while that key has not been replaced by a remote result. A later remote omission must not resurrect an old local value.

Invalid bootstrap input is a configuration error. Offline startup may use bootstrap values, or an empty data set. If an SDK also supports offline cache reuse, document that behavior and apply the same context and precedence rules.

## Persistent cache (optional)

Persistence SHOULD improve startup without becoming a prerequisite for evaluation. If offered, it MUST:

- Isolate deployments/environments and full user contexts, including same-key attribute changes.
- Preserve each remote data set and its cursor consistently; reuse neither in isolation.
- Preserve logical commit order when replacing the same cache entry. Once a newer snapshot is durably stored, an older asynchronous write MUST NOT replace it, even if that write finishes later. Determine order from accepted in-memory commits, not server flag versions, cursor magnitude, or wall-clock timestamps: equal-version updates may change values, and a full replacement may reduce the cursor. Validate write authority atomically with durable replacement, or serialize writes to provide equivalent behavior. If the newer snapshot was never persisted, recovery may use the last successfully stored coherent snapshot; this does not require every update to reach disk before process termination.
- Treat corrupt, incompatible, or mismatched cache entries as unavailable and obtain fresh data.
- Continue with in-memory data when storage access fails or capacity is exhausted.
- Document retention, expiration, and how applications can clear stored user data.

A cache miss during Identify clears access to the old context's values. An outage alone must not expire usable in-memory results for the active context. Cache cleanup and optional cross-tab or cross-process sharing must preserve these boundaries.
