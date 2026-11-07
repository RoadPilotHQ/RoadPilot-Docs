# RoadPilot architecture overview

This page describes how the RoadPilot repositories fit together as of **Phase 4.3**. It is an overview only. The authoritative specifications live in the source repositories and are linked below; they are not repeated here. Where this page and a source repository disagree, the source repository is correct.

## Repository responsibilities

| Repository | Responsibility | State |
|---|---|---|
| [RoadPilot](https://github.com/RoadPilotHQ/RoadPilot) | Desktop application (Tauri, React): dashboard, telemetry engine, calculated values, visualization bridge | Early prototype; never controls the game |
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
  RoadPilot app  ----------X----- no sender yet
   (Rust client, engine,
    calculated values)
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
- **No RoadPilot component sends commands yet**, and there is no steering activation UI and no steering controller.

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
- A future sender must refresh commands at least every 50 ms, independently of the application's 10 Hz telemetry engine. This is a **requirement**, not an existing capability.
- Manual control is **not guaranteed** while the RoadPilot steering axis is bound. How the game combines several devices bound to one function is unverified.

## Dependencies between repositories

| From | To | Kind |
|---|---|---|
| RoadPilot | RoadPilot-Telemetry | Rust client crate, pinned to a Git revision in `Cargo.toml`; the shared test fixture is copied together with the revision |
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
| 4.4 | Not started | Planned |

## Verification status

- **ATS:** the schema 1 telemetry plugin (speed, RPM, gear) was live-tested in ATS 1.62.x. The schema 2 fields, including wheel geometry, have not been tested in a running game.
- **ETS2:** telemetry is implemented through the same code path but has **not been live-tested**.
- **Input plugin:** automated tests only (including a fake-host run of the real DLL over the real pipe). Whether the game accepts the axes, the steering range and sign, and the interaction with manual controls are **unverified** in ATS and ETS2.
- Automated tests cover protocols, conversions and state handling; they do not prove that the game delivers or applies values.

## Not implemented

- A RoadPilot input sender and a steering activation UI.
- Any steering controller or autonomous driving logic.
- Map-based route, lane or road geometry in the application.
