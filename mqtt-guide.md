# MQTT Classroom Environmental Monitoring System

## Laboratory Instructional Guide

**Course:** ITEC 111 - Integrated Programming and Technologies 2  
**Topic:** MQTT and Asynchronous Messaging  
**Platform:** Mosquitto, Node.js, JavaScript ES modules, MQTT.js  
**Suggested duration:** 3-4 hours  
**Companion activity:** [Topic 12 Laboratory Exercise](topic-12-mqtt-classroom-monitoring-laboratory-exercise.md)

## Overview

In this collaborative laboratory, the class will operate one simulated environmental monitoring system. Student groups take different MQTT roles rather than building isolated copies of the same program.

- **Broker host:** runs Mosquitto and coordinates the topic namespace.
- **Sensor groups:** each runs a Node.js publisher with a unique device ID.
- **Monitoring group:** runs a Node.js subscriber that receives all telemetry and status messages.
- **Controller group:** publishes commands to particular devices.
- **Optional dashboard group:** builds a Node.js MQTT-to-WebSocket bridge and browser dashboard.

All communication stays on one computer or the classroom local network. No cloud broker or public internet service is required.

## Learning Outcomes

After completing the laboratory, students should be able to:

1. Configure a local MQTT broker and verify client connectivity.
2. Explain broker, publisher, subscriber, Topic Name, Topic Filter, and payload roles.
3. Publish validated JSON telemetry from several simulated devices.
4. Use exact, single-level (`+`), and multi-level (`#`) subscriptions.
5. Select and observe MQTT QoS levels.
6. use retained messages to represent the latest device state.
7. Configure a Last Will and Testament for unexpected disconnection.
8. Send commands to a specific device and observe its response.
9. Test reconnection without confusing a graceful shutdown with a failure.
10. Compare MQTT topic-based distribution with WebSocket broadcasting.

## Collaborative Group Assignment

The instructor may adapt the table to the class size.

| Group | Role | Primary topic access | Required output |
|---|---|---|---|
| Group 1 | Broker and integration team | all laboratory topics | running Mosquitto broker and connection information |
| Groups 2-4 | Sensor publisher teams | own device subtree | one simulator per group with a unique device ID |
| Group 5 | Monitoring team | `itec111/classroom/#` | terminal monitor showing telemetry, status, commands, and metadata |
| Group 6 | Controller team | command topics | commands addressed to individual devices |
| Group 7, optional | Dashboard team | telemetry and status filters | Node.js MQTT/WebSocket bridge and browser dashboard |

### Small-class alternative

If there are fewer groups, combine roles:

- Group 1: broker and monitor;
- Groups 2-3: sensors;
- Group 4: controller and optional dashboard.

### One-computer alternative

Every group uses a separate terminal on the same laboratory computer and connects to:

```text
mqtt://localhost:1883
```

### Classroom-LAN alternative

One designated computer runs Mosquitto. Other groups connect to that computer's private IPv4 address:

```text
mqtt://192.168.x.x:1883
```

This remains local-only because traffic stays on the classroom network. The broker host may need to allow inbound TCP port `1883` through its firewall. Do not expose the laboratory broker to the public internet.

## System Architecture

```text
Sensor Group A ---- telemetry/status ----\
Sensor Group B ---- telemetry/status -----\
Sensor Group C ---- telemetry/status ------> Mosquitto broker
Controller Group -- device commands ------/        |
                                                   +--> Monitoring Group
                                                   |
                                                   +--> Dashboard bridge --WebSocket--> Browser
```

MQTT clients communicate through the broker. The optional browser does not connect directly to raw MQTT/TCP; the Node.js bridge subscribes through MQTT and broadcasts selected events through WebSockets.

## Shared Topic Contract

Use this root namespace for the whole class:

```text
itec111/classroom
```

| Topic pattern | Publisher | Subscriber | Purpose | QoS | Retained |
|---|---|---|---|---:|---|
| `itec111/classroom/devices/{deviceId}/telemetry` | sensor | monitor/dashboard | temperature and humidity reading | 1 | no |
| `itec111/classroom/devices/{deviceId}/status` | sensor or broker Will | monitor/dashboard | `online` or `offline` state | 1 | yes |
| `itec111/classroom/devices/{deviceId}/commands` | controller | addressed sensor | device command | 1 | no |
| `itec111/classroom/devices/{deviceId}/events` | sensor | monitor/dashboard | command result or device event | 1 | no |

