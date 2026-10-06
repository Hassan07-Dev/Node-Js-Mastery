# Project 5 (Capstone): E-Commerce REST API

This is the project that ties the entire course together. Everything you have learned across five weeks — Node core, the event loop, Express routing and middleware, Mongoose modelling, JWT authentication and role-based authorization, validation, centralized error handling, security hardening, file uploads, logging, testing, API documentation, and deployment — converges into a single, coherent, production-shaped backend.

This is also your **portfolio centerpiece** and your single best interview talking point. When an interviewer says "tell me about a backend project you built," this is the one you describe: not because it is the largest, but because it forces you to make and defend real engineering decisions at every layer — how you structure code, how you protect a route, how you keep a controller thin, how you turn a cart into an order atomically, how you ship it in a container. Build it once, end to end, and you will be able to *build it, explain it, and defend it.*

> Note: This guide targets **Node.js current LTS** and **Express 5** (the default `express` on npm since 2024). Code is CommonJS (`require`) by default — the most common style in production codebases and interviews — with notes on ES Modules where it matters. The database is **MongoDB + Mongoose**. We call out **Express v4 vs v5 differences** wherever they affect what you type.

---

## Overview

We are building the backend for an online store — a **JSON REST API** with no frontend. A web or mobile client (or Postman) talks to it over HTTP. Concretely, the API exposes these capabilities:

- **Users & Auth.** Register, log in, and receive a **JWT**. Passwords are hashed with bcrypt. Every user has a **role** (`customer` or `admin`). Customers can shop; admins can manage the catalog.
- **Categories.** A small CRUD resource (e.g. "Electronics", "Books"). Admin-only writes; public reads. Products reference a category.
- **Products.** The heart of the catalog. Full CRUD, **admin-only writes**, public reads. Each product can have an **uploaded image** (handled with **Multer**). The product *list* endpoint supports **pagination, filtering, sorting, field selection, and full-text-style search** — the exact feature set interviewers love to probe.
- **Cart.** Each customer has exactly one cart. Add items, change quantities, remove items, clear it. The cart stores a snapshot relationship to products and recomputes totals.
- **Orders.** **Create an order from the cart** — the single most interesting flow in the app. This validates stock, snapshots prices, **decrements inventory**, empties the cart, and records the order, ideally inside a **transaction** so it is all-or-nothing. Orders then move through a **status** lifecycle (`pending → paid → shipped → delivered`, or `cancelled`).

Around the core domain we wrap the things that make it *production-shaped* rather than a toy:

- **Layered architecture** — `routes → controllers → services → models` — so business logic is testable and controllers stay thin.
- **Centralized error handling** with a custom `ApiError` class and a single error-handling middleware.
- **Input validation** with `express-validator` at the edge of the system.
- **Security middleware**: `helmet`, `cors`, `express-rate-limit`, and `express-mongo-sanitize`.
- **Environment configuration** with `dotenv` and **fail-fast validation** of required variables at startup.
- **Logging** with **Winston** (application logs) and **Morgan** (HTTP access logs, piped into Winston).
- **A few real tests** with **Jest + Supertest** running against an in-memory MongoDB.
- **Swagger / OpenAPI docs** served at `/api-docs`.
- **A `Dockerfile` + `docker-compose.yml`** that brings the API and MongoDB up together with one command.

By the end you will have a repository you can push to GitHub, run with `docker compose up`, and demo live in an interview.

---

## What You'll Practice

- Designing a **clean layered architecture** and justifying each layer.
- Modelling related data in Mongoose: **references**, **subdocuments**, indexes, virtuals, and `pre`/`post` hooks.
- Implementing **JWT auth** end to end: hashing, signing, verifying, and an `authenticate` + `authorize(...roles)` middleware pair.
- Building a **robust product query layer**: pagination, filtering by range/category, sorting, field limiting, and search — driven entirely by query strings.
- Handling **file uploads** safely with Multer: storage, file-type and size limits, and serving uploaded files.
- Writing a **transactional "checkout"** that turns a cart into an order without overselling.
- Centralizing **error handling** and **async error propagation** so you never repeat `try/catch` in controllers.
- Hardening an API with **security middleware** and sane defaults.
- Wiring **structured logging** and **HTTP access logs**.
- Writing **integration tests** for protected routes with Supertest.
- Producing **interactive API docs** with Swagger.
- **Containerizing** the app and its database for reproducible runs and deployment.

---

## Prerequisites & Setup

You should have completed (or be comfortable with) the Week 3–5 material: Express routing & middleware, Mongoose, authentication/authorization, error handling & validation, file uploads, logging, testing, and the architecture module. You will need:

- **Node.js LTS** (`node --version` → v20+ or v22+).
- **MongoDB** — either a local install, a free **MongoDB Atlas** cluster, or (recommended for this project) the **Dockerized Mongo** from our `docker-compose.yml`.
- An API client: **Postman**, **Thunder Client**, or `curl`.

### Create the project and install dependencies

```bash
# 1. Make the project folder and initialize npm
mkdir ecommerce-api && cd ecommerce-api
npm init -y

# 2. Runtime dependencies
npm install express mongoose dotenv \
  bcryptjs jsonwebtoken \
  express-validator \
  helmet cors express-rate-limit express-mongo-sanitize \
  multer \
  morgan winston \
  swagger-jsdoc swagger-ui-express

# 3. Dev dependencies (auto-reload + testing)
npm install --save-dev nodemon jest supertest mongodb-memory-server cross-env
```

What each package does:

| Package | Role |
| --- | --- |
| `express` | The web framework (v5). |
| `mongoose` | ODM for MongoDB — schemas, models, queries, transactions. |
| `dotenv` | Loads `.env` into `process.env`. |
| `bcryptjs` | Password hashing (pure-JS, no native build headaches in Docker). |
| `jsonwebtoken` | Sign & verify JWT access tokens. |
| `express-validator` | Declarative request validation/sanitization. |
| `helmet` | Secure HTTP response headers. |
| `cors` | Cross-Origin Resource Sharing control. |
| `express-rate-limit` | Throttles abusive clients. |
| `express-mongo-sanitize` | Strips `$`/`.` keys to block NoSQL injection. |
| `multer` | `multipart/form-data` parsing for file uploads. |
| `morgan` | HTTP request access logging. |
| `winston` | Structured, leveled application logging. |
| `swagger-jsdoc` + `swagger-ui-express` | Generate & serve OpenAPI docs. |
| `nodemon` | Restart on file change (dev). |
| `jest`, `supertest`, `mongodb-memory-server` | Test runner, HTTP assertions, ephemeral Mongo. |
| `cross-env` | Set env vars cross-platform in npm scripts. |

> WARNING — Pitfall: We use **`bcryptjs`** (pure JavaScript), not `bcrypt` (native C++). `bcrypt` needs a compiler toolchain and frequently breaks inside slim Docker images. `bcryptjs` is a drop-in API-compatible alternative with no build step. If you ever see "node-gyp" errors in Docker, this is why.

> WARNING — Pitfall: `express-mongo-sanitize@^2` mutates `req.query`, which is **read-only in Express 5** and throws. We address this in the security section with a safe wrapper that sanitizes `body`/`params` and rebuilds `query` without reassigning it. Do not skip that note.

### `package.json` scripts

Open `package.json` and replace the `"scripts"` block:

```json
{
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js",
    "test": "cross-env NODE_ENV=test jest --runInBand --detectOpenHandles",
    "test:watch": "cross-env NODE_ENV=test jest --watch --runInBand"
  }
}
```

`--runInBand` forces Jest to run test files serially (one shared in-memory Mongo, no port clashes). `--detectOpenHandles` helps you find a connection you forgot to close.

### Folder structure

We use a **layered, feature-aware** structure. Routes define the HTTP surface, controllers translate HTTP ↔ domain, services hold business logic, models define data. Cross-cutting concerns (config, middleware, utils) live in their own folders.

```
ecommerce-api/
├── .env                       # local secrets (gitignored)
├── .env.example               # template, committed
├── .gitignore
├── .dockerignore
├── Dockerfile
├── docker-compose.yml
├── jest.config.js
├── package.json
├── uploads/                   # Multer writes product images here (gitignored, kept by .gitkeep)
│   └── .gitkeep
├── src/
│   ├── server.js              # entrypoint: load env, connect DB, start HTTP server
│   ├── app.js                 # builds & configures the Express app (no listen)
│   ├── config/
│   │   ├── env.js             # load + validate environment variables (fail-fast)
│   │   ├── db.js              # Mongoose connection helper
│   │   ├── logger.js          # Winston logger instance
│   │   └── swagger.js         # OpenAPI spec via swagger-jsdoc
│   ├── models/
│   │   ├── user.model.js
│   │   ├── category.model.js
│   │   ├── product.model.js
│   │   ├── cart.model.js
│   │   └── order.model.js
│   ├── services/
│   │   ├── auth.service.js
│   │   ├── category.service.js
│   │   ├── product.service.js
│   │   ├── cart.service.js
│   │   └── order.service.js
│   ├── controllers/
│   │   ├── auth.controller.js
│   │   ├── category.controller.js
│   │   ├── product.controller.js
│   │   ├── cart.controller.js
│   │   └── order.controller.js
│   ├── routes/
│   │   ├── index.js           # mounts all feature routers under /api/v1
│   │   ├── auth.routes.js
│   │   ├── category.routes.js
│   │   ├── product.routes.js
│   │   ├── cart.routes.js
│   │   └── order.routes.js
│   ├── middleware/
│   │   ├── auth.middleware.js     # authenticate + authorize(...roles)
│   │   ├── validate.middleware.js # runs express-validator chains
│   │   ├── upload.middleware.js   # Multer config
│   │   ├── error.middleware.js    # notFound + central error handler
│   │   └── security.middleware.js # mongo-sanitize wrapper (Express 5 safe)
│   ├── validators/
│   │   ├── auth.validator.js
│   │   ├── category.validator.js
│   │   ├── product.validator.js
│   │   ├── cart.validator.js
│   │   └── order.validator.js
│   └── utils/
│       ├── ApiError.js        # operational error class
│       ├── catchAsync.js      # async wrapper to forward errors to next()
│       ├── apiFeatures.js     # parse pagination/filter/sort/search query
│       └── token.js           # sign/verify JWT helpers
└── tests/
    ├── setup.js               # boot in-memory Mongo before tests
    ├── auth.test.js
    └── product.test.js
```

> BEST PRACTICE: Keep `server.js` (process concerns: env, DB connection, `listen`, signal handling) separate from `app.js` (the configured Express app). This single split is what makes Supertest testing trivial — tests import `app` and never bind a port.

---

## Step 1 — Configuration, Logging & the App Skeleton

We build from the inside out: configuration and logging first, then the Express app, then the entrypoint. Everything else plugs into this skeleton.

### `.gitignore` and `.dockerignore`

#### `.gitignore`

```gitignore
node_modules/
.env
uploads/*
!uploads/.gitkeep
coverage/
npm-debug.log*
```

#### `.dockerignore`

```dockerignore
node_modules
npm-debug.log
.env
.git
coverage
tests
uploads
```

### `.env.example` (commit this) and `.env` (do not)

#### `.env.example`

```bash
# Server
NODE_ENV=development
PORT=5000

# Database
MONGODB_URI=mongodb://localhost:27017/ecommerce

# JWT
JWT_SECRET=replace-me-with-a-long-random-string
JWT_EXPIRES_IN=1d

# CORS (comma-separated list of allowed origins, or * for all)
CORS_ORIGIN=*

# Rate limiting
RATE_LIMIT_WINDOW_MS=900000
RATE_LIMIT_MAX=100
```

Copy it to `.env` and fill in a real secret:

```bash
cp .env.example .env
# Generate a strong JWT secret:
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"
```

> WARNING — Pitfall: Never commit `.env`. A leaked `JWT_SECRET` lets anyone forge admin tokens. The `.env.example` documents the shape without leaking values.

### `src/config/env.js` — load and validate environment (fail-fast)

