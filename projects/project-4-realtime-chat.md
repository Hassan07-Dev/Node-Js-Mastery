# Project 4: Real-Time Chat Application (Socket.IO)

Up to this point in the course every project has lived inside the HTTP request/response model: the client asks, the server answers, the connection closes. That model is wonderful for REST APIs, but it has a hard ceiling — **the server can never speak first.** A chat application breaks that assumption completely. When Alice types a message, your server must *push* it to Bob, Carol, and everyone else in the room *right now*, without anyone polling. That is the world of **persistent, bidirectional, low-latency connections**, and you are about to build a production-shaped one.

This guide assumes you have already read **Module 28 (Real-Time Apps with WebSockets & Socket.IO)** and **Module 21 (Authentication: Sessions & JWT)**. We will not re-derive *why* WebSockets exist or *what* a JWT is from zero — instead we apply both, end to end, to a real product. By the end you will have a runnable chat server with rooms, presence, typing indicators, persisted history, and JWT-secured sockets, plus a single-file browser client you can open in two tabs to watch messages fly.

> TIP: Type every file. The whole point of a project guide is muscle memory. Reading `socket.join(room)` teaches recognition; typing it (and then debugging why your message showed up twice) teaches recall — which is what an interviewer is actually testing.

---

## Overview

We are building **a multi-room real-time chat service**, structured the way a real backend team would structure it (config, models, services, controllers, REST routes, socket handlers, centralized error handling), not as one giant `server.js`.

Concretely, the finished application does all of the following:

- **Authenticates over HTTP first.** A user registers and logs in via plain REST endpoints (`POST /api/auth/register`, `POST /api/auth/login`) and receives a **JWT**. This reuses everything you learned in Project 3.
- **Authenticates the socket handshake.** The browser opens a Socket.IO connection carrying that JWT. A handshake middleware (`io.use(...)`) verifies it *before* the `connection` event fires, so every message is reliably attributed to a real user — clients cannot spoof their identity.
- **Supports multiple chat rooms.** Users join and leave named rooms (`general`, `random`, `nodejs`, …). Messages broadcast only to the room they were sent to.
- **Shows an online-users / presence list per room.** When you join a room you see who else is there; everyone is notified as people come and go. Presence is correct even when one user has multiple tabs open.
- **Shows typing indicators.** "Alice is typing…" appears for others in the room and clears automatically.
- **Persists every message to MongoDB** with Mongoose, and **loads recent history** (paginated) the moment you join a room, so a refresh does not lose the conversation.
- **Uses acknowledgements** so the client knows its message was actually saved (request/response semantics *over* a socket), and emits validation errors back to just the sender.
- **Is scaling-aware.** We explain exactly what breaks the instant you run a second Node process, and wire in the **Redis adapter** so broadcasts and rooms span every process.

You will run it, open two browser tabs as two different users, and watch real-time chat work. Then the Extensions section pushes you toward private DMs, read receipts, and deeper pagination — the kind of features that turn "I built a chat demo" into "I built a chat *system*" in an interview.

---

## What You'll Practice

This project is the capstone of the real-time material and pulls in a surprising amount of the whole course:

- **Attaching Socket.IO to an existing Express + Node `http` server** (and *why* you attach it to the bare `http.Server`, not to the Express `app`).
- **`io.use()` handshake middleware** to authenticate connections with a JWT pulled from `socket.handshake.auth`.
- **The full emit-targeting cheat sheet** in anger: `socket.emit` (sender), `io.emit` (everyone), `socket.broadcast.emit` (everyone-but-sender), `io.to(room)` (room incl. sender), `socket.to(room)` (room excl. sender), and `io.to(userId)` for direct messages.
- **Rooms** for room membership, broadcasting, and leaving; understanding that **every socket auto-joins a room named after its own `socket.id`**.
- **Rooms vs namespaces** — when each is the right tool (few namespaces, many rooms).
- **Acknowledgements (callbacks)** with `.timeout()` and the promise-based `socket.emitWithAck()`, and why acks must be **idempotent**.
- **Presence done right** — tracking `userId → Set<socketId>` so multi-tab users count once, and emitting online/offline on the *first* and *last* connection only.
- **The `connection` / `disconnecting` / `disconnect` lifecycle** — and the subtle reason `disconnecting` (not `disconnect`) is where you still know which rooms a socket was in.
- **Persistence and pagination** with Mongoose: indexing messages by room + time, loading the most-recent N, and cursoring backward through history.
- **Centralized error handling** for both the REST side (Express error middleware) and the socket side (a wrapper that catches handler errors and acks them back).
- **Environment-based config** and a clean, layered folder structure.
- **Horizontal scaling** with the `@socket.io/redis-adapter`, sticky sessions, and moving presence into Redis.

---

## Prerequisites & Setup

You need:

- **Node.js current LTS** (v20+ or v22+). Check with `node --version`.
- **MongoDB** — either a local `mongod` or a free **MongoDB Atlas** cluster. Have a connection string ready.
- **(Optional for the scaling section) Redis** — local `redis-server` or a managed instance. The core app runs fine without it.
- A browser. No build tooling, framework, or bundler is required for the client — it is a single static HTML file.

### Create the project and install dependencies

```bash
# Create and enter the project directory
mkdir realtime-chat && cd realtime-chat

# Initialize package.json
npm init -y

# Runtime dependencies
npm install express socket.io mongoose jsonwebtoken bcryptjs dotenv cors

# Dev dependency: auto-restart on file changes
npm install --save-dev nodemon

# OPTIONAL — only needed for the multi-process scaling section (Step 11)
npm install @socket.io/redis-adapter redis
```

What each package is for:

| Package | Role |
|---|---|
| `express` | HTTP framework for the REST endpoints (auth) and serving the static client. |
| `socket.io` | The real-time engine — server side of the WebSocket/Engine.IO protocol. |
| `mongoose` | MongoDB ODM for the `User` and `Message` models. |
| `jsonwebtoken` | Sign/verify the JWTs used for both REST and the socket handshake. |
| `bcryptjs` | Hash passwords. (Pure-JS, no native build step — friendlier than `bcrypt` on some machines.) |
| `dotenv` | Load `.env` into `process.env`. |
| `cors` | Configure cross-origin access for the REST API (Socket.IO has its *own* CORS option). |
| `nodemon` (dev) | Restart the server when files change. |
| `@socket.io/redis-adapter`, `redis` (optional) | Fan out broadcasts/rooms across multiple Node processes. |

> WARNING — Pitfall: Socket.IO's CORS configuration is **separate** from the Express `cors` middleware. The `cors` package handles your `/api/...` routes; the Socket.IO handshake is configured via the `cors` option passed to `new Server(...)`. Configuring one does not configure the other — you usually need both.

### Configure npm scripts

Open `package.json` and set the `scripts` block:

```json
{
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js"
  }
}
```

> TIP: This project uses **CommonJS** (`require`) to stay consistent with the rest of the course. If you prefer ES Modules, add `"type": "module"` to `package.json` and convert `require`/`module.exports` to `import`/`export` — every snippet here maps one-to-one. The Socket.IO and Mongoose APIs are identical under both.

### Folder structure

Create this layout. It separates concerns the way a real team would — HTTP and real-time live side by side but each has its own home.

```
realtime-chat/
├── .env                      # secrets & config (never commit this)
├── .env.example              # template documenting required vars (DO commit)
├── .gitignore
├── package.json
├── public/                   # static client served by Express
│   └── index.html            # the entire browser test client (one file)
└── src/
    ├── server.js             # entry point: builds http server, attaches io, starts listening
    ├── app.js                # builds & configures the Express app (REST + static)
    ├── config/
    │   ├── env.js            # reads/validates environment variables
    │   └── db.js             # Mongoose connection helper
    ├── models/
    │   ├── User.js           # user schema (auth)
    │   └── Message.js        # chat message schema (persistence)
    ├── services/
    │   ├── auth.service.js    # register/login business logic
    │   └── message.service.js # save/load/paginate messages
    ├── controllers/
    │   └── auth.controller.js # HTTP handlers for auth
    ├── routes/
    │   └── auth.routes.js     # /api/auth/* router
    ├── middleware/
    │   ├── auth.js            # JWT helpers: sign + an Express "protect" guard
    │   └── error.js           # centralized Express error handler + 404
    ├── sockets/
    │   ├── index.js           # initializes io, registers handshake auth + handlers
    │   ├── auth.socket.js     # io.use() handshake middleware (verifies JWT)
    │   ├── presence.js        # in-memory presence registry (userId -> Set<socketId>)
    │   └── chat.handlers.js   # all chat events: join/leave/message/typing
    └── utils/
        ├── AppError.js        # operational error class (status + message)
        └── socketWrap.js      # wraps async socket handlers to catch & ack errors
```

