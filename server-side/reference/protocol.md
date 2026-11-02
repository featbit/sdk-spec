# Server Protocol Reference

[Specification index](../README.md)

These wire contracts complement the seven behavioral modules. They define server interoperability, not internal SDK architecture.

## Endpoint construction and authentication

The streaming endpoint is resolved relative to the configured base:

```text
streaming?type=server&token=<connection-token>
```

A base ending in `/featbit/` preserves that prefix. SDK documentation MUST explain this behavior or provide explicit, documented normalization. Query parameters MUST be URI-encoded.

SDKs SHOULD identify their language and server-side SDK consistently through `User-Agent`, using `X-FeatBit-User-Agent` as a fallback when the runtime restricts that header.

The current connection token format MUST be preserved unless the server protocol changes:

1. Remove trailing `=` characters from the environment secret, producing `secret`.
2. Obtain current Unix time in milliseconds as a decimal string `t`.
3. Choose an insertion index `p`. Compute `max(floor(random_0_to_1 * length(secret)), 2)`; valid secrets must be long enough for this index.
4. Encode decimal digits using the following map.
5. Produce `encode(p, 3) + encode(length(t), 2) + secret[0:p] + encode(t, length(t)) + secret[p:]`.

| Digit | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Encoded character | Q | B | W | S | P | H | D | X | Z | U |

`encode(n, width)` left-pads the decimal representation with zeroes, keeps the rightmost `width` digits, then substitutes each digit. Generate a fresh token on every connection attempt. Do not substitute JWT, Base64 encoding, or an invented bearer-token scheme. This format is reversible obfuscation, so token-bearing URLs MUST be treated as credentials in diagnostics.

See the [.NET SDK's `ConnectionToken`](https://github.com/featbit/featbit-dotnet-sdk/blob/main/src/FeatBit.ServerSdk/Transport/ConnectionToken.cs) for a reference implementation of this algorithm.

## Message protocol

The SDK sends a JSON text message immediately after each successful connection:

```json
{"messageType":"data-sync","data":{"timestamp":0}}
```

`timestamp = 0` requests a full data set. Otherwise it is the maximum stored entity version, expressed as an integer Unix millisecond timestamp, and requests changes relative to that cursor. The store version includes archived records.

The server response envelope is:

```json
{
  "messageType": "data-sync",
  "data": {
    "eventType": "full",
    "featureFlags": [],
    "segments": []
  }
}
```

`eventType` is `full` or `patch`. The server may send a full replacement even when the client requested changes. Unknown top-level message types MUST be ignored without corrupting state. An unknown sync event type MUST NOT commit data or complete initialization.

The periodic application keepalive message is:

```json
{"messageType":"ping","data":{}}
```

This is separate from WebSocket protocol ping frames. The server responds to this message with a `pong` message:

```json
{"messageType":"pong","data":{}}
```

Detecting or acting on the `pong` response is optional and left to each SDK's discretion. An application-level pong deadline is not required: SDKs are not required to detect or act on the absence of a `pong`. An SDK MAY add a documented liveness deadline if compatible with the server; it MUST NOT require an undocumented response.

Each logical JSON message MUST retain its WebSocket message boundary. Fragmented frames MUST be reassembled before parsing. Implementations MUST serialize concurrent outbound writes so that keepalive and sync JSON cannot interleave or be accidentally concatenated into one JSON message.


## Required evaluation data

| Entity | Required fields and meaning |
| --- | --- |
| Common stored entity | `updatedAt` as an ISO-8601 timestamp, converted to signed 64-bit Unix milliseconds; `isArchived` as a deletion marker. |
| Feature flag | `id`, `key`, `variationType`, `variations`, `targetUsers`, `rules`, `isEnabled`, `disabledVariationId`, `fallthrough`, `exptIncludeAllTargets`. |
| Variation | `id`, `value`; the value is a string on the wire, including boolean, numeric, and JSON values. |
| Individual target | `keyIds` list and `variationId`. |
| Target rule | `name`, optional `dispatchKey`, `includedInExpt`, `conditions`, and rollout `variations`. |
| Fallthrough | Optional `dispatchKey`, `includedInExpt`, and rollout `variations`. |
| Rollout variation | `id`, two-element `rollout` interval, and `exptRollout`. |
| Condition | `property`, `op`, and string `value`. Lists are JSON arrays encoded inside that string. |
| Segment | `id`, `included`, `excluded`, and `rules`; each segment rule contains `conditions`. |

Feature flags also carry the optional boolean `insightsEnabled`, which defaults to `true` when absent and controls [evaluation event collection](../spec/events.md#what-is-recorded).

An entity's `updatedAt` is its synchronization version; it is not an evaluation-event timestamp. Preserve millisecond precision and timezone offsets during conversion. Do not replace versions with local receive times.

Unknown JSON fields MUST be tolerated to allow compatible server additions. Preserve targeting and rollout array order. Null or malformed required structures MUST be handled according to [error isolation](../spec/synchronization.md#invalid-data), not silently interpreted as empty targeting rules.


## Wire payloads

Send a JSON array of payload event objects via:

```text
POST <event-base>/api/public/insight/track
Authorization: <environment-secret>
Content-Type: application/json; charset=utf-8
User-Agent: <language-specific-server-sdk-identifier>
```

The authorization value is the raw environment secret, without an added `Bearer` prefix. Endpoint path resolution follows [endpoint construction and authentication](#endpoint-construction-and-authentication).

An evaluation event has this structure:

```json
[
  {
    "user": {
      "keyId": "user-123",
      "name": "Example User",
      "customizedProperties": [{"name": "country", "value": "us"}]
    },
    "variations": [
      {
        "featureFlagKey": "checkout-v2",
        "variation": {"id": "variation-on", "value": "true"},
        "timestamp": 1700000000000,
        "sendToExperiment": true
      }
    ]
  }
]
```

A custom metric event has this structure. `<server-supported-sdk-app-type>` is a placeholder that MUST be replaced with the target service's identifier for the SDK; it is not a literal wire value:

```json
[
  {
    "user": {"keyId": "user-123", "name": "", "customizedProperties": []},
    "metrics": [
      {
        "appType": "<server-supported-sdk-app-type>",
        "route": "index/metric",
        "type": "CustomEvent",
        "eventName": "checkout-completed",
        "numericValue": 1.0,
        "timestamp": 1700000000000
      }
    ]
  }
]
```

SDKs MUST select the corresponding server-supported SDK `appType` and document it; preserve `route`, `type`, and the other field names. An integration test MUST verify the chosen identifier with the target service.

User custom attributes are an array of `{name, value}` objects, not a dictionary. Variation values remain strings. Evaluation and metric objects may be mixed in one request. Send individual payload objects without deduplication or aggregation; SDKs MUST NOT silently sample, merge, or deduplicate recorded events.

