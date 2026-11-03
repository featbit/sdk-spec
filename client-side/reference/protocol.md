# Client Protocol Reference

[Specification index](../README.md)

These wire details complement the behavioral modules. They describe service interoperability, not internal SDK architecture. Examples use a fictitious user and flag.

## Endpoints and authentication

Resolve these paths beneath their configured base URLs, preserving deployment prefixes:

| Purpose | Endpoint | Authentication |
| --- | --- | --- |
| Streaming | `streaming?type=client&token=<connection-token>` | Connection token generated from the client SDK key. |
| Polling | `api/public/sdk/client/latest-all?timestamp=<cursor>` | HTTP `Authorization` containing the raw client SDK key. |
| Events | `api/public/insight/track` | HTTP `Authorization` containing the raw client SDK key. |

URI-encode query values. HTTP JSON requests use `Content-Type: application/json`. Do not add a `Bearer` prefix or use a server-side secret. Identify the SDK and version through `User-Agent` where allowed and `X-User-Agent` for the client protocol when the runtime restricts that header.

The connection token uses the same encoding as the server SDK, with the **client SDK key** as input:

1. Remove trailing `=` characters from the key, producing `key`.
2. Let `t` be the current Unix millisecond timestamp as decimal text.
3. Choose `p = max(floor(random_0_to_1 * length(key)), 2)`.
4. Produce `encode(p, 3) + encode(length(t), 2) + key[0:p] + encode(t, length(t)) + key[p:]`.

`encode(n, width)` left-pads with decimal zeroes, keeps the rightmost `width` digits, and substitutes digits using this map:

| Digit | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Encoded character | Q | B | W | S | P | H | D | X | Z | U |

Generate a fresh token per connection attempt. Valid keys must support the insertion index. This reversible encoding does not make token-bearing URLs safe to log.

## User payload

Synchronization and analytics use this user shape. Custom attributes are named entries in an array, not a dictionary on the wire:

```json
{
  "keyId": "user-123",
  "name": "Example User",
  "customizedProperties": [
    {"name": "country", "value": "us"},
    {"name": "age", "value": "30"},
    {"name": "isPremium", "value": "true"},
    {"name": "region", "value": null},
    {"name": "city"}
  ]
}
```

**Non-null values in `customizedProperties[].value` MUST be JSON strings. JSON `null` and omitted values are accepted.** This requirement applies to both synchronization and event payloads. Numeric and boolean attributes must be represented as strings, such as `"30"` and `"true"`, rather than the JSON number `30` or boolean `true`. JSON numbers, booleans, arrays, and objects are not valid attribute values. SDKs MUST ensure this requirement is met before sending the payload, including when applications supply user objects directly.

During custom-attribute rule evaluation, the Evaluation Server MUST distinguish an absent attribute from a present attribute with a null, omitted, or empty value. **If a rule references an attribute that is absent from the user payload, the condition MUST immediately evaluate to non-match.** Do not substitute an empty string and then apply the comparison operator. This applies to negative operators such as `NotEqual` as well, so a rule requiring that condition does not match.

| User attribute input | Rule-matching behavior |
| --- | --- |
| `{"name": "country", "value": null}` | Attribute is present; compare using an empty string. |
| `{"name": "country"}` | Attribute is present; compare using an empty string. |
| No `country` attribute | Any condition referencing `country` is a non-match; no value comparison is performed. |
| `{"name": "country", "value": ""}` | Attribute is present; compare using the supplied empty string. |
| `{"name": "country", "value": "null"}` | Compare using the literal string `"null"`. |

For example, when `country` is absent, both `country Equal ""` and `country NotEqual "US"` are non-matches. When `country` is explicitly present with an empty string, both comparisons match. A present attribute with a null or omitted value follows the same comparison behavior as an explicit empty string; omitting the whole attribute is a different case. Converting null to the literal string `"null"` also changes its meaning.

## WebSocket messages

Send a data-sync request after connection and when identifying a new context:

```json
{
  "messageType": "data-sync",
  "data": {
    "timestamp": 0,
    "user": {"keyId": "user-123", "name": "Example User", "customizedProperties": []}
  }
}
```

