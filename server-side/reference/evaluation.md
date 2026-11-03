# Evaluation Compatibility Reference

[Specification index](../README.md) | [Evaluation behavior](../spec/evaluation.md)

These rules define results that must agree across languages. Internal APIs, storage structures, and execution models are unrestricted.

## Rules and condition operators

Conditions within one rule are ANDed and short-circuit on failure. Rules are ordered alternatives. An empty condition list matches vacuously; malformed or null condition lists MUST NOT be normalized into an empty matching rule.

Before applying an operator, look up the condition property and check whether it exists. A missing attribute, or an empty or whitespace-only property name, MUST return false for every ordinary condition, including negative operators such as `NotEqual`, `NotContain`, `NotMatchRegex`, and `NotOneOf`. Do not implement negative operators by negating the result of an entire positive condition, because that would turn a missing attribute into a match. If the attribute exists and its value is `""`, apply the selected operator normally using that empty string.

| Wire operator | Semantics |
| --- | --- |
| `LessThan`, `LessEqualThan` | Numeric `<`, `<=`. |
| `BiggerThan`, `BiggerEqualThan` | Numeric `>`, `>=`. Preserve these exact wire names. |
| `Equal`, `NotEqual` | Case-sensitive string equality/inequality, not numeric equality. |
| `Contains`, `NotContain` | Case-sensitive substring match or its negation. |
| `StartsWith`, `EndsWith` | Case-sensitive prefix/suffix match. |
| `MatchRegex`, `NotMatchRegex` | Regex search match or its negation; not implicitly a full-string match. |
| `IsOneOf`, `NotOneOf` | Exact string membership/non-membership in a JSON-encoded string array. |
| `IsTrue`, `IsFalse` | Case-insensitive comparison to `true` or `false`; condition value is otherwise unused. |
| Unknown operator | False, including an unknown name that sounds like a negated operator. |

Return false when an attribute is missing or either operand is null, even for boolean and negative operators. Numeric parse failures and NaN comparisons return false. Empty strings remain actual operands when the attribute exists. `[]` is a valid membership list: `IsOneOf` is false and `NotOneOf` is true for an existing, non-null user operand.

Use locale-independent string operations and finite numeric parsing. An invalid list JSON value, invalid regex, or regex timeout MUST become a contained malformed-data outcome for the affected flag, rather than being inverted into a successful negative condition. Regex execution MUST have a bounded runtime or use an engine with bounded evaluation characteristics. SDKs MUST document their supported regex dialect and test common patterns across languages; unsupported patterns MUST NOT be silently reinterpreted.

## Segment matching

The following special condition properties bypass ordinary operator dispatch:

- `User is in segment`: true if any referenced segment matches.
- `User is not in segment`: true if no referenced segment matches.

For these properties, `value` is a JSON-encoded array of segment ID strings and `op` is ignored. A valid empty ID array matches no segments, so its positive form is false and its negative form is true.

Within one segment, evaluate in this order:

1. If the user is explicitly excluded, return false.
2. Otherwise, if explicitly included, return true.
3. Otherwise, return true if any segment rule matches; all conditions within that rule must match.
4. Otherwise return false.

Exclusion therefore wins when the user appears in both lists. Empty segment rules obey the same AND/OR semantics as ordinary rules. Evaluate segment rules with ordinary condition operators; recursive segment references are not allowed in FeatBit.

## Deterministic percentage rollout

Cross-language rollout compatibility is mandatory. For a rule or fallthrough:

```text
attribute = user.key                         if dispatchKey is null/empty/whitespace
            user.valueOf(dispatchKey) ?? ""        otherwise
dispatchInput = flag.key + attribute        // no delimiter, salt, or normalization
digest = MD5(UTF8(dispatchInput))
n = signed_int32_little_endian(digest[0:4])
bucket = abs(float64(n) / -2147483648.0)
```

The bucket lies in `[0, 1]`, inclusive. Convert to floating point before taking the absolute value to avoid signed-integer overflow. Explicit little-endian decoding is required; native-endian APIs MUST NOT determine the result. Do not use a language's built-in hash, unsigned decoding, a hash of hexadecimal text, or modulo 100.

For a valid interval `[lower, upper]`, match as follows:

```text
if lower == 0 and 1 - upper < 0.00001: true
else if lower == 0 and upper == 0: false
else: lower <= bucket and bucket <= upper
```

Both boundaries are inclusive. Adjacent intervals can both match their shared boundary; the first variation in stored order wins. Preserve the near-one shortcut exactly. Missing custom dispatch attributes produce `flag.key + ""`; do not silently substitute the user key. Distribution percentages are not recomputed or normalized by the SDK.

These inputs are already-combined hash keys, not separate flag/user arguments:

| Hash input | Expected bucket |
| --- | --- |
| `test-value` | `0.14653629204258323` |
| `qKPKh1S3FolC` | `0.9105919692665339` |
| `3eacb184-2d79-49df-9ea7-edd4f10e4c6f` | `0.08994403155520558` |

These vectors define the expected hash results. Conformance tests MUST additionally cover UTF-8 non-ASCII input, negative signed hashes, both interval endpoints, and the near-one shortcut.

See the [.NET SDK's `DispatchAlgorithm`](https://github.com/featbit/featbit-dotnet-sdk/blob/main/src/FeatBit.ServerSdk/Evaluation/DispatchAlgorithm.cs) for a reference implementation of this algorithm.