A server that boots with a missing `JWT_SECRET` is a server that will mysteriously 500 later. We validate at startup and **crash immediately** with a clear message if anything required is missing.

```js
// src/config/env.js
// Loads environment variables and validates them ONCE at startup.
// If a required variable is missing, we throw so the process never starts in a broken state.
const dotenv = require("dotenv");
const path = require("path");

// In test mode, load .env.test if present; otherwise the default .env.
const envFile = process.env.NODE_ENV === "test" ? ".env.test" : ".env";
dotenv.config({ path: path.resolve(process.cwd(), envFile) });

// Helper: read a required variable or throw.
function required(name, fallback) {
  const value = process.env[name] ?? fallback;
  if (value === undefined || value === "") {
    throw new Error(`Missing required environment variable: ${name}`);
  }
  return value;
}

const env = {
  nodeEnv: process.env.NODE_ENV || "development",
  port: parseInt(process.env.PORT || "5000", 10),

  // In test mode the in-memory server supplies the URI, so it isn't strictly required.
  mongoUri:
    process.env.NODE_ENV === "test"
      ? process.env.MONGODB_URI // may be undefined; tests inject it
      : required("MONGODB_URI"),

  jwtSecret: required("JWT_SECRET", process.env.NODE_ENV === "test" ? "test-secret" : undefined),
  jwtExpiresIn: process.env.JWT_EXPIRES_IN || "1d",

  corsOrigin: process.env.CORS_ORIGIN || "*",

  rateLimit: {
    windowMs: parseInt(process.env.RATE_LIMIT_WINDOW_MS || "900000", 10), // 15 min
    max: parseInt(process.env.RATE_LIMIT_MAX || "100", 10),
  },
};

module.exports = env;
```

> BEST PRACTICE: Centralize all access to `process.env` in one module. The rest of the codebase imports a typed-ish `env` object, never `process.env` directly. This makes the full configuration surface visible in one file and trivially mockable in tests.

### `src/config/logger.js` — Winston

```js
// src/config/logger.js
// A single Winston logger used everywhere. JSON in production (machine-readable for
// log aggregators); colorized, human-friendly output in development.
const winston = require("winston");
const env = require("./env");

const { combine, timestamp, printf, colorize, json, errors } = winston.format;

// Pretty single-line format for local dev.
const devFormat = combine(
  colorize(),
  timestamp({ format: "HH:mm:ss" }),
  errors({ stack: true }), // include stack traces for Error objects
  printf(({ level, message, timestamp, stack }) => {
    return `${timestamp} ${level}: ${stack || message}`;
  })
);

// Structured JSON for production.
const prodFormat = combine(timestamp(), errors({ stack: true }), json());

const logger = winston.createLogger({
  level: env.nodeEnv === "development" ? "debug" : "info",
  format: env.nodeEnv === "production" ? prodFormat : devFormat,
  transports: [new winston.transports.Console()],
  // Don't exit the process on a handled exception transport error.
  exitOnError: false,
  // Silence logs during tests to keep output clean.
  silent: env.nodeEnv === "test",
});

module.exports = logger;
```

### `src/config/db.js` — Mongoose connection

```js
// src/config/db.js
// Connects to MongoDB via Mongoose and wires up connection event logging.
const mongoose = require("mongoose");
const logger = require("./logger");

async function connectDB(uri) {
  // Fail fast if the initial connection can't be established within 10s.
  mongoose.set("strictQuery", true);

  mongoose.connection.on("connected", () => logger.info("MongoDB connected"));
  mongoose.connection.on("error", (err) => logger.error(`MongoDB error: ${err.message}`));
  mongoose.connection.on("disconnected", () => logger.warn("MongoDB disconnected"));

  await mongoose.connect(uri, {
    serverSelectionTimeoutMS: 10000,
  });

  return mongoose.connection;
}

async function disconnectDB() {
  await mongoose.connection.close();
}

module.exports = { connectDB, disconnectDB };
```

### Utilities: `ApiError`, `catchAsync`, `token`

These three tiny helpers are used pervasively. Build them now.

#### `src/utils/ApiError.js`

```js
// src/utils/ApiError.js
// An "operational" error — an expected failure (bad input, not found, unauthorized)
// as opposed to a programmer bug. The central error handler treats these specially:
// it trusts their statusCode and message and returns them to the client.
class ApiError extends Error {
  constructor(statusCode, message, details = undefined) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = true; // distinguishes expected errors from bugs
    this.details = details; // optional array/object of field errors
    Error.captureStackTrace(this, this.constructor);
  }

  // Convenience factories for the common cases — read nicely at call sites.
  static badRequest(msg = "Bad Request", details) {
    return new ApiError(400, msg, details);
  }
  static unauthorized(msg = "Unauthorized") {
    return new ApiError(401, msg);
  }
  static forbidden(msg = "Forbidden") {
    return new ApiError(403, msg);
  }
  static notFound(msg = "Resource not found") {
    return new ApiError(404, msg);
  }
  static conflict(msg = "Conflict") {
    return new ApiError(409, msg);
  }
}

module.exports = ApiError;
```

#### `src/utils/catchAsync.js`

```js
// src/utils/catchAsync.js
// Wraps an async route handler so any rejected promise is forwarded to Express's
// error handler via next(err). This removes try/catch from every controller.
//
// NOTE ON EXPRESS 5: Express 5 actually awaits returned promises from handlers and
// forwards rejections to next() automatically. We keep this wrapper anyway because
// (a) it is explicit and portable to v4, and (b) it documents intent. With v5 you
// could drop it for plain `async` handlers — but mixing styles is confusing, so we
// use the wrapper consistently throughout.
module.exports = function catchAsync(fn) {
  return function (req, res, next) {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
};
```

#### `src/utils/token.js`

```js
// src/utils/token.js
// Thin wrappers around jsonwebtoken so signing config lives in exactly one place.
const jwt = require("jsonwebtoken");
const env = require("../config/env");

// Sign an access token. `payload` should be small — just identity + role.
function signToken(payload) {
  return jwt.sign(payload, env.jwtSecret, { expiresIn: env.jwtExpiresIn });
}

// Verify and decode a token; throws if invalid/expired (caller handles).
function verifyToken(token) {
  return jwt.verify(token, env.jwtSecret);
}

module.exports = { signToken, verifyToken };
```

> BEST PRACTICE: Keep JWT **payloads minimal** — typically just the user id and role. Never put a password hash, email, or anything secret in a JWT: the payload is Base64-encoded, **not encrypted**, and is fully readable by anyone who holds the token.

---

## Step 2 — Security Middleware, Error Handling & the Express App

### `src/middleware/security.middleware.js` — Express-5-safe Mongo sanitization

`express-mongo-sanitize` defends against **NoSQL injection**: a malicious payload like `{ "email": { "$gt": "" } }` can turn a `findOne` into "match any user." The library strips keys containing `$` or `.`. The catch: its default middleware reassigns `req.query`, which **Express 5 made a read-only getter**, throwing `Cannot set property query`. We sanitize `body`/`params` in place and rebuild `query` without reassigning the property.

```js
// src/middleware/security.middleware.js
// NoSQL-injection sanitization that is safe under Express 5 (read-only req.query).
const mongoSanitize = require("express-mongo-sanitize");

function sanitizeRequest(req, _res, next) {
  // body and params are plain writable objects — sanitize in place.
  if (req.body) mongoSanitize.sanitize(req.body, { replaceWith: "_" });
  if (req.params) mongoSanitize.sanitize(req.params, { replaceWith: "_" });

  // req.query is a getter in Express 5; we cannot reassign it. Mutate its keys in place.
  if (req.query) {
    for (const key of Object.keys(req.query)) {
      // Drop any key that contains a Mongo operator character.
      if (/[$.]/.test(key)) {
        delete req.query[key];
      } else {
        mongoSanitize.sanitize(req.query[key], { replaceWith: "_" });
      }
    }
  }
  next();
}

module.exports = { sanitizeRequest };
```

> WARNING — Pitfall: In **Express 4** you could simply `app.use(mongoSanitize())`. In **Express 5** that throws because `req.query` is no longer assignable. The wrapper above is the portable fix. (`express-validator`'s sanitizers on `req.body` also help, but defense-in-depth is the point.)

### `src/middleware/error.middleware.js` — 404 + central error handler

Two pieces: a `notFound` that turns unmatched routes into a 404 `ApiError`, and the **single** error-handling middleware (four arguments) that formats every error response.

```js
// src/middleware/error.middleware.js
const ApiError = require("../utils/ApiError");
const logger = require("../config/logger");
const env = require("../config/env");

// Reached only when no route matched. Forward a 404 to the error handler.
function notFound(req, _res, next) {
  next(ApiError.notFound(`Route not found: ${req.method} ${req.originalUrl}`));
}

// The ONE error handler. Express identifies it by its 4-arg signature.
// eslint-disable-next-line no-unused-vars
function errorHandler(err, req, res, next) {
  let error = err;

  // --- Normalize common non-ApiError errors into ApiError ---

  // Mongoose bad ObjectId (e.g. GET /products/not-an-id)
  if (err.name === "CastError") {
    error = ApiError.badRequest(`Invalid ${err.path}: ${err.value}`);
  }
  // Mongoose schema validation failure
  else if (err.name === "ValidationError") {
    const details = Object.values(err.errors).map((e) => ({
      field: e.path,
      message: e.message,
    }));
    error = ApiError.badRequest("Validation failed", details);
  }
  // Duplicate key (e.g. email already registered)
  else if (err.code === 11000) {
    const field = Object.keys(err.keyValue || {})[0] || "field";
    error = ApiError.conflict(`Duplicate value for ${field}`);
  }
  // JWT errors
  else if (err.name === "JsonWebTokenError") {
    error = ApiError.unauthorized("Invalid token");
  } else if (err.name === "TokenExpiredError") {
    error = ApiError.unauthorized("Token expired");
  }
  // Multer upload errors (size/type)
  else if (err.name === "MulterError") {
    error = ApiError.badRequest(`Upload error: ${err.message}`);
  }

  // Anything not an ApiError by now is an unexpected programmer error → 500.
  if (!(error instanceof ApiError)) {
    error = new ApiError(500, "Internal Server Error");
    error.isOperational = false;
  }

  // Log: full stack for non-operational (real bugs); concise line for operational.
  if (!error.isOperational || error.statusCode >= 500) {
    logger.error(`${req.method} ${req.originalUrl} -> ${error.statusCode}`, { stack: err.stack });
  } else {
    logger.warn(`${req.method} ${req.originalUrl} -> ${error.statusCode}: ${error.message}`);
  }

  const body = {
    success: false,
    message: error.message,
  };
  if (error.details) body.details = error.details;
  // Only leak stack traces outside production.
  if (env.nodeEnv !== "production" && !error.isOperational) {
    body.stack = err.stack;
  }

  res.status(error.statusCode).json(body);
}

module.exports = { notFound, errorHandler };
```

> WARNING — Pitfall: The error handler **must be registered last**, after all routes, and it **must have exactly four parameters** (`err, req, res, next`). If you write three, Express treats it as a normal middleware and your errors fall through to the default HTML error page.

### `src/app.js` — build the configured Express app

This module wires up middleware in the correct order and mounts routes. It does **not** call `listen` — that is `server.js`'s job, which is what makes the app testable.

```js
// src/app.js
const express = require("express");
const helmet = require("helmet");
const cors = require("cors");
const rateLimit = require("express-rate-limit");
const morgan = require("morgan");
const path = require("path");
const swaggerUi = require("swagger-ui-express");

const env = require("./config/env");
const logger = require("./config/logger");
const swaggerSpec = require("./config/swagger");
const { sanitizeRequest } = require("./middleware/security.middleware");
const { notFound, errorHandler } = require("./middleware/error.middleware");
const apiRouter = require("./routes");

const app = express();

// Trust the first proxy (needed for correct client IPs behind nginx / a load balancer,
// which makes rate-limiting accurate). Adjust for your deployment topology.
app.set("trust proxy", 1);

// --- Security & parsing middleware (order matters) ---

// 1. Secure headers.
app.use(helmet());

// 2. CORS. corsOrigin "*" allows any origin; otherwise pass a list.
app.use(
  cors({
    origin: env.corsOrigin === "*" ? true : env.corsOrigin.split(",").map((s) => s.trim()),
    credentials: true,
  })
);

// 3. Body parsers (with a size cap to blunt large-payload DoS).
app.use(express.json({ limit: "10kb" }));
app.use(express.urlencoded({ extended: true, limit: "10kb" }));

// 4. NoSQL-injection sanitization (Express-5-safe wrapper).
app.use(sanitizeRequest);

// 5. HTTP access logging via Morgan, piped into Winston.
const morganFormat = env.nodeEnv === "development" ? "dev" : "combined";
app.use(
  morgan(morganFormat, {
    stream: { write: (message) => logger.info(message.trim()) },
    skip: () => env.nodeEnv === "test",
  })
);

// 6. Global rate limiter (protects every route; auth routes get a stricter one too).
app.use(
  rateLimit({
    windowMs: env.rateLimit.windowMs,
    max: env.rateLimit.max,
    standardHeaders: true, // RateLimit-* headers
    legacyHeaders: false,
    message: { success: false, message: "Too many requests, please try again later." },
  })
);

// --- Static files: serve uploaded product images ---
app.use("/uploads", express.static(path.join(__dirname, "..", "uploads")));

// --- Health check (useful for Docker/k8s probes & load balancers) ---
app.get("/health", (_req, res) => {
  res.json({ status: "ok", uptime: process.uptime(), timestamp: new Date().toISOString() });
});

// --- API docs ---
app.use("/api-docs", swaggerUi.serve, swaggerUi.setup(swaggerSpec));

// --- Versioned API routes ---
app.use("/api/v1", apiRouter);

// --- 404 + central error handler (always LAST) ---
app.use(notFound);
app.use(errorHandler);

module.exports = app;
```

> WARNING — Pitfall: Middleware order is not cosmetic. Body parsers must run **before** anything that reads `req.body` (sanitizer, validators, controllers). The error handler must run **after** the routes. Get this wrong and you will chase phantom "undefined body" or "errors not caught" bugs.

### `src/server.js` — the entrypoint

```js
// src/server.js
// Process-level concerns: connect the DB, start the HTTP server, handle shutdown.
const app = require("./app");
const env = require("./config/env");
const logger = require("./config/logger");
const { connectDB, disconnectDB } = require("./config/db");

let server;

async function start() {
  await connectDB(env.mongoUri);
  server = app.listen(env.port, () => {
    logger.info(`Server running in ${env.nodeEnv} mode on port ${env.port}`);
    logger.info(`API docs at http://localhost:${env.port}/api-docs`);
  });
}

