# Agent guidance

- Keep `README.md` short: project purpose and links to relevant documentation.
- Put agent instructions in `AGENTS.md`, not in `README.md`.
- Read only the documents relevant to the task. Follow their links when you need more detail.
- Keep changes narrow. Do not change hardware, I/O mappings or target settings without explicit authorization.
- Record task-specific progress and verification results in the PR, not in project documentation.

## PLC project structure

- `S85.tsproj` owns system composition, task scheduling, EtherCAT configuration and I/O mappings.
- `S85/S85.plcproj` owns `MAIN`, application integration, simulation and test code.
- `S85.Core/S85.Core.tspproj` wraps the standalone PLC library in `S85.Core/S85_Core.plcproj`.
- The Core library owns environment-independent application logic and contracts, including `FB_Home`, the Workshop and Entryway location blocks and contracts, `FB_Workshop`, `FB_Light` and `FB_Button`. Consumers reference it through Library Manager; hardware, simulation and tests stay in the application project.

## Documentation index

- [TwinCAT XAE workflow](docs/twincat-xae.md): read before starting XAE, building PLC code or investigating toolchain failures.
- [Workshop setup and checks](docs/workshop-light.md): read for Workshop logic, HW/SIM defines, I/O mappings, tests or commissioning checks.

## Verification

- Compile all relevant configurations before treating PLC changes as verified. Source review alone is not compilation evidence.
- For Workshop changes, follow the HW/SIM build and isolated TcUnit requirements in its document. Report missing evidence explicitly.
- Never deploy automatically to PROD. Activation, PLC download and runtime changes require explicit user authorization for the target and operation.
