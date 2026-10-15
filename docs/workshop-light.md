# Workshop setup and checks

The production application uses hardware inputs and outputs for the Workshop and Entryway.
The LOCAL application provides simulation and TcUnit tests.

## Project split

- PROD uses `S85.PROD.tsproj` and `S85/S85_PROD.plcproj`.
- Its PLC application and instance are `S85_PROD` and `S85_PROD Instance`.
- PROD `MAIN` always reads hardware inputs, calls `FB_Home` and writes hardware outputs.
- PROD has no simulation, tests, TcUnit or compiler defines.
- LOCAL uses `Local/S85.LOCAL.tsproj` and `Local/S85.LOCAL/S85_LOCAL.plcproj`.
- Its PLC application is `S85_LOCAL`, with instance `S85_LOCAL Instance`.
- LOCAL contains simulation adapters and TcUnit tests. Both systems use shared Core logic from `S85.Core`.

## Scoped builds and tests

- Build PROD with `Release | TwinCAT OS (ARMV8-A)`.
- Build LOCAL with the existing `Release | TwinCAT OS (x64)` configuration.
- Select only the required system project root in Solution Explorer.
- Use `Build.RebuildSelection` for a native scoped clean rebuild.
- Do not build the whole solution to verify PROD or LOCAL.
- Run simulation and TcUnit tests only in LOCAL, never on the production CX.
- Do not add PROD compiler defines to switch hardware or simulation behavior.

## LOCAL runtime and tests

### Start the existing runtime

Start the existing UmRT instance manually after the Windows VM starts.
The verified launcher is `C:\ProgramData\Beckhoff\TwinCAT\3.1\Runtimes\UmRT_Default\Start.bat`.
From Command Prompt, run:

```bat
set "TWINCAT3DIR=C:\Program Files (x86)\Beckhoff\TwinCAT\3.1\"
cd /d "C:\ProgramData\Beckhoff\TwinCAT\3.1\Runtimes\UmRT_Default"
Start.bat
```

The existing batch starts `TcSystemServiceUm.exe` with `/min`; this minimizes its window but does not hide it.
Do not add a Windows service, scheduled task, wrapper or autostart entry.
Starting UmRT alone does not confirm that `S85_LOCAL` is loaded or running.
Check the application identity and state before further actions.
Do not assume a Boot Project restores the application after an UmRT or VM restart.
Application restore after restart and automatic UmRT startup are not verified.

### Confirm LOCAL context before online work

Every LOCAL online operation needs separate, explicit user authorization.
This includes Login, Download, Start, Stop, Logout and online symbol reads.
Before an authorized operation, confirm the native system project is `S85.LOCAL`.
Confirm the System Manager PLC node is `TIPC^S85_LOCAL` and the active PLC project is `S85_LOCAL`.
Confirm the selected target is `UmRT_Default`.
Confirm the target AMS Net ID is `199.4.42.250.1.1` and the PLC port is `851`.
Confirm the LOCAL project's platform is `TwinCAT OS (x64)` in System Manager.
Do not rely only on DTE selection or the solution-wide platform selector.
Stop if any value differs. Do not switch targets to make them match.

Starting UmRT, starting a PLC application and downloading an application are separate actions.
Do not download or activate the configuration at every startup.
First confirm that the expected `S85_LOCAL` application is loaded and check its PLC state.
If it is loaded and stopped, an authorized Start can run it without a new download.
If it is absent or outdated, download only after separate authorization and target checks.
If Login prompts for a download, stop unless that download is separately authorized.

### Simulation and TcUnit results

LOCAL `MAIN` reads these simulation inputs and writes these simulation outputs:

- `GVL_WorkshopSimulation.bButton`
- `GVL_EntrywaySimulation.bButton`
- `GVL_EntrywaySimulation.bUpperButton`
- `GVL_WorkshopSimulation.bLightRelay`
- `GVL_EntrywaySimulation.bLightRelay`
- `GVL_EntrywaySimulation.bTemporaryLightRelay`

`PlcTask` calls `MAIN` and `PRG_Tests`; `PRG_Tests` calls `TcUnit.RUN()`.
The test suites are `fbButtonTests`, `fbLightTests`, `fbWorkshopTests`, `fbHomeTests` and `fbLocalIntegrationTests`.
Run them only in the authorized LOCAL PLC session.

A Watch List is not required to check the results.
After separate authorization for LOCAL online reads, use the XAE PLC Symbol Browser or an approved ADS symbol reader.
Expand `PRG_Tests`, then each suite instance and its `Tests` array.
Confirm the expected test counts: Button 5, Light 4, Workshop 1, Home 1 and LocalIntegration 3.
Zero or missing tests are not a pass.
Read `NumberOfTests` and inspect indices `1` through that count.
Each active test must have `TestIsFinished=TRUE`, `TestIsFailed=FALSE` and `TestIsSkipped=FALSE`.
Each `AssertionMessage` must be empty; total failed and skipped counts must both be zero.
Do not treat an old Error List summary as a new test run.

### Stop and disconnect

PLC Stop halts the LOCAL PLC application but leaves the UmRT process running.
Logout ends the XAE online session. It does not issue PLC Stop or stop UmRT; a running application can continue.
Stopping UmRT stops its runtime process and every PLC application hosted by that instance.
These actions are different and each online action needs separate authorization.
Do not terminate `TcSystemServiceUm.exe` from Task Manager or assume closing XAE stops it.
No verified UmRT stop batch is documented here; use only an approved native stop control.

## Hardware mappings

The production system maps these BOOL variables through Device 1 (EtherCAT), Term 1 (EK1200):

- `GVL_WorkshopIO.bButton` at `%I*`: Term 3 (ED1809), DI Channel 1, Input.
- `GVL_EntrywayIO.bButton` at `%I*`: Term 3 (ED1809), DI Channel 2, Input.
- `GVL_EntrywayIO.bUpperButton` at `%I*`: Term 3 (ED1809), DI Channel 3, Input.
- `GVL_WorkshopIO.bLightRelay` at `%Q*`: Term 4 (ED2809), DO Channel 1, Output.
- `GVL_EntrywayIO.bLightRelay` at `%Q*`: Term 4 (ED2809), DO Channel 2, Output.
- `GVL_EntrywayIO.bTemporaryLightRelay` at `%Q*`: Term 4 (ED2809), DO Channel 3, Output.

The production mapping is stored in `S85.PROD.tsproj`.

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

Use the [TwinCAT XAE workflow](twincat-xae.md) for solution paths, scoped rebuilds and toolchain limits.

Build PROD and LOCAL separately when a change affects both systems.
Run TcUnit only through the LOCAL test project and its isolated test runtime.
Do not treat unexecuted tests as a pass.
Record task-specific results in the PR, not in project documentation.

Check manually before deployment:

- Start with input FALSE and confirm output OFF.
- Set input TRUE and confirm output ON.
- Hold TRUE for several PLC cycles and confirm output stays ON.
- Release, then press again, and confirm output OFF.
- Restart with input TRUE and confirm output stays OFF until release and a new press.