// --- Graceful shutdown: stop accepting connections, close DB, then exit ---
async function shutdown(signal) {
  logger.warn(`${signal} received. Shutting down gracefully...`);
  if (server) {
    server.close(async () => {
      await disconnectDB();
      logger.info("Closed out remaining connections. Bye.");
      process.exit(0);
    });
    // Force-exit if it hangs.
    setTimeout(() => {
      logger.error("Could not close in time, forcing shutdown.");
      process.exit(1);
    }, 10000).unref();
  } else {
    process.exit(0);
  }
}

// --- Crash safety: log and exit on truly unexpected failures ---
process.on("unhandledRejection", (reason) => {
  logger.error("Unhandled Rejection:", reason);
  shutdown("unhandledRejection");
});
process.on("uncaughtException", (err) => {
  logger.error("Uncaught Exception:", err);
  process.exit(1); // an uncaught exception leaves the process in an unknown state
});
process.on("SIGTERM", () => shutdown("SIGTERM"));
process.on("SIGINT", () => shutdown("SIGINT"));

start().catch((err) => {
  logger.error("Failed to start server:", err);
  process.exit(1);
});
```

> BEST PRACTICE: **Graceful shutdown** matters in containers. When Docker/Kubernetes sends `SIGTERM`, you want to finish in-flight requests and close the DB cleanly rather than dropping connections. The `.unref()` on the force-exit timer keeps it from holding the process open on its own.

---

## Step 3 — Models

Five Mongoose models. Each owns its schema, indexes, virtuals, and hooks. Define them before services because services depend on them.

### `src/models/user.model.js`

```js
// src/models/user.model.js
const mongoose = require("mongoose");
const bcrypt = require("bcryptjs");

const userSchema = new mongoose.Schema(
  {
    name: { type: String, required: [true, "Name is required"], trim: true, maxlength: 80 },
    email: {
      type: String,
      required: [true, "Email is required"],
      unique: true, // creates a unique index
      lowercase: true,
      trim: true,
      match: [/^\S+@\S+\.\S+$/, "Invalid email format"],
    },
    password: {
      type: String,
      required: [true, "Password is required"],
      minlength: 8,
      select: false, // never returned by default queries — must be explicitly selected
    },
    role: { type: String, enum: ["customer", "admin"], default: "customer" },
  },
  { timestamps: true } // adds createdAt / updatedAt
);

// Hash the password before saving — but only if it changed (so updates don't double-hash).
userSchema.pre("save", async function () {
  if (!this.isModified("password")) return;
  this.password = await bcrypt.hash(this.password, 12);
});

// Instance method to verify a plaintext password against the stored hash.
userSchema.methods.comparePassword = function (candidate) {
  return bcrypt.compare(candidate, this.password);
};

// Strip sensitive/internal fields from every JSON serialization.
userSchema.set("toJSON", {
  transform(_doc, ret) {
    delete ret.password;
    delete ret.__v;
    return ret;
  },
});

module.exports = mongoose.model("User", userSchema);
```

> WARNING — Pitfall: `select: false` on `password` means a normal `User.findOne()` will **not** include the hash. For login you must explicitly opt in: `User.findOne({ email }).select("+password")`. Forgetting this makes `comparePassword` crash on `undefined`.

> WARNING — Pitfall: Arrow functions break Mongoose hooks and methods — they don't bind `this` to the document. Always use `function () {}` for `pre`, `methods`, and `virtual` getters.

### `src/models/category.model.js`

```js
// src/models/category.model.js
const mongoose = require("mongoose");

const categorySchema = new mongoose.Schema(
  {
    name: { type: String, required: true, unique: true, trim: true, maxlength: 60 },
    slug: { type: String, required: true, unique: true, lowercase: true, trim: true },
    description: { type: String, trim: true, maxlength: 300 },
  },
  { timestamps: true }
);

// Auto-generate a URL-friendly slug from the name before validation runs.
categorySchema.pre("validate", function () {
  if (this.name && (this.isModified("name") || !this.slug)) {
    this.slug = this.name
      .toLowerCase()
      .trim()
      .replace(/[^a-z0-9]+/g, "-") // non-alphanumerics → hyphens
      .replace(/^-+|-+$/g, ""); // trim leading/trailing hyphens
  }
});

module.exports = mongoose.model("Category", categorySchema);
```

### `src/models/product.model.js`

```js
// src/models/product.model.js
const mongoose = require("mongoose");

const productSchema = new mongoose.Schema(
  {
    name: { type: String, required: [true, "Name is required"], trim: true, maxlength: 140 },
    description: { type: String, trim: true, maxlength: 2000, default: "" },
    price: { type: Number, required: [true, "Price is required"], min: [0, "Price cannot be negative"] },
    stock: { type: Number, required: true, min: [0, "Stock cannot be negative"], default: 0 },
    category: { type: mongoose.Schema.Types.ObjectId, ref: "Category", required: true },
    // Relative path served via /uploads; null when no image uploaded.
    image: { type: String, default: null },
    brand: { type: String, trim: true, default: "" },
    isActive: { type: Boolean, default: true },
  },
  { timestamps: true }
);

// Text index powers keyword search across name + description.
productSchema.index({ name: "text", description: "text" });
// Compound index that matches our most common filter+sort (by category, newest first).
productSchema.index({ category: 1, createdAt: -1 });
// Price index helps range filters and price sorts.
productSchema.index({ price: 1 });

// Virtual: a fully-qualified image URL (computed, not stored).
// Note: needs the request host to be truly absolute; here we expose the relative path
// and let clients prefix the host, which is simpler and host-agnostic.
productSchema.set("toJSON", { virtuals: true });

module.exports = mongoose.model("Product", productSchema);
```

> TIP: A MongoDB collection can have **only one text index**, but it may span multiple fields (as above). For richer search (typo tolerance, relevance tuning) you would graduate to Atlas Search or Elasticsearch — mention that in interviews to show you know the ceiling of `$text`.

### `src/models/cart.model.js`

A cart belongs to one user and holds line items. Each item references a product and stores the quantity plus a **price snapshot** (so a later price change does not silently alter what is "in the cart"). We compute totals as virtuals after populating products.

```js
// src/models/cart.model.js
const mongoose = require("mongoose");

const cartItemSchema = new mongoose.Schema(
  {
    product: { type: mongoose.Schema.Types.ObjectId, ref: "Product", required: true },
    quantity: { type: Number, required: true, min: [1, "Quantity must be at least 1"] },
    // Price captured when the item was added; refreshed on quantity changes.
    priceSnapshot: { type: Number, required: true, min: 0 },
  },
  { _id: false } // line items don't need their own _id
);

const cartSchema = new mongoose.Schema(
  {
    user: { type: mongoose.Schema.Types.ObjectId, ref: "User", required: true, unique: true },
    items: { type: [cartItemSchema], default: [] },
  },
  { timestamps: true, toJSON: { virtuals: true }, toObject: { virtuals: true } }
);

// Virtual: total number of units in the cart.
cartSchema.virtual("itemCount").get(function () {
  return this.items.reduce((sum, i) => sum + i.quantity, 0);
});

// Virtual: subtotal in currency units.
cartSchema.virtual("subtotal").get(function () {
  return this.items.reduce((sum, i) => sum + i.priceSnapshot * i.quantity, 0);
});

module.exports = mongoose.model("Cart", cartSchema);
```

### `src/models/order.model.js`

An order is an **immutable snapshot** of a purchase. We copy product name and price into each line item so the order is meaningful even if the product is later edited or deleted.

```js
// src/models/order.model.js
const mongoose = require("mongoose");

const orderItemSchema = new mongoose.Schema(
  {
    product: { type: mongoose.Schema.Types.ObjectId, ref: "Product", required: true },
    name: { type: String, required: true }, // snapshot of product name at purchase
    price: { type: Number, required: true, min: 0 }, // snapshot of unit price
    quantity: { type: Number, required: true, min: 1 },
  },
  { _id: false }
);

const ORDER_STATUSES = ["pending", "paid", "shipped", "delivered", "cancelled"];

const orderSchema = new mongoose.Schema(
  {
    user: { type: mongoose.Schema.Types.ObjectId, ref: "User", required: true, index: true },
    items: { type: [orderItemSchema], required: true, validate: (v) => v.length > 0 },
    totalAmount: { type: Number, required: true, min: 0 },
    status: { type: String, enum: ORDER_STATUSES, default: "pending", index: true },
    shippingAddress: {
      fullName: { type: String, required: true },
      line1: { type: String, required: true },
      city: { type: String, required: true },
      postalCode: { type: String, required: true },
      country: { type: String, required: true },
    },
    paidAt: { type: Date },
    deliveredAt: { type: Date },
  },
  { timestamps: true }
);

