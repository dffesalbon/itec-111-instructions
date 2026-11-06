# Building a Paired WebRTC Voice and Video Room

## Overview

In this activity, one pair builds and tests a browser-based voice/video room. The **Host** runs a Node.js WebSocket signaling server and a browser client. The **Joiner** runs the same browser client on a second laptop. Both use the same LAN and room code.

The finished application demonstrates:

- room registration through WebSocket signaling;
- local camera and microphone capture;
- SDP offer-and-answer negotiation;
- ICE candidate exchange;
- encrypted WebRTC audio and video;
- one `RTCPeerConnection` for each remote participant;
- microphone, camera, and leave-room controls; and
- connection-state observation and cleanup.

The graded run uses two participants. The structure also supports optional additional participants by creating a separate peer connection for every participant pair.

## Expected Output (please refer to the activity document for actual/final the deliverables)

Each pair produces a working room application containing:

- a form for signaling-server address, participant name, and room code;
- local and dynamically created remote-video tiles;
- visible WebSocket and WebRTC states;
- microphone, camera, and leave-room buttons;
- a Node.js signaling server using `ws`;
- validated room, offer, answer, and ICE messages;
- peer cleanup after departure; and
- console evidence for the lifecycle observation.

## Pairing and Hosting Rules

1. Student A is the **Host** and Student B is the **Joiner**.
2. The Host runs `signaling-server.js` and Browser A.
3. The Joiner runs Browser B from the same project on a second laptop.
4. Both laptops use the same LAN and room code.
5. Both students load the client from `localhost` so media capture has a trustworthy development origin.
6. Both students use headphones to avoid acoustic feedback.
7. Do not expose this development server directly to the public internet.

### Pair registration record

| Field | Pair entry |
|---|---|
| Host student |  |
| Joiner student |  |
| Room code |  |
| Host IPv4 address |  |
| Host signaling URL |  |
| Test date and time |  |

## Required Topology

```text
Host laptop                              Joiner laptop
├── Node.js WebSocket signaling server   └── Browser B on localhost
└── Browser A on localhost                         │
         │                                         │
         └──────── WebSocket signaling ────────────┘

Browser A ═════════ encrypted WebRTC media ═══════ Browser B
```

WebSocket carries registration, offers, answers, ICE candidates, and departure notices. WebRTC carries audio/video directly when the network permits. A TURN server may relay encrypted peer traffic when a direct path fails.

## Signaling Message Contract

| Type | Direction | Purpose |
|---|---|---|
| `join-room` | browser to server | Register in a room |
| `room-state` | server to newcomer | Confirm room and list existing peers |
| `peer-joined` | server to existing peers | Announce a newcomer |
| `offer` | peer to peer through server | Forward an SDP offer |
| `answer` | peer to peer through server | Forward an SDP answer |
| `ice-candidate` | peer to peer through server | Forward an ICE candidate |
| `leave-room` | browser to server | Leave voluntarily |
| `peer-left` | server to room | Request peer cleanup |
| `error` | server to browser | Report invalid input |

The server derives `from_id` from the registered socket. It never trusts a client-supplied sender identity.

## Application Rules

- Names contain 2-30 characters.
- Room codes contain 3-30 letters, numbers, underscores, or hyphens.
- A participant joins before sending targeted signaling messages.
- Messages are forwarded only to a target in the sender's room.
- A room contains at most four participants.
- The newcomer creates offers for participants already in the room.
- Each browser stores `participantId -> RTCPeerConnection`.
- Early ICE candidates are queued until the remote description is ready.
- Local tracks, peer connections, and remote tiles are removed during cleanup.
- Complete SDP text and credentials must not appear in screenshots.

---

## Step 1 - Create the Project

Complete this on both laptops, or copy the finished folder to the Joiner laptop.

```bash
mkdir webrtc-room
cd webrtc-room
npm init -y
npm install ws
npm install --save-dev vite
mkdir client
touch signaling-server.js client/index.html client/app.js client/styles.css
```

Windows PowerShell file creation:

```powershell
New-Item -ItemType Directory client
New-Item signaling-server.js
New-Item client/index.html
New-Item client/app.js
New-Item client/styles.css
```

Expected structure:

```text
webrtc-room/
├── package.json
├── signaling-server.js
└── client/
    ├── app.js
    ├── index.html
    └── styles.css
```

Add `type` and replace the `scripts` section in `package.json`. Keep the generated dependency entries.

```json
{
  "type": "module",
  "scripts": {
    "signal": "node signaling-server.js",
    "signal:watch": "node --watch signaling-server.js",
    "client": "vite client"
  }
}
```