Zero requests a full refresh. Otherwise use the committed remote cursor for the exact context represented by the accompanying user. Preserve each logical JSON message's WebSocket boundary.

The response contains user-specific evaluated results:

```json
{
  "messageType": "data-sync",
  "data": {
    "eventType": "full",
    "userKeyId": "user-123",
    "featureFlags": [
      {
        "id": "checkout-v2",
        "variation": "true",
        "variationType": "boolean",
        "timestamp": 1700000000000,
        "matchReason": "target match",
        "insightsEnabled": true,
        "variationOptions": [{"id": "variation-on", "value": "true"}]
      }
    ]
  }
}
```

`eventType` is `full` or `patch`. `userKeyId` identifies the response user but does not resolve same-key or superseded-context ordering by itself. There is no client-local targeting or segment evaluation step.

The application keepalive message is:

```json
{"messageType":"ping","data":null}
```

This is separate from WebSocket protocol ping frames. Any additional liveness policy must be compatible with the service and documented; receiving keepalive traffic alone cannot establish data readiness.

## Flag fields

| Field | Meaning |
| --- | --- |
| `id` | Public flag key, rather than a separate internal flag identifier. |
| `variation` | Selected string value, including boolean, numeric, and JSON values encoded as strings. |
| `variationType` | Declared type: `string`, `boolean`, `number`, or `json`. |
| `timestamp` | Remote flag version in integer Unix milliseconds. Preserve precision; do not replace with receive time. |
| `matchReason` | Server explanation. The exact value `flag archived` marks an archived result. |
| `variationOptions` | Server variation metadata with `id` and string `value`; used to identify the selected variation for analytics. |
| `insightsEnabled` | Optional boolean controlling evaluation event collection for this flag; defaults to `true` when absent. An explicit `false` suppresses collection. |

Preserve variation identifiers without inventing or renumbering them. Match the selected string against `variationOptions[].value` for analytics. Missing or ambiguous metadata must not fabricate an assignment; omit the event while retaining a usable evaluation result.

Unknown fields must not break compatible messages. Apply [invalid-data rules](../spec/synchronization.md#invalid-and-unrelated-data) to malformed required data. Missing optional analytics metadata alone need not make an otherwise usable flag unreadable. Local bootstrap is an SDK input, not a server response; it does not require remote timestamps or analytics metadata.

## Polling

Send `POST <polling-base>/api/public/sdk/client/latest-all?timestamp=<cursor>` with the user object as the JSON body. The body is the user itself, without the WebSocket message wrapper. The initial cursor is zero unless a matching remote cache is being resumed.

HTTP `200` contains the same data-sync response envelope shown above. HTTP `304` has no new data and follows the matching-data restrictions in [Polling mode](../spec/synchronization.md#polling-mode-optional). Do not substitute the server-side GET endpoint or omit user context.

## Event payloads

Send a JSON array to `POST <event-base>/api/public/insight/track`. An evaluation payload is:

```json
[
  {
    "user": {"keyId": "user-123", "name": "Example User", "customizedProperties": []},
    "variations": [
      {
        "featureFlagKey": "checkout-v2",
        "variation": {"id": "variation-on", "value": "true"},
        "timestamp": 1700000001000
      }
    ]
  }
]
```

A metric payload is:

```json
[
  {
    "user": {"keyId": "user-123", "name": "Example User", "customizedProperties": []},
    "metrics": [
      {
        "route": "index/metric",
        "type": "CustomEvent",
        "appType": "Browser-Client-SDK",
        "eventName": "checkout-completed",
        "numericValue": 1.0,
        "timestamp": 1700000001000
      }
    ]
  }
]
```

Event timestamps describe the original application call, not the flag version or send time. Evaluation and metric objects may share a request after [client-side deduplication](../spec/events.md#collection-and-deduplication).

`Browser-Client-SDK` is the browser SDK identifier used in this example. Each SDK must use and document its server-supported `appType` and verify it with the target service. Preserve `route`, `type`, and the other field names.