module.exports = mongoose.model("Order", orderSchema);
module.exports.ORDER_STATUSES = ORDER_STATUSES;
```

> BEST PRACTICE: **Snapshot prices and names into orders.** An order is a historical record; it must not change when a product's price is updated next month. This is one of the most common e-commerce design questions in interviews — referencing the product *alone* is a bug.

---

## Step 4 — The Query Layer: `apiFeatures` (pagination, filter, sort, search)

The product list endpoint is the showpiece for query handling. Rather than scatter query parsing across controllers, we encapsulate it in a reusable helper that takes a Mongoose query and the request's `req.query` and applies all five transforms.

```js
// src/utils/apiFeatures.js
// Builds a Mongoose query from URL query-string parameters:
//   filtering:  ?price[gte]=10&price[lte]=50&category=<id>&brand=Acme
//   search:     ?search=phone           (uses the text index)
//   sorting:    ?sort=-price,createdAt   (- prefix = descending)
//   fields:     ?fields=name,price       (projection)
//   pagination: ?page=2&limit=20
class ApiFeatures {
  constructor(query, queryString) {
    this.query = query; // a Mongoose Query (Model.find())
    this.queryString = queryString; // req.query
    this.meta = {}; // pagination metadata, filled in paginate()
  }

  filter() {
    // Copy and remove non-filter keys so they don't leak into the Mongo filter.
    const queryObj = { ...this.queryString };
    ["page", "sort", "limit", "fields", "search"].forEach((k) => delete queryObj[k]);

    // Convert ?price[gte]=10 -> { price: { $gte: 10 } } by prefixing operators with $.
    let mongoFilter = {};
    for (const [key, value] of Object.entries(queryObj)) {
      if (value && typeof value === "object") {
        // e.g. value = { gte: "10", lte: "50" }
        mongoFilter[key] = {};
        for (const [op, opVal] of Object.entries(value)) {
          if (["gt", "gte", "lt", "lte", "ne", "in"].includes(op)) {
            mongoFilter[key][`$${op}`] = isNaN(opVal) ? opVal : Number(opVal);
          }
        }
      } else {
        mongoFilter[key] = value;
      }
    }
    this.query = this.query.find(mongoFilter);
    return this;
  }

  search() {
    if (this.queryString.search) {
      // $text uses the product text index (name + description).
      this.query = this.query.find({ $text: { $search: this.queryString.search } });
    }
    return this;
  }

  sort() {
    if (this.queryString.sort) {
      // "-price,createdAt" -> "-price createdAt" (Mongoose's sort syntax)
      const sortBy = this.queryString.sort.split(",").join(" ");
      this.query = this.query.sort(sortBy);
    } else {
      this.query = this.query.sort("-createdAt"); // newest first by default
    }
    return this;
  }

  limitFields() {
    if (this.queryString.fields) {
      const fields = this.queryString.fields.split(",").join(" ");
      this.query = this.query.select(fields);
    } else {
      this.query = this.query.select("-__v"); // drop the version key by default
    }
    return this;
  }

  paginate() {
    const page = Math.max(parseInt(this.queryString.page, 10) || 1, 1);
    // Cap limit to protect the DB from "?limit=1000000" abuse.
    const limit = Math.min(Math.max(parseInt(this.queryString.limit, 10) || 12, 1), 100);
    const skip = (page - 1) * limit;
    this.query = this.query.skip(skip).limit(limit);
    this.meta = { page, limit }; // total/pages added by the service after counting
    return this;
  }
}

module.exports = ApiFeatures;
```

> WARNING — Pitfall: Always **cap `limit`**. Without the `Math.min(..., 100)`, a single request for `?limit=10000000` can drag the whole database. Capping is the cheapest abuse defense you can add.

> TIP: `skip/limit` pagination is simple and perfect here, but `skip` gets slower the deeper you page (it still scans skipped docs). For very large datasets or infinite scroll, mention **cursor/keyset pagination** (`?after=<lastId>` with `_id > lastId`) in interviews — it's O(1) per page.

---

## Step 5 — Auth: Service, Middleware, Validator, Controller, Routes

We follow the layered pattern for every resource. Auth first, since everything else is protected by it.

### `src/services/auth.service.js`

```js
// src/services/auth.service.js
const User = require("../models/user.model");
const ApiError = require("../utils/ApiError");
const { signToken } = require("../utils/token");

async function register({ name, email, password, role }) {
  // Service-level guard against duplicates (DB unique index is the ultimate guarantee).
  const exists = await User.findOne({ email });
  if (exists) throw ApiError.conflict("Email already registered");

  // Only allow self-registration as a customer. Admins are created by other admins/seeds.
  const user = await User.create({ name, email, password, role: "customer" });

  const token = signToken({ sub: user._id.toString(), role: user.role });
  return { user, token };
}

async function login({ email, password }) {
  // Must explicitly select the password (select:false on the schema).
  const user = await User.findOne({ email }).select("+password");
  // Same generic message whether email or password is wrong — avoids user enumeration.
  if (!user || !(await user.comparePassword(password))) {
    throw ApiError.unauthorized("Invalid email or password");
  }
  const token = signToken({ sub: user._id.toString(), role: user.role });
  return { user, token };
}

async function getMe(userId) {
  const user = await User.findById(userId);
  if (!user) throw ApiError.notFound("User not found");
  return user;
}

module.exports = { register, login, getMe };
```

> BEST PRACTICE: Return an **identical error message** for "no such email" and "wrong password." Distinct messages let attackers enumerate which emails are registered. Also note we hard-code `role: "customer"` on registration so nobody can self-promote to admin by sending `"role":"admin"` in the body.

### `src/middleware/auth.middleware.js`

```js
// src/middleware/auth.middleware.js
const User = require("../models/user.model");
const ApiError = require("../utils/ApiError");
const catchAsync = require("../utils/catchAsync");
const { verifyToken } = require("../utils/token");

// Verifies the Bearer token, loads the user, and attaches it to req.user.
const authenticate = catchAsync(async (req, _res, next) => {
  const header = req.headers.authorization || "";
  if (!header.startsWith("Bearer ")) {
    throw ApiError.unauthorized("Authentication token missing");
  }
  const token = header.split(" ")[1];

  // verifyToken throws on invalid/expired — normalized by the error handler.
  const decoded = verifyToken(token);

  // Re-fetch the user so deleted/disabled accounts can't keep using old tokens.
  const user = await User.findById(decoded.sub);
  if (!user) throw ApiError.unauthorized("User no longer exists");

  req.user = user;
  next();
});

// Role gate. Use AFTER authenticate: authorize("admin").
function authorize(...roles) {
  return (req, _res, next) => {
    if (!req.user || !roles.includes(req.user.role)) {
      return next(ApiError.forbidden("You do not have permission to perform this action"));
    }
    next();
  };
}

module.exports = { authenticate, authorize };
```

> BEST PRACTICE: Re-load the user from the DB inside `authenticate` instead of trusting the token blindly. It costs one indexed lookup and immediately revokes access for deleted or banned users — a token alone can't be "un-issued" before it expires.

### `src/middleware/validate.middleware.js`

```js
// src/middleware/validate.middleware.js
const { validationResult } = require("express-validator");
const ApiError = require("../utils/ApiError");

// Runs after a set of express-validator chains; collects and formats any errors.
function validate(req, _res, next) {
  const result = validationResult(req);
  if (result.isEmpty()) return next();

  const details = result.array().map((e) => ({ field: e.path, message: e.msg }));
  next(ApiError.badRequest("Validation failed", details));
}

module.exports = { validate };
```

### `src/validators/auth.validator.js`

```js
// src/validators/auth.validator.js
const { body } = require("express-validator");

const registerRules = [
  body("name").trim().notEmpty().withMessage("Name is required").isLength({ max: 80 }),
  body("email").trim().isEmail().withMessage("Valid email is required").normalizeEmail(),
  body("password")
    .isLength({ min: 8 })
    .withMessage("Password must be at least 8 characters")
    .matches(/\d/)
    .withMessage("Password must contain a number"),
];

const loginRules = [
  body("email").trim().isEmail().withMessage("Valid email is required").normalizeEmail(),
  body("password").notEmpty().withMessage("Password is required"),
];

module.exports = { registerRules, loginRules };
```

### `src/controllers/auth.controller.js`

```js
// src/controllers/auth.controller.js
// Controllers are THIN: pull inputs off req, call a service, shape the response.
const authService = require("../services/auth.service");
const catchAsync = require("../utils/catchAsync");

const register = catchAsync(async (req, res) => {
  const { user, token } = await authService.register(req.body);
  res.status(201).json({ success: true, data: { user, token } });
});

const login = catchAsync(async (req, res) => {
  const { user, token } = await authService.login(req.body);
  res.status(200).json({ success: true, data: { user, token } });
});

const me = catchAsync(async (req, res) => {
  // req.user is set by authenticate.
  const user = await authService.getMe(req.user._id);
  res.status(200).json({ success: true, data: { user } });
});

module.exports = { register, login, me };
```

### `src/routes/auth.routes.js`

```js
// src/routes/auth.routes.js
const express = require("express");
const rateLimit = require("express-rate-limit");
const authController = require("../controllers/auth.controller");
const { authenticate } = require("../middleware/auth.middleware");
const { validate } = require("../middleware/validate.middleware");
const { registerRules, loginRules } = require("../validators/auth.validator");

const router = express.Router();

// Stricter limiter on auth endpoints to slow brute-force/credential-stuffing.
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 10,
  message: { success: false, message: "Too many auth attempts, try again later." },
});

router.post("/register", authLimiter, registerRules, validate, authController.register);
router.post("/login", authLimiter, loginRules, validate, authController.login);
router.get("/me", authenticate, authController.me);

module.exports = router;
```

> TIP: Layer a **stricter rate limiter on `/login` and `/register`** than the global one. Login is the highest-value target for brute force; a low cap there (e.g. 10/15 min) costs legitimate users nothing but seriously hampers attackers.

---

## Step 6 — Categories (a complete CRUD slice)

Categories are the simplest full resource — use them to internalize the route → controller → service rhythm before the bigger ones.

### `src/services/category.service.js`

```js
// src/services/category.service.js
const Category = require("../models/category.model");
const Product = require("../models/product.model");
const ApiError = require("../utils/ApiError");

async function createCategory(data) {
  return Category.create(data);
}

async function listCategories() {
  return Category.find().sort("name");
}

async function getCategory(id) {
  const category = await Category.findById(id);
  if (!category) throw ApiError.notFound("Category not found");
  return category;
}

async function updateCategory(id, data) {
  const category = await Category.findByIdAndUpdate(id, data, {
    new: true, // return the updated doc
    runValidators: true, // enforce schema rules on update
  });
  if (!category) throw ApiError.notFound("Category not found");
  return category;
}

async function deleteCategory(id) {
  // Block deletion if products still reference this category — protects referential integrity.
  const inUse = await Product.countDocuments({ category: id });
  if (inUse > 0) {
    throw ApiError.conflict(`Cannot delete: ${inUse} product(s) still use this category`);
  }
  const category = await Category.findByIdAndDelete(id);
  if (!category) throw ApiError.notFound("Category not found");
  return category;
}

module.exports = { createCategory, listCategories, getCategory, updateCategory, deleteCategory };
```

### `src/validators/category.validator.js`

```js
// src/validators/category.validator.js
const { body, param } = require("express-validator");

const idRule = [param("id").isMongoId().withMessage("Invalid category id")];

const createRules = [
  body("name").trim().notEmpty().withMessage("Name is required").isLength({ max: 60 }),
  body("description").optional().trim().isLength({ max: 300 }),
];

const updateRules = [
  ...idRule,
  body("name").optional().trim().notEmpty().isLength({ max: 60 }),
  body("description").optional().trim().isLength({ max: 300 }),
];

module.exports = { idRule, createRules, updateRules };
```

### `src/controllers/category.controller.js`

```js
// src/controllers/category.controller.js
const categoryService = require("../services/category.service");
const catchAsync = require("../utils/catchAsync");

const create = catchAsync(async (req, res) => {
  const category = await categoryService.createCategory(req.body);
  res.status(201).json({ success: true, data: category });
});

const list = catchAsync(async (_req, res) => {
  const categories = await categoryService.listCategories();
  res.json({ success: true, results: categories.length, data: categories });
});

const getOne = catchAsync(async (req, res) => {
  const category = await categoryService.getCategory(req.params.id);
  res.json({ success: true, data: category });
});

const update = catchAsync(async (req, res) => {
  const category = await categoryService.updateCategory(req.params.id, req.body);
  res.json({ success: true, data: category });
});

