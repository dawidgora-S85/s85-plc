# Agent guidance

- Keep `README.md` short: project purpose and links to relevant documentation.
- Put agent instructions in `AGENTS.md`, not in `README.md`.
- Read only the documents relevant to the task. Follow their links when you need more detail.
- Keep changes narrow. Do not change hardware, I/O mappings or target settings without explicit authorization.
- Record task-specific progress and verification results in the PR, not in project documentation.

## PLC project structure

- `S85.sln` contains the `S85.PROD`, `S85.LOCAL` and `S85.Core` system projects.
- `S85.PROD/S85.PROD.tsproj` owns production system composition, task scheduling, EtherCAT configuration and I/O mappings.
- The production system uses `S85.PROD/S85/S85_PROD.plcproj`, PLC application `S85_PROD`, instance `S85_PROD Instance` and TMC `S85.PROD/S85/S85_PROD.tmc`.
- `S85.LOCAL/S85.LOCAL.tsproj` owns the local system. It uses `S85.LOCAL/S85.LOCAL/S85_LOCAL.plcproj`, PLC application `S85_LOCAL` and instance `S85_LOCAL Instance`.
- `S85.Core/S85.Core.tspproj` wraps the shared PLC library in `S85.Core/S85_Core.plcproj`.
- Core owns shared, environment-independent home logic and contracts. `Application` contains the home application and its DUTs and POUs. `Locations/Workshop`, `Locations/Entryway` and `Locations/Hall` contain their location adapters. `Domains/Lighting` owns neutral shared light behavior.
- `FB_ButtonLight` in `Domains/Lighting/POUs` provides shared button-toggle and light-command behavior; location adapters compose it for their circuits.
- Both applications reference Core through Library Manager. PROD adapters connect hardware; LOCAL adapters connect simulation.
- LOCAL uses x64 and keeps simulation under `Infrastructure/IO/Simulation/{POUs,GVLs}` and tests under `Tests`.
- PROD uses ARMv8 and keeps physical Beckhoff I/O under `Infrastructure/IO/Beckhoff/{POUs,GVLs}`. It contains no simulation or TcUnit tests.
- Production contains no test code or environment-switching compiler defines.
- Keep the physical project roots `S85.Core/`, `S85.LOCAL/S85.LOCAL/` and `S85.PROD/S85/`. The system project folders contain their related system configuration.

## Documentation index

- [TwinCAT XAE workflow](docs/twincat-xae.md): read before starting XAE, building PLC code or investigating toolchain failures.
- [Workshop setup and checks](docs/workshop-light.md): read for hardware mappings, LOCAL simulation, tests and commissioning checks.

## Verification

- Build only the affected system project. Do not use a solution-wide build for PROD or LOCAL verification.
- For a clean rebuild, select the system project root in Solution Explorer and use native `Build.RebuildSelection`.
- Use `Release | TwinCAT OS (ARMV8-A)` for PROD and the existing `Release | TwinCAT OS (x64)` configuration for LOCAL.
- Confirm that Output reports `Rebuild All` for the selected project. Record `LastBuildInfo`, errors, warnings and fresh artifacts.
- Run simulation and TcUnit tests only in LOCAL. Never include tests or simulation in the production project.
- Source review alone is not compilation evidence. Report skipped verification explicitly.
- Never deploy automatically to PROD. Activation, PLC download and runtime changes require explicit user authorization for the target and operation.
