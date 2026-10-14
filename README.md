# AudioX

AudioX is a Linux-first digital audio workstation (DAW) written in Rust, built for hobbyist and professional music production. It aims to combine a familiar, pattern-based production workflow with a native Linux experience.

## Status

AudioX is in development. This workspace currently contains planning documentation, not a runnable application, and no roadmap phase is recorded as complete. The capabilities below describe development requirements, not features available today. Build and installation instructions will be added when a runnable version is ready to share.

## Vision

AudioX targets the breadth of music-production features and plugin formats found in recent FL Studio releases, with familiar workflows, layout, and interactions. The goal is to support the full creative process: sketching patterns, arranging tracks, recording audio, shaping sounds, and mixing a project.

This is a long-term direction, not a current guarantee of feature parity or compatibility with every plugin. FL Studio project-file compatibility is not a stated goal.

## Initial Priorities

- **Composition and arrangement:** pattern sequencing, a piano roll, and a timeline for building complete tracks.
- **Audio production:** recording, audio editing, and mixing.
- **Plugins and automation:** hosting instruments and effects, with automation for evolving sounds and mixes.
- **Built-in tools:** instruments, effects, and sample tools for making music within AudioX.

## Technical Direction

| Area | Target |
| --- | --- |
| Platform | Linux-first; x86-64 MVP, Linux ARM64 qualification later |
| Language | Stable Rust with narrow, audited FFI boundaries |
| Audio and MIDI | Native PipeWire integration |
| User interface | Native Rust layout using egui/eframe with wgpu; no webview |
| Audio engine | Planar 32-bit float, sample-offset event scheduling, prepared acyclic routing |
| Device qualification | 44.1/48/96 kHz and 64-1024-frame quanta on published reference configurations |
| Plugin hosting | Native Linux VST3 for MVP; legacy VST2 conditional on lawful SDK access |
| Plugin isolation | Separate discovery process and one worker process per active instance |
| Projects | Versioned `.audiox` archives with JSON metadata, managed media, and plugin state |
| Export | Shared live/offline engine; MVP stereo WAV in 24-bit PCM or 32-bit float |
| Operation | Local and offline-capable, with no mandatory account or telemetry |

VST names a format family; VST2 and VST3 require separate host adapters. Format support does not imply that every plugin binary will run on Linux. Windows/macOS ports, cross-platform plugin bridges, and FL Studio project import are outside the committed scope. Native plugin editors will have a tested display-system compatibility matrix; generic parameter controls are required for MVP.

## Specification and Architecture

The detailed documents define what to build, how it must be built, and what evidence is needed before a release:

- [Technical specification](docs/spec.md): platform spec sheet, MVP and full-release capabilities, timing, recording, mixing, plugins, persistence, export, and measurable acceptance targets.
- [Architectural design](docs/design.md): mandatory technology choices, module ownership, thread/process boundaries, real-time restrictions, graph execution, plugin IPC, transactional saves, and test rules.
- [Delivery roadmap](docs/roadmap.md): dependency-ordered phases from an empty workspace through MVP to a full-featured DAW, with deliverables, exit gates, and risks.

The architecture sets strict boundaries: the UI cannot own audio timing; the audio callback cannot allocate, lock, perform I/O, or wait for plugins; queues and buffers must have bounded capacities; and project saves must preserve the previous good file on interruption. Live playback and offline export must use the same processing graph. Changes to these rules require a documented architecture decision.

The reference performance target is a 32-track session at 48 kHz/256 frames with 8 active built-in instruments, 32 built-in effects, and 100 active automation lanes. MVP requires a 30-minute run without engine deadline misses on declared reference hardware. These are targets to validate, not current benchmark results or latency guarantees.

## Zero to Full DAW

| Phase | Focus | Completion outcome |
| --- | --- | --- |
| P0 | Workspace, licensing, UI/audio/plugin feasibility, reference hardware | Validated foundation |
| P1 | Headless engine, timing, routing, latency compensation, render tests | Reliable engine prototype |
| P2 | Patterns, piano roll, arrangement, instruments, project persistence | Internal composition alpha |
| P3 | Recording, VST3, automation, mixing, recovery, packaging | MVP |
| P4 | Tempo maps, comping, sidechains, freeze, stems, deeper automation | Production beta |
| P5 | Stretching, warping, expanded DSP, advanced MIDI, analysis | Feature-complete candidate |
| P6 | Compatibility, accessibility, ARM64, soak tests, release qualification | Full-featured DAW |
| P7 | Maintenance and separately approved additions | Continuing releases |

**MVP means a complete session:** program patterns and notes, arrange clips, record audio, use built-in and VST3 instruments/effects, mix and automate, save and reopen without losing state, and export a playable WAV. Plugin failures must not crash the application or block its audio callback.

**Full-featured means production depth:** advanced recording/editing, tempo changes, sidechains, multichannel buses, freeze and stem workflows, expanded instruments/effects, advanced MIDI, and qualified long-session reliability. It does not promise identical DSP, proprietary content, project interchange, or parity with every future FL Studio release. Legacy VST2 remains conditional and does not block the rest of this release.

Phases are acceptance-gated, not calendar promises. See the roadmap for exact release checks and deferred features.

## Contributing

AudioX is taking shape, and input from both developers and musicians can help guide it. Useful areas include Rust development, Linux audio integration, plugin hosting, interface design, and practical production workflows.

When proposing a feature or change, identify the workflow it supports, its roadmap phase, the affected architectural boundaries, and how it will be tested. Licensing and contribution procedures still need to be established in P0 before distribution; no open-source license is implied by this README.

## Relationship to FL Studio

AudioX is an independent implementation. Resemblance to FL Studio is aesthetic; the project has no access to internal Fruity Loops / FL Studio code or tooling. References to FL Studio describe design inspiration and feature goals, not shared implementation or official compatibility.
