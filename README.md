# S85 PLC

TwinCAT 3 PLC software for the S85 home automation system.

Core contains shared, environment-independent home logic. LOCAL provides simulation and TcUnit tests on x64. PROD connects to physical Beckhoff I/O on ARMv8.

- [TwinCAT XAE workflow](docs/twincat-xae.md): project structure, builds and toolchain guidance.
- [Workshop setup and checks](docs/workshop-light.md): Workshop, Entryway and Hall wiring, LOCAL tests and commissioning.
- [Adding a light](docs/adding-light.md): Core configuration and LOCAL and PROD adapters.
- [MQTT light integration](docs/mqtt-light.md): broker configuration, topics and commissioning.
- [Agent guidance](AGENTS.md): project boundaries and verification rules.
