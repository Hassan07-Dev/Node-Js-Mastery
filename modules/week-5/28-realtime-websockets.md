# Real-Time Apps with WebSockets & Socket.IO

Everything you have built so far in this course follows one fundamental shape: a client sends an HTTP **request**, your Express server computes a **response**, and the connection is done. That request/response model — pull-based, client-initiated, stateless, short-lived — is the bedrock of the web, and it is *perfect* for fetching a web page, submitting a form, or hitting a REST endpoint. But it has a hard structural limit: **the server can never speak first.** If new data appears on the server (a chat message, a stock tick, another player's move, a "your order shipped" event), the server has no way to push it to a client that is just sitting there. The client must ask.

Real-time applications — chat, collaborative editors, multiplayer games, live dashboards, notifications, presence ("who's online"), live sports scores — invert that assumption. They need a **persistent, bidirectional, low-latency channel** where either side can send a message at any moment. That is exactly what the **WebSocket** protocol provides, and what **Socket.IO** wraps in a batteries-included, production-grade library.

In this module you will build a precise mental model of *why* HTTP alone is not enough, work through the spectrum of techniques people invented to fake real-time over HTTP (polling, long-polling, Server-Sent Events), then learn the WebSocket protocol itself via the lightweight `ws` library, before going deep on Socket.IO — the de-facto standard for real-time Node apps. We finish by scaling Socket.IO across multiple processes with Redis and building a complete chat application that ties every concept together.

> TIP: As in the rest of the course we use the `node:` prefix for built-in modules (e.g. `require('node:http')`) so it is unambiguous we mean the built-in. `ws`, `socket.io`, and `socket.io-client` are npm packages, imported by plain name.

## What You'll Learn

- **Why the HTTP request/response model cannot do real-time**, and the precise trade-offs between **polling, long-polling, Server-Sent Events (SSE), and WebSockets**.
- **How the WebSocket protocol actually works** — the HTTP `Upgrade` handshake, frames, the `ws://`/`wss://` schemes — and how to build a server and browser client with the **`ws`** library.
- **What Socket.IO adds on top of raw WebSockets**: automatic reconnection, transport fallback, multiplexing, rooms, namespaces, acknowledgements, and a richer event API — and the crucial fact that **Socket.IO is *not* WebSocket** and clients are not interchangeable.
- How to **stand up a Socket.IO server** (standalone and attached to an existing Express/`http` server) and **connect a browser and a Node client**.
- **Emitting and listening to custom events**, sending structured data both ways, and the full **emit cheat sheet** (to one socket, to everyone, to everyone-except-sender, to a room).
- **Broadcasting, rooms, and namespaces** — what each is for and how they compose.
- **Acknowledgements (callbacks)** to get request/response semantics over a socket, including timeouts and the promise-based API.
- Handling **`connection`/`disconnect`** robustly and implementing **presence** (an accurate online-users list).
- **Authenticating socket connections** with handshake middleware and **JWTs**.
- **Scaling Socket.IO across processes and machines** with the **Redis adapter** (and why you need it the moment you run more than one Node process).
- A complete, runnable **chat application** that combines events, rooms, presence, acks, and auth.

---

## 1. Why HTTP Request/Response Is Not Enough

Let us be precise about the limitation, because interviewers love this question and a vague answer ("HTTP is slow") will sink you.

The defining characteristic of HTTP/1.1 and HTTP/2 at the application level is that **communication is initiated by the client**. A server cannot originate a message to a client out of the blue; it can only respond to a request the client has already made. The connection's *direction of initiation* is one-way (client → server), even though both bytes flow both ways within a single transaction.

Consider a chat app. Alice and Bob are both looking at the same conversation. Alice sends a message. Your server now *has* new data destined for Bob's screen — but Bob's browser is not currently asking for anything. With plain HTTP, the server is stuck holding a message it cannot deliver. The only way Bob ever sees it is if Bob's browser **asks again**. So the entire history of "real-time on the web" is a history of clever ways to get the client to ask in a way that *feels* like the server pushed.

There are four main approaches. Here is the landscape before we dive into each:

| Technique | Direction | Connection model | Latency | Server overhead | Browser API | Best for |
|---|---|---|---|---|---|---|
| **Short polling** | Client pulls | New request every N seconds | Up to N seconds | High (constant requests) | `fetch`/`setInterval` | Simple, low-frequency updates; legacy support |
| **Long polling** | Client pulls, server holds | Request held open until data or timeout | Near-instant | Medium–High | `fetch` | Real-time-ish without WebSockets; broad compatibility |
| **Server-Sent Events (SSE)** | Server → client only | One long-lived HTTP response, streamed | Near-instant | Low–Medium | `EventSource` | Server-to-client streams (notifications, feeds, tickers) |
| **WebSockets** | Full duplex (both ways) | One persistent TCP connection | Lowest | Low per-connection | `WebSocket` | Chat, games, collaboration, anything bidirectional |

### 1.1 Short polling

The client asks "anything new?" on a fixed interval.

```js
// Browser: short polling every 3 seconds.
setInterval(async () => {
  const res = await fetch('/api/messages?since=' + lastSeenId);
  const messages = await res.json();
  messages.forEach(renderMessage);            // render whatever is new
}, 3000);
```

- **Pros:** trivially simple; works everywhere; stateless on the server.
- **Cons:** wasteful — most requests return *nothing*. Latency is bounded by the interval (set it to 3s and a message can be 3s late). Cutting the interval to feel "real-time" multiplies request volume and crushes the server. It is the worst of both worlds at scale: high overhead *and* high latency.

### 1.2 Long polling

The client makes a request, and the server **does not respond immediately**. It holds the connection open until either new data is available (respond instantly) or a timeout elapses (respond empty). The client immediately re-requests. This eliminates the latency floor of short polling — data is delivered the instant it exists — at the cost of keeping connections open.

```js
// Browser: long polling — re-issues a request as soon as the previous one resolves.
async function poll() {
  try {
    // The server will hold this request open until there's data (or it times out).
    const res = await fetch('/api/messages/longpoll?since=' + lastSeenId);
    if (res.ok) {
      const messages = await res.json();
      messages.forEach(renderMessage);
    }
  } catch (err) {
    await new Promise((r) => setTimeout(r, 1000)); // back off on error
  } finally {
    poll(); // immediately wait for the next batch
  }
}
poll();
```

Long polling powered real-time chat (including early Facebook Chat and the original Socket.IO fallback) for years. It is still the most compatible real-time transport, and — importantly — it is exactly what **Socket.IO falls back to** when WebSockets are unavailable.

> WARNING — Pitfall: Long polling looks like a persistent connection but it is really a rapid sequence of separate HTTP requests. Each one re-runs your middleware, re-parses cookies, re-authenticates. It does not give you a true push channel, and message ordering across reconnects is your problem to solve (hence the `since` cursor).

### 1.3 Server-Sent Events (SSE)

SSE is a real browser standard (`EventSource`) for **server-to-client streaming over a single, long-lived HTTP response**. The server sets `Content-Type: text/event-stream` and keeps writing `data:` lines forever; the browser parses them into events. It includes automatic reconnection and an event-id mechanism for resuming.

```js
// server-sse.js — a minimal SSE endpoint with the http module.
const http = require('node:http');

http.createServer((req, res) => {
  if (req.url === '/events') {
    res.writeHead(200, {
      'Content-Type': 'text/event-stream', // the SSE content type
      'Cache-Control': 'no-cache',
      Connection: 'keep-alive',
    });
    // Push a timestamp every second over the SAME response stream.
    const id = setInterval(() => {
      res.write(`data: ${JSON.stringify({ time: Date.now() })}\n\n`); // note the double \n
    }, 1000);
    req.on('close', () => clearInterval(id)); // stop when the client disconnects
    return;
  }
  res.writeHead(404).end();
}).listen(3000);
```

```js
// Browser: consuming SSE is a one-liner.
const source = new EventSource('/events');
source.onmessage = (event) => {
  const payload = JSON.parse(event.data);
  console.log('server pushed:', payload);
};
// Reconnection is automatic; you do not write a reconnect loop.
```

- **Pros:** dead simple; native browser support with auto-reconnect; runs over ordinary HTTP (great with proxies/firewalls); ideal for one-way feeds (notifications, live scores, log tails, AI token streaming).
- **Cons:** **one direction only** (server → client). To send client → server you make a normal HTTP request alongside it. Limited to text (no binary). Over HTTP/1.1, browsers cap connections per domain (~6), which constrains many SSE streams at once (HTTP/2 fixes this with multiplexing).

> TIP: If your requirement is genuinely *server pushes to client and the client rarely talks back* — dashboards, notifications, streaming LLM responses — SSE is often the right, simplest tool. Reach for WebSockets when you truly need **bidirectional** low-latency messaging.

### 1.4 WebSockets

WebSockets give you a **single, persistent, full-duplex TCP connection** between client and server. After an initial HTTP handshake, the connection is "upgraded" and both sides can send messages independently, at any time, with minimal per-message overhead. This is the foundation for everything else in this module.

The conceptual leap: with HTTP you have *N* short transactions; with WebSockets you have *one long-lived conversation*. The server keeps an object representing each connected client and can call `.send()` on it whenever it likes.

---

## 2. The WebSocket Protocol and the `ws` Library

### 2.1 How the protocol works

A WebSocket connection **starts life as an HTTP request**. The client sends a normal `GET` with special headers asking to switch protocols:

```
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

If the server agrees, it replies with `101 Switching Protocols`:

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

From that moment the TCP connection is no longer HTTP — it speaks the WebSocket framing protocol (RFC 6455). Key facts to internalize:

- The handshake reuses the existing HTTP port, so WebSockets work on **80/443** and pass through most firewalls and load balancers that already allow web traffic.
- The URL scheme is **`ws://`** (unencrypted) or **`wss://`** (TLS, the WebSocket equivalent of HTTPS). **Always use `wss://` in production.**
- Data travels in lightweight **frames** with a tiny header (as little as 2 bytes), not as repeated HTTP requests — that is why per-message overhead is so low.
- It is **full-duplex**: both sides can send simultaneously. There is no notion of "request" and "response" at the protocol level — just messages.
- The protocol has built-in **ping/pong** control frames for keepalive and liveness detection.

> WARNING — Pitfall: Because a WebSocket starts as an HTTP `Upgrade` request and then *leaves* HTTP behind, **HTTP-centric infrastructure can break it.** Reverse proxies (nginx), load balancers, and CDNs must be explicitly configured to pass the `Upgrade`/`Connection` headers and to allow long-lived connections, or the handshake fails or the socket dies after an idle timeout.

### 2.2 The `ws` library — a server and a client

[`ws`](https://github.com/websockets/ws) is the most popular, fast, spec-compliant WebSocket implementation for Node. It does **only** WebSockets — no rooms, no reconnection, no fallbacks. It is the right tool when you want the raw protocol with minimal abstraction (and it is, in fact, what Socket.IO uses under the hood on the server).

```bash
npm install ws
```

A minimal echo server:

```js
// ws-server.js — a raw WebSocket echo server using the `ws` library.
const { WebSocketServer } = require('ws');

const wss = new WebSocketServer({ port: 8080 }); // listens on ws://localhost:8080

wss.on('connection', (socket, req) => {
  // `socket` represents ONE connected client. `req` is the upgrade HTTP request.
  console.log('client connected from', req.socket.remoteAddress);

  // `message` is a Buffer (or array of Buffers). Convert to string for text.
  socket.on('message', (data, isBinary) => {
    const text = isBinary ? data : data.toString();
    console.log('received:', text);
    socket.send(`echo: ${text}`); // send a message back to THIS client
  });

  socket.on('close', (code, reason) => {
    console.log('client disconnected', code, reason.toString());
  });

  socket.on('error', (err) => console.error('socket error', err));

  socket.send('welcome! send me something.'); // server speaks first — impossible with plain HTTP
});

console.log('WebSocket server listening on ws://localhost:8080');
```

The matching **browser** client uses the native `WebSocket` API (no library needed):

```html
<!-- Browser client — uses the built-in WebSocket constructor. -->
<script>
  const ws = new WebSocket('ws://localhost:8080');

  ws.addEventListener('open', () => {
    console.log('connected');
    ws.send('hello from the browser'); // client can send any time after open
  });

  ws.addEventListener('message', (event) => {
    console.log('server says:', event.data); // event.data is a string (or Blob for binary)
  });

  ws.addEventListener('close', () => console.log('connection closed'));
  ws.addEventListener('error', (err) => console.error('ws error', err));
</script>
```

A **Node** client (the `ws` package also ships a client):

```js
// ws-client.js — connecting from Node with the `ws` package.
const WebSocket = require('ws');

const ws = new WebSocket('ws://localhost:8080');
ws.on('open', () => ws.send('hello from node'));
ws.on('message', (data) => console.log('server says:', data.toString()));
ws.on('close', () => console.log('closed'));
```

### 2.3 Broadcasting and keepalive with `ws`

`ws` gives you a primitive `wss.clients` set; broadcasting is manual:

```js
// Broadcast a message to every connected client.
function broadcast(wss, message) {
  for (const client of wss.clients) {
    if (client.readyState === client.OPEN) { // only send to live sockets
      client.send(message);
    }
  }
}
```

Detecting dead connections is also your job. TCP can silently die (laptop sleeps, network drops) without a `close` event. The protocol's ping/pong solves this — you must implement the loop yourself:

```js
// Heartbeat: terminate sockets that stop answering pings.
function heartbeat() { this.isAlive = true; }

wss.on('connection', (socket) => {
  socket.isAlive = true;
  socket.on('pong', heartbeat); // browser auto-replies to ping with pong
});

const interval = setInterval(() => {
  for (const socket of wss.clients) {
    if (socket.isAlive === false) return socket.terminate(); // missed last ping → kill it
    socket.isAlive = false;
    socket.ping(); // request a pong before the next tick
  }
}, 30000);

wss.on('close', () => clearInterval(interval));
```

> BEST PRACTICE: If you use raw `ws` in production you must hand-roll heartbeats, reconnection (on the client), message framing/protocol design, and broadcasting groups. That is a *lot* of plumbing. The moment you need any of those, Socket.IO has already solved them — which is exactly why it exists.

### 2.4 Attaching `ws` to an existing HTTP/Express server

You rarely want a separate port. Share your Express server's port by handling the `upgrade` event yourself, or let `ws` do it:

```js
// Share one port between Express (HTTP) and ws (WebSocket).
const http = require('node:http');
const express = require('express');
const { WebSocketServer } = require('ws');

const app = express();
app.get('/', (req, res) => res.send('HTTP and WS share this port'));

const server = http.createServer(app);        // Express handles normal HTTP
const wss = new WebSocketServer({ server });  // ws handles upgrade requests on the same server

wss.on('connection', (socket) => socket.send('hi over websocket'));

server.listen(3000); // ws://localhost:3000 and http://localhost:3000 coexist
```

---

## 3. Socket.IO: What It Adds Over Raw WebSockets

Socket.IO is a real-time **engine + protocol + library** built on top of WebSocket (and HTTP long-polling). It is the most popular real-time solution in the Node ecosystem because raw WebSockets, while elegant, leave you to build a small mountain of production concerns yourself. Here is what Socket.IO provides:

| Concern | Raw WebSocket (`ws`) | Socket.IO |
|---|---|---|
| **Automatic reconnection** | You write it | Built-in, with exponential backoff |
| **Transport fallback** | None (WS or nothing) | Falls back to HTTP long-polling when WS is blocked |
| **Named/custom events** | One `message` channel; you invent your own routing | `socket.on('eventName', ...)` first-class |
| **Acknowledgements** | Manual correlation IDs | Built-in callback acks (+ timeouts, promises) |
| **Rooms / groups** | Manual `Set` bookkeeping | `socket.join('room')` and `io.to('room').emit()` |
| **Namespaces** | None | `/admin`, `/chat` multiplexed over one connection |
| **Broadcasting** | Manual loop over clients | `socket.broadcast.emit()`, `io.emit()` |
| **Multi-server scaling** | Build your own pub/sub | Official Redis (and other) adapters |
| **Binary support** | Frames only | Transparent (Buffers, ArrayBuffer, Blob) |
| **Auto JSON encoding** | You `JSON.stringify`/`parse` | Automatic for objects |
| **Heartbeat / liveness** | You implement ping/pong | Built-in |

> WARNING — Pitfall (interview favorite): **Socket.IO is NOT WebSocket.** It is a *separate protocol* that *uses* WebSocket as one of its transports. A browser's native `WebSocket` cannot talk to a Socket.IO server, and a Socket.IO client cannot connect to a plain `ws` server. You must use `socket.io` on the server and `socket.io-client` (or the bundled browser client) on the client. They are matched pairs.

### 3.1 Engine.IO vs Socket.IO

Two layers, both maintained by the same team:

- **Engine.IO** is the low-level transport layer. It establishes the connection, handles the long-polling ⇆ WebSocket upgrade, and provides the heartbeat. It guarantees a connection can always be established (starting on long-polling, then *upgrading* to WebSocket if possible).
- **Socket.IO** is the high-level layer on top of Engine.IO that adds events, acknowledgements, broadcasting, rooms, namespaces, and the multi-server adapter API.

> TIP: This is why a Socket.IO connection in your Network tab often *starts* as `polling` (`transport=polling`) and then you see a WebSocket connection appear — Engine.IO connected immediately via long-polling and then silently upgraded. That upgrade is by design and is one of the reliability features you get for free.

---

## 4. Setting Up a Socket.IO Server and Connecting a Client

```bash
npm install socket.io        # server
npm install socket.io-client # for a Node client; the browser client is served automatically
```

### 4.1 Server attached to an Express/HTTP server (the standard setup)

```js
// server.js — Socket.IO attached to an Express + http server.
const http = require('node:http');
const express = require('express');
const { Server } = require('socket.io');

const app = express();
app.use(express.static('public')); // serves index.html + client JS

const server = http.createServer(app); // create the raw http server explicitly
const io = new Server(server, {
  cors: { origin: 'http://localhost:5173' }, // configure if client is on a different origin
});

io.on('connection', (socket) => {
  // This fires once per connected client. `socket` is that client's handle.
  console.log('a user connected:', socket.id); // socket.id is a unique, per-connection id

  socket.on('disconnect', (reason) => {
    console.log('user disconnected:', socket.id, reason);
  });
});

server.listen(3000, () => console.log('listening on http://localhost:3000'));
```

> WARNING — Pitfall: You must pass the **`http.Server`** to `new Server(server)`, not the Express `app`. Socket.IO needs to hook the server's `upgrade` event; the Express `app` is just a request handler. A very common bug is `new Server(app)` — it silently fails to handle upgrades.

> WARNING — Pitfall: **CORS.** Socket.IO's HTTP handshake is subject to the browser's same-origin policy. If your frontend is served from a different origin (different host/port — e.g. a Vite dev server on `:5173` talking to an API on `:3000`), you must set the `cors` option on the server *and* point the client at the full URL. Forgetting this produces a confusing "connection failing" with CORS errors in the console.

### 4.2 The browser client

Socket.IO serves its client bundle automatically at `/socket.io/socket.io.js`:

```html
<!-- public/index.html -->
<script src="/socket.io/socket.io.js"></script>
<script>
  // No URL needed when the page is served from the same origin as the server.
  const socket = io();

  socket.on('connect', () => {
    console.log('connected with id', socket.id);
  });

  socket.on('disconnect', (reason) => {
    console.log('disconnected:', reason);
  });
</script>
```

If the client is on a **different origin**, pass the server URL:

```js
// Different-origin client (e.g. a React/Vite app talking to an API server).
import { io } from 'socket.io-client';
const socket = io('http://localhost:3000', {
  withCredentials: true, // send cookies if you authenticate via cookie
});
```

### 4.3 A Node.js client

Handy for tests, server-to-server links, or bots:

```js
// node-client.js — a Socket.IO client running in Node.
const { io } = require('socket.io-client');

const socket = io('http://localhost:3000');
socket.on('connect', () => console.log('connected as', socket.id));
socket.emit('chat:message', 'hello from a node client');
```

### 4.4 Standalone server (no Express)

You do not need Express; Socket.IO can create its own HTTP server:

```js
// Standalone — Socket.IO creates and listens on its own server.
const { Server } = require('socket.io');
const io = new Server(3000, { cors: { origin: '*' } });
io.on('connection', (socket) => console.log('connected', socket.id));
```

> TIP: `socket.id` changes on every (re)connection — including after an automatic reconnect. **Never use `socket.id` as a stable user identity.** Use it for transient routing within a single connection only; derive real identity from your auth layer (Section 9).

---

## 5. Emitting and Listening to Custom Events

The heart of Socket.IO is the **event** model. Instead of one generic `message` channel (as in raw WS), you `emit` and `on` *named* events and pass any JSON-serializable arguments (Socket.IO encodes/decodes them for you; binary like `Buffer`/`ArrayBuffer` is supported transparently).

```js
// SERVER
io.on('connection', (socket) => {
  // Listen for a custom event emitted by this client.
  socket.on('chat:message', (text) => {
    console.log('got message:', text);
    // Send a custom event back to the SAME client.
    socket.emit('chat:ack', { received: text, at: Date.now() });
  });

  // Events can carry multiple, structured arguments.
  socket.on('cursor:move', (x, y) => {
    console.log(`cursor at ${x},${y}`);
  });
});
```

```js
// CLIENT
socket.emit('chat:message', 'Hello, server!'); // emit a custom event with a string payload
socket.on('chat:ack', (data) => console.log('server acked', data));

socket.emit('cursor:move', 120, 340); // multiple arguments are fine
```

Both directions are symmetric: server can `emit` to the client and the client can `emit` to the server, with identical syntax.

> BEST PRACTICE: **Namespace your event names** with a prefix and colon (`chat:message`, `game:move`, `presence:update`). It keeps a growing event surface organized, avoids collisions with reserved events, and reads clearly in logs. Reserved event names you must NOT use for your own events include `connect`, `connect_error`, `disconnect`, `disconnecting`, and `newListener`.

> WARNING — Pitfall: The **last argument of an `emit` is treated as an acknowledgement callback if it is a function.** So `socket.emit('save', data, () => {})` does *not* send the function as data — it registers an ack handler (Section 8). Never pass functions as plain payload.

### 5.1 Volatile, compression, and other emit modifiers

```js
// Volatile: drop the message if the socket is not ready (e.g. mid-reconnect).
// Great for high-frequency, disposable data like game positions where the next
// update makes a dropped one irrelevant.
socket.volatile.emit('game:tick', state);

// Disable per-message compression for tiny/already-compressed payloads.
socket.compress(false).emit('telemetry', sample);
```

---

## 6. Broadcasting, Rooms, and Namespaces

These three features are how you control *who receives* an emit. Understanding the targeting semantics precisely is essential — and a frequent interview topic.

### 6.1 The emit targeting cheat sheet

```js
io.on('connection', (socket) => {
  // 1. To the sender only:
  socket.emit('hello', data);

  // 2. To EVERYONE, including the sender:
  io.emit('announcement', data);

  // 3. To everyone EXCEPT the sender (this is "broadcast"):
  socket.broadcast.emit('user:joined', socket.id);

  // 4. To everyone in a room (including sender if they're in it):
  io.to('room42').emit('room:msg', data);

  // 5. To everyone in a room EXCEPT the sender:
  socket.to('room42').emit('room:msg', data);

  // 6. To multiple rooms at once (union — a socket in both gets ONE copy):
  io.to('room1').to('room2').emit('multi', data);

  // 7. To a specific socket by id (every socket auto-joins a room named after its id):
  io.to(someSocketId).emit('direct', data);

  // 8. To all EXCEPT a room:
  io.except('vip').emit('public', data);
});
```

> WARNING — Pitfall: `io.emit(...)` includes the sender; `socket.broadcast.emit(...)` excludes the sender. Mixing these up causes the classic "I see my own message twice" or "I never see other people's messages" bug. Likewise, `io.to(room).emit` includes a sender who is in the room, while `socket.to(room).emit` excludes the sender. Choose based on whether the originator should receive their own event.

### 6.2 Rooms

A **room** is a server-side label you attach to sockets so you can address a *group* with one emit. Rooms are an arbitrary string; a socket can be in many rooms; joining/leaving is cheap. Rooms exist **only on the server** — the client cannot see or join rooms directly; it asks the server (via an event) and the server decides.

```js
io.on('connection', (socket) => {
  socket.on('room:join', (roomName) => {
    socket.join(roomName);                 // add this socket to the room
    socket.to(roomName).emit('room:joined', { id: socket.id }); // tell others in the room
  });

  socket.on('room:leave', (roomName) => {
    socket.leave(roomName);                // remove from the room
  });

  socket.on('room:message', ({ room, text }) => {
    // Send to everyone in the room except the sender:
    socket.to(room).emit('room:message', { from: socket.id, text });
  });
});
```

Key room facts:

- **Every socket automatically joins a room named after its own `socket.id`.** That is how `io.to(socketId).emit()` works for direct messages.
- Rooms are created and destroyed automatically — a room exists as long as at least one socket is in it, and disappears when empty.
- When a socket disconnects, it leaves all its rooms automatically.
- You can inspect membership: `io.in('room').fetchSockets()` returns the matching socket instances; `io.sockets.adapter.rooms` is the low-level map.

```js
// How many clients are currently in a room?
const sockets = await io.in('room42').fetchSockets();
console.log(`${sockets.length} clients in room42`);
```

### 6.3 Namespaces

A **namespace** is a separate communication channel multiplexed over the *same* physical connection, identified by a path (`/admin`, `/chat`). Use namespaces to split your application into logical modules with **separate event handlers, middleware, and access control** — without opening extra TCP connections.

```js
// SERVER — define namespaces.
const chat = io.of('/chat');
chat.on('connection', (socket) => {
  socket.on('message', (m) => chat.emit('message', m)); // only /chat clients receive this
});

const admin = io.of('/admin');
admin.use((socket, next) => {
  // Namespace-scoped middleware — e.g. require an admin token here.
  if (socket.handshake.auth.role === 'admin') return next();
  next(new Error('forbidden'));
});
admin.on('connection', (socket) => {
  socket.emit('dashboard:init', getStats());
});
```

```js
// CLIENT — connect to a specific namespace.
const chatSocket = io('/chat');
const adminSocket = io('/admin', { auth: { role: 'admin' } });
```

> TIP: **Namespaces vs Rooms** — the distinction interviewers probe. *Namespaces* are coarse-grained, defined by the developer up front, each with its own middleware and handlers (think: separate "apps" — `/chat`, `/admin`, `/notifications`). *Rooms* are fine-grained, created dynamically at runtime, and exist *within* a namespace to group sockets (think: individual chat channels, game lobbies, document sessions). Rule of thumb: few namespaces, many rooms. The default namespace is `/`.

---

## 7. Acknowledgements: Request/Response Over Sockets

Events are fire-and-forget by default — `emit` does not tell you whether the other side received or processed the message. **Acknowledgements** give you request/response semantics: the *emitter* passes a callback, and the *receiver* invokes it (optionally with data) to reply. This is how you implement "save this and tell me if it worked" over a socket.

```js
// CLIENT — pass a callback as the LAST argument; the server calls it back.
socket.emit('order:create', { item: 'coffee', qty: 2 }, (response) => {
  if (response.ok) console.log('created order', response.id);
  else console.error('failed:', response.error);
});
```

```js
// SERVER — the handler receives the callback as the last parameter.
socket.on('order:create', (order, ack) => {
  try {
    const id = saveOrder(order);   // do the work
    ack({ ok: true, id });         // "respond" to the emitter
  } catch (err) {
    ack({ ok: false, error: err.message });
  }
});
```

### 7.1 Timeouts and the promise API

A naive ack callback waits forever if the server never replies (or the socket dies). Use `.timeout()`:

```js
// CLIENT — fail if no ack within 5 seconds.
socket
  .timeout(5000)
  .emit('order:create', order, (err, response) => {
    if (err) return console.error('timed out / no ack');
    console.log('server replied:', response);
  });
```

Combine `.timeout()` with `.emitWithAck()` for clean async/await:

```js
// CLIENT — promise-based ack with a timeout. Much nicer in async code.
try {
  const response = await socket.timeout(5000).emitWithAck('order:create', order);
  console.log('created:', response.id);
} catch (err) {
  console.error('no ack within 5s');
}
```

> WARNING — Pitfall: Acks are **not** guaranteed delivery. If the connection drops after the server processed the event but before the ack reached the client, the client sees a timeout even though the work succeeded. For anything that must not be applied twice (payments, "create order"), design your server handler to be **idempotent** — e.g. the client sends a client-generated request id and the server de-duplicates.

> BEST PRACTICE: Always set a `.timeout()` on acks you actually depend on. A hung ack with no timeout is a memory leak (the callback is retained) and a UX dead-end (a spinner that never stops).

---

## 8. Connect, Disconnect, and Presence (Online Users)

### 8.1 The connection lifecycle

```js
io.on('connection', (socket) => {
  // Fired after a successful connection (and after auth middleware passes).

  socket.on('disconnecting', (reason) => {
    // Fires BEFORE the socket leaves its rooms — socket.rooms is still populated here.
    // Perfect for "notify each room this user was in".
    for (const room of socket.rooms) {
      socket.to(room).emit('presence:left', { id: socket.id });
    }
  });

  socket.on('disconnect', (reason) => {
    // Fires after the socket has left all rooms. socket.rooms is now empty.
    console.log('gone:', socket.id, 'because', reason);
  });
});
```

Common `disconnect` reasons you should recognize:

| Reason | Meaning |
|---|---|
| `transport close` | The connection was lost (network drop, tab closed) |
| `ping timeout` | The client stopped responding to heartbeats |
| `client namespace disconnect` | The client called `socket.disconnect()` |
| `server namespace disconnect` | The server called `socket.disconnect()` |
| `transport error` | The transport encountered an error |

> TIP: Use **`disconnecting`** (not `disconnect`) when you need to notify the rooms a user was in, because `socket.rooms` is already empty by the time `disconnect` fires.

### 8.2 Presence — an accurate online-users list

"Presence" is the feature that shows who is online. The naive approach (count `connection` events) is wrong because **one user can have multiple connections** (multiple tabs, phone + laptop) and reconnections create new `socket.id`s. Track presence by **user identity**, not by socket, with a reference count.

```js
// presence.js — track online users robustly across multiple tabs/devices.
const online = new Map(); // userId -> Set of socketIds

function addConnection(userId, socketId) {
  if (!online.has(userId)) online.set(userId, new Set());
  online.get(userId).add(socketId);
  return online.get(userId).size === 1; // true => user just came online (first connection)
}

function removeConnection(userId, socketId) {
  const set = online.get(userId);
  if (!set) return false;
  set.delete(socketId);
  if (set.size === 0) {
    online.delete(userId);
    return true; // true => user just went fully offline (last connection closed)
  }
  return false;
}

const listOnline = () => [...online.keys()];

module.exports = { addConnection, removeConnection, listOnline };
```

```js
// Wiring presence into the server (userId comes from auth — see Section 9).
const { addConnection, removeConnection, listOnline } = require('./presence');

io.on('connection', (socket) => {
  const userId = socket.data.userId; // set during auth middleware

  if (addConnection(userId, socket.id)) {
    io.emit('presence:online', userId); // broadcast only on the FIRST connection
  }
  socket.emit('presence:list', listOnline()); // send the current roster to the newcomer

  socket.on('disconnect', () => {
    if (removeConnection(userId, socket.id)) {
      io.emit('presence:offline', userId); // broadcast only when the LAST connection closes
    }
  });
});
```

> WARNING — Pitfall: An **in-memory** presence map is per-process. With multiple Node processes behind a load balancer (Section 10), each process sees only its own connections, so your "online users" list is wrong. For multi-process presence, store the state in **Redis** (e.g. a hash/set per user with TTL) so every process shares one source of truth — and use the Redis adapter so the broadcasts reach all processes.

---

## 9. Authenticating Socket Connections

Sockets are long-lived, so you authenticate **once at connection time** using **handshake middleware**, then trust the established identity for the life of that connection (with the caveats below). Never trust a `userId` the client merely *claims* in event payloads — derive it from a verified token.

### 9.1 Handshake middleware

`io.use()` registers middleware that runs for every incoming connection *before* `connection` fires. Call `next()` to accept, or `next(new Error(...))` to reject.

```js
// SERVER — JWT authentication at the handshake.
const jwt = require('jsonwebtoken');

io.use((socket, next) => {
  // The client sends credentials in the `auth` object of the handshake.
  const token = socket.handshake.auth?.token;
  if (!token) return next(new Error('UNAUTHENTICATED'));

  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET); // throws if invalid/expired
    socket.data.userId = payload.sub;   // attach verified identity to the socket
    socket.data.role = payload.role;    // socket.data is the per-socket store for app data
    next();                             // accept the connection
  } catch (err) {
    next(new Error('INVALID_TOKEN'));   // reject — client gets a connect_error
  }
});
```

```js
// CLIENT — provide the token via `auth`.
const socket = io('http://localhost:3000', {
  auth: { token: localStorage.getItem('jwt') },
});

socket.on('connect_error', (err) => {
  // Rejections from auth middleware land here, with err.message you threw.
  if (err.message === 'INVALID_TOKEN') redirectToLogin();
});
```

> BEST PRACTICE: Send the token in the **`auth`** option, not the query string. Query strings are logged by proxies and servers and end up in access logs; `auth` is sent in the Engine.IO handshake payload, which is not logged as a URL. For browser flows where the token lives in an HttpOnly cookie, set `withCredentials: true` on the client and read the cookie from `socket.handshake.headers.cookie` on the server.

### 9.2 Token expiry on long-lived connections

A connection can outlive its JWT. The handshake validated the token once, but a socket open for hours will not re-check. Two robust patterns:

1. **Re-auth events:** the client periodically sends a fresh token (`socket.emit('auth:refresh', newToken)`); the server re-verifies and updates `socket.data`, disconnecting on failure.
2. **Server-initiated revocation:** when you revoke a user (logout, ban), look up their sockets and `disconnect()` them. The `socket.id` rooms + a `userId -> socketIds` map (your presence map) make this a one-liner.

```js
// Force-disconnect every socket of a revoked user.
function kickUser(io, userId, socketIds) {
  for (const id of socketIds) {
    io.sockets.sockets.get(id)?.disconnect(true); // true = close the underlying connection
  }
}
```

> WARNING — Pitfall: `io.use()` middleware runs **once per connection**, not per event. Do not put per-message authorization there. For per-event checks (e.g. "can this user post to this room?"), validate inside the event handler using `socket.data` identity, or use per-event middleware: `socket.use(([event, ...args], next) => { ... })`.

---

## 10. Scaling Socket.IO Across Processes with the Redis Adapter

This is the single most important production topic, and a guaranteed senior interview question.

### 10.1 The problem

A single Node process can only use one CPU core, so production runs **multiple Node processes** (via the `cluster` module, PM2, or multiple containers/pods behind a load balancer). But Socket.IO connections are **stateful and live in one process**: the set of connected sockets, rooms, and broadcast targets is **in-memory, local to that process**.

So with two processes A and B:

- A user connected to process A and a user connected to process B are invisible to each other.
- `io.emit('msg')` on process A reaches only A's clients; B's clients never see it.
- `io.to('room42').emit(...)` only reaches room42 members **on the same process**.

Your real-time app silently breaks: messages reach some users and not others depending on which process the load balancer happened to route them to.

### 10.2 The solution: an adapter + Redis pub/sub

An **adapter** is the component Socket.IO uses to route messages between servers. The default in-memory adapter only knows about the local process. The **Redis adapter** replaces it: when you broadcast, it publishes the message to Redis, and every other Socket.IO process — subscribed to the same Redis channels — receives it and forwards it to *its* local clients. The result: `io.emit`, rooms, and namespaces work correctly across all processes as if it were one server.

```bash
npm install @socket.io/redis-adapter redis
```

```js
// server.js — multi-process-ready Socket.IO with the Redis adapter.
const { Server } = require('socket.io');
const { createClient } = require('redis');
const { createAdapter } = require('@socket.io/redis-adapter');

const io = new Server(3000);

const pubClient = createClient({ url: 'redis://localhost:6379' });
const subClient = pubClient.duplicate(); // the adapter needs a separate subscriber client

await Promise.all([pubClient.connect(), subClient.connect()]);
io.adapter(createAdapter(pubClient, subClient)); // <- now all servers share broadcasts

io.on('connection', (socket) => {
  // io.emit / io.to(room).emit now reach clients on EVERY process, not just this one.
  socket.on('chat:message', (m) => io.emit('chat:message', m));
});
```

That is the entire change needed for broadcasts, rooms, and namespaces to work cluster-wide. Each process still holds its own connections, but the adapter fans out broadcasts via Redis.

### 10.3 Sticky sessions — the other half

There is a second, independent requirement many people miss. Socket.IO's Engine.IO layer **starts on HTTP long-polling and then upgrades to WebSocket**, and the long-polling phase uses *multiple HTTP requests* that must all reach the **same process** (because the session lives there). If your load balancer round-robins those handshake requests across processes, the handshake fails with errors like `Session ID unknown`.

The fix is **sticky sessions** (a.k.a. session affinity): the load balancer must route all requests from a given client to the same backend, typically by hashing the client IP or a cookie.

> WARNING — Pitfall: You need **both** the Redis adapter (so broadcasts cross processes) **and** sticky sessions (so each client's handshake stays on one process). They solve different problems. Setting up Redis but forgetting sticky sessions yields intermittent connection failures that are maddening to debug. Note: if you can *force* the WebSocket-only transport (`transports: ['websocket']` on the client), there is a single connection and sticky sessions are not required — but you lose the long-polling fallback.

```js
// Forcing WebSocket-only on the client (skips long-polling → no sticky-session need,
// at the cost of fallback compatibility). Use only if your infra guarantees WS works.
const socket = io('https://api.example.com', { transports: ['websocket'] });
```

### 10.4 Example: clustering on one machine

```js
// cluster-server.js — N worker processes sharing connections via Redis.
const cluster = require('node:cluster');
const os = require('node:os');

if (cluster.isPrimary) {
  for (let i = 0; i < os.availableParallelism(); i++) cluster.fork();
} else {
  // Each worker runs the Redis-adapter server from 10.2.
  require('./server');
}
// In production behind a load balancer you'd also need sticky sessions across machines;
// @socket.io/sticky helps when clustering on a single host.
```

---

## 11. A Small Chat Example Tying It Together

This complete, runnable example combines: JWT-ish auth at the handshake, namespaces is omitted for brevity but rooms (chat channels), presence, broadcasting, and acknowledgements. (Swap the toy `verifyToken` for real JWT verification from Section 9.)

```js
// chat-server.js — a complete chat server.
const http = require('node:http');
const express = require('express');
const { Server } = require('socket.io');

const app = express();
app.use(express.static('public'));
const server = http.createServer(app);
const io = new Server(server);

// --- Toy auth: in real life, jwt.verify(token, secret). ---
function verifyToken(token) {
  if (!token) throw new Error('no token');
  // Pretend the token IS "username:userId" for the demo.
  const [username, userId] = token.split(':');
  if (!username || !userId) throw new Error('bad token');
  return { username, userId };
}

// --- Presence: userId -> Set<socketId>, and a lookup of names. ---
const online = new Map();
const names = new Map(); // userId -> username

io.use((socket, next) => {
  try {
    const { username, userId } = verifyToken(socket.handshake.auth?.token);
    socket.data.userId = userId;
    socket.data.username = username;
    names.set(userId, username);
    next();
  } catch (err) {
    next(new Error('UNAUTHENTICATED'));
  }
});

io.on('connection', (socket) => {
  const { userId, username } = socket.data;

  // Presence bookkeeping (handles multiple tabs per user).
  if (!online.has(userId)) online.set(userId, new Set());
  const firstConn = online.get(userId).size === 0;
  online.get(userId).add(socket.id);
  if (firstConn) io.emit('presence:online', { userId, username });
  socket.emit('presence:list', [...online.keys()].map((id) => ({ userId: id, username: names.get(id) })));

  // Join a room (chat channel), and ack with the current member count.
  socket.on('room:join', async (room, ack) => {
    socket.join(room);
    socket.to(room).emit('system', `${username} joined #${room}`);
    const members = await io.in(room).fetchSockets();
    ack?.({ ok: true, room, members: members.length });
  });

  socket.on('room:leave', (room) => {
    socket.leave(room);
    socket.to(room).emit('system', `${username} left #${room}`);
  });

  // Send a message to a room; ack so the client can confirm delivery.
  socket.on('room:message', ({ room, text }, ack) => {
    const message = { from: username, userId, text, at: Date.now() };
    io.to(room).emit('room:message', message); // include sender so their UI updates uniformly
    ack?.({ ok: true, at: message.at });
  });

  socket.on('disconnect', () => {
    const set = online.get(userId);
    set?.delete(socket.id);
    if (set && set.size === 0) {
      online.delete(userId);
      io.emit('presence:offline', { userId, username });
    }
  });
});