## Step 2 - Create the Room Interface

**Exercise alignment:** This step begins **Part A, Task 5 - Voice/video room controls** by creating the participant form, local and remote media areas, state display, and control buttons.

Create `client/index.html`:

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>ITEC 111 WebRTC Room</title>
  <link rel="stylesheet" href="/styles.css">
</head>
<body>
  <main class="app-shell">
    <header>
      <div><p class="eyebrow">ITEC 111</p><h1>WebRTC Voice and Video Room</h1></div>
      <span id="socket-state" class="status disconnected">Disconnected</span>
    </header>

    <section id="join-panel" class="panel">
      <h2>Join a room</h2>
      <form id="join-form">
        <label>Signaling server
          <input id="server-url" value="ws://localhost:3000" required>
        </label>
        <label>Participant name
          <input id="participant-name" minlength="2" maxlength="30" required>
        </label>
        <label>Room code
          <input id="room-code" minlength="3" maxlength="30" placeholder="ROOM-01" required>
        </label>
        <button type="submit">Join room</button>
      </form>
      <p id="join-feedback" class="feedback" aria-live="polite"></p>
    </section>

    <section id="room-panel" class="panel" hidden>
      <div class="room-bar">
        <div><p>Room</p><h2 id="active-room">--</h2></div>
        <p><span id="participant-count">1</span> participant(s)</p>
      </div>

      <section class="video-grid">
        <article class="video-card">
          <div class="card-heading"><strong id="local-label">Local preview</strong><span>you</span></div>
          <video id="local-video" autoplay playsinline muted></video>
        </article>
        <div id="remote-videos" class="remote-grid"></div>
      </section>

      <div class="controls">
        <button id="mute-button" type="button">Mute microphone</button>
        <button id="camera-button" type="button">Turn camera off</button>
        <button id="leave-button" class="danger" type="button">Leave room</button>
      </div>

      <section class="state-panel">
        <h2>Connection observations</h2>
        <pre id="state-log" aria-live="polite"></pre>
      </section>
    </section>
  </main>
  <script type="module" src="/app.js"></script>