Create the empty tree now so the files have somewhere to live:

```bash
mkdir -p public src/config src/models src/services src/controllers \
  src/routes src/middleware src/sockets src/utils
```

---

## Step 1 — Environment config & secrets

Configuration that differs between machines or must stay secret (DB URI, JWT secret, port) belongs in the environment, never in source. We read it once, validate it, and export a frozen object so the rest of the app imports clean values instead of poking at `process.env` everywhere.

**`.env`** — your real, local values. This file is git-ignored.

```bash
# .env
NODE_ENV=development
PORT=3000

# MongoDB connection string (local example shown; swap for your Atlas URI)
MONGODB_URI=mongodb://127.0.0.1:27017/realtime_chat

# Secret used to sign & verify JWTs. Use a long random string in production.
JWT_SECRET=change-me-to-a-long-random-string-min-32-chars
JWT_EXPIRES_IN=7d

# Where the browser client is served from / allowed to connect from.
# In dev the client is same-origin, so this can be the server's own URL.
CLIENT_ORIGIN=http://localhost:3000

# OPTIONAL — only used in the scaling section (Step 11)
REDIS_URL=redis://127.0.0.1:6379
```

**`.env.example`** — committed so teammates know what to set (no real secrets):

```bash
# .env.example
NODE_ENV=development
PORT=3000
MONGODB_URI=mongodb://127.0.0.1:27017/realtime_chat
JWT_SECRET=
JWT_EXPIRES_IN=7d
CLIENT_ORIGIN=http://localhost:3000
REDIS_URL=redis://127.0.0.1:6379
```

**`.gitignore`**:

```bash
# .gitignore
node_modules/
.env
npm-debug.log*
```

**`src/config/env.js`** — load, validate, and freeze config:

```js
// src/config/env.js
// Load environment variables from .env into process.env (must happen first).
require('dotenv').config();

// Helper: fail fast if a required variable is missing. A misconfigured server
// that crashes at boot is far better than one that "works" then leaks/forges tokens.
function required(name) {
  const value = process.env[name];
  if (!value) {
    throw new Error(`Missing required environment variable: ${name}`);
  }
  return value;
}

const env = {
  NODE_ENV: process.env.NODE_ENV || 'development',
  PORT: Number(process.env.PORT) || 3000,
  MONGODB_URI: required('MONGODB_URI'),
  JWT_SECRET: required('JWT_SECRET'),
  JWT_EXPIRES_IN: process.env.JWT_EXPIRES_IN || '7d',
  CLIENT_ORIGIN: process.env.CLIENT_ORIGIN || 'http://localhost:3000',
  REDIS_URL: process.env.REDIS_URL || 'redis://127.0.0.1:6379',
};

// Object.freeze prevents accidental mutation of config at runtime.
module.exports = Object.freeze(env);
```

> BEST PRACTICE: Validate config at startup and **crash on missing secrets**. Forgetting `JWT_SECRET` and silently signing tokens with `undefined` is a real, exploitable bug. Failing loudly at boot is the cheapest possible time to catch it.

---

## Step 2 — Database connection

A tiny, reusable Mongoose connector. We keep connection concerns out of `server.js` so the entry point stays readable.

**`src/config/db.js`**:

```js
// src/config/db.js
const mongoose = require('mongoose');
const env = require('./env');

async function connectDB() {
  // Mongoose buffers commands until connected, but we await so a bad URI
  // fails the boot rather than hanging the first query silently.
  await mongoose.connect(env.MONGODB_URI);
  console.log(`[db] connected: ${mongoose.connection.host}/${mongoose.connection.name}`);

  // Surface connection drops after the initial connect (network blips, Atlas failover).
  mongoose.connection.on('error', (err) => {
    console.error('[db] connection error:', err.message);
  });
  mongoose.connection.on('disconnected', () => {
    console.warn('[db] disconnected');
  });

  return mongoose.connection;
}

module.exports = connectDB;
```

---

## Step 3 — Models (User & Message)

### `src/models/User.js`

Standard auth user (consistent with Project 3). Passwords are hashed in a `pre('save')` hook; `password` is excluded from queries by default via `select: false`.

```js
// src/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema(
  {
    username: {
      type: String,
      required: [true, 'username is required'],
      unique: true,
      trim: true,
      minlength: [3, 'username must be at least 3 characters'],
      maxlength: [30, 'username must be at most 30 characters'],
      // Allow letters, numbers, underscore, dash — keeps display names sane.
      match: [/^[a-zA-Z0-9_-]+$/, 'username may only contain letters, numbers, _ and -'],
    },
    password: {
      type: String,
      required: [true, 'password is required'],
      minlength: [6, 'password must be at least 6 characters'],
      select: false, // never return the hash unless explicitly requested
    },
  },
  { timestamps: true } // adds createdAt / updatedAt
);

// Hash the password whenever it is set/changed (not on every save).
userSchema.pre('save', async function hashPassword(next) {
  if (!this.isModified('password')) return next();
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
  next();
});

// Instance method to verify a plaintext password against the stored hash.
userSchema.methods.comparePassword = function comparePassword(candidate) {
  return bcrypt.compare(candidate, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### `src/models/Message.js`

The heart of persistence. Each message records *who* sent it, *which room* it belongs to, the text, and the timestamps. We store a denormalized `username` snapshot alongside the `sender` reference so rendering history never needs a join/`populate` — a common and pragmatic read-optimization for chat.

```js
// src/models/Message.js
const mongoose = require('mongoose');

const messageSchema = new mongoose.Schema(
  {
    room: {
      type: String,
      required: true,
      index: true, // we query by room constantly
    },
    sender: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User',
      required: true,
    },
    // Denormalized snapshot of the sender's name at send time. Reading history
    // does not require populate(), and the name stays correct even if the user
    // later renames. (Trade-off: a rename won't retroactively update old messages.)
    username: {
      type: String,
      required: true,
    },
    text: {
      type: String,
      required: [true, 'message text is required'],
      trim: true,
      maxlength: [2000, 'message too long (max 2000 chars)'],
    },
  },
  { timestamps: true } // createdAt is our chronological ordering key
);

// Compound index: fetch a room's messages newest-first efficiently, and
// paginate backward by createdAt. This is THE index that makes history fast.
messageSchema.index({ room: 1, createdAt: -1 });

module.exports = mongoose.model('Message', messageSchema);
```

> BEST PRACTICE: Index every field you filter or sort on at scale. A chat app's hottest query is "give me the last N messages in room X, newest first." The compound `{ room: 1, createdAt: -1 }` index serves both the filter (`room`) and the sort (`createdAt`) from a single index — without it, MongoDB scans the whole collection on every room join.

---

## Step 4 — JWT helpers & the Express auth guard

These helpers are shared by the REST routes *and* the socket handshake — one signing function, one verifying function, used everywhere identity matters.

**`src/middleware/auth.js`**:

```js
// src/middleware/auth.js
const jwt = require('jsonwebtoken');
const env = require('../config/env');
const AppError = require('../utils/AppError');

// Create a signed token carrying the minimal identity claims we need.
function signToken(user) {
  // Keep the payload small: only what every request needs to identify the user.
  return jwt.sign(
    { sub: user._id.toString(), username: user.username },
    env.JWT_SECRET,
    { expiresIn: env.JWT_EXPIRES_IN }
  );
}

// Verify a token string and return its decoded payload. Throws on invalid/expired.
function verifyToken(token) {
  return jwt.verify(token, env.JWT_SECRET);
}

// Express middleware: require a valid "Authorization: Bearer <token>" header.
// Used to protect HTTP routes (the socket side has its own handshake guard).
function protect(req, res, next) {
  const header = req.headers.authorization || '';
  const [scheme, token] = header.split(' ');

  if (scheme !== 'Bearer' || !token) {
    return next(new AppError('Authentication required', 401));
  }

  try {
    const payload = verifyToken(token);
    // Attach a small, trusted identity object for downstream handlers.
    req.user = { id: payload.sub, username: payload.username };
    next();
  } catch (err) {
    return next(new AppError('Invalid or expired token', 401));
  }
}

