# Topic 12 Laboratory Exercise

## Collaborative MQTT Classroom Environmental Monitoring System

**Course:** ITEC 111 - Integrated Programming and Technologies 2  
**Technology:** Mosquitto, Node.js, JavaScript ES modules, MQTT.js  
**Companion guide:** [MQTT Classroom Monitoring Laboratory Guide](topic-12-mqtt-classroom-monitoring-laboratory-guide.md)

## Intended Outcome

As a class, build and test a local MQTT system in which multiple student groups publish simulated temperature and humidity readings, another group monitors the shared topic hierarchy, and a controller group sends commands to particular devices. Demonstrate topic matching, payload handling, QoS, retained status, reconnection, and Last Will and Testament behavior.

The optional dashboard group may bridge MQTT messages to browser clients through a Node.js WebSocket server.

## Collaboration Rules

1. The instructor assigns each group exactly one primary role.
2. Sensor groups receive unique device IDs and must not change another group's namespace.
3. The broker group provides one local broker URL to the class.
4. All groups use the shared message and topic contracts from the instructional guide.
5. Every group observes the whole-class integration demonstration, even if it did not implement every component.
6. Each group records its assigned-role answers and evidence in this exercise document.
7. The class must not use a public broker, real credentials, personal data, or production device data.

## Assigned Roles

Complete this table before starting.

| Item | Assignment |
|---|---|
| Group number and members |  |
| Primary role |  |
| MQTT broker URL |  |
| Assigned device ID, if applicable |  |
| Partner/dependent groups |  |
| Evidence recorder |  |

---

## Shared Integration Tasks

Every group participates in Tasks 1-4 and 9-12. Complete the implementation tasks assigned to your role in Tasks 5-8.

### Task 1 - Confirm the shared contract

Before coding, record the exact topic patterns for:

- telemetry;
- retained status;
- device commands; and
- device events.

Copy one example payload for each message type. Identify which fields provide identity, ordering, observation time, command correlation, and measurement units.

**Required evidence:** completed contract table in this exercise document.

### Task 2 - Verify the broker

Use `mosquitto_pub` and `mosquitto_sub` to perform the smoke test on `itec111/classroom/test`.

**Required evidence:** broker log or two-terminal screenshot showing `broker-ready` with its topic.

### Task 3 - Configure unique client identity

Use a unique MQTT Client ID for every simultaneously connected process. Sensor groups must use their instructor-assigned `DEVICE_ID`.

Demonstrate what happens when two temporary clients deliberately use the same Client ID, then restore unique IDs. Do not perform this collision test during evidence collection by other groups.

**Required evidence:** brief observation of the collision and explanation of why the broker disconnected one client.

### Task 4 - Validate message data

Each Node.js consumer must:

- convert the MQTT payload `Buffer` to text;
- catch JSON parsing failures;
- verify the expected message fields and value types before use; and
- continue running after a malformed test message.

**Required evidence:** one valid and one rejected malformed message.

---

## Role Implementation Tasks

### Task 5 - Broker and integration group

The broker group must:

1. create an isolated classroom Mosquitto configuration on port `1883`;
2. run Mosquitto with verbose logging;
3. distribute the correct local broker URL;
4. confirm connections from at least three sensor clients and one monitor;
5. keep a timeline of the whole-class demonstration;
6. stop and restart the broker during the reconnection trial; and
7. explain why anonymous, unencrypted access is restricted to this isolated activity.

**Required evidence:** configuration file, broker startup log, connected-client evidence, and restart timeline.

### Task 6 - Sensor publisher groups

Each sensor group must implement one Node.js process that:

1. connects using its unique Client ID;
2. publishes temperature and humidity every 3 seconds initially;
3. uses the required telemetry JSON contract;
4. increments `sequenceNo` and creates a new UUID `messageId` per observation;
5. publishes telemetry with QoS 1 and `retain: false`;
6. publishes retained `online` status after connecting;
7. configures a retained `offline` Will before connecting;
8. subscribes only to its exact command topic;
9. supports `sampleNow` and validated `setInterval` commands;
10. publishes a correlated result on its event topic; and
11. logs connect, publish, command, offline, reconnect, and error events.

**Required evidence:** five consecutive readings, online status, one applied command, one event response, and one reconnect.

