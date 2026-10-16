# MQTT light integration

The MQTT integration connects the home application to a broker. Both LOCAL and PROD load configuration from operating-system files. The example files contain placeholders and do not provide working credentials.

## Configuration files

The PLC reads these paths:

| System | Configuration | Secrets |
| --- | --- | --- |
| LOCAL | `C:\ProgramData\S85\config\mqtt.conf` | `C:\ProgramData\S85\secrets\mqtt.conf` |
| PROD | `/etc/s85/config/mqtt.conf` | `/etc/s85/secrets/mqtt.conf` |

Copy `config/mqtt.conf.example` and `config/mqtt.secrets.conf.example` to the matching paths. Set values for the actual broker and provision credentials in the secrets file. Do not copy example values as live settings. The secret example is blank because the parser rejects missing credentials.

The files use ASCII `key=value` lines. Empty lines are allowed. Comments, unknown keys, duplicate keys, and invalid key names are rejected. The parser does not trim whitespace, so values must match the required format exactly. Each file must be no larger than 2048 bytes.

The configuration file requires `MQTT_HOST`, `MQTT_PORT`, `MQTT_CLIENT_ID`, `MQTT_TLS`, and `MQTT_CA_FILE`. Use `true` or `false` for `MQTT_TLS`. TLS requires a non-empty CA file path. The secrets file requires non-empty `MQTT_USERNAME` and `MQTT_PASSWORD` values.

File access depends on the actual runtime account. Grant that account read access to both files and the CA file. On Windows, check the file ACLs for the account running the LOCAL UmRT runtime. On Linux, check ownership and permissions for the PROD runtime account. OS file access and runtime owner permissions have not been verified here.

## Topics and payloads

Each light uses `ON` and `OFF` payloads. Set topics use QoS 1 and do not retain messages. State topics use QoS 1 and retain the latest state.

| Light | Set topic | State topic |
| --- | --- | --- |
| Workshop main | `s85/locations/H.01-09/lights/main/set` | `s85/locations/H.01-09/lights/main/state` |
| Entryway main | `s85/locations/H.01-01/lights/main/set` | `s85/locations/H.01-01/lights/main/state` |
| Entryway temporary | `s85/locations/H.01-01/lights/temporary/set` | `s85/locations/H.01-01/lights/temporary/state` |

The PLC derives topics from each light's location and ID metadata. It accepts only exact `ON` or `OFF` commands on the three set topics. It accepts commands while connected, including duplicate absolute commands. Commands queue in order, up to 16 entries; a full queue rejects new commands. A reconnect clears queued commands and publishes a fresh state snapshot.

Retained state can be stale while the PLC is offline. The integration does not define discovery, broker birth handling, or availability topics. A client must not treat retained state as proof that the PLC is online.

Retain handling two suppresses stored retained `SET` messages when the PLC subscribes. Session expiry zero disables the offline session queue. Clean-start wire behavior remains unverified and must be checked during commissioning. The PLC cannot detect or reject a new retained `SET` message published while its subscription is active. It accepts that message like any other valid light command. These settings do not authorize historical replay.

## Broker access and commissioning

Use a dedicated broker identity for each PLC. Give it write access only to the three state topics and read access only to the three set topics. Give Home Assistant read access only to state topics and write access only to set topics. Apply these rules in the broker ACL; do not rely on a Home Assistant plugin to enforce them.

Before enabling commands, test with a non-production broker or an approved commissioning setup:

1. Seed a retained `ON` command while the PLC is disconnected. Connect the PLC and check that the retained command does not execute.
2. Publish `ON` and `OFF` while connected. Confirm the intended light changes and the state topic reports the current value.
3. Disconnect after publishing commands, then reconnect. Check for command replay and confirm the PLC publishes a fresh snapshot.
4. Publish duplicate commands and commands to all three topics. Confirm absolute commands do not toggle state and the intended light receives each command.
5. Confirm broker ACLs reject writes and reads outside each identity's allowed topics.

Do not enable commands on PROD until the broker behavior and ACLs pass commissioning. No broker acceptance results are recorded here.

## Home Assistant

For Home Assistant OS, install and configure the official Mosquitto add-on. Create dedicated broker account(s), then configure the Home Assistant MQTT integration with the broker details. Do not put real credentials in this example or in source control.

`config/home-assistant-mqtt-lights.yaml.example` provides manual YAML configuration for three lights. Merge its light entries under the existing `mqtt:` key in Home Assistant configuration. Do not add a duplicate `mqtt:` key. The entries use QoS 1, disable retained commands and optimistic state, and assign stable unique IDs.

If the broker uses TLS, configure trusted CA access for the Home Assistant MQTT integration. Confirm the add-on's TLS settings before choosing a port or CA path; this example does not assume either.

The PLC marks internal configuration fields with `TcNoSymbol` to reduce symbol export. This is not encryption and does not prove protection in memory or in a boot project. Protect the files and runtime access accordingly.

## Configuration errors

The loader fails closed if it cannot read or parse either file. `MAIN.nMqttConfigError` identifies these stages: 1 means open, 2 means read, 3 means size or NUL byte, 4 means close, 5 means parse, and 6 means an empty path. `MAIN.nMqttSystemError` exposes the underlying ADS error code without file contents. Correct the files or access, then restart the PLC application only with separate authorization. The loader does not automatically retry after an error.