module.exports = { signToken, verifyToken, protect };
```

---

## Step 5 — Error plumbing (AppError + Express handler)

Two small but important pieces: a typed error class so we can distinguish *operational* errors (bad input, 401) from *programmer* errors (a thrown `TypeError`), and a centralized Express error handler so route code never sends ad-hoc error responses.

**`src/utils/AppError.js`**:

```js
// src/utils/AppError.js
// An operational error we deliberately throw and expect (vs. an unexpected crash).
class AppError extends Error {
  constructor(message, statusCode = 500) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = true; // flag so the handler knows this was anticipated
    Error.captureStackTrace(this, this.constructor);
  }
}

module.exports = AppError;
```

**`src/middleware/error.js`**:

```js
// src/middleware/error.js
const env = require('../config/env');
const AppError = require('../utils/AppError');

// 404 handler — reached when no route matched. Forwards to the error handler.
function notFound(req, res, next) {
  next(new AppError(`Not found: ${req.method} ${req.originalUrl}`, 404));
}

// Centralized Express error handler. MUST have 4 args for Express to recognize it.
function errorHandler(err, req, res, next) {
  // Translate common Mongoose errors into clean 4xx responses.
  if (err.name === 'ValidationError') {
    const messages = Object.values(err.errors).map((e) => e.message);
    err = new AppError(messages.join('; '), 400);
  }
  if (err.code === 11000) {
    // Duplicate key (e.g. username already taken)
    const field = Object.keys(err.keyValue || {})[0] || 'field';
    err = new AppError(`${field} already exists`, 409);
  }

  const statusCode = err.statusCode || 500;
  const isOperational = err.isOperational === true;

  // Log unexpected (non-operational) errors with the full stack for debugging.
  if (!isOperational) {
    console.error('[unhandled error]', err);
  }

  res.status(statusCode).json({
    status: 'error',
    message: isOperational ? err.message : 'Something went wrong',
    // Only leak stack traces in development.
    ...(env.NODE_ENV === 'development' && !isOperational ? { stack: err.stack } : {}),
  });
}

module.exports = { notFound, errorHandler };
```

> TIP: An Express error handler is identified **only** by its arity — it must declare exactly four parameters `(err, req, res, next)`. Drop the unused `next` and Express treats it as a normal middleware and your errors fall through to the default HTML error page.

---

## Step 6 — Auth service, controller, routes

Thin controllers, fat-ish services. The service does the work; the controller adapts HTTP to it.

**`src/services/auth.service.js`**:

```js
// src/services/auth.service.js
const User = require('../models/User');
const AppError = require('../utils/AppError');
const { signToken } = require('../middleware/auth');

async function register({ username, password }) {
  if (!username || !password) {
    throw new AppError('username and password are required', 400);
  }
  // The model's pre-save hook hashes the password; unique index guards duplicates.
  const user = await User.create({ username, password });
  const token = signToken(user);
  return { user: { id: user._id, username: user.username }, token };
}

async function login({ username, password }) {
  if (!username || !password) {
    throw new AppError('username and password are required', 400);
  }
  // password is select:false, so explicitly request it for comparison.
  const user = await User.findOne({ username }).select('+password');
  if (!user || !(await user.comparePassword(password))) {
    // Same message for both cases — don't reveal whether the username exists.
    throw new AppError('Invalid credentials', 401);
  }
  const token = signToken(user);
  return { user: { id: user._id, username: user.username }, token };
}

module.exports = { register, login };
```

**`src/controllers/auth.controller.js`**:

```js
// src/controllers/auth.controller.js
const authService = require('../services/auth.service');

// Small wrapper so we don't repeat try/catch in every async controller.
const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);

const register = asyncHandler(async (req, res) => {
  const result = await authService.register(req.body);
  res.status(201).json({ status: 'success', data: result });
});

const login = asyncHandler(async (req, res) => {
  const result = await authService.login(req.body);
  res.status(200).json({ status: 'success', data: result });
});

module.exports = { register, login };
```

**`src/routes/auth.routes.js`**:

```js
// src/routes/auth.routes.js
const express = require('express');
const { register, login } = require('../controllers/auth.controller');

const router = express.Router();

router.post('/register', register); // POST /api/auth/register
router.post('/login', login);       // POST /api/auth/login

module.exports = router;
```

---

## Step 7 — Message service (persistence & pagination)

This is where chat history lives. Two operations: **save a message** and **load a page of recent messages**. We design pagination correctly from the start using a **cursor** (a `before` timestamp), because offset pagination (`skip`) breaks when new messages arrive mid-scroll.

**`src/services/message.service.js`**:

```js
// src/services/message.service.js
const Message = require('../models/Message');
const AppError = require('../utils/AppError');

const MAX_PAGE = 100;
const DEFAULT_PAGE = 30;

// Persist a single chat message. Returns a plain, client-safe object.
async function saveMessage({ room, senderId, username, text }) {
  const trimmed = (text || '').trim();
  if (!trimmed) throw new AppError('Message text is required', 400);
  if (trimmed.length > 2000) throw new AppError('Message too long (max 2000)', 400);
  if (!room) throw new AppError('Room is required', 400);

  const doc = await Message.create({
    room,
    sender: senderId,
    username,
    text: trimmed,
  });

  return serialize(doc);
}

// Load a page of a room's history, newest-first, then reversed to chronological
// order for display. `before` is a cursor: pass the oldest createdAt you have to
// fetch the page before it (i.e. scroll-back / "load older").
async function getMessages({ room, limit = DEFAULT_PAGE, before } = {}) {
  if (!room) throw new AppError('Room is required', 400);

  const pageSize = Math.min(Number(limit) || DEFAULT_PAGE, MAX_PAGE);

  const query = { room };
  if (before) {
    const cursor = new Date(before);
    if (!Number.isNaN(cursor.getTime())) {
      query.createdAt = { $lt: cursor }; // strictly older than the cursor
    }
  }

  // Fetch newest-first (uses the { room, createdAt:-1 } index), capped at pageSize+1
  // so we can tell the client whether MORE older messages exist.
  const docs = await Message.find(query)
    .sort({ createdAt: -1 })
    .limit(pageSize + 1)
    .lean(); // plain JS objects, faster — we don't need Mongoose documents here

  const hasMore = docs.length > pageSize;
  const page = hasMore ? docs.slice(0, pageSize) : docs;

  // Reverse to chronological (oldest → newest) for natural top-to-bottom display.
  const messages = page.reverse().map(serializeLean);

  // The cursor to fetch the NEXT older page is the oldest createdAt in this page.
  const nextCursor = messages.length ? messages[0].createdAt : null;

  return { messages, hasMore, nextCursor };
}

// Normalize a Mongoose document for the wire.
function serialize(doc) {
  return {
    id: doc._id.toString(),
    room: doc.room,
    senderId: doc.sender.toString(),
    username: doc.username,
    text: doc.text,
    createdAt: doc.createdAt.toISOString(),
  };
}

// Same, but for a .lean() plain object.
function serializeLean(doc) {
  return {
    id: doc._id.toString(),
    room: doc.room,
    senderId: doc.sender.toString(),
    username: doc.username,
    text: doc.text,
    createdAt: new Date(doc.createdAt).toISOString(),
  };
}

module.exports = { saveMessage, getMessages };
```

> WARNING — Pitfall: **Do not paginate chat with `.skip(offset)`.** While a user scrolls back, new messages keep arriving at the top, which shifts every offset and causes duplicated or skipped messages. **Cursor pagination** (`createdAt < before`) is stable: it anchors to a fixed point in time, immune to new inserts. This is a frequent system-design interview point.

---

## Step 8 — The Express app (REST + static client)

Now assemble the HTTP layer: JSON body parsing, CORS for the API, the auth router, the static client, the 404, and the centralized error handler. Note this module **exports the app but does not start a server** — `server.js` owns the lifecycle so it can also attach Socket.IO.

**`src/app.js`**:

```js
// src/app.js
const path = require('node:path');
const express = require('express');
const cors = require('cors');

const env = require('./config/env');
const authRoutes = require('./routes/auth.routes');
const { notFound, errorHandler } = require('./middleware/error');

function createApp() {
  const app = express();

  // Parse JSON bodies for the REST API.
  app.use(express.json());

  // CORS for the REST API. (Socket.IO has its OWN cors option — see sockets/index.js.)
  app.use(cors({ origin: env.CLIENT_ORIGIN, credentials: true }));

  // Health check — handy for load balancers and quick "is it up?" curls.
  app.get('/health', (req, res) => res.json({ status: 'ok', uptime: process.uptime() }));

  // REST routes.
  app.use('/api/auth', authRoutes);

  // Serve the static browser client from /public at the site root.
  app.use(express.static(path.join(__dirname, '..', 'public')));

  // 404 for anything unmatched, then the centralized error handler LAST.
  app.use(notFound);
  app.use(errorHandler);

  return app;
}

