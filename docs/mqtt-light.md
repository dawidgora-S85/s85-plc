# MQTT light integration

The MQTT integration connects the home application to a broker. LOCAL and PROD load configuration from operating-system files. Example files contain placeholders, not working credentials.

## Configuration files

| System | Configuration | Secrets |
| --- | --- | --- |
| LOCAL | `C:\ProgramData\S85\config\mqtt.conf` | `C:\ProgramData\S85\secrets\mqtt.conf` |
| PROD | `/etc/s85/config/mqtt.conf` | `/etc/s85/secrets/mqtt.conf` |

Copy `config/mqtt.conf.example` and `config/mqtt.secrets.conf.example` to the matching paths. Set actual broker values and provision credentials in the secrets file. Do not copy example values as live settings. The secret example is blank because the parser rejects missing credentials.

The files use ASCII `key=value` lines. Empty lines are allowed. Comments, unknown keys, duplicate keys, and invalid key names are rejected. The parser does not trim whitespace. Values must match the required format exactly. Each file must be no larger than 2048 bytes.

The configuration file requires `MQTT_HOST`, `MQTT_PORT`, `MQTT_CLIENT_ID`, `MQTT_TLS`, and `MQTT_CA_FILE`. Use `true` or `false` for `MQTT_TLS`. TLS requires a non-empty CA file path. The secrets file requires non-empty `MQTT_USERNAME` and `MQTT_PASSWORD` values.

File access depends on the runtime account. Grant that account read access to both files and the CA file. Check Windows file ACLs for the account running LOCAL UmRT. Check Linux ownership and permissions for the PROD runtime account. OS file access and runtime owner permissions have not been verified here.

## Topics and discovery

Light commands use exact `ON` and `OFF` payloads. Set topics use QoS 1 without retention. State topics use QoS 1 and retain the latest state.

Each `MAIN` explicitly selects its environment namespace. LOCAL selects test namespace `s85/test`; PROD selects production namespace `s85`. The namespace applies to light command and state topics, discovery nodes, and availability.

| Light | LOCAL set topic | LOCAL state topic | PROD set topic | PROD state topic |
| --- | --- | --- | --- | --- |
| Workshop main | `s85/test/locations/H.01-09/lights/main/set` | `s85/test/locations/H.01-09/lights/main/state` | `s85/locations/H.01-09/lights/main/set` | `s85/locations/H.01-09/lights/main/state` |
| Entryway main | `s85/test/locations/H.01-01/lights/main/set` | `s85/test/locations/H.01-01/lights/main/state` | `s85/locations/H.01-01/lights/main/set` | `s85/locations/H.01-01/lights/main/state` |
| Entryway temporary | `s85/test/locations/H.01-01/lights/temporary/set` | `s85/test/locations/H.01-01/lights/temporary/state` | `s85/locations/H.01-01/lights/temporary/set` | `s85/locations/H.01-01/lights/temporary/state` |
| Hall main group | `s85/test/locations/H.01-02/lights/main/set` | `s85/test/locations/H.01-02/lights/main/state` | `s85/locations/H.01-02/lights/main/set` | `s85/locations/H.01-02/lights/main/state` |
| Hall outer spots | `s85/test/locations/H.01-02/lights/outer_spots/set` | `s85/test/locations/H.01-02/lights/outer_spots/state` | `s85/locations/H.01-02/lights/outer_spots/set` | `s85/locations/H.01-02/lights/outer_spots/state` |
| Hall middle spot | `s85/test/locations/H.01-02/lights/middle_spot/set` | `s85/test/locations/H.01-02/lights/middle_spot/state` | `s85/locations/H.01-02/lights/middle_spot/set` | `s85/locations/H.01-02/lights/middle_spot/state` |

The catalog has six logical MQTT lights. The three existing lights keep their IDs and topics. Hall uses location `H.01-02`, verified as House H, Floor 1, LocationNumber 2. Its IDs are `main`, `outer_spots`, and `middle_spot`.

Hall `main` is a group light, not an independent latch. Its SET command applies the requested absolute state to both spot lights. Its STATE is ON when either member is ON. Commands to `outer_spots` and `middle_spot` affect only that member. The physical Hall upper button turns both spots OFF if either is ON; otherwise it turns both ON. The lower button keeps its existing Entryway main-light behavior.

