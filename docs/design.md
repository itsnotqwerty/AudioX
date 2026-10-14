# AudioX Architectural Design

This document is the normative implementation guide for the
[technical specification](spec.md). It describes the architecture to build, not
existing modules. MUST and MUST NOT are mandatory within the applicable
[roadmap phase](roadmap.md). Architectural changes require a written decision
record with alternatives, consequences, migration impact, and evidence that the
replacement still meets the release gates.

## 1. Fixed Design Choices

| Decision | Rule | Reason |
| --- | --- | --- |
| First-party implementation | Stable Rust; explicit, audited FFI boundaries | Keep ownership and thread safety visible |
| Platform | Linux first, PipeWire-native audio and MIDI | Integrate with the target desktop audio graph |
| UI | egui/eframe with wgpu; Rust layout, no HTML/webview | Share Rust types without introducing a web runtime |
| Engine/UI relationship | Headless engine, message-driven UI | UI responsiveness cannot determine audio timing |
| Audio graph | Precompiled acyclic graph, ordered block processing | Bounded work and explicit routing/latency |
| DSP representation | Planar `f32` buffers, fixed prepared capacities | Predictable memory layout and bounded processing |
| Plugin execution | Separate scanner; one worker process per active instance | Contain crashes and missed deadlines |
| Persistence | Versioned JSON in a `.audiox` ZIP container | Inspectable metadata with portable media |
| Timing authority | Engine sample clock plus explicit musical-time conversion | Avoid UI-clock drift and ambiguous timing |
| Distribution | Offline-capable application, no required service | Projects remain usable without accounts or connectivity |

The UI toolkit choice MUST be exercised in P0 for Linux input, scaling,
accessibility, and graphics behavior. A failed feasibility gate requires a
documented architecture revision, not an undocumented second UI stack. Exact
crate versions, minimum supported Rust version, and runtime version floors MUST
be pinned during P0 after validation; they are not invented in this document.

## 2. Ownership and Module Boundaries

Use a Cargo workspace with the following logical boundaries. These are planned
module/crate names; this documentation does not imply they already exist.

| Boundary | Owns | MUST NOT depend on |
| --- | --- | --- |
| `audiox-model` | Typed IDs, musical data, commands, schema-facing data types | UI, PipeWire, plugin SDKs |
| `audiox-dsp` | First-party processors, smoothing, resampling adapters | UI, project archives, plugin discovery |
| `audiox-engine` | Scheduling, compiled graph, transport, prepared audio buffers | UI, filesystem operations, plugin SDK APIs |
| `audiox-platform-pipewire` | Device graph integration, negotiated formats, audio/MIDI I/O | UI widgets, project serialization |
| `audiox-plugin-protocol` | Versioned IPC messages, shared-memory layouts, stable handles | UI toolkit, format SDKs |
| `audiox-plugin-worker` | Format adapters, plugin instances, state and editor lifecycle | Application document mutation |
| `audiox-project` | Save/load, schema migration, media collection, recovery | Live callback state |
| `audiox-app` | Document controller, commands, job orchestration, egui views | Direct plugin ABI calls or callback-owned mutable memory |

Dependencies MUST remain acyclic. Shared types belong in the narrowest lower-level
boundary that owns their meaning; a catch-all utility crate is not a substitute
for ownership. Use mature, license-compatible crates for serialization, archives,
codecs, resampling, and platform integration after qualifying their behavior.

The document controller is the single writer of editable project state. The
engine owns mutable runtime DSP state. The UI reads document snapshots and emits
typed commands; it MUST NOT mutate the engine or serialize a live DSP graph.

## 3. Thread and Process Model

| Execution context | Permitted work | Communication |
| --- | --- | --- |
| UI thread | Input, layout, rendering, document command dispatch | Typed controller commands and read-only snapshots |
| Control executor | Validate edits, maintain history, coordinate jobs and workers | Bounded queues and revisioned snapshots |
| Graph-preparation worker | Compile routes, allocate buffers, prepare processors and compensation | Prepared graph handles to engine |
| PipeWire real-time callback | Transfer input/output, schedule events, process prepared graph | Preallocated queues, atomics, shared-memory slots |
| Media/disk workers | Decode, resample, prefetch, record writes, peaks, save/export jobs | Bounded audio rings and job results |
| Plugin scanner process | Discover and validate one candidate at a time | Timed, versioned IPC |
| Plugin instance process | Plugin DSP, parameter/state API, native editor event loop | Audio/event shared memory plus control IPC |

No queue may grow without bound. Each queue MUST document capacity, producer,
consumer, ordering, ownership transfer, and overflow behavior. Use established
bounded queue primitives and validate their real-time behavior; do not implement
a bespoke lock-free queue without an independently reviewed need.

### Real-Time Prohibitions