module.exports = createApp;
```

---

## Step 9 — The real-time layer (the core of the project)

This is the part you came for. We split it into four files so each concern is testable and readable:

1. `presence.js` — an in-memory registry that answers "who is online, and in which rooms?"
2. `auth.socket.js` — handshake middleware that verifies the JWT before `connection`.
3. `chat.handlers.js` — the actual events: join, leave, message, typing.
4. `index.js` — wires it all into a configured `Server` instance.

### 9.1 Presence registry

Presence is deceptively tricky. The naive approach ("add `socket.id` to a list on connect, remove on disconnect") breaks the moment a user opens a second tab: closing one tab marks them offline even though they are still here. The fix is to track **users**, not **sockets** — a map of `userId → Set<socketId>`. A user is online while that set is non-empty; they go offline only when the *last* socket disconnects.

**`src/sockets/presence.js`**:

```js
// src/sockets/presence.js
// In-memory presence registry. (Single-process only — Step 11 moves this to Redis.)

// userId -> { username, sockets: Set<socketId> }
const users = new Map();

// room -> Set<userId>   (which users are currently in each room)
const rooms = new Map();

// Record that a socket connected for a user. Returns true if this is the user's
// FIRST connection (i.e. they just came online), so callers can broadcast once.
function addUser(userId, username, socketId) {
  let entry = users.get(userId);
  const isFirst = !entry;
  if (!entry) {
    entry = { username, sockets: new Set() };
    users.set(userId, entry);
  }
  entry.sockets.add(socketId);
  return isFirst;
}

// Record that a socket disconnected. Returns true if it was the user's LAST
// socket (i.e. they just went fully offline).
function removeSocket(userId, socketId) {
  const entry = users.get(userId);
  if (!entry) return false;
  entry.sockets.delete(socketId);
  if (entry.sockets.size === 0) {
    users.delete(userId);
    // Also drop them from every room's membership set.
    for (const members of rooms.values()) members.delete(userId);
    return true;
  }
  return false;
}

function joinRoom(room, userId) {
  if (!rooms.has(room)) rooms.set(room, new Set());
  rooms.get(room).add(userId);
}

function leaveRoom(room, userId) {
  const members = rooms.get(room);
  if (members) {
    members.delete(userId);
    if (members.size === 0) rooms.delete(room);
  }
}

// The roster for a room: [{ userId, username }, ...]
function getRoomUsers(room) {
  const members = rooms.get(room);
  if (!members) return [];
  return [...members].map((userId) => ({
    userId,
    username: users.get(userId)?.username || 'unknown',
  }));
}

module.exports = { addUser, removeSocket, joinRoom, leaveRoom, getRoomUsers };
```

> WARNING — Pitfall: **`socket.id` is per-connection, not per-user.** It changes on every reconnect and there is one per tab. Never use it as a stable user identity, and never key presence on it directly. Key on the `userId` from the verified JWT; track the socket ids underneath.

### 9.2 Handshake authentication middleware

`io.use(...)` runs **once per connection, before the `connection` event fires**. If we call `next(err)`, the connection is refused and the client receives a `connect_error`. This is exactly where we verify the JWT and stash the identity on `socket.data`, so every later handler can trust `socket.data.user` without re-checking.

**`src/sockets/auth.socket.js`**:

```js
// src/sockets/auth.socket.js
const { verifyToken } = require('../middleware/auth');

// Socket.IO handshake middleware: authenticate BEFORE the connection is accepted.
function socketAuth(socket, next) {
  // The browser sends the token in the handshake `auth` object:
  //   io(url, { auth: { token } })
  // Fall back to the Authorization header for non-browser clients if you like.
  const token =
    socket.handshake.auth?.token ||
    (socket.handshake.headers?.authorization || '').replace(/^Bearer\s+/i, '');

  if (!token) {
    // Calling next() with an Error rejects the handshake -> client 'connect_error'.
    return next(new Error('Authentication required'));
  }

  try {
    const payload = verifyToken(token);
    // Stash a trusted identity on socket.data — available in every handler.
    socket.data.user = { id: payload.sub, username: payload.username };
    next();
  } catch (err) {
    next(new Error('Invalid or expired token'));
  }
}

module.exports = socketAuth;
```

> BEST PRACTICE: Put auth in **handshake middleware (`io.use`)**, not inside individual event handlers. Rejecting at the handshake means an unauthenticated client never even reaches your `connection` logic — no half-connected zombies, no per-event token checks scattered everywhere. Authenticate once, trust `socket.data.user` thereafter.

### 9.3 The socket error wrapper

Socket handlers are async and touch the database, so they can throw. An unhandled rejection in a handler will crash the process (Node ≥ 15). We wrap each handler so errors are caught, logged, and acked back to the sender instead.

**`src/utils/socketWrap.js`**:

```js
// src/utils/socketWrap.js
const AppError = require('./AppError');

// Wrap an async socket handler so thrown errors are caught and reported to the
// sender via the acknowledgement callback (if the client provided one).
function withAck(handler) {
  return async (...args) => {
    // If the client passed an ack callback, it's the LAST argument.
    const maybeAck = args[args.length - 1];
    const ack = typeof maybeAck === 'function' ? maybeAck : null;

    try {
      await handler(...args);
    } catch (err) {
      const isOperational = err instanceof AppError;
      if (!isOperational) console.error('[socket handler error]', err);
      const message = isOperational ? err.message : 'Internal server error';
      // Report the failure back to the caller in a consistent shape.
      if (ack) ack({ ok: false, error: message });
    }
  };
}

module.exports = { withAck };
```

> WARNING — Pitfall: An unhandled promise rejection inside a socket event handler does **not** automatically get reported to the client and, on modern Node, can terminate the whole server. Always wrap async socket handlers (or attach a `connection`-level error strategy). The ack channel is the natural place to surface "your action failed."

### 9.4 The chat event handlers

Now the events themselves. We use **namespaced event names** (`chat:message`, `room:join`) for clarity in logs and to avoid colliding with Socket.IO's reserved events. Each room is a Socket.IO **room**; the socket's own `userId` room (auto-created) is left as-is for future DMs.

**`src/sockets/chat.handlers.js`**:

```js
// src/sockets/chat.handlers.js
const messageService = require('../services/message.service');
const presence = require('./presence');
const { withAck } = require('../utils/socketWrap');
const AppError = require('../utils/AppError');

// Basic room-name validation: keep it to a safe, predictable charset.
function assertValidRoom(room) {
  if (typeof room !== 'string' || !/^[a-zA-Z0-9_-]{1,50}$/.test(room)) {
    throw new AppError('Invalid room name', 400);
  }
}

