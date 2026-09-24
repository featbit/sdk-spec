# iOS Requirements

[Mobile supplement](README.md) | [Android](android.md) | [Acceptance checklist](conformance.md)

These requirements apply together with the shared mobile profile and core specification. Support for macOS, watchOS, tvOS, and extensions must be declared separately; an option for another Apple platform does not imply iOS support.

## Application and scene integration

Observe application visibility or accept equivalent host lifecycle signals. For scene-based applications, define how multiple scenes determine the client's foreground state; one scene entering the background must not incorrectly suspend a client still serving a foreground scene. Temporary inactivity must not be confused with permanent closure.

If lifecycle integration is an optional package or host adapter, document its installation, initial visibility, detachment, and ownership. Apply known state before network work starts. Do not retain a view controller or scene merely to keep the SDK alive.

Connectivity/path notifications are hints. Reconcile network changes without creating duplicate connections; handle failed requests when the reported path is available. Explicit offline configuration always takes precedence over a network recovery signal.

## Suspension and resumption

iOS SDKs MUST use the shared background-pause policy. Do not expose a background polling interval as a promise that iOS will execute it. Normal operation must not require background modes, capabilities, or entitlements to keep Flag synchronization alive.

Any optional background-transition event flush must be bounded by both the SDK budget and available platform execution time. If a finite background task is used, end it on completion or expiration and prevent further attempts after expiration. Never claim the task guarantees delivery or execution after force termination.

The platform expiration handler MUST apply the shared transition-cutoff rules: invalidate active attempts, cancel supported requests, and retain unacknowledged work under bounded policy before ending the background task, without waiting for network completion. Late responses cannot alter newer delivery attempts or revive the expired task.

On foreground return, recreate or validate transport state through fresh synchronization rather than assuming the previous connection survived suspension. Resume timers without catch-up bursts. Process termination callbacks are not a persistence or event-delivery mechanism.

## API, storage, and distribution

Provide synchronous in-memory reads and asynchronous bounded waits using Swift-appropriate APIs. Completion closures and async/await are implementation choices; Objective-C support is optional and must be declared. UI-facing listener callbacks SHOULD run on the main thread; do not put network or disk work on the main actor to achieve callback consistency.

Use application-private storage when persistence is offered. Handle data-protection unavailability as a recoverable cache miss. Document backup/restore, reinstall and anonymous-key behavior, and clear operations. Sharing with app extensions or App Groups is optional and requires explicit coordination; it must not happen implicitly.

Publish the supported iOS and Swift/Xcode versions, package integration, and optional interoperability. Ship applicable privacy-manifest resources with each supported distribution format and verify their inclusion in a consuming application. Disclose actual collection and required-reason API use; a manifest must describe this implementation rather than copy another SDK's declarations. Do not collect device/application attributes without the documented opt-in.

Validate foreground/background transitions, termination and relaunch, offline startup, and protected storage behavior on supported devices. Simulator-only checks are insufficient evidence for suspension and process-lifetime behavior; report real-device coverage separately.
