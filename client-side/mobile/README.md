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
| Enter background | After any configured grace period, stop foreground synchronization, heartbeat, reconnect attempts, and periodic event delivery. Preserve current-context local data. |
| Return to foreground | Reconcile visibility, network, offline, closure, and active context; resume one synchronization source promptly when eligible. Resume bounded event delivery without replaying missed timer ticks. |
| Network unavailable | Suspend synchronization attempts and event delivery when notified; keep usable data. |
| Network available | Resume only if the current lifecycle policy allows it. Repeated signals must not create duplicate sources. |
| Process termination | No shutdown callback or final delivery is assumed. Only previously persisted state can be recovered. |
| Explicit close | Follow the core bounded close contract; lifecycle or network callbacks must never restart the instance. |

Lifecycle suspension applies to both WebSocket and optional Polling. A background polling option affects Flag synchronization only; it does not implicitly enable background analytics.

On background entry, an online SDK MAY attempt one best-effort flush within a finite transition budget. Existing event requests may finish within that budget; no new retries or requests may start after it expires. If the platform suspends execution sooner, do not claim completion. Do not require a background service, special background entitlement, wake lock, or battery exemption for normal Flag operation.

Every replacement synchronizer, including one created by Identify or transport fallback, MUST inherit the latest lifecycle and network state. Superseded connections, delayed reconnects, timers, and already-running response handlers must not regain authority to commit data. Serialize logical message commits in transport order and preserve coherent batch visibility.

On resume, reuse only a cursor paired with the exact active context's committed remote data; otherwise request a full refresh. Never derive the cursor from the phone clock. Do not treat an old socket, heartbeat, or a lifecycle callback as evidence of successful synchronization.

## Startup, readiness, and Identify

Creation and startup MUST NOT wait for network access or perform blocking disk I/O on the UI thread. Provide asynchronous, bounded waiting for initialization. Before cache loading finishes, local reads may return the applicable bootstrap or fallback. After loading, matching cached values may be read without waiting for the network. Normal variation and bulk reads use memory only.

Expose data availability separately from synchronization freshness and lifecycle pause. Pausing does not erase the fact that the current context synchronized earlier; it does mean updates are not currently being received. A never-synchronized context cannot become online-ready merely because the app is backgrounded.

Identify follows the [core identity contract](../spec/identity.md). In particular:

- The most recently accepted Identify defines the active context. An implementation may coalesce work but MUST NOT leave a superseded context active while waiting for its network operation to finish.
- Clear access to the previous context's values immediately at the accepted transition, including bulk reads. Only matching cache, applicable bootstrap, or fallback may serve the new context.
- Attribute changes under the same key and A → B → A sequences require the same isolation as different keys.
- Settle superseded waits with a supersession outcome. Do not report them as successful synchronization of the latest user.
- When online but lifecycle-paused or disconnected, establish the local context and defer network work. An Identify wait may time out; it cannot succeed without the applicable remote commit. Configured offline mode retains its distinct core completion behavior.
- A timeout ends only the caller's wait. It does not close the client or cancel recovery for the latest context. Document language-level cancellation separately; cancelling a caller's wait MUST NOT implicitly close a shared client.

If an SDK exposes an explicit start operation, repeated or concurrent calls MUST share initialization rather than create additional polling loops or connections. A closed client cannot restart.

## Persistence and anonymous identity