server.listen(3000, () => console.log('chat on http://localhost:3000'));
```

```html
<!-- public/index.html — a minimal chat client. -->
<!doctype html>
<input id="name" placeholder="username" />
<button id="login">Connect</button>
<div id="app" hidden>
  <input id="room" value="general" />
  <button id="join">Join</button>
  <ul id="log"></ul>
  <input id="msg" placeholder="message" />
  <button id="send">Send</button>
</div>
<script src="/socket.io/socket.io.js"></script>
<script>
  let socket;
  const log = (t) => {
    const li = document.createElement('li');
    li.textContent = t;
    document.getElementById('log').append(li);
  };

  document.getElementById('login').onclick = () => {
    const username = document.getElementById('name').value || 'anon';
    // Toy token "username:userId"; replace with a real JWT in production.
    socket = io({ auth: { token: `${username}:${Date.now()}` } });

    socket.on('connect', () => { document.getElementById('app').hidden = false; });
    socket.on('connect_error', (e) => log('auth failed: ' + e.message));
    socket.on('system', (t) => log('* ' + t));
    socket.on('room:message', (m) => log(`${m.from}: ${m.text}`));
    socket.on('presence:online', (u) => log(`+ ${u.username} online`));
    socket.on('presence:offline', (u) => log(`- ${u.username} offline`));
  };

  document.getElementById('join').onclick = () => {
    const room = document.getElementById('room').value;
    // Use an ack to confirm the join and learn the member count.
    socket.emit('room:join', room, (res) => log(`joined #${res.room} (${res.members} online)`));
  };

  document.getElementById('send').onclick = () => {
    const room = document.getElementById('room').value;
    const text = document.getElementById('msg').value;
    socket.timeout(5000).emit('room:message', { room, text }, (err, res) => {
      if (err) log('(message not confirmed)');
      else document.getElementById('msg').value = '';
    });
  };