// Register every chat event for a single connected socket.
function registerChatHandlers(io, socket) {
  const user = socket.data.user; // set by the handshake auth middleware

  // --- JOIN A ROOM -------------------------------------------------------
  // Client: socket.emit('room:join', { room }, ack)
  socket.on(
    'room:join',
    withAck(async ({ room } = {}, ack) => {
      assertValidRoom(room);

      // A socket may be in many rooms; here we model "switch to one room" by
      // leaving any non-personal rooms first (optional — adjust to taste).
      for (const r of socket.rooms) {
        if (r !== socket.id && r !== room) {
          socket.leave(r);
          if (presence.leaveRoom(r, user.id)) { /* room emptied */ }
          io.to(r).emit('room:users', { room: r, users: presence.getRoomUsers(r) });
          socket.to(r).emit('room:left', { room: r, user });
        }
      }

      socket.join(room);            // add this socket to the Socket.IO room
      presence.joinRoom(room, user.id);

      // Load recent history for this room (persisted in MongoDB).
      const history = await messageService.getMessages({ room, limit: 30 });

      // Tell everyone in the room (including us) the updated roster.
      io.to(room).emit('room:users', { room, users: presence.getRoomUsers(room) });
      // Tell OTHERS in the room that we joined (exclude sender via socket.to).
      socket.to(room).emit('room:joined', { room, user });

      // Acknowledge to the joiner with history + current roster.
      if (ack) {
        ack({
          ok: true,
          room,
          messages: history.messages,
          hasMore: history.hasMore,
          nextCursor: history.nextCursor,
          users: presence.getRoomUsers(room),
        });
      }
    })
  );

  // --- LEAVE A ROOM ------------------------------------------------------
  socket.on(
    'room:leave',
    withAck(async ({ room } = {}, ack) => {
      assertValidRoom(room);
      socket.leave(room);
      presence.leaveRoom(room, user.id);
      io.to(room).emit('room:users', { room, users: presence.getRoomUsers(room) });
      socket.to(room).emit('room:left', { room, user });
      if (ack) ack({ ok: true, room });
    })
  );

  // --- SEND A MESSAGE ----------------------------------------------------
  // Client: socket.emit('chat:message', { room, text }, ack)
  socket.on(
    'chat:message',
    withAck(async ({ room, text } = {}, ack) => {
      assertValidRoom(room);

      // Guard: only members of the room may post to it.
      if (!socket.rooms.has(room)) {
        throw new AppError('Join the room before sending messages', 403);
      }

      // Persist first, THEN broadcast — so everyone (incl. sender) sees the
      // same authoritative record (with a real id and server timestamp).
      const saved = await messageService.saveMessage({
        room,
        senderId: user.id,
        username: user.username,
        text,
      });

      // Broadcast to everyone in the room INCLUDING the sender, so all clients
      // render the identical saved message. (We do NOT optimistically render
      // on the client; the ack + this event are the source of truth.)
      io.to(room).emit('chat:message', saved);

      // Acknowledge delivery to the sender (request/response over a socket).
      if (ack) ack({ ok: true, message: saved });
    })
  );

  // --- LOAD OLDER HISTORY (pagination) ----------------------------------
  // Client: socket.emit('chat:history', { room, before }, ack)
  socket.on(
    'chat:history',
    withAck(async ({ room, before } = {}, ack) => {
      assertValidRoom(room);
      const page = await messageService.getMessages({ room, before, limit: 30 });
      if (ack) ack({ ok: true, ...page });
    })
  );

  // --- TYPING INDICATORS -------------------------------------------------
  // Fire-and-forget (no ack): low value, high frequency.
  socket.on('typing:start', ({ room } = {}) => {
    if (typeof room !== 'string' || !socket.rooms.has(room)) return;
    // Tell OTHERS in the room (exclude sender).
    socket.to(room).emit('typing:start', { room, user });
  });

  socket.on('typing:stop', ({ room } = {}) => {
    if (typeof room !== 'string' || !socket.rooms.has(room)) return;
    socket.to(room).emit('typing:stop', { room, user });
  });
}

module.exports = registerChatHandlers;
```

> WARNING — Pitfall: `io.to(room).emit(...)` includes the sender; `socket.to(room).emit(...)` excludes the sender. For the actual chat message we use `io.to(room)` deliberately, so the sender's screen renders the *server's* saved record (with its real id and timestamp) rather than a guessed optimistic copy. For "X joined" / "X is typing" we use `socket.to(room)` — you should never tell yourself you joined or that you're typing.

### 9.5 Wiring it together: the Socket.IO server

`index.js` creates the `Server`, registers the handshake auth, and on every `connection` registers the chat handlers and manages presence + the lifecycle (`connection` → `disconnecting` → `disconnect`).

**`src/sockets/index.js`**:

```js
// src/sockets/index.js
const { Server } = require('socket.io');
const env = require('../config/env');
const socketAuth = require('./auth.socket');
const registerChatHandlers = require('./chat.handlers');
const presence = require('./presence');

function initSocketServer(httpServer) {
  const io = new Server(httpServer, {
    // Socket.IO's OWN CORS config (separate from Express cors).
    cors: {
      origin: env.CLIENT_ORIGIN,
      credentials: true,
    },
    // Tune connection-state recovery so brief network blips don't lose state.
    connectionStateRecovery: {
      maxDisconnectionDuration: 2 * 60 * 1000, // 2 minutes
      skipMiddlewares: true,
    },
  });

  // Handshake authentication — runs once per connection, before 'connection'.
  io.use(socketAuth);

  io.on('connection', (socket) => {
    const user = socket.data.user;
    console.log(`[socket] connected: ${user.username} (${socket.id})`);

    // Mark this user online; broadcast a global presence event only on FIRST socket.
    const cameOnline = presence.addUser(user.id, user.username, socket.id);
    if (cameOnline) {
      io.emit('presence:online', { userId: user.id, username: user.username });
    }

    // Register all chat events for this connection.
    registerChatHandlers(io, socket);

    // 'disconnecting' fires BEFORE the socket leaves its rooms — so socket.rooms
    // is still populated here. This is the right place to notify each room.
    socket.on('disconnecting', () => {
      for (const room of socket.rooms) {
        if (room === socket.id) continue; // skip the personal room
        presence.leaveRoom(room, user.id);
        // Notify others in the room; roster will be recomputed after leave.
        socket.to(room).emit('room:left', { room, user });
        // Recompute roster excluding this user.
        socket.to(room).emit('room:users', {
          room,
          users: presence.getRoomUsers(room).filter((u) => u.userId !== user.id),
        });
      }
    });

    // 'disconnect' fires after the socket has left all rooms.
    socket.on('disconnect', (reason) => {
      const wentOffline = presence.removeSocket(user.id, socket.id);
      if (wentOffline) {
        io.emit('presence:offline', { userId: user.id, username: user.username });
      }
      console.log(`[socket] disconnected: ${user.username} (${socket.id}) — ${reason}`);
    });
  });

  return io;
}

module.exports = initSocketServer;
```

> WARNING — Pitfall: The difference between **`disconnecting`** and **`disconnect`** is exam-worthy. During `disconnecting`, `socket.rooms` still contains every room the socket was in — that is your only chance to know which rooms to notify. By the time `disconnect` fires, the socket has already left all rooms and `socket.rooms` is empty (just the personal room is gone too). Do room-aware cleanup in `disconnecting`; do global cleanup in `disconnect`.

---

## Step 10 — The entry point: `server.js`

Finally, the lifecycle owner. It connects to Mongo, builds the Express app, creates a Node `http.Server` from it, attaches Socket.IO to that **bare http server**, and starts listening. It also handles graceful shutdown.

**`src/server.js`**:

```js
// src/server.js
const http = require('node:http');

const env = require('./config/env');
const connectDB = require('./config/db');
const createApp = require('./app');
const initSocketServer = require('./sockets');

async function start() {
  // 1. Connect to MongoDB first — no point starting if the DB is unreachable.
  await connectDB();

  // 2. Build the Express app (REST + static client).
  const app = createApp();

  // 3. Create a raw Node HTTP server FROM the Express app.
  //    Socket.IO must attach to the http.Server, not to `app` directly —
  //    it needs the underlying server to handle the WebSocket 'upgrade' event.
  const httpServer = http.createServer(app);

  // 4. Attach Socket.IO to the same http server (shares the port).
  const io = initSocketServer(httpServer);

  // 5. Start listening.
  httpServer.listen(env.PORT, () => {
    console.log(`[http] listening on http://localhost:${env.PORT} (${env.NODE_ENV})`);
    console.log(`[ws]   socket.io ready`);
  });

  // --- Graceful shutdown ---------------------------------------------------
  const shutdown = (signal) => {
    console.log(`\n[shutdown] received ${signal}, closing...`);
    io.close(() => console.log('[shutdown] socket.io closed'));
    httpServer.close(() => {
      console.log('[shutdown] http server closed');
      process.exit(0);
    });
    // Force-exit if connections refuse to close in time.
    setTimeout(() => process.exit(1), 10000).unref();
  };
  process.on('SIGINT', () => shutdown('SIGINT'));
  process.on('SIGTERM', () => shutdown('SIGTERM'));
}

start().catch((err) => {
  console.error('[fatal] failed to start server:', err);
  process.exit(1);
});
```

> WARNING — Pitfall: You **cannot** pass the Express `app` to `new Server(app)` and expect WebSockets to work. Express's `app` is a request handler, not a server; the WebSocket `upgrade` happens at the `http.Server` level. Always do `http.createServer(app)` and attach Socket.IO to *that*. Getting this wrong produces a server where HTTP works but sockets silently fail to upgrade.

---

## Step 11 — Scaling across processes with the Redis adapter

Everything above works perfectly — **on a single process.** The instant you run two Node processes (PM2 cluster mode, multiple containers, multiple machines behind a load balancer), it breaks in two specific ways:

1. **Broadcasts don't cross processes.** `io.to('general').emit(...)` only reaches sockets connected to *this* process. A user on process A never sees a message broadcast from process B. By default Socket.IO uses an in-memory adapter that knows only about local sockets.
2. **Your presence map is per-process.** The in-memory `Map` in `presence.js` only knows who is connected to *this* process, so the roster is wrong.

### Fix #1: the Redis adapter (cross-process broadcasting)

The `@socket.io/redis-adapter` uses Redis pub/sub so that when one process broadcasts to a room, *every* process publishes to its own local room members. Rooms and broadcasts then behave as if there were a single process.

Update `src/sockets/index.js` to optionally attach the adapter:

```js
// src/sockets/index.js (additions — the rest stays the same)
const { createClient } = require('redis');
const { createAdapter } = require('@socket.io/redis-adapter');

