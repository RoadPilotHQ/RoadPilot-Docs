# RoadPilot architecture overview

This page describes how the RoadPilot repositories fit together as of **Phase 4.4**. It is an overview only. The authoritative specifications live in the source repositories and are linked below; they are not repeated here. Where this page and a source repository disagree, the source repository is correct.

## Repository responsibilities

| Repository | Responsibility | State |
|---|---|---|
| [RoadPilot](https://github.com/RoadPilotHQ/RoadPilot) | Desktop application (Tauri, React): dashboard, telemetry engine, calculated values, visualization bridge, supervised control pipeline | Early prototype; never controls the game (control pipeline in dry run only) |
| [RoadPilot-Telemetry](https://github.com/RoadPilotHQ/RoadPilot-Telemetry) | Read-only game telemetry: native plugin (SCS Telemetry SDK) and Rust reader | Implemented (schema 2) |
| [RoadPilot-SDK](https://github.com/RoadPilotHQ/RoadPilot-SDK) | Input plugin (SCS Input SDK): throttle, brake and steering axes fed from a local pipe | Prototype, protocol v1 and v2 |
| [RoadPilot-Visualization](https://github.com/RoadPilotHQ/RoadPilot-Visualization) | Read-only visualization components and development tools | See its README |
| [RoadPilot-Map](https://github.com/RoadPilotHQ/RoadPilot-Map) | Offline map data processing (Rust library and CLI) | Early development |
| [RoadPilot-Language](https://github.com/RoadPilotHQ/RoadPilot-Language) | Translation bundles | See its README |
| [RoadPilot-Plugin](https://github.com/RoadPilotHQ/RoadPilot-Plugin) | Plugin architecture and SDK | See its README |
| [RoadPilot-Website](https://github.com/RoadPilotHQ/RoadPilot-Website) | Official website | See its README |
| [RoadPilot-Discord](https://github.com/RoadPilotHQ/RoadPilot-Discord) | Discord community infrastructure | See its README |
| **RoadPilot-Docs** (this repository) | Central documentation | This page and the README |

Each repository is developed, versioned and released on its own. There is no monorepo and no shared source code.

## Telemetry and input architecture

The SCS SDK has two separate parts. RoadPilot uses them in two separate plugins that share no code, pipe or protocol.

```
                  game (ETS2 / ATS)
                   |            ^
   Telemetry SDK   |            |   Input SDK
   (read-only)     v            |   (generic input device,
                                |    axes bound by the user in game)
  roadpilot-telemetry.dll   roadpilot-input.dll
   \\.\pipe\RoadPilot.        \\.\pipe\roadpilot-input-v1
   Telemetry.v1                   ^
        |                         |
        v                         |
  RoadPilot app  ----------X----- pipe never opened:
   (Rust client, engine,          dry run only
    calculated values,
    control pipeline)
        |  loopback WebSocket 127.0.0.1:17891 (read-only)
        v
  RoadPilot-Visualization
```

### Telemetry (read-only)

- The plugin in RoadPilot-Telemetry reads SDK channels and configuration events and writes schema 2 JSON frames over a persistent named pipe. The desktop side hosts the pipe, decodes, validates and orders the frames. Specification: [PROTOCOL-V2.md](https://github.com/RoadPilotHQ/RoadPilot-Telemetry/blob/main/docs/PROTOCOL-V2.md), design: [ARCHITECTURE.md](https://github.com/RoadPilotHQ/RoadPilot-Telemetry/blob/main/docs/ARCHITECTURE.md).
- The RoadPilot application normalizes units and axes, calculates derived values and samples at 10 Hz for the dashboard. See [docs/telemetry.md](https://github.com/RoadPilotHQ/RoadPilot/blob/main/docs/telemetry.md).
- Telemetry provides no control. The steering, throttle and brake values in the frames are readings of the game state.
- The SCS SDK provides no road, lane, traffic, weather or planned-route geometry; those are not telemetry fields.

### Input (RoadPilot-SDK)

- The plugin registers one generic input device, "RoadPilot Input", with three float axes: throttle (index 0), brake (1) and steering (2). The user binds the axes manually in the game controls menu. Binding instructions: [RoadPilot-SDK README](https://github.com/RoadPilotHQ/RoadPilot-SDK#manual-axis-assignment-in-game).
- The pipe protocol has two versions. **v1** (16 bytes) carries throttle and brake. **v2** (20 bytes) adds a steering value and a flags byte. v1 clients remain supported and can never activate steering. Authoritative specification: [PROTOCOL.md](https://github.com/RoadPilotHQ/RoadPilot-SDK/blob/main/docs/PROTOCOL.md).
- **No RoadPilot component sends commands to the game.** The RoadPilot application encodes protocol v2 messages in dry run (see [Control pipeline](#control-pipeline-roadpilot-dry-run)) and never opens the pipe. There is no steering or speed controller.

### Wheel geometry and wheelbase

- The telemetry plugin reads the truck's wheel configuration from the SDK (`truckConfig.wheelCount`, `hookPositionM`, `wheels[]` with position, radius, steerable and powered flags; at most 14 wheels) and sends it unchanged in SDK vehicle space (X right, Y up, Z backwards).
- The wheelbase is calculated only in the RoadPilot application, never in the plugin. It is absent when the wheel data is incomplete or implausible. Details: [docs/telemetry.md](https://github.com/RoadPilotHQ/RoadPilot/blob/main/docs/telemetry.md#wheel-geometry-and-wheelbase).
- The SDK has no channel for overall vehicle length or width.

### Steering input safety mechanisms

Summary of what is implemented in RoadPilot-SDK. Exact values and rules are specified in [PROTOCOL.md](https://github.com/RoadPilotHQ/RoadPilot-SDK/blob/main/docs/PROTOCOL.md#version-2-steering-additive).

- Steering is disarmed by default and never armed automatically. It requires both the general `enabled` flag and the dedicated `steering_enabled` flag.
- The value is clamped to -1.0..1.0. Its magnitude can grow by at most 2.0 units per second; reductions are immediate and a direction change goes through zero.
- Steering is released to neutral after 150 ms without a valid message; throttle and brake use a separate 250 ms timeout.
- Invalid messages, disconnects, an explicit disarm and device deactivation return to a safe state (all values 0, steering not active). Neutral values are reported every frame.
- A sender must refresh commands at least every 50 ms, independently of the application's 10 Hz telemetry engine. The dry-run sender in RoadPilot runs on its own 40 ms timer, but on Windows the measured intervals reach 55 ms, so this **requirement is not yet met reliably** and must be resolved before any live transmission.
- Manual control is **not guaranteed** while the RoadPilot steering axis is bound. How the game combines several devices bound to one function is unverified.

## Control pipeline (RoadPilot, dry run)

Phase 4.4 added a supervised control foundation to the RoadPilot application. It runs in **dry run only**: commands are generated, bounded and encoded in the exact protocol v2 format, then logged and discarded. There is no setting, flag or command that enables transmission; that requires a separate, explicitly authorized phase. Authoritative description: [docs/control.md](https://github.com/RoadPilotHQ/RoadPilot/blob/main/docs/control.md).

```
telemetry engine (one pipe server) --watch channel--> dashboard (10 Hz)
                                                  '--> control runtime (every frame)
  freshness monitor -> safety supervisor -> command generator (limits, rate limit)
  -> protocol v2 encoder -> dry-run sink (discards) ; bounded log and status -> UI
```

- **Telemetry input:** control reads the raw schema 2 frames through a second receiver of the telemetry engine's own channel. There is exactly one telemetry pipe server.
- **Freshness:** data older than 150 ms, paused, invalid or unsettled is unusable. Every discontinuity (reconnect gap, session or game change, pause, invalid frame) starts a new vehicle epoch.
- **Supervisor:** Disabled, Armed, Active, Fault. Arming is manual, needs fresh telemetry within a 5 m/s test speed cap and binds to the current epoch. Any safety event disarms Armed and faults Active; nothing re-arms automatically, a fault needs an explicit reset and a new arm, and Disarm is always accepted and processed first.
- **Commands:** steering ±0.15 with a 0.5 per second rate limit, throttle 0.10, brake 1.0; brake overrides throttle; non-finite or out-of-limit values cannot be constructed.
- **Sender:** a 40 ms monotonic schedule that skips missed deadlines, an independent watchdog, and a single disarm after any fault, manual disarm, shutdown or task abort.
- **Application:** a Control section in the dashboard shows the state, Dry Run mode, freshness, safety events, timing, watchdog and recent decisions, with Arm, Activate, Reset Fault and an always-available Disarm. Every output is marked "Not transmitted (Dry Run)".

## Dependencies between repositories

| From | To | Kind |
|---|---|---|
| RoadPilot | RoadPilot-Telemetry | Rust client crate, pinned to a Git revision in `Cargo.toml`; the shared test fixture is copied together with the revision. The dashboard and the control pipeline read the same frame channel; there is one pipe server |
| RoadPilot | RoadPilot-SDK | No code or runtime link. RoadPilot encodes the input protocol v2, tested byte for byte against the SDK's test vector, but never opens the input pipe (dry run) |
| RoadPilot | RoadPilot-Visualization | Loopback WebSocket stream (`roadpilot-visualization-frame` schema 2), read-only; no code dependency |
| RoadPilot-SDK | other RoadPilot repositories | None. It builds against the SCS SDK 1.16 headers (not stored in the repository) |
| RoadPilot-Telemetry plugin | SCS SDK 1.16 | Build time only, headers not stored |
| RoadPilot-Map | RoadPilot-Telemetry | Optional: its validation tooling can import frame snapshot lines produced by the Telemetry console reader; no runtime link |
| RoadPilot | RoadPilot-Map | No runtime integration yet |

## Development phases

| Phase | Description | Status |
|---|---|---|
| 4.1 | Telemetry capability audit | Completed |
| 4.2 | Wheel geometry implementation | Completed |
| 4.3 | Steering input support | Completed |
| 4.4 | Control foundation in dry run (RoadPilot) | Completed |
| 4.5 | Not started | Planned |

## Verification status

- **ATS:** the schema 1 telemetry plugin (speed, RPM, gear) was live-tested in ATS 1.62.x. The schema 2 fields, including wheel geometry, have not been tested in a running game.
- **ETS2:** telemetry is implemented through the same code path but has **not been live-tested**.
- **Input plugin:** automated tests only (including a fake-host run of the real DLL over the real pipe). Whether the game accepts the axes, the steering range and sign, and the interaction with manual controls are **unverified** in ATS and ETS2.
- **Control pipeline (RoadPilot):** covered by automated tests (unit, virtual-time, real-time lifecycle and Tauri IPC tests) and checked in the running desktop application without a game (rendering, a refused Arm through the real command path, orderly shutdown on exit). **Not verified in ATS or ETS2 gameplay.** Thresholds are not calibrated against live gameplay, and manual override is unverified. Details and outstanding risks: [docs/control.md](https://github.com/RoadPilotHQ/RoadPilot/blob/main/docs/control.md#verification-status).
- Automated tests cover protocols, conversions and state handling; they do not prove that the game delivers or applies values.

## Not implemented

- Live transmission to the input plugin (the RoadPilot control pipeline runs in dry run only).
- Any steering or speed controller, or autonomous driving logic.
- Single-instance protection across separate RoadPilot application processes.
- Map-based route, lane or road geometry in the application.
