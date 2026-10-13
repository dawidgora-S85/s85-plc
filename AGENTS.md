# Agent guidance

- Keep `README.md` short: project purpose and links to relevant documentation.
- Put agent instructions in `AGENTS.md`, not in `README.md`.
- Keep this file concise. Split detailed guidance into separate Markdown files and link them here.
- Record task-specific progress and verification results in the PR, not in project documentation.

## Documentation index

- [Workshop setup and checks](docs/workshop-light.md): compiler defines, I/O mappings, safety checks and tests.

## Verification

- For Workshop changes, check TwinCAT HW/SIM builds and run TcUnit. Report missing evidence explicitly.
- Do not activate a controller configuration or download PLC code without explicit user authorization.