The callback and first-party DSP MUST NOT allocate or free heap memory, acquire
locks, wait on workers, perform filesystem/network I/O, spawn tasks, call UI APIs,
format logs, or unwind. They MUST NOT use channels whose send or receive can
block. No destructor with unbounded work may run on this path.

Prepared plans are installed at block boundaries through bounded ownership
transfer. Retired plans, buffers, and reference-counted objects go to an off-thread
reclamation queue; the callback MUST NOT release the final owning reference.
If retirement capacity is exhausted, retain the current plan and defer the swap.

Diagnostics use fixed-size events or counters, formatted elsewhere. Meter updates
may be overwritten by newer values. Document commands MUST be rejected visibly
when full, not silently dropped. Critical stop/panic signals need a reserved
bounded path. Recording overflow MUST stop and flag the take; playback underflow
MUST produce defined silence and an error counter rather than stall.

## 4. Audio Graph and Processing

The control side validates routing and compiles an immutable execution plan with
topological node order, channel mappings, buffer assignments, event routes, and
latency compensation. Mutable processor state is separately owned by the engine.
Reject feedback cycles, invalid bus layouts, and resource requirements exceeding
the prepared budget before activating an edit.

MVP uses one first-party graph execution thread per engine instance, driven by
PipeWire. Plugin workers run independently; general parallel graph scheduling is
deferred until benchmarks justify it. No per-track task spawning in the callback.

Graph replacement MUST keep unaffected processor state and tails where possible.
For MVP, edits requiring processor migration or reallocation MUST stop playback
explicitly and rebuild off-thread. Live-safe edits may use a bounded ramp or
crossfade with its extra buffers and CPU included in admission checks. Do not
promise uninterrupted arbitrary graph rewiring before it is implemented.

Buffers use planar `f32` samples; gains are smoothed using sample-time ramps, not
UI frame timing. DSP units MUST define reset, silence, tail, bypass, denormal, and
non-finite-sample behavior. Non-finite plugin output is contained at the worker
boundary and reported. First-party DSP is tested for finite output within its
documented input ranges.

### Quantum and Latency Rules

PipeWire's negotiated quantum is not assumed to remain constant. Preallocate a
bounded quantum adapter between device buffers and the configured plugin-worker
block size. Use sequence-numbered shared-memory slots and explicitly scheduled
pipeline delay; never wait in the callback for the current input block to return.

Worker outputs are consumed only at their scheduled deadline. Missing/late output
uses preallocated silence for instruments or latency-aligned dry bypass for
compatible effects. Discard stale results. A timeout is not permission to reuse
audio from an unrelated block. Repeated failures quarantine the instance until
the user requests a restart.

Every node reports intrinsic and adapter latency. Compute cumulative path latency
and align parallel paths with prepared delay lines, including dry/bypass paths.
Dynamic latency changes require off-thread recompilation; if capacity is
insufficient, stop and rebuild rather than truncate the requested delay. This
aligns playback but cannot remove live input's physical or pipeline latency.

Rate changes, oversized quantum changes, and device loss trigger a controlled
stop/reconfiguration with all notes released. The callback outputs silence while
unprepared. UI notification, device reconnection, and resource allocation occur
off-thread. Recordings MUST NOT silently combine samples at different rates.

## 5. Musical Timing and Events

Store musical positions and durations as signed 64-bit integer ticks at 9,600
ticks per quarter note. Use signed 64-bit absolute sample positions for engine
time and a double-precision tempo conversion with one documented rounding rule.
MVP supports fixed tempo and meter; future tempo maps must preserve these APIs.
Calculate boundaries from absolute positions, not repeated rounded increments.

Events carry a target sample position, stable destination, and deterministic
tie-break order. Compile bounded lookahead away from the callback and consume
events over half-open block intervals. First-party processors split processing at
event boundaries; format adapters translate offsets into their supported event
representation. Do not claim sample accuracy where a plugin only accepts
block-rate parameter changes.

Seek, stop, loop wrap, device restart, and offline render start each have explicit
note-release, sustain, automation-chase, and DSP-reset policies. Transport changes
increment an epoch so stale queued work and plugin output can be discarded.
PipeWire MIDI timestamps are mapped to the engine clock; the UI clock is never
the recording timestamp source.

## 6. Plugin Boundary

Implement a format-neutral host contract for ports, event support, parameters,
state blobs, transport, latency, tails, and editor capabilities. Format-specific
SDK types MUST remain inside adapters in the worker process. Native Linux VST3
is the MVP adapter; VST2 requires a separate legal and technical qualification.

Shared-memory layouts MUST use fixed-width fields, explicit versioning, validated
sizes, sequence numbers, and bounded arrays. Never transmit Rust pointers or rely
on Rust's default memory layout across processes. Process handles and Unix-domain
control sockets are managed off the real-time thread.

