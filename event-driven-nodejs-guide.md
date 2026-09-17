# Event-Driven File Monitoring with Node.js

## Overview

In this activity, a Node.js publisher monitors a text file. When the file's content changes, the publisher emits a `file.changed` event. Independent subscribers react to the event without being called directly by the publisher.

The subscribers will:

- Display an event notification in the terminal
- Record the event in an audit log
- Calculate text statistics
- Write an uppercase version of the text to another file

```text
data/input.txt changes
          ↓
File watcher detects the change
          ↓
Publisher reads and compares the content
          ↓
Publisher emits file.changed
          ├── Console subscriber
          ├── Audit subscriber
          ├── Statistics subscriber
          └── Uppercase-output subscriber
```

This example uses only built-in Node.js modules. It does not require Express, MQTT, RabbitMQ, Kafka, or another message broker.

## Objectives

After completing the demonstration, students should be able to:

1. Explain the roles of publishers, events, event buses, and subscribers.
2. Monitor a file using Node.js `watch()`.
3. Publish a structured event with `EventEmitter.emit()`.
4. Subscribe to an event with `EventEmitter.on()`.
5. Prevent duplicate processing when the file system reports several notifications for one edit.
6. Separate change detection from the actions triggered by the change.
7. Handle file and subscriber errors without terminating the application unexpectedly.
8. Explain the limitations of an in-process event bus.

## Behavior

The application watches:

```text
data/input.txt
```

After a meaningful change, it creates an event similar to:

```json
{
  "event_id": "d14a4fe5-7af7-47d1-a95e-df473469dd07",
  "type": "file.changed",
  "occurred_at": "2026-09-13T08:30:00.000Z",
  "source": "text-file-publisher",
  "data": {
    "path": "data/input.txt",
    "size_bytes": 54,
    "content_hash": "...",
    "content": "The new contents of the text file"
  }
}
```

The event describes a fact that already happened. The name uses past tense: `file.changed`.

---

## Step 1 - Create the Project

Create and enter the project directory:

```bash
mkdir file-events-demo
cd file-events-demo
npm init -y
```

No external npm dependency is required. The application uses these built-in modules:

- `node:events`
- `node:fs`
- `node:fs/promises`
- `node:crypto`
- `node:path`

---

## Step 2 - Configure ES Modules

Open `package.json`. Add `"type": "module"` and replace the scripts section:

```json
{
  "name": "file-events-demo",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "node src/app.js"
  }
}
```

Keep other fields that npm generated.

Do not use Node's watch mode for this demonstration. The application itself must continue running while it watches `data/input.txt`.

---

## Step 3 - Create the Project Structure

Create these directories and files:

```text
file-events-demo/
├── package.json
├── data/
│   └── input.txt
├── output/
└── src/
    ├── app.js
    ├── event-bus.js
    ├── publisher.js
    └── subscribers.js
```

Add initial text to `data/input.txt`:

```text
This is the initial text.
```

The application will create these files after it processes changes:

```text
output/audit.log
output/statistics.json
output/uppercase.txt
```

---

## Step 4 - Create the Event Bus

Open `src/event-bus.js`:

```javascript
import { EventEmitter } from 'node:events';

export const eventBus = new EventEmitter({
  captureRejections: true
});

eventBus.on('error', (error) => {
  console.error('[EVENT BUS ERROR]', error);
});
```

### What this code does

- `EventEmitter` provides the in-process event bus.
- `captureRejections: true` forwards rejected promises from asynchronous subscribers to the `error` event.
- The `error` subscriber logs listener failures.

### Important warning

If an EventEmitter emits `error` without an error listener, Node.js throws the error and may terminate the process. Register the listener before the application can publish events.

---

## Step 5 - Create the File-Change Publisher

Open `src/publisher.js`:

