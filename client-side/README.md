# FeatBit Client-Side SDK Specification

This specification describes the behavior of FeatBit client-side SDKs across languages and platforms. It focuses on what applications configure, how the current user's flag results are managed, and what applications can expect from the public API.

**Status: Draft.** Version 1.0 is not yet a released conformance target.

## Core model

A client-side SDK serves **one active user context at a time**. FeatBit evaluates targeting rules on the server and delivers the resulting flag values for that user. The SDK synchronizes and stores those results, returns values locally, notifies the application about changes, and sends usage and metric events.

Targeting rules, segments, percentage allocation, and experiment sampling remain server responsibilities. Client applications use a client SDK key; they must not contain a server-side environment secret.

## Core specification

| Module | Scope |
| --- | --- |
| 1. [User Configuration](spec/configuration.md) | Client SDK key, endpoints, startup, synchronization mode, offline use, and event settings. |
| 2. [User Identity](spec/identity.md) | Active user, attribute changes, identity switching, and isolation. |
| 3. [Data Synchronization](spec/synchronization.md) | WebSocket, optional Polling, initialization, freshness, and recovery. |
| 4. [Data Storage](spec/storage.md) | Local results, versioned updates, bootstrap, and optional persistent cache. |
| 5. [Feature Flag Evaluation](spec/evaluation.md) | Local reads, value conversion, result details, and fallback. |
| 6. [Event Processing](spec/events.md) | Evaluation and metric events, deduplication, bounded delivery, and flush. |
| 7. [Public API](spec/public-api.md) | Application-facing operations, notifications, and lifecycle. |
| 8. [General Principles](spec/general.md) | Error containment, privacy, resource use, and compatibility. |

Read the modules in this order for an overview. Requirement levels and scope are defined in [General Principles](spec/general.md). All eight modules apply together. API spelling, storage technology, scheduling, and framework integration follow each language's conventions.

## Mobile supplement

The [mobile supplement](mobile/README.md) adds a shared Android/iOS lifecycle and resource-use profile without replacing the core modules or wire protocol. It includes [Android requirements](mobile/android.md), [iOS requirements](mobile/ios.md), and a [mobile acceptance checklist](mobile/conformance.md). Core conformance alone does not imply conformance to this additional profile.

## Supporting references

The [protocol reference](reference/protocol.md) retains the wire details needed for interoperability. The [acceptance checklist](spec/conformance.md) summarizes observable scenarios to verify. [Shared fixtures](fixtures/README.md) are planned; no executable shared fixture suite is currently included.

The [JavaScript Client SDK](https://github.com/featbit/featbit-js-client-sdk) is a behavioral reference, not a mandatory architecture or proof of conformance. Existing SDK behavior must be checked against this draft before claiming compliance.
