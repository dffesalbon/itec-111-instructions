# Building a Paired-Group Web Chat with Node.js and WebSockets

## Overview

In this laboratory, two partner groups build and test a private, local-network WebSocket chat session. One group hosts the Node.js server while the partner connects through the host's local IPv4 address. The server uses one session code and the two group identifiers registered by the pair.

The finished application demonstrates:

- a WebSocket connection and its lifecycle;
- a WebSocket opening and closing handshake;
- joining and leaving a paired chat session;
- selective broadcasting to accepted participants;
- presence notifications and participant counts;
- validated JSON messages;
- reconnection and automatic session re-entry;
- heartbeat checks for unresponsive clients; and
- safe error handling and interface updates.

## Learning Objectives

After completing the laboratory, students should be able to:

1. Create a Node.js HTTP server that also accepts WebSocket connections.
2. Explain the `CONNECTING`, `OPEN`, `CLOSING`, and `CLOSED` states.
3. Design and validate a JSON message contract.
4. Maintain per-client state such as display name and current session.
5. Broadcast messages only to clients in the same session.
6. Handle joins, leaves, errors, and unexpected disconnections.
7. Reconnect with exponential backoff and rejoin a previous session.
8. Explain related WebSocket concepts including `ws`/`wss`, close codes, ping/pong, backpressure, and origin validation.

## Expected Output

Each pair must produce a working chat application containing:

- a join screen for display name, group identifier, and pair-session code;
- a chat screen with session name, connection status, participant count, message list, and message form;
- a Node.js server using the `ws` library;
- server configuration containing exactly two allowed group identifiers;
- rejection of an unregistered group identifier or incorrect session code;
- join and leave system messages;
- chat messages containing sender, session, event ID, and timestamp;
- server-side validation;
- automatic reconnection and session re-entry; and
- evidence of a successful conversation between the two partner groups.

## Pairing and Hosting Rules

1. Two groups form one activity pair.
2. The pair records its two group identifiers, for example `GROUP-01` and `GROUP-07`.
3. The pair creates a unique session code, for example `PAIR-01-07`.
4. The pair chooses one **host group** and one **connecting group**.
5. The host group runs the Node.js server and shares its local IPv4 URL, such as `http://192.168.x.x:3000`.
6. The pair uses its registered group identifiers and session code throughout the activity.
7. Multiple members of either registered group may join for testing.

The host and connecting group must be on the same local network. The host must allow Node.js through the operating-system firewall. Do not expose the development server directly to the public internet.

### Pair registration record

| Field | Pair entry |
|---|---|
| Host group |  |
| Connecting group |  |
| Session code |  |
| Host IPv4 address |  |
| Host member responsible for server |  |
| Connecting member responsible for client test |  |

## Activity Flow

```text
Two groups form one activity pair
                        ↓
The pair selects a host and creates a unique session code
                        ↓
Host configures exactly two allowed group identifiers
                        ↓
Members enter display name, group identifier, and session code
                        ↓
Server accepts only the registered pair and stores client state
                        ↓
The two groups exchange WebSocket chat messages
                        ↓
Pair tests unauthorized join, invalid input, disconnect, and reconnect
                        ↓
Students save evidence and answer analysis/reflection questions
```

## Message Contract

Every application message is a JSON object with a string `type`.

### Client-to-server messages

| Type | Purpose | Required fields |
|---|---|---|
| `session.join` | Join or rejoin the pair session | `name`, `group`, `session` |
| `chat.send` | Submit a message to the current session | `text` |
| `session.leave` | Leave the current session | None |

### Server-to-client messages

| Type | Purpose | Important fields |
|---|---|---|
| `server.welcome` | Confirms the socket connection | `clientId`, `timestamp` |
| `session.joined` | Confirms pair-session membership | `session`, `name`, `group`, `participants` |
| `session.presence` | Announces a join or leave | `action`, `name`, `group`, `participants` |
| `chat.message` | Delivers a pair-session message | `eventId`, `session`, `sender`, `group`, `text`, `timestamp` |
| `error` | Reports an application error | `code`, `message` |

## Application Rules

