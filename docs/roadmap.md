# AudioX Roadmap: Zero to Full DAW

This is a dependency-ordered delivery plan, not a schedule or a statement of
completed work. No phase is marked complete in this documentation-only workspace.
The [specification](spec.md) defines behavior; the [architecture](design.md)
defines implementation constraints. A phase closes only when its exit evidence
exists and unresolved failures are recorded.

## Release Map

| Phase | Outcome | Depends on | Release meaning |
| --- | --- | --- | --- |
| P0 | Foundation and feasibility | None | Architecture and build baseline |
| P1 | Reliable headless audio engine | P0 | Engine prototype, not a DAW |
| P2 | Composition and project vertical slice | P1 | Internal alpha |
| P3 | Recording, plugins, mixing, and release hardening | P2 | MVP |
| P4 | Production workflow depth | P3 | Production beta |
| P5 | Advanced instruments and editing | P4 | Feature-complete candidate |
| P6 | Compatibility, accessibility, and release qualification | P5 | Full-featured DAW |
| P7 | Maintenance and separately approved expansion | P6 | Continuing releases |

Research spikes may run in parallel, but a feature is not done merely because its
UI exists. Each needs persisted state, undo semantics where applicable, failure
handling, engine integration, and tests. There are no date estimates until team
capacity and P0 feasibility results are known.

## P0: Start from Zero

### Deliverables

- Establish the Rust workspace and module dependency rules from the architecture.
- Choose a project license and contribution policy; review SDK, binding, crate,
	and bundled-asset licenses before distributing any dependency or content.
- Pin toolchain/MSRV, dependencies, CI formatting/lint/test checks, and release
	profiles. Record supported distro, PipeWire, display, and GPU baselines.
- Define project schema v1, stable IDs, document commands, engine control protocol,
	plugin IPC, and bounded-resource policies.
- Prove egui/eframe + wgpu on reference Wayland and X11 systems, including keyboard
	focus, scaling, accessibility integration, and graphics error behavior.
- Prove PipeWire audio/MIDI I/O, device/rate/quantum discovery, and controlled
	reconfiguration with a small disposable test graph.
- Prove lawful native Linux VST3 discovery and out-of-process processing using a
	redistributable test plugin; measure shared-memory pipeline overhead.
- Publish reference hardware and deterministic session fixtures for performance
	gates, including the 32-track workload specified in the technical spec.

### Exit Gate

A fresh development machine can build and run the native UI and headless audio
spikes using documented steps. Audio and plugin IPC operate without callback
allocation or waiting. Measured latency fits an explicitly recorded budget.
Blocking SDK/toolkit issues have a written resolution or architecture revision.
No end-user DAW functionality is claimed at this stage.

## P1: Engine Foundation

### Deliverables

- Implement prepared planar buffers, deterministic block scheduling, transport,
	fixed-tempo musical-time conversion, and the offline clock.
- Compile acyclic routing graphs with gain, pan, buses, sends, meters, and bounded
	ownership transfer/reclamation.
- Add sample-offset note/automation events, smoothing, note release on transport
	changes, and latency-compensated parallel paths.
- Integrate PipeWire, quantum adaptation, underrun/overrun counters, and controlled
	recovery from device loss or unsupported parameter changes.
- Add basic sample playback, a gain processor, and an offline WAV render harness.
- Add allocation instrumentation, timing fixtures, queue-overflow tests, and
	transport seek/loop tests.

### Exit Gate

The headless engine renders known sample and MIDI fixtures with correct event
boundaries and latency alignment. Live and offline built-in fixture outputs match
within the declared tolerance. A 30-minute foundation workload has no engine
deadline misses on reference hardware; all resource limits and failures are
observable. This is an engine prototype, not the final reference-workload gate.

## P2: First Complete Composition

### Deliverables

- Build transport, browser, pattern rack, piano roll, playlist, and basic mixer
	views backed by typed document commands and undo/redo.
- Implement pattern reuse/make-unique, note/velocity editing, snapping, and basic
	audio clip operations with waveform caching.
- Add WAV/FLAC import, worker-side resampling, managed media, a sample instrument,
	and a basic subtractive synthesizer.