### Task 7 - Monitoring subscriber group

The monitoring group must implement a Node.js process that:

1. subscribes to the shared namespace;
2. prints receipt time, Topic Name, decoded payload, QoS, retain flag, and duplicate flag;
3. displays messages from at least three unique sensor devices;
4. performs exact, `+`, and `#` filter trials;
5. rejects malformed JSON without stopping;
6. identifies retained status received after a fresh subscription;
7. observes commands and correlated device events; and
8. logs offline and reconnect states during a broker restart.

**Required evidence:** annotated logs for the three filter trials and a retained status message with `retain: true`.

### Task 8 - Controller group

The controller group must implement a Node.js command publisher that:

1. accepts a target device ID through configuration;
2. creates a unique `commandId`;
3. includes `issuedAt` and `issuedBy`;
4. publishes only to the target device's exact command topic;
5. sends `sampleNow` to one sensor;
6. sends a valid `setInterval` to a second sensor;
7. sends one invalid command for rejection testing;
8. uses QoS 1 and `retain: false`; and
9. correlates each accepted command with the device event containing the same `commandId`.

**Required evidence:** three published commands, two matching result events, and proof that an unaddressed device did not change behavior.

### Optional Task 8A - Dashboard group

The dashboard group must implement:

1. a Node.js MQTT subscriber;
2. a WebSocket server for browser clients;
3. a `Map` containing the latest reading and state per device;
4. a snapshot sent when a browser connects;
5. live WebSocket broadcasting of normalized MQTT events;
6. a browser table or cards for all devices;
7. separate MQTT and WebSocket connection indicators; and
8. safe DOM updates using `textContent`.

**Required evidence:** at least three devices updating in one browser plus a second browser receiving live broadcasts.

---

## Protocol Behavior Tasks

### Task 9 - Topic-filter experiment

Complete the table from actual observations.

| Subscription filter | Expected matching messages | Actual matching devices/message types | Result |
|---|---|---|---|
| `itec111/classroom/devices/group-02-sensor/telemetry` | one device's telemetry only |  |  |
| `itec111/classroom/devices/+/telemetry` | every device's telemetry |  |  |
| `itec111/classroom/devices/#` | telemetry, status, commands, and events |  |  |
| `itec111/+/devices/+/status` | status with exactly one level in each `+` position |  |  |
| `itec111/classroom/+/telemetry` | no match for the defined telemetry hierarchy |  |  |

Explain each mismatch by counting topic levels.

### Task 10 - QoS experiment

Perform and record these trials. The monitor must use `packet.qos` rather than assuming the result.

| Trial | Publication QoS | Subscription maximum | Expected delivered QoS | Actual QoS | Notes |
|---:|---:|---:|---:|---:|---|
| 1 | 0 | 2 | 0 |  |  |
| 2 | 1 | 0 | 0 |  |  |
| 3 | 1 | 1 | 1 |  |  |

Then answer:

- Did QoS 1 visibly duplicate a message? Why does one run not prove duplicates are impossible?
- Which QoS would you choose for frequent environmental readings? For a rare safety alert? Justify both.

### Task 11 - Retained and Will experiment

Perform these in order:

1. Start a sensor and wait for retained `online` status.
2. Start a fresh monitor and record the immediate retained status.
3. Confirm that previous telemetry is not replayed.
4. gracefully stop one sensor and record its explicit offline status;
5. unexpectedly terminate a different sensor and wait for the broker-published Will;
6. start another fresh monitor and confirm that the latest retained state is `offline`.

Complete the comparison.

| Observation | Who published it? | When was it produced? | Retain flag observed | Meaning |
|---|---|---|---|---|
| Fresh subscriber receives `online` |  |  |  |  |
| Graceful `offline` |  |  |  |  |
| Unexpected-disconnect `offline` |  |  |  |  |

### Task 12 - Reconnection experiment

With at least two sensors and the monitor running:

1. record the last `sequenceNo` from each sensor;
2. stop the broker for 10-15 seconds;
3. record client offline/reconnect messages;
4. restart the broker;
5. verify resubscription and resumed telemetry;
6. determine whether any sequence numbers are missing; and
7. explain the result using the laboratory's clean-session and nonpersistent-broker configuration.