</script>
```

To make this multi-process-ready, add the Redis adapter from Section 10 and ensure sticky sessions on your load balancer.

---

## Common Pitfalls & Gotchas

- **Confusing Socket.IO with WebSocket.** They are different protocols; clients/servers are not interchangeable. A native browser `WebSocket` cannot talk to a Socket.IO server.
- **Passing the Express `app` instead of the `http.Server`** to `new Server()`. Socket.IO must hook the server's `upgrade` event; passing `app` breaks the WebSocket transport.
- **Forgetting CORS** when the client is on a different origin. Set `cors` on the server *and* the full URL (plus `withCredentials` if using cookies) on the client.
- **`io.emit` vs `socket.broadcast.emit`** — the former includes the sender, the latter excludes it. The cause of "double messages" and "I never see others' messages."
- **Using `socket.id` as a user identity.** It changes on every reconnect. Derive identity from auth and keep a `userId -> socketIds` map.
- **In-memory rooms/presence across multiple processes.** Without the Redis adapter, broadcasts and rooms only work within one process; users on different processes can't reach each other.
- **Forgetting sticky sessions** when scaling. The long-polling handshake needs all requests on one process or you get `Session ID unknown`.
- **Acks without timeouts.** A never-arriving ack leaks the callback and hangs the UI. Always `.timeout()`.
- **Treating acks as guaranteed delivery.** A dropped connection after processing yields a false timeout — make sensitive handlers idempotent.
- **Auth only at handshake, never revisited.** Long-lived sockets outlive JWTs; add a refresh/revocation strategy.
- **Heavy/blocking work inside event handlers.** Socket.IO runs on the single-threaded event loop; a synchronous CPU-heavy handler stalls *every* connection. Offload to workers/queues.
- **Memory leaks from listeners.** On the client, re-registering `socket.on(...)` on every component mount (e.g. React) stacks duplicate handlers. Register once and clean up with `socket.off(...)`.
- **Sending huge payloads.** Socket.IO buffers messages; large or unbounded payloads (file uploads via socket) cause memory pressure. Use HTTP/object storage for big blobs.

## Best Practices

- **Attach Socket.IO to your existing HTTP server** and reuse the port (80/443). Use `wss://` in production.
- **Namespace event names** (`chat:message`, `game:move`) and keep a documented event contract between client and server.
- **Authenticate at the handshake** via `socket.handshake.auth`, store verified identity on `socket.data`, and never trust client-claimed identity in payloads.
- **Validate every inbound event's payload** server-side (shape, size, authorization) — sockets are an attack surface just like HTTP routes.
- **Use acknowledgements with timeouts** for operations the client must confirm; design idempotent handlers for critical actions.
- **Use rooms for dynamic grouping and namespaces for static modules.** Few namespaces, many rooms.
- **Plan for scale from day one:** the Redis adapter + sticky sessions. Keep shared state (presence, sessions) in Redis, not process memory.
- **Keep handlers fast and non-blocking;** offload CPU-heavy or long work to queues/workers.
- **Handle reconnection on the client** gracefully — re-join rooms and re-sync state in the `connect` handler, since `socket.id` and room membership reset on reconnect.
- **Throttle/debounce high-frequency client events** (typing indicators, cursor moves) and use `volatile` emits for disposable high-rate data.
- **Set sensible Engine.IO options** (`pingInterval`, `pingTimeout`, `maxHttpBufferSize`) to bound memory and detect dead connections.