- Support keyboard audition and basic timestamped PipeWire MIDI input/recording.
- Implement schema v1 saves/loads, portable media collection, autosave/recovery,
	missing-media placeholders, and archive validation.
- Expose offline WAV export with range, format, tails, and cancellation.

### Exit Gate

A user can create a short arrangement with two instrument parts and an imported
audio clip, edit it, save, close, reopen, and export it without losing musical
data. Tests cover linked patterns, undo/redo, relocation to another directory,
missing media, malformed archives, interrupted saves, and export cancellation.
This is an internal alpha: recording and third-party plugin hosting are not yet
required to be release-ready.

## P3: MVP

### Deliverables

- Add input/output selection, track arming, mono/stereo recording, input monitoring,
	metronome/count-in, latency alignment, and recoverable recording writes.
- Complete mixer inserts, buses, sends, mute/solo semantics, peak/clipping meters,
	and latency compensation for worker-hosted plugins.
- Add editable step/linear automation lanes for built-in and plugin parameters.
- Ship native Linux VST3 instrument/effect hosting: scan cache, quarantine,
	per-instance workers, generic controls, state persistence, missing-plugin
	placeholders, crash/hang containment, and restart behavior.
- Qualify a redistributable instrument/effect fixture pair and at least one
	independently developed native Linux VST3 plugin; publish exact tested versions.
- Complete the MVP device set: sample instrument, subtractive synth, gain/pan,
	equalizer, and delay, with documented presets and redistributable demo content.
- Complete 24-bit PCM/32-bit float WAV export and a real-time render fallback.
- Package the application with installation steps, runtime requirements, license
	notices, limitations, and a reproducible versioned demo session.

### MVP Acceptance Session

1. Start a new project and select a PipeWire input/output configuration.
2. Program a sample-based pattern and a synthesizer part in the piano roll.
3. Arrange those parts with imported audio and record a new mono/stereo take.
4. Load a native Linux VST3 instrument and effect; save their configured state.
5. Mix through inserts and a send bus, then automate an audible parameter.
6. Save, quit, reopen, and verify clips, notes, media, routing, automation, and
	 plugin state are preserved.
7. Export the arrangement to WAV, compare its timing with live playback, and
	 confirm that the file is playable outside AudioX.
8. Kill or hang a test plugin and confirm that the application remains responsive,
	 audio follows the documented fallback, and the project can still be recovered.

### Exit Gate

All MVP requirements in the specification pass, including the complete reference
workload, callback/UI timing gates, tested sample-rate/quantum matrix, Wayland/X11
application checks, interrupted-save/recording recovery, and documented plugin
compatibility. Core workflows are keyboard-operable. No known reproducible data
loss, callback deadlock, or uncontained plugin crash is acceptable for MVP.

### Explicitly Not Required for MVP

- Legacy VST2, Windows-plugin bridging, other plugin families, or universal native
	plugin-editor compatibility; generic controls are mandatory.
- Tempo/meter maps, advanced MIDI expression, external clock sync, or MIDI output.
- Time stretching, pitch correction, comping, sidechains, multichannel buses,
	freeze, stem export, or the expanded instrument/effect library.
- ARM64 qualification, Windows/macOS ports, cloud services, or FL Studio project
	import.

## P4: Production Workflow Depth

### Deliverables

- Add tempo/meter maps, swing/groove, arrangement markers, regions, track folders,
	and grouped editing, with schema migrations and timing regression fixtures.
- Add punch/loop recording, take lanes, comping, and crossfades.
- Add sidechains, multichannel bus negotiation, freeze/unfreeze, bounce in place,
	and stem export with defined shared-effect handling.
- Add touch/latch/write automation, modulation links, preset browsing, and
	MIDI file import/export plus controller mapping.
- Improve plugin editor window handling on qualified display configurations;
	preserve generic controls and worker isolation.

### Exit Gate

Representative multitrack recording and arrangement projects survive edit/undo,
freeze/unfreeze, save/reopen, and stem rendering. Tempo changes remain aligned
across notes, audio, and automation. Latency tests include sidechains and dynamic
plugin latency. Projects from every previously released schema migrate without
overwriting their originals. The MVP reference workload remains within budget.

