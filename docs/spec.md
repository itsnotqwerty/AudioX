# AudioX Technical Specification

This is the target specification, not an inventory of implemented features. The
current workspace contains documentation only; implementation progress has not
been verified here. Release gates are defined in the [roadmap](roadmap.md), and
implementation rules are defined in the [architecture guide](design.md).

## Scope and Requirement Levels

- **MUST** is a release requirement for the milestone assigned to the feature.
- **SHOULD** is a preferred outcome; exceptions must be documented and justified.
- **MAY** is optional and cannot block that milestone.
- **MVP** means the first complete, testable music-production workflow.
- **Full** means the defined full-featured release, not unlimited FL Studio parity.

AudioX is a Linux-first, independently implemented Rust DAW. Familiarity with FL
Studio informs its aesthetic and production workflow; no proprietary source code,
tooling, bundled content, or project-file compatibility is assumed.

## Platform and Runtime Spec Sheet

| Area | Required direction | Delivery boundary |
| --- | --- | --- |
| Operating system | Linux desktop; x86-64 reference target | MVP; Linux ARM64 after MVP |
| Language | Stable Rust for first-party application and DSP code | From foundation |
| Audio system | Native PipeWire client; no separate ALSA/JACK backend | MVP |
| UI | Native Rust layout using egui/eframe and wgpu; no webview | From foundation |
| Display systems | Wayland and X11 for the application; plugin editors qualified separately | MVP |
| Engine samples | Planar 32-bit floating-point audio | MVP |
| Sample rates | Qualify 44.1, 48, and 96 kHz; follow negotiated device rate | MVP |
| Buffer sizes | Qualify 64, 128, 256, 512, and 1024-frame PipeWire quanta | MVP |
| Channels | Mono and stereo sources, stereo buses and master | MVP; multichannel buses in Full |
| Processing | One shared graph implementation for live playback and offline rendering | MVP |
| Timing | 64-bit sample positions; musical time at 9,600 ticks per quarter note | MVP |
| Plugins | VST family hosting, starting with native Linux VST3 | MVP VST3; conditional legacy VST2 later |
| Project storage | Versioned `.audiox` ZIP container, JSON manifest, managed media | MVP |
| Primary export | Stereo WAV, 24-bit PCM and 32-bit float | MVP |
| Distribution | Reproducible release build, declared runtime dependencies, license inventory | MVP |

The engine MUST size its prepared resources for negotiated parameters before
starting playback. Values outside the tested matrix MUST be reported as
unqualified; unsupported values MUST produce a recoverable error, not a crash.
The qualified Linux distributions, PipeWire versions, and graphics configurations
must be recorded during P0 and published with each release.

## Composition and Arrangement

### MVP

- Pattern rack with variable pattern length, per-step note triggering, velocity,
	mute, and solo; pattern content is reusable across the arrangement.
- Piano roll with note creation, selection, move, resize, delete, duplicate,
	quantize, velocity editing, and grid snapping.
- Playlist with pattern and audio clips, move/trim/split/duplicate operations,
	track ordering, mute, solo, and a visible playhead.
- Explicit linked-pattern editing; making a pattern unique creates an independent
	copy rather than silently changing all instances.
- Play, stop, pause, seek, loop range, metronome, count-in, fixed project tempo,
	and fixed time signature. The default project uses 4/4.
- Computer-keyboard audition and basic PipeWire MIDI input with timestamped note
	and sustain events, live monitoring, recording, and a panic/all-notes-off action.
- Undo and redo for document edits; transport movement and meter updates are not
	document edits.

### Full

- Tempo and time-signature maps, swing, groove templates, advanced quantization,
	scale aids, and per-note expression where instruments support it.
- Arrangement markers, regions, grouped editing, takes, comping, and track folders.
- MIDI file import/export, MIDI output, controller mapping, clock synchronization,
	and measured external-instrument latency compensation.

## Audio Recording and Editing

### MVP