const remove = catchAsync(async (req, res) => {
  await categoryService.deleteCategory(req.params.id);
  res.status(204).send();
});

module.exports = { create, list, getOne, update, remove };
```

### `src/routes/category.routes.js`

```js
// src/routes/category.routes.js
const express = require("express");
const categoryController = require("../controllers/category.controller");
const { authenticate, authorize } = require("../middleware/auth.middleware");
const { validate } = require("../middleware/validate.middleware");
const { idRule, createRules, updateRules } = require("../validators/category.validator");

const router = express.Router();

// Public reads.
router.get("/", categoryController.list);
router.get("/:id", idRule, validate, categoryController.getOne);

// Admin-only writes.
router.post("/", authenticate, authorize("admin"), createRules, validate, categoryController.create);
router.patch("/:id", authenticate, authorize("admin"), updateRules, validate, categoryController.update);
router.delete("/:id", authenticate, authorize("admin"), idRule, validate, categoryController.remove);

module.exports = router;
```

---

## Step 7 — Products with Image Upload (Multer)

This is the catalog and the upload flow combined.

### `src/middleware/upload.middleware.js` — Multer config

```js
// src/middleware/upload.middleware.js
const multer = require("multer");
const path = require("path");
const crypto = require("crypto");
const ApiError = require("../utils/ApiError");

// Disk storage: write to /uploads with a collision-proof, sanitized filename.
const storage = multer.diskStorage({
  destination: (_req, _file, cb) => cb(null, path.join(__dirname, "..", "..", "uploads")),
  filename: (_req, file, cb) => {
    // e.g. 1718800000000-3f9a2b.png — never trust the original filename for the path.
    const unique = `${Date.now()}-${crypto.randomBytes(4).toString("hex")}`;
    const ext = path.extname(file.originalname).toLowerCase();
    cb(null, `${unique}${ext}`);
  },
});

// Only allow common image types — check BOTH the mimetype and the extension.
const ALLOWED = ["image/jpeg", "image/png", "image/webp", "image/gif"];
function fileFilter(_req, file, cb) {
  const extOk = /\.(jpe?g|png|webp|gif)$/i.test(file.originalname);
  if (ALLOWED.includes(file.mimetype) && extOk) {
    cb(null, true);
  } else {
    cb(ApiError.badRequest("Only image files (jpg, png, webp, gif) are allowed"));
  }
}

const upload = multer({
  storage,
  fileFilter,
  limits: { fileSize: 2 * 1024 * 1024, files: 1 }, // 2 MB, single file
});

// Export the specific single-field uploader we need ("image").
module.exports = { uploadProductImage: upload.single("image") };
```

> WARNING — Pitfall: Never build the saved path from `file.originalname`. A malicious name like `../../server.js` is a **path-traversal** attack. We generate our own random filename and only borrow the extension. Validate the mimetype *and* the extension — clients can lie about either alone.

> WARNING — Pitfall: When a route uses `upload.single("image")`, the request is `multipart/form-data`, so text fields arrive as **strings** (`req.body.price` is `"199.99"`, not a number) and the file is on `req.file`. Your validators and service must coerce types (`Number(price)`).

### `src/services/product.service.js`

```js
// src/services/product.service.js
const fs = require("fs/promises");
const path = require("path");
const Product = require("../models/product.model");
const Category = require("../models/category.model");
const ApiError = require("../utils/ApiError");
const ApiFeatures = require("../utils/apiFeatures");

async function assertCategoryExists(categoryId) {
  const exists = await Category.exists({ _id: categoryId });
  if (!exists) throw ApiError.badRequest("Category does not exist");
}

async function createProduct(data, file) {
  await assertCategoryExists(data.category);
  const payload = {
    ...data,
    price: Number(data.price),
    stock: data.stock !== undefined ? Number(data.stock) : 0,
    image: file ? `/uploads/${file.filename}` : null,
  };
  return Product.create(payload);
}

async function listProducts(queryString) {
  // Build the data query through the feature pipeline...
  const features = new ApiFeatures(Product.find().populate("category", "name slug"), queryString)
    .filter()
    .search()
    .sort()
    .limitFields()
    .paginate();

  const products = await features.query;

  // ...and count matching docs (same filter, ignoring pagination) for accurate metadata.
  const countFeatures = new ApiFeatures(Product.find(), queryString).filter().search();
  const total = await Product.countDocuments(countFeatures.query.getFilter());

  const { page, limit } = features.meta;
  return {
    results: products.length,
    pagination: { total, page, limit, pages: Math.ceil(total / limit) },
    data: products,
  };
}

async function getProduct(id) {
  const product = await Product.findById(id).populate("category", "name slug");
  if (!product) throw ApiError.notFound("Product not found");
  return product;
}

async function updateProduct(id, data, file) {
  if (data.category) await assertCategoryExists(data.category);

  const product = await Product.findById(id);
  if (!product) throw ApiError.notFound("Product not found");

  // Coerce numeric fields coming from multipart strings.
  if (data.price !== undefined) data.price = Number(data.price);
  if (data.stock !== undefined) data.stock = Number(data.stock);

  // If a new image was uploaded, delete the old file (best-effort) and set the new path.
  if (file) {
    if (product.image) await safeUnlink(product.image);
    data.image = `/uploads/${file.filename}`;
  }

  Object.assign(product, data);
  await product.save(); // triggers validators + hooks
  return product;
}

async function deleteProduct(id) {
  const product = await Product.findByIdAndDelete(id);
  if (!product) throw ApiError.notFound("Product not found");
  if (product.image) await safeUnlink(product.image);
  return product;
}