## Key Interview Points

- **Why not just HTTP for real-time?** HTTP is client-initiated; the server cannot push. Real-time needs a persistent, bidirectional channel — that is WebSockets.
- **Polling vs long-polling vs SSE vs WebSockets.** Short polling = periodic requests (wasteful, latency-bound). Long polling = server holds the request until data (low latency, still many requests). SSE = one-way server→client stream over HTTP with auto-reconnect. WebSockets = full-duplex persistent connection. Know each one's direction and trade-offs.
- **How does a WebSocket connection start?** As an HTTP `GET` with `Upgrade: websocket`; the server replies `101 Switching Protocols`; then the TCP connection speaks the WebSocket framing protocol. Schemes: `ws://` / `wss://`.
- **Is Socket.IO the same as WebSocket?** No. Socket.IO is its own protocol that *uses* WebSocket (and long-polling) as transports. Clients/servers must both be Socket.IO. It adds reconnection, fallback, rooms, namespaces, acks, and multiplexing.
- **What does Socket.IO add over `ws`?** Automatic reconnection, transport fallback, named events, acknowledgements, rooms, namespaces, broadcasting helpers, binary/JSON encoding, heartbeats, and the multi-server adapter.
- **`io.emit` vs `socket.emit` vs `socket.broadcast.emit`?** `socket.emit` → sender only; `io.emit` → everyone incl. sender; `socket.broadcast.emit` → everyone except sender. `io.to(room)` includes a sender in the room; `socket.to(room)` excludes the sender.
- **Rooms vs namespaces?** Namespaces = developer-defined channels with their own handlers/middleware (`/admin`, `/chat`), multiplexed over one connection. Rooms = dynamic, runtime groupings of sockets *within* a namespace. Few namespaces, many rooms.
- **What is an acknowledgement?** A callback passed as the last `emit` argument that the receiver invokes to "respond," giving request/response semantics. Pair with `.timeout()` / `emitWithAck()`; acks are not guaranteed delivery — make handlers idempotent.
- **How do you authenticate a socket?** Handshake middleware (`io.use`) reading `socket.handshake.auth.token`, verifying a JWT, and storing identity on `socket.data`. Handle token expiry via refresh events or server-side revocation.
- **How do you scale Socket.IO across processes?** The **Redis adapter** publishes broadcasts via Redis pub/sub so every process forwards to its local clients — making `io.emit`/rooms work cluster-wide — **plus sticky sessions** so each client's long-polling handshake stays on one process.
- **Why do you need sticky sessions?** Engine.IO's long-polling handshake uses multiple HTTP requests that must hit the same process; round-robining them causes `Session ID unknown`. (Avoidable only by forcing WebSocket-only transport.)
- **How do you implement reliable presence?** Track by user identity with a `userId -> Set<socketId>` reference count (multiple tabs/devices), emit online on first connection and offline on last; store in Redis for multi-process accuracy.
- **Why is `socket.id` unsafe as identity?** It is per-connection and changes on every reconnect.