- Import WAV and FLAC with explicit channel/rate validation; decode and resample
	away from the audio callback. Imported source files remain unmodified.
- Select PipeWire inputs and outputs, arm tracks, monitor input, and record mono
	or stereo audio to managed media using a bounded disk-writing queue.
- Show monitoring mode and expected latency; do not silently enable both software
	monitoring and an assumed hardware-monitoring path.
- Non-destructive clip trimming, splitting, fades, and clip gain, with cached
	waveform previews and stable references to source media.
- Detect disk-full, device-loss, and recording-buffer-overflow conditions. Stop
	affected recording safely and retain recoverable captured audio.
- Fixed-rate recordings with documented input/output latency alignment; no silent
	retiming when an audio device changes sample rate.

### Full

- Punch and loop recording, take lanes, comping, crossfades, and batch consolidation.
- Time stretching, pitch shifting, transient detection, slicing, and warping using
	a qualified, license-compatible DSP library rather than a new ad hoc algorithm.
- Additional codecs only when decoder, encoder, and distribution terms are clear.

## Mixing, Routing, and Automation

### MVP

- Per-track gain, pan, mute, solo, metering, ordered insert effects, buses, and
	pre/post-fader sends with explicit stereo pan-law behavior.
- Acyclic audio routing; reject connections that create feedback. Internal
	accumulation MUST remain floating point and MUST NOT hard-clip intermediate
	buses merely because their level exceeds 0 dBFS.
- Plugin delay compensation across parallel paths, including plugin-worker
	buffering; display uncompensated live-input latency separately.
- Automation clips/lanes with read, off, and explicit point editing; step and
	linear interpolation, stable parameter IDs, and documented value ranges.
- Sample-offset scheduling for notes and automation. Plugin delivery accuracy is
	limited by each format and plugin and MUST NOT be overstated.
- Clipping indicators and peak meters. Export MUST NOT silently normalize or add
	a limiter; integer conversion MUST have defined clipping and dither behavior.

### Full

- Sidechain routing, multichannel buses, track freeze/unfreeze, bounce in place,
	and batch stem export with documented shared-effect behavior.
- Touch, latch, and write automation; modulation links with explicit evaluation
	order and no unbounded feedback.
- Loudness and true-peak metering, phase analysis, and advanced routing inspection.

## Instruments and Effects

The MVP MUST include a sample instrument with pitch/envelope controls, a basic
subtractive synthesizer, gain/pan utilities, an equalizer, and a delay. Each device
MUST expose stable automation parameters, serializable presets, and consistent
bypass/reset behavior. Demo assets MUST have documented redistribution rights.

Full adds a drum-oriented sampler, expanded synthesis, a compressor, limiter,
reverb, filtering, distortion, modulation effects, and sample slicing. Built-in
tools need to cover a practical production session, not duplicate proprietary
instruments by name or internal design.

## Plugin Hosting

| Capability | MVP | Full |
| --- | --- | --- |
| Native Linux VST3 instruments and effects | Required | Expanded qualified compatibility matrix |
| Discovery and validation | Separate scanner process, timeout, cache, quarantine | Broader metadata and diagnostics |
| Runtime execution | Separate process per instance | Same isolation contract |
| Controls | Generic parameter editor required | Native editors on qualified configurations |
| Native plugin windows | Best effort on explicitly tested display combinations | Published Wayland/X11 support matrix |
| State | Save/restore opaque state and parameter identity | Preset browsing and migration diagnostics |
| Timing | Notes, transport, automation, latency, tail handling | Additional expression and bus layouts |
| Legacy VST2 | Not required | Conditional on lawful SDK access and distribution review |

VST is a format family, not a third independent ABI alongside VST2 and VST3. Each
actual ABI needs its own adapter. No SDK availability or redistribution permission
is assumed, particularly for legacy VST2. A blocked VST2 adapter MUST be reported
as deferred; it does not block the rest of the full-featured release.