```javascript
import { createHash, randomUUID } from 'node:crypto';
import { watch } from 'node:fs';
import { readFile, stat } from 'node:fs/promises';
import path from 'node:path';
import { eventBus } from './event-bus.js';

function calculateHash(content) {
  return createHash('sha256').update(content).digest('hex');
}

export async function startFilePublisher(filePath) {
  const absolutePath = path.resolve(filePath);
  const directory = path.dirname(absolutePath);
  const filename = path.basename(absolutePath);

  let previousHash = null;
  let debounceTimer = null;
  let processing = false;
  let processAgain = false;

  async function publishCurrentContent() {
    if (processing) {
      processAgain = true;
      return;
    }

    processing = true;

    try {
      const content = await readFile(absolutePath, 'utf8');
      const fileInfo = await stat(absolutePath);
      const contentHash = calculateHash(content);

      if (contentHash === previousHash) {
        return;
      }

      previousHash = contentHash;

      const event = {
        event_id: randomUUID(),
        type: 'file.changed',
        occurred_at: new Date().toISOString(),
        source: 'text-file-publisher',
        data: {
          path: path.relative(process.cwd(), absolutePath),
          size_bytes: fileInfo.size,
          content_hash: contentHash,
          content
        }
      };

      console.log(`[PUBLISHER] Publishing ${event.type}`);
      eventBus.emit(event.type, event);
    } catch (error) {
      eventBus.emit('error', error);
    } finally {
      processing = false;

      if (processAgain) {
        processAgain = false;
        await publishCurrentContent();
      }
    }
  }

  await publishCurrentContent();

  const watcher = watch(directory, (eventType, changedFilename) => {
    if (changedFilename && changedFilename !== filename) {
      return;
    }

    clearTimeout(debounceTimer);

    debounceTimer = setTimeout(() => {
      publishCurrentContent();
    }, 150);
  });

  watcher.on('error', (error) => {
    eventBus.emit('error', error);
  });

  console.log(`[PUBLISHER] Watching ${absolutePath}`);

  return watcher;
}
```

### Publisher responsibilities

The publisher:

1. Watches the directory containing the target file.
2. Responds only to notifications involving the target filename.
3. Waits briefly for a file-writing operation to settle.
4. Reads the current file content.
5. Calculates a SHA-256 content hash.
6. Ignores notifications when the content did not change.
7. Creates and emits a structured `file.changed` event.

### Why watch the directory?

Some editors save a file by creating a temporary file and replacing the original. Watching the directory is often more reliable for demonstrations than attaching the watcher only to the original file object.

### Why debounce the notification?

A single save can produce several file-system notifications. The 150-millisecond timer groups rapid notifications before reading the file.

### Why compare hashes?

`watch()` reports file-system activity, not guaranteed semantic content changes. The hash prevents subscribers from processing the same content again.

---

## Step 6 - Create the Subscribers

Open `src/subscribers.js`:

```javascript
import { mkdir, appendFile, writeFile } from 'node:fs/promises';
import { eventBus } from './event-bus.js';

function countWords(content) {
  const text = content.trim();
  return text === '' ? 0 : text.split(/\s+/).length;
}

function countLines(content) {
  const text = content.trimEnd();
  return text === '' ? 0 : text.split(/\r?\n/).length;
}

eventBus.on('file.changed', (event) => {
  console.log(
    `[CONSOLE] ${event.data.path} changed at ${event.occurred_at}`
  );
});

eventBus.on('file.changed', async (event) => {
  await mkdir('output', { recursive: true });
  await appendFile(
    'output/audit.log',
    `${JSON.stringify(event)}\n`,
    'utf8'
  );

  console.log(`[AUDIT] Stored event ${event.event_id}`);
});

eventBus.on('file.changed', async (event) => {
  const statistics = {
    event_id: event.event_id,
    calculated_at: new Date().toISOString(),
    source_path: event.data.path,
    characters: event.data.content.length,
    words: countWords(event.data.content),
    lines: countLines(event.data.content),
    size_bytes: event.data.size_bytes
  };

  await mkdir('output', { recursive: true });
  await writeFile(
    'output/statistics.json',
    `${JSON.stringify(statistics, null, 2)}\n`,
    'utf8'
  );

  console.log(
    `[STATISTICS] ${statistics.words} words, ${statistics.lines} lines`
  );
});

eventBus.on('file.changed', async (event) => {
  await mkdir('output', { recursive: true });
  await writeFile(
    'output/uppercase.txt',
    event.data.content.toUpperCase(),
    'utf8'
  );

  console.log('[TRANSFORM] Updated output/uppercase.txt');
});
```