## Practice Exercises

Work these in order; each builds on the last. Try before reading the solutions.

1. **Echo + timestamp (raw `ws`).** Using the `ws` library, build a server that, for each message a client sends, replies with `{ original, length, at }` as JSON. Add a heartbeat that terminates clients that miss a ping.
2. **Socket.IO broadcast counter.** A Socket.IO server tracks a shared integer. Clients emit `counter:increment`; the server increments and broadcasts `counter:update` with the new value to **all** clients (including the sender). On connect, send the newcomer the current value.
3. **Rooms-based chat.** Let clients `room:join` a named room and `room:message` to it. Messages should reach everyone in the room *except* the sender (the sender renders their own message optimistically). Notify a room when someone joins or leaves.
4. **Acknowledged "create task" with validation and timeout.** Client emits `task:create` with a payload and an ack. The server validates (`title` required, ≤ 100 chars), saves to an in-memory array, and acks `{ ok, id }` or `{ ok: false, error }`. The client uses `emitWithAck` with a 3-second timeout and logs the result.
5. **JWT auth + presence.** Add handshake JWT auth (you may use `jsonwebtoken`). On connect, attach `userId`. Maintain an accurate online list across multiple tabs (reference counting). Broadcast `presence:online`/`presence:offline` only on the first/last connection for a user.
6. **Make it multi-process (stretch).** Take exercise 5 and add the Redis adapter so broadcasts and presence work across two clustered Node processes. Describe what additionally needs to be true at the load-balancer layer.

