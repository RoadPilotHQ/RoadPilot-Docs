# RoadPilot-Docs

RoadPilot-Docs is the central documentation for the entire RoadPilot ecosystem. It currently contains an [architecture overview](docs/architecture.md) and this README; more documentation is planned.

> **Status: early.** This repository contains documentation only, no source code. The [architecture overview](docs/architecture.md) describes the other repositories as of Phase 4.3 and links to their authoritative specifications. The user guides, API reference and other items below are **planned**, not existing.

## Purpose

This repository is meant to collect user and developer documentation in one place, so that every RoadPilot component is documented consistently.

## Key responsibilities

- User documentation
- Installation guides
- Developer documentation
- Architecture documentation
- API documentation
- Contribution guidelines
- Troubleshooting guides

## Architecture overview

The following structure is a proposal and may change as development starts:

- **User guides**: installation, usage and troubleshooting.
- **Developer guides**: setting up a development environment and working on each repository.
- **Architecture**: the overall structure of the RoadPilot ecosystem and how the repositories relate.
- **API reference**: documentation of the interfaces between components, once they are defined.
- **Contributing**: shared contribution guidelines for all RoadPilot repositories.

## Integration with the RoadPilot ecosystem

RoadPilot is split into independent repositories. Each one is developed, versioned and released on its own; there is no monorepo and no shared source code.

| Repository | Role |
|---|---|
| [RoadPilot](https://github.com/RoadPilotHQ/RoadPilot) | Main desktop application |
| [RoadPilot-Map](https://github.com/RoadPilotHQ/RoadPilot-Map) | Map processing and geographic data |
| [RoadPilot-Telemetry](https://github.com/RoadPilotHQ/RoadPilot-Telemetry) | Read-only game telemetry plugin and reader |
| [RoadPilot-SDK](https://github.com/RoadPilotHQ/RoadPilot-SDK) | Game input plugin (throttle, brake, steering axes) |
| [RoadPilot-Language](https://github.com/RoadPilotHQ/RoadPilot-Language) | Translations and localization |
| [RoadPilot-Visualization](https://github.com/RoadPilotHQ/RoadPilot-Visualization) | Visualization components and development tools |
| [RoadPilot-Plugin](https://github.com/RoadPilotHQ/RoadPilot-Plugin) | Plugin architecture and SDK |
| [RoadPilot-Website](https://github.com/RoadPilotHQ/RoadPilot-Website) | Official website |
| [RoadPilot-Discord](https://github.com/RoadPilotHQ/RoadPilot-Discord) | Discord community infrastructure |
| **RoadPilot-Docs** (this repository) | Central documentation |

RoadPilot-Docs documents all other RoadPilot repositories. Each repository keeps its own README and its own specifications; cross-cutting documentation lives here and links to them instead of repeating them. See [docs/architecture.md](docs/architecture.md) for responsibilities, the telemetry and input architecture, dependencies between repositories and verification status.

## Development status

| Area | Status |
|---|---|
| Source code | None (documentation only) |
| Architecture | Overview available: [docs/architecture.md](docs/architecture.md) |
| Releases | None |

Development phases of the RoadPilot project:

| Phase | Description | Status |
|---|---|---|
| 4.1 | Telemetry capability audit | Completed |
| 4.2 | Wheel geometry implementation | Completed |
| 4.3 | Steering input support | Completed |
| 4.4 | Not started | Planned |

## Planned roadmap

Planned work, in no fixed order and without dates:

- [ ] Choose the documentation tooling and structure
- [x] Write the ecosystem architecture overview (initial version, [docs/architecture.md](docs/architecture.md))
- [ ] Write the shared contribution guidelines
- [ ] Add installation and user guides once a release exists
- [ ] Add developer and API documentation as interfaces are defined
- [ ] Add troubleshooting guides

## Contributing

RoadPilot is at an early stage and the contribution process is still being defined. Shared guidelines are planned in [RoadPilot-Docs](https://github.com/RoadPilotHQ/RoadPilot-Docs).

Until then, please open an issue or start a discussion before starting any work, so it can be coordinated. All contributions, issues and documentation must be written in English.

## License

Licensing is currently undecided. No license has been selected for this repository yet. Until a license is published here, no permission is granted to use, copy, modify or distribute its contents beyond what the GitHub Terms of Service allow.

---

Euro Truck Simulator 2 and American Truck Simulator are trademarks of SCS Software. RoadPilot is not an official SCS Software product.
