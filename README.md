# S85 PLC

Beckhoff TwinCAT 3 PLC software for the S85 home automation system. This document describes setup and manual verification for Workshop H-L1-09. It is a test procedure, not evidence that the project has been compiled, downloaded, or deployed.

## Application structure

`MAIN` owns startup registration, SIM/HW selection, and the cyclic input -> `FB_Home` -> output pipeline. It has no location names or hardware addresses. It does not need a second application orchestration FB. `FB_Home` registers and calls location FBs; `FB_Location_Workshop` owns its buttons and `FB_Light` instances. `GVL_WorkshopConfig.stMainLight` holds the Workshop main light's identity and button binding in one place. A bounded `FB_LightCatalog` validates identities and allocates runtime handles once on startup. Handles are internal array indexes, not stable MQTT identifiers.

Inputs and outputs use `ST_HomeInputs` and `ST_HomeOutputs` with bounded light arrays and explicit validity bits. SIM and Beckhoff adapters map semantic buttons and commanded relay states. The HW-only `%I*` and `%Q*` variables live in `GVL_BeckhoffIO`, not in `MAIN` or domain logic. Failed registration or output binding skips configuration-indexed input reads and drives invalid light outputs OFF. A broker outage does not stop valid local lighting.

After input acquisition, the MQTT adapter consumes commands staged during its previous cycle, exactly once, before `FB_Home` runs. After the output adapter, the same shared MQTT client observes the current authoritative light states and may stage a command for the next PLC scan. This is deliberate one-scan command latency based on the last observed connection state, not proof that the network remains connected at actuation time. Unit-test builds do not run the client.

The generic MQTT integration registers the sealed catalog once, caches topics and Discovery payloads per exposed light, and owns one MQTT5 client. `FB_MqttLightProtocol` handles exact-topic ON/OFF routing, initial synchronization, HA Birth, periodic snapshots, and fair bounded publication selection. `FB_MqttLightBuilder` derives topics, unique IDs, and escaped JSON through `Tc3_JsonXml`; a failed buffer copy prevents publication. Only binary lights are modeled. A later sensor or blind integration should add its own typed behavior, not pretend it is a light.

### Extending the house

**A second Workshop light, `workbench`:** in `S85/Configuration/Locations/GVLs/GVL_WorkshopConfig.TcGVL`, add one `ST_LightDefinition` with the same Workshop location code, slug, and name, `sId := 'workbench'`, `sName := 'Workbench'`, and an unused button index such as `2`. In `S85/Configuration/Locations/POUs/FB_Location_Workshop.TcPOU`, add a button FB, light FB, and registered handle; register the new definition alongside `main` and process its button, one-scan absolute command, and output state by handle. Add focused tests in `S85/Tests/Configuration/Locations/POUs/FB_Location_WorkshopTests.TcPOU`. To operate a real circuit, add a separate verified `%I*`/`%Q*` binding in `S85/Infrastructure/IO/Beckhoff/GVLs/GVL_BeckhoffIO.TcGVL` and map it in the Beckhoff adapters; extend the SIM adapters if needed. **No change to `MAIN`, the generic catalog, or MQTT client/builder/protocol is needed.** The three Workshop location metadata fields are presently repeated within each light definition; if more Workshop devices make this noisy, consolidate them into location constants rather than repeating literal strings.

**A Garage location:** add `GVL_GarageConfig` and `FB_Location_Garage` under `S85/Configuration/Locations/`, with its own stable location code, slug, light ID, display names, and unused button slot. Add only its instance, registration forwarding, and cyclic call to `S85/Application/POUs/FB_Home.TcPOU`; add its TcUnit suite and register new TwinCAT objects in `S85/S85.plcproj`. Extend the SIM and Beckhoff adapters for its signals, adding physical GVL variables and manual XAE links only when real hardware is verified. **`MAIN` and generic MQTT code remain unchanged.**

## XAE setup

1. Open `S85.tsproj` in Windows TwinCAT XAE 3.1.4026.26.
2. In the Library Manager, install and resolve `Tc3_IotBase` (TF6701), `Tc3_JsonXml`, `Tc2_Utilities`, and `TcUnit`. Resolve any other project references as required by the installed XAE environment.
3. Select and verify the intended target XAR, its license (including MQTT 5 support), and broker reachability before connecting or downloading. Do not assume a target or license is available.