Windows/macOS plugin binaries, Wine bridges, AU, AAX, CLAP, and LV2 are outside the
committed scope. No automatic loading of plugins from project-supplied executable
paths is allowed. Missing plugins MUST retain state and routing in placeholders.
Process isolation is fault containment, not a security sandbox: installed plugins
remain native code that the user must trust.

## Projects and Persistence

- A project stores stable IDs, tracks, patterns, clips, routing, automation,
	transport settings, plugin state, and media metadata in a versioned schema.
- Saves MUST be transactional. An interrupted save MUST leave the previous good
	project available; recent recovery snapshots MUST be discoverable on restart.
- MVP saves MUST collect imported/recorded media into the project for portability.
	Source paths can be retained as metadata but cannot be required for playback.
- Recovery MUST preserve finalized takes and recoverable portions of interrupted
	recordings; autosave MUST NOT block the audio thread.
- Missing or corrupt media MUST produce a repairable placeholder, not silently
	disappear. Future project versions MUST be rejected without overwriting them.
- UI layout may be stored separately; resetting the UI MUST NOT alter musical
	content. Plugin binaries MUST NOT be embedded in project archives.

## Export and Render Correctness

MVP MUST export a selected range or the full arrangement to WAV at the project
sample rate, with explicit output format, tail duration/limit, destination,
overwrite confirmation, cancellation, and actionable failures. PCM export MUST
offer documented dither; floating-point export MUST preserve valid headroom.

Offline rendering uses the same routing, timing, automation, and compensation as
playback. Deterministic built-in-only fixtures MUST match live-engine captures
within a declared numerical tolerance after latency alignment. Random sources
must be seeded for these tests. Third-party plugins can be nondeterministic or
require real-time rendering; support a real-time export fallback and disclose
that distinction rather than promising bit-identical results for all plugins.

## Quality and Acceptance Targets

These are gates to measure, not benchmark claims. P0 MUST define the reference
hardware, operating system, versions, workload fixtures, and measurement tools.

| Area | Release acceptance target |
| --- | --- |
| Core session | Create a pattern and synth part, record audio, mix, automate, save, reopen, and export without data loss |
| Reference workload | 32 stereo tracks, 8 active built-in instruments, 32 built-in effects, and 100 active automation lanes at 48 kHz/256 frames |
| Real-time stability | 30-minute reference run with zero engine deadline misses; record PipeWire xruns separately and investigate them |
| Callback budget | On the reference workload, p99 callback time below 70% of the quantum duration; every deadline miss reported |
| Interactive performance | Target 60 Hz on reference display; p95 input-to-visible-update under 50 ms while reference playback runs |
| Timing | Built-in note and automation fixture boundaries within one sample of the scheduled position |
| Recovery | Fault-injection tests for interrupted save, full disk, plugin crash/hang, and interrupted recording |
| Plugin containment | A killed or hung plugin MUST NOT terminate the application or stall the audio callback |
| Round-trip | Project reopen preserves document data, media identity, automation, routing, and restorable plugin state |
| Soak | Full release: 8-hour mixed playback/record/save run without crash, unbounded memory growth, or unexplained audio gaps |

End-to-end latency MUST be measured and reported with device, sample rate, quantum,
adapter buffering, plugin latency, and monitoring configuration. A 64-frame buffer
alone is not a round-trip latency guarantee. Third-party plugin stress tests are
reported separately from the reproducible built-in reference workload.

## Usability and Release Readiness

- Native, resizable workspaces with pattern rack, piano roll, arrangement, mixer,
	browser, transport, and consistent selection/focus behavior.
- Keyboard access to core workflows, configurable shortcuts, readable scaling,
	visible focus, and no information conveyed only by color.
- Accessibility integration MUST be evaluated in P0 and tested on supported Linux
	configurations; unsupported assistive-technology paths must be documented.
- No mandatory account, cloud connection, or network access to create and export
	music. No telemetry by default; diagnostic uploads require explicit consent.
- Publish installation steps, known limitations, tested configurations, project
	compatibility policy, and license notices before distributing the MVP.