### Solutions

**Solution 1 — Echo + timestamp with heartbeat.**

```js
// solution1.js
const { WebSocketServer } = require('ws');
const wss = new WebSocketServer({ port: 8080 });

function heartbeat() { this.isAlive = true; }

wss.on('connection', (socket) => {
  socket.isAlive = true;
  socket.on('pong', heartbeat);

  socket.on('message', (data, isBinary) => {
    const original = isBinary ? '<binary>' : data.toString();
    socket.send(JSON.stringify({ original, length: original.length, at: Date.now() }));
  });
});

const interval = setInterval(() => {
  for (const s of wss.clients) {
    if (s.isAlive === false) { s.terminate(); continue; }
    s.isAlive = false;
    s.ping();
  }
}, 30000);
wss.on('close', () => clearInterval(interval));
```

**Solution 2 — Broadcast counter.**

```js
// solution2.js
const { Server } = require('socket.io');
const io = new Server(3000);

let count = 0;

io.on('connection', (socket) => {
  socket.emit('counter:update', count); // send current value to the newcomer

  socket.on('counter:increment', () => {
    count += 1;
    io.emit('counter:update', count); // io.emit => everyone, including the sender
  });
});
```

**Solution 3 — Rooms-based chat.**

```js
// solution3.js
const { Server } = require('socket.io');
const io = new Server(3000);

io.on('connection', (socket) => {
  socket.on('room:join', (room) => {
    socket.join(room);
    socket.to(room).emit('system', `${socket.id} joined`); // others in the room only
  });

  socket.on('room:message', ({ room, text }) => {
    // socket.to(room) excludes the sender; their UI shows the message optimistically.
    socket.to(room).emit('room:message', { from: socket.id, text });
  });

  socket.on('disconnecting', () => {
    // socket.rooms still populated here — notify each room the user was in.
    for (const room of socket.rooms) {
      if (room !== socket.id) socket.to(room).emit('system', `${socket.id} left`);
    }
  });
});
```

