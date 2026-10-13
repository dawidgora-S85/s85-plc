# Workshop setup and checks

The application controls one Workshop button and one light relay without MQTT.

## Build modes

- The default compiler define is `WORKSHOP_HW`.
- Define exactly one of `WORKSHOP_HW` or `WORKSHOP_SIM`. Both or neither produce a compile error.
- To simulate, replace `WORKSHOP_HW` with `WORKSHOP_SIM` in the active configuration.
- Watch `GVL_WorkshopSimulation.bButton` and `GVL_WorkshopSimulation.bLightRelay` in TwinCAT.
- The legacy dummy-room visualization is not active.

## Hardware mappings

Hardware mode uses `GVL_WorkshopIO.bButton` at `%I*` and `GVL_WorkshopIO.bLightRelay` at `%Q*`.
`S85.tsproj` saves these mappings under Device 1 (EtherCAT), Term 1 (EK1200):

- Button: Term 3 (ED1809), DI Channel 1, Input.
- Light relay: Term 4 (ED2809), DO Channel 1, Output.

## Before activation

- Confirm these channel assignments against the actual wiring.
- Confirm the intended XAR target before connecting or activating the configuration.
- Do not treat the target saved in the project as proof of controller identity.
- Verify button and relay polarities and the safe output state against the wiring and hardware documentation.
- The application expects a normally-low button that reads TRUE when pressed and an active-high relay command.
- If input polarity differs, invert the mapped input before passing it to the application.

## Button behavior

At startup, the relay command is OFF.
The application ignores a button already reading TRUE until release and a new press.
Each new press toggles the relay; holding or releasing the button leaves the relay unchanged.

## Verification

Use the [TwinCAT XAE workflow](twincat-xae.md) for solution prerequisites, the verified build route and toolchain limitations.

Before merging logic changes, build HW and SIM modes and verify that invalid define combinations fail.
Run TcUnit only in an explicitly configured, isolated test environment, never on the production CX.
Record actual results in the PR; unexecuted tests are not a pass.

`FB_WorkshopTests` is registered with the existing TcUnit test program.
`PlcTask` schedules only `MAIN`; `PRG_Tests` and the TcUnit reference remain in the project but tests no longer run automatically.
A separate test execution setup is required and is not provided here.

Check manually before deployment:

- Start with input FALSE and confirm output OFF.
- Set input TRUE and confirm output ON.
- Hold TRUE for several PLC cycles and confirm output stays ON.
- Release, then press again, and confirm output OFF.
- Restart with input TRUE and confirm output stays OFF until release and a new press.