The scanner runs with time and resource limits, caches results by binary identity,
and quarantines crashing candidates. Discovery requires configured trusted search
roots; opening a project cannot cause arbitrary executable paths to be scanned or
launched. Workers inherit only required file descriptors and resources. This is
crash containment; a malicious plugin can still exercise user-level privileges.

Plugin state operations and lifecycle changes may lock inside third-party code.
Schedule them outside the host callback and serialize them against that plugin's
processing according to its SDK contract. A worker may temporarily miss output
while state is captured; the engine uses its defined fallback, never waits.
Manual save MUST report state-capture failure and leave the previous good save
intact. Recovery may retain the last valid plugin state with an explicit warning.

Native editors run on the worker's required event-loop thread in separately
managed windows. Do not embed cross-process plugin widgets in egui or pretend
X11 windows work natively on Wayland. Qualify XWayland/native paths separately;
generic parameter controls remain available when an editor cannot open.

## 7. Project State, Media, and Undo

All durable entities use stable UUIDs, never vector indexes or runtime pointers.
Document edits are typed, validated transactions with inverse data for undo.
Coalesce a drag into one undoable edit. External plugin-editor changes require
explicit synchronization; arbitrary opaque plugin changes are not assumed to be
fully undoable. Asset deletion waits until no document/history/recovery reference
requires the media.

The `.audiox` archive contains `manifest.json`, content-addressed media under
`media/`, and opaque state under `plugins/`. The manifest records schema version,
application version, IDs, media hashes, and format-qualified plugin identifiers.
Hashes verify media identity; they do not make third-party state safe to execute.
An adjacent recovery workspace holds active recordings and journals until packed
into a successful project save; never record directly into the ZIP archive.

Save from a revisioned immutable document snapshot, collect plugin state under a
coordinated capture policy, validate references, and write a temporary archive in
the destination directory. Flush file contents, atomically rename on the same
filesystem, and flush the parent directory. If the filesystem cannot provide the
required semantics, report that limitation rather than claim crash-safe saving.
Edits after the captured revision remain marked unsaved.

Loads MUST validate paths, entry counts, expanded sizes, hashes, and schema before
extracting content or preparing audio. Reject archive traversal, duplicate paths,
symlinks, and decompression bombs. Treat project metadata and plugin state as
untrusted input. Newer unsupported schemas open with a clear error; migrations
are explicit, tested, and never overwrite the only original copy.

## 8. Media, Export, and Resource Budgets

Media workers decode/resample into prepared caches and streaming rings; the
callback only reads ready samples. Disk writers consume recording rings and
finalize recoverable chunks. RAM budgets, prefetch horizons, queue capacities,
maximum events per block, and permitted graph sizes MUST be explicit settings or
documented limits, with observable admission failures.

Offline export instantiates the same engine with an offline clock and independent
runtime state. It may wait on plugin workers outside a real-time callback, but
must preserve scheduling and latency semantics. Export progress and cancellation
are bounded and MUST NOT leave a partial file presented as a successful export.
Offer real-time rendering for plugins that cannot render correctly offline.

No exporter may substitute a simplified mixer. Reuse codec and DSP libraries
after license review and correctness tests. Do not implement custom FFT, sample
rate conversion, or time-stretch algorithms solely to avoid a dependency.

## 9. Safety, Diagnostics, and Test Gates

- Keep `unsafe` limited to audited platform, FFI, and shared-memory boundaries;
	each use MUST state ownership, lifetime, alignment, and thread assumptions.
- First-party audio paths MUST avoid panic-based error handling. Callback entry
	points MUST prevent unwinding across FFI; error containment is not a substitute
	for prevalidated, non-panicking processing.
- Application code uses structured errors. Logs include version and timing data,
	but project paths, media names, and plugin state require redaction for sharing.
- CI MUST run formatting, linting, unit/integration tests, schema fixtures, DSP
	reference tests, and a headless render suite. Hardware timing runs separately on
	the published reference configuration, not on an arbitrary shared CI runner.
- Add callback allocation/deallocation instrumentation, queue saturation tests,
	scheduling/latency impulse tests, and controlled plugin crash/hang fixtures.
- Use fuzzing for project parsing and IPC validation, and applicable Rust safety
	tooling for unsafe boundaries. Fuzz targets have explicit resource limits.
- Dependency upgrades MUST preserve project fixtures and plugin compatibility
	tests. Pin the application lockfile and review license/security changes.

## 10. Decisions Required Before Implementation Gates

P0 must select and record license terms, SDK/binding provenance, exact dependencies,
Rust version, supported Linux runtime floors, reference hardware, resource limits,
and distribution packaging. These are unresolved inputs, not permission to relax
the fixed architecture. Later additions such as ARM64, multichannel routing, and
advanced DSP libraries require their own qualification before release claims.

Full Windows/macOS ports, Windows-plugin bridges, collaborative editing, cloud
services, and FL Studio project import require a separate scope decision. No
current module should pretend those integrations already exist.