</body>
</html>
```

## Step 3 - Style the Interface

**Exercise alignment:** This step supports **Part A, Task 5 - Voice/video room controls** by making participant identities, media tiles, connection states, and control states visible during testing.

Create `client/styles.css`:

```css
:root { font-family: Inter, system-ui, sans-serif; color: #e8fff5; background: #071a14; }
* { box-sizing: border-box; }
body { margin: 0; min-height: 100vh; background: #071a14; }
button, input { font: inherit; }
.app-shell { width: min(1180px, calc(100% - 2rem)); margin: 2rem auto; }
header, .room-bar, .card-heading, .controls { display: flex; align-items: center; justify-content: space-between; gap: 1rem; }
h1, h2, p { margin-top: 0; }
.eyebrow { color: #63d7a0; font-weight: 700; letter-spacing: .08em; }
.panel { margin-top: 1.5rem; padding: 1.25rem; border: 1px solid #285d49; border-radius: 1rem; background: #0d2b21; }
#join-form { display: grid; gap: 1rem; }
label { display: grid; gap: .4rem; }
input, button { border: 1px solid #3f8065; border-radius: .6rem; padding: .75rem; }
input { color: #e8fff5; background: #071a14; }
button { cursor: pointer; color: #052117; background: #70e0ab; font-weight: 700; }
.danger { color: #fff; background: #a43737; border-color: #d15c5c; }
.status { padding: .45rem .75rem; border-radius: 999px; font-weight: 700; }
.connected { background: #176b45; }
.connecting { background: #90600c; }
.disconnected { background: #8a2e2e; }
.feedback { min-height: 1.5rem; color: #ffd38a; }
.video-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(280px, 1fr)); gap: 1rem; }
.remote-grid { display: contents; }
.video-card { overflow: hidden; border: 1px solid #326c56; border-radius: .85rem; background: #06110d; }
.card-heading { padding: .7rem .85rem; background: #123d2f; }
.card-heading span { color: #a8cbbb; }
video { display: block; width: 100%; aspect-ratio: 16 / 9; background: #020705; object-fit: cover; }
.controls { justify-content: flex-start; flex-wrap: wrap; margin-top: 1rem; }
.state-panel { margin-top: 1.25rem; }
#state-log { min-height: 11rem; max-height: 20rem; overflow: auto; padding: 1rem; border-radius: .7rem; background: #06110d; color: #aef1ce; white-space: pre-wrap; }
@media (max-width: 640px) { header, .room-bar { align-items: flex-start; flex-direction: column; } }
```

## Step 4 - Create the WebSocket Signaling Server

**Exercise alignment:** This step implements **Part A, Task 1 - Room-aware signaling server**, including room registration, server-assigned participant identity, targeted signaling, validation, join/leave notices, the four-participant limit, and empty-room cleanup.

Create `signaling-server.js`:

```javascript
import { randomUUID } from 'node:crypto';
import { WebSocketServer, WebSocket } from 'ws';

const port = Number(process.env.PORT) || 3000;
const maximumRoomSize = 4;
const rooms = new Map(); // roomId -> Map(participantId, socket)
const wss = new WebSocketServer({ host: '0.0.0.0', port, maxPayload: 64 * 1024 });

function sendJson(socket, message) {
  if (socket.readyState === WebSocket.OPEN) socket.send(JSON.stringify(message));
}

function sendError(socket, code, message) {
  sendJson(socket, { type: 'error', code, message });
}

function normalizeCode(value) {
  return typeof value === 'string' ? value.trim().toUpperCase() : '';
}

function currentRoom(socket) {
  return socket.roomId ? rooms.get(socket.roomId) : undefined;
}

function publicParticipant(socket) {
  return { participant_id: socket.participantId, participant_name: socket.participantName };
}

function leaveRoom(socket) {
  const room = currentRoom(socket);
  if (!room) return;
  const oldRoomId = socket.roomId;
  room.delete(socket.participantId);
  socket.roomId = null;
  for (const peer of room.values()) {
    sendJson(peer, {
      type: 'peer-left',
      participant_id: socket.participantId,
      participant_name: socket.participantName
    });
  }
  console.log(`[leave] ${socket.participantName} left ${oldRoomId}`);
  if (room.size === 0) rooms.delete(oldRoomId);
}

function joinRoom(socket, message) {
  const roomId = normalizeCode(message.room_id);
  const participantName = typeof message.participant_name === 'string'
    ? message.participant_name.trim() : '';

  if (!/^[A-Z0-9_-]{3,30}$/.test(roomId)) {
    sendError(socket, 'INVALID_ROOM', 'Room code must contain 3-30 letters, numbers, underscores, or hyphens.');
    return;
  }
  if (participantName.length < 2 || participantName.length > 30) {
    sendError(socket, 'INVALID_NAME', 'Participant name must contain 2-30 characters.');
    return;
  }

  if (socket.roomId) leaveRoom(socket);
  const room = rooms.get(roomId) ?? new Map();
  if (room.size >= maximumRoomSize) {
    sendError(socket, 'ROOM_FULL', `Room limit is ${maximumRoomSize} participants.`);
    return;
  }

  const existingParticipants = [...room.values()].map(publicParticipant);
  socket.roomId = roomId;
  socket.participantName = participantName;
  room.set(socket.participantId, socket);
  rooms.set(roomId, room);

  sendJson(socket, { type: 'room-state', room_id: roomId, participants: existingParticipants });
  for (const peer of room.values()) {
    if (peer !== socket) sendJson(peer, { type: 'peer-joined', participant: publicParticipant(socket) });
  }
  console.log(`[join] ${participantName} joined ${roomId}; participants=${room.size}`);
}

function relayToPeer(socket, message) {
  const room = currentRoom(socket);
  if (!room) {
    sendError(socket, 'NOT_IN_ROOM', 'Join a room before signaling a peer.');
    return;
  }
  const target = room.get(message.target_id);
  if (!target || target === socket) {
    sendError(socket, 'INVALID_TARGET', 'Target is not another participant in this room.');
    return;
  }

  const forwarded = {
    type: message.type,
    from_id: socket.participantId,
    from_name: socket.participantName
  };
  if (message.type === 'ice-candidate') {
    if (!message.candidate || typeof message.candidate !== 'object') {
      sendError(socket, 'INVALID_CANDIDATE', 'ICE candidate is missing or invalid.');
      return;
    }
    forwarded.candidate = message.candidate;
  } else {
    if (!message.description || typeof message.description !== 'object') {
      sendError(socket, 'INVALID_DESCRIPTION', 'Session description is missing or invalid.');
      return;
    }
    forwarded.description = message.description;
  }
  sendJson(target, forwarded);
  console.log(`[relay] ${message.type}: ${socket.participantName} -> ${target.participantName}`);
}

function handleMessage(socket, rawData) {
  let message;
  try { message = JSON.parse(rawData.toString()); }
  catch { sendError(socket, 'INVALID_JSON', 'Message must be valid JSON.'); return; }

  if (!message || typeof message !== 'object' || Array.isArray(message)) {
    sendError(socket, 'INVALID_MESSAGE', 'Message must be a JSON object.');
    return;
  }
  switch (message.type) {
    case 'join-room': joinRoom(socket, message); break;
    case 'offer':
    case 'answer':
    case 'ice-candidate': relayToPeer(socket, message); break;
    case 'leave-room': leaveRoom(socket); break;
    default: sendError(socket, 'UNKNOWN_TYPE', 'Message type is not supported.');
  }
}

wss.on('connection', socket => {
  socket.participantId = randomUUID();
  socket.participantName = 'unregistered participant';
  socket.roomId = null;
  sendJson(socket, { type: 'welcome', participant_id: socket.participantId });
  socket.on('message', rawData => handleMessage(socket, rawData));
  socket.on('close', () => leaveRoom(socket));
  socket.on('error', error => console.error(`[socket error] ${error.message}`));
});

wss.on('listening', () => console.log(`WebRTC signaling server listening on ws://0.0.0.0:${port}`));

function shutDown() {
  for (const socket of wss.clients) socket.close(1001, 'Server shutting down');
  wss.close(() => process.exit(0));
}
process.on('SIGINT', shutDown);
process.on('SIGTERM', shutDown);
```

## Step 5 - Initialize the Browser Client

**Exercise alignment:** This step implements **Part A, Task 3 - Peer management** and the main data structures required by **Task 7 - Multi-participant readiness**. Each remote participant receives a separate peer connection and remote-video view.

Create `client/app.js`:

```javascript
const byId = id => document.querySelector(`#${id}`);
const joinPanel = byId('join-panel');
const roomPanel = byId('room-panel');
const joinForm = byId('join-form');
const serverUrlInput = byId('server-url');
const participantNameInput = byId('participant-name');
const roomCodeInput = byId('room-code');
const joinFeedback = byId('join-feedback');
const socketState = byId('socket-state');
const activeRoom = byId('active-room');
const participantCount = byId('participant-count');
const localLabel = byId('local-label');
const localVideo = byId('local-video');
const remoteVideos = byId('remote-videos');
const muteButton = byId('mute-button');
const cameraButton = byId('camera-button');
const leaveButton = byId('leave-button');
const stateLog = byId('state-log');

let socket;
let localStream;
let currentRoomId;
const peers = new Map();
const peerNames = new Map();
const remoteViews = new Map();
const pendingCandidates = new Map();
const rtcConfiguration = {
  iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
};

function logState(scope, message) {
  const line = `[${new Date().toLocaleTimeString()}] ${scope}: ${message}`;
  console.log(line);
  stateLog.textContent += `${line}\n`;
  stateLog.scrollTop = stateLog.scrollHeight;
}

function setSocketState(text, className) {
  socketState.textContent = text;
  socketState.className = `status ${className}`;
}

function sendSignal(message) {
  if (!socket || socket.readyState !== WebSocket.OPEN) return false;
  socket.send(JSON.stringify(message));
  return true;
}

function candidateType(candidate) {
  return candidate?.type ?? candidate?.candidate?.match(/ typ (\w+)/)?.[1] ?? 'unknown';
}

function updateParticipantCount() {
  participantCount.textContent = String(1 + peers.size);
}

function createRemoteView(remoteId) {
  if (remoteViews.has(remoteId)) return remoteViews.get(remoteId).video;
  const card = document.createElement('article');
  card.className = 'video-card';
  const heading = document.createElement('div');
  heading.className = 'card-heading';
  const name = document.createElement('strong');
  name.textContent = peerNames.get(remoteId) ?? 'Remote participant';
  const state = document.createElement('span');
  state.textContent = 'new';
  const video = document.createElement('video');
  video.autoplay = true;
  video.playsInline = true;
  heading.append(name, state);
  card.append(heading, video);
  remoteVideos.append(card);
  remoteViews.set(remoteId, { card, video, state });
  return video;
}

function removePeer(remoteId) {
  if (!peers.has(remoteId) && !remoteViews.has(remoteId)) return;
  peers.get(remoteId)?.close();
  peers.delete(remoteId);
  pendingCandidates.delete(remoteId);
  remoteViews.get(remoteId)?.card.remove();
  remoteViews.delete(remoteId);
  peerNames.delete(remoteId);
  updateParticipantCount();
  logState('Cleanup', `removed peer ${remoteId}`);
}

function createPeer(remoteId, remoteName = 'Remote participant') {
  if (peers.has(remoteId)) return peers.get(remoteId);
  peerNames.set(remoteId, remoteName);
  createRemoteView(remoteId);
  const peer = new RTCPeerConnection(rtcConfiguration);
  peers.set(remoteId, peer);
  for (const track of localStream.getTracks()) peer.addTrack(track, localStream);

  peer.addEventListener('icecandidate', ({ candidate }) => {
    if (!candidate) { logState(remoteName, 'ICE gathering complete'); return; }
    logState(remoteName, `local candidate type=${candidateType(candidate)}`);
    sendSignal({ type: 'ice-candidate', target_id: remoteId, candidate });
  });
  peer.addEventListener('track', ({ streams: [stream] }) => {
    const video = createRemoteView(remoteId);
    if (video.srcObject !== stream) video.srcObject = stream;
    logState(remoteName, `remote tracks=${stream.getTracks().map(track => track.kind).join(', ')}`);
  });
  peer.addEventListener('signalingstatechange', () => logState(remoteName, `signalingState=${peer.signalingState}`));
  peer.addEventListener('icegatheringstatechange', () => logState(remoteName, `iceGatheringState=${peer.iceGatheringState}`));
  peer.addEventListener('iceconnectionstatechange', () => logState(remoteName, `iceConnectionState=${peer.iceConnectionState}`));
  peer.addEventListener('connectionstatechange', () => {
    logState(remoteName, `connectionState=${peer.connectionState}`);
    const view = remoteViews.get(remoteId);
    if (view) view.state.textContent = peer.connectionState;
    if (peer.connectionState === 'failed' || peer.connectionState === 'closed') removePeer(remoteId);
  });
  updateParticipantCount();
  return peer;
}

async function addOrQueueCandidate(remoteId, candidate) {
  const peer = peers.get(remoteId) ?? createPeer(remoteId, peerNames.get(remoteId));
  if (peer.remoteDescription) {
    await peer.addIceCandidate(candidate);
    logState(peerNames.get(remoteId) ?? remoteId, `added remote candidate type=${candidateType(candidate)}`);
  } else {
    const queue = pendingCandidates.get(remoteId) ?? [];
    queue.push(candidate);
    pendingCandidates.set(remoteId, queue);
    logState(peerNames.get(remoteId) ?? remoteId, 'queued ICE candidate');
  }
}

async function flushCandidates(remoteId) {
  const queue = pendingCandidates.get(remoteId) ?? [];
  for (const candidate of queue) await peers.get(remoteId).addIceCandidate(candidate);
  pendingCandidates.delete(remoteId);
  if (queue.length) logState(peerNames.get(remoteId), `added ${queue.length} queued candidate(s)`);
}
```

The STUN entry is suitable for a classroom demonstration with internet access. The instructor may replace it with an institution-provided STUN/TURN configuration.

## Step 6 - Add Offer, Answer, ICE, and Server-Message Handling

**Exercise alignment:** This step implements **Part A, Task 4 - Negotiation**. It sends offers from the newcomer, returns answers from existing participants, forwards ICE candidates, and queues candidates until the remote description is ready.

Continue `client/app.js`:

```javascript
async function createAndSendOffer(participant) {
  peerNames.set(participant.participant_id, participant.participant_name);
  const peer = createPeer(participant.participant_id, participant.participant_name);
  await peer.setLocalDescription(await peer.createOffer());
  sendSignal({
    type: 'offer',
    target_id: participant.participant_id,
    description: peer.localDescription
  });
  logState(participant.participant_name, 'sent SDP offer');
}

async function handleOffer(message) {
  peerNames.set(message.from_id, message.from_name);
  const peer = createPeer(message.from_id, message.from_name);
  await peer.setRemoteDescription(message.description);
  await flushCandidates(message.from_id);
  await peer.setLocalDescription(await peer.createAnswer());
  sendSignal({ type: 'answer', target_id: message.from_id, description: peer.localDescription });
  logState(message.from_name, 'received offer and sent SDP answer');
}

async function handleAnswer(message) {
  const peer = peers.get(message.from_id);
  if (!peer) throw new Error('Received an answer for an unknown peer.');
  await peer.setRemoteDescription(message.description);
  await flushCandidates(message.from_id);
  logState(message.from_name, 'received SDP answer');
}

async function handleServerMessage(rawData) {
  const message = JSON.parse(rawData);
  switch (message.type) {
    case 'welcome':
      logState('WebSocket', `assigned participant id=${message.participant_id}`);
      break;
    case 'room-state':
      currentRoomId = message.room_id;
      activeRoom.textContent = currentRoomId;
      joinPanel.hidden = true;
      roomPanel.hidden = false;
      logState('Room', `joined ${currentRoomId}; existing peers=${message.participants.length}`);
      for (const participant of message.participants) await createAndSendOffer(participant);
      break;
    case 'peer-joined':
      peerNames.set(message.participant.participant_id, message.participant.participant_name);
      logState('Room', `${message.participant.participant_name} joined; waiting for newcomer offer`);
      break;
    case 'offer': await handleOffer(message); break;
    case 'answer': await handleAnswer(message); break;
    case 'ice-candidate':
      peerNames.set(message.from_id, message.from_name);
      await addOrQueueCandidate(message.from_id, message.candidate);
      break;
    case 'peer-left':
      logState('Room', `${message.participant_name} left`);
      removePeer(message.participant_id);
      break;
    case 'error':
      joinFeedback.textContent = `${message.code}: ${message.message}`;
      logState('Server error', `${message.code}: ${message.message}`);
      break;
    default: logState('WebSocket', `ignored unknown type=${message.type}`);
  }
}
```

The newcomer is the offerer. Existing participants wait and answer. This prevents competing offers during the initial join.

## Step 7 - Capture Media, Connect, and Join

**Exercise alignment:** This step implements **Part A, Task 2 - Local media** by requesting camera and microphone permission, showing a muted local preview, and handling permission or connection failure safely.

Continue `client/app.js`:

```javascript
async function acquireLocalMedia() {
  localStream = await navigator.mediaDevices.getUserMedia({
    audio: true,
    video: { width: 640, height: 360 }
  });
  localVideo.srcObject = localStream;
  logState('Media', `local tracks=${localStream.getTracks().map(track => track.kind).join(', ')}`);
}

async function connectToSignalingServer(serverUrl) {
  setSocketState('Connecting', 'connecting');
  socket = new WebSocket(serverUrl);
  socket.addEventListener('message', event => {
    handleServerMessage(event.data).catch(error => logState('Client error', error.message));
  });
  socket.addEventListener('close', event => {
    setSocketState('Disconnected', 'disconnected');
    logState('WebSocket', `closed code=${event.code}`);
  });
  socket.addEventListener('error', () => logState('WebSocket', 'connection error'));
  await new Promise((resolve, reject) => {
    socket.addEventListener('open', resolve, { once: true });
    socket.addEventListener('error', () => reject(new Error('Unable to reach signaling server.')), { once: true });
  });
  setSocketState('Connected', 'connected');
  logState('WebSocket', `readyState=${socket.readyState} OPEN`);
}

joinForm.addEventListener('submit', async event => {
  event.preventDefault();
  joinFeedback.textContent = '';
  const name = participantNameInput.value.trim();
  const roomId = roomCodeInput.value.trim().toUpperCase();
  try {
    await acquireLocalMedia();
    await connectToSignalingServer(serverUrlInput.value.trim());
    localLabel.textContent = name;
    sendSignal({ type: 'join-room', participant_name: name, room_id: roomId });
  } catch (error) {
    joinFeedback.textContent = error.message;
    setSocketState('Disconnected', 'disconnected');
    for (const track of localStream?.getTracks() ?? []) track.stop();
    localStream = null;
  }
});
```

## Step 8 - Add Controls and Cleanup

**Exercise alignment:** This step completes **Part A, Task 5 - Voice/video room controls** and **Task 6 - Cleanup and rejoining** by implementing mute, camera on/off, leave-room behavior, track stopping, peer closure, and interface cleanup.

Finish `client/app.js`:

```javascript
muteButton.addEventListener('click', () => {
  const track = localStream?.getAudioTracks()[0];
  if (!track) return;
  track.enabled = !track.enabled;
  muteButton.textContent = track.enabled ? 'Mute microphone' : 'Unmute microphone';
  logState('Media', `microphone enabled=${track.enabled}`);
});

cameraButton.addEventListener('click', () => {
  const track = localStream?.getVideoTracks()[0];
  if (!track) return;
  track.enabled = !track.enabled;
  cameraButton.textContent = track.enabled ? 'Turn camera off' : 'Turn camera on';
  logState('Media', `camera enabled=${track.enabled}`);
});

function closeEveryPeer() {
  for (const remoteId of [...peers.keys()]) removePeer(remoteId);
}

function leaveRoom() {
  sendSignal({ type: 'leave-room' });
  closeEveryPeer();
  for (const track of localStream?.getTracks() ?? []) track.stop();
  localStream = null;
  localVideo.srcObject = null;
  if (socket?.readyState === WebSocket.OPEN) socket.close(1000, 'Participant left');
  currentRoomId = null;
  roomPanel.hidden = true;
  joinPanel.hidden = false;
  participantCount.textContent = '1';
  muteButton.textContent = 'Mute microphone';
  cameraButton.textContent = 'Turn camera off';
  logState('Cleanup', 'local peers closed, media stopped, and interface cleared');
}

leaveButton.addEventListener('click', leaveRoom);
window.addEventListener('pagehide', () => {
  if (currentRoomId) sendSignal({ type: 'leave-room' });
  closeEveryPeer();
  for (const track of localStream?.getTracks() ?? []) track.stop();
  if (socket?.readyState === WebSocket.OPEN) socket.close(1000, 'Page hidden');
});

// Activity-only helpers for the required invalid-message tests.
window.labTools = {
  sendRaw(value) {
    if (socket?.readyState === WebSocket.OPEN) socket.send(value);
  },
  sendSignal
};
```

`window.labTools` is included only so students can perform the invalid-JSON and invalid-target tests from the browser developer console. Production applications should not expose internal signaling helpers globally.

## Step 9 - Install the Project on the Joiner Laptop

**Exercise alignment:** This setup is required before completing the two-laptop tests in **Part B - Required Paired Test Matrix**.

Copy the complete `webrtc-room` folder to the Joiner laptop. Inside the copied folder, run:

```bash
npm install
```

The Joiner does not run `npm run signal`; the Joiner only runs the local web client.

## Step 10 - Start the Host Services

**Exercise alignment:** This step prepares the Host for **Part B, Tests 1-2**, covering a signaling server listening on `0.0.0.0` and room registration from both laptops.

Open two Host terminals in the project folder.

Terminal 1:

```bash
npm run signal
```

Terminal 2:

```bash
npm run client
```

Open the displayed Vite localhost address, normally `http://localhost:5173`. The Host uses:

```text
ws://localhost:3000
```

Find the Host LAN IPv4 address using `ipconfig` on Windows or `ifconfig` on macOS/Linux. Look for an address similar to `192.168.x.x` or `10.x.x.x`. Allow Node.js or port `3000` through the firewall when prompted.

## Step 11 - Start the Joiner Client

**Exercise alignment:** This step completes the network arrangement required by **Part B, Tests 1-2** and establishes the Joiner's connection to the Host signaling server.

On the Joiner laptop:

```bash
cd webrtc-room
npm run client
```

Open the Joiner's own localhost URL. In the signaling field, use the Host address, for example:

```text
ws://192.168.1.25:3000
```

The Joiner opens the page through `localhost`; only the WebSocket field points to the Host. Both students enter the same room code and different participant names.

## Step 12 - Required Paired Procedure

**Exercise alignment:** Use this procedure to complete **Part B - Required Paired Test Matrix**, particularly Tests 2 and 4-10: room registration, offer/answer exchange, ICE exchange, two-way media, controls, departure, and rejoining.

### Host

1. Connect both laptops to the same LAN.
2. Record the Host IPv4 address.
3. Start the signaling server and local Vite client.
4. Use `ws://localhost:3000`.
5. Grant media permission and join first.

### Joiner

1. Start the local Vite client.
2. Use `ws://HOST_LAN_IP:3000`.
3. Grant media permission.
4. Enter the Host's room code and join second.

### Both students

1. Confirm both local previews.
2. Observe room registration.
3. Confirm offer, answer, and ICE logs.
4. Wait for `connectionState=connected`.
5. Verify two-way voice and video.
6. Test mute/unmute and camera off/on.
7. Have the Joiner leave and verify Host cleanup.
8. Rejoin and confirm a fresh connection.
9. Record observations and screenshots.

## Step 13 - Major Exercise: Connection-Lifecycle Observation

**Exercise alignment:** This step directly completes **Part C - Major Connection Lifecycle Observation** in the companion exercise, including its observation table and required interpretation.

Complete the table after WebSocket opening, during negotiation, after connection, and after the Joiner leaves.

| Observation | Host | Joiner |
|---|---|---|
| WebSocket state after connection |  |  |
| Signaling-state sequence |  |  |
| ICE-gathering-state sequence |  |  |
| ICE-connection-state sequence |  |  |
| Overall connection-state sequence |  |  |
| Local candidate types discovered |  |  |
| Remote tracks received |  |  |
| Media working |  |  |
| State after partner leaves |  |  |
| Resources removed or stopped |  |  |

Record sequences in order, such as `new -> checking -> connected`. Browser and network differences are acceptable. ICE may end at `connected` or `completed`, and a LAN may expose only `host` candidates.

Write three to five sentences that identify a shared sequence, one offerer/answerer difference, and how the final states prove signaling, connectivity, and media succeeded.

## Step 14 - Required Screenshot Evidence

This step provides the evidence required under **Part C - Major Connection Lifecycle Observation, Required Screenshot Set** in the activity. Use the screenshots produced in this step when completing that part of the exercise; a separate screenshot set is not required.

Submit screenshots of:

1. **Signaling server and room registration** — server plus registered room/participants.
2. **Offer, answer, and ICE logs** — concise activity without complete SDP or credentials.
3. **Final connected voice/video call** — identities, room, remote media, and connected state.
4. **Mute or camera-off control test** — selected control and visible result.
5. **Cleanup after the Joiner leaves** — remote tile removed and participant/peer state cleaned.

Each caption contains the screenshot number, test stage, observed result, and capturing participant's name.

## Step 15 - Test Errors and Recovery

**Exercise alignment:** These checks support **Part B - Required Paired Test Matrix**, especially room isolation, invalid signaling input, invalid targets, departure cleanup, and successful rejoining. Record each applicable result in the test matrix.

1. Enter an invalid room code; expect `INVALID_ROOM`.
2. Enter a one-character name; expect `INVALID_NAME`.
3. Try a fifth participant; expect `ROOM_FULL`.
4. Use different room codes; confirm no call forms.
5. Use an incorrect Host IP; expect WebSocket failure.
6. Deny media permission; confirm the client does not join.
7. Make the Joiner leave; confirm Host cleanup.
8. Rejoin; confirm a new offer, answer, ICE exchange, and connection.
9. While connected, open the browser developer console and run `window.labTools.sendRaw('{bad json')`; expect `INVALID_JSON` while the server remains running. Record this as **Part B, Test 11**.
10. Run the following in the developer console; expect `INVALID_TARGET`. Record this as **Part B, Test 12**.

```javascript
window.labTools.sendSignal({
  type: 'offer',
  target_id: 'participant-does-not-exist',
  description: { type: 'offer', sdp: 'activity-test' }
});
```

These two tests intentionally send invalid application messages. Perform them only against the pair's classroom signaling server.

## Troubleshooting Checklist

### WebSocket does not connect

- Confirm the Host ran `npm run signal` and the server listens on `0.0.0.0:3000`.
- Recheck the Host IPv4 address and `ws://HOST_IP:3000`.
- Confirm both laptops use the same LAN.
- Allow Node.js or port `3000` through the Host firewall.
- Check whether the LAN blocks device-to-device communication.

### Camera or microphone does not open

- Open each client from its own `http://localhost:5173`.
- Check browser permissions and available devices.
- Close applications that exclusively control the camera or microphone.

### Signaling works but media does not connect

- Confirm both students use the same room.
- Inspect offer, answer, and ICE logs.
- Confirm queued candidates are later added.
- Try another LAN or hotspot if peer traffic is filtered.
- Restrictive/public networks may require instructor-provided TURN credentials.

### Echo occurs or rejoining fails

- Use headphones and keep local video muted.
- Confirm the old socket closed, tracks stopped, and Host logged departure.
- Refresh after a coding error leaves stale state.

## Optional Multi-Participant Extension

**Exercise alignment:** This section supports **Part A, Task 7 - Multi-participant readiness**. Students must explain the design, but an actual third or fourth participant remains optional.

Another pair may join the same room after the graded run. Each browser should add remote tiles and create one peer connection for every other participant without restarting existing calls. Limit the mesh to four participants. Larger Discord-like rooms normally use an SFU instead of a browser mesh.

## Completion Checklist

- [ ] Both laptops contain the complete project and dependencies.
- [ ] The signaling server listens on `0.0.0.0:3000`.
- [ ] Both clients load through `localhost`.
- [ ] Host and Joiner register in one room.
- [ ] Local and remote media appear.
- [ ] Offer, answer, and ICE exchange succeeds.
- [ ] Both peer states reach `connected`.
- [ ] Microphone and camera controls work.
- [ ] Departure removes the remote tile and peer.
- [ ] Rejoining creates a fresh connection.
- [ ] Both students complete the observation table.
- [ ] The pair captures exactly five screenshots.

For the assessed tasks, required evidence, questions, and submission requirements, refer to the activity document.

## References

- W3C, *WebRTC: Real-Time Communication in Browsers*: <https://www.w3.org/TR/webrtc/>
- MDN, *Signaling and video calling*: <https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Signaling_and_video_calling>
- MDN, *Lifetime of a WebRTC session*: <https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Session_lifetime>
- MDN, *MediaDevices.getUserMedia()*: <https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia>
- RFC 8825, *Overview: Real-Time Protocols for Browser-Based Applications*: <https://www.rfc-editor.org/rfc/rfc8825.html>
- RFC 8835, *Transports for WebRTC*: <https://www.rfc-editor.org/rfc/rfc8835.html>
- `ws` documentation and examples: <https://github.com/websockets/ws>