**Required evidence:** before, during, and after logs from at least one publisher and the monitor.

---

## Required Integration Test Matrix

The class performs the complete matrix. Each group records the tests relevant to its role plus Tests 1, 11, 12, 13, 14, and 15.

| No. | Scenario | Expected result | Actual result | Evidence |
|---:|---|---|---|---|
| 1 | CLI broker smoke test | subscriber receives topic and `broker-ready` |  |  |
| 2 | First sensor connects | retained `online` status appears |  |  |
| 3 | Three sensors run | unique IDs and interleaved telemetry appear |  |  |
| 4 | Exact telemetry filter | only the named device is received |  |  |
| 5 | `+` telemetry filter | telemetry from every one-level device ID is received |  |  |
| 6 | `#` device filter | all descendant message types are received |  |  |
| 7 | Valid `sampleNow` command | only target publishes an immediate reading and result event |  |  |
| 8 | Valid `setInterval` command | only target changes interval and confirms command ID |  |  |
| 9 | Invalid command | target ignores/rejects it without crashing |  |  |
| 10 | Malformed JSON publication | consumer reports invalid data and remains running |  |  |
| 11 | Fresh monitor starts | retained status arrives; old telemetry does not |  |  |
| 12 | Sensor exits gracefully | explicit offline status is received |  |  |
| 13 | Sensor stops unexpectedly | broker publishes offline Will |  |  |
| 14 | Broker restarts | clients reconnect; monitor resumes matching messages |  |  |
| 15 | QoS trials | delivered QoS matches the lower applicable level |  |  |
| 16 | Duplicate Client ID trial | previous client with that ID is disconnected |  |  |
| 17 | Optional dashboard | current snapshot and live updates appear in browser |  |  |

---

## Analysis Questions

Answer each in two to five technically specific sentences unless a trace or table is requested. Base answers on your implementation and captured evidence.

1. Trace one sensor reading from creation to the monitoring console. Explain serialization, Topic Name selection, broker topic-filter matching, transport, decoding, parsing, and the roles of `messageId`, `sequenceNo`, and timestamps.
2. Using evidence from the exact, `+`, and `#` trials, distinguish Topic Names from Topic Filters and explain why unique device IDs and MQTT Client IDs are necessary.
3. Explain why telemetry uses `retain: false` while status uses `retain: true`. Compare retained delivery, graceful offline status, and the Last Will message using evidence from the activity.
4. Explain the results of the three QoS trials and why QoS 1 consumers must tolerate duplicate messages. Describe how the message fields can help detect loss, duplication, or delay.
5. Trace a targeted command and its correlated result event, then explain what happened during the broker restart. Include the importance of command validation, exact command topics, reconnection settings, and at least three security controls needed in production.

## MQTT and WebSocket Analysis

Answer Questions 1-6. If the optional dashboard was implemented, support Answers 4-6 with dashboard evidence.

1. In the MQTT system, which component decides that a publication matches a subscriber: the sensor, broker, or monitoring client?
2. In the earlier room-chat WebSocket system, which component iterated over connections and selected broadcast recipients?
3. Compare MQTT topics/subscriptions with WebSocket rooms. Which behavior is protocol-defined, and which was application code?
4. Why does the dashboard bridge maintain a `Map` and send a snapshot to a newly connected browser?
5. What happens to a WebSocket broadcast when no browser is connected? Compare this with a retained MQTT status publication.
6. Complete the table:

| Dimension | MQTT activity | WebSocket room-chat activity |
|---|---|---|
| Central component |  |  |
| Routing selector |  |  |
| One-to-many behavior |  |  |
| Latest-state replay |  |  |
| Delivery levels |  |  |
| Unexpected client-loss mechanism |  |  |
| Best fit in this course |  |  |

## Payload and Topic Diagnosis

For each item, identify the problem, provide a correction, and explain the likely effect.