### Subscriber responsibilities

- The console subscriber displays a short notification.
- The audit subscriber preserves every event as one JSON line.
- The statistics subscriber calculates information about the latest content.
- The transformation subscriber writes an uppercase copy.

The publisher does not import or call any of these subscriber functions. The event name and payload form the connection between them.

---

## Step 7 - Start the Application

Open `src/app.js`:

```javascript
import './subscribers.js';
import { startFilePublisher } from './publisher.js';

const watchedFile = 'data/input.txt';

const watcher = await startFilePublisher(watchedFile);

function stopApplication(signal) {
  console.log(`\n[APP] Received ${signal}. Stopping watcher.`);
  watcher.close();
  process.exit(0);
}

process.on('SIGINT', () => stopApplication('SIGINT'));
process.on('SIGTERM', () => stopApplication('SIGTERM'));

console.log('[APP] Edit data/input.txt to publish an event.');
console.log('[APP] Press Ctrl+C to stop.');
```

### Startup order

1. Importing `subscribers.js` registers all listeners.
2. `startFilePublisher()` reads the initial content and starts the watcher.
3. Signal handlers close the watcher before exiting.

Subscribers must be registered before the publisher emits the initial event.

---

## Step 8 - Run the Publisher

Start the application:

```bash
npm start
```

Expected initial output:

```text
[PUBLISHER] Publishing file.changed
[CONSOLE] data/input.txt changed at ...
[PUBLISHER] Watching .../data/input.txt
[APP] Edit data/input.txt to publish an event.
[APP] Press Ctrl+C to stop.
[AUDIT] Stored event ...
[STATISTICS] 5 words, 1 lines
[TRANSFORM] Updated output/uppercase.txt
```

Messages from asynchronous subscribers may complete in a different order.

---

## Step 9 - Trigger an Event

Keep the Node.js application running. Open `data/input.txt` in your editor and replace its contents:

```text
Event-driven programs react when something meaningful happens.
One event can notify several independent subscribers.
```

Save the file.

Expected terminal output:

```text
[PUBLISHER] Publishing file.changed
[CONSOLE] data/input.txt changed at ...
[AUDIT] Stored event ...
[STATISTICS] 13 words, 2 lines
[TRANSFORM] Updated output/uppercase.txt
```

The exact word count depends on the text entered.

---

## Step 10 - Inspect Subscriber Outputs

Open `output/uppercase.txt`. Its content should be uppercase:

```text
EVENT-DRIVEN PROGRAMS REACT WHEN SOMETHING MEANINGFUL HAPPENS.
ONE EVENT CAN NOTIFY SEVERAL INDEPENDENT SUBSCRIBERS.
```

Open `output/statistics.json`:

```json
{
  "event_id": "...",
  "calculated_at": "...",
  "source_path": "data/input.txt",
  "characters": 112,
  "words": 13,
  "lines": 2,
  "size_bytes": 112
}
```

Open `output/audit.log`. It should contain one JSON object per processed version of the file.

### Discussion

`statistics.json` and `uppercase.txt` represent the latest state. Each change overwrites them. `audit.log` represents history, so each event is appended.

---

## Step 11 - Test Duplicate Detection

Save `data/input.txt` again without changing its content.

The file system may report activity, but the publisher should not emit a new event because the SHA-256 hash has not changed.

Verify that:

- No new `[PUBLISHER] Publishing file.changed` message appears.
- `audit.log` does not gain another event for identical content.
- The event ID in `statistics.json` remains unchanged.