Persistent Flag storage follows [Data Storage](../spec/storage.md#persistent-cache-optional). Cache identity MUST include deployment/environment and the full evaluation context, not just the user key. Differences in attribute ordering alone must not prevent matching equivalent contexts.

When persistence is implemented:

- Store schema version, data, origin/analytics metadata, and remote cursor consistently. An interrupted write must yield either a usable coherent snapshot or a cache miss, never an advanced cursor with missing data.
- Bound retained contexts and bytes; document retention, expiration, eviction, and whether system backup may include the cache.
- Treat corruption, unsupported format, inaccessible protected storage, and capacity failure as cache unavailability. Continue in memory without blocking evaluation.
- Provide asynchronous clearing of SDK-owned persisted user data with a completion outcome. Define its scope and prevent in-flight old writes from undoing the clear. Document whether subsequent active-context updates may create new entries.
- Closing is not logout or erasure. Logout requires an explicit Identify; clearing persisted data alone does not change the active user. Document the sequence for logout with erasure.

If automatic anonymous keys are offered, generate an application-scoped random identifier and persist it according to the documented policy. Do not use hardware or advertising identifiers as the default anonymous key. Specify reset, uninstall/reinstall, backup restore, and explicit-clear behavior without promising identity continuity the platform cannot guarantee. Logging in does not implicitly link or merge anonymous analytics.

## Event collection and delivery

Apply the [core event contract](../spec/events.md), including its current per-flush-group deduplication for evaluation and custom metric events. A longer flush interval can combine more identical Track calls; SDK documentation MUST explain that Track is not guaranteed to deliver one event per invocation. Mobile SDKs must not silently change this contract to match another vendor's analytics model.

Lifecycle pause alone is not configured offline mode. While execution is available, valid Track and eligible evaluation calls MUST continue to follow core collection rules using bounded buffers; the SDK MUST NOT discard them solely because visibility changed. Delivery follows the lifecycle policy. Overflow remains observable, and events disabled or configured offline suppress collection and delivery as required by the core.

Capture the context, attributes, selected result, and original timestamp before deferring event processing. Successful conversion must precede evaluation-event recording. Bootstrap and fallback values must not generate evaluation events; matching cached remote results remain subject to first-synchronization eligibility. The event path must not start an unbounded coroutine/task or HTTP request for every evaluation.

Explicit flush while lifecycle-paused MUST NOT implicitly resume ordinary network activity. Its waiting form must resolve within its budget and report a deferred/timeout outcome rather than delivery success when retained work cannot run. Close may attempt its bounded final flush when execution and networking are available, but must respect explicit offline/events-disabled settings and must not claim delivery when suspended.

If durable event buffering is offered, preserve original payloads and contexts across restart. Bound disk use and age, isolate environments, document encryption/backup choices and replay scheduling, and include queued events in data-erasure controls. Disabling events or configured offline mode suppresses replay. Persisted, previously valid events do not require re-evaluating flags on replay. Retries and lost responses may still cause duplicates; durability is not exactly-once delivery.

## Concurrency, callbacks, and ownership

Public operations MUST support documented concurrent use without combining one context's Flag with another context's analytics. Protect accepted identity transitions and data commits, not merely individual map writes. Locks, serial queues, actors, or generation-scoped writers are implementation choices.

Listener callbacks SHOULD default to the platform UI thread. SDKs MUST document callback/Flow/stream execution context, cancellation, ordering, buffering, and coalescing. Never invoke application callbacks while holding internal state locks. A slow or failing subscriber must not block synchronization, grow an unbounded queue, or prevent other subscribers from operating.

Release SDK-owned lifecycle/network observers, timers, connections, and workers on close. Separately installed adapters MUST provide idempotent detach and document ownership; late adapter callbacks to a closed client have no effect. Do not retain screens, view controllers, or activities for the lifetime of the client. SDK teardown must not shut down caller-owned executors or networking resources without explicit ownership transfer.

## Platform documentation and verification

Each SDK MUST publish its supported OS/toolchain range, dependency and package requirements, lifecycle installation, callback context, privacy/data-clearing behavior, and optional capability matrix. Unsupported process sharing, app extensions, or background modes must be explicit. Protect client keys and user data in diagnostics under the core rules; never log connection-token URLs.

Use the [mobile acceptance checklist](conformance.md) in addition to core conformance. No implementation architecture or existing vendor SDK is proof that these scenarios pass.