- A client must join a session before sending chat messages.
- The server is configured for exactly one session and two allowed group identifiers.
- A client must provide a group identifier matching one of those two groups.
- Display names contain 2 to 30 characters.
- Session codes contain 3 to 30 letters, numbers, underscores, or hyphens.
- Chat messages contain 1 to 500 characters after trimming.
- Messages are broadcast only to clients accepted into the configured pair session.
- User text is rendered with `textContent`, never `innerHTML`.
- Session membership is stored on the server, not trusted from each `chat.send` message.
- Chat history exists only in memory and is lost when the server restarts.

---

## Step 1 - Create the Project

```bash
mkdir realtime-session-chat
cd realtime-session-chat
npm init -y
npm install ws
```

Update `package.json`:

```json
{
  "type": "module",
  "scripts": {
    "start": "node src/server.js",
    "dev": "node --watch src/server.js"
  }
}
```

Create this structure:

```text
realtime-session-chat/
├── package.json
├── public/
│   ├── app.js
│   ├── index.html
│   └── styles.css
└── src/
    └── server.js
```

## Step 2 - Create the Chat Interface

Create `public/index.html`:

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>ITEC 111 Session Chat</title>
  <link rel="stylesheet" href="/styles.css">
</head>
<body>
  <main class="app-shell">
    <header>
      <div><p>ITEC 111</p><h1>Session-Based Web Chat</h1></div>
      <span id="connection-status" class="status disconnected">Disconnected</span>
    </header>

    <section id="join-view" class="panel">
      <h2>Join your paired-group session</h2>
      <form id="join-form">
        <label>Display name
          <input id="display-name" minlength="2" maxlength="30" required>
        </label>
        <label>Group identifier
          <input id="group-id" minlength="3" maxlength="20"
            placeholder="GROUP-01" required>
        </label>
        <label>Session code
          <input id="session-code" minlength="3" maxlength="30" required>
        </label>
        <button type="submit">Join session</button>
      </form>
      <p id="join-feedback" aria-live="polite"></p>
    </section>

    <section id="chat-view" class="panel" hidden>
      <div class="session-bar">
        <div><p>Session</p><h2 id="active-session">--</h2></div>
        <p><span id="participant-count">0</span> participant(s)</p>
        <button id="leave-button" type="button">Leave session</button>
      </div>
      <ol id="message-list" class="messages" aria-live="polite"></ol>
      <form id="message-form">
        <input id="message-text" maxlength="500" autocomplete="off"
          placeholder="Write a message" required>
        <button type="submit">Send</button>
      </form>
      <p id="chat-feedback" aria-live="polite"></p>
    </section>
  </main>
  <script type="module" src="/app.js"></script>
