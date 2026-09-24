# Mobile Acceptance Checklist

[Mobile supplement](README.md) | [Core checklist](../spec/conformance.md) | [Android](android.md) | [iOS](ios.md)

Run these scenarios in addition to core conformance. They are a draft acceptance target, not an executable suite or evidence that any existing SDK passes. Optional features require their rows only when implemented. Use deterministic clocks, storage/transport fakes, and controlled response ordering for logic checks; use platform integration tests for lifecycle behavior.

## Shared scenarios

| ID | Scenario | Expected outcome |
| --- | --- | --- |
| M01 | Cold start offline or while the service is unreachable | No UI-thread network/disk wait; applicable local values become readable asynchronously, otherwise fallback. Configured offline emits no network or analytics. |
| M02 | Cache is available before initial online synchronization | Values are readable; online initialization is pending; cached evaluation events remain suppressed until context confirmation. |
| M03 | Startup wait expires, then service recovers | Timeout settles only that wait; later valid commit establishes readiness without constructing another client. |
| M04 | Repeated/concurrent start calls | One authoritative synchronization source; all waits settle according to their budgets and the shared outcome. |
| M05 | Create in background; later foreground | Known background state prevents initial synchronization; foreground resumes only when network/offline/close state permits. |
| M06 | Background during Streaming and, separately, Polling | After the configured grace, no foreground requests, heartbeat, or reconnect work begins; local values remain readable and freshness/pause are observable. |
| M07 | Background/foreground/network signals oscillate | Pending pauses/retries cannot create duplicate sources or stop a newer source incorrectly. No catch-up timer bursts. |
| M08 | Identify while already backgrounded | New local context takes effect; replacement source stays paused. Online completion waits for a valid remote commit or reports timeout; foreground synchronizes the latest context. |
| M09 | A → B; B has no matching cache | Individual and bulk reads never expose A's results as B's, before or after B's full response. |
| M10 | Same key with new attributes; then A → B → A with delayed responses | Old data/cursors are not reused as matching state; superseded responses cannot write, notify, or complete the newest wait. |
| M11 | Concurrent evaluation, Identify, Track, and batch update | Each operation observes a coherent context/value combination. Events retain the original user, attributes, value, and call time. |
| M12 | Full {A, B}, then full {A}, then empty full | B disappears, then remote A disappears; applicable bootstrap follows core precedence. No half-committed batch is visible. |
| M13 | Equal/older patches, archive/restore, phone clock jumps | Equal updates follow synchronization order; older updates are ignored; archived flags are hidden; only committed remote versions advance cursor. |
| M14 | Malformed Flag beside valid siblings; invalid envelope | Valid siblings follow core skip rules; an invalid envelope does not change data, cursor, or readiness. |
| M15 | Disconnect with 1000; reject with 4003; close during retry | Normal disconnection recovers when lifecycle-eligible; rejection and close stop recovery; delayed work cannot revive the source. |
| M16 | High-rate evaluations and Track during a slow network | Bounded memory, tasks, delivery concurrency, and payloads; overflow is observable. No HTTP request or unbounded task per read. |
| M17 | Type conversion failure, bootstrap read, and cached read before confirmation | No evaluation event. Successful eligible conversion records using captured remote metadata. |
| M18 | Duplicate Track calls in one flush group and in separate groups | Core deduplication is preserved, including first timestamp; changing mobile timers does not silently replace the contract. |
| M19 | Background during event delivery and explicit flush while paused | Transition work obeys its finite budget; periodic delivery stops. A waiting flush settles without implicitly resuming networking or claiming unsent events were delivered. |
| M20 | Slow/throwing listener; listener evaluates, unsubscribes, or closes | Documented callback context/order; errors contained; no internal-lock deadlock or unbounded backlog. |
| M21 | Close during startup, Identify, cache loading, or event delivery | Bounded cleanup and settled waits; no late commits, new callbacks, or reconnects; retained values readable without analytics. Repeated close is safe. |
| M22 | Inspect logs and disabled analytics | No keys/token URLs or default raw user payloads; disabling analytics does not remove attributes required for server evaluation. |

## Optional persistence and identity scenarios

| ID | Scenario | Expected outcome |
| --- | --- | --- |
| P01 | Relaunch with matching cache, changed attributes, and another environment | Only the full matching context/deployment cache is eligible. |
| P02 | Terminate during cache write; corrupt or incompatible cache; disk unavailable/full | Coherent snapshot or cache miss; never mismatched data/cursor; memory evaluation continues. |
| P03 | Cache count/byte limits and expiry | Documented bounds and eviction; an outage alone does not expire active in-memory data. |
| P04 | Clear persisted data during an in-flight write | Completion reflects the documented scope; old pending writes cannot recreate erased data. Subsequent writes follow the documented policy. |
| P05 | Anonymous start/relaunch, login/logout, explicit reset, backup restore | Identity lifetime follows documentation; no inferred account linking or hardware-derived default key. |
| P06 | Durable event restart, replay failure, expiry, disable events, erasure | Original payloads retained; disk/retry bounds enforced; no replay while offline/disabled; erasure includes queued user data; no exactly-once claim. |

## Platform integration scenarios

| Platform | Required checks |
| --- | --- |
| Android | Activity navigation/rotation and multi-window; initial network absence; Wi-Fi/cellular overlap and handover; Doze/App Standby and process recreation; adapter detach/close; release build with shrinking. Optional background polling must preserve identity and switch back without overlapping sources. |
| iOS | Background and foreground notifications; multiple scenes; suspension while requests are active; force termination and relaunch without cleanup callbacks; finite background task expiration if used; protected storage unavailable; package privacy resources in a consuming app. |

Record SDK/service versions, OS/device coverage, optional settings, test commands, and outcomes. Distinguish mocked logic checks, simulator/emulator checks, and real-device checks. Shared protocol fixtures do not establish platform lifecycle correctness, and passing lifecycle tests does not establish core conformance.