Discovery uses one Home Assistant device per logical light. The device name joins its location and light names with ` - `. Names use UTF-8 and are each limited to 64 bytes. Their combined name is limited to 137 bytes, including the separator and the LOCAL `TEST: ` prefix. PROD has no prefix. The light component uses `name: null`, so its entity inherits the device name. Each device declares `dev`, `o`, and `cmps` in its device-discovery payload. The shared manufacturer is `S85` and model is `PLC Light`. See [Home Assistant MQTT device discovery](https://www.home-assistant.io/integrations/mqtt/#device-discovery-payload) and [MQTT light](https://www.home-assistant.io/integrations/light.mqtt/).

Friendly location names are `Warsztat`, `Wiatrołap`, and `Hol`. Light names are `Oświetlenie główne`, `Oświetlenie tymczasowe`, `Oświetlenie`, `Spoty zewnętrzne`, and `Spot środkowy`, as applicable. Discovery text uses UTF-8 `STRING` literals. Validation checks UTF-8 strictly. Serialization escapes JSON text. The discovery payload limit remains 2047 bytes.

The discovery topic is `homeassistant/device/s85_test/<identity>/config` in LOCAL and `homeassistant/device/s85/<identity>/config` in PROD. The identity is `s85_<HEXlocation>_light_<HEXid>`. Hex encodes every byte using uppercase digits, making the identity injective. For example, location `H.01-09` and ID `main` encode as `s85_482E30312D3039_light_6D61696E`. A location or ID change changes the discovery topic. Reconciliation can publish the new identity and remove the old topic when its obsolete value is observed in the active namespace. A name-only change keeps the identity and topic, then republishes the discovery configuration. This does not imply an immediate reconnect or guarantee cleanup without the required observations.

The `s85/test/availability` topic serves LOCAL. The `s85/locations/availability` topic serves PROD. Discovery advertises this shared device availability topic. The PLC configures a retained QoS 1 `offline` Last Will before connecting, and publishes retained `online` after its connection and publication sequence succeeds locally.

On startup, publish a retained `offline` fence before replaying discovery or state. The PLC publishes retained `online` only after it observes matching broker discovery values and locally accepts fresh STATE publications for the current replay. STATE values use retained QoS 1, but the PLC does not observe them through a STATE subscription. A successful library call means local acceptance, not broker PUBACK or Home Assistant processing. A graceful disconnect can attempt an `offline` publication, but delivery is best effort and not PUBACK-confirmed. See Beckhoff's [MQTT Last Will sample](https://infosys.beckhoff.com/content/1033/tf6701_tc3_iot_communication_mqtt/14019927307.html).

Availability reports broker-observed MQTT connectivity only. A broker can publish the Will after it observes a client disconnect. A PLC Stop may leave the TCP connection alive under `TcSystemServiceUm`, so LWT does not guarantee PLC Stop detection. Stop/Start may preserve the function block and TCP connection, with no reconnect or replay. MQTT message expiry is not an offline signal.

An independently authorized heartbeat supervisor could detect missed cyclic updates. It would need a Home Assistant condition for an expired heartbeat, in addition to MQTT connectivity. This watchdog and condition are not implemented. On a new MQTT connection, the PLC accepts an offline fence, establishes command and Home Assistant birth subscriptions, then performs discovery, state, and online publication. The default Home Assistant birth topic and payload are `homeassistant/status` and `online`. Home Assistant can customize them.

The PLC derives topics from each light's location and ID metadata. Commands are accepted only while connected and only on the exact set topics. Duplicate absolute commands are valid. Commands queue in order, up to 16 entries; a full queue rejects new commands. A reconnect clears queued commands and publishes a fresh state snapshot. Retain handling two suppresses retained `SET` messages when subscribing, and session expiry zero disables offline session queuing. Clean-start wire behavior remains unverified. A retained command published while subscribed is accepted like any other valid command.

Every exact `online` payload on `homeassistant/status` is observed. When received in Ready, it requests fresh Discovery and STATE work through a non-fencing refresh. While startup, recovery, or that refresh is in progress, further status messages coalesce without resetting progress or scheduling a trailing replay. An `online` received after the refresh completes requests another refresh. This behavior does not depend on an observed `offline` transition, a previous status value, or a birth generation. A birth refresh does not disconnect or resubscribe, reset the command FIFO, or fence commands. While the MQTT connection is healthy, `bReady` remains true as command readiness during the refresh, even while `eLifecyclePhase` is `Discovery` or `StateSnapshot`; the phase shows that synchronization is in progress, not that work is settled. Availability remains `online`. Genuine catalog, invalid-data, transport, or coverage failures still follow their existing fail-closed paths. In a forced replay, each desired Discovery item requires fresh local publication acceptance followed by matching observed topic and payload bytes. STATE is considered complete on local publication acceptance of the current PLC state, not on PUBACK or Home Assistant processing.

## Discovery reconciliation and migration

Discovery reconciliation is designed around broker-observed retained values, not a persistent local registry or publish acknowledgements. The controller subscribes to retained discovery configuration in its own namespace and compares the observed topics and payloads with the current light catalog. It republishes desired configurations that are absent or different, and publishes an empty retained payload for obsolete topics it observes in that namespace. This supports recovery after a PLC restart without depending on volatile topic history.

The catalog must be wholly valid before reconciliation can delete anything. An unready or invalid catalog is not authoritative and cannot trigger cleanup. A ready catalog with zero lights is authoritative empty. The desired set is bounded to 16 entries. The orphan work queue holds 32 topic-only entries. An observed own-namespace discovery topic that cannot be admitted because the queue is full latches a coverage fault; admitted cleanup may continue, but readiness remains false. Reconnects and replay do not clear that fault, and no automatic coverage-triggered reconnect is scheduled. A restart alone is not evidence of complete coverage. Reset policy and proof of coverage require separate operational handling.

### Receive buffers and coverage faults

`FB_IotMqtt5Client` processes MQTT callbacks internally and queues messages during `Execute`; this project does not register a custom callback. Beckhoff documents callback topic as `VAR_IN_OUT CONSTANT STRING` and payload as a pointer plus length ([callback API](https://infosys.beckhoff.com/content/1033/tf6701_tc3_iot_communication_mqtt/12560873867.html)). Queued messages expose `nTopicSize` as `UINT` and `nPayloadSize` as `UDINT` ([message sizes](https://infosys.beckhoff.com/content/1033/tf6701_tc3_iot_communication_mqtt/14629604107.html)). `GetTopic` takes `nTopicSize` as destination byte capacity ([GetTopic](https://infosys.beckhoff.com/content/1033/tf6701_tc3_iot_communication_mqtt/14629668875.html)), but the documentation does not establish terminator, truncation, or Boolean-result semantics. The PLC therefore requires a successful getter and validates the bounded copy: content length must match the advertised size or be one byte shorter with a verified final NUL.

The topic buffer is `STRING(255)`, or 256 bytes including its terminator. Content may be at most 255 bytes; an advertised size of 256 is accepted only when byte 255 is the verified NUL. Failed, oversized, unterminated, or mismatched topic copies are unclassifiable: the PLC rejects them and latches `bDiscoveryCoverageFault` rather than using possibly truncated bytes to claim ownership or publish cleanup. A valid exact foreign topic is ignored or rejected without latching the fault. The diagnostic remains set until the function-block instance is reinitialized. The output is not retained across a full restart or download, but that is not proof that complete broker coverage was restored. There is no automatic fault reset or coverage-triggered reconnect. Ordinary transport reconnect behavior remains; it does not clear the fault.

Payload copies are validated separately. `GetPayload` needs an extra destination byte when forced NUL termination is requested ([GetPayload](https://infosys.beckhoff.com/content/1033/tf6701_tc3_iot_communication_mqtt/14629701643.html)). Discovery uses `STRING(2047)`, allowing up to 2047 payload bytes plus NUL. If payload copying fails or exceeds that limit while the topic is exact, the topic remains available for safe reconciliation; payload bytes are not trusted. SET uses `STRING(3)` for exact 2-byte `ON` or 3-byte `OFF`, and Home Assistant Birth uses `STRING(8)` for exact 6-byte `online` or 7-byte `offline`.

The message queue counts messages discarded because of queue or message-size limits. Beckhoff lists default limits of 100 kB per message, 1000 entries, and 1000 kB total queue size ([queue limits](https://infosys.beckhoff.com/content/1033/tf6701_tc3_iot_communication_mqtt/14021292811.html)); the installed library parameters have not been verified, and this project does not change them. A lost-message count delta latches the coverage fault. The PLC does not reset that fault based on a replay or reconnect. Operational reset policy and evidence of restored coverage must be agreed separately. SDK internals, actual callback wire behavior, and broker delivery remain unverified; LOCAL helper tests exercise project logic, not an SDK or broker end-to-end path.

Publish acceptance only schedules retry. An admitted desired item settles only after the matching topic and exact payload bytes are observed. A replay request adds a fresh-publication requirement: local acceptance of a fresh retained publication must precede the matching observation. An admitted tombstone settles only after an empty value for its topic is observed. Conflicting values reopen work. The retry delays are 1, 2, 4, 8, 16, 32, then 60 seconds, capped at 60 seconds. Detected discovery coverage loss latches a fault and blocks readiness. No automatic coverage-triggered reconnect or replay can clear it; restart alone is not evidence of complete coverage. A settled catalog does not periodically force a reconnect. During startup, recovery, genuine catalog change, or other fenced replay/unsettled work, availability remains offline behind the fence, commands are rejected, and queued commands are discarded on fencing. A healthy Home Assistant Birth refresh is the exception: availability stays online, commands remain enabled, and the command FIFO is preserved. These are controller contracts; broker delivery and runtime behavior still require E2E verification. If an observation for already admitted work is lost, that work remains unsettled and retries continue according to the bounded backoff. This does not cover a historical retained configuration omitted from the initial replay and never delivered again: the PLC cannot admit or reconcile a topic it never observes. Stable connections continue observing future messages, but neither `bReady` nor settled known work proves that the complete historical retained inventory was received. MQTT observations are not causal acknowledgements, so stale matching duplicates are not proof that a particular publish was applied.

A separately authorized external audit could use a read-only subscriber restricted to the exact LOCAL or PROD discovery namespace, with a fresh subscription to inspect broker-side retained inventory without resetting the PLC session or subscriptions. MQTT provides no replay end marker, so this cannot prove completeness from silence. It also does not feed observations to the PLC and therefore does not automatically reconcile topics the PLC missed. No such audit mechanism is implemented.

Eventual convergence assumes a finite stable retained topic set, fair repeated delivery of subscription observations and publish attempts, valid broker permissions, and a stable valid catalog. It does not guarantee convergence under continuing topic churn, denied permissions, or permanently lost delivery. Home Assistant may be offline during cleanup and retain stale device or entity registry entries. Removing broker retained configuration does not guarantee HA registry removal.

Broker-observed reconciliation applies to discovery only. Fresh STATE publications are locally accepted, not broker-observed by the PLC. STATE uses retained QoS 1. Existing exact `ON`/`OFF` SET payloads, non-retained SET behavior, Variant C device discovery, location names, and availability policy remain unchanged. Zero-length retained cleanup behavior and runtime integration still require E2E verification.

Manual inspection or cleanup is not the normal reconciliation mechanism. If a separate operator-authorized diagnostic is needed, restrict every subscription or publication to the LOCAL own namespace and verify each exact topic. Never use wildcards for deletion, touch PROD configuration or ACLs, or remove an unknown topic. Use protected client credentials.

The old `config/home-assistant-mqtt-lights.yaml.example` is deprecated. Do not add its entries alongside device discovery. Remove old YAML configuration only after user-approved backup and inspection of automations and entity references. Stable new entity IDs can still require a one-time migration.

## Broker access and commissioning

Use a dedicated broker identity for each PLC. Apply ACLs at the broker. The following is the intended least-privilege matrix, not verified broker policy:

| Client | Read or subscribe | Write or publish |
| --- | --- | --- |
| LOCAL PLC identity | `s85/test/locations/+/lights/+/set`, `homeassistant/status`, `homeassistant/device/s85_test/+/config` | `s85/test/locations/+/lights/+/state`, `s85/test/availability`, `homeassistant/device/s85_test/+/config` |
| PROD PLC identity | `s85/locations/+/lights/+/set`, `homeassistant/status`, `homeassistant/device/s85/+/config` | `s85/locations/+/lights/+/state`, `s85/locations/availability`, `homeassistant/device/s85/+/config` |
| Home Assistant | `s85/test/locations/+/lights/+/state`, `s85/locations/+/lights/+/state`, `s85/test/availability`, `s85/locations/availability`, `homeassistant/device/s85_test/+/config`, `homeassistant/device/s85/+/config` | `s85/test/locations/+/lights/+/set`, `s85/locations/+/lights/+/set`, `homeassistant/status` |

The PLC needs read/subscribe access to its own discovery wildcard as well as write/publish access to that namespace. Do not grant broad `homeassistant/#` access. Actual ACLs and enforcement have not been verified. PROD use remains gated on approved ACL and broker commissioning.

Before enabling commands, test with a non-production broker or an approved commissioning setup:

1. Seed a retained `ON` command while the PLC is disconnected. Connect the PLC and confirm the command does not execute.
2. Publish `ON` and `OFF` while connected. Confirm the intended light changes and its state topic reports the current value.
3. Publish commands, disconnect, then reconnect. Check for command replay and confirm a fresh state snapshot.
4. Publish duplicate commands to light topics. Confirm absolute commands do not toggle state and each topic controls its intended light. Confirm Hall group state follows the OR of its members.
5. Confirm discovery creates six devices, one per logical light, without duplicate entities. Rename a location or light and confirm the identity and topic stay unchanged while discovery updates.
6. With separate authorization, perform a manual end-to-end test using the LOCAL broker and all six logical lights with Home Assistant. Observe only the exact LOCAL namespace externally. The shared `homeassistant/status` topic affects other clients, so do not publish to it without separate authorization; no automatic test should publish there. Publish exact `online` at QoS 1 with `retain=false`. Observe fresh retained Discovery publications and STATE publications for current PLC state, then confirm Ready. Publish `online` again after Ready and confirm another Discovery and STATE replay in the same MQTT session, without duplicate devices or entities. Send a burst of `online` messages during active Discovery and during STATE synchronization; confirm progress does not reset, livelock, or produce a trailing replay. Confirm SET commands still operate, STATE reflects actual PLC values, command priority remains physical over MQTT as implemented, and Availability remains online.
7. Change a light ID, then remove a light. In LOCAL, verify broker-observed reconciliation for the current own namespace, including after a full PLC download. Do not infer settlement from publish acceptance alone.
8. Disconnect the broker and observe availability. Treat PLC Stop detection as unproven unless an independent watchdog is separately tested.
9. Confirm ACLs reject reads and writes outside each identity's allowed topics, including cross-namespace discovery writes.

Do not enable commands on PROD until broker behavior and ACLs pass commissioning. No broker acceptance results are recorded here.

## Home Assistant setup

For Home Assistant OS, install and configure the official Mosquitto add-on. Create dedicated broker accounts and configure the Home Assistant MQTT integration. Do not put real credentials in examples or source control. HAOS and Mosquitto settings are outside this project and have not been changed.

If the broker uses TLS, configure trusted CA access for the Home Assistant MQTT integration. Confirm the add-on TLS settings before choosing a port or CA path.

The PLC marks internal configuration fields with `TcNoSymbol` to reduce symbol export. This is not encryption and does not prove protection in memory or in a boot project. Protect files and runtime access accordingly.

## Configuration errors

The loader fails closed if it cannot read or parse either file. `MAIN.nMqttConfigError` identifies these stages: 1 means open, 2 means read, 3 means size or NUL byte, 4 means close, 5 means parse, and 6 means an empty path. `MAIN.nMqttSystemError` exposes the underlying ADS error code without file contents. Correct the files or access, then restart the PLC application only with separate authorization. The loader does not automatically retry after an error.

## Verification and authorization

Follow the [TwinCAT XAE workflow](twincat-xae.md) for scoped rebuilds and configurations. Rebuilds are offline checks and do not prove runtime or broker behavior.

Run simulation and TcUnit only in the authorized LOCAL session. Follow [Workshop setup and checks](workshop-light.md) for target checks and online authorization.

Every LOCAL or PROD online operation requires separate user authorization for the target and action. This includes Login, Download, Start, Stop, Logout and online reads.

No Home Assistant OS, Mosquitto, production broker, or PROD runtime changes are authorized by this document. Any platform or runtime operation requires separate user authorization for the target and action. Never deploy automatically to PROD.