1. A publisher sends to `itec111/classroom/devices/+/telemetry`.
2. Two groups both use `DEVICE_ID=classroom-sensor`.
3. A status publication uses `retain: false`.
4. Every telemetry reading uses `retain: true` on the same topic.
5. The controller publishes `setInterval` to `itec111/classroom/devices/#`.
6. A sensor accepts `{ "type": "setInterval", "intervalMs": 0 }`.
7. A consumer calls `JSON.parse(payload)` without converting or catching errors.
8. A QoS 1 handler inserts every received `messageId` into a database without duplicate checking.
9. A Will payload says `online`.
10. A student claims the Will test succeeded after pressing `Ctrl+C`, even though the program publishes its own graceful offline message.

Then correct this invalid telemetry payload:

```json
{
  "deviceId": 2,
  "observedAt": "today",
  "sequenceNo": -1,
  "temperature": "29.4",
  "relativeHumidityPct": 140
}
```

## Reflection Questions

1. What is your main takeaway from this module about using MQTT for communication and monitoring systems?

---

## Exercise Document Requirements

Each group submits only this completed exercise document. Record the following directly in the document:

1. answers for the assigned-role tasks;
2. completed required integration test rows;
3. answers to the Analysis Questions, MQTT and WebSocket Analysis, Payload and Topic Diagnosis, and Reflection Questions; and
4. labeled evidence for each required task or test, with brief captions and corresponding task or test numbers.

Paste or embed evidence in the appropriate section of the exercise document. Do not submit source code, configuration files, package files, ZIP archives, or separate evidence files.

### Minimum evidence by role

| Role | Minimum evidence |
|---|---|
| Broker | smoke test, configuration, multiple connections, restart timeline |
| Sensor | five readings, retained online, accepted command, event, reconnect, unexpected stop |
| Monitor | three devices, exact/`+`/`#` filters, retained flag, Will, reconnection |
| Controller | two valid targeted commands, correlated events, one rejected invalid command |
| Dashboard | three device views, retained snapshot, live update in two browsers, separate connection states |

Do not include credentials, personal information, or unrelated process logs in the exercise document or its evidence.

---

## Rubric

| Criterion | Excellent | Satisfactory | Needs improvement | Points |
|---|---|---|---|---:|
| Shared MQTT contract | Exact topics and interoperable payloads; identity, time, units, and correlation are clear | Minor contract inconsistency that does not prevent integration | Topics or payloads prevent group interoperability | 15 |
| Assigned-role implementation | All required behaviors work reliably and errors are handled | Core role works with one or two minor weaknesses | Role is incomplete or depends on manual message fabrication | 25 |
| Topic-filter understanding | Exact, `+`, and `#` trials are correct and well explained | Trials work with a minor explanation error | Filters are guessed or evidence contradicts claims | 10 |
| QoS, retained, and Will behavior | All are demonstrated and accurately distinguished | Most behaviors work with minor evidence gaps | Features are configured but not meaningfully tested | 15 |
| Commands and inter-group integration | Targeted command and correlated result work across groups | Integration works with minor manual intervention | Commands affect wrong clients or cannot be correlated | 10 |
| Reconnection and failure testing | Broker/device failures are distinguished and recovery is documented | Recovery works but analysis is incomplete | Test is missing or normal shutdown is misreported as failure | 10 |
| Analysis and protocol comparison | Technically precise answers tied to evidence; MQTT/WebSocket distinctions are clear | Generally correct but partly generic | Major misconceptions or missing answers | 10 |
| Answers and evidence | Answers and labeled evidence are organized, complete, and directly recorded in the exercise document | Complete with minor organization gaps | Substantial answers or evidence are missing | 5 |
| **Total** |  |  |  | **100** |

### Optional dashboard bonus

Award up to **10 bonus points** for a reliable MQTT-to-WebSocket dashboard that maintains latest device state, sends a new-browser snapshot, safely renders data, and clearly separates MQTT connection state from WebSocket connection state. The total recorded grade remains subject to the instructor's grading policy.

### Critical deductions

- Reusing another active group's MQTT Client ID: no credit for client-identity testing until corrected.
- Publishing commands to every device instead of an exact target: no credit for targeted command behavior.
- Claiming retained telemetry is message history: return the retained-message analysis for correction.
- Reporting a graceful shutdown as a demonstrated Will: no credit for the unexpected-disconnect test.
- Consumer crashes on malformed classroom input: no credit for payload-error handling.
- Public broker, exposed classroom broker, or submitted secrets: stop the activity and follow the instructor's security/submission policy.