---

## Step 12 - Trigger Rapid Changes

Modify and save the file several times quickly. The debounce timer should group notifications that occur within 150 milliseconds.

### Observation questions

1. How many times did the file system report the change?
2. How many events reached the audit log?
3. Did the output files represent the latest content?
4. Could an intermediate edit be missed?

Debouncing reduces duplicate work, but it can intentionally collapse rapid intermediate states. This tradeoff is acceptable for the demo but may not be acceptable for every application.

---

## Step 13 - Add Another Subscriber

Add this subscriber to `src/subscribers.js`:

```javascript
eventBus.on('file.changed', (event) => {
  const containsImportant = event.data.content
    .toLowerCase()
    .includes('important');

  if (containsImportant) {
    console.log('[KEYWORD] The file contains the word "important".');
  }
});
```

Restart the application. Add the word `important` to `data/input.txt` and save it.

### Key observation

The new behavior required no change to `publisher.js`. This demonstrates loose coupling between the event producer and its consumers.

---

## Step 14 - Demonstrate a Subscriber Failure

Temporarily add this subscriber:

```javascript
eventBus.on('file.changed', async () => {
  throw new Error('Simulated subscriber failure');
});
```

Change and save the input file. The terminal should display:

```text
[EVENT BUS ERROR] Error: Simulated subscriber failure
```

The application handles the rejected promise because the EventEmitter uses `captureRejections: true` and has an `error` listener.

Remove the simulated subscriber after the demonstration.

---

## Step 15 - Demonstrate `once()`

Add this subscriber before starting the application:

```javascript
eventBus.once('file.changed', (event) => {
  console.log(`[FIRST CHANGE] Received ${event.event_id}`);
});
```

Change the file twice. The `[FIRST CHANGE]` message should appear only once.

- `on()` processes every matching event.
- `once()` removes the subscriber after its first invocation.

---

## Step 16 - EventEmitter Timing

Node.js invokes registered EventEmitter listeners synchronously and in registration order. However, an asynchronous listener returns a promise when it reaches its first `await`. `emit()` does not wait for all asynchronous work to complete.

Add temporary timing messages:

```javascript
console.log('[PUBLISHER] Before emit');
eventBus.emit(event.type, event);
console.log('[PUBLISHER] After emit');
```

Compare these messages with the audit, statistics, and transformation completion messages.

### Discussion questions

1. Which subscriber code runs before `emit()` returns?
2. Which file operations finish later?
3. What happens if the process exits before asynchronous subscribers finish?

---

## Step 17 - Publisher and Subscriber Comparison

| Responsibility | Publisher | Subscriber |
|---|---|---|
| Detect file activity | Yes | No |
| Read and hash the file | Yes | No |
| Create the event | Yes | No |
| Know every reaction | No | No |
| React to `file.changed` | No | Yes |
| Produce audit or derived files | No | Yes |

The publisher knows what happened. Subscribers decide what to do because it happened.

---

## Step 18 - Limitations of the Demo

This application demonstrates event-driven structure inside one Node.js process. It is not a durable messaging system.

- Events disappear when the process is stopped.
- Other processes and computers cannot subscribe.
- `emit()` provides no delivery acknowledgment.
- Failed subscribers have no built-in retry queue.
- File watching behavior can vary by operating system and file system.
- Debouncing may skip intermediate file versions.
- A slow synchronous subscriber delays every later subscriber.
- An event payload containing full file content may become expensive for large files.
- The audit log needs rotation and protection in a production application.

Use a broker or durable queue when events must cross process boundaries, survive restarts, support retries, or reach remote consumers. MQTT will address a related distributed publish-subscribe model in a later course topic.

---

## Step 19 - Student Extension Activities

Choose at least one extension.

### Option A - Keyword events

When the input contains `ERROR`, emit a second event named `keyword.detected`.

Suggested payload:

```json
{
  "event_id": "...",
  "type": "keyword.detected",
  "occurred_at": "...",
  "source": "keyword-subscriber",
  "data": {
    "keyword": "ERROR",
    "source_event_id": "...",
    "path": "data/input.txt"
  }
}
```

Create a subscriber that stores keyword events in `output/alerts.log`.

### Option B - Word-frequency output

Create `output/frequencies.json` containing the five most frequent words. Ignore capitalization and punctuation.

### Option C - Multiple watched files

Watch every `.txt` file in `data/`. Include the changed filename in the event payload and ignore non-text files.

### Option D - Content history

Save each changed version under `output/history/` using its timestamp or event ID as the filename.

### Option E - HTTP status endpoint

Add a small Express server that returns the latest statistics and recent events. Keep the file watcher as the publisher.

### Option F - Replace EventEmitter with MQTT

Publish the same event payload to an MQTT topic such as:

```text
files/input/changed
```

Compare in-process subscribers with subscribers running in separate programs.

---

## Completion Checklist

- [ ] `npm start` runs without an error.
- [ ] The application monitors `data/input.txt`.
- [ ] A content change emits `file.changed`.
- [ ] Every event has an ID, type, timestamp, source, and data object.
- [ ] Identical content does not produce another event.
- [ ] Rapid notifications are debounced.
- [ ] The console subscriber receives the event.
- [ ] `output/audit.log` stores event history.
- [ ] `output/statistics.json` describes the latest text.
- [ ] `output/uppercase.txt` contains the transformed text.
- [ ] Subscriber errors reach the `error` listener.
- [ ] The student can explain why EventEmitter is not a message broker.

## Suggested Submission Evidence

1. Project source code without `node_modules`
2. Screenshot of the watcher running
3. Original and changed versions of `data/input.txt`
4. Terminal output showing the published event and subscribers
5. `output/audit.log`
6. `output/statistics.json`
7. `output/uppercase.txt`
8. Evidence that saving identical content did not add an event
9. One completed extension activity
10. A short explanation of loose coupling in the application

## Group Laboratory Analysis

After completing the demonstration, work with your assigned group on [Topic 6 Event-Driven File Monitoring: Laboratory Analysis Activity](topic-06-event-driven-laboratory-analysis-activity.md). The analysis questions, evidence requirements, and rubric are kept in that separate activity document.

## Troubleshooting

### The application reports that `input.txt` does not exist

Start the program from the project root and confirm that `data/input.txt` exists.

### No event appears after saving

Confirm that the application is still running and that you edited the correct file. Try changing the actual content rather than saving an identical version.

### Several notifications occur for one save

This can happen with file-system watchers. Confirm that the debounce timer and content-hash comparison are present.

### Output files do not appear

Check the terminal for `[EVENT BUS ERROR]`. Confirm that the application can create and write to the `output` directory.

### The process terminates after a subscriber error

Confirm that the `error` listener is registered on the same `eventBus` instance used by the publisher and subscribers.

### Changes to subscriber code do not take effect

Stop the process with Ctrl+C and start it again. This guide does not use Node watch mode.

### The watcher reacts to an output file

Confirm that the watcher monitors only the `data` directory and filters for `input.txt`. The generated files belong in `output`.

## References

- Zalando SE, *Zalando RESTful API and Event Guidelines*, **pp. 119-127**, event types, schemas, compatibility, and service-interface guidance

### External references

- Node.js, [Events](https://nodejs.org/api/events.html), **Class: EventEmitter**, **Asynchronous vs. synchronous**, **Handling events only once**, **Error events**, and **Capture rejections of promises**
- Node.js, [File system](https://nodejs.org/api/fs.html), **`fs.watch()`** and file-system watching caveats
- Node.js, [Crypto](https://nodejs.org/api/crypto.html), **`crypto.createHash()`** and **`crypto.randomUUID()`**
- Node.js, [Do not block the Event Loop](https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop), **A quick review of Node**