The checked-in project default is `S85_SIM`. This is the SIM configuration: `S85_TEST_BUILD` is off, `PRG_Tests` is not called, and `MAIN` passes no MQTT host and `bConfigurationReady := FALSE`. Consequently, the default does not run TcUnit tests or connect to a broker.

## SIM test and integration builds

### TcUnit test build

Use only a nonproduction XAR:

1. In the PLC project's build properties, set Compile definitions exactly to `S85_SIM,S85_TEST_BUILD`.
2. Clean and rebuild in XAE, then download to the selected SIM/DEV target only.
3. Run the TcUnit tests and inspect the test results in XAE. A successful build alone is not a test result.

With `S85_TEST_BUILD` defined, the MQTT adapter is not run and workshop MQTT commands are disabled. Never use this definition for hardware or a production target.

### Canary-first build stage (no broker connection)

The existing `S85_SIM` and `S85_SIM,S85_TEST_BUILD` builds remain broker-free. The proposed `S85_MQTT_CANARY` definition is a separate, nonproduction canary stage: use it only together with `S85_SIM`, and never together with `S85_TEST_BUILD`, `S85_HW`, or `S85_PROD`. This stage permits only direct PLC file loading of the canary configuration described below; it is not permission to connect to a broker. Keep the literal `bConnect := FALSE` in the canary build and verify that there is no broker traffic. Do not claim runtime loading or behavior works until it has been tested in XAE on the intended XAR.

For the canary build, compile definitions must be exactly `S85_SIM,S85_MQTT_CANARY`. Use a SIM/DEV XAR only. Inspect ADS-visible symbols, the generated `.tmc`, and boot/export artifacts with synthetic canary values. Confirm that neither username nor password is included in symbols or exported/boot artifacts; `TcNoSymbol` is not guaranteed to hide credential memory, and library FB internals may expose retained credentials. If any value is exposed, stop and do not enable a connection.

There is no live authentication in this stage. A later live test requires credentials rotated to a dedicated SIM-only account, a narrowly scoped ACL, an isolated network, completed target/runtime security checks, and separate explicit approval. Port 1883 without TLS is permitted only as a later, explicitly approved isolated-lab exception: credentials and traffic are plaintext and can be observed on the network. Do not use real credentials for the canary stage.

## MQTT configuration and credential security

The generic files under `config/` remain examples for future integration; the PLC does not load them. The `mqtt.canary*.conf.example` files contain synthetic placeholders for the canary build only. That build now includes a PLC file loader, but its compilation and startup behavior still need XAE validation. TF6701 does not read these files directly. The `/etc/...` paths and file API behavior on the Linux target remain unverified. TLS and CA-file path support also require target validation.

For Windows XAR canary tests, manually provision these target-local paths outside the repository and outside XAE boot/export:

- Nonsecret configuration: `C:\ProgramData\S85\sim\config\mqtt.conf`
- Credentials: `C:\ProgramData\S85\sim\secrets\mqtt.conf`

The canary nonsecret file must contain exactly these required keys and values, one per line: `MQTT_MODE=CANARY`, `MQTT_HOST=canary.invalid`, `MQTT_PORT=1883`, and `MQTT_TLS=false`. The secrets file must contain exactly `MQTT_USERNAME=CANARY-<locally-chosen-suffix>` and `MQTT_PASSWORD=CANARY-<different-locally-chosen-suffix>`. Choose distinct synthetic suffixes locally. Never use or copy a real password from conversation, and never put real secrets in Git, PLC symbols, logs, or the HMI. Each file is narrow ASCII `KEY=value` with CRLF line endings, at most 2048 bytes, no unknown keys, and no duplicate keys. The optional tracked templates contain placeholders only and are not target-ready files.

Provision manually on the Windows target using a local editor: create the directories and files directly at the paths above, without pasting real secrets into chat, a command line, or shell/PowerShell history. Verify the XAR account and ACLs permit only the intended runtime access before testing. Keep all real DEV/SIM and PROD configuration and credentials target-local and separate. For a later live test, use a dedicated account/ACL and prefer TLS with certificate verification; the example CA path in the generic template is illustrative and path support is unverified.