## P5: Advanced Creative Tools

### Deliverables

- Qualify DSP dependencies and add time stretching, pitch shifting, warping,
	transient detection, and sample slicing.
- Expand the drum sampler and synthesis tools; add compressor, limiter, reverb,
	filters, distortion, and modulation effects.
- Add scale aids, advanced quantization, supported per-note expression, MIDI
	output, clock sync, and external-instrument latency calibration.
- Add loudness/true-peak metering and phase/routing inspection.
- Expand codec support only after technical and licensing qualification.

### Exit Gate

Long-form composition, sampled/percussive production, and multitrack recording
sessions exercise the complete built-in toolset. DSP fixtures test rate changes,
extreme parameter values, automation continuity, silence, tails, and finite
output. Algorithm quality/performance and bundled-content rights are documented.
External MIDI timing is measured against the declared reference setup.

## P6: Full-Featured DAW Release

### Deliverables

- Qualify Linux ARM64, expanded device configurations, Wayland/X11 plugin editors,
	HiDPI/multiple displays, accessibility paths, and sustained large-session use.
- Grow the native Linux plugin compatibility matrix across instruments, effects,
	bus layouts, state restoration, automation, latency changes, and offline modes.
- Implement/qualify a legacy VST2 adapter only if lawful SDK access and
	distribution are resolved. Otherwise publish the deferral explicitly.
- Complete keyboard/focus and assistive-technology testing; document any remaining
	unsupported paths without claiming universal accessibility.
- Optimize measured CPU/memory bottlenecks, and add parallel graph execution only
	through an architecture decision backed by profiling and determinism tests.
- Finalize user/developer documentation, migration policy, release packaging,
	reproducible example sessions, and regression triage procedures.

### Exit Gate

All unconditional Full requirements in the specification pass, alongside MVP
regression tests. Complete an 8-hour playback/record/save soak, parser/IPC fuzzing,
recovery fault injection, plugin containment tests, and installation tests on the
published support matrix. Release notes distinguish supported, experimental,
conditional/deferred, and out-of-scope capabilities. Conditional VST2 support is
not required to claim this defined release; any other unmet Full requirement
keeps it a candidate until implemented or explicitly re-scoped in the spec.

Full-featured means a reliable, end-to-end DAW across composition, recording,
editing, mixing, plugins, automation, and export. It does not mean every feature
of every future FL Studio release, identical DSP, proprietary content, or project
interchange.

## P7: Maintenance and Future Scope

- Continue performance, compatibility, accessibility, security, and recovery work.
- Maintain schema migrations and a versioned, qualified plugin/device matrix.
- Evaluate additional plugin formats, operating systems, project interchange,
	and collaboration only through separate scope and architecture decisions.
- Add tools based on demonstrated production workflows rather than an undefined
	promise of perpetual feature parity.

## Risks and Decision Gates

| Risk | Required response | Gate |
| --- | --- | --- |
| Legacy VST2 SDK access/distribution | Establish lawful path or defer the adapter | P0 review; P6 decision |
| Worker IPC latency and deadline variance | Measure early; bound buffering and prove fallback/compensation | P0/P1, recheck P3 |
| Native plugin editors on Wayland/X11 | Test real plugins; publish matrix and retain generic controls | P0 spike, P3 baseline, P6 breadth |
| Toolkit accessibility/platform gaps | Qualify early and revise architecture if core needs cannot be met | P0, P3, P6 |
| Save/record corruption | Transactional persistence, recovery fixtures, disk-failure injection | P2/P3 onward |
| Time-stretch/codec licensing or quality | Qualify a library before committing the integration | P5 |
| Unbounded parity expectations | Use this versioned spec as release scope; approve additions explicitly | Every phase |

## Completion Evidence

Every phase closeout MUST record the tested revision, hardware/runtime versions,
automated test results, reproducible manual workflow steps, relevant performance
captures, compatibility limitations, and unresolved issues. A checkbox or a
screenshot alone is not evidence of correct audio behavior or durable saves.