### Required device IDs

The instructor assigns one lowercase, hyphenated ID to each sensor group:

```text
group-02-sensor
group-03-sensor
group-04-sensor
```

Never reuse a device ID or MQTT Client ID while both clients are connected. A broker normally disconnects the existing connection when another client connects with the same Client ID.

## Message Contracts

### Telemetry payload

```json
{
  "messageId": "06f8fb43-6f96-4b45-8894-fd128260ccce",
  "deviceId": "group-02-sensor",
  "observedAt": "2026-09-25T02:30:00.000Z",
  "sequenceNo": 7,
  "temperatureC": 28.4,
  "relativeHumidityPct": 71.2
}
```

### Status payload

```json
{
  "deviceId": "group-02-sensor",
  "state": "online",
  "changedAt": "2026-09-25T02:29:45.000Z"
}
```

The Will payload uses the same structure with `state` set to `offline`. Its `changedAt` value is created when the client connects, so it describes when the Will was prepared rather than the exact failure-detection time.

### Command payload

```json
{
  "commandId": "06537ac6-e57c-45bb-a7e6-d0915317cb43",
  "type": "setInterval",
  "intervalMs": 5000,
  "issuedAt": "2026-09-25T02:32:00.000Z",
  "issuedBy": "group-06-controller"
}
```

Supported commands:

| Type | Additional field | Behavior |
|---|---|---|
| `sampleNow` | none | immediately publishes one reading |
| `setInterval` | `intervalMs` from `1000` to `30000` | changes the publication interval |

### Event payload

```json
{
  "eventId": "09e23278-f373-4905-afcc-fd5c59f58e85",
  "deviceId": "group-02-sensor",
  "type": "commandApplied",
  "commandId": "06537ac6-e57c-45bb-a7e6-d0915317cb43",
  "occurredAt": "2026-09-25T02:32:00.450Z",
  "detail": "Publishing interval changed to 5000 ms"
}
```

---

## Part 1 - Install and Start Mosquitto

Install Mosquitto using the package appropriate for the laboratory operating system. Confirm the executables:

```bash
mosquitto -h
mosquitto_pub -h
mosquitto_sub -h
```

Create `mosquitto-lab.conf` on the broker computer:

```conf
listener 1883
allow_anonymous true
persistence false
log_type all
```

Start the broker from the directory containing the configuration:

```bash
mosquitto -c mosquitto-lab.conf -v
```

`allow_anonymous true` is acceptable only for this isolated classroom exercise. A production broker must require authentication, authorization, and encrypted transport.

### Broker smoke test

Open two terminals. In the first:

```bash
mosquitto_sub -h localhost -t 'itec111/classroom/test' -v
```

In the second:

```bash
mosquitto_pub -h localhost -t 'itec111/classroom/test' -m 'broker-ready'
```

Replace `localhost` with the broker host's private IPv4 address for the classroom-LAN arrangement.

### Checkpoint 1

The subscriber must print:

```text
itec111/classroom/test broker-ready
```

Record the broker URL and distribute it to every group.

---

## Part 2 - Create the Shared Node.js Project

Each programming group creates its own copy of this structure:

```text
mqtt-classroom-lab/
├── package.json
├── evidence/
└── src/
    ├── controller.js
    ├── monitor.js
    └── sensor.js
```

Initialize the project:

```bash
mkdir mqtt-classroom-lab
cd mqtt-classroom-lab
npm init -y
npm install mqtt
mkdir src evidence
```

Update `package.json`:

```json
{
  "name": "mqtt-classroom-lab",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "sensor": "node src/sensor.js",
    "monitor": "node src/monitor.js",
    "controller": "node src/controller.js"
  },
  "dependencies": {
    "mqtt": "^5.0.0"
  }
}
```

Keep the version generated by npm if it is newer and tested by the class.

### Runtime configuration

Programs read these environment variables:

