# TwinCAT XAE workflow

Read this document for XAE startup, PLC builds and toolchain diagnostics.
For compiler defines, I/O mappings and tests, read [Workshop setup and checks](workshop-light.md).

## Confirmed environment and limits

- Windows with TwinCAT XAE 4026.27 and TcXaeShell based on Visual Studio 2022.
- PROD hardware mode (`WORKSHOP_HW`): build PASS on `Release | TwinCAT OS (ARMV8-A)`, with 0 PLC errors and 0 compiler warnings.
- This is the verified hardware build, not a separate solution configuration named `PROD`.
- SIM is unverified. Do not infer a SIM pass from the hardware build.
- The verified route uses the TcXaeShell GUI and DTE from Windows PowerShell in STA.
- A build generates local artifacts. It does not deploy them or verify runtime behavior on the controller.

The 2026-10-13 baseline applies only to this environment and configuration.
Build the configurations relevant to each PLC change and record fresh results in the PR.

## Prepare the solution

- `S85.sln` is tracked. It includes `S85.tsproj` and the standalone library wrapper `S85.Core/S85.Core.tspproj`.
- The library code is in `S85.Core/S85_Core.plcproj`.
- For a temporary project copy, copy `S85.sln` unchanged and preserve both relative project paths.
- If `S85.sln` is missing, stop and report the prerequisite. Do not create a replacement for verification.

## Verified build route

1. Start the TcXaeShell GUI normally.
2. Start Windows PowerShell in STA, for example with `powershell.exe -NoProfile -STA`.
3. Attach to the running instance using `Marshal.GetActiveObject("TcXaeShell.DTE.17.0")` from `System.Runtime.InteropServices`.
4. Open the existing solution through DTE's `Solution.Open`. Use an unchanged temporary copy when protecting the working project from generated metadata.
5. Check the active configuration and platform in XAE. Do not assume the platform from an earlier command or build.
6. Call `SolutionBuild.Build(true)` on the DTE solution and wait for completion.
7. Check `SolutionBuild.LastBuildInfo`: `0` means success. Inspect Output and Error List for the entire solution.
8. Record the actual configuration, platform, errors, warnings and generated artifacts. Report unbuilt configurations separately.

GUI startup, DTE attachment and compilation succeeded when launched from OpenCode.
This does not establish that every unattended startup method works.

## Diagnose failures by stage

- **XAE environment:** the tested CLI `/Build` invocation exited with code `1` before loading TwinCAT build packages or producing a build log.
- Its cause remains unknown. Even with the solution present, that invocation did not provide compilation evidence.
- Use the verified GUI/DTE route instead of assuming CLI support or changing the PLC project.
- **Startup log:** `SyncLock called without an initialized synchronization object` comes from `Microsoft Visual Studio Appid Stub` during shell initialization.
- The same message appears during successful GUI startup and compilation. It is not an established build blocker or a PLC compiler error.
- **PLC compilation:** use the completed build result and compiler diagnostics, not shell startup messages, to identify PLC errors.
- **Warnings:** report compiler warnings separately from errors and from XAE ActivityLog entries.

## Deployment boundary

Do not activate a configuration, download PLC code or restart the runtime as part of build verification.
PROD deployment requires separate, explicit authorization for the target and operation.
For authorized commissioning, follow the target and safety checks in [Workshop setup and checks](workshop-light.md#before-activation).

## Reference

- [Beckhoff: compiling PLC projects through the Automation Interface](https://infosys.beckhoff.com/content/1033/tc3_automationinterface/21842451979.html).
