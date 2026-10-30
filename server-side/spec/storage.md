# Data Storage

[Specification index](../README.md) | [General principles](general.md)

The SDK keeps the environment's flags and segments locally so evaluation can continue without contacting the server.

## Stored information

Store the information needed to select variations: flag identity and state, variations, individual targets, ordered rules, default rollout, experiment settings, and referenced segments. Each entity also carries its server-provided version and archive marker.

Preserve server values and rule order. Flag keys and user keys are case-sensitive. Flag and segment identities must remain distinct; segment identifier parsing and lookup must use the same representation. Unknown fields must not break compatible updates. See the [wire data model](../reference/protocol.md#required-evaluation-data) for field names.

## Updates

| Update | Required effect |
| --- | --- |
| Full data set | Replace all previous flags and segments, regardless of old versions. Missing entities disappear; an empty full response clears the store. |
| New entity in a patch | Insert it. |
| Newer version in a patch | Replace the previous entity. |
| Equal or older version in a patch | Ignore it. |
| Archived entity | Hide it from evaluation and enumeration. Only identity, version, and archived state are needed for archived entities; omit all other data from storage. |
| Newer active version of an archived entity | Restore it. |

Versions come from server update timestamps, preserving millisecond precision. They MUST NOT be replaced with local receive times. The synchronization cursor is the maximum committed entity version, including archived entities, or zero for an empty store.

A full replacement can discard previous archive records. Invalid entities follow the explicit [full/patch skip rules](synchronization.md#invalid-data).

## Consistent local reads

Each evaluation, including its segment lookups, MUST observe one coherent committed view. Full replacement or a patch batch must not expose a partially applied update. Bulk evaluation must likewise use a consistent view.

Callers MUST NOT be able to alter internal data through returned objects or by mutating previously supplied input. These are behavioral guarantees; the storage structure and concurrency mechanism are language-specific.

A network outage MUST NOT expire the last usable data. Persistent storage is optional. If offered, it must isolate environments, preserve data and cursor consistently, and document when cached data is eligible for evaluation.