| Variable | Default | Purpose |
|---|---|---|
| `MQTT_URL` | `mqtt://localhost:1883` | broker URL |
| `DEVICE_ID` | none | required unique sensor identifier |
| `PUBLISH_INTERVAL_MS` | `3000` | initial telemetry interval |
| `TARGET_DEVICE_ID` | none | command recipient |

Example on macOS/Linux:

```bash
MQTT_URL=mqtt://localhost:1883 DEVICE_ID=group-02-sensor npm run sensor
```

PowerShell equivalent:

```powershell
$env:MQTT_URL='mqtt://localhost:1883'
$env:DEVICE_ID='group-02-sensor'
npm run sensor
```

---

## Part 3 - Build the Sensor Publisher

Create `src/sensor.js`:

```javascript
import { randomUUID } from 'node:crypto';
import mqtt from 'mqtt';

const brokerUrl = process.env.MQTT_URL || 'mqtt://localhost:1883';
const deviceId = process.env.DEVICE_ID;
const root = 'itec111/classroom/devices';

if (!deviceId || !/^[a-z0-9]+(?:-[a-z0-9]+)*$/.test(deviceId)) {
  throw new Error('Set DEVICE_ID to a unique lowercase, hyphenated value.');
}

let intervalMs = Number(process.env.PUBLISH_INTERVAL_MS || 3000);
if (!Number.isInteger(intervalMs) || intervalMs < 1000 || intervalMs > 30000) {
  throw new Error('PUBLISH_INTERVAL_MS must be an integer from 1000 to 30000.');
}

const topics = {
  telemetry: `${root}/${deviceId}/telemetry`,
  status: `${root}/${deviceId}/status`,
  commands: `${root}/${deviceId}/commands`,
  events: `${root}/${deviceId}/events`
};

const offlinePayload = JSON.stringify({
  deviceId,
  state: 'offline',
  changedAt: new Date().toISOString()
});

const client = mqtt.connect(brokerUrl, {
  clientId: `sensor-${deviceId}`,
  clean: true,
  reconnectPeriod: 2000,
  will: {
    topic: topics.status,
    payload: offlinePayload,
    qos: 1,
    retain: true
  }
});

let sequenceNo = 0;
let publishTimer;

function publishJson(topic, value, options = {}) {
  client.publish(topic, JSON.stringify(value), options, error => {
    if (error) console.error('Publish failed:', error.message);
  });
}

function createReading() {
  return {
    messageId: randomUUID(),
    deviceId,
    observedAt: new Date().toISOString(),
    sequenceNo: sequenceNo++,
    temperatureC: Number((24 + Math.random() * 8).toFixed(1)),
    relativeHumidityPct: Number((55 + Math.random() * 30).toFixed(1))
  };
}

function publishReading() {
  const reading = createReading();
  publishJson(topics.telemetry, reading, { qos: 1, retain: false });
  console.log('Published telemetry:', reading);
}

function startPublishing() {
  clearInterval(publishTimer);
  publishTimer = setInterval(publishReading, intervalMs);
}

function publishEvent(command, detail) {
  publishJson(topics.events, {
    eventId: randomUUID(),
    deviceId,
    type: 'commandApplied',
    commandId: command.commandId,
    occurredAt: new Date().toISOString(),
    detail
  }, { qos: 1, retain: false });
}

client.on('connect', session => {
  console.log(`Connected to ${brokerUrl}; sessionPresent=${session.sessionPresent}`);

  client.subscribe(topics.commands, { qos: 1 }, error => {
    if (error) console.error('Command subscription failed:', error.message);
  });

  publishJson(topics.status, {
    deviceId,
    state: 'online',
    changedAt: new Date().toISOString()
  }, { qos: 1, retain: true });

  startPublishing();
});

client.on('message', (topic, payload) => {
  if (topic !== topics.commands) return;

  let command;
  try {
    command = JSON.parse(payload.toString());
  } catch {
    console.warn('Ignored command with invalid JSON.');
    return;
  }

  if (
    !command ||
    typeof command !== 'object' ||
    Array.isArray(command) ||
    typeof command.commandId !== 'string' ||
    typeof command.type !== 'string' ||
    typeof command.issuedAt !== 'string'
  ) {
    console.warn('Ignored command that does not satisfy the command contract.');
    return;
  }

  if (command.type === 'sampleNow') {
    publishReading();
    publishEvent(command, 'Immediate sample published');
    return;
  }

  if (
    command.type === 'setInterval' &&
    Number.isInteger(command.intervalMs) &&
    command.intervalMs >= 1000 &&
    command.intervalMs <= 30000
  ) {
    intervalMs = command.intervalMs;
    startPublishing();
    publishEvent(command, `Publishing interval changed to ${intervalMs} ms`);
    return;
  }

  console.warn('Ignored unsupported or invalid command:', command);
});

client.on('reconnect', () => console.log('Reconnecting...'));
client.on('offline', () => console.log('Client is offline.'));
client.on('error', error => console.error('MQTT error:', error.message));

async function shutdown(signal) {
  console.log(`Received ${signal}; disconnecting gracefully.`);
  clearInterval(publishTimer);

  publishJson(topics.status, {
    deviceId,
    state: 'offline',
    changedAt: new Date().toISOString()
  }, { qos: 1, retain: true });

  setTimeout(() => client.end(false, () => process.exit(0)), 250);
}

process.on('SIGINT', () => shutdown('SIGINT'));
process.on('SIGTERM', () => shutdown('SIGTERM'));
```

