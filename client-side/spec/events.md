# Event Processing

[Specification index](../README.md) | [General principles](general.md)

Events support usage analytics and experimentation. Collection must remain inexpensive; delivery runs independently of local evaluation and synchronization.

## What is recorded

| Operation or condition | Event behavior |
| --- | --- |
| Successful individual evaluation of a remote result | Record after conversion succeeds, once the active context has completed online synchronization. |
| Matching remote cache before the active context's first successful synchronization | Return usable values, but suppress evaluation events until synchronization confirms that context. |
| Stale remote result after successful synchronization | Remains eligible; a temporary outage does not change its origin. |
| Local bootstrap result | No evaluation event, even after other flags synchronize. |
| Fallback, conversion failure, or bulk read | No evaluation event. |
| Track | Record a named metric for the active user; numeric value defaults to 1.0. Initialization is not required. |
| Offline, events disabled, terminal event-delivery failure, or client closing/closed | Suppress new event collection. Offline, disabled, and terminal-delivery states also suppress delivery. |

Capture the user, attributes, value, and timestamp at the call. Later Identify calls, input mutation, and retries MUST NOT change that event.

For evaluation events, use the server's selected variation metadata and `sendToExperiment` decision. Do not perform experiment sampling locally. If the selected variation cannot be mapped to valid analytics metadata, omit the event and report a diagnostic while preserving the evaluated value.

Track requires a non-empty event name and a finite numeric value. Reject invalid input without affecting evaluation or synchronization.

## Collection and deduplication

Use bounded buffering without waiting for network delivery. When capacity is exhausted, drop the new event and make the loss observable. Bound total retained work, payload sizes, batch sizes, and delivery concurrency.

The client-side event contract requires **deduplication within each flush group**:

- Events are duplicates when their complete logical payloads are equal except for timestamp. Compare user attributes by meaning, not object property order.
- Keep the first occurrence and its original timestamp. Apply this to evaluation events and custom metric events separately.
- Different users, attributes, flag variations, experiment eligibility, event names, or metric values are not duplicates.
- Split into delivery batches after deduplication. Do not deduplicate across separate flush groups.

For example, two identical Track calls in one flush group produce one metric payload; the same call in a later group produces another. **Track therefore does not guarantee one delivered event per invocation.** This behavior differs from the server-side event contract and must be documented for applications that count occurrences.

Send periodically and on explicit flush. Additional size-based triggers MAY be offered. Exact intervals and batch sizes are SDK choices. Invalid events must not block unrelated valid events.

## Flush-group boundaries

A flush group is a logical collection boundary, not an HTTP request or a retry attempt. Each accepted event belongs to exactly one group. The SDK MUST atomically seal the current group on an explicit flush, a periodic flush trigger that actually executes, an enabled size trigger, or close. Later events belong to a new group. Sealing an empty group need not retain a record. Concurrent triggers must neither duplicate events nor split ownership of an event.

Sealing fixes membership and the first-occurrence deduplication result before batching. A blocked network does not prevent an explicit flush from sealing its group; sealing alone is not delivery completion. Retries preserve the sealed payloads. Queue bounds apply to open groups, sealed groups, and in-flight work together; creating groups must not bypass capacity limits.

Lifecycle integrations MUST seal the open group when entering background and again before admitting foreground calls on resume. Calls collected while backgrounded belong to a separate group, unless an explicit flush or another actually executed trigger seals it sooner. If no trigger executes during the background interval, identical calls in that interval deduplicate together; document this counting consequence. Do not synthesize periodic boundaries for timer ticks missed during suspension. A transition flush uses the group sealed at background entry rather than merging it with subsequent background events.

If events survive process restart, preserve group membership and deduplication state. Seal any recovered open group before accepting new events. Never merge recovered groups with each other or with new-session groups for deduplication. Physical batch packing may combine groups only if logical event multiplicity is preserved. No additional group identifier is required on the wire. See the [mobile supplement](../mobile/README.md#event-collection-and-delivery) for suspension and delivery policy.

## Delivery and failures

| Outcome | Behavior |
| --- | --- |
| HTTP 2xx | Batch accepted. |
| HTTP 400, 408, 429; server errors; transient network failure | Eligible for bounded retry. |
| Other HTTP 4xx | Stop subsequent event delivery for this client; preserve evaluation and synchronization. |
| Shutdown deadline reached | Stop outstanding delivery within the cleanup budget. |

Use finite attempts, request deadlines, and documented retry delays. Retries preserve the original payload. A terminal delivery failure must prevent subsequent batches from starting, including batches from the same flush group.

On terminal event-delivery failure, atomically stop accepting events and retry scheduling, and finalize all retained events not already acknowledged as failed/dropped. This includes open groups, sealed groups, and outstanding requests; cancel outstanding requests where supported. Make the loss observable. Late responses must not reverse finalized outcomes or restart delivery; a canceled request may still have reached the server. Existing waiting flushes settle promptly with the failure outcome rather than waiting for their timeout, unless already settled. Subsequent flushes report the terminal failure without networking or claiming delivery. Evaluation and Flag synchronization continue independently.

If events are persisted, remove or durably invalidate these finalized records so a new client cannot replay them. Pending writes must not recreate replayable records after invalidation. Storage failure must be observable; if invalidation cannot be committed, report that durable cleanup is incomplete and document that a later process may recover old records. Do not claim durable erasure or exactly-once delivery in that case. A new client may accept new events but does not intentionally retry records already finalized by this policy.

Delivery is best effort. Overflow, exhausted retries, application suspension, and process exit can lose events; lost responses can cause duplicate delivery. Flush-group deduplication does not guarantee exactly-once delivery.

## Flush and shutdown

Provide a way to request prompt processing and a bounded way to wait for completion. These may be forms of the same operation. A waiting flush covers events accepted before the call, including earlier in-flight work. Later events need not delay it.

Completion means those events were delivered or reached a final failure/drop outcome. Report timeout separately; do not describe processing completion as guaranteed delivery. Document whether delivery continues after a waiting timeout.

Close stops new collection and attempts a bounded final flush before releasing resources. A full buffer, failed request, or suspended runtime must not create an unbounded close. See [Public API](public-api.md#lifecycle).