// ...inside initSocketServer(httpServer), after `const io = new Server(...)`:
async function attachRedisAdapter(io) {
  const pubClient = createClient({ url: env.REDIS_URL });
  const subClient = pubClient.duplicate(); // sub must be a separate connection
  await Promise.all([pubClient.connect(), subClient.connect()]);
  io.adapter(createAdapter(pubClient, subClient));
  console.log('[ws] redis adapter attached — broadcasts now span all processes');
}
```

Then call it from `start()` in `server.js` when Redis is configured:

```js
// src/server.js — call after initSocketServer, before listen(), if using Redis:
//   const io = initSocketServer(httpServer);
//   if (process.env.USE_REDIS === 'true') await attachRedisAdapter(io);
// (export attachRedisAdapter from sockets/index.js, or return both from init)
```

### Fix #2: presence in Redis

Move the presence registry out of process memory into Redis so the roster is globally accurate. A clean model:

- A Redis **set** per room: `room:<name>:users` holding `userId`s.
- A Redis **hash** or per-user **set** of socket ids with a short TTL/heartbeat so crashed processes don't leave ghosts.
- On the *first* socket for a user → broadcast online; on the *last* → broadcast offline (use Redis atomic ops like `SADD`/`SREM` and check cardinality with `SCARD`).

Because the Redis adapter fans the resulting `room:users` / `presence:*` emits out to every process, all clients stay in sync.

### What the load balancer must also do

The adapter is necessary but not sufficient. At the infrastructure layer:

| Requirement | Why |
|---|---|
| **Sticky sessions (session affinity)** | Socket.IO's HTTP long-polling handshake spans multiple requests that must all hit the *same* process, or you get `Session ID unknown` errors. (Alternatively force `transports: ['websocket']` to skip polling and avoid stickiness — at the cost of the polling fallback.) |
| **Forward `Upgrade`/`Connection` headers** | The reverse proxy (nginx, ALB) must pass these so the WebSocket upgrade succeeds. |
| **Raise idle timeouts** | WebSockets are long-lived; default proxy idle timeouts will kill them. |

> BEST PRACTICE: Add the Redis adapter *before* you ever run a second process — not after you've shipped and users are reporting "I don't see other people's messages." The single-process app and the multi-process app differ by exactly this one wiring change, so build it knowing the upgrade path.

---

## Step 12 — The browser test client

A single static file, served from `/public`, with no build step. It lets you log in (or register), connect a JWT-authenticated socket, switch rooms, see the roster, see typing indicators, send messages, and load older history. Open it in **two browser tabs as two different users** to watch real-time chat.

**`public/index.html`**:

```html
<!-- public/index.html -->
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>Realtime Chat</title>
    <!-- Socket.IO client is served automatically by the server at this path. -->
    <script src="/socket.io/socket.io.js"></script>
    <style>
      body { font-family: system-ui, sans-serif; margin: 0; display: flex; height: 100vh; }
      #sidebar { width: 220px; border-right: 1px solid #ddd; padding: 12px; overflow:auto; }
      #main { flex: 1; display: flex; flex-direction: column; }
      #messages { flex: 1; overflow: auto; padding: 12px; }
      #composer { display: flex; padding: 12px; border-top: 1px solid #ddd; gap: 8px; }
      #composer input { flex: 1; padding: 8px; }
      .msg { margin: 4px 0; }
      .msg .who { font-weight: 600; }
      .meta { color: #888; font-size: 12px; }
      #typing { height: 18px; color: #888; font-size: 13px; padding: 0 12px; }
      .auth { padding: 12px; }
      button { padding: 8px 10px; cursor: pointer; }
      ul { list-style: none; padding-left: 0; }
    </style>
  </head>
  <body>
    <!-- LOGIN PANEL (shown until authenticated) -->
    <div id="login" class="auth">
      <h3>Login / Register</h3>
      <input id="username" placeholder="username" /><br /><br />
      <input id="password" type="password" placeholder="password" /><br /><br />
      <button onclick="auth('login')">Login</button>
      <button onclick="auth('register')">Register</button>
      <p id="authError" style="color:red"></p>
    </div>

    <!-- CHAT UI (hidden until authenticated) -->
    <div id="sidebar" style="display:none">
      <h3 id="me"></h3>
      <h4>Rooms</h4>
      <ul id="roomList"></ul>
      <input id="newRoom" placeholder="new room" />
      <button onclick="joinRoom(document.getElementById('newRoom').value)">Join</button>
      <h4>In this room</h4>
      <ul id="userList"></ul>
    </div>
    <div id="main" style="display:none">
      <div id="messages"></div>
      <button id="loadOlder" onclick="loadOlder()" style="display:none">Load older</button>
      <div id="typing"></div>
      <div id="composer">
        <input id="text" placeholder="Type a message…" autocomplete="off" />
        <button onclick="send()">Send</button>
      </div>
    </div>

    <script>
      const ROOMS = ['general', 'random', 'nodejs'];
      let socket = null;
      let token = null;
      let me = null;
      let currentRoom = null;
      let nextCursor = null;
      let typingTimeout = null;

      // --- AUTH over REST -------------------------------------------------
      async function auth(kind) {
        const username = document.getElementById('username').value.trim();
        const password = document.getElementById('password').value;
        const res = await fetch('/api/auth/' + kind, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ username, password }),
        });
        const body = await res.json();
        if (!res.ok) {
          document.getElementById('authError').textContent = body.message || 'Auth failed';
          return;
        }
        token = body.data.token;
        me = body.data.user;
        startChat();
      }

      // --- CONNECT the authenticated socket -------------------------------
      function startChat() {
        document.getElementById('login').style.display = 'none';
        document.getElementById('sidebar').style.display = 'block';
        document.getElementById('main').style.display = 'flex';
        document.getElementById('me').textContent = '👤 ' + me.username;

        // Pass the JWT in the handshake `auth` object — the server reads
        // socket.handshake.auth.token in its io.use() middleware.
        socket = io({ auth: { token } });

        socket.on('connect_error', (err) => {
          alert('Socket connection failed: ' + err.message);
        });

        // Incoming chat message (server-authoritative copy).
        socket.on('chat:message', (msg) => {
          if (msg.room === currentRoom) renderMessage(msg);
        });

        // Roster updates for a room.
        socket.on('room:users', ({ room, users }) => {
          if (room === currentRoom) renderUsers(users);
        });

        socket.on('room:joined', ({ user }) => systemLine(`${user.username} joined`));
        socket.on('room:left', ({ user }) => systemLine(`${user.username} left`));

        // Typing indicators.
        socket.on('typing:start', ({ room, user }) => {
          if (room === currentRoom) showTyping(user.username);
        });
        socket.on('typing:stop', ({ room }) => {
          if (room === currentRoom) document.getElementById('typing').textContent = '';
        });

        renderRoomList();
        joinRoom('general'); // auto-join a default room
      }

      // --- ROOMS ----------------------------------------------------------
      function renderRoomList() {
        const ul = document.getElementById('roomList');
        ul.innerHTML = '';
        ROOMS.forEach((r) => {
          const li = document.createElement('li');
          li.innerHTML = `<button onclick="joinRoom('${r}')">#${r}</button>`;
          ul.appendChild(li);
        });
      }

      function joinRoom(room) {
        room = (room || '').trim();
        if (!room) return;
        // emitWithAck-style: pass a callback as the last arg to get the ack.
        socket.emit('room:join', { room }, (ack) => {
          if (!ack.ok) return alert('Join failed: ' + ack.error);
          currentRoom = ack.room;
          nextCursor = ack.nextCursor;
          document.getElementById('messages').innerHTML = '';
          ack.messages.forEach(renderMessage); // render history
          renderUsers(ack.users);
          document.getElementById('loadOlder').style.display = ack.hasMore ? 'block' : 'none';
          systemLine(`— joined #${room} —`);
        });
      }

      // --- PAGINATION: load older messages --------------------------------
      function loadOlder() {
        if (!nextCursor) return;
        socket.emit('chat:history', { room: currentRoom, before: nextCursor }, (ack) => {
          if (!ack.ok) return;
          const box = document.getElementById('messages');
          // Prepend older messages (in chronological order) above current ones.
          ack.messages.reverse().forEach((msg) => prependMessage(msg));
          nextCursor = ack.nextCursor;
          document.getElementById('loadOlder').style.display = ack.hasMore ? 'block' : 'none';
        });
      }

      // --- SEND a message (with ack) --------------------------------------
      function send() {
        const input = document.getElementById('text');
        const text = input.value.trim();
        if (!text || !currentRoom) return;
        socket.emit('chat:message', { room: currentRoom, text }, (ack) => {
          if (!ack.ok) alert('Send failed: ' + ack.error);
        });
        input.value = '';
        socket.emit('typing:stop', { room: currentRoom });
      }

      // --- TYPING ---------------------------------------------------------
      document.addEventListener('input', (e) => {
        if (e.target.id !== 'text' || !currentRoom) return;
        socket.emit('typing:start', { room: currentRoom });
        clearTimeout(typingTimeout);
        // Auto-stop after 1.5s of inactivity.
        typingTimeout = setTimeout(() => {
          socket.emit('typing:stop', { room: currentRoom });
        }, 1500);
      });

      // Enter to send.
      document.addEventListener('keydown', (e) => {
        if (e.target.id === 'text' && e.key === 'Enter') send();
      });

      // --- RENDER helpers -------------------------------------------------
      function renderMessage(msg) {
        const box = document.getElementById('messages');
        const div = messageNode(msg);
        box.appendChild(div);
        box.scrollTop = box.scrollHeight;
      }
      function prependMessage(msg) {
        const box = document.getElementById('messages');
        box.insertBefore(messageNode(msg), box.firstChild);
      }
      function messageNode(msg) {
        const div = document.createElement('div');
        div.className = 'msg';
        const time = new Date(msg.createdAt).toLocaleTimeString();
        // textContent (not innerHTML) to avoid XSS from message text.
        const who = document.createElement('span');
        who.className = 'who';
        who.textContent = msg.username + ': ';
        const txt = document.createElement('span');
        txt.textContent = msg.text;
        const meta = document.createElement('span');
        meta.className = 'meta';
        meta.textContent = '  ' + time;
        div.append(who, txt, meta);
        return div;
      }
      function systemLine(text) {
        const box = document.getElementById('messages');
        const div = document.createElement('div');
        div.className = 'meta';
        div.textContent = text;
        box.appendChild(div);
        box.scrollTop = box.scrollHeight;
      }
      function renderUsers(users) {
        const ul = document.getElementById('userList');
        ul.innerHTML = '';
        users.forEach((u) => {
          const li = document.createElement('li');
          li.textContent = '🟢 ' + u.username;
          ul.appendChild(li);
        });
      }
      function showTyping(name) {
        document.getElementById('typing').textContent = name + ' is typing…';
      }
    </script>
  </body>
