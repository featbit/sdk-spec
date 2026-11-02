# Event Processing

[Specification index](../README.md) | [General principles](general.md)

Events support usage analytics and experimentation. Collection must remain inexpensive for the application, while delivery runs independently from evaluation and synchronization.

## What is recorded

| Operation or condition | Event behavior |
| --- | --- |
| Successful typed evaluation | Record one event after conversion succeeds when insights collection is enabled for the flag, using the selected variation ID and original string value. |
| Fallback, including `WrongType` or `ClientNotReady` | Do not record an evaluation event. |
| Bulk evaluation | Do not record evaluation events. |
| Track | Record a named custom event; the numeric value defaults to 1.0. Initialization is not required. |
| Offline, events disabled, or client closed | Suppress event collection and delivery. |

Record the user context and timestamp as they were at the call. Later caller changes and retries must not alter them. Experiment eligibility is independent of the selected value and follows the [shared sampling rules](../reference/evaluation.md#experiment-eligibility).

`insightsEnabled` controls whether this flag's evaluation events are sent and defaults to `true` when absent. When false, do not record evaluation events for this flag. This setting does not change the evaluation result or override SDK-wide event suppression.

## Efficient collection and delivery

Accept events into bounded local storage without waiting for network delivery. When capacity is exhausted, drop the new event and expose the loss through diagnostics or an API result. Evaluation must still return promptly.

Send events in batches, triggered periodically or by an explicit flush. SDKs SHOULD also send when enough events accumulate to fill a batch. Bound batch size, pending work, retained payload size, and concurrent delivery so prolonged outages cannot grow event memory without limit. Document how the capacity setting relates to total buffering.

Batching groups events into requests; it MUST NOT silently sample, merge, or deduplicate them. Delivery order is not guaranteed. An invalid event must not prevent unrelated valid events from being processed.

The [protocol reference](../reference/protocol.md#wire-payloads) defines the HTTP endpoint, authentication, and payloads. Queue types, worker counts, dispatch stages, and scheduling are implementation choices.

## Failures and retries

| Outcome | Behavior |
| --- | --- |
| HTTP 2xx | Batch accepted. |
| HTTP 400, 408, 429; server errors; transient network failures | Eligible for bounded retry. |
| Other HTTP 4xx | Stop subsequent event delivery for this client; preserve synchronization and evaluation. |
| Explicit shutdown cancellation | Stop pending delivery promptly. |

Use documented attempt limits, delays, and request deadlines. A deadline may exhaust the current delivery budget; it must not create indefinite retries. Retries preserve the original payload and timestamp. Once delivery is terminal, no additional requests may start; already in-flight requests may finish.

Delivery is best effort. Overflow, exhausted retries, shutdown deadlines, and process exit can lose events; lost responses can cause duplicate delivery. Do not promise durable or exactly-once delivery.

## Flush and shutdown

`Flush` requests prompt processing and returns without waiting for the server. A waiting form (`FlushAndWait`) must identify the events accepted before the call and wait for their processing to finish, including work already in progress, within the requested timeout. Later events need not delay that call.

Processing completion means those events were sent or reached a final failure/drop outcome; it is not proof that every event was delivered. Report completion separately from timeout and document whether delivery continues after a waiting timeout.

Close stops accepting new events and attempts a bounded final flush. Full buffers or delivery failures must not prevent close from completing. See [Public API](public-api.md#lifecycle).