### Start several sensor groups

Each group runs one command with a unique ID:

```bash
DEVICE_ID=group-02-sensor npm run sensor
DEVICE_ID=group-03-sensor npm run sensor
DEVICE_ID=group-04-sensor npm run sensor
```

Add `MQTT_URL=mqtt://192.168.x.x:1883` before the command when using the classroom-LAN broker.

### Checkpoint 2

The broker log should show separate clients. Every sensor terminal should publish a reading every few seconds.

---

## Part 4 - Build the Monitoring Subscriber

Create `src/monitor.js`:

```javascript
import mqtt from 'mqtt';

const brokerUrl = process.env.MQTT_URL || 'mqtt://localhost:1883';
const filter = 'itec111/classroom/devices/#';

const client = mqtt.connect(brokerUrl, {
  clientId: `monitor-${process.pid}`,
  clean: true,
  reconnectPeriod: 2000
});

function isValidForTopic(topic, value) {
  if (!value || typeof value !== 'object' || Array.isArray(value)) return false;

  if (topic.endsWith('/telemetry')) {
    return typeof value.messageId === 'string' &&
      typeof value.deviceId === 'string' &&
      typeof value.observedAt === 'string' &&
      Number.isInteger(value.sequenceNo) &&
      typeof value.temperatureC === 'number' &&
      typeof value.relativeHumidityPct === 'number';
  }

  if (topic.endsWith('/status')) {
    return typeof value.deviceId === 'string' &&
      ['online', 'offline'].includes(value.state) &&
      typeof value.changedAt === 'string';
  }

  if (topic.endsWith('/commands')) {
    return typeof value.commandId === 'string' &&
      typeof value.type === 'string' &&
      typeof value.issuedAt === 'string';
  }

  if (topic.endsWith('/events')) {
    return typeof value.eventId === 'string' &&
      typeof value.deviceId === 'string' &&
      typeof value.commandId === 'string' &&
      typeof value.occurredAt === 'string';
  }

  return false;
}

client.on('connect', () => {
  client.subscribe(filter, { qos: 1 }, (error, grants) => {
    if (error) {
      console.error('Subscription failed:', error.message);
      return;
    }
    console.log('Subscribed:', grants);
  });
});

client.on('message', (topic, payload, packet) => {
  const receivedAt = new Date().toISOString();
  let decoded;

  try {
    decoded = JSON.parse(payload.toString());
  } catch {
    decoded = payload.toString();
  }

  const contractValid = isValidForTopic(topic, decoded);

  console.log({
    receivedAt,
    topic,
    qos: packet.qos,
    retain: packet.retain,
    duplicate: packet.dup,
    contractValid,
    payload: decoded
  });
});

client.on('reconnect', () => console.log('Monitor reconnecting...'));
client.on('error', error => console.error('MQTT error:', error.message));

process.on('SIGINT', () => client.end(false, () => process.exit(0)));
```