</body>
</html>
```

Create `public/styles.css`. Students may personalize the design, but connection state and messages must remain readable:

```css
:root { color-scheme: dark; font-family: system-ui, sans-serif; background: #07111f; color: #eaf2ff; }
* { box-sizing: border-box; }
body { margin: 0; }
.app-shell { width: min(100% - 2rem, 860px); margin: 2rem auto; }
header, .session-bar, #message-form { display: flex; gap: 1rem; align-items: center; justify-content: space-between; }
.panel { margin-top: 1.5rem; padding: 1.25rem; border: 1px solid #29415f; border-radius: 1rem; background: #101e31; }
#join-form { display: grid; gap: 1rem; }
label { display: grid; gap: .4rem; }
input, button { padding: .75rem; border-radius: .5rem; }
#message-text { flex: 1; }
.status { padding: .45rem .75rem; border-radius: 999px; }
.connected { background: #14532d; }
.connecting, .reconnecting { background: #854d0e; }
.disconnected { background: #7f1d1d; }
.messages { height: 22rem; overflow-y: auto; padding: 1rem; background: #07111f; border-radius: .75rem; }
.messages li { margin-bottom: .8rem; }
.system-message { color: #9fb6d1; font-style: italic; }
```

## Step 3 - Create the HTTP and WebSocket Server

Create `src/server.js`:

```javascript
import http from 'node:http';
import { readFile } from 'node:fs/promises';
import { extname, join } from 'node:path';
import { fileURLToPath } from 'node:url';
import { randomUUID } from 'node:crypto';
import { WebSocketServer, WebSocket } from 'ws';

const port = Number(process.env.PORT) || 3000;
const configuredSession = (process.env.SESSION_CODE ?? 'PAIR-01-02')
  .trim().toUpperCase();
const allowedGroups = new Set(
  (process.env.ALLOWED_GROUPS ?? 'GROUP-01,GROUP-02')
    .split(',')
    .map(value => value.trim().toUpperCase())
    .filter(Boolean)
);

if (allowedGroups.size !== 2) {
  throw new Error('ALLOWED_GROUPS must contain exactly two group identifiers.');
}

const publicDirectory = fileURLToPath(new URL('../public/', import.meta.url));
const publicFiles = new Map([['/', 'index.html'], ['/app.js', 'app.js'], ['/styles.css', 'styles.css']]);
const contentTypes = { '.html': 'text/html; charset=utf-8', '.js': 'text/javascript; charset=utf-8', '.css': 'text/css; charset=utf-8' };

const server = http.createServer(async (request, response) => {
  const pathname = new URL(request.url, `http://${request.headers.host}`).pathname;
  const publicFile = publicFiles.get(pathname);
  if (!publicFile) {
    response.writeHead(404, { 'Content-Type': 'text/plain; charset=utf-8' });
    response.end('Not found');
    return;
  }
  try {
    const filePath = join(publicDirectory, publicFile);
    const file = await readFile(filePath);
    response.writeHead(200, { 'Content-Type': contentTypes[extname(filePath)] });
    response.end(file);
  } catch {
    response.writeHead(500, { 'Content-Type': 'text/plain; charset=utf-8' });
    response.end('Unable to load the application');
  }
});

const wss = new WebSocketServer({ noServer: true });

server.on('upgrade', (request, socket, head) => {
  const pathname = new URL(request.url, `http://${request.headers.host}`).pathname;
  if (pathname !== '/ws') {
    socket.destroy();
    return;
  }
  wss.handleUpgrade(request, socket, head, webSocket => {
    wss.emit('connection', webSocket);
  });
});
```

## Step 4 - Add Session Helpers

```javascript
function sendJson(socket, message) {
  if (socket.readyState === WebSocket.OPEN) socket.send(JSON.stringify(message));
}

function acceptedClients() {
  return [...wss.clients].filter(client =>
    client.readyState === WebSocket.OPEN &&
    client.session === configuredSession &&
    allowedGroups.has(client.groupId)
  );
}

function broadcastToPair(message) {
  const payload = JSON.stringify(message);
  for (const client of acceptedClients()) client.send(payload);
}

function participantCount() {
  return acceptedClients().length;
}

function sendError(socket, code, message) {
  sendJson(socket, { type: 'error', code, message });
}

function announcePresence(action, socket) {
  broadcastToPair({
    type: 'session.presence', action,
    name: socket.displayName,
    group: socket.groupId,
    participants: participantCount(),
    timestamp: new Date().toISOString()
  });
}
```

The server decides membership from its own configured session and allowed-group list. It does not allow a sender to choose an arbitrary destination in each chat message.

## Step 5 - Validate and Process Messages

```javascript
function normalizeSession(value) {
  return typeof value === 'string' ? value.trim().toUpperCase() : '';
}

function leaveCurrentSession(socket) {
  if (!socket.session) return;
  socket.session = null;
  announcePresence('left', socket);
}

function handleJoin(socket, message) {
  const name = typeof message.name === 'string' ? message.name.trim() : '';
  const session = normalizeSession(message.session);
  const group = normalizeSession(message.group);
  if (name.length < 2 || name.length > 30) {
    sendError(socket, 'INVALID_NAME', 'Name must contain 2 to 30 characters.');
    return;
  }
  if (!/^[A-Z0-9_-]{3,30}$/.test(session)) {
    sendError(socket, 'INVALID_SESSION', 'Session code has an invalid format.');
    return;
  }
  if (session !== configuredSession) {
    sendError(socket, 'SESSION_NOT_ALLOWED', 'This server is not assigned to that session.');
    return;
  }
  if (!allowedGroups.has(group)) {
    sendError(socket, 'GROUP_NOT_ALLOWED', 'This group is not registered for the paired session.');
    return;
  }
  if (socket.session) leaveCurrentSession(socket);
  socket.displayName = name;
  socket.groupId = group;
  socket.session = session;
  sendJson(socket, {
    type: 'session.joined', session, name, group,
    participants: participantCount(),
    timestamp: new Date().toISOString()
  });
  announcePresence('joined', socket);
}

function handleChat(socket, message) {
  if (!socket.session) {
    sendError(socket, 'NOT_IN_SESSION', 'Join a session before sending messages.');
    return;
  }
  const text = typeof message.text === 'string' ? message.text.trim() : '';
  if (text.length < 1 || text.length > 500) {
    sendError(socket, 'INVALID_CHAT', 'Message must contain 1 to 500 characters.');
    return;
  }
  broadcastToPair({
    type: 'chat.message', eventId: randomUUID(), session: socket.session,
    sender: socket.displayName, group: socket.groupId, text,
    timestamp: new Date().toISOString()
  });
}

function handleMessage(socket, rawData) {
  let message;
  try { message = JSON.parse(rawData.toString()); }
  catch {
    sendError(socket, 'INVALID_JSON', 'Message must be valid JSON.');
    return;
  }
  if (!message || typeof message !== 'object' || Array.isArray(message)) {
    sendError(socket, 'INVALID_MESSAGE', 'Message must be a JSON object.');
    return;
  }
  switch (message.type) {
    case 'session.join': handleJoin(socket, message); break;
    case 'chat.send': handleChat(socket, message); break;
    case 'session.leave': leaveCurrentSession(socket); break;
    default: sendError(socket, 'UNKNOWN_TYPE', 'Message type is not supported.');
  }
}
```

## Step 6 - Implement the Connection Lifecycle

```javascript
wss.on('connection', socket => {
  socket.clientId = randomUUID();
  socket.displayName = null;
  socket.groupId = null;
  socket.session = null;
  socket.isAlive = true;

  sendJson(socket, {
    type: 'server.welcome', clientId: socket.clientId,
    timestamp: new Date().toISOString()
  });

  socket.on('message', rawData => handleMessage(socket, rawData));
  socket.on('pong', () => { socket.isAlive = true; });
  socket.on('error', error => console.error(`WebSocket error: ${error.message}`));
  socket.on('close', () => leaveCurrentSession(socket));
});
```

## Step 7 - Add Heartbeat and Shutdown Logic

```javascript
const heartbeatTimer = setInterval(() => {
  for (const socket of wss.clients) {
    if (socket.isAlive === false) {
      socket.terminate();
      continue;
    }
    socket.isAlive = false;
    socket.ping();
  }
}, 30_000);

server.listen(port, '0.0.0.0', () => {
  console.log(`Chat server running at http://localhost:${port}`);
});

function shutDown() {
  clearInterval(heartbeatTimer);
  for (const socket of wss.clients) socket.close(1001, 'Server shutting down');
  server.close(() => process.exit(0));
}

process.on('SIGINT', shutDown);
process.on('SIGTERM', shutDown);
```

## Step 8 - Create the Browser Client

Create `public/app.js`:

```javascript
const byId = id => document.querySelector(`#${id}`);
const joinView = byId('join-view');
const chatView = byId('chat-view');
const joinForm = byId('join-form');
const messageForm = byId('message-form');
const leaveButton = byId('leave-button');
const displayNameInput = byId('display-name');
const groupIdInput = byId('group-id');
const sessionCodeInput = byId('session-code');
const messageTextInput = byId('message-text');
const statusElement = byId('connection-status');
const activeSessionElement = byId('active-session');
const participantCountElement = byId('participant-count');
const messageList = byId('message-list');
const joinFeedback = byId('join-feedback');
const chatFeedback = byId('chat-feedback');

let socket;
let reconnectAttempt = 0;
let reconnectTimer;
let intentionallyClosed = false;
let desiredIdentity = null;

function webSocketUrl() {
  const protocol = location.protocol === 'https:' ? 'wss:' : 'ws:';
  return `${protocol}//${location.host}/ws`;
}

function setStatus(text, className) {
  statusElement.textContent = text;
  statusElement.className = `status ${className}`;
}

function send(message) {
  if (!socket || socket.readyState !== WebSocket.OPEN) return false;
  socket.send(JSON.stringify(message));
  return true;
}

function connect() {
  clearTimeout(reconnectTimer);
  setStatus(reconnectAttempt ? 'Reconnecting' : 'Connecting', 'connecting');
  socket = new WebSocket(webSocketUrl());

  socket.addEventListener('open', () => {
    reconnectAttempt = 0;
    setStatus('Connected', 'connected');
    if (desiredIdentity) send({ type: 'session.join', ...desiredIdentity });
  });
  socket.addEventListener('message', event => handleServerMessage(event.data));
  socket.addEventListener('error', () => {
    chatFeedback.textContent = 'A connection error occurred.';
  });
  socket.addEventListener('close', event => {
    setStatus('Disconnected', 'disconnected');
    if (!intentionallyClosed && event.code !== 1000) scheduleReconnect();
  });
}

function scheduleReconnect() {
  const base = Math.min(1000 * 2 ** reconnectAttempt, 30_000);
  const delay = base + Math.random() * 500;
  reconnectAttempt += 1;
  setStatus(`Reconnecting in ${Math.ceil(delay / 1000)}s`, 'reconnecting');
  reconnectTimer = setTimeout(connect, delay);
}
```

## Step 9 - Render Messages and Handle Forms

Continue `public/app.js`:

```javascript
function addSystemMessage(text) {
  const item = document.createElement('li');
  item.className = 'system-message';
  item.textContent = text;
  messageList.append(item);
  messageList.scrollTop = messageList.scrollHeight;
}

function addChatMessage(message) {
  const item = document.createElement('li');
  const time = new Date(message.timestamp).toLocaleTimeString();
  item.textContent = `[${time}] ${message.group} / ${message.sender}: ${message.text}`;
  messageList.append(item);
  messageList.scrollTop = messageList.scrollHeight;
}

function handleServerMessage(rawData) {
  let message;
  try { message = JSON.parse(rawData); }
  catch { addSystemMessage('Ignored an invalid server message.'); return; }

  switch (message.type) {
    case 'server.welcome': break;
    case 'session.joined':
      joinView.hidden = true;
      chatView.hidden = false;
      activeSessionElement.textContent = message.session;
      participantCountElement.textContent = message.participants;
      joinFeedback.textContent = '';
      addSystemMessage(`You joined ${message.session}.`);
      break;
    case 'session.presence':
      participantCountElement.textContent = message.participants;
      addSystemMessage(`${message.group} / ${message.name} ${message.action} the session.`);
      break;
    case 'chat.message': addChatMessage(message); break;
    case 'error':
      chatFeedback.textContent = `${message.code}: ${message.message}`;
      joinFeedback.textContent = `${message.code}: ${message.message}`;
      break;
    default: addSystemMessage(`Ignored unknown message type: ${message.type}`);
  }
}

joinForm.addEventListener('submit', event => {
  event.preventDefault();
  desiredIdentity = {
    name: displayNameInput.value.trim(),
    group: groupIdInput.value.trim(),
    session: sessionCodeInput.value.trim()
  };
  if (!send({ type: 'session.join', ...desiredIdentity })) {
    joinFeedback.textContent = 'Wait for the server connection.';
  }
});

messageForm.addEventListener('submit', event => {
  event.preventDefault();
  chatFeedback.textContent = '';
  if (!send({ type: 'chat.send', text: messageTextInput.value })) {
    chatFeedback.textContent = 'Cannot send while disconnected.';
    return;
  }
  messageTextInput.value = '';
});

leaveButton.addEventListener('click', () => {
  send({ type: 'session.leave' });
  desiredIdentity = null;
  messageList.replaceChildren();
  chatView.hidden = true;
  joinView.hidden = false;
});

window.addEventListener('pagehide', () => {
  intentionallyClosed = true;
  clearTimeout(reconnectTimer);
  if (socket && socket.readyState < WebSocket.CLOSING) {
    socket.close(1000, 'Page hidden');
  }
});

connect();
```

When the connection reopens, the browser resends `session.join` using `desiredIdentity`. This restores session membership, but messages sent while the client was offline are not recovered.

## Step 10 - Configure and Run the Pair Server

The host group configures the server for exactly one pair. On macOS or Linux:

```bash
SESSION_CODE=PAIR-01-07 ALLOWED_GROUPS=GROUP-01,GROUP-07 npm run dev
```

On PowerShell:

```powershell
$env:SESSION_CODE='PAIR-01-07'
$env:ALLOWED_GROUPS='GROUP-01,GROUP-07'
npm run dev
```

The host opens `http://localhost:3000`; the connecting group uses `http://HOST_IP_ADDRESS:3000`.

Each participant enters a recognizable display name, one of the two registered group identifiers, and the pair's session code.

## Step 11 - Required Inter-Group Conversation

Each paired group completes this exchange:

1. Group A introduces itself.
2. Group B acknowledges the introduction.
3. Group A asks one question about the WebSocket handshake, lifecycle, messages, heartbeat, or reconnection.
4. Group B answers through the chat.
5. Group B asks a different technical question.
6. Group A answers.
7. One participant disconnects and reconnects.
8. The partner group confirms the leave and rejoin messages.
9. Both groups send a final confirmation containing the same agreed session keyword.

Save screenshots showing both group names, the session code, messages from both sides, and a successful rejoin. Do not exchange personal or sensitive information.

## Step 12 - Test Pair Access and Errors

1. Connect members from both registered groups and exchange a message.
2. Attempt to join using an unregistered group identifier; expect `GROUP_NOT_ALLOWED`.
3. Attempt to join using the wrong session code; expect `SESSION_NOT_ALLOWED`.
4. Send `chat.send` before joining; expect `NOT_IN_SESSION`.
5. Send invalid JSON; expect `INVALID_JSON`.
6. Try session `??`; expect `INVALID_SESSION`.
7. Send an empty or 501-character message; expect `INVALID_CHAT`.
8. Stop the server, observe increasing retry delays, restart it, and confirm automatic re-entry.

## Step 13 - WebSocket Concept Review

Before submission, each pair explains how its implementation demonstrates:

- the HTTP Upgrade opening handshake;
- the four WebSocket ready states;
- full-duplex text/JSON messaging;
- server-side selective broadcasting;
- ping/pong heartbeat control frames;
- the Close frame and close codes;
- reconnection with capped exponential backoff and jitter;
- `ws://` for local development and `wss://` for TLS-protected deployment; and
- `bufferedAmount` as a signal that a slow client may need backpressure handling.

## Completion Checklist

- [ ] The server serves the chat interface and accepts `/ws` upgrades.
- [ ] Students can enter a display name and session code.
- [ ] Session codes are normalized and validated.
- [ ] The server is configured with exactly two allowed group identifiers.
- [ ] An unregistered group identifier and incorrect session code are rejected.
- [ ] Join and leave messages and participant counts are correct.
- [ ] User messages render as text, not HTML.
- [ ] Invalid input produces structured errors without crashing the server.
- [ ] Heartbeat logic removes unresponsive clients.
- [ ] Reconnection uses exponential backoff and jitter.
- [ ] A reconnected client automatically rejoins its session.
- [ ] Partner groups complete and document the required conversation.

## Required Technical Documentation

Each pair submits one shared `README.md` that includes:

1. Pair registration and identification of the host group.
2. Architecture diagram showing browsers, the Node.js host, HTTP file delivery, and `/ws` WebSocket connection.
3. Installation and configuration commands for `SESSION_CODE` and `ALLOWED_GROUPS`.
4. Separate operating instructions for the host group and connecting group.
5. Client-to-server and server-to-client message tables with JSON examples.
6. Validation rules and application error codes.
7. A join-to-chat sequence trace and a disconnect-to-rejoin sequence trace.
8. Completed test results with evidence references.
9. Known limitations: no durable history, no recovery of offline messages, typed group identity is not authentication, and local-network deployment only.
10. Documentation for the selected extension.

## References

- RFC 6455, *The WebSocket Protocol*, §§4, 5, and 7: <https://www.rfc-editor.org/rfc/rfc6455>
- MDN, *Writing WebSocket client applications*: <https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API/Writing_WebSocket_client_applications>
- `ws` documentation and examples: <https://github.com/websockets/ws>
