# Agent guidance

- Keep `README.md` short: project purpose and links to relevant documentation.
- Put agent instructions in `AGENTS.md`, not in `README.md`.
- Read only the documents relevant to the task. Follow their links when you need more detail.
- Keep changes narrow. Do not change hardware, I/O mappings or target settings without explicit authorization.
- Record task-specific progress and verification results in the PR, not in project documentation.

## PLC project structure

- `S85.PROD/S85.PROD.tsproj` owns production system composition, task scheduling, EtherCAT configuration and I/O mappings.
- `S85.LOCAL/S85.LOCAL.tsproj` owns the local system and its simulation.
- `S85.Core/S85.Core.tspproj` wraps the shared PLC library in `S85.Core/S85_Core.plcproj`.
- Core owns shared, environment-independent home logic and contracts. `Application` contains the home application and its DUTs and POUs. `Locations/Workshop`, `Locations/Entryway` and `Locations/Hall` contain their location adapters. `Domains/Lighting` owns neutral shared light behavior.
- `FB_ButtonLight` in `Domains/Lighting/POUs` provides shared button-toggle and light-command behavior; location adapters compose it for their circuits.
- Both applications reference Core through Library Manager. PROD adapters connect hardware; LOCAL adapters connect simulation.
- LOCAL uses x64 and keeps simulation under `Infrastructure/IO/Simulation/{POUs,GVLs}` and tests under `Tests`.
- PROD uses ARMv8 and keeps physical Beckhoff I/O under `Infrastructure/IO/Beckhoff/{POUs,GVLs}`. It contains no simulation or TcUnit tests.
- Production contains no test code or environment-switching compiler defines.
- Keep the physical project roots `S85.Core/`, `S85.LOCAL/S85.LOCAL/` and `S85.PROD/S85/`. The system project folders contain their related system configuration.

## Documentation index

- [TwinCAT XAE workflow](docs/twincat-xae.md): read before starting XAE, building PLC code or investigating toolchain failures. This is the canonical build procedure.
- [Workshop setup and checks](docs/workshop-light.md): read for hardware mappings, LOCAL simulation, tests and commissioning checks.
- [Adding a light](docs/adding-light.md): Core configuration and LOCAL and PROD adapters.
- [MQTT light integration](docs/mqtt-light.md): broker configuration, topics and commissioning.

## Verification

- Follow the [TwinCAT XAE workflow](docs/twincat-xae.md) for scoped clean rebuilds, configurations and required evidence. Build only the affected system project, not the whole solution.
- Run simulation and TcUnit tests only in LOCAL. Never include tests or simulation in the production project.
- Source review alone is not compilation evidence. Report skipped verification explicitly.
- Every LOCAL or PROD online operation requires explicit user authorization for the target and operation. This includes Login, Download, Start, Stop, Logout and online reads.
- Never deploy automatically to PROD. Activation, PLC download and runtime changes require explicit user authorization for the target and operation.