Run it:

```bash
npm run monitor
```

### Exact and wildcard subscription tests

Temporarily change `filter` and restart the monitor for each test:

```text
itec111/classroom/devices/group-02-sensor/telemetry
itec111/classroom/devices/+/telemetry
itec111/classroom/devices/#
```

Record which messages each filter receives.

### Checkpoint 3

The monitor must show readings from at least three unique device IDs and print each message's topic, payload, received time, QoS, retain flag, and duplicate flag.

---

## Part 5 - Build the Controller

Create `src/controller.js`:

```javascript
import { randomUUID } from 'node:crypto';
import mqtt from 'mqtt';

const brokerUrl = process.env.MQTT_URL || 'mqtt://localhost:1883';
const targetDeviceId = process.env.TARGET_DEVICE_ID;
const [commandType = 'sampleNow', rawInterval] = process.argv.slice(2);

if (!targetDeviceId) {
  throw new Error('Set TARGET_DEVICE_ID to the device that should receive the command.');
}

const topic = `itec111/classroom/devices/${targetDeviceId}/commands`;
const command = {
  commandId: randomUUID(),
  type: commandType,
  issuedAt: new Date().toISOString(),
  issuedBy: 'classroom-controller',
  ...(commandType === 'setInterval' ? { intervalMs: Number(rawInterval) } : {})
};

const client = mqtt.connect(brokerUrl, {
  clientId: `controller-${process.pid}`,
  clean: true
});

client.on('connect', () => {
  client.publish(topic, JSON.stringify(command), { qos: 1, retain: false }, error => {
    if (error) {
      console.error('Command failed:', error.message);
      client.end();
      return;
    }

    console.log('Command published:', { topic, command });
    client.end();
  });
});

client.on('error', error => console.error('MQTT error:', error.message));
```

Send commands:

```bash
TARGET_DEVICE_ID=group-02-sensor npm run controller -- sampleNow
TARGET_DEVICE_ID=group-03-sensor npm run controller -- setInterval 5000
```

### Checkpoint 4

Only the addressed device should apply the command. The monitoring group must observe both the command and the device's resulting event.

---

## Part 6 - Test Retained Status and Last Will

### Retained-message test

1. Keep the sensor groups running until each publishes retained `online` status.
2. Stop the monitor.
3. Start the monitor again.
4. Observe that the latest retained status arrives immediately with `retain: true`.
5. Confirm that old telemetry readings are not replayed because telemetry uses `retain: false`.

### Graceful-disconnect test

Press `Ctrl+C` in one sensor terminal. Its shutdown handler publishes retained `offline` before closing normally. The broker should not need to publish the Will.

### Unexpected-disconnect test

To test the Will, terminate a sensor without allowing its shutdown handler to run. Use the operating system's process manager or close/kill the terminal process. Do not use this technique on unrelated processes.

The broker detects the lost connection and publishes the configured retained `offline` Will. Detection may not be immediate because MQTT keep-alive and the operating system influence the delay.

### Checkpoint 5

Capture evidence of:

- immediate retained state after a fresh monitor starts;
- graceful offline publication; and
- broker-published offline Will after an unexpected device loss.

---

## Part 7 - Test QoS and Reconnection

### QoS observations

Run three controlled trials by changing the telemetry publish QoS and the monitor's requested QoS:

| Trial | Published QoS | Subscription maximum QoS | Expected delivered QoS |
|---:|---:|---:|---:|
| 1 | 0 | 2 | 0 |
| 2 | 1 | 0 | 0 |
| 3 | 1 | 1 | 1 |

MQTT.js reports the delivered QoS in `packet.qos`. Do not claim that QoS 1 always produces a visible duplicate; it permits duplicates but a healthy local run may deliver only once.

### Reconnection test

1. Run the monitor and at least two sensors.
2. Stop Mosquitto for 10-15 seconds.
3. Observe `offline` and `reconnect` events in client terminals.
4. Restart Mosquitto with the same configuration.
5. Confirm that clients reconnect and subscriptions are restored.
6. Record which messages were lost while the broker was unavailable.

Because the exercise uses `clean: true` and no broker persistence, it does not promise offline message storage.

---

