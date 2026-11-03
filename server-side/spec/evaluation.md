# Feature Flag Evaluation

[Specification index](../README.md) | [General principles](general.md)

Evaluation uses local flags, segments, and a supplied user context. It MUST NOT make a network request or wait for event delivery.

## User context

A user has a stable string key, an optional name (empty by default), and custom string attributes. Reserved attributes `keyId` and `name` take precedence over custom attributes.

Attribute lookup distinguishes missing values from present empty strings. Missing attributes and blank property names fail ordinary conditions, including negative comparisons. A present empty string is compared normally.

## Selection order

For a typed evaluation:

1. If the client has not initialized, return the caller's fallback with `ClientNotReady`.
2. Find the active flag. A missing or archived flag returns an `Error` fallback.
3. If the flag is disabled, select its configured disabled variation.
4. Otherwise, use the first individual target containing the user key.
5. Otherwise, use the first rule whose conditions all match, then select its rollout variation.
6. If no target or rule matches, use the flag's default rollout.
7. Convert the selected value to the requested type, then record the successful evaluation event.

The configured disabled variation and default rollout are normal results. They are distinct from the application-supplied fallback used when evaluation fails.

Rules and targets retain their server-defined order. Conditions within a rule use AND; rules are ordered alternatives. A matched rule with an invalid rollout or missing selected variation returns an error rather than trying a later rule.

## Matching and percentage rollout

Ordinary conditions support string comparisons, numeric comparisons, booleans, list membership, and regular expressions. Comparisons must not depend on the machine's locale. Unsupported operators do not match.

For segments, explicit exclusion wins over inclusion; otherwise any matching segment rule includes the user. A referenced segment's malformed or missing data must not silently produce a successful negative segment condition.

Percentage rollout is deterministic: the same flag, dispatch attribute, and configuration must produce the same variation in every SDK. Use the configured dispatch attribute, or the user key when none is configured. Do not substitute a different hash or normalize server percentages.

Exact operator names, boundary behavior, and rollout vectors are defined in the [evaluation compatibility reference](../reference/evaluation.md). These details are necessary to keep user assignments consistent across languages.

## Values and reasons

SDKs support boolean, string, integer, and floating-point results. Separate float/double methods and numeric ranges follow the language's type system and must be documented. JSON values are available through the string API; parsed JSON helpers are optional.

Convert the selected string value rather than rejecting it solely because of the flag's declared type. Strings remain unchanged; boolean conversion accepts case-insensitive true/false rather than truthiness. Integer conversion rejects fractional values and overflow. Numeric conversion must be locale-independent and reject non-finite values.

| Reason | Meaning |
| --- | --- |
| Off | Selected the configured disabled variation. |
| TargetMatch | Matched an individual target. |
| RuleMatch | Matched a targeting rule. |
| Fallthrough | Selected the default rollout. |
| ClientNotReady | Initial usable data is unavailable. |
| WrongType | The selected value cannot be converted to the requested type. |
| Error | Missing flag, invalid input/data, or another contained evaluation failure. |

Detailed results include flag key, value, variation ID, reason category, and explanatory text. Successful results contain the selected variation ID. Fallback results contain the caller's exact fallback and an empty variation ID. Reason text may vary; reason categories are the stable contract.

## Failure behavior

Recoverable evaluation errors MUST return fallback without escaping through the public API. Invalid rule expressions, broken segment references, and malformed selected variations affect only the flag being evaluated. SDKs need not inspect unused evaluation paths.

Evaluation errors MUST NOT roll back storage, change the synchronization cursor, or trigger resynchronization. Event-processing errors MUST NOT change a successfully computed result. Fallback results, including WrongType, produce no evaluation events.

Bulk evaluation returns raw-string details for active flags without events. One bad flag must not prevent healthy results; its detail reports Error with no selected variation and an empty string value. It reads the committed store without the typed API initialization gate; an empty store returns an empty collection.
