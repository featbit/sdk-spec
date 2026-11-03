# Acceptance Checklist

[Specification index](../README.md)

Verify observable behavior using each SDK's native tests and integration checks against the supported FeatBit service. This draft checklist is not a claim that the shared fixture suite exists or that a released conformance target is available.

| Module | Representative checks |
| --- | --- |
| Configuration | Required credentials and valid URLs; documented defaults; invalid input produces a non-throwing error; offline without credentials; later option mutation does not affect the client. |
| Synchronization | Empty full response initializes; handshake alone does not; startup timeout allows recovery; outages and normal closure reconnect; 4003 stops recovery; close finishes waits and prevents reconnect. |
| Invalid synchronization data | Invalid envelopes preserve state; bad entities do not block valid siblings; full/patch skip behavior and cursor are correct; logs identify ignored/skipped data safely; later messages still work. |
| Storage | Full replacement and clearing; newer/equal/older patches; compact archived flags and segments; newer restoration; coherent flag/segment reads during updates; caller mutation isolation. |
| Evaluation | Selection precedence; missing versus empty attributes; each operator; segment exclusion precedence and broken references; shared rollout vectors and boundaries; typed conversion and exact fallback; per-flag bulk error isolation. |
| Events | Record only successful typed results after conversion; Track before readiness; no bulk/fallback/offline/closed analytics; original user/timestamp preserved; correct selected variation metadata and wire payloads. |
| Delivery | Batching and bounded buffering; overflow; recoverable versus terminal responses; finite retries; invalid event isolation; flush includes earlier in-flight work; timeout differs from completion and delivery success. |
| Public API and principles | Non-throwing recoverable errors across public operations; isolated clients; concurrent use; repeated and bounded close; evaluation from retained data after close; diagnostics redact sensitive information. |
| Notifications, if implemented | Committed data and readiness visible before callbacks; effective patches and segment updates; ordered notifications; subscriber error isolation; safe callback-triggered close. |

Compare semantic JSON rather than property order. Evaluation fixtures should specify flags, segments, user, requested type, fallback, expected value/variation/reason, and analytics eligibility. Rollout cases must include UTF-8, signed hash interpretation, interval boundaries, and missing dispatch attributes.

SDK documentation must state runtime support, numeric ranges, regex differences, configuration defaults, optional capabilities, and best-effort event-delivery semantics. Omitting optional notifications does not block core conformance.