For stage A, build with synthetic canary values only, inspect ADS/TMC/boot artifacts, and verify the literal `bConnect := FALSE` and absence of broker traffic. Do not infer secrecy from `TcNoSymbol`: credentials held in memory or by library FBs may not be hidden. The Linux CX file API, paths, runtime account, and permissions remain unverified. No live authentication is allowed until credentials have been rotated to a dedicated SIM account on an isolated network and separate approval is obtained. Before any later deployment, fail closed if a validated secure configuration source is absent; validate actual file access and permissions on the intended target, and inspect generated artifacts. Do not deploy until XAE evidence and security checks are complete.

## Hardware / production preparation

For a hardware or production configuration, the intended Compile definitions are `S85_HW,S85_PROD`. Do not define `S85_TEST_BUILD` for hardware or production. The source declares hardware process-image variables but does not establish verified physical channel mappings:

1. In XAE, manually link `GVL_BeckhoffIO.bWorkshopMainButton AT %I*` to the verified digital input and `GVL_BeckhoffIO.bWorkshopMainLightRelay AT %Q*` to the verified relay digital output for this installation. Previously linked `MAIN.*ProcessImage` symbols must be relinked manually; do not guess channels.
2. Verify the actual terminal/channel assignment, input/output polarity, stop behavior, and fail-safe behavior on the intended machine before enabling the output.
3. Do not infer or guess an ED2809 channel from the project files. No physical channel is documented here as verified.
4. Confirm target, XAR, license, broker configuration, and operational authorization before any download or live test. `PRG_Tests` must not be part of a production task. Production deployment is blocked until secure configuration loading, artifact inspection, and target permission and TLS validation are evidenced.

## MQTT contract

| Purpose | Topic | Behavior |
| --- | --- | --- |
| Home Assistant discovery | `homeassistant/light/s85_workshop_main/config` | Retained discovery configuration for the Workshop Main light. |
| Light command | `s85/locations/H-L1-09/lights/main/set` | Accepts `ON` or `OFF`; commands set the requested state and are idempotent. Subscription is QoS 1 and is configured not to replay retained messages on subscription. |
| Light state | `s85/locations/H-L1-09/lights/main/state` | Retained `ON` or `OFF` state; published after state changes and as a 60-second snapshot. |
| PLC availability | `s85/plc/status` | Retained `online`; the MQTT last will publishes retained `offline` on an unexpected disconnect. |
| PLC heartbeat | `s85/plc/heartbeat` | Incrementing, non-retained counter, published every 10 seconds while connected and subscribed. |
| Home Assistant birth | `homeassistant/status` | An `online` message triggers discovery, current state, and availability publication again. |

The retained online availability message and MQTT last will do not guarantee freshness: an old retained `online` value can remain if the broker or PLC stops in ways that do not deliver the will. A Home Assistant or broker heartbeat watchdog consumer with a timeout is required to reliably mark the device unavailable. In the PLC code, a successful `Publish` or `Subscribe` method return indicates local method acceptance only; it is not confirmation of MQTT `PUBACK` or `SUBACK`. Validate protocol behavior in the XAE runtime and broker.

## Manual broker test checklist

Run only against the isolated broker using a nonproduction SIM/DEV XAR. Observe messages on the topics above and record results; do not treat this checklist as already passed.

- Verify an existing retained command is rejected and not applied on subscription. A live retained publish after subscription can still be delivered, so retained publishes to the command topic MUST be prohibited by broker ACLs or publisher policy. Do not use them as a valid command path. Confirm non-retained commands change state and repeated identical commands remain idempotent.
- Verify Home Assistant birth (`homeassistant/status` payload `online`) causes discovery, current state, and availability to be published again.
- Interrupt and restore the connection. Verify reconnect behavior and inspect QoS 1 delivery for duplicates. Commands are absolute `SET` values, not exactly-once or freshness-guaranteed: a redelivered command can overwrite a later local change.
- Verify retained state snapshots are sent every 60 seconds, the non-retained heartbeat counter advances every 10 seconds, and the online availability / last-will behavior is as expected.
- Restart the broker while the PLC is offline. Check what retained status and state remain, then verify fresh discovery, state, and online status after the PLC reconnects. Separately verify that the watchdog timeout handles stale retained `online` status.