</html>
```

> TIP: Notice the client renders message text with `textContent`, never `innerHTML`. Chat is user-generated content; rendering it as HTML is a classic stored-XSS hole. Render as text on the client, and consider sanitizing/escaping on the server too if you ever store rich content.

> TIP: The `/socket.io/socket.io.js` script is served automatically by the Socket.IO server — you don't add it to `public/` yourself. The client version it serves always matches your installed server version, which prevents protocol-mismatch headaches.

---

## How to Run & Test

### 1. Start MongoDB

```bash
# Local install:
mongod --dbpath /your/data/path
# …or just use an Atlas connection string in MONGODB_URI and skip this.
```

### 2. Start the server

```bash
# Development (auto-restart on changes):
npm run dev

# Production-style:
npm start
```

Expected console output:

```text
[db] connected: 127.0.0.1/realtime_chat
[http] listening on http://localhost:3000 (development)
[ws]   socket.io ready
```

### 3. Test the REST auth with curl

Register a user:

```bash
curl -s -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"secret123"}'
```

Sample response:

```json
{
  "status": "success",
  "data": {
    "user": { "id": "665f...", "username": "alice" },
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6..."
  }
}
```

Log in (same shape, status 200):

```bash
curl -s -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"secret123"}'
```

Wrong password returns a clean 401:

```json
{ "status": "error", "message": "Invalid credentials" }
```

Health check:

```bash
curl -s http://localhost:3000/health
# {"status":"ok","uptime":12.34}
```

### 4. Test real-time chat in the browser

1. Open **http://localhost:3000** in a browser tab. Register as `alice`.
2. Open a **second tab** (or a private window). Register as `bob`.
3. In both tabs, you'll auto-join `#general`. Each tab's sidebar shows the roster — you should see both `alice` and `bob` listed.
4. Type in Alice's tab. Bob's tab shows "**alice is typing…**".
5. Send the message. It appears in **both** tabs instantly (server-authoritative copy with timestamp).
6. Switch Bob to `#random` (click the room button). Alice's `#general` roster updates to drop Bob; messages Alice sends now only reach others in `#general`.
7. Refresh either tab and re-login: the **history** for the current room loads automatically (proving persistence), and the "Load older" button pages further back.

### 5. Inspect persisted messages

```bash
# In the mongo shell:
mongosh realtime_chat --eval 'db.messages.find().sort({createdAt:-1}).limit(5).pretty()'
```

You'll see documents with `room`, `sender`, `username`, `text`, and `createdAt`.

### 6. (Optional) Test a programmatic socket client with Node

Useful for understanding the handshake and acks without the browser:

```js
// test-client.js — run with: node test-client.js
const { io } = require('socket.io-client'); // npm i -D socket.io-client

// Paste a token you got from /api/auth/login:
const token = 'PASTE_JWT_HERE';

const socket = io('http://localhost:3000', { auth: { token } });

socket.on('connect', () => {
  console.log('connected:', socket.id);
  // Join a room and print the ack (history + roster).
  socket.emit('room:join', { room: 'general' }, (ack) => {
    console.log('join ack:', ack);
    // Send a message with an ack.
    socket.emit('chat:message', { room: 'general', text: 'hello from node' }, (a) => {
      console.log('send ack:', a);
    });
  });
});

socket.on('chat:message', (m) => console.log('message:', m.username, m.text));
socket.on('connect_error', (e) => console.error('connect_error:', e.message));
```

> WARNING — Pitfall: A `connect_error` with message `Authentication required` or `Invalid or expired token` means the handshake middleware rejected you. Double-check you're passing `{ auth: { token } }` (not an `Authorization` header in the browser) and that the token isn't expired. This is *working as designed* — it proves your auth guard is doing its job.

---

## Code Walkthrough / Key Concepts Recap

A guided tour of the decisions that matter, mapped to what an interviewer will probe.

### Why attach Socket.IO to `http.Server`, not `app`

Express's `app` is a *request listener function*, not a server. The WebSocket handshake is an HTTP request with `Upgrade: websocket`, handled at the `http.Server`'s `upgrade` event — a level Express doesn't expose. So we do `http.createServer(app)` and give that server to both `listen()` and `new Server(httpServer)`. They share one port; HTTP requests go to Express, upgrade requests go to Socket.IO.

### Handshake auth vs per-event auth

We authenticate **once** in `io.use()`, before `connection`. The verified identity lives on `socket.data.user`, trusted by every handler. The alternative — checking a token inside every event handler — is repetitive, error-prone, and still lets unauthenticated sockets connect and consume resources. Handshake auth rejects them at the door.

### The emit-targeting matrix (memorize this)

| Goal | Call | Includes sender? |
|---|---|---|
| Reply to just the sender | `socket.emit(event, data)` | n/a (only the sender) |
| Everyone connected | `io.emit(event, data)` | Yes |
| Everyone except the sender | `socket.broadcast.emit(event, data)` | No |
| A room, including the sender | `io.to(room).emit(event, data)` | Yes (if in room) |
| A room, excluding the sender | `socket.to(room).emit(event, data)` | No |
| One specific user (DM) | `io.to(userId).emit(event, data)` | depends (uses the auto-room) |

