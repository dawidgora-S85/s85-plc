# TwinCAT XAE workflow

Read this document for XAE startup, PLC builds and toolchain diagnostics.
For hardware mappings, read [Workshop setup and checks](workshop-light.md).
For LOCAL runtime setup and test results, read [LOCAL runtime and tests](workshop-light.md#local-runtime-and-tests).

## Solution structure

- `S85.sln` contains the PROD system, LOCAL system and shared Core library.
- `S85.PROD.tsproj` is the production system project.
- It uses `S85/S85_PROD.plcproj`, PLC application `S85_PROD` and instance `S85_PROD Instance`.
- `Local/S85.LOCAL.tsproj` is the local system project.
- It uses `Local/S85.LOCAL/S85_LOCAL.plcproj`, PLC application `S85_LOCAL` and instance `S85_LOCAL Instance`.
- Both applications use the library in `S85.Core/S85_Core.plcproj` through `S85.Core/S85.Core.tspproj`.

## PLC application layout

- The physical project roots remain `S85.Core/`, `Local/S85.LOCAL/` and `S85/`.
- Core's `Application` folder contains the home application DUTs and POUs. `Locations/Workshop` and `Locations/Entryway` each contain their DUTs and POUs.
- `Domains/Lighting` and `Shared` remain unchanged. `FB_Workshop` stays in `Locations/Workshop/POUs` as behavior separate from the `FB_Location_Workshop` contract adapter.
- LOCAL uses x64, with simulation under `Infrastructure/IO/Simulation/{POUs,GVLs}` and tests under `Tests`.
- PROD uses ARMv8 and physical Beckhoff I/O under `Infrastructure/IO/Beckhoff/{POUs,GVLs}`.
- Core provides shared, environment-independent home logic. LOCAL provides simulation and TcUnit tests, while PROD connects to physical Beckhoff I/O.

## Build configurations

- Build PROD with `Release | TwinCAT OS (ARMV8-A)`.
- Build LOCAL with the existing `Release | TwinCAT OS (x64)` configuration.
- PROD contains hardware application code only. LOCAL contains simulation and TcUnit tests.
- Do not use compiler defines to switch between PROD and LOCAL.

## Open the repository solution

1. Start TcXaeShell normally, without Administrator privileges.
2. Open the repository solution at `C:\S85\s85-plc\S85.sln`.
3. For DTE automation, use Windows PowerShell in STA, such as `powershell.exe -NoProfile -STA`.
4. Attach with `Marshal.GetActiveObject("TcXaeShell.DTE.17.0")` from `System.Runtime.InteropServices`.
5. Confirm `Solution.FullName` is the repository solution. Confirm the selected system project has the expected name, full path and GUID.
6. Confirm the solution, projects and open documents are saved. Stop if XAE shows a conversion or repair prompt.

Do not create a temporary project copy for repository verification.
Do not use `Save All` or discard dirty buffers to continue.

## Scoped clean rebuild

1. Select only the affected system project root in Solution Explorer.
2. Select the project's required configuration and platform in XAE.
3. Use native `Build.RebuildSelection`. It rebuilds the selected project with the native clean-and-build operation.
4. Do not use solution-wide `Build`, `Rebuild` or `SolutionBuild.Build(true)` for PROD or LOCAL verification.
5. Do not use `SolutionBuild.BuildProject` as evidence of a clean rebuild. It can perform an incremental build.
6. Confirm Output says `Rebuild All` for the intended project and configuration.
7. Record `SolutionBuild.LastBuildInfo`, errors, warnings, elapsed time and fresh artifact timestamps.

`LastBuildInfo=0` means the build succeeded. Inspect the selected project's Output and Error List.
A build creates local artifacts. It does not deploy them or verify controller runtime behavior.

## Diagnose toolchain failures

- A tested CLI `/Build` invocation exited with code `1` before loading TwinCAT build packages or producing a build log.
- Its cause remains unknown. Do not assume CLI support or change project configuration to work around it.
- A startup message about an uninitialized synchronization object can come from the Visual Studio shell.
- Do not treat that message as a PLC compiler error when the build completed successfully.
- Report compiler warnings separately from errors and XAE ActivityLog entries.

## Deployment boundary

Build verification is offline. Do not activate a configuration, download PLC code or restart a runtime.
PROD deployment requires separate, explicit authorization for the target and operation.
For authorized commissioning, follow [Before activation](workshop-light.md#before-activation).

## Reference

- [Beckhoff: compiling PLC projects through the Automation Interface](https://infosys.beckhoff.com/content/1033/tc3_automationinterface/21842451979.html).
