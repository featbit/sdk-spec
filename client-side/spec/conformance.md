# Acceptance Checklist

[Specification index](../README.md)

Verify observable behavior using each SDK's native tests and integration checks against the supported FeatBit service. This checklist is a draft target, not a claim that a shared executable fixture suite exists or that a particular SDK already conforms.

| Area | Representative checks |
| --- | --- |
| Configuration | Online requires a client key, user, and applicable endpoints; invalid input starts no background work; defaults are documented; options cannot mutate after creation. |
| Offline | No key or endpoints required; valid local values or an empty set establish readiness; no network requests or analytics. |
| Identity | Invalid user preserves current state; Alice → Bob cannot expose Alice's values; same-key attribute changes refresh; late Alice → Bob → Alice responses cannot complete the newest Identify; timeout and close settle waits. |
| User payload | Non-null custom attribute values are strings; null and omitted values are accepted in synchronization and event payloads; raw numbers, booleans, arrays, and objects are rejected. Verify against the service that absent attributes cause non-match before comparison, including `Equal ""` and `NotEqual "US"`. Present attributes with null or omitted values are compared as empty strings; explicit empty strings are compared normally, and the literal string `"null"` remains distinct. |
| Initialization | Connection alone does not initialize; accepted empty full/patch responses do; cache/bootstrap can serve values before Ready; startup timeout permits later recovery. |
| WebSocket | Correct client authentication and user payload; cursor reset/reuse; application keepalive; normal and abnormal disconnection recover; rejection and close stop recovery. |
| Polling, if implemented | Client POST endpoint and user body; serialized requests and finite timeout; cursor follows committed data; `304` only confirms matching server data; failures recover or terminate as documented. |
| Polling fallback, if implemented | Disabled by default; explicit initial-failure trigger; old connection stops; only the current mode may commit; identity is preserved. |
| Invalid data | Invalid envelopes and superseded responses preserve state and waits; bad entries do not block valid siblings; full/patch skip behavior and cursor are correct; diagnostics remain safe. |
| Storage | Full replacement and clearing; newer/equal/older patches; archive and restore; coherent batch reads; bootstrap supersedes cache and requests a full refresh; local timestamps do not advance the remote cursor; caller mutation isolation. |
| Cache, if implemented | Environment and full-context isolation; miss on changed attributes; corrupt cache ignored; data and cursor stay paired; storage failure falls back to memory; documented clearing/retention. Reverse asynchronous write completion cannot overwrite a newer durable commit, including equal-version value changes and full replacements with a lower cursor; verify recovery after restart. |
| Evaluation | Local reads before/after readiness and during outages; exact fallback and reason; case-sensitive keys; type conversion boundaries; archive exclusion; bulk reads produce no analytics and isolate bad entries. |
| Events | Successful conversion precedes recording; no bootstrap/fallback/bulk analytics; cached results wait for context confirmation; Track before readiness; original user/timestamp preserved across Identify and retries; server experiment metadata preserved. |
| Deduplication and delivery | Identical evaluation and metric events deduplicate within a flush group; distinct payloads and separate groups do not; first timestamp retained; bounded buffers/batches/retries; terminal failure stops new collection and delivery, finalizes retained work, settles flushes with failure, and fences late responses. Durable invalidation prevents replay and stale-write resurrection; cleanup failure is observable. |
| Flush and close | Earlier in-flight work included; timeout differs from completion and delivery success; repeated close and full-buffer close finish; no late commits, reconnects, or new callbacks; retained values remain readable. |
| Notifications and diagnostics | Values/status visible before callbacks; equal-version value change notifies; duplicates do not; Identify exposes changed keys; subscriber errors isolated; callback-triggered close safe; sensitive data redacted. |

Additional flush-group checks MUST cover atomic sealing on explicit/periodic/size/close triggers; later events entering a new group; a flush while delivery is blocked sealing without claiming delivery; retries preserving membership; and bounds across open, sealed, and in-flight work. If lifecycle integration or event persistence is implemented, verify background/resume boundaries, no synthetic missed-timer groups, and no cross-group deduplication when replaying after restart. See [flush-group boundaries](events.md#flush-group-boundaries) and the [mobile checklist](../mobile/conformance.md).

Compare semantic JSON rather than property order. Future fixtures should include the initial context, local data sources, incoming responses, identity changes, operations, and expected values, reasons, notifications, cursors, and event eligibility. They must cover ordering and isolation as well as individual calls.

SDK documentation must declare optional capabilities. Missing optional Polling, bootstrap, persistence, or platform lifecycle integration does not block core conformance; a capability that is implemented must satisfy its applicable rules.