**Solution 4 — Acknowledged create with validation + timeout.**

```js
// solution4-server.js
const { Server } = require('socket.io');
const io = new Server(3000);
const tasks = [];
let nextId = 1;

io.on('connection', (socket) => {
  socket.on('task:create', (payload, ack) => {
    if (!payload?.title || typeof payload.title !== 'string') {
      return ack?.({ ok: false, error: 'title is required' });
    }
    if (payload.title.length > 100) {
      return ack?.({ ok: false, error: 'title too long' });
    }
    const task = { id: nextId++, title: payload.title, done: false };
    tasks.push(task);
    ack?.({ ok: true, id: task.id });
  });
});
```

```js
// solution4-client.js
const { io } = require('socket.io-client');
const socket = io('http://localhost:3000');

(async () => {
  try {
    const res = await socket.timeout(3000).emitWithAck('task:create', { title: 'Learn sockets' });
    console.log(res.ok ? `created #${res.id}` : `rejected: ${res.error}`);
  } catch {
    console.error('no ack within 3s');
  }
})();
```

**Solution 5 — JWT auth + accurate presence.**

```js
// solution5.js
const http = require('node:http');
const express = require('express');
const jwt = require('jsonwebtoken');
const { Server } = require('socket.io');

const app = express();
const server = http.createServer(app);
const io = new Server(server);

