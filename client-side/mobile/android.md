# Android Requirements

[Mobile supplement](README.md) | [iOS](ios.md) | [Acceptance checklist](conformance.md)

These requirements apply together with the shared mobile profile and core specification. Android API names below identify integration points, not mandatory dependencies.

## Application and connectivity integration

Use application/process visibility rather than treating each Activity as a separate client session. Activity navigation, rotation, and recreation MUST NOT create duplicate clients or connections. Multi-window and brief Activity transitions must be covered by the documented visibility policy.

Lifecycle integration MAY use process lifecycle facilities, application callbacks, or explicit host hooks. If supplied separately, document installation and detachment, required thread, and the behavior before attachment. Apply known initial visibility before allowing synchronization; manual integration must define its initial state explicitly.

Connectivity handling MUST account for multiple networks and transitions between Wi-Fi, cellular, and VPN. Losing one network does not prove all connectivity is lost. Initialize connectivity state, reconcile it on changes, and let actual request failures drive recovery even when Android reports network availability. Missing optional connectivity access must degrade to bounded request-based recovery rather than crash the app.

The core SDK MUST NOT retain an Activity, View, or lifecycle owner beyond its intended lifetime. Use application-scoped resources for a session-wide client.

## Background behavior

The default is the shared foreground-only synchronization policy. An SDK MAY expose explicit background polling, disabled by default, if Polling is implemented. It MUST:

- Validate Polling support, polling endpoint, and applicable intervals/request/retry limits at online creation, even when the foreground mode is WebSocket and automatic fallback is disabled. Missing or invalid settings produce the core non-throwing configuration error before any background activity. Offline creation retains the core endpoint exemption. Document the background interval, minimum, request/retry limits, scheduler behavior, and whether settings are shared with foreground polling.
- Stop the foreground stream before background polling can commit updates; preserve the active context and its cursor.
- Respect Doze, App Standby, background restrictions, and process death without promising an exact interval or persistent execution.
- Avoid accumulating missed polls. Returning to foreground restores the transport selected by the configured foreground mode and its existing fallback policy promptly, with only one authoritative data source. Background polling is a temporary lifecycle transport, not a fallback reset. If WebSocket already fell back to Polling, foregrounding alone must not retry WebSocket unless the documented fallback policy permits it. Directly configured Polling resumes its foreground polling schedule. Stop the replaced source and reject its late commits during either transition.
- Leave analytics under the shared event lifecycle policy; background polling is not permission to start background event delivery.

When this option is enabled, background creation or Identify may start eligible polling for the latest context and complete after its valid remote response commits. Do not require a foreground transition for success. If execution or network access is unavailable, defer work and apply the shared wait deadlines. Default-policy pause cases in the acceptance checklist do not apply to this explicitly enabled mode; verify both policies separately. Terminal failures retain precedence over either mode.

Do not require a foreground service, persistent notification, wake lock, exact alarm, or battery-optimization exemption for standard SDK use. An optional system scheduler integration must disclose its permissions and scheduling limits; it is not a real-time guarantee.

## API, storage, and distribution

Provide language-appropriate asynchronous waiting and synchronous in-memory reads. Kotlin coroutines/Flow and Java-compatible adapters are choices; document caller cancellation and thread behavior. Listener delivery SHOULD default to the main thread and follow the shared callback contract.

Use application-private storage when persistence is offered. Document Android backup/restore behavior, anonymous-key reset, cache migration, and data clearing. Multi-process cache or client sharing is optional; do not imply that a thread-safe store provides cross-process consistency.

Publish minimum OS version, Java/Kotlin/toolchain compatibility, required manifest entries, transitive dependencies, and artifact coordinates. Verify release builds with shrinking/obfuscation as well as debug builds, supplying consumer rules only where needed. The normal integration must not require dangerous runtime permissions.

Validate lifecycle, network handover, background restrictions, and process recreation on supported Android devices or emulators; report which checks were performed on real devices.