We use `io.to(room)` for the chat message (sender should see the saved copy) and `socket.to(room)` for "joined/left/typing" (you shouldn't notify yourself).

### Rooms vs namespaces

This is a top interview question. They are *not* the same axis:

| | **Namespace** | **Room** |
|---|---|---|
| What it is | A separate communication channel (path like `/admin`, `/chat`) multiplexed over one connection | A dynamic, server-side group of sockets within a namespace |
| Created | Defined in code (`io.of('/admin')`) | On the fly (`socket.join('room42')`) |
| Has its own | Middleware, event handlers, connection logic | Nothing of its own — just a label for broadcasting |
| Client sees it | Client explicitly connects to the namespace | Invisible to the client; purely a server grouping |
| Typical count | A handful (logical app modules) | Many, dynamic (one per chat room, game, document) |
| Use for | Splitting an app into modules with separate access control | Grouping sockets for targeted broadcast |

Rule of thumb: **few namespaces, many rooms.** Our chat rooms are *rooms* (dynamic, many, just broadcast groups). If we added an admin dashboard with its own auth and events, that would be a *namespace*.

### Acknowledgements

An ack gives you request/response semantics over the otherwise fire-and-forget socket. The client passes a callback as the **last** argument of `emit`; the server invokes it with a result. We use acks to confirm a message was *saved* (not just sent) and to deliver validation errors back to only the sender. Two cautions:

- **Acks are not guaranteed** — if the connection drops before the server calls back, the callback never fires. Use `socket.timeout(ms).emit(...)` or `await socket.emitWithAck(...)` with a timeout, and design handlers to be **idempotent** so a client retry doesn't double-insert.
- The **last function argument is *always* interpreted as an ack callback**. Never pass a function as real payload.

### Presence done right

We key presence on **`userId`**, not `socket.id`, tracking a `Set` of socket ids per user. A user is online while that set is non-empty. We broadcast `presence:online` only on the *first* socket and `presence:offline` only on the *last* — so a multi-tab user counts once and doesn't flicker offline when closing one tab.

### The lifecycle: `connection` → `disconnecting` → `disconnect`

- `connection`: socket authenticated and ready; register handlers, mark online.
- `disconnecting`: fires *before* the socket leaves its rooms — `socket.rooms` is still populated, so this is where we notify each room and update rosters.
- `disconnect`: fires after rooms are left; do global cleanup (decrement presence, maybe broadcast offline).

### Persistence + cursor pagination

We persist every message *before* broadcasting so all clients render the same authoritative record. History loads on join (most-recent 30), and "load older" pages backward with a **cursor** (`createdAt < before`), which is stable under concurrent inserts — unlike `skip/offset`. The compound index `{ room: 1, createdAt: -1 }` makes both the join-load and the page-back queries index-only.

### Scaling

A single process needs nothing extra. Two+ processes need the **Redis adapter** (so broadcasts/rooms cross processes) *and* **sticky sessions** at the load balancer (so the polling handshake stays on one process), *and* presence moved into Redis (so the roster is global). Build the adapter wiring early — the single→multi upgrade is exactly that one change.

---

## Extensions & Challenges

Each of these turns the demo into a more complete system and gives you concrete interview talking points.

### 1. Private direct messages (DMs)

Every socket auto-joins a room named after its own `socket.id`, and we already key presence by `userId`. To DM, target the *recipient's user room*:

- On connect, `socket.join('user:' + user.id)` so each user has a stable personal room across tabs.
- Add a `dm:send` event: `socket.emit('dm:send', { toUserId, text }, ack)`.
- Server: persist with a `Message` variant (add an optional `recipient` field and a `type: 'dm'`), then `io.to('user:' + toUserId).to('user:' + senderId).emit('dm:message', saved)` so both participants' tabs receive it.
- Decide your DM "room key" (e.g. sorted pair of user ids) so DM history is queryable.

### 2. Read receipts

- Add `readBy: [ObjectId]` (or a separate `Receipt` collection for scale) to `Message`.
- Client emits `chat:read` with the latest message id it has displayed when the window is focused.
- Server records the read and emits `chat:read` to the room/sender so senders see "Seen by Bob".
- For groups, prefer a separate `receipts` collection keyed by `{ messageId, userId }` to avoid unbounded array growth on hot messages.

### 3. Deeper message pagination & infinite scroll

- We already return `hasMore` and `nextCursor`. Wire the client's scroll-to-top to call `chat:history` automatically (intersection observer on a sentinel element at the top).
- Preserve scroll position when prepending older messages (record `scrollHeight` before, restore the delta after).

### 4. Other worthwhile additions

- **Message editing/deletion** with `editedAt`/`deletedAt` and a broadcast so all clients update in place.
- **Rate limiting** per socket (e.g. token bucket on `chat:message`) to prevent spam/flooding.
- **Server-side input sanitization** (escape/strip HTML) in addition to client `textContent`.
- **Reconnection UX**: listen for `reconnect`, re-join the current room, and re-fetch missed messages by cursor.
- **Unread counts** per room using presence + last-read timestamps.
- **File/image attachments** (upload via REST → store URL → send a message referencing it).
- **Tests**: spin up the server and a `socket.io-client`, assert join/message/ack/presence flows.

---

## Common Errors & Fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| Client gets `connect_error: Authentication required` | No token sent, or sent wrong | Connect with `io(url, { auth: { token } })`. The browser handshake reads `socket.handshake.auth.token`, not the `Authorization` header. |
| `connect_error: Invalid or expired token` | Token expired / signed with a different secret | Re-login to get a fresh token. Ensure `JWT_SECRET` matches the one used to sign. |
| Sockets don't connect, console shows **CORS** errors | Socket.IO CORS not configured (it's separate from Express `cors`) | Set the `cors` option in `new Server(httpServer, { cors: { origin } })`. Point the client at the full URL if cross-origin. |
| Messages work, but you **see your own message twice** | Optimistic local render *plus* the `io.to(room)` broadcast | Don't optimistically render; rely on the broadcast/ack. Or render optimistically and use `socket.broadcast`/dedupe by message id. |
| You **never see other people's** messages | Used `socket.to(room)` (excludes sender) where you meant everyone, or vice-versa; or not joined to the room | Use `io.to(room)` to include the sender for the saved message; confirm both clients `room:join` the same room. |
| WebSocket fails to upgrade; HTTP works fine | Passed `app` to `new Server(app)`, or proxy strips `Upgrade` headers | Attach to `http.createServer(app)`; configure nginx/ALB to forward `Upgrade`/`Connection` and allow long-lived connections. |
| Roster shows a user **still online after they closed every tab** | Presence keyed on `socket.id`, or cleanup not in `disconnecting` | Key presence on `userId` with a `Set<socketId>`; do room cleanup in `disconnecting` (where `socket.rooms` is still populated). |
| Closing **one of several tabs** marks the user offline | Removing the user on any disconnect | Only broadcast offline when the *last* socket for that user disconnects (Set becomes empty). |
| `MongooseError`/`MongoServerError` on save: `E11000 duplicate key` | Re-registering an existing username | Caught by the error handler → 409. For messages, ensure you're not setting a duplicate `_id`. |
| In a 2-process setup, users on different processes **don't see each other** | No Redis adapter; in-memory adapter is process-local | Attach `@socket.io/redis-adapter` and run Redis; add sticky sessions at the LB. |
| `Session ID unknown` errors behind a load balancer | Long-polling handshake hitting different processes | Enable **sticky sessions**, or force `transports: ['websocket']` on the client to skip polling. |
| Server crashes on a bad message | Unhandled rejection in an async socket handler | Wrap handlers (our `withAck`) so errors are caught and acked instead of crashing the process. |
| History doesn't load on join | Forgot to `await` the service, or wrong room name | Ensure `room:join` returns history in its ack and the client renders `ack.messages`. Validate room names match. |
| Typing indicator **never clears** | No `typing:stop` on send/blur/timeout | Emit `typing:stop` on send and after an inactivity timeout; clear the indicator on receiving it. |

> BEST PRACTICE: When real-time "just doesn't work," debug in this order: (1) does the socket even **connect** (watch `connect`/`connect_error`)? (2) is the client **in the room** it thinks it is (log `socket.rooms` server-side)? (3) are you using the **right emit target** (`io.to` vs `socket.to`)? Ninety percent of chat bugs are one of these three.

---

You now have a complete, layered, production-shaped real-time chat application: REST auth, JWT-secured socket handshake, multi-room broadcasting, accurate multi-tab presence, typing indicators, persisted+paginated history, acknowledgements, centralized error handling, and a clear path to horizontal scale. Build it, open two tabs, break it on purpose, fix it — then be ready to explain every decision above in an interview.
