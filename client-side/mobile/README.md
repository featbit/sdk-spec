# Mobile Client SDK Supplement

[Client-side specification](../README.md) | [Android](android.md) | [iOS](ios.md) | [Acceptance checklist](conformance.md)

**Status: Draft.** This supplement defines additional observable behavior for FeatBit Android and iOS SDKs. It is a design target, not a claim that an existing implementation conforms.

## Scope and relationship to the core

All eight [core modules](../README.md#core-specification) and the [protocol reference](../reference/protocol.md) continue to apply. Requirement levels follow [General Principles](../spec/general.md#scope-and-requirement-levels). A mobile-profile implementation MUST satisfy the shared requirements here and its platform requirements. Platform documents specialize runtime integration; they do not override identity, evaluation, storage, or event semantics.

Mobile SDKs still receive server-evaluated results. They do not implement targeting rules or experiment allocation. No particular language, concurrency primitive, HTTP library, storage technology, or class structure is required.

The following remain shared contracts, not mobile-specific choices: full/patch processing, remote cursors, equal-version acceptance, archive handling, complete-context isolation, conversion, event eligibility, Track deduplication, and flush outcomes. Changes to these contracts belong in the core specification with cross-SDK acceptance cases.

## Default mobile profile

| Setting | Mobile behavior |
| --- | --- |
| Foreground synchronization | WebSocket by default; explicitly selected Polling is optional under the core rules. |
| Lifecycle integration | Required capability. It MAY be automatic or supplied through a documented adapter/manual hooks; applications must have a complete integration path. |
| Background synchronization | Pause by default on both platforms. Android MAY offer explicit background polling under its platform requirements. |
| Transition grace period | Zero by default after the integration reports background. An optional, bounded, documented grace period MAY reduce connection churn. |
| Event flush interval | SHOULD default to 30 seconds; expose a positive configurable interval. |
| Event retention | Bounded in-memory buffering by default. Publish capacity, batch/concurrency limits, overflow behavior, and retry limits. |
| Persistent Flag cache | SHOULD be supported and enabled by default when provided; allow disabling and clearing it. |
| Persistent event queue | Optional and disabled by default; no process-exit delivery guarantee. |
| Anonymous key generation | Optional; an explicit initial user is otherwise required. |
| Automatic device/application attributes | Optional and disabled by default; document each collected field. |

SDKs MUST document defaults and ranges not fixed here, including caller wait budgets, request deadlines, cache limits, and shutdown timeout. Never copy a transport or event timer into a different subsystem merely because both need periodic work.

## Lifecycle and network state

Treat application visibility, network availability, configured offline mode, context freshness, and permanent closure as separate concerns. Background suspension MUST NOT set the user's offline configuration, clear valid in-memory results, or permanently close the client. Connectivity signals are hints; an available network does not establish service reachability or readiness.

| Transition | Required behavior |
| --- | --- |
| Creation in foreground | Establish applicable local data, then begin the configured online synchronization. |
| Creation in background | Establish applicable local data; defer synchronization under the background policy. Do not briefly connect before applying a known background state. |
| Enter background | Immediately stop periodic event delivery. After any configured grace period, stop foreground synchronization, heartbeat, and reconnect attempts. Preserve current-context local data. |
| Return to foreground | Reconcile visibility, network, offline, closure, subsystem terminal failures, and active context; resume one synchronization source promptly when eligible. Resume bounded event delivery without replaying missed timer ticks. |
| Network unavailable | Suspend synchronization attempts and event delivery when notified; keep usable data. |
| Network available | Resume only if the current lifecycle policy allows it. Repeated signals must not create duplicate sources. |
| Process termination | No shutdown callback or final delivery is assumed. Only previously persisted state can be recovered. |
| Explicit close | Follow the core bounded close contract; lifecycle or network callbacks must never restart the instance. |

The default background-pause policy applies to both WebSocket and optional Polling. Explicitly enabled Android background polling is the exception: creation and Identify in background may synchronize through that polling source when platform execution, network, and client state permit. A valid remote commit may complete initialization or Identify without a foreground transition. It does not implicitly enable background analytics; iOS retains the pause policy.

Permanent closure, explicit offline configuration, and terminal subsystem failure take precedence over lifecycle/network recovery. A synchronization rejection such as WebSocket 4003 remains terminal for that client even after Identify, transport fallback, or foreground/network changes. A terminal event-delivery HTTP failure stops subsequent event delivery independently, without stopping otherwise healthy Flag synchronization. Synchronization failure alone must not disable an otherwise eligible event subsystem. Recoverable failures retain their normal retry policy. These terminal states are not reset by automatic recovery; a new, appropriately configured client is needed to retry the failed subsystem.

The grace period applies only to Flag synchronization, not analytics. On background entry, stop ordinary event sends/retries immediately; an online SDK MAY attempt one best-effort transition flush, covering work accepted before that transition, within a finite budget measured from background entry. Existing event requests may finish within that budget; no new event requests or retries may start after it expires while still backgrounded, except the separately bounded final flush on close. Without transition flushing, cancel outstanding event requests where supported and retain unacknowledged work under the normal bounded policy. A longer grace period cannot extend event delivery, and a longer flush budget cannot extend foreground synchronization. Returning to foreground ends the transition allowance and resumes ordinary eligible delivery without duplicate workers. If the platform suspends execution sooner, do not claim completion. Do not require a background service, special background entitlement, wake lock, or battery exemption for normal Flag operation.

Transition flushing ends at the earlier of its SDK deadline or withdrawal of platform execution permission. Bound each participating request by its own deadline and this transition cutoff. At cutoff, stop transition scheduling, invalidate its outstanding delivery attempts, and cancel their requests where supported; cleanup must not wait for a longer request timeout. Keep unacknowledged events in their existing groups under normal capacity, age, and retry limits for later eligible delivery; this interruption is not a terminal subsystem failure and does not reset retry accounting. Late responses from invalidated attempts, including success or terminal HTTP errors, may only release request resources: they cannot acknowledge/drop retained events, settle waits as delivered, change subsystem state, or interfere with a newer attempt. Already accepted acknowledgements remain valid. Cancellation cannot guarantee the server did not receive an event, so later retry may duplicate delivery. Requests still physically outstanding remain subject to the delivery concurrency limit. If foregrounding ends the transition early, invalidate its cutoff callbacks and either transfer live attempts to ordinary delivery without resetting their request deadlines or cancel/invalidate them before retrying; never start duplicate workers for the same retained work. Apply the same cancellation and late-response rules when transition flushing is disabled. If execution is suspended, perform pending cleanup at the first opportunity under the time-budget rules.

Every replacement synchronizer, including one created by Identify or transport fallback, MUST inherit the latest lifecycle and network state. Superseded connections, delayed reconnects, timers, and already-running response handlers must not regain authority to commit data. Serialize logical message commits in transport order and preserve coherent batch visibility.

On resume, reuse only a cursor paired with the exact active context's committed remote data; otherwise request a full refresh. Never derive the cursor from the phone clock. Do not treat an old socket, heartbeat, or a lifecycle callback as evidence of successful synchronization.

## Startup, readiness, and Identify

Creation and startup MUST NOT wait for network access or perform blocking disk I/O on the UI thread. Provide asynchronous, bounded waiting for initialization. Before cache loading finishes, local reads may return the applicable bootstrap or fallback. After loading, matching cached values may be read without waiting for the network. Normal variation and bulk reads use memory only.

Expose data availability separately from synchronization freshness and lifecycle pause. Pausing does not erase the fact that the current context synchronized earlier; it does mean updates are not currently being received. A never-synchronized context cannot become online-ready merely because the app is backgrounded.

Asynchronous cache reads MUST be authorized at commit time, just like network responses. Bind each load to the accepted context transition and local initialization stage. A result may initialize that stage only if no remote response has committed for it, no applicable bootstrap takes precedence, and the client/load has not been invalidated. Identify (including A → B → A), clear of the relevant cache, and close invalidate outstanding old loads. Revalidate and apply atomically with those operations; ignored loads cannot change values, cursor, readiness, notifications, or write the old snapshot back to disk. A fresh load for the new context may still proceed.

Identify follows the [core identity contract](../spec/identity.md). In particular:

- The most recently accepted Identify defines the active context. An implementation may coalesce work but MUST NOT leave a superseded context active while waiting for its network operation to finish.
- Clear access to the previous context's values immediately at the accepted transition, including bulk reads. Only matching cache, applicable bootstrap, or fallback may serve the new context.
- Attribute changes under the same key and A → B → A sequences require the same isolation as different keys.
- Settle superseded waits with a supersession outcome. Do not report them as successful synchronization of the latest user.
- When online but lifecycle-paused or disconnected, establish the local context and defer network work. An Identify wait may time out; it cannot succeed without the applicable remote commit. Configured offline mode retains its distinct core completion behavior.
- A timeout ends only the caller's wait. It does not close the client or cancel recovery for the latest context. Document language-level cancellation separately; cancelling a caller's wait MUST NOT implicitly close a shared client.

If an SDK exposes an explicit start operation, repeated or concurrent calls MUST share initialization rather than create additional polling loops or connections. A closed client cannot restart.

## Time budgets and suspension

Use monotonic elapsed time that includes device sleep/suspension for startup and Identify waits, waiting flush, close, grace periods, transition-flush budgets, request deadlines, and retry delays. These durations MUST NOT depend on changes to the calendar clock. Capture deadlines when the operation is accepted; do not restart the budget on foreground return or network recovery. Event timestamps and connection-token timestamps retain their protocol-defined wall-clock meaning; remote cursors remain server-derived versions.

An OS-suspended process cannot run a timeout callback. Once execution resumes, settle overdue unresolved waits promptly as timeout before later work can report them successful. A result already settled before its deadline keeps that outcome even if callback delivery is delayed. A wait timeout does not cancel latest-context synchronization recovery. Expired grace/transition budgets take effect before new work begins; expired requests cannot extend their original budget, and any retry is a new eligible attempt. An elapsed retry delay permits at most the next attempt when lifecycle and terminal state allow it, not a burst of missed attempts. Close continues bounded cleanup at the first execution opportunity without reopening networking after its budget expires.

Monotonic deadlines belong to one process/boot lifetime; do not persist and reuse them as restart timestamps. Document a separate conservative age/expiry policy for durable data under wall-clock changes. Test date/time jumps separately from suspension and timeout behavior.

## Persistence and anonymous identity

Persistent Flag storage follows [Data Storage](../spec/storage.md#persistent-cache-optional). Cache identity MUST include deployment/environment and the full evaluation context, not just the user key. Differences in attribute ordering alone must not prevent matching equivalent contexts.

When persistence is implemented:

- Store schema version, data, origin/analytics metadata, and remote cursor consistently. An interrupted write must yield either a usable coherent snapshot or a cache miss, never an advanced cursor with missing data.
- Follow the core cache-write ordering rule: an older asynchronous snapshot cannot overwrite a newer durable commit for the same cache entry. Use logical commit order, including equal-version changes and full replacements with lower cursors; checking only server versions or checking authority before an asynchronous write starts is insufficient.
- Bound retained contexts and bytes; document retention, expiration, eviction, and whether system backup may include the cache.
- Treat corruption, unsupported format, inaccessible protected storage, and capacity failure as cache unavailability. Continue in memory without blocking evaluation.
- Provide asynchronous clearing of SDK-owned persisted user data with a completion outcome. Define its scope and prevent in-flight old writes from undoing the clear. Document whether subsequent active-context updates may create new entries.
- Closing is not logout or erasure. Logout requires an explicit Identify; clearing persisted data alone does not change the active user. Document the sequence for logout with erasure.

If automatic anonymous keys are offered, generate an application-scoped random identifier and persist it according to the documented policy. Do not use hardware or advertising identifiers as the default anonymous key. Specify reset, uninstall/reinstall, backup restore, and explicit-clear behavior without promising identity continuity the platform cannot guarantee. Logging in does not implicitly link or merge anonymous analytics.

## Automatic device/application attributes (optional)

When offered, collect only a documented, explicitly enabled field set. Publish a schema with each wire name, meaning, source, string representation, and platform availability. Use the reserved custom-attribute prefix `featbit.sdk.` for these fields; never replace the user key/name or flatten device objects into an incompatible wire payload. The feature reserves this prefix only when enabled. Reject caller attributes with that prefix through the normal configuration/Identify validation outcome, leaving an existing context unchanged; neither automatic nor caller values silently win. Other caller attributes remain unchanged.

Sample the enabled field set at creation and each accepted Identify before cache lookup or network work. Store an immutable snapshot as part of the effective context used by synchronization, caching, and event capture. Omit unavailable fields instead of inventing empty strings, `null` strings, or defaults. Do not mutate these attributes during evaluation, event sending, or foreground recovery. Applications refresh them by calling Identify again, even with the same caller-supplied user; recompute automatic fields before comparing effective contexts. A changed snapshot requires the usual new-context isolation and synchronization. An unchanged snapshot does not itself create an attribute change.

Any additional explicit refresh API MUST use the same Identify transition and outcomes. Queued events retain their earlier snapshot. Disabling analytics does not remove these attributes from synchronization; disabling automatic collection in a newly created client must perform no such collection and must not match a cache for a different effective context. Do not infer advertising/hardware identity or account linking from this feature.

## Event collection and delivery

Apply the [core event contract](../spec/events.md), including its current per-flush-group deduplication for evaluation and custom metric events. A longer flush interval can combine more identical Track calls; SDK documentation MUST explain that Track is not guaranteed to deliver one event per invocation. Mobile SDKs must not silently change this contract to match another vendor's analytics model.

Follow the [flush-group boundaries](../spec/events.md#flush-group-boundaries) independently of delivery permission. Seal at background entry (before any grace period), explicit flush, executed periodic/size triggers, close, and foreground resumption. With no intervening trigger, all events collected during one background interval share a group and identical Track calls deduplicate together. Resume seals that group before foreground collection; no missed timer ticks create artificial groups. Durable replay preserves old groups, seals any recovered open group, and never deduplicates across a restart boundary or combines old and new groups semantically. Grouping remains subject to the total buffer limit.

Lifecycle pause alone is not configured offline mode. While execution is available, valid Track and eligible evaluation calls MUST continue to follow core collection rules using bounded buffers; the SDK MUST NOT discard them solely because visibility changed. Delivery follows the lifecycle policy. Overflow remains observable, and events disabled or configured offline suppress collection and delivery as required by the core.

Capture the context, attributes, selected result, and original timestamp before deferring event processing. Successful conversion must precede evaluation-event recording. Bootstrap and fallback values must not generate evaluation events; matching cached remote results remain subject to first-synchronization eligibility. The event path must not start an unbounded coroutine/task or HTTP request for every evaluation.

Explicit flush while lifecycle-paused MUST NOT implicitly resume ordinary network activity. It still seals the current group. Its waiting form must honor the [time-budget rules](#time-budgets-and-suspension) and report a deferred/timeout outcome rather than delivery success when retained work cannot run. Close may attempt its bounded final flush when execution and networking are available, but must respect explicit offline/events-disabled and terminal-delivery settings and must not claim delivery when suspended.

If durable event buffering is offered, preserve original payloads and contexts across restart. Bound disk use and age, isolate environments, document encryption/backup choices and replay scheduling, and include queued events in data-erasure controls. Disabling events or configured offline mode suppresses replay. Persisted, previously valid events do not require re-evaluating flags on replay. Retries and lost responses may still cause duplicates; durability is not exactly-once delivery.

Terminal event failure follows the [core failure policy](../spec/events.md#delivery-and-failures): suppress new collection, finalize retained work, settle waiting flushes with failure, and remove or invalidate finalized durable records. This is not a lifecycle pause; foregrounding does not restore collection or replay. Report incomplete durable cleanup if storage prevents it.

## Concurrency, callbacks, and ownership

Public operations MUST support documented concurrent use without combining one context's Flag with another context's analytics. Protect accepted identity transitions and data commits, not merely individual map writes. Locks, serial queues, actors, or generation-scoped writers are implementation choices.

Listener callbacks SHOULD default to the platform UI thread. SDKs MUST document callback/Flow/stream execution context, cancellation, ordering, buffering, and coalescing. Never invoke application callbacks while holding internal state locks. A slow or failing subscriber must not block synchronization, grow an unbounded queue, or prevent other subscribers from operating.

Release SDK-owned lifecycle/network observers, timers, connections, and workers on close. Separately installed adapters MUST provide idempotent detach and document ownership; late adapter callbacks to a closed client have no effect. Do not retain screens, view controllers, or activities for the lifetime of the client. SDK teardown must not shut down caller-owned executors or networking resources without explicit ownership transfer.

## Platform documentation and verification

Each SDK MUST publish its supported OS/toolchain range, dependency and package requirements, lifecycle installation, callback context, privacy/data-clearing behavior, and optional capability matrix. Unsupported process sharing, app extensions, or background modes must be explicit. Protect client keys and user data in diagnostics under the core rules; never log connection-token URLs.

Use the [mobile acceptance checklist](conformance.md) in addition to core conformance. No implementation architecture or existing vendor SDK is proof that these scenarios pass.