// Delete an uploaded file by its public path; swallow "not found" so it never breaks the request.
async function safeUnlink(publicPath) {
  try {
    const abs = path.join(__dirname, "..", "..", publicPath.replace(/^\//, ""));
    await fs.unlink(abs);
  } catch (_err) {
    /* file already gone — ignore */
  }
}

module.exports = { createProduct, listProducts, getProduct, updateProduct, deleteProduct };
```

### `src/validators/product.validator.js`

```js
// src/validators/product.validator.js
const { body, param } = require("express-validator");

const idRule = [param("id").isMongoId().withMessage("Invalid product id")];

// Note: with multipart/form-data, numbers arrive as strings — validate accordingly.
const createRules = [
  body("name").trim().notEmpty().withMessage("Name is required").isLength({ max: 140 }),
  body("price").notEmpty().withMessage("Price is required").isFloat({ min: 0 }).withMessage("Price must be >= 0"),
  body("stock").optional().isInt({ min: 0 }).withMessage("Stock must be a non-negative integer"),
  body("category").notEmpty().withMessage("Category is required").isMongoId().withMessage("Invalid category id"),
  body("description").optional().trim().isLength({ max: 2000 }),
  body("brand").optional().trim().isLength({ max: 80 }),
];

const updateRules = [
  ...idRule,
  body("name").optional().trim().notEmpty().isLength({ max: 140 }),
  body("price").optional().isFloat({ min: 0 }),
  body("stock").optional().isInt({ min: 0 }),
  body("category").optional().isMongoId(),
  body("description").optional().trim().isLength({ max: 2000 }),
  body("brand").optional().trim().isLength({ max: 80 }),
];

module.exports = { idRule, createRules, updateRules };
```

### `src/controllers/product.controller.js`

```js
// src/controllers/product.controller.js
const productService = require("../services/product.service");
const catchAsync = require("../utils/catchAsync");

const create = catchAsync(async (req, res) => {
  // req.file is set by Multer (uploadProductImage) when an image was sent.
  const product = await productService.createProduct(req.body, req.file);
  res.status(201).json({ success: true, data: product });
});

const list = catchAsync(async (req, res) => {
  const result = await productService.listProducts(req.query);
  res.json({ success: true, ...result });
});

const getOne = catchAsync(async (req, res) => {
  const product = await productService.getProduct(req.params.id);
  res.json({ success: true, data: product });
});

const update = catchAsync(async (req, res) => {
  const product = await productService.updateProduct(req.params.id, req.body, req.file);
  res.json({ success: true, data: product });
});

const remove = catchAsync(async (req, res) => {
  await productService.deleteProduct(req.params.id);
  res.status(204).send();
});

module.exports = { create, list, getOne, update, remove };
```

### `src/routes/product.routes.js`

```js
// src/routes/product.routes.js
const express = require("express");
const productController = require("../controllers/product.controller");
const { authenticate, authorize } = require("../middleware/auth.middleware");
const { validate } = require("../middleware/validate.middleware");
const { uploadProductImage } = require("../middleware/upload.middleware");
const { idRule, createRules, updateRules } = require("../validators/product.validator");

const router = express.Router();

// Public reads (with full pagination/filter/sort/search on the list).
router.get("/", productController.list);
router.get("/:id", idRule, validate, productController.getOne);

// Admin-only writes. Multer runs BEFORE validators so req.body (text fields) is populated.
router.post(
  "/",
  authenticate,
  authorize("admin"),
  uploadProductImage, // parses multipart, fills req.body + req.file
  createRules,
  validate,
  productController.create
);

router.patch(
  "/:id",
  authenticate,
  authorize("admin"),
  uploadProductImage,
  updateRules,
  validate,
  productController.update
);

router.delete("/:id", authenticate, authorize("admin"), idRule, validate, productController.remove);

module.exports = router;
```

> WARNING — Pitfall: The Multer middleware must run **before** `express-validator` on upload routes. Multer is what parses `multipart/form-data` and populates `req.body`; if validators run first they see an empty body and reject everything.

---

## Step 8 — Cart

The cart is the staging area before checkout. One cart per user; the service always finds-or-creates the caller's cart so the controller never juggles "does a cart exist yet."

### `src/services/cart.service.js`

```js
// src/services/cart.service.js
const Cart = require("../models/cart.model");
const Product = require("../models/product.model");
const ApiError = require("../utils/ApiError");

// Always returns THIS user's cart, creating an empty one on first use.
async function getOrCreateCart(userId) {
  let cart = await Cart.findOne({ user: userId });
  if (!cart) cart = await Cart.create({ user: userId, items: [] });
  return cart;
}

// Return the cart with products populated (for display).
async function getCart(userId) {
  const cart = await getOrCreateCart(userId);
  return cart.populate("items.product", "name price image stock");
}

async function addItem(userId, productId, quantity) {
  const product = await Product.findById(productId);
  if (!product || !product.isActive) throw ApiError.notFound("Product not found");

  const cart = await getOrCreateCart(userId);
  const existing = cart.items.find((i) => i.product.toString() === productId);
  const newQty = (existing ? existing.quantity : 0) + quantity;

  // Don't let the cart exceed available stock.
  if (newQty > product.stock) {
    throw ApiError.badRequest(`Only ${product.stock} unit(s) in stock`);
  }

  if (existing) {
    existing.quantity = newQty;
    existing.priceSnapshot = product.price; // refresh snapshot to current price
  } else {
    cart.items.push({ product: productId, quantity, priceSnapshot: product.price });
  }
  await cart.save();
  return cart.populate("items.product", "name price image stock");
}

async function updateItem(userId, productId, quantity) {
  const cart = await getOrCreateCart(userId);
  const item = cart.items.find((i) => i.product.toString() === productId);
  if (!item) throw ApiError.notFound("Item not in cart");

  if (quantity <= 0) {
    // Quantity 0 means remove the line.
    cart.items = cart.items.filter((i) => i.product.toString() !== productId);
  } else {
    const product = await Product.findById(productId);
    if (!product) throw ApiError.notFound("Product not found");
    if (quantity > product.stock) throw ApiError.badRequest(`Only ${product.stock} unit(s) in stock`);
    item.quantity = quantity;
    item.priceSnapshot = product.price;
  }
  await cart.save();
  return cart.populate("items.product", "name price image stock");
}

async function removeItem(userId, productId) {
  const cart = await getOrCreateCart(userId);
  cart.items = cart.items.filter((i) => i.product.toString() !== productId);
  await cart.save();
  return cart.populate("items.product", "name price image stock");
}

async function clearCart(userId) {
  const cart = await getOrCreateCart(userId);
  cart.items = [];
  await cart.save();
  return cart;
}

module.exports = { getCart, addItem, updateItem, removeItem, clearCart };
```

### `src/validators/cart.validator.js`

```js
// src/validators/cart.validator.js
const { body, param } = require("express-validator");

const addRules = [
  body("productId").isMongoId().withMessage("Valid productId is required"),
  body("quantity").isInt({ min: 1 }).withMessage("Quantity must be a positive integer"),
];

const updateRules = [
  param("productId").isMongoId().withMessage("Invalid productId"),
  body("quantity").isInt({ min: 0 }).withMessage("Quantity must be 0 or more (0 removes the item)"),
];

const productIdParam = [param("productId").isMongoId().withMessage("Invalid productId")];

module.exports = { addRules, updateRules, productIdParam };
```

### `src/controllers/cart.controller.js`

```js
// src/controllers/cart.controller.js
const cartService = require("../services/cart.service");
const catchAsync = require("../utils/catchAsync");

const get = catchAsync(async (req, res) => {
  const cart = await cartService.getCart(req.user._id);
  res.json({ success: true, data: cart });
});

const add = catchAsync(async (req, res) => {
  const cart = await cartService.addItem(req.user._id, req.body.productId, req.body.quantity);
  res.status(201).json({ success: true, data: cart });
});

const update = catchAsync(async (req, res) => {
  const cart = await cartService.updateItem(req.user._id, req.params.productId, req.body.quantity);
  res.json({ success: true, data: cart });
});

const remove = catchAsync(async (req, res) => {
  const cart = await cartService.removeItem(req.user._id, req.params.productId);
  res.json({ success: true, data: cart });
});

const clear = catchAsync(async (req, res) => {
  await cartService.clearCart(req.user._id);
  res.status(204).send();
});

module.exports = { get, add, update, remove, clear };
```

### `src/routes/cart.routes.js`

```js
// src/routes/cart.routes.js
const express = require("express");
const cartController = require("../controllers/cart.controller");
const { authenticate } = require("../middleware/auth.middleware");
const { validate } = require("../middleware/validate.middleware");
const { addRules, updateRules, productIdParam } = require("../validators/cart.validator");

const router = express.Router();

// Every cart route requires a logged-in user; apply authenticate to the whole router.
router.use(authenticate);

router.get("/", cartController.get);
router.post("/items", addRules, validate, cartController.add);
router.patch("/items/:productId", updateRules, validate, cartController.update);
router.delete("/items/:productId", productIdParam, validate, cartController.remove);
router.delete("/", cartController.clear);

module.exports = router;
```

> TIP: `router.use(authenticate)` applies the guard to **every** route below it — cleaner than repeating `authenticate` on each line. Order still matters: middleware added before the routes runs for all of them.

---

## Step 9 — Orders (Checkout with a Transaction)

This is the keystone flow. "Create order from cart" must be **atomic**: validate stock, snapshot prices, decrement inventory, record the order, and clear the cart — all or nothing. We use a **Mongoose transaction** (a MongoDB multi-document transaction) so a failure midway rolls everything back.

> Note: Multi-document transactions require a **replica set** (or Atlas). A standalone `mongod` does not support them. Our `docker-compose.yml` starts Mongo as a single-node replica set precisely so transactions work locally. The service degrades gracefully if transactions are unavailable (see the comment).

### `src/services/order.service.js`

```js
// src/services/order.service.js
const mongoose = require("mongoose");
const Cart = require("../models/cart.model");
const Product = require("../models/product.model");
const Order = require("../models/order.model");
const ApiError = require("../utils/ApiError");

// Create an order from the user's current cart — atomically.
async function createOrderFromCart(userId, shippingAddress) {
  const cart = await Cart.findOne({ user: userId });
  if (!cart || cart.items.length === 0) {
    throw ApiError.badRequest("Your cart is empty");
  }

  const session = await mongoose.startSession();
  try {
    let order;
    await session.withTransaction(async () => {
      const orderItems = [];
      let total = 0;

      // Validate stock and build snapshotted line items.
      for (const item of cart.items) {
        const product = await Product.findById(item.product).session(session);
        if (!product || !product.isActive) {
          throw ApiError.badRequest(`Product ${item.product} is no longer available`);
        }
        if (product.stock < item.quantity) {
          throw ApiError.badRequest(`Insufficient stock for "${product.name}" (have ${product.stock})`);
        }

        // Decrement stock atomically with a guard so two concurrent checkouts can't oversell.
        const updated = await Product.findOneAndUpdate(
          { _id: product._id, stock: { $gte: item.quantity } }, // condition prevents oversell
          { $inc: { stock: -item.quantity } },
          { new: true, session }
        );
        if (!updated) {
          throw ApiError.conflict(`Stock changed for "${product.name}", please retry`);
        }

        const lineTotal = product.price * item.quantity;
        total += lineTotal;
        orderItems.push({
          product: product._id,
          name: product.name, // snapshot
          price: product.price, // snapshot
          quantity: item.quantity,
        });
      }

      // Create the order (must pass session and use array form within a transaction).
      const created = await Order.create(
        [
          {
            user: userId,
            items: orderItems,
            totalAmount: Number(total.toFixed(2)),
            status: "pending",
            shippingAddress,
          },
        ],
        { session }
      );
      order = created[0];

      // Empty the cart as part of the same transaction.
      cart.items = [];
      await cart.save({ session });
    });

    return order;
  } finally {
    await session.endSession();
  }
}

async function listMyOrders(userId) {
  return Order.find({ user: userId }).sort("-createdAt");
}

async function getOrder(orderId, requester) {
  const order = await Order.findById(orderId);
  if (!order) throw ApiError.notFound("Order not found");
  // Customers can only see their own orders; admins can see any.
  if (requester.role !== "admin" && order.user.toString() !== requester._id.toString()) {
    throw ApiError.forbidden("You cannot view this order");
  }
  return order;
}

// Admin: list all orders (could be paginated like products).
async function listAllOrders() {
  return Order.find().populate("user", "name email").sort("-createdAt");
}

// Admin: update order status, with a few business rules.
async function updateStatus(orderId, status) {
  const order = await Order.findById(orderId);
  if (!order) throw ApiError.notFound("Order not found");

  if (order.status === "cancelled") {
    throw ApiError.conflict("Cannot change status of a cancelled order");
  }

  order.status = status;
  if (status === "paid") order.paidAt = new Date();
  if (status === "delivered") order.deliveredAt = new Date();
  await order.save();
  return order;
}

// Customer/Admin: cancel a pending order and restock its items.
async function cancelOrder(orderId, requester) {
  const session = await mongoose.startSession();
  try {
    let order;
    await session.withTransaction(async () => {
      order = await Order.findById(orderId).session(session);
      if (!order) throw ApiError.notFound("Order not found");
      if (requester.role !== "admin" && order.user.toString() !== requester._id.toString()) {
        throw ApiError.forbidden("You cannot cancel this order");
      }
      if (!["pending", "paid"].includes(order.status)) {
        throw ApiError.conflict(`Cannot cancel an order that is ${order.status}`);
      }

      // Restock each item.
      for (const item of order.items) {
        await Product.updateOne({ _id: item.product }, { $inc: { stock: item.quantity } }, { session });
      }
      order.status = "cancelled";
      await order.save({ session });
    });
    return order;
  } finally {
    await session.endSession();
  }
}

module.exports = {
  createOrderFromCart,
  listMyOrders,
  getOrder,
  listAllOrders,
  updateStatus,
  cancelOrder,
};
```

> BEST PRACTICE: The conditional update `findOneAndUpdate({ _id, stock: { $gte: qty } }, { $inc: { stock: -qty } })` is the crux of preventing **oversell** under concurrency. Two simultaneous checkouts for the last unit cannot both succeed: the second one's condition fails and `updated` is `null`. This atomic compare-and-decrement is a classic interview answer for "how do you avoid race conditions on inventory."

> WARNING — Pitfall: Inside a transaction, `Model.create([...docs], { session })` must take an **array** and a `session` option, and every query in the transaction must `.session(session)`. A query that forgets the session runs outside the transaction and won't be rolled back.

### `src/validators/order.validator.js`

```js
// src/validators/order.validator.js
const { body, param } = require("express-validator");
const { ORDER_STATUSES } = require("../models/order.model");

const createRules = [
  body("shippingAddress.fullName").trim().notEmpty().withMessage("Full name is required"),
  body("shippingAddress.line1").trim().notEmpty().withMessage("Address line is required"),
  body("shippingAddress.city").trim().notEmpty().withMessage("City is required"),
  body("shippingAddress.postalCode").trim().notEmpty().withMessage("Postal code is required"),
  body("shippingAddress.country").trim().notEmpty().withMessage("Country is required"),
];

const idRule = [param("id").isMongoId().withMessage("Invalid order id")];

const statusRules = [
  ...idRule,
  body("status")
    .isIn(ORDER_STATUSES)
    .withMessage(`Status must be one of: ${ORDER_STATUSES.join(", ")}`),
];

module.exports = { createRules, idRule, statusRules };
```

### `src/controllers/order.controller.js`

```js
// src/controllers/order.controller.js
const orderService = require("../services/order.service");
const catchAsync = require("../utils/catchAsync");

const create = catchAsync(async (req, res) => {
  const order = await orderService.createOrderFromCart(req.user._id, req.body.shippingAddress);
  res.status(201).json({ success: true, data: order });
});

const listMine = catchAsync(async (req, res) => {
  const orders = await orderService.listMyOrders(req.user._id);
  res.json({ success: true, results: orders.length, data: orders });
});

const getOne = catchAsync(async (req, res) => {
  const order = await orderService.getOrder(req.params.id, req.user);
  res.json({ success: true, data: order });
});

const listAll = catchAsync(async (_req, res) => {
  const orders = await orderService.listAllOrders();
  res.json({ success: true, results: orders.length, data: orders });
});

const updateStatus = catchAsync(async (req, res) => {
  const order = await orderService.updateStatus(req.params.id, req.body.status);
  res.json({ success: true, data: order });
});

const cancel = catchAsync(async (req, res) => {
  const order = await orderService.cancelOrder(req.params.id, req.user);
  res.json({ success: true, data: order });
});

module.exports = { create, listMine, getOne, listAll, updateStatus, cancel };
```

### `src/routes/order.routes.js`

```js
// src/routes/order.routes.js
const express = require("express");
const orderController = require("../controllers/order.controller");
const { authenticate, authorize } = require("../middleware/auth.middleware");
const { validate } = require("../middleware/validate.middleware");
const { createRules, idRule, statusRules } = require("../validators/order.validator");

const router = express.Router();

router.use(authenticate); // all order routes require auth

// Customer endpoints.
router.post("/", createRules, validate, orderController.create); // checkout from cart
router.get("/my", orderController.listMine);
router.get("/:id", idRule, validate, orderController.getOne);
router.patch("/:id/cancel", idRule, validate, orderController.cancel);

// Admin endpoints.
router.get("/", authorize("admin"), orderController.listAll);
router.patch("/:id/status", authorize("admin"), statusRules, validate, orderController.updateStatus);

module.exports = router;
```

> WARNING — Pitfall: Route order matters. `GET /my` must be declared **before** `GET /:id`, or Express will treat `my` as an `:id` value and try to look up an order with id `"my"`. Static segments before dynamic ones.

### `src/routes/index.js` — mount everything

```js
// src/routes/index.js
const express = require("express");

const authRoutes = require("./auth.routes");
const categoryRoutes = require("./category.routes");
const productRoutes = require("./product.routes");
const cartRoutes = require("./cart.routes");
const orderRoutes = require("./order.routes");

const router = express.Router();

router.use("/auth", authRoutes);
router.use("/categories", categoryRoutes);
router.use("/products", productRoutes);
router.use("/cart", cartRoutes);
router.use("/orders", orderRoutes);

module.exports = router;
```

All of these mount under `/api/v1` (see `app.js`), so the full paths are `/api/v1/auth/login`, `/api/v1/products`, etc.

---

## Step 10 — Swagger / OpenAPI Docs

`swagger-jsdoc` turns JSDoc-style comments into an OpenAPI spec; `swagger-ui-express` serves an interactive UI. We define a base spec plus reusable schemas and a Bearer security scheme; per-route docs can be added as JSDoc `@openapi` comments in the route files.

### `src/config/swagger.js`

```js
// src/config/swagger.js
const swaggerJSDoc = require("swagger-jsdoc");
const path = require("path");

const options = {
  definition: {
    openapi: "3.0.3",
    info: {
      title: "E-Commerce REST API",
      version: "1.0.0",
      description: "Capstone e-commerce backend: auth, products, cart, orders.",
    },
    servers: [{ url: "/api/v1", description: "API v1" }],
    components: {
      securitySchemes: {
        bearerAuth: { type: "http", scheme: "bearer", bearerFormat: "JWT" },
      },
      schemas: {
        Product: {
          type: "object",
          properties: {
            _id: { type: "string" },
            name: { type: "string" },
            description: { type: "string" },
            price: { type: "number" },
            stock: { type: "integer" },
            category: { type: "string" },
            image: { type: "string", nullable: true },
            brand: { type: "string" },
          },
        },
        Error: {
          type: "object",
          properties: {
            success: { type: "boolean", example: false },
            message: { type: "string" },
            details: { type: "array", items: { type: "object" } },
          },
        },
      },
    },
    security: [{ bearerAuth: [] }], // default: most routes need a token
  },
  // Scan route files for @openapi JSDoc blocks (add them incrementally).
  apis: [path.join(__dirname, "..", "routes", "*.js")],
};

module.exports = swaggerJSDoc(options);
```

### Example route annotation

Add a block like this above a handler in `product.routes.js` to document it (optional but impressive in a portfolio):

```js
/**
 * @openapi
 * /products:
 *   get:
 *     summary: List products (paginated, filterable, searchable)
 *     security: []          # public, no auth
 *     parameters:
 *       - in: query
 *         name: page
 *         schema: { type: integer, default: 1 }
 *       - in: query
 *         name: limit
 *         schema: { type: integer, default: 12 }
 *       - in: query
 *         name: search
 *         schema: { type: string }
 *       - in: query
 *         name: sort
 *         schema: { type: string, example: "-price,createdAt" }
 *     responses:
 *       200:
 *         description: A page of products
 */
router.get("/", productController.list);
```

Once running, open **`http://localhost:5000/api-docs`** to browse and try endpoints. Click **Authorize**, paste a token (`Bearer` is added for you), and protected routes become callable from the UI.

---

## Step 11 — Tests (Jest + Supertest + in-memory Mongo)

We run integration tests against a real-but-ephemeral MongoDB via `mongodb-memory-server`. Tests import `app` (never start a port) and hit it with Supertest.

### `jest.config.js`

```js
// jest.config.js
module.exports = {
  testEnvironment: "node",
  setupFilesAfterEnv: ["<rootDir>/tests/setup.js"],
  testMatch: ["<rootDir>/tests/**/*.test.js"],
  coveragePathIgnorePatterns: ["/node_modules/"],
};
```

### `tests/setup.js`

```js
// tests/setup.js
// Boots an in-memory MongoDB once for the whole run; cleans collections between tests.
const mongoose = require("mongoose");
const { MongoMemoryReplSet } = require("mongodb-memory-server");

let replset;

beforeAll(async () => {
  // A replica set so transactions (used in checkout) work in tests too.
  replset = await MongoMemoryReplSet.create({ replSet: { count: 1 } });
  const uri = replset.getUri();
  await mongoose.connect(uri);
});

afterEach(async () => {
  // Wipe all collections so each test starts clean.
  const collections = mongoose.connection.collections;
  for (const key of Object.keys(collections)) {
    await collections[key].deleteMany({});
  }
});

afterAll(async () => {
  await mongoose.disconnect();
  if (replset) await replset.stop();
});
```

> WARNING — Pitfall: We use `MongoMemoryReplSet` (not the plain `MongoMemoryServer`) because the checkout flow uses transactions, which require a replica set. With a standalone in-memory server those tests would throw "Transaction numbers are only allowed on a replica set member."

### `tests/auth.test.js`

```js
// tests/auth.test.js
const request = require("supertest");
const app = require("../src/app");

describe("Auth", () => {
  const creds = { name: "Ada", email: "ada@example.com", password: "secret123" };

  it("registers a new user and returns a token", async () => {
    const res = await request(app).post("/api/v1/auth/register").send(creds);
    expect(res.statusCode).toBe(201);
    expect(res.body.success).toBe(true);
    expect(res.body.data.token).toBeDefined();
    expect(res.body.data.user.password).toBeUndefined(); // never leak the hash
    expect(res.body.data.user.role).toBe("customer");
  });

  it("rejects duplicate emails", async () => {
    await request(app).post("/api/v1/auth/register").send(creds);
    const res = await request(app).post("/api/v1/auth/register").send(creds);
    expect(res.statusCode).toBe(409);
  });

  it("rejects weak passwords with 400 and field details", async () => {
    const res = await request(app)
      .post("/api/v1/auth/register")
      .send({ ...creds, password: "123" });
    expect(res.statusCode).toBe(400);
    expect(res.body.details.some((d) => d.field === "password")).toBe(true);
  });

  it("logs in and returns a token", async () => {
    await request(app).post("/api/v1/auth/register").send(creds);
    const res = await request(app)
      .post("/api/v1/auth/login")
      .send({ email: creds.email, password: creds.password });
    expect(res.statusCode).toBe(200);
    expect(res.body.data.token).toBeDefined();
  });

  it("rejects bad credentials with a generic 401", async () => {
    await request(app).post("/api/v1/auth/register").send(creds);
    const res = await request(app)
      .post("/api/v1/auth/login")
      .send({ email: creds.email, password: "wrongpass" });
    expect(res.statusCode).toBe(401);
  });
});
```

### `tests/product.test.js`

```js
// tests/product.test.js
const request = require("supertest");
const app = require("../src/app");
const Category = require("../src/models/category.model");
const User = require("../src/models/user.model");
const { signToken } = require("../src/utils/token");

// Helper: create an admin directly and return a valid Bearer token.
async function adminToken() {
  const admin = await User.create({
    name: "Admin",
    email: "admin@example.com",
    password: "secret123",
    role: "admin",
  });
  return signToken({ sub: admin._id.toString(), role: "admin" });
}

describe("Products", () => {
  let token;
  let categoryId;

  beforeEach(async () => {
    token = await adminToken();
    const cat = await Category.create({ name: "Electronics" });
    categoryId = cat._id.toString();
  });

  it("blocks product creation without a token", async () => {
    const res = await request(app)
      .post("/api/v1/products")
      .field("name", "Phone")
      .field("price", "199")
      .field("category", categoryId);
    expect(res.statusCode).toBe(401);
  });

  it("creates a product as admin (multipart fields)", async () => {
    const res = await request(app)
      .post("/api/v1/products")
      .set("Authorization", `Bearer ${token}`)
      .field("name", "Phone")
      .field("price", "199.99")
      .field("stock", "5")
      .field("category", categoryId);
    expect(res.statusCode).toBe(201);
    expect(res.body.data.name).toBe("Phone");
    expect(res.body.data.price).toBe(199.99);
  });

  it("lists products with pagination metadata", async () => {
    await request(app)
      .post("/api/v1/products")
      .set("Authorization", `Bearer ${token}`)
      .field("name", "Phone")
      .field("price", "199")
      .field("category", categoryId);

    const res = await request(app).get("/api/v1/products?limit=10&page=1");
    expect(res.statusCode).toBe(200);
    expect(res.body.pagination).toMatchObject({ page: 1, limit: 10, total: 1, pages: 1 });
  });

  it("validates product input (missing price)", async () => {
    const res = await request(app)
      .post("/api/v1/products")
      .set("Authorization", `Bearer ${token}`)
      .field("name", "NoPrice")
      .field("category", categoryId);
    expect(res.statusCode).toBe(400);
  });
});
```

Run them:

```bash
npm test
```

> TIP: Tests that import `app` instead of `server` are fast and parallel-friendly because they never bind a TCP port — Supertest spins up an ephemeral server per request internally. This is exactly why we split `app.js` from `server.js`.

---

## Step 12 — Docker & docker-compose

We containerize the API and run MongoDB as a single-node replica set (so transactions work).

### `Dockerfile`

```dockerfile
# Dockerfile — multi-stage-ish single image for a Node API.
FROM node:22-alpine

# Create app dir and a non-root user for safety.
WORKDIR /usr/src/app

# Install deps first (better layer caching: only re-runs when lockfile changes).
COPY package*.json ./
RUN npm ci --omit=dev

# Copy source.
COPY . .

# Ensure the uploads dir exists and is writable.
RUN mkdir -p uploads && chown -R node:node /usr/src/app

USER node

EXPOSE 5000

# Use the production start script.
CMD ["node", "src/server.js"]
```

> BEST PRACTICE: Run containers as a **non-root user** (`USER node`). If the process is ever compromised, the blast radius is limited. Also `npm ci --omit=dev` produces a smaller, reproducible image with no dev tooling shipped to production.

### `docker-compose.yml`

```yaml
# docker-compose.yml — API + MongoDB (single-node replica set for transactions).
services:
  mongo:
    image: mongo:7
    container_name: ecommerce-mongo
    # Start as a replica set named "rs0" so Mongoose transactions work.
    command: ["--replSet", "rs0", "--bind_ip_all"]
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db
    healthcheck:
      # Initialize the replica set once, then report healthy.
      test: >
        mongosh --quiet --eval "
          try { rs.status() }
          catch (e) { rs.initiate({_id:'rs0', members:[{_id:0, host:'mongo:27017'}]}) }
        "
      interval: 10s
      timeout: 10s
      retries: 10
      start_period: 20s

  api:
    build: .
    container_name: ecommerce-api
    depends_on:
      mongo:
        condition: service_healthy
    environment:
      NODE_ENV: production
      PORT: 5000
      # Use the replica-set-aware URI so transactions are supported.
      MONGODB_URI: mongodb://mongo:27017/ecommerce?replicaSet=rs0
      JWT_SECRET: change-me-in-prod
      JWT_EXPIRES_IN: 1d
      CORS_ORIGIN: "*"
      RATE_LIMIT_WINDOW_MS: 900000
      RATE_LIMIT_MAX: 100
    ports:
      - "5000:5000"
    volumes:
      # Persist uploaded images across container restarts.
      - uploads-data:/usr/src/app/uploads

volumes:
  mongo-data:
  uploads-data:
```

Bring it all up:

```bash
docker compose up --build
# API:  http://localhost:5000
# Docs: http://localhost:5000/api-docs
```

> WARNING — Pitfall: A plain `mongo` container does **not** support transactions. The `--replSet rs0` command plus the healthcheck's `rs.initiate(...)` turns it into a single-node replica set, and the API's URI includes `?replicaSet=rs0`. Without all three, the checkout transaction throws.

> TIP: `depends_on` with `condition: service_healthy` makes the API wait until Mongo's healthcheck (which also initializes the replica set) passes — no more "connection refused" on first boot.

---

## How to Run & Test

### Run locally (without Docker)

```bash
# 1. Make sure MongoDB is running locally as a replica set, OR use the Dockerized Mongo:
docker compose up mongo        # starts just Mongo (replica set) on :27017

# 2. Configure env (point MONGODB_URI at the replica set):
#    MONGODB_URI=mongodb://localhost:27017/ecommerce?replicaSet=rs0
# 3. Start the API in watch mode:
npm run dev
```

You should see:

```
Server running in development mode on port 5000
API docs at http://localhost:5000/api-docs
```

### Sample requests (curl)

Set a base URL and we'll capture the token into a shell variable.

```bash
BASE=http://localhost:5000/api/v1
```

**1. Register a customer**

```bash
curl -s -X POST $BASE/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name":"Ada Lovelace","email":"ada@example.com","password":"secret123"}'
```

Sample response:

```json
{
  "success": true,
  "data": {
    "user": {
      "_id": "6650f1a2c3d4e5f600000001",
      "name": "Ada Lovelace",
      "email": "ada@example.com",
      "role": "customer",
      "createdAt": "2026-06-19T10:00:00.000Z",
      "updatedAt": "2026-06-19T10:00:00.000Z"
    },
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

**2. Log in and capture the token**

```bash
TOKEN=$(curl -s -X POST $BASE/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"ada@example.com","password":"secret123"}' \
  | node -e "process.stdin.on('data',d=>console.log(JSON.parse(d).data.token))")
echo "$TOKEN"
```

**3. Create an admin** (for catalog management). The simplest path is to register, then flip the role directly in the DB once (admins shouldn't be self-registerable). Using `mongosh`:

```bash
# In mongosh against the ecommerce DB:
db.users.updateOne({ email: "admin@example.com" }, { $set: { role: "admin" } })
```

Then log in as that admin and capture `ADMIN_TOKEN` the same way as step 2.

**4. Create a category (admin)**

```bash
curl -s -X POST $BASE/categories \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"Electronics","description":"Phones, laptops, and gadgets"}'
```

**5. Create a product with an image (admin, multipart)**

Note this is `multipart/form-data`, so we use `-F`, not `-d`:

```bash
curl -s -X POST $BASE/products \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -F "name=Smartphone X" \
  -F "price=699.99" \
  -F "stock=10" \
  -F "brand=Acme" \
  -F "category=<CATEGORY_ID>" \
  -F "image=@/path/to/phone.jpg"
```

Sample response (truncated):

```json
{
  "success": true,
  "data": {
    "_id": "6650f2b0c3d4e5f600000010",
    "name": "Smartphone X",
    "price": 699.99,
    "stock": 10,
    "image": "/uploads/1718800000000-3f9a2b.jpg",
    "category": "6650f200c3d4e5f600000005"
  }
}
```

The image is now reachable at `http://localhost:5000/uploads/1718800000000-3f9a2b.jpg`.

**6. List products with query features**

```bash
# Page 1, 5 per page, only electronics, price 100–800, newest first, name+price only:
curl -s "$BASE/products?page=1&limit=5&category=<CATEGORY_ID>&price[gte]=100&price[lte]=800&sort=-createdAt&fields=name,price"

# Full-text-ish search:
curl -s "$BASE/products?search=smartphone"
```

Sample list response shape:

```json
{
  "success": true,
  "results": 1,
  "pagination": { "total": 1, "page": 1, "limit": 5, "pages": 1 },
  "data": [{ "_id": "6650f2b0...", "name": "Smartphone X", "price": 699.99 }]
}
```

**7. Add to cart (customer)**

```bash
curl -s -X POST $BASE/cart/items \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"productId":"<PRODUCT_ID>","quantity":2}'
```

**8. Checkout — create an order from the cart (customer)**

```bash
curl -s -X POST $BASE/orders \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "shippingAddress": {
      "fullName": "Ada Lovelace",
      "line1": "12 Analytical Engine Rd",
      "city": "London",
      "postalCode": "EC1A 1BB",
      "country": "UK"
    }
  }'
```

Sample response (truncated):

```json
{
  "success": true,
  "data": {
    "_id": "6650f3c0c3d4e5f600000020",
    "items": [{ "product": "6650f2b0...", "name": "Smartphone X", "price": 699.99, "quantity": 2 }],
    "totalAmount": 1399.98,
    "status": "pending"
  }
}
```

After this, the product's `stock` is `8` and the cart is empty.

**9. Admin updates order status**

```bash
curl -s -X PATCH $BASE/orders/<ORDER_ID>/status \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"status":"shipped"}'
```

### Postman tips

- Create an **environment** with `baseUrl`, `token`, and `adminToken` variables.
- In the **login** request's *Tests* tab, save the token automatically:
  ```js
  pm.environment.set("token", pm.response.json().data.token);
  ```
- For protected requests, set Authorization type to **Bearer Token** and use `{{token}}`.
- For product create, choose **Body → form-data**, add text fields plus a `image` field of type **File**.

---

## Code Walkthrough / Key Concepts Recap

A guided tour of *why* the pieces fit together — this is also your interview script.

### The request lifecycle, end to end

Take `POST /api/v1/products` with an image:

1. **`server.js`** has already connected Mongo and started `app.listen`.
2. **`app.js`** runs global middleware in order: `helmet` (headers) → `cors` → body parsers → `sanitizeRequest` (NoSQL-injection guard) → `morgan` (logs the request) → `rateLimit`.
3. The request matches `/api/v1` → `productRoutes` → the `POST /` handler chain.
4. **`authenticate`** reads the `Bearer` token, verifies it, loads `req.user`. **`authorize("admin")`** checks the role.
5. **`uploadProductImage`** (Multer) parses `multipart/form-data`, writes the file to `/uploads`, and fills `req.body` (text) and `req.file`.
6. **`createRules` + `validate`** run express-validator; on failure they short-circuit to the error handler with a 400 and field details.
7. **`product.controller.create`** is thin: it pulls `req.body`/`req.file` and calls the service.
8. **`product.service.createProduct`** holds the logic: verify the category exists, coerce numbers, build the doc, persist it.
9. The controller sends `201` with a consistent `{ success, data }` envelope.
10. If anything threw, **`catchAsync`** forwarded it to the **central error handler**, which normalized it (CastError, ValidationError, duplicate key, Multer, JWT…) and returned a clean JSON error.

### Why the layers

| Layer | Responsibility | Knows about HTTP? | Knows about the DB? |
| --- | --- | --- | --- |
| Route | URL + method + which middleware run | Yes | No |
| Controller | Read `req`, call a service, write `res` | Yes | No |
| Service | Business logic & orchestration | No | Yes (via models) |
| Model | Schema, validation, persistence | No | Yes |

The payoff: business logic (services) is **testable without HTTP**, controllers are trivial, and swapping the web framework or the persistence layer touches only one ring.

### Concepts that show up in interviews

- **Stateless JWT auth**: token carries identity + role; server stores no session. We still re-load the user to honor deletions. Trade-off vs. sessions: scalability (no shared session store) vs. revocation difficulty (mitigated by short expiry + re-fetch).
- **RBAC** via `authorize(...roles)` — a higher-order middleware. Composes cleanly: `authenticate, authorize("admin")`.
- **NoSQL injection** and why `{ "$gt": "" }` in a body is dangerous; sanitization + validation as defense-in-depth.
- **Idempotency & atomicity** in checkout: transactions + a conditional `$inc` to prevent oversell under concurrency.
- **Price snapshotting** in carts/orders so historical records don't mutate with the catalog.
- **Pagination strategies**: offset (`skip/limit`) vs. cursor; capping `limit`.
- **Centralized error handling** and the operational-vs-programmer error distinction (`isOperational`).
- **Graceful shutdown** on `SIGTERM` for zero-downtime container restarts.
- **The `app`/`server` split** enabling fast Supertest tests.

### Express v4 vs v5 notes that mattered here

| Topic | Express 4 | Express 5 (used here) |
| --- | --- | --- |
| Async errors | Must `try/catch` or wrap; unhandled rejections crash | Rejected promises auto-forwarded to `next` (we still wrap for clarity/portability) |
| `req.query` | Writable object | Read-only getter → broke naive `mongo-sanitize`; we wrote a safe wrapper |
| Path syntax | `*` wildcards, optional `?` | Stricter path-to-regexp; named wildcards (`/*splat`) |
| `res.status(x).send()` | Lenient | Stricter; we use explicit JSON envelopes |

---

## Extensions & Challenges

Pick a few to deepen the project and give yourself more to talk about.

1. **Stripe payments.** Add a `POST /orders/:id/pay` that creates a Stripe **PaymentIntent**, returns the `client_secret`, and a **webhook** endpoint (`/webhooks/stripe`) that flips the order to `paid` on `payment_intent.succeeded`. Verify the webhook signature; make the handler idempotent (a webhook can fire twice).
2. **Product reviews.** A `Review` model (`product`, `user`, `rating 1–5`, `comment`), one review per user per *purchased* product. Recompute the product's average rating with a `post('save')` hook or an aggregation. Add `?sort=-rating`.
3. **Wishlists.** A per-user list of product references with add/remove endpoints and a "move to cart" action.
4. **Refresh tokens.** Add short-lived access tokens + long-lived refresh tokens stored hashed in the DB, with rotation and a logout/`revoke` endpoint.
5. **Email.** Send an order-confirmation email (Nodemailer + a dev SMTP like Mailhog) from a **background job** so the request isn't blocked.
6. **Caching.** Cache the product list/detail in Redis with a short TTL; invalidate on write. Measure the latency difference.
7. **Image processing.** Pipe Multer through **sharp** to resize/compress uploads and strip EXIF, or upload to S3 instead of local disk.
8. **Coupons/discounts.** A `Coupon` model applied at checkout with validation (expiry, min spend, usage limits).
9. **Full CI.** A GitHub Actions workflow that runs `npm test` (with the in-memory replica set) and builds the Docker image on every push.
10. **Observability.** Add a request id (`X-Request-Id`) per request and include it in every log line and error response for traceability.

---

## Common Errors & Fixes

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `Transaction numbers are only allowed on a replica set member` | Mongo is standalone, not a replica set | Run Mongo with `--replSet rs0` and `rs.initiate()`; use a `?replicaSet=rs0` URI (our compose does this). For tests use `MongoMemoryReplSet`. |
| `Cannot set property query of #<IncomingMessage>` | `express-mongo-sanitize` reassigning `req.query` on Express 5 | Use the `sanitizeRequest` wrapper that mutates keys in place. |
| `data and salt arguments required` from bcrypt on login | `password` not selected (it's `select:false`) | Use `User.findOne({ email }).select("+password")`. |
| `req.body` is empty on product create | Multer didn't run, or it ran after validators | Put `uploadProductImage` **before** the validators; ensure the client sends `multipart/form-data`. |
| Numbers stored as strings / validation passes wrongly | multipart fields are strings | Coerce with `Number(...)` in the service; validate with `isFloat`/`isInt`. |
| `404 Route not found` for `GET /orders/my` | `/:id` declared before `/my` | Declare static routes before dynamic ones. |
| `CastError: Cast to ObjectId failed` | A bad id in the URL | Already normalized to 400 by the error handler; add `isMongoId()` validators for clearer messages. |
| `E11000 duplicate key error` | Unique field collision (email, category name/slug) | Returned as 409 by the error handler; surface a friendly message client-side. |
| Errors return Express's HTML page, not JSON | Error handler missing/misplaced or has 3 args | It must be registered **last** and have **four** params `(err, req, res, next)`. |
| JWT works locally, 401 in Docker | Different `JWT_SECRET` between environments | Set the same `JWT_SECRET` everywhere; tokens signed with one secret can't be verified with another. |
| Uploaded images 404 after container restart | uploads dir not persisted | Mount a volume for `/usr/src/app/uploads` (our compose does). |
| `node-gyp` build failure in Docker | Using native `bcrypt` | Switch to `bcryptjs` (we do). |
| Rate limiter blocks everyone behind a proxy | Wrong client IP detected | Set `app.set('trust proxy', 1)` (we do) so the real client IP is used. |
| Mongoose `this` is undefined in a hook | Used an arrow function | Use `function () {}` for hooks, methods, and virtual getters. |

---

You now have a complete, runnable, production-shaped e-commerce API: layered architecture, JWT auth with roles, a rich product query layer, safe uploads, a transactional checkout that can't oversell, security middleware, structured logging, tests, Swagger docs, and a one-command Docker setup. Push it to GitHub, pin the README with a quick-start and an architecture diagram, and you have a portfolio project you can **build, run, and defend** — exactly what the rest of this course was preparing you for.
