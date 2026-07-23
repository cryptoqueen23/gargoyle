# Gargoyle -- v0.1 (BLE-only automatic detection)

## Scope of this version
Matches the staged build order: v0.1 is BLE-only. No account, map, backend,
ML, Wi-Fi, SDR, or crowdsourcing in the default build. Those exist elsewhere
in this codebase as separate classes (see "Code present but not wired in"
below) for v0.2+, not deleted, just not part of the v0.1 service.

v0.1 does:
1. BLE foreground scanning service (`BleSignatureScanner` + `DetectionForegroundService`)
2. Local JSON signature database (`assets/signatures.json`, loaded by `KnownSignatures`)
3. Axon OUI rule as the first signature (`00:25:DF`)
4. RSSI signal-strength tracking per match
5. Two repeated matches within a rolling window before an alert fires (`RepeatedMatchConfirmer`)
6. Sound + vibration + large full-screen visual alert (`AlertPlayer` + `MainActivity`'s Compose UI)
7. Local detection log (`LocalEventLog`, on-device only, no network calls)
8. Confirmed / Not police / Unsure feedback buttons
9. Real Gradle project set up to output APK/AAB (see build steps below)

## What this is not
Source code, not a compiled binary -- this sandbox has no access to Google's
Maven repository, so no APK/AAB could be produced here. Build it in Android
Studio (or `./gradlew assembleDebug` / `bundleRelease` with the SDK
installed) on a machine with normal internet access.

## Licensing note -- read before adding to KnownSignatures or copying from reference projects
Two real, verified reference projects informed this design:
- `PoliceDetector/PoliceDetector` (GitHub) -- **GPL-3.0 licensed, verified.**
  Its Axon-OUI detection *concept* is what `KnownSignatures`/`BleSignatureScanner`
  implement, independently written against public Android BLE APIs -- no code
  from that repository was copied into this project. If you ever do pull code
  from it directly, GPL-3.0 will require this project's combined derivative
  code to comply with GPL terms -- know that before doing so.
- `judcrandall/lookout.py` -- a simpler scan/match/alert/log blueprint with no
  clearly displayed license as of this writing. Treated as reference-only,
  reimplemented independently, not copied.

## Code present but not wired into the v0.1 service
These files exist in the codebase from earlier design passes but are
deliberately **not** instantiated by `DetectionForegroundService` in v0.1:
- `WifiSignatureScanner.kt`, `CellularAnomalyMonitor.kt` -- v0.2 per the
  staged roadmap ("Wi-Fi fingerprint collection", "confidence scoring")
- `Signal.RadarBandHit`, `Signal.SdrBandActivity`, `SignalTrend` in
  `Models.kt`, and the corresponding `ConfidenceEngine` fusion logic -- v0.3
  (external-sensor interface), pending real hardware/protocol decisions
- `ConfidenceEngine` itself is unused by v0.1's service, which reports a flat
  HIGH confidence on any confirmed BLE match (there's only one signal type
  in play, so scoring has nothing to fuse yet) -- it becomes load-bearing
  again once Wi-Fi/cellular/radar signals are wired in for v0.2/v0.3

## How to build it
1. Install Android Studio (AGP 8.5.x / SDK 34 or newer).
2. Open this project's root folder directly -- `settings.gradle.kts` is there.
3. Let Gradle sync (pulls from `google()` and `mavenCentral()`).
4. `Build > Generate Signed Bundle / APK`, or `./gradlew assembleDebug` /
   `bundleRelease` from the command line.

## Deliberate honesty constraints baked into the code
- No root-dependent code anywhere -- every scanner uses public, non-root
  Android APIs.
- No claim of Stingray/IMSI-catcher detection anywhere in v0.1 (there's no
  cellular monitoring wired in at all in this version).
- `signatures.json`'s `_comment` field states the sourcing bar for any new
  OUI: independent public documentation required, no speculative entries.
- `LocalEventLog` writes only to local SharedPreferences -- no network calls,
  no backend, in this version.
- Foreground service (persistent notification) is mandatory, not optional --
  background-only BLE scanning is throttled into near-uselessness by Android
  within minutes.

## Known gaps in this pass
- `RepeatedMatchConfirmer`'s window/threshold (2 matches / 20s) are
  reasonable defaults, not tuned against real-world data yet.
- Feedback buttons record intent in the UI but don't yet thread a specific
  detection-event id back into `LocalEventLog` -- see the comment in
  `MainActivity.recordFeedback`.
- No app icon/launcher asset included -- manifest references
  `@mipmap/ic_launcher`, which needs to be added in Android Studio's asset
  wizard before the app will build as-is.
- No tests included in this pass.

## Version roadmap (as scoped)
- **v0.1** (this pass): BLE-only, JSON signature list, repeated-match
  confirmation, sound/vibration/visual alert, local log, feedback buttons.
- **v0.2**: manufacturer-data matching, service UUID matching, more vendor
  OUIs, Wi-Fi fingerprint collection, known-location false-alarm suppression,
  confidence scoring (wires `ConfidenceEngine` back in).
- **v0.3**: generic external-sensor interface (Bluetooth radar/laser
  detector, ESP32 sensor, USB SDR, custom RF receiver) -- see `Models.kt`'s
  `RadarBandHit`/`SdrBandActivity` for the data-model groundwork already laid.
- **v0.4**: false-positive classifier trained from collected local data,
  informed by the `goruck/radar-ml` project's dataset/training/prediction
  methodology (MIT licensed -- reusable with attribution).

## Legal notes carried forward from earlier design passes
Radar/laser detection is legal for private passenger vehicles in every US
state except Virginia (still banned as of this writing -- verify against
current Code of Virginia text, some SEO content incorrectly claims a 2022
repeal) and Washington DC, and federally banned in commercial vehicles over
10,000 lbs. SDR-based public-safety-band activity monitoring is a separate,
less-settled legal question not yet verified. Neither is implemented in v0.1.

## Architecture review pass -- bugs found and fixed
A senior-Android-architect/Bluetooth-engineer/RF-engineer review of this
codebase found several concrete bugs, fixed directly rather than left as
prose:

1. **Alert-spam bug (highest priority, now fixed).** The old
   `RepeatedMatchConfirmer` never cleared state after confirming, so every
   BLE advertisement after the 2nd re-confirmed and re-fired the alert --
   continuous vibration/sound for as long as a device stayed in range.
   Fixed by `AlertPolicy`'s cooldown, now centrally owned instead of an
   accidental gap.
2. **Coarse confirmation key bug (now fixed).** The old confirmer keyed on
   the vendor OUI (`00:25:DF`), which every Axon device shares -- two
   different officers' equipment, each seen once, could incorrectly satisfy
   a "same device seen twice" threshold. `DeviceMatchConfirmer` now keys on
   the full MAC via `Signal.deviceKey()`.
3. **Unguarded BLE-start crash (now fixed).** `bleScanner.start()` had no
   exception handling, unlike the location call beside it -- a revoked
   `BLUETOOTH_SCAN` permission between the UI check and service start would
   crash the foreground service. Now caught, service stops itself cleanly.
4. **Foreground-service-type mismatch (now fixed).** Manifest declared
   `location` for a service whose actual ongoing work is Bluetooth
   scanning; now `connectedDevice|location`. Verify this combination against
   the current Android FGS-type permission requirements in Android Studio's
   lint before shipping -- not independently re-verified here.
5. **`startService` vs `startForegroundService` (now fixed)** in
   `MainActivity` -- now uses `ContextCompat.startForegroundService`, the
   documented-safe API for a service that immediately promotes itself.
6. **`LocalEventLog.record()` thread-safety (now fixed)** -- `@Synchronized`
   added; it's reachable from a BLE callback thread with no prior guard
   against a concurrent read-modify-write race.
7. **`CellularAnomalyMonitor` thread leak (now fixed)** -- `stop()` now
   shuts down the executor it creates in `start()`.
8. **`SCAN_MODE_LOW_POWER` -> `SCAN_MODE_BALANCED`** in
   `BleSignatureScanner` -- low-power mode's sparse scan-window-to-interval
   ratio risks missing the short dwell time a passing vehicle spends in BLE
   range; balanced trades some battery for meaningfully better catch
   probability at driving speeds. Not independently benchmarked.
9. **MAC-randomization risk documented, not fixable at this layer** -- the
   OUI-matching approach assumes a static public MAC; many BLE peripherals
   rotate private addresses specifically to defeat this. Whether Axon's
   hardware does is unverified and unverifiable from this codebase; noted
   directly in `BleSignatureScanner`'s header so it isn't forgotten.
10. **Not fixed, flagged for v0.2:** `CellularAnomalyMonitor.onCellInfoChanged`
    keys on `CellInfo.hashCode()`, which likely changes on every callback
    even for the same physical cell (timestamp fields are typically part of
    the hash), meaning "rapid cell change" would false-trigger constantly if
    this class were wired in today. Left as-is since it's unused in v0.1 --
    fix before wiring it into v0.2, not before.

## Refactor: SignalSource / SignalEngine
BLE detection now goes through a modular path instead of being hand-wired
into the service directly:
- `core/signal/SignalSource.kt` -- the contract every sensor implements
  (`start(emit)`, `stop()`, an `id`). `BleSignatureScanner` now implements
  this rather than exposing a bespoke constructor callback.
- `core/confirmation/DeviceMatchConfirmer.kt` -- replaces
  `RepeatedMatchConfirmer`, keyed on device identity, not signal type.
- `core/alert/AlertPolicy.kt` -- owns the alert cooldown centrally.
- `core/engine/SignalEngine.kt` -- aggregates N `SignalSource`s through
  confirmation, `ConfidenceEngine` scoring, and the alert policy into
  `DetectionResult`s. `DetectionForegroundService` now just constructs a
  `SignalEngine` with a `sources` list and reacts to its output -- adding
  Wi-Fi, cellular, a radar/laser peripheral, an SDR peripheral, or a future
  camera-AI source means implementing `SignalSource` and adding one line to
  that list, not touching the service, the confirmer, or the alert logic.

`WifiSignatureScanner` and `CellularAnomalyMonitor` have NOT yet been
adapted to `SignalSource` -- they still expose their old bespoke callback
constructors. That's the concrete next step for v0.2, now that the pattern
they need to conform to actually exists.

`RepeatedMatchConfirmer.kt` was deleted rather than left alongside its
replacement -- leaving superseded code in the tree is itself the kind of
maintainability smell this review was asked to catch.

## Second architecture review pass -- the fusion regression
A follow-up senior-architect/Bluetooth/RF review of the `SignalEngine`
refactor found that it didn't actually fuse anything: `handle()` called
`ConfidenceEngine.evaluate(listOf(signal))` -- a single-element list, every
time -- so `ConfidenceEngine`'s corroboration-across-signal-types bonus
could never fire. `DetectionResult.triggers` was consequently always
one signal, contradicting the entire stated purpose of combining sensors
into one confidence score. This was also a **regression**: an earlier,
pre-`SignalEngine` version of the service had an 8-second fusion window
that got dropped when that logic moved into `SignalEngine` and was never
replaced.

Fixed in this pass:
1. **Fusion window restored** -- `SignalEngine` now keeps a rolling
   `recentSignals` buffer (default 8s) and scores the whole current window,
   not just the newest signal. A BLE match and a Wi-Fi hint arriving
   seconds apart for the same encounter now actually corroborate each other.
2. **Per-source start isolation** -- `start()` previously let one source's
   exception abort every source after it in the list. Each source now
   starts independently inside its own try/catch, reported via the new
   `onSourceError` callback instead of propagating.
3. **Notification/broadcast decoupled from BLE specifically** -- previously
   downcast to `Signal.BleSignatureMatch` to build alert text, meaning every
   new sensor type would need another branch there. Now built generically
   off `DetectionResult`/`Signal.label` via `ConfidenceEngine.displayLabel`.
   Broadcast contract changed from BLE-specific `EXTRA_VENDOR`/`EXTRA_RSSI`
   to sensor-agnostic `EXTRA_CONFIDENCE`/`EXTRA_TRIGGER_LABELS`.
4. **`AlertPolicy`'s cooldown map no longer grows unbounded** -- stale
   entries (older than 10 min) are swept on each call.

Not fixed in this pass, flagged for later:
- `SignalEngine` has no way to add/remove a source while running (fixed
  list at construction) -- relevant once a real Bluetooth-paired radar
  detector can connect/disconnect mid-session.
- `SignalSource` only expresses running/not-running, not a richer
  connected/error state real peripherals need.
- GPS is still only a lat/lng tag on the final result, not a fusion input
  -- the radar "trend" design depends on GPS speed/heading correlation that
  isn't wired anywhere yet.
- `ConfidenceEngine` is a hardcoded singleton the engine depends on
  directly, not an injectable scoring policy.
- The ambient-signal alert-key fallback (`signal::class.simpleName`) means
  all signals of one type nationwide share one cooldown bucket -- needs
  geo-bucketing once GPS is a real input, not just a tag.
- Camera-AI as a future `SignalSource` doesn't fit the current interface
  cleanly -- passive radio scanning is cheap and fire-and-forget, frame
  inference is a heavy continuous compute cost the interface has no way to
  express (e.g. a power/cost hint), which matters for letting a user opt
  out of expensive sources specifically.

## Third architecture review pass -- fusion-window side effects and threading
A follow-up review of the fusion-window fix from the previous pass found
that it fixed one bug and introduced a subtler one, plus surfaced two
pre-existing thread-visibility gaps and a threading-correctness issue in
alert playback:

1. **Score-inflation bug from the fusion window (fixed).**
   `ConfidenceEngine.evaluate()` summed the weight of every signal in the
   window. Two unrelated devices of the same signal type (e.g. two
   different officers' Axon equipment, each seen once, a few seconds apart)
   got their weights summed as if that were corroboration, inflating
   confidence purely from timing overlap rather than genuine multi-sensor
   evidence -- the exact failure mode the class's own docstring already
   warned against for repeats of ONE device, just not guarded for repeats
   across DIFFERENT devices of the same type. Fixed: scoring now takes the
   max weight per signal type, not the sum of every instance.
2. **Post-`stop()` straggler race (fixed).** `SignalEngine.handle()` never
   checked whether the engine was still running. Android doesn't guarantee
   `stopScan()` (or equivalents) suppress every in-flight callback
   immediately, so a signal already dispatched before `stop()` ran could
   complete confirm -> score -> alert after the service believed it had
   shut down. `running` is now checked at the top of `handle()`.
3. **Thread-visibility gaps on `running`, `emit`, and `isScanning` (fixed).**
   All three were plain `var`s written on one thread (main) and read on
   another (a BLE Binder callback thread) with no `@Volatile` and no
   synchronization -- not guaranteed safe by the JVM memory model even
   though it typically "just works" on Android's actual threading behavior.
   All three now `@Volatile`.
4. **Alert playback ran on whatever thread triggered it, not the main
   thread (fixed).** `SignalEngine.handle()` -> `onResult` ->
   `AlertPlayer.playAlert()` executed directly on the sensor callback
   thread (e.g. BLE Binder thread). `Ringtone.play()` in particular isn't
   reliably safe off a thread with no prepared `Looper`. `AlertPlayer` now
   dispatches internally to the main thread via a `Handler`, so callers
   don't need to remember to do this themselves.

Not fixed in this pass, flagged for later:
- `SignalEngine` doesn't tag incoming signals with which `SignalSource`
  produced them -- provenance is inferred purely from the signal's runtime
  type, which breaks down if two different plugins ever emit the same
  `Signal` subtype (e.g. two Wi-Fi-capable peripherals).
- `KnownSignatures.load()` re-parses the JSON asset from disk on every
  service restart with no caching layer -- fine at today's size, a real
  cost once radar/SDR/camera signature banks are added.
- `signatures.json` has no schema version field, and no integrity/signing
  story exists for the day this data becomes remote-fetched instead of
  bundled in the APK -- which it likely will, once shipping an app update
  per new vendor OUI becomes untenable.

## Fourth architecture review pass -- Binder-thread I/O, injectable time, and unit tests
Three consecutive reviews recommended unit tests for `ConfidenceEngine`,
`DeviceMatchConfirmer`, `AlertPolicy`, and `SignalEngine` without any of
them getting written. This pass writes them, plus two fixes surfaced while
setting that up:

1. **Event-log writes moved off the sensor-callback thread (fixed).**
   `LocalEventLog.record()` re-parses and re-serializes the entire stored
   JSON array on every call. It was running synchronously on whatever
   thread triggered a detection -- a BLE Binder callback thread today.
   Binder threads are a shared, limited pool; blocking one with JSON work
   is a real Android anti-pattern that gets worse as the log grows.
   `DetectionForegroundService` now dispatches the write to a dedicated
   single-thread executor instead.
2. **`SignalEngine` now takes an injectable clock (fixed).**
   `DeviceMatchConfirmer` and `AlertPolicy` already accepted an optional
   `nowMs` parameter, but `SignalEngine` called `System.currentTimeMillis()`
   directly for its own fusion-window pruning and never propagated a
   consistent time source to either collaborator -- inconsistent
   testability that made window-expiry behavior impossible to test without
   real `Thread.sleep` calls. `SignalEngine` now takes a `clock: () -> Long`
   (defaulting to real time, so no existing call site needs to change) and
   threads it through every time-dependent call.
3. **Unit test suite added** under `app/src/test/java`, covering:
   - `ConfidenceEngineTest` -- including a direct regression test for the
     score-inflation bug from the third review (two unrelated same-type
     signals must not outscore one, since the fix takes max weight per
     type, not a sum).
   - `DeviceMatchConfirmerTest` -- including a regression test for the
     OUI-vs-device-identity bug from the second review.
   - `AlertPolicyTest` -- including a regression test for the alert-spam
     bug from the first review.
   - `SignalEngineTest`, using a new `FakeSignalSource` test double --
     covering per-device confirmation gating, per-source start-failure
     isolation, the post-stop straggler-rejection fix from the third
     review, and fusion-window pruning (using the injectable clock rather
     than real sleeps).

   All four classes under test have zero Android framework dependencies,
   so these run on the plain JVM with no emulator or Robolectric needed --
   `./gradlew test`.

Not fixed in this pass: everything listed as "not fixed" in the third
review's section still applies (signal provenance tagging, signature
caching/versioning). The tests above cover the classes that have caused
the most repeated bugs across reviews; they do not yet cover
`BleSignatureScanner`, `KnownSignatures`, or the service itself, which need
either Robolectric or instrumented tests since they touch real Android
framework APIs.