const SECRET = process.env.JWT_SECRET || 'dev-secret';
const online = new Map(); // userId -> Set<socketId>

io.use((socket, next) => {
  try {
    const payload = jwt.verify(socket.handshake.auth?.token, SECRET);
    socket.data.userId = payload.sub;
    next();
  } catch {
    next(new Error('UNAUTHENTICATED'));
  }
});

io.on('connection', (socket) => {
  const { userId } = socket.data;
  if (!online.has(userId)) online.set(userId, new Set());
  const first = online.get(userId).size === 0;
  online.get(userId).add(socket.id);
  if (first) io.emit('presence:online', userId);
  socket.emit('presence:list', [...online.keys()]);

  socket.on('disconnect', () => {
    const set = online.get(userId);
    set.delete(socket.id);
    if (set.size === 0) {
      online.delete(userId);
      io.emit('presence:offline', userId);
    }
  });
});

server.listen(3000);
```

**Solution 6 — Multi-process (stretch).**

```js
// solution6.js — add the Redis adapter to Solution 5's server.
const { createClient } = require('redis');
const { createAdapter } = require('@socket.io/redis-adapter');
// ... build `io` exactly as in Solution 5, then:

const pub = createClient({ url: process.env.REDIS_URL || 'redis://localhost:6379' });
const sub = pub.duplicate();
await Promise.all([pub.connect(), sub.connect()]);
io.adapter(createAdapter(pub, sub)); // broadcasts/rooms now span all processes
```

What else must be true at the load-balancer layer:

- **Sticky sessions (session affinity)** so all of a client's long-polling handshake requests reach the same process; otherwise `Session ID unknown` errors. (Or force `transports: ['websocket']` to use a single connection and skip stickiness, sacrificing fallback.)
- The proxy/LB must **forward the `Upgrade`/`Connection` headers** and allow long-lived connections (raise idle timeouts) so WebSocket upgrades succeed.
- For **presence specifically**, the in-memory `online` map is still per-process; move it to **Redis** (e.g. a set per `userId` with TTL) so the roster is globally accurate, then broadcast presence changes (which the Redis adapter fans out to every process).

## Summary / Cheat Sheet

| Topic | Key takeaway |
|---|---|
| **Why real-time** | HTTP is client-initiated; the server can't push. Need a persistent, bidirectional channel. |
| **Polling** | Periodic requests. Simple but wasteful and latency-bound. |
| **Long polling** | Server holds the request until data. Low latency, many requests; Socket.IO's fallback. |
| **SSE** | One-way server→client stream over HTTP, auto-reconnect (`EventSource`). Great for feeds. |
| **WebSocket** | Full-duplex persistent TCP after an HTTP `Upgrade` → `101`. `ws://`/`wss://`. |
| **`ws` library** | Minimal raw WebSockets. You build reconnection, rooms, heartbeats yourself. |
| **Socket.IO** | Own protocol over WS+polling. Adds reconnect, fallback, events, acks, rooms, namespaces, scaling. Not interchangeable with native WS. |
| **Server setup** | `new Server(httpServer, { cors })`; attach to `http.Server`, not the Express `app`. |
| **Emit targeting** | `socket.emit`=sender; `io.emit`=all; `socket.broadcast.emit`=all but sender; `io.to(room)`/`socket.to(room)`=room (incl./excl. sender). |
| **Rooms** | Dynamic server-side groups; auto-join `socket.id` room; auto-leave on disconnect. |
| **Namespaces** | Static channels (`/admin`) with own middleware/handlers. Few namespaces, many rooms. |
| **Acks** | Callback as last `emit` arg; use `.timeout()` / `emitWithAck()`; not guaranteed → be idempotent. |
| **Lifecycle** | `connection`, `disconnecting` (rooms still set), `disconnect`. |
| **Presence** | Track by `userId -> Set<socketId>`; emit online/offline on first/last connection. |
| **Auth** | `io.use()` handshake middleware verifies JWT from `socket.handshake.auth.token` → `socket.data`. |
| **Scaling** | Redis adapter (broadcasts cross processes) **+** sticky sessions (handshake affinity). |
| **Identity** | `socket.id` is per-connection; never use it as a stable user id. |
