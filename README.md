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

### Broker integration on SIM

1. Leave `S85_TEST_BUILD` off and use a SIM/DEV target with Compile definitions `S85_SIM`.
2. The adapter uses a fixed MQTT client ID `s85-plc` and shared fixed topics below. Use an isolated DEV/SIM broker and separate dedicated DEV/SIM credentials and ACLs. Never connect a test instance to a production broker.
3. The checked-in MQTT integration is deliberately disabled. Do not attempt broker integration until a secure configuration source, credentials, and startup behavior have been implemented and validated for the target. An online write cannot provision credentials or enable MQTT in the checked-in build.

## MQTT configuration and credential security

The files under `config/` are examples of a proposed plain UTF-8 ASCII `KEY=value` format only. They are not a runtime configuration interface: the PLC does not currently load these files, and no TwinCAT credential-file API or loader has been implemented or verified. Do not treat copying these files to a target as configuration. `Tc3_IotBase` (TF6701) documentation does not establish that `/etc/...` paths work on the target or provide a documented credential-file API. TLS support, CA-file path support, and startup configuration must be validated on the actual target; TLS is preferred, not assumed to work.

The proposed target paths are:

- Nonsecret configuration: `/etc/s85/config/mqtt.conf`
- Credentials: `/etc/s85/secrets/mqtt.conf`

Keep separate DEV/SIM and PROD configuration and credentials outside the repository. Use a dedicated MQTT ACL user for each environment, restrict it to required topics, and prefer TLS with certificate verification. Never put secrets in Git, PLC symbols, logs, or the HMI. The example host and credentials are placeholders only. The example CA path `/etc/s85/certs/mqtt-ca.pem` is illustrative; path support is unverified.

Before any deployment, fail closed if a validated secure configuration source is absent. First implement and validate how the runtime obtains host, port, TLS/CA settings, and credentials without exposing credentials in PLC symbols or other artifacts. In Windows XAE, inspect the generated `.tmc`, ADS-visible symbols, and boot artifacts to verify that credentials are not exposed. On the target, verify the actual TwinCAT runtime uid/gid and access to the configured files. Create the directories manually; use root/admin ownership only if runtime access is verified. Restrict the secrets directory to `0700` or appropriate group traversal, and the secret file to `0600` only when its owner and runtime access have been verified. Nonsecret config may be `0644` when appropriate. Validate permissions, TLS certificate verification, configuration parsing, startup, and restart behavior on the intended nonproduction target before considering production. Do not deploy until parent XAE evidence and these security checks are complete.

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