## Part 8 - Optional Browser Dashboard Group

The optional dashboard adds Express and `ws`:

```bash
npm install express ws
```

The team builds:

```text
Browser <-- WebSocket --> Node.js bridge <-- MQTT --> Mosquitto
```

Minimum requirements:

- subscribe to `itec111/classroom/devices/+/telemetry` and `.../+/status`;
- decode and validate incoming JSON;
- keep the latest reading and state per device in a `Map`;
- broadcast a normalized event to every open browser WebSocket;
- send the current device snapshot to a newly connected browser;
- render device ID, temperature, humidity, state, observed time, and received time safely with `textContent`;
- show MQTT and WebSocket connection states separately.

The dashboard is optional because it combines two protocols. The MQTT broker performs topic-based routing and retained-state handling; the WebSocket server broadcasts events to currently connected browsers unless the team implements its own filtering and state replay.

---

## Part 9 - Whole-Class Demonstration Sequence

1. Broker group announces the broker URL.
2. Monitoring group starts the wildcard subscriber.
3. Sensor groups connect one at a time; the class observes retained `online` states.
4. Each sensor publishes at least five readings.
5. Monitor tests exact, `+`, and `#` filters.
6. Controller sends `sampleNow` to one device.
7. Controller changes another device's interval.
8. A fresh monitor demonstrates retained status.
9. One sensor exits gracefully.
10. Another sensor is stopped unexpectedly to demonstrate its Will.
11. Broker is restarted to test client reconnection.
12. Optional dashboard group demonstrates MQTT-to-WebSocket delivery.
13. Class compares broker topic matching with WebSocket broadcasting.

## Troubleshooting

| Symptom | Likely cause | Check |
|---|---|---|
| `ECONNREFUSED` | broker is stopped or URL/port is wrong | broker log, `MQTT_URL`, port `1883` |
| Remote group cannot connect | listener/firewall/network issue | broker configuration and private IP |
| One client repeatedly disconnects | duplicate MQTT Client ID | unique `DEVICE_ID` and `clientId` |
| Monitor receives nothing | filter does not match or subscription failed | print grants; compare every topic level |
| JSON parsing fails | producer sent plain text or malformed JSON | inspect raw payload and publisher code |
| Retained status is missing | publish did not set `retain: true` | publisher options and broker log |
| Will does not appear | client disconnected gracefully or broker has not detected loss | use unexpected termination and wait for keep-alive detection |
| All devices respond to one command | controller published to a shared/wildcard-like design | publish to the exact device command topic |
| Browser misses earlier telemetry | WebSocket server has no history by default | maintain and send a latest-state snapshot |

## Safety and Production Limitations

- This laboratory permits anonymous plain-text MQTT only on an isolated local network.
- Do not transmit personal data, passwords, access tokens, or real production telemetry.
- A production deployment should use TLS, authenticated identities, topic-level authorization, payload validation, rate limits, monitoring, and durable broker configuration where required.
- Client-supplied timestamps and values are untrusted input.
- Retained messages are latest-state records, not an audit trail or time-series database.
- QoS is a transport-level delivery choice; it does not make application side effects transactional.

## Completion Checklist

- [ ] Mosquitto broker passed the smoke test.
- [ ] At least three unique sensor clients published telemetry.
- [ ] Monitor tested exact, `+`, and `#` subscriptions.
- [ ] Controller addressed commands to two particular devices.
- [ ] Retained state appeared for a fresh subscriber.
- [ ] QoS trials recorded the delivered QoS.
- [ ] Graceful and unexpected disconnects were distinguished.
- [ ] Clients reconnected after a broker restart.
- [ ] MQTT and WebSocket behavior was compared.
- [ ] Evidence contains no secrets or personal information.

## References

- *MQTT Version 5.0*, `refs/MQTT-v5.0.pdf`: CONNECT and Will, pp. 30-44; PUBLISH and retained messages, pp. 54-61; SUBSCRIBE, pp. 71-76; QoS, pp. 94-98; topics and subscriptions, pp. 98-103.
- MQTT.js official documentation: <https://github.com/mqttjs/MQTT.js>
- Eclipse Mosquitto documentation: <https://mosquitto.org/documentation/>
