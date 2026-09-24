# User Configuration

[Specification index](../README.md) | [General principles](general.md)

Configuration selects the FeatBit environment, initial user, and network behavior. Public option names and configuration styles may follow each language's conventions.

## Common settings

| Setting | Expected behavior |
| --- | --- |
| Client SDK key | Required online; identifies access to one environment's client-side results. Never substitute a server-side secret. |
| Initial user | Required in both online and offline modes; follows [User Identity](identity.md). |
| Streaming server URL | Configurable WebSocket base URL using `ws` or `wss`. |
| Synchronization mode | WebSocket by default and required; user-selectable Polling when implemented. |
| Event server URL | Configurable HTTP base URL using `http` or `https`; may differ from synchronization endpoints. |
| Offline mode | Disabled by default. When enabled, perform no network communication and suppress analytics. |
| Disable events | False by default. When enabled, suppress evaluation and metric event collection and delivery. Synchronization continues. |
| Bootstrap values, if supported | Application-supplied local flag values; useful for startup and offline use. |
| Persistent cache, if supported | Reuse results for the matching environment and user context. Document enablement and retention. |
| Startup wait | Bound how long an application waits for initial synchronization. Expiry does not stop recovery. |
| Event flush interval and capacity | Control delivery frequency and bound retained events. |
| Network and retry limits | Bound connection attempts and requests; pace recovery during outages. |
| Shutdown timeout | Bound the final event-processing attempt and cleanup. |
| Logging | Application-controlled diagnostics with documented defaults. |

SDKs MUST expose the client key, initial user, relevant endpoints, synchronization mode, offline mode, event enablement, startup wait, and main event capacity/flush settings. Lower-level limits MAY use documented defaults. Exact durations and retry schedules are not cross-language requirements.

Document defaults, units, supported ranges, and URL path handling. Preserve deployment prefixes such as `/featbit/`. Use secure URL schemes on untrusted networks.

## Optional Polling settings

SDKs implementing [Polling mode](synchronization.md#polling-mode-optional) MUST expose synchronization mode, polling server URL, and polling interval. The names below describe language-neutral configuration concepts; public option names may follow each language's conventions.

| Setting | Default and availability | Behavior and validation |
| --- | --- | --- |
| Synchronization mode | Defaults to `WebSocket`. `Polling` is available when implemented. | Select the mode at client creation. Selecting `Polling` starts polling without a WebSocket connection. Reject unsupported modes before starting background activity. |
| Polling server URL | SDK-specific default, if provided; otherwise required online when Polling is selected, automatic Polling fallback is enabled, or optional background polling is enabled. | HTTP base URL using `http` or `https`, pointing to a service that supports the client polling API. Resolve `api/public/sdk/client/latest-all` beneath this base, preserving deployment prefixes. Send the active user using the [client polling protocol](../reference/protocol.md#polling). Validate the URL before startup. |
| Polling interval | SDKs MUST provide and document a default, units, and supported range. | Positive, finite duration controlling normal polling frequency. Document whether the interval is measured from request start or completion. The initial request starts promptly without waiting for this interval. Requests MUST NOT overlap or accumulate to catch up after a delay. |
| Polling request timeout | Positive, finite SDK-specific default; MAY be exposed separately or through shared network settings. | Bound each HTTP request, including response-body retrieval. A timeout retains committed data and follows the polling recovery policy. This is separate from the application's startup wait. |
| Polling retry policy | SDKs MUST document the default policy; controls MAY be grouped with shared network settings. | Specify the delay after transient failures, how delays change across repeated failures, maximum delay, and any jitter. Honor applicable `Retry-After` responses. Recovery continues until success, terminal rejection, or close, regardless of startup wait expiry. After success, resume the normal polling interval. |
| Enable automatic Polling fallback | Optional capability; disabled by default. If offered, expose an enable/disable setting. | Permit a switch to Polling after an initial WebSocket connection failure. Requires WebSocket as the selected mode, Polling support, and valid polling settings. Document the exact trigger and whether or when WebSocket is later retried. Stop the replaced connection; only one mode may commit synchronization results at a time. |

A client configured directly for Polling does not require a streaming URL. Online automatic fallback requires valid settings for both modes. Event endpoints, event flush intervals, and event retry settings control analytics delivery independently; they do not determine polling frequency or recovery.

Validate applicable polling settings before starting background activity, and report invalid input through the common non-throwing configuration error behavior. Configured mode and polling settings remain stable after creation; an enabled automatic fallback may change the active transport according to its documented policy. Exact timeout values, intervals, and retry delays remain SDK-specific rather than shared cross-language constants.

## Validation and startup

Validate configuration and the initial user before starting background activity. Invalid input MUST produce a clear, non-throwing creation or validation error. Do not silently connect to a different environment or use an unsupported mode. Validate an event endpoint only when event delivery is enabled.

Offline startup requires neither a key nor remote endpoints. It establishes the supplied local values, or an empty data set, and becomes ready. An empty offline client returns `FlagNotFound` for unknown flags.

Online startup may expose matching cached or bootstrap values before the first server response. Local availability alone does not complete online initialization. See [readiness](synchronization.md#initialization-and-readiness) and [local source precedence](storage.md#bootstrap-and-local-source-precedence).

Effective configuration MUST remain stable after creation. Changing the active user is an explicit [Identify](identity.md#changing-the-active-user) operation; mutating the original options or user object must not reconfigure a running client.
