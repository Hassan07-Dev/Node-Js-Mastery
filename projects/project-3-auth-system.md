# Project 3: Authentication & Authorization System

This is the project where authentication stops being a topic you've *read about* and becomes a system you've *built, broken, fixed, and can defend in an interview*. You already studied the theory in Module 21 (Authentication) and Module 22 (Authorization & Web Security). Now we assemble all of it into a single, production-shaped Express + Mongoose service.

We are not building a toy that "logs you in." We are building the kind of auth foundation you would actually drop into a real product: registration with correctly-hashed passwords, login that issues a short-lived JWT **access token** plus a long-lived **refresh token** in an `httpOnly` cookie, middleware that protects routes and loads `req.user`, role-based authorization, a password-reset-by-email flow, rate limiting, secure cookie flags, and a clean centralized error handler. Then we go further with the extensions that separate juniors from seniors: email verification, account lockout after repeated failures, and refresh-token **rotation** with reuse detection.

> **Version note:** This guide targets **Node.js current LTS (v20+/v22+)**, **Express 5** (the current major — Express 4 differences are flagged where they matter), **Mongoose 8**, and plain JavaScript with **CommonJS `require`** by default. An ES Modules variant is shown at the end. Every library used (`bcrypt`, `jsonwebtoken`, `cookie-parser`, `express-rate-limit`, `nodemailer`, `validator`, `dotenv`, `helmet`) is current and actively maintained.

---

## Overview

By the end of this project you will have a runnable HTTP API with the following endpoints:

| Method | Path | Auth required | What it does |
| --- | --- | --- | --- |
| `POST` | `/api/v1/auth/register` | No | Create a user, hash the password, return tokens |
| `POST` | `/api/v1/auth/login` | No | Verify credentials, return access token + set refresh cookie |
| `POST` | `/api/v1/auth/refresh` | Refresh cookie | Issue a new access token (and rotate the refresh token) |
| `POST` | `/api/v1/auth/logout` | No (cookie) | Clear the refresh cookie and revoke the refresh token |
| `GET` | `/api/v1/auth/me` | Access token | Return the currently logged-in user |
| `POST` | `/api/v1/auth/forgot-password` | No | Email a password-reset link |
| `PATCH` | `/api/v1/auth/reset-password/:token` | No | Set a new password using the reset token |
| `PATCH` | `/api/v1/auth/update-password` | Access token | Change password while logged in |
| `GET` | `/api/v1/users` | Access token + `admin` | Admin-only: list all users |
| `DELETE` | `/api/v1/users/:id` | Access token + `admin` | Admin-only: delete a user |

Concretely, the request flow looks like this:

```text
                       ┌─────────────────────────────────────────────┐
  Client (curl/Postman)│                  Express App                  │
        │              │                                              │
  POST /register ──────┼─▶ validate ─▶ hash password ─▶ save User      │
        │              │        │                                      │
        │◀────────────┼─ 201 { accessToken } + Set-Cookie: refresh… │
        │              │                                              │
  POST /login ─────────┼─▶ find user (+password) ─▶ bcrypt.compare    │
        │◀────────────┼─ 200 { accessToken } + Set-Cookie: refresh…  │
        │              │                                              │
  GET /me  ────────────┼─▶ protect() ─▶ verify JWT ─▶ load req.user   │
   Authorization:      │                                              │
   Bearer <access>     │                                              │
        │◀────────────┼─ 200 { user }                                │
        │              │                                              │
  DELETE /users/:id ───┼─▶ protect() ─▶ restrictTo('admin') ─▶ delete │
        │◀────────────┼─ 204 (or 403 if not admin)                   │
        └─────────────┴──────────────────────────────────────────────┘
```

Under the hood we will use the **layered architecture** you practiced in Project 2: thin **routes**, **controllers** that handle HTTP, **services** that hold business logic, **models** for data, plus shared **middleware**, **utils**, and **config**. This separation is not ceremony — it is what makes auth code testable and keeps security logic in one place instead of scattered across handlers.

---

## What You'll Practice

- **Password hashing with bcrypt** — salts, cost factor, and a Mongoose `pre('save')` hook that hashes only when the password changes.
- **JWT issuance & verification** — `jwt.sign`/`jwt.verify`, short-lived access tokens, and the access-vs-refresh token split.
- **Refresh tokens in `httpOnly` cookies** — why they live in cookies, the secure cookie flags, and **rotation with reuse detection**.
- **`protect` middleware** — extracting a Bearer token, verifying it, loading `req.user`, and handling the "password changed after token issued" edge case.
- **Role-based authorization** — a reusable `restrictTo('admin', ...)` middleware and the 401-vs-403 distinction.
- **Secure error handling** — never leaking *which* field failed login, a custom `AppError`, a `catchAsync` wrapper, and a centralized error middleware that behaves differently in dev vs prod.
- **Rate limiting** — throttling login/auth endpoints to blunt brute-force and credential-stuffing attacks.
- **Password reset by email** — generating a cryptographically random reset token, storing only its **hash**, expiring it, and emailing a link with **Nodemailer** (using a dev inbox).
- **Environment configuration & secrets** — a validated config layer and a `.env.example` you can hand to a teammate.
- **Production hardening** — `helmet`, body-size limits, trust proxy, graceful shutdown, and the extensions (email verification, account lockout) that real systems ship.

---

## Prerequisites & Setup

You need:

- **Node.js LTS** — verify with `node --version` (expect v20+ or v22+).
- **MongoDB** — either a local `mongod` or a free **MongoDB Atlas** cluster. A connection string is all you need.
- **An API client** — `curl` (used throughout) or Postman/Thunder Client.

### Create the project and install dependencies

```bash
# 1. Make the project folder and enter it
mkdir auth-system && cd auth-system

# 2. Initialise package.json
npm init -y

# 3. Runtime dependencies
npm install express mongoose bcrypt jsonwebtoken cookie-parser \
            express-rate-limit nodemailer validator dotenv helmet

# 4. Dev dependency: auto-restart on file changes (Node 18+ also has --watch)
npm install --save-dev nodemon
```

What each package is for:

| Package | Role in this project |
| --- | --- |
| `express` | HTTP framework, routing, middleware |
| `mongoose` | MongoDB ODM — the `User` schema/model |
| `bcrypt` | Password hashing (also used to hash refresh/reset tokens) |
| `jsonwebtoken` | Sign & verify JWT access/refresh tokens |
| `cookie-parser` | Parse the `Cookie` header into `req.cookies` |
| `express-rate-limit` | Throttle auth endpoints |
| `nodemailer` | Send the password-reset email (dev inbox) |
| `validator` | Email validation in the schema |
| `dotenv` | Load `.env` into `process.env` |
| `helmet` | Sensible secure HTTP response headers |
| `nodemon` (dev) | Restart the server on save |

> TIP: `bcrypt` is a native (C++) addon and needs build tools. If `npm install bcrypt` fails on your machine, swap it for the pure-JavaScript **`bcryptjs`** (`npm i bcryptjs`) — the API is identical, just change the `require`. It's slightly slower but never fails to compile.

### Configure npm scripts

Open `package.json` and add a `scripts` block (and `"type": "commonjs"` to be explicit):

```json
{
  "name": "auth-system",
  "version": "1.0.0",
  "type": "commonjs",
  "main": "src/server.js",
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js"
  }
}
```

### Folder structure

We use a `src/` directory with the layered structure. Create the tree below (you'll fill each file in the build steps):

```text
auth-system/
├── .env                      # real secrets (gitignored — never commit)
├── .env.example              # template teammates copy to .env
├── .gitignore
├── package.json
└── src/
    ├── server.js             # entry point: connect DB, start HTTP server
    ├── app.js                # build the Express app (middleware + routes)
    ├── config/
    │   └── env.js            # load + validate environment variables
    ├── models/
    │   └── user.model.js     # Mongoose User schema (hashing, helpers)
    ├── services/
    │   ├── auth.service.js    # business logic: register/login/reset
    │   ├── token.service.js   # sign/verify JWTs, cookie helpers
    │   └── email.service.js   # Nodemailer transport + send helpers
    ├── controllers/
    │   ├── auth.controller.js # HTTP layer for /auth routes
    │   └── user.controller.js # HTTP layer for /users routes
    ├── middleware/
    │   ├── protect.js         # verify access token, load req.user
    │   ├── restrictTo.js      # role-based authorization
    │   ├── rateLimiter.js     # express-rate-limit configs
    │   └── errorHandler.js    # centralized error middleware + 404
    ├── routes/
    │   ├── index.js           # mounts all route modules under /api/v1
    │   ├── auth.routes.js     # /api/v1/auth/*
    │   └── user.routes.js     # /api/v1/users/*
    └── utils/
        ├── AppError.js        # operational error class
        └── catchAsync.js      # async error-forwarding wrapper
```

> BEST PRACTICE: Keep **routes thin, controllers HTTP-only, and business rules in services.** A controller's job is to read the request, call a service, and shape the response. When auth logic lives in services, you can unit-test "can this user log in?" without spinning up Express.

---

## Step 1 — Environment config & secrets

Auth code is full of secrets (the JWT signing key, the DB URL, SMTP credentials). Hard-coding them is the single most common way real apps leak. We centralize and **validate** them so the app refuses to start if a critical secret is missing — failing loudly at boot is far better than failing silently at 2 a.m.

### `.env.example`

This file is committed and documents every variable. Teammates copy it to `.env` and fill in real values.

```bash
# .env.example  — copy to .env and fill in real values (NEVER commit .env)

# ---- Server ----
NODE_ENV=development
PORT=3000

# ---- Database ----
# Local:  mongodb://127.0.0.1:27017/auth-system
# Atlas:  mongodb+srv://<user>:<pass>@cluster0.xxxxx.mongodb.net/auth-system
MONGODB_URI=mongodb://127.0.0.1:27017/auth-system

# ---- JWT ----
# Generate strong secrets, e.g.:  node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"
JWT_ACCESS_SECRET=replace-with-a-long-random-string
JWT_REFRESH_SECRET=replace-with-a-DIFFERENT-long-random-string
JWT_ACCESS_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d

# ---- Cookies ----
# Days the refresh cookie lasts (should match JWT_REFRESH_EXPIRES_IN)
REFRESH_COOKIE_EXPIRES_DAYS=7

# ---- bcrypt ----
BCRYPT_SALT_ROUNDS=12

# ---- Password reset ----
PASSWORD_RESET_EXPIRES_MIN=10

# ---- Email (Nodemailer) ----
# In dev, leave these blank to use an auto-created Ethereal test inbox.
# In prod, use real SMTP credentials (Mailtrap, SES, Postmark, etc.).
EMAIL_HOST=
EMAIL_PORT=587
EMAIL_USERNAME=
EMAIL_PASSWORD=
EMAIL_FROM="Auth System <no-reply@example.com>"

# Base URL used to build links inside emails
APP_BASE_URL=http://localhost:3000
```

> WARNING — Pitfall: **Use two different secrets** for access and refresh tokens. If they share a key, a leaked access token's signing key also forges refresh tokens. And never use a short, guessable secret — a JWT signed with `"secret"` can be cracked offline in seconds.

### `.gitignore`

```bash
# .gitignore
node_modules/
.env
*.log
.DS_Store
```

### `src/config/env.js`

We read `process.env` once, validate the required keys, coerce types, and export a frozen config object. The rest of the app imports from here — it never touches `process.env` directly.

```js
// src/config/env.js
'use strict';

// Load variables from .env into process.env (must run before we read them).
require('dotenv').config();

// Variables the app cannot run without. We fail fast if any are missing.
const REQUIRED = ['MONGODB_URI', 'JWT_ACCESS_SECRET', 'JWT_REFRESH_SECRET'];

const missing = REQUIRED.filter((key) => !process.env[key]);
if (missing.length > 0) {
  // Throwing here stops the process at boot with a clear message.
  throw new Error(
    `Missing required environment variables: ${missing.join(', ')}\n` +
      `Copy .env.example to .env and fill them in.`
  );
}

// Small helper: parse an int env var with a fallback default.
const int = (value, fallback) => {
  const n = Number.parseInt(value, 10);
  return Number.isNaN(n) ? fallback : n;
};

const config = {
  env: process.env.NODE_ENV || 'development',
  isProd: (process.env.NODE_ENV || 'development') === 'production',
  port: int(process.env.PORT, 3000),

  mongoUri: process.env.MONGODB_URI,

  jwt: {
    accessSecret: process.env.JWT_ACCESS_SECRET,
    refreshSecret: process.env.JWT_REFRESH_SECRET,
    accessExpiresIn: process.env.JWT_ACCESS_EXPIRES_IN || '15m',
    refreshExpiresIn: process.env.JWT_REFRESH_EXPIRES_IN || '7d',
  },

  cookie: {
    refreshExpiresDays: int(process.env.REFRESH_COOKIE_EXPIRES_DAYS, 7),
  },

  bcryptSaltRounds: int(process.env.BCRYPT_SALT_ROUNDS, 12),

  passwordResetExpiresMin: int(process.env.PASSWORD_RESET_EXPIRES_MIN, 10),

  email: {
    host: process.env.EMAIL_HOST,
    port: int(process.env.EMAIL_PORT, 587),
    username: process.env.EMAIL_USERNAME,
    password: process.env.EMAIL_PASSWORD,
    from: process.env.EMAIL_FROM || 'Auth System <no-reply@example.com>',
  },

  appBaseUrl: process.env.APP_BASE_URL || 'http://localhost:3000',
};

// Object.freeze prevents accidental mutation of config at runtime.
module.exports = Object.freeze(config);
```

> BEST PRACTICE: A **validated config module** is a cheap insurance policy. The day someone deploys without `JWT_ACCESS_SECRET`, the app refuses to boot with a precise message — instead of running fine until the first login silently signs tokens with `undefined`.

---

## Step 2 — Shared utilities: `AppError` & `catchAsync`

Before writing any feature, we set up two tiny utilities that every controller and the error handler depend on. They eliminate repetitive `try/catch` and give us a clean way to throw *operational* errors (expected failures like "wrong password") versus *programmer* errors (bugs).

### `src/utils/AppError.js`

```js
// src/utils/AppError.js
'use strict';

/**
 * Operational error: an expected failure we created on purpose
 * (validation failed, not found, not authorized, ...).
 * The centralized error handler trusts these and shows their message
 * to clients. Anything that is NOT an AppError is treated as a bug.
 */
class AppError extends Error {
  constructor(message, statusCode) {
    super(message);

    this.statusCode = statusCode;
    // 4xx -> 'fail', 5xx -> 'error' (a common JSEND-ish convention)
    this.status = String(statusCode).startsWith('4') ? 'fail' : 'error';
    this.isOperational = true; // marks "safe to show the client"

    // Keep this constructor out of the stack trace for cleaner logs.
    Error.captureStackTrace(this, this.constructor);
  }
}

module.exports = AppError;
```

### `src/utils/catchAsync.js`

Express 5 forwards rejected promises from async handlers to the error middleware automatically; Express 4 does not. To make the code work identically on both — and to be explicit — we wrap async handlers so any thrown/rejected error reaches `next()`.

```js
// src/utils/catchAsync.js
'use strict';

/**
 * Wraps an async route handler so that any rejected promise is passed
 * to Express's error-handling middleware via next(err).
 * Lets controllers be written without try/catch.
 *
 * Usage: router.get('/me', catchAsync(getMe))
 */
module.exports = function catchAsync(fn) {
  return (req, res, next) => {
    // fn(...) returns a promise; .catch forwards any error to next().
    Promise.resolve(fn(req, res, next)).catch(next);
  };
};
```

> TIP: Without `catchAsync` (and on Express 4), a `throw` inside an `async` handler becomes an **unhandled promise rejection** — the request hangs until it times out instead of returning a clean error. This wrapper is the single most useful pattern for async Express apps.

---

## Step 3 — The `User` model

The model is where most of the security correctness lives: passwords are hashed before save, the password hash is never selected by default, and we add instance methods for comparing passwords and generating reset tokens. Getting this layer right means the rest of the app physically *cannot* leak a password.

### `src/models/user.model.js`

```js
// src/models/user.model.js
'use strict';

const crypto = require('node:crypto');
const mongoose = require('mongoose');
const bcrypt = require('bcrypt');
const validator = require('validator');
const config = require('../config/env');

const userSchema = new mongoose.Schema(
  {
    name: {
      type: String,
      required: [true, 'Name is required'],
      trim: true,
      maxlength: [60, 'Name cannot exceed 60 characters'],
    },

    email: {
      type: String,
      required: [true, 'Email is required'],
      unique: true, // creates a unique index (see note below)
      lowercase: true, // normalise so Foo@x.com === foo@x.com
      trim: true,
      validate: {
        validator: (v) => validator.isEmail(v),
        message: 'Please provide a valid email',
      },
    },

    password: {
      type: String,
      required: [true, 'Password is required'],
      minlength: [8, 'Password must be at least 8 characters'],
      // CRITICAL: never return the hash in normal queries.
      select: false,
    },

    role: {
      type: String,
      enum: ['user', 'admin'],
      default: 'user',
    },

    // Set whenever the password changes. protect() uses it to invalidate
    // tokens issued before a password change.
    passwordChangedAt: Date,

    // Password-reset flow: we store only the HASH of the reset token.
    passwordResetToken: String,
    passwordResetExpires: Date,

    // Email verification (extension) — included now so the schema is stable.
    isEmailVerified: { type: Boolean, default: false },
    emailVerificationToken: String,
    emailVerificationExpires: Date,

    // Account lockout (extension).
    loginAttempts: { type: Number, default: 0 },
    lockUntil: Date,

    active: { type: Boolean, default: true, select: false },
  },
  {
    timestamps: true, // adds createdAt / updatedAt
    toJSON: {
      // Strip sensitive/internal fields from any JSON we send out.
      transform(doc, ret) {
        delete ret.password;
        delete ret.passwordResetToken;
        delete ret.passwordResetExpires;
        delete ret.emailVerificationToken;
        delete ret.emailVerificationExpires;
        delete ret.loginAttempts;
        delete ret.lockUntil;
        delete ret.__v;
        return ret;
      },
    },
  }
);

// ---- Virtual: convenient lock status -------------------------------------
userSchema.virtual('isLocked').get(function () {
  return Boolean(this.lockUntil && this.lockUntil.getTime() > Date.now());
});

// ---- Hash password before saving -----------------------------------------
userSchema.pre('save', async function (next) {
  // Only hash if the password field was actually modified (e.g. on create
  // or password change) — otherwise we'd re-hash an already-hashed value.
  if (!this.isModified('password')) return next();

  this.password = await bcrypt.hash(this.password, config.bcryptSaltRounds);
  next();
});

// ---- Stamp passwordChangedAt when the password changes -------------------
userSchema.pre('save', function (next) {
  // Skip on brand-new docs and when password didn't change.
  if (!this.isModified('password') || this.isNew) return next();

  // Subtract 1s: tokens are sometimes issued a hair before this save
  // completes; the buffer prevents a valid token from being wrongly rejected.
  this.passwordChangedAt = new Date(Date.now() - 1000);
  next();
});

// ---- Exclude soft-deleted users from all find queries --------------------
userSchema.pre(/^find/, function (next) {
  // 'this' is the query. Only return users that are not deactivated.
  this.where({ active: { $ne: false } });
  next();
});

// ---- Instance method: compare a candidate password to the stored hash ----
userSchema.methods.correctPassword = async function (candidate) {
  // 'this.password' must have been selected explicitly (it's select:false).
  return bcrypt.compare(candidate, this.password);
};

// ---- Instance method: was the password changed AFTER a token was issued? -
userSchema.methods.changedPasswordAfter = function (jwtIssuedAtSeconds) {
  if (!this.passwordChangedAt) return false;
  const changedAtSeconds = Math.floor(
    this.passwordChangedAt.getTime() / 1000
  );
  return jwtIssuedAtSeconds < changedAtSeconds;
};

// ---- Instance method: create a password-reset token ----------------------
// Returns the PLAINTEXT token (emailed to the user). We store only its hash,
// so a database leak does not expose usable reset tokens.
userSchema.methods.createPasswordResetToken = function () {
  const resetToken = crypto.randomBytes(32).toString('hex');

  this.passwordResetToken = crypto
    .createHash('sha256')
    .update(resetToken)
    .digest('hex');

  this.passwordResetExpires = new Date(
    Date.now() + config.passwordResetExpiresMin * 60 * 1000
  );

  return resetToken; // plaintext — goes in the email link only
};

// ---- Instance method: create an email-verification token (extension) -----
userSchema.methods.createEmailVerificationToken = function () {
  const verifyToken = crypto.randomBytes(32).toString('hex');

  this.emailVerificationToken = crypto
    .createHash('sha256')
    .update(verifyToken)
    .digest('hex');

  // Verification links can live longer than reset links (24h here).
  this.emailVerificationExpires = new Date(Date.now() + 24 * 60 * 60 * 1000);

  return verifyToken;
};

module.exports = mongoose.model('User', userSchema);
```

A few design decisions worth internalizing:

- **`select: false` on `password`** means `User.findOne({ email })` returns a document *without* the hash. To check a password at login you must opt in with `.select('+password')`. This makes leaking the hash an explicit, visible act in code rather than a default.
- **We hash the reset token before storing it.** The email contains the raw token; the DB stores `sha256(token)`. If your DB leaks, attackers can't use the stored hashes to reset passwords (they'd need the original random bytes).
- **`changedPasswordAfter`** is the mechanism that makes "log out everywhere" / "passwords changed → old tokens die" possible even with stateless JWTs.

> WARNING — Pitfall: The `unique: true` on `email` is **not a validator** — it's a MongoDB index. It only works once the index is built, and a duplicate insert throws a MongoServerError with code `11000`, *not* a Mongoose ValidationError. Our error handler (Step 9) translates that code into a clean 400. Also: if duplicates already exist when the index is created, the index build fails silently — start with a clean collection.

---

## Step 4 — The token service

All JWT logic lives in one place. Controllers never call `jwt.sign` directly — they ask the token service. This is what lets you change the algorithm, expiry, or cookie flags in exactly one file.

### `src/services/token.service.js`

```js
// src/services/token.service.js
'use strict';

const jwt = require('jsonwebtoken');
const config = require('../config/env');

// ---- Sign an ACCESS token (short-lived, sent in Authorization header) ----
function signAccessToken(user) {
  // Keep the payload minimal. Never put the password or secrets in a JWT —
  // the payload is only base64-encoded, NOT encrypted; anyone can read it.
  return jwt.sign(
    { sub: user.id, role: user.role },
    config.jwt.accessSecret,
    { expiresIn: config.jwt.accessExpiresIn }
  );
}

// ---- Sign a REFRESH token (long-lived, stored in an httpOnly cookie) -----
function signRefreshToken(user) {
  return jwt.sign(
    { sub: user.id },
    config.jwt.refreshSecret,
    { expiresIn: config.jwt.refreshExpiresIn }
  );
}

// ---- Verify helpers (throw on invalid/expired — callers wrap them) -------
function verifyAccessToken(token) {
  return jwt.verify(token, config.jwt.accessSecret);
}

function verifyRefreshToken(token) {
  return jwt.verify(token, config.jwt.refreshSecret);
}

// ---- Cookie options for the refresh token --------------------------------
// httpOnly  : JS cannot read it (mitigates XSS token theft)
// secure    : only sent over HTTPS (enabled in production)
// sameSite  : 'strict'/'lax' mitigate CSRF; 'lax' lets top-level navigations work
// path      : scope the cookie to the refresh endpoint so it isn't sent everywhere
function refreshCookieOptions() {
  const maxAgeMs = config.cookie.refreshExpiresDays * 24 * 60 * 60 * 1000;
  return {
    httpOnly: true,
    secure: config.isProd, // true in prod (HTTPS), false in local dev
    sameSite: 'strict',
    path: '/api/v1/auth', // sent only to auth routes (refresh/logout)
    maxAge: maxAgeMs,
  };
}

// Attach the refresh token as an httpOnly cookie on the response.
function setRefreshCookie(res, refreshToken) {
  res.cookie('refreshToken', refreshToken, refreshCookieOptions());
}

// Clear the cookie (logout). Options (except maxAge) must match those used
// when setting it, or the browser won't clear it.
function clearRefreshCookie(res) {
  res.clearCookie('refreshToken', {
    httpOnly: true,
    secure: config.isProd,
    sameSite: 'strict',
    path: '/api/v1/auth',
  });
}

module.exports = {
  signAccessToken,
  signRefreshToken,
  verifyAccessToken,
  verifyRefreshToken,
  setRefreshCookie,
  clearRefreshCookie,
};
```

> WARNING — Pitfall: A JWT payload is **signed, not encrypted**. Anyone can paste the token into jwt.io and read `sub` and `role`. Never put anything secret in it. The signature only proves the payload wasn't *tampered with* — it does not hide the contents.

> BEST PRACTICE: Access tokens should be **short-lived** (5–15 minutes). The damage window of a stolen access token is then tiny. The long-lived refresh token, which can mint new access tokens, lives in an `httpOnly` cookie so JavaScript (and thus most XSS payloads) can't read it.

---

## Step 5 — The email service

The password-reset flow emails a link. In development we don't want to wire up a real mail provider, so we use **Ethereal** — a fake SMTP service Nodemailer can create on the fly that captures messages and gives you a preview URL. In production you supply real SMTP credentials via env vars.

### `src/services/email.service.js`

```js
// src/services/email.service.js
'use strict';

const nodemailer = require('nodemailer');
const config = require('../config/env');

// Create (and cache) a Nodemailer transport.
// - If real SMTP creds are provided, use them.
// - Otherwise create an Ethereal test account (dev-only fake inbox).
let cachedTransport = null;

async function getTransport() {
  if (cachedTransport) return cachedTransport;

  if (config.email.host && config.email.username) {
    // Real SMTP (prod or a service like Mailtrap/SES/Postmark)
    cachedTransport = nodemailer.createTransport({
      host: config.email.host,
      port: config.email.port,
      secure: config.email.port === 465, // 465 = implicit TLS
      auth: {
        user: config.email.username,
        pass: config.email.password,
      },
    });
  } else {
    // Dev fallback: Ethereal generates throwaway credentials.
    const testAccount = await nodemailer.createTestAccount();
    cachedTransport = nodemailer.createTransport({
      host: 'smtp.ethereal.email',
      port: 587,
      secure: false,
      auth: { user: testAccount.user, pass: testAccount.pass },
    });
    // eslint-disable-next-line no-console
    console.log('📧 Using Ethereal test inbox:', testAccount.user);
  }

  return cachedTransport;
}

/**
 * Send an email. Returns the Nodemailer info object.
 * In dev with Ethereal, logs a preview URL you can open in a browser.
 */
async function sendEmail({ to, subject, text, html }) {
  const transport = await getTransport();

  const info = await transport.sendMail({
    from: config.email.from,
    to,
    subject,
    text,
    html,
  });

  const previewUrl = nodemailer.getTestMessageUrl(info);
  if (previewUrl) {
    // eslint-disable-next-line no-console
    console.log('📧 Preview email at:', previewUrl);
  }

  return info;
}

// ---- Specific email: password reset --------------------------------------
async function sendPasswordResetEmail(user, resetUrl) {
  const subject = 'Your password reset link (valid for 10 minutes)';
  const text =
    `Hi ${user.name},\n\n` +
    `You (or someone) requested a password reset.\n` +
    `Open this link to set a new password:\n${resetUrl}\n\n` +
    `If you didn't request this, ignore this email — your password is unchanged.`;
  const html = `
    <p>Hi ${user.name},</p>
    <p>You (or someone) requested a password reset.</p>
    <p><a href="${resetUrl}">Click here to set a new password</a> (valid for 10 minutes).</p>
    <p>If you didn't request this, ignore this email — your password is unchanged.</p>
  `;

  return sendEmail({ to: user.email, subject, text, html });
}

// ---- Specific email: email verification (extension) ----------------------
async function sendVerificationEmail(user, verifyUrl) {
  const subject = 'Verify your email address';
  const text =
    `Hi ${user.name},\n\nVerify your email by opening:\n${verifyUrl}`;
  const html = `
    <p>Hi ${user.name},</p>
    <p><a href="${verifyUrl}">Click here to verify your email</a>.</p>
  `;
  return sendEmail({ to: user.email, subject, text, html });
}

module.exports = { sendEmail, sendPasswordResetEmail, sendVerificationEmail };
```

> TIP: Ethereal gives you a clickable **preview URL** in the terminal — you never need a real inbox to test the reset flow. For a UI-rich fake inbox, **Mailtrap** is another popular dev choice; just put its SMTP creds in `.env`.

---

## Step 6 — The auth service (business logic)

This is the heart of the system. The service owns the *rules*: how registration works, how login validates credentials without leaking which field was wrong, and how the reset flow operates. Controllers stay thin and just translate between HTTP and these functions.

### `src/services/auth.service.js`

```js
// src/services/auth.service.js
'use strict';

const crypto = require('node:crypto');
const User = require('../models/user.model');
const AppError = require('../utils/AppError');
const tokenService = require('./token.service');
const emailService = require('./email.service');
const config = require('../config/env');

// Bundle the two tokens a successful auth produces.
function issueTokens(user) {
  return {
    accessToken: tokenService.signAccessToken(user),
    refreshToken: tokenService.signRefreshToken(user),
  };
}

// ---- Register a new user -------------------------------------------------
async function register({ name, email, password }) {
  // We deliberately pick only allowed fields — a client cannot set role:'admin'
  // by sending it in the body (mass-assignment protection).
  const user = await User.create({ name, email, password });
  const tokens = issueTokens(user);
  return { user, tokens };
}

// ---- Login ---------------------------------------------------------------
async function login({ email, password }) {
  // password is select:false, so explicitly include it for the comparison.
  const user = await User.findOne({ email }).select('+password');

  // SECURITY: identical error whether the email is unknown OR the password
  // is wrong. Leaking "no such user" lets attackers enumerate valid emails.
  const genericError = new AppError('Incorrect email or password', 401);

  if (!user) {
    // Optional hardening: run a dummy bcrypt compare so response time for a
    // missing user matches a wrong-password user (mitigates timing attacks).
    await User.prototype.correctPassword.call(
      { password: '$2b$12$invalidinvalidinvalidinvalidinvalidinvalidinv' },
      password
    ).catch(() => {});
    throw genericError;
  }

  const isCorrect = await user.correctPassword(password);
  if (!isCorrect) throw genericError;

  const tokens = issueTokens(user);
  return { user, tokens };
}

// ---- Refresh: exchange a valid refresh token for a new access token ------
async function refresh(refreshToken) {
  if (!refreshToken) throw new AppError('No refresh token provided', 401);

  let payload;
  try {
    payload = tokenService.verifyRefreshToken(refreshToken);
  } catch {
    throw new AppError('Invalid or expired refresh token', 401);
  }

  const user = await User.findById(payload.sub);
  if (!user) throw new AppError('User no longer exists', 401);

  // Rotate: issue a brand-new pair (see the rotation extension for reuse
  // detection backed by a stored token hash).
  const tokens = issueTokens(user);
  return { user, tokens };
}

// ---- Update password while logged in -------------------------------------
async function updatePassword({ userId, currentPassword, newPassword }) {
  const user = await User.findById(userId).select('+password');
  if (!user) throw new AppError('User not found', 404);

  const ok = await user.correctPassword(currentPassword);
  if (!ok) throw new AppError('Your current password is incorrect', 401);

  user.password = newPassword; // pre('save') hook hashes it + stamps changedAt
  await user.save();

  const tokens = issueTokens(user); // fresh tokens after password change
  return { user, tokens };
}

// ---- Forgot password: generate token + email the link --------------------
async function forgotPassword(email) {
  const user = await User.findOne({ email });

  // SECURITY: always respond the same way, even if the email isn't registered,
  // so attackers can't use this endpoint to discover which emails exist.
  if (!user) return;

  const resetToken = user.createPasswordResetToken();
  // Skip schema validators here: we're only touching reset fields.
  await user.save({ validateBeforeSave: false });

  const resetUrl =
    `${config.appBaseUrl}/api/v1/auth/reset-password/${resetToken}`;

  try {
    await emailService.sendPasswordResetEmail(user, resetUrl);
  } catch (err) {
    // If sending fails, roll back the token so the user can retry cleanly.
    user.passwordResetToken = undefined;
    user.passwordResetExpires = undefined;
    await user.save({ validateBeforeSave: false });
    throw new AppError(
      'Failed to send the reset email. Please try again later.',
      500
    );
  }
}

// ---- Reset password using the emailed token ------------------------------
async function resetPassword({ rawToken, newPassword }) {
  // Hash the incoming raw token and look it up; also require not-expired.
  const hashedToken = crypto
    .createHash('sha256')
    .update(rawToken)
    .digest('hex');

  const user = await User.findOne({
    passwordResetToken: hashedToken,
    passwordResetExpires: { $gt: new Date() },
  });

  if (!user) throw new AppError('Token is invalid or has expired', 400);

  // Set the new password and clear the reset fields.
  user.password = newPassword; // hashed + changedAt stamped by hooks
  user.passwordResetToken = undefined;
  user.passwordResetExpires = undefined;
  await user.save();

  const tokens = issueTokens(user);
  return { user, tokens };
}

module.exports = {
  register,
  login,
  refresh,
  updatePassword,
  forgotPassword,
  resetPassword,
};
```

> WARNING — Pitfall: The most common login security bug is returning `404 "User not found"` for an unknown email and `401 "Wrong password"` for a known email. That difference lets an attacker **enumerate valid accounts**. Always return one identical, generic error.

> BEST PRACTICE: In `forgotPassword`, return the same success response whether or not the email exists. Otherwise the endpoint becomes a free email-existence checker for spammers and credential-stuffers.

---

## Step 7 — Controllers (the HTTP layer)

Controllers read the request, call a service, set cookies, and shape the JSON response. They contain **no business logic** — that all lives in services. Notice every handler is wrapped in `catchAsync`.

### `src/controllers/auth.controller.js`

```js
// src/controllers/auth.controller.js
'use strict';

const catchAsync = require('../utils/catchAsync');
const AppError = require('../utils/AppError');
const authService = require('../services/auth.service');
const tokenService = require('../services/token.service');

// Helper: send the access token in the body + refresh token in an httpOnly cookie.
function sendAuthResponse(res, statusCode, user, tokens) {
  tokenService.setRefreshCookie(res, tokens.refreshToken);
  res.status(statusCode).json({
    status: 'success',
    accessToken: tokens.accessToken,
    // user.toJSON() strips password & sensitive fields (see model transform).
    data: { user },
  });
}

// POST /auth/register
const register = catchAsync(async (req, res) => {
  const { name, email, password } = req.body;
  const { user, tokens } = await authService.register({ name, email, password });
  sendAuthResponse(res, 201, user, tokens);
});

// POST /auth/login
const login = catchAsync(async (req, res, next) => {
  const { email, password } = req.body;
  if (!email || !password) {
    return next(new AppError('Please provide email and password', 400));
  }
  const { user, tokens } = await authService.login({ email, password });
  sendAuthResponse(res, 200, user, tokens);
});

// POST /auth/refresh  (reads refresh token from the httpOnly cookie)
const refresh = catchAsync(async (req, res) => {
  const refreshToken = req.cookies?.refreshToken;
  const { user, tokens } = await authService.refresh(refreshToken);
  sendAuthResponse(res, 200, user, tokens);
});

// POST /auth/logout
const logout = catchAsync(async (req, res) => {
  tokenService.clearRefreshCookie(res);
  // (Extension: also revoke the stored refresh token server-side.)
  res.status(200).json({ status: 'success', message: 'Logged out' });
});

// GET /auth/me  (protect middleware has already loaded req.user)
const getMe = catchAsync(async (req, res) => {
  res.status(200).json({ status: 'success', data: { user: req.user } });
});

// PATCH /auth/update-password  (logged-in password change)
const updatePassword = catchAsync(async (req, res, next) => {
  const { currentPassword, newPassword } = req.body;
  if (!currentPassword || !newPassword) {
    return next(
      new AppError('Provide currentPassword and newPassword', 400)
    );
  }
  const { user, tokens } = await authService.updatePassword({
    userId: req.user.id,
    currentPassword,
    newPassword,
  });
  sendAuthResponse(res, 200, user, tokens);
});

// POST /auth/forgot-password
const forgotPassword = catchAsync(async (req, res, next) => {
  const { email } = req.body;
  if (!email) return next(new AppError('Please provide your email', 400));

  await authService.forgotPassword(email);

  // Always the same response regardless of whether the email exists.
  res.status(200).json({
    status: 'success',
    message: 'If that email is registered, a reset link has been sent.',
  });
});

// PATCH /auth/reset-password/:token
const resetPassword = catchAsync(async (req, res, next) => {
  const { newPassword } = req.body;
  if (!newPassword) {
    return next(new AppError('Please provide a new password', 400));
  }
  const { user, tokens } = await authService.resetPassword({
    rawToken: req.params.token,
    newPassword,
  });
  sendAuthResponse(res, 200, user, tokens);
});

module.exports = {
  register,
  login,
  refresh,
  logout,
  getMe,
  updatePassword,
  forgotPassword,
  resetPassword,
};
```

### `src/controllers/user.controller.js`

The admin-only endpoints — simple CRUD, but only reachable after `protect` + `restrictTo('admin')`.

```js
// src/controllers/user.controller.js
'use strict';

const catchAsync = require('../utils/catchAsync');
const AppError = require('../utils/AppError');
const User = require('../models/user.model');

// GET /users  (admin only)
const listUsers = catchAsync(async (req, res) => {
  const users = await User.find();
  res.status(200).json({
    status: 'success',
    results: users.length,
    data: { users },
  });
});

// DELETE /users/:id  (admin only) — soft delete via active flag
const deleteUser = catchAsync(async (req, res, next) => {
  const user = await User.findById(req.params.id);
  if (!user) return next(new AppError('No user found with that ID', 404));

  user.active = false;
  await user.save({ validateBeforeSave: false });

  res.status(204).send(); // 204 No Content
});

module.exports = { listUsers, deleteUser };
```

---

## Step 8 — Auth middleware: `protect`, `restrictTo`, rate limiting

These three middlewares are the gatekeepers of the API.

### `src/middleware/protect.js`

`protect` is **authentication** enforcement: it extracts the Bearer token, verifies it, loads the user, and checks the password wasn't changed after the token was issued. On success it attaches `req.user`.

```js
// src/middleware/protect.js
'use strict';

const catchAsync = require('../utils/catchAsync');
const AppError = require('../utils/AppError');
const tokenService = require('../services/token.service');
const User = require('../models/user.model');

const protect = catchAsync(async (req, res, next) => {
  // 1) Get the token from the Authorization header: "Bearer <token>"
  let token;
  const authHeader = req.headers.authorization;
  if (authHeader && authHeader.startsWith('Bearer ')) {
    token = authHeader.split(' ')[1];
  }

  if (!token) {
    return next(new AppError('You are not logged in. Please log in.', 401));
  }

  // 2) Verify the signature & expiry. Throws if invalid/expired.
  let payload;
  try {
    payload = tokenService.verifyAccessToken(token);
  } catch (err) {
    // jwt throws TokenExpiredError vs JsonWebTokenError — both -> 401.
    return next(new AppError('Invalid or expired token. Log in again.', 401));
  }

  // 3) Confirm the user still exists (could have been deleted).
  const currentUser = await User.findById(payload.sub);
  if (!currentUser) {
    return next(
      new AppError('The user for this token no longer exists.', 401)
    );
  }

  // 4) Reject tokens issued before the user's last password change.
  if (currentUser.changedPasswordAfter(payload.iat)) {
    return next(
      new AppError('Password was changed recently. Log in again.', 401)
    );
  }

  // 5) Grant access — attach the user for downstream handlers.
  req.user = currentUser;
  next();
});

module.exports = protect;
```

### `src/middleware/restrictTo.js`

`restrictTo` is **authorization** enforcement. It's a *factory*: you call it with the allowed roles and it returns a middleware. It must run *after* `protect` (it relies on `req.user`).

```js
// src/middleware/restrictTo.js
'use strict';

const AppError = require('../utils/AppError');

/**
 * Usage: router.delete('/:id', protect, restrictTo('admin'), handler)
 * Pass one or more allowed roles. 403 if the user's role isn't included.
 */
function restrictTo(...allowedRoles) {
  return (req, res, next) => {
    // protect must have run first and set req.user.
    if (!req.user) {
      return next(new AppError('Authentication required.', 401));
    }
    if (!allowedRoles.includes(req.user.role)) {
      // 403 Forbidden: authenticated, but not permitted.
      return next(
        new AppError('You do not have permission to perform this action.', 403)
      );
    }
    next();
  };
}

module.exports = restrictTo;
```

### `src/middleware/rateLimiter.js`

Brute-forcing logins is trivial without rate limiting. We define a strict limiter for auth-sensitive routes and a looser one for general API traffic.

```js
// src/middleware/rateLimiter.js
'use strict';

const rateLimit = require('express-rate-limit');

// Strict limiter for login / forgot-password / reset — blunt brute force.
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 10, // max 10 requests per IP per window for these routes
  standardHeaders: true, // send RateLimit-* headers
  legacyHeaders: false,
  message: {
    status: 'fail',
    message: 'Too many attempts. Please try again in 15 minutes.',
  },
});

// Generous global limiter applied to the whole API.
const globalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 300,
  standardHeaders: true,
  legacyHeaders: false,
  message: {
    status: 'fail',
    message: 'Too many requests, please slow down.',
  },
});

module.exports = { authLimiter, globalLimiter };
```

> WARNING — Pitfall: `express-rate-limit` keys by IP, which it reads from `req.ip`. Behind a proxy/load balancer (Heroku, Nginx, Render), `req.ip` is the proxy's IP unless you set `app.set('trust proxy', 1)`. Without it, **every user shares one bucket** and you'll rate-limit everyone at once. We set this in `app.js`.

> BEST PRACTICE: The in-memory store is fine for a single instance, but it resets on restart and isn't shared across instances. For multi-instance production, back the limiter with **Redis** (`rate-limit-redis`) so the limit is global.

---

## Step 9 — Centralized error handling

A single error middleware turns every thrown error — `AppError`, Mongoose validation errors, duplicate-key errors, JWT errors — into a consistent JSON shape. Crucially, it behaves differently in dev (full detail) vs prod (no leaking stack traces or internals).

### `src/middleware/errorHandler.js`

```js
// src/middleware/errorHandler.js
'use strict';

const AppError = require('../utils/AppError');
const config = require('../config/env');

// ---- 404 handler: any route that didn't match -> AppError ---------------
function notFound(req, res, next) {
  next(new AppError(`Cannot ${req.method} ${req.originalUrl}`, 404));
}

// ---- Translate known Mongoose/Mongo errors into clean AppErrors ----------
function handleCastError(err) {
  return new AppError(`Invalid ${err.path}: ${err.value}`, 400);
}

function handleDuplicateKey(err) {
  // err.keyValue looks like { email: 'foo@bar.com' }
  const field = Object.keys(err.keyValue || {})[0] || 'field';
  return new AppError(`That ${field} is already in use.`, 400);
}

function handleValidationError(err) {
  const messages = Object.values(err.errors).map((e) => e.message);
  return new AppError(`Invalid input: ${messages.join('. ')}`, 400);
}

// ---- Dev vs prod response shapes -----------------------------------------
function sendDev(err, res) {
  res.status(err.statusCode || 500).json({
    status: err.status || 'error',
    message: err.message,
    error: err,
    stack: err.stack, // full stack in dev only
  });
}

function sendProd(err, res) {
  if (err.isOperational) {
    // Trusted, expected error: show its message.
    return res.status(err.statusCode).json({
      status: err.status,
      message: err.message,
    });
  }
  // Unknown/programming error: don't leak details.
  // eslint-disable-next-line no-console
  console.error('💥 UNEXPECTED ERROR:', err);
  return res.status(500).json({
    status: 'error',
    message: 'Something went wrong.',
  });
}

// ---- The Express error middleware (must have 4 args) ---------------------
// eslint-disable-next-line no-unused-vars
function errorHandler(err, req, res, next) {
  // Default code/status if not set.
  err.statusCode = err.statusCode || 500;
  err.status = err.status || 'error';

  if (config.isProd) {
    // Normalise specific error types into operational AppErrors.
    let error = err;
    if (err.name === 'CastError') error = handleCastError(err);
    else if (err.code === 11000) error = handleDuplicateKey(err);
    else if (err.name === 'ValidationError') error = handleValidationError(err);
    sendProd(error, res);
  } else {
    sendDev(err, res);
  }
}

module.exports = { notFound, errorHandler };
```

> WARNING — Pitfall: Express recognizes an error-handling middleware **only if it declares four parameters** `(err, req, res, next)`. Drop the unused `next` and Express treats it as a normal middleware and your error formatting silently never runs. Keep all four even though the linter complains.

> WARNING — Express 4 vs 5: In Express 4, errors thrown in `async` handlers are **not** caught automatically — you must use `catchAsync` (we do). Express 5 forwards rejected promises to the error handler for you, but wrapping with `catchAsync` is still good for explicitness and Express-4 compatibility.

---

## Step 10 — Routes

Routes wire URLs to controllers and stack the middleware in the correct order. They are intentionally boring — that's a sign the logic lives where it should.

### `src/routes/auth.routes.js`

```js
// src/routes/auth.routes.js
'use strict';

const express = require('express');
const authController = require('../controllers/auth.controller');
const protect = require('../middleware/protect');
const { authLimiter } = require('../middleware/rateLimiter');

const router = express.Router();

// --- Public routes (rate-limited where brute-force is a risk) ---
router.post('/register', authController.register);
router.post('/login', authLimiter, authController.login);
router.post('/refresh', authController.refresh);
router.post('/logout', authController.logout);

router.post('/forgot-password', authLimiter, authController.forgotPassword);
router.patch('/reset-password/:token', authController.resetPassword);

// --- Protected routes (require a valid access token) ---
router.get('/me', protect, authController.getMe);
router.patch('/update-password', protect, authController.updatePassword);

module.exports = router;
```

### `src/routes/user.routes.js`

```js
// src/routes/user.routes.js
'use strict';

const express = require('express');
const userController = require('../controllers/user.controller');
const protect = require('../middleware/protect');
const restrictTo = require('../middleware/restrictTo');

const router = express.Router();

// Every /users route requires authentication...
router.use(protect);
// ...and admin authorization. router.use applies to all routes below it.
router.use(restrictTo('admin'));

router.get('/', userController.listUsers);
router.delete('/:id', userController.deleteUser);

module.exports = router;
```

> TIP: `router.use(protect)` followed by `router.use(restrictTo('admin'))` is a clean way to guard an entire router. Every route declared *after* those lines inherits both gates — no need to repeat them per route.

### `src/routes/index.js`

```js
// src/routes/index.js
'use strict';

const express = require('express');
const authRoutes = require('./auth.routes');
const userRoutes = require('./user.routes');

const router = express.Router();

// Health check — handy for load balancers and quick smoke tests.
router.get('/health', (req, res) => {
  res.status(200).json({ status: 'success', message: 'API is healthy' });
});

router.use('/auth', authRoutes);
router.use('/users', userRoutes);

module.exports = router;
```

---

## Step 11 — Assemble the app & start the server

Two files remain: `app.js` builds the configured Express app (no listening, which makes it testable), and `server.js` connects to MongoDB and starts the HTTP server with graceful shutdown.

### `src/app.js`

```js
// src/app.js
'use strict';

const express = require('express');
const helmet = require('helmet');
const cookieParser = require('cookie-parser');

const apiRoutes = require('./routes');
const { globalLimiter } = require('./middleware/rateLimiter');
const { notFound, errorHandler } = require('./middleware/errorHandler');

const app = express();

// Trust the first proxy (needed for correct req.ip behind a proxy, and for
// 'secure' cookies to work when TLS terminates at the proxy). See rate-limit note.
app.set('trust proxy', 1);

// --- Security & parsing middleware (order matters) ---
app.use(helmet()); // sensible secure HTTP headers
app.use(express.json({ limit: '10kb' })); // parse JSON, cap body size (anti-DoS)
app.use(cookieParser()); // populate req.cookies
app.use(globalLimiter); // global rate limit

// --- Routes ---
app.use('/api/v1', apiRoutes);

// --- 404 + centralized error handler (must be LAST) ---
app.use(notFound);
app.use(errorHandler);

module.exports = app;
```

### `src/server.js`

```js
// src/server.js
'use strict';

const mongoose = require('mongoose');
const app = require('./app');
const config = require('./config/env');

// Catch synchronous programmer errors thrown outside Express.
process.on('uncaughtException', (err) => {
  // eslint-disable-next-line no-console
  console.error('💥 UNCAUGHT EXCEPTION! Shutting down...', err);
  process.exit(1);
});

async function start() {
  // 1) Connect to MongoDB first — no point starting HTTP without a DB.
  await mongoose.connect(config.mongoUri);
  // eslint-disable-next-line no-console
  console.log('✅ MongoDB connected');

  // 2) Start the HTTP server.
  const server = app.listen(config.port, () => {
    // eslint-disable-next-line no-console
    console.log(`🚀 Server listening on http://localhost:${config.port} (${config.env})`);
  });

  // 3) Graceful shutdown: handle async errors and termination signals.
  process.on('unhandledRejection', (err) => {
    // eslint-disable-next-line no-console
    console.error('💥 UNHANDLED REJECTION! Shutting down...', err);
    server.close(() => process.exit(1));
  });

  process.on('SIGTERM', () => {
    // eslint-disable-next-line no-console
    console.log('👋 SIGTERM received. Closing server...');
    server.close(() => {
      mongoose.connection.close(false).then(() => process.exit(0));
    });
  });
}

start().catch((err) => {
  // eslint-disable-next-line no-console
  console.error('💥 Failed to start:', err);
  process.exit(1);
});
```

> BEST PRACTICE: Splitting `app.js` (builds the app) from `server.js` (starts it) is the standard testable pattern. Your test suite can `require('./app')` and hit it with Supertest **without binding to a port** — and the same `app` powers production.

---

## How to Run & Test

### 1. Configure and start

```bash
# Copy the env template and edit secrets
cp .env.example .env

# Generate strong JWT secrets and paste them into .env
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"

# Make sure MongoDB is running (local) or your Atlas URI is in .env

# Start in dev (auto-restart)
npm run dev
```

You should see `✅ MongoDB connected` and `🚀 Server listening on http://localhost:3000`.

### 2. Smoke test the health endpoint

```bash
curl -s http://localhost:3000/api/v1/health
# {"status":"success","message":"API is healthy"}
```

### 3. Register a user

```bash
curl -s -X POST http://localhost:3000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name":"Ada Lovelace","email":"ada@example.com","password":"Sup3rSecret!"}'
```

Sample response (note: no password field, and a `Set-Cookie: refreshToken=...` header is returned):

```json
{
  "status": "success",
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "data": {
    "user": {
      "_id": "6655f0c2a1b2c3d4e5f60718",
      "name": "Ada Lovelace",
      "email": "ada@example.com",
      "role": "user",
      "isEmailVerified": false,
      "createdAt": "2026-06-19T10:00:00.000Z",
      "updatedAt": "2026-06-19T10:00:00.000Z"
    }
  }
}
```

### 4. Log in and capture the cookie + token

`curl -c cookies.txt` saves the refresh cookie to a jar; we'll reuse it for `/refresh` and `/logout`.

```bash
curl -s -c cookies.txt -X POST http://localhost:3000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"ada@example.com","password":"Sup3rSecret!"}'
```

Grab the `accessToken` from the response and store it in a shell variable for convenience:

```bash
ACCESS=$(curl -s -c cookies.txt -X POST http://localhost:3000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"ada@example.com","password":"Sup3rSecret!"}' \
  | node -e "let d='';process.stdin.on('data',c=>d+=c).on('end',()=>console.log(JSON.parse(d).accessToken))")
echo "$ACCESS"
```

### 5. Call a protected route (`/me`)

```bash
curl -s http://localhost:3000/api/v1/auth/me \
  -H "Authorization: Bearer $ACCESS"
```

```json
{ "status": "success", "data": { "user": { "name": "Ada Lovelace", "email": "ada@example.com", "role": "user" } } }
```

Without the header you get a clean 401:

```bash
curl -s http://localhost:3000/api/v1/auth/me
# {"status":"fail","message":"You are not logged in. Please log in."}
```

### 6. Refresh the access token

Uses the cookie jar (no Bearer header needed — the refresh token is in the cookie):

```bash
curl -s -b cookies.txt -c cookies.txt -X POST \
  http://localhost:3000/api/v1/auth/refresh
# -> new accessToken + a rotated refresh cookie
```

### 7. Wrong-password login (note the generic message)

```bash
curl -s -X POST http://localhost:3000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"ada@example.com","password":"wrong"}'
# {"status":"fail","message":"Incorrect email or password"}
```

The *exact same* message comes back for an unknown email — that's the anti-enumeration design working.

### 8. Test role-based authorization

Listing users requires `admin`. Ada is a `user`, so she gets 403:

```bash
curl -s http://localhost:3000/api/v1/users \
  -H "Authorization: Bearer $ACCESS"
# {"status":"fail","message":"You do not have permission to perform this action."}
```

To create an admin, promote a user directly in MongoDB (clients can never set their own role), then log in as that user:

```bash
# In mongosh (or Compass):
# use auth-system
# db.users.updateOne({ email: "ada@example.com" }, { $set: { role: "admin" } })
```

Re-login to get a token whose payload says `role: 'admin'`, then `GET /users` returns 200.

### 9. Password-reset flow

```bash
# Step 1: request a reset link
curl -s -X POST http://localhost:3000/api/v1/auth/forgot-password \
  -H "Content-Type: application/json" \
  -d '{"email":"ada@example.com"}'
# {"status":"success","message":"If that email is registered, a reset link has been sent."}
```

Check your terminal — the email service logs an **Ethereal preview URL** and the reset link contains the token. Then:

```bash
# Step 2: use the token from the link to set a new password
curl -s -X PATCH http://localhost:3000/api/v1/auth/reset-password/PASTE_RAW_TOKEN_HERE \
  -H "Content-Type: application/json" \
  -d '{"newPassword":"My-N3w-Passw0rd!"}'
```

### 10. Logout

```bash
curl -s -b cookies.txt -X POST http://localhost:3000/api/v1/auth/logout
# {"status":"success","message":"Logged out"}  — refresh cookie cleared
```

> TIP: In **Postman**, enable "Automatically follow redirects" off and turn on the cookie jar (Settings → General). Create a collection variable `accessToken` and add a *Tests* script to login: `pm.collectionVariables.set('accessToken', pm.response.json().accessToken)`. Then set the Authorization type to "Bearer Token" with value `{{accessToken}}` on protected requests.

### Sample requests summary table

| Scenario | Command shape | Expected status |
| --- | --- | --- |
| Register | `POST /auth/register` with name/email/password | 201 |
| Login (valid) | `POST /auth/login` | 200 + Set-Cookie |
| Login (invalid) | `POST /auth/login` wrong password | 401 generic |
| Get me (with token) | `GET /auth/me` + Bearer | 200 |
| Get me (no token) | `GET /auth/me` | 401 |
| Admin route as user | `GET /users` + user token | 403 |
| Refresh | `POST /auth/refresh` + cookie | 200 |
| Rate limit tripped | 11th `POST /auth/login` in 15 min | 429 |

---

## Code Walkthrough / Key Concepts Recap

Now that it runs, let's connect the moving parts to the *why* — this is the part interviewers probe.

### The request lifecycle of a protected call

When `GET /api/v1/auth/me` arrives with `Authorization: Bearer <token>`:

1. `helmet`, `express.json`, `cookieParser`, and `globalLimiter` run (app-level middleware).
2. The router matches `/api/v1/auth/me` and runs `protect`.
3. `protect` pulls the token, calls `tokenService.verifyAccessToken` (which runs `jwt.verify` — checking the **signature** with `JWT_ACCESS_SECRET` and the **expiry**), loads the user by `payload.sub`, and verifies `changedPasswordAfter`. It sets `req.user`.
4. The `getMe` controller reads `req.user` and responds. `user.toJSON()` strips sensitive fields via the schema `transform`.
5. Any error thrown along the way is forwarded by `catchAsync` to `errorHandler`, which formats it.

### Why hashing, not encryption

Passwords are **hashed**, never encrypted. Encryption is reversible (there's a key that decrypts) — hashing is one-way. bcrypt additionally:

- **Salts** every password (a random value stored *inside* the hash string), so two users with the same password get different hashes and precomputed "rainbow tables" are useless.
- Has a tunable **cost factor** (our `BCRYPT_SALT_ROUNDS=12`), making each hash deliberately slow (~hundreds of ms). Slowness is a *feature*: it caps how many guesses an attacker can make per second.

| | Encryption | Hashing (bcrypt) |
| --- | --- | --- |
| Reversible? | Yes (with key) | No |
| Use for passwords? | No | **Yes** |
| Same input → same output? | Yes | No (salted) |
| Tunable cost? | No | Yes (rounds) |

### Access vs refresh tokens

| | Access token | Refresh token |
| --- | --- | --- |
| Lifetime | Short (15 min) | Long (7 days) |
| Sent how | `Authorization: Bearer` header | `httpOnly` cookie |
| Purpose | Authorize each API call | Mint new access tokens |
| Readable by JS? | Yes (client holds it) | **No** (httpOnly) |
| If stolen | Small damage window | Large — so we keep it out of JS reach & rotate it |

The split is a deliberate tradeoff: stateless access tokens make every request fast (no DB lookup needed beyond `protect`'s user load), while the refresh token lets us keep sessions alive without making access tokens long-lived.

### Why the refresh token is a cookie, not in the body

If you stored the refresh token in `localStorage`, any XSS (malicious injected JS) could read and exfiltrate it. An `httpOnly` cookie is invisible to JavaScript, so XSS can't steal it. The cost is **CSRF** exposure (cookies are sent automatically), which we mitigate with `sameSite: 'strict'` and by scoping the cookie `path` to `/api/v1/auth`.

### 401 vs 403 — say it correctly in interviews

- **401 Unauthorized** → actually means *unauthenticated* — "I don't know who you are" (no/invalid token). Produced by `protect`.
- **403 Forbidden** → *unauthorized* in the access-control sense — "I know who you are, you just can't do this." Produced by `restrictTo`.

### Not leaking which field failed

The single generic `'Incorrect email or password'` is a security control, not laziness. Combined with the dummy bcrypt compare for unknown users, it resists both **account enumeration** and **timing attacks**. The forgot-password endpoint follows the same principle.

### The model is the last line of defense

`select: false` on `password`, the `toJSON` transform, and the pre-save hashing hook mean that even a sloppy controller can't accidentally return or store a plaintext password. Push security invariants down to the lowest layer that can enforce them.

---

## Extensions & Challenges

These take the project from "solid" to "senior-level." Each builds on what you have.

### Extension A — Email verification

Require users to confirm their email before they can log in (or before granting elevated capabilities).

1. On register, generate a verification token (the model already has `createEmailVerificationToken`) and email a link.
2. Add a route `GET /auth/verify-email/:token` that hashes the token, finds the matching unexpired user, sets `isEmailVerified = true`, and clears the token fields.
3. In `login`, reject unverified users with a clear (but still non-enumerating) error or a `403`.

```js
// src/controllers/auth.controller.js  (add)
const crypto = require('node:crypto');
const User = require('../models/user.model');

// GET /auth/verify-email/:token
const verifyEmail = catchAsync(async (req, res, next) => {
  const hashed = crypto
    .createHash('sha256')
    .update(req.params.token)
    .digest('hex');

  const user = await User.findOne({
    emailVerificationToken: hashed,
    emailVerificationExpires: { $gt: new Date() },
  });
  if (!user) return next(new AppError('Verification link is invalid or expired', 400));

  user.isEmailVerified = true;
  user.emailVerificationToken = undefined;
  user.emailVerificationExpires = undefined;
  await user.save({ validateBeforeSave: false });

  res.status(200).json({ status: 'success', message: 'Email verified. You can now log in.' });
});
```

Wire it in `register` (service) by creating the token, saving, and emailing `sendVerificationEmail(user, verifyUrl)`, and add `router.get('/verify-email/:token', verifyEmail)`.

### Extension B — Account lockout

After N consecutive failed logins, lock the account for a cooldown to defeat targeted brute force (complements IP rate limiting, which an attacker can dodge by rotating IPs).

```js
// src/models/user.model.js  (add instance methods)
const MAX_ATTEMPTS = 5;
const LOCK_TIME_MS = 15 * 60 * 1000; // 15 minutes

userSchema.methods.registerFailedLogin = async function () {
  // If a previous lock has expired, reset the counter.
  if (this.lockUntil && this.lockUntil.getTime() < Date.now()) {
    this.loginAttempts = 1;
    this.lockUntil = undefined;
  } else {
    this.loginAttempts += 1;
    if (this.loginAttempts >= MAX_ATTEMPTS) {
      this.lockUntil = new Date(Date.now() + LOCK_TIME_MS);
    }
  }
  await this.save({ validateBeforeSave: false });
};

userSchema.methods.resetLoginAttempts = async function () {
  if (this.loginAttempts !== 0 || this.lockUntil) {
    this.loginAttempts = 0;
    this.lockUntil = undefined;
    await this.save({ validateBeforeSave: false });
  }
};
```

Then in `auth.service.login`: if `user.isLocked`, throw a generic 401 (don't reveal the lock); on wrong password call `registerFailedLogin()`; on success call `resetLoginAttempts()`.

> WARNING — Pitfall: Lockout enables a **denial-of-service against a known victim** (an attacker can deliberately lock someone out). Mitigate by locking on the *email+IP* combination, using exponential backoff instead of hard locks, or alerting the user by email — discuss these tradeoffs in interviews.

### Extension C — Refresh-token rotation with reuse detection

Right now `/refresh` issues new tokens but doesn't track them, so a stolen refresh token works until it expires. Production systems **rotate** (issue a new refresh token each time and invalidate the old one) and **detect reuse** (if an already-used token appears again, someone copied it — revoke the whole family).

Add a `refreshTokens` array to the user (or a separate collection) storing **hashes** of issued refresh tokens with metadata:

```js
// src/models/user.model.js (add to schema)
refreshTokens: [
  {
    tokenHash: String,      // sha256 of the issued refresh token
    createdAt: { type: Date, default: Date.now },
    expiresAt: Date,
  },
],
```

On login/refresh, store `sha256(refreshToken)`. On `/refresh`:

1. Verify the JWT signature.
2. Compute `sha256(incoming)` and look for it in `user.refreshTokens`.
3. **If absent** → the token was already rotated out (reuse!). Treat as compromise: clear *all* `refreshTokens` (force re-login everywhere) and return 401.
4. **If present** → remove that entry, push a new one, issue new tokens.

```js
// src/services/auth.service.js  (rotation-aware refresh — sketch)
const crypto = require('node:crypto');

async function refreshWithRotation(refreshToken) {
  if (!refreshToken) throw new AppError('No refresh token provided', 401);

  let payload;
  try { payload = tokenService.verifyRefreshToken(refreshToken); }
  catch { throw new AppError('Invalid or expired refresh token', 401); }

  const user = await User.findById(payload.sub).select('+refreshTokens');
  if (!user) throw new AppError('User no longer exists', 401);

  const incomingHash = crypto.createHash('sha256').update(refreshToken).digest('hex');
  const match = user.refreshTokens.find((t) => t.tokenHash === incomingHash);

  if (!match) {
    // Reuse detected: nuke every session for this user.
    user.refreshTokens = [];
    await user.save({ validateBeforeSave: false });
    throw new AppError('Refresh token reuse detected. Please log in again.', 401);
  }

  // Remove the used token, mint and store a new one.
  user.refreshTokens = user.refreshTokens.filter((t) => t.tokenHash !== incomingHash);
  const tokens = issueTokens(user);
  user.refreshTokens.push({
    tokenHash: crypto.createHash('sha256').update(tokens.refreshToken).digest('hex'),
    expiresAt: new Date(Date.now() + config.cookie.refreshExpiresDays * 86400000),
  });
  await user.save({ validateBeforeSave: false });

  return { user, tokens };
}
```

This also makes **logout** meaningful server-side: remove that token's hash so it can never refresh again.

### Extension D — Input validation with a schema library

Replace the hand-rolled `if (!email)` checks with a validation middleware using **Zod** or **express-validator**, returning structured field-level errors. This centralizes validation and keeps controllers clean.

### Extension E — ES Modules variant

Convert one file to ESM to practice the difference. Set `"type": "module"` in `package.json`, rename `require`/`module.exports`:

```js
// src/utils/AppError.js  (ESM version)
export default class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.statusCode = statusCode;
    this.status = String(statusCode).startsWith('4') ? 'fail' : 'error';
    this.isOperational = true;
    Error.captureStackTrace(this, this.constructor);
  }
}
```

```js
// importing it
import AppError from './utils/AppError.js'; // note: .js extension is REQUIRED in ESM
```

> TIP: In ESM you **must** include the file extension in relative imports (`./foo.js`), and there's no `__dirname`/`__filename` — derive them via `import.meta.url`. These gotchas trip up everyone converting a CommonJS app.

### Challenge checklist

- [ ] Add a `POST /auth/resend-verification` endpoint (rate-limited).
- [ ] Add `GET /users/:id` (admin) and `PATCH /me` (self-update name/email — re-trigger verification on email change).
- [ ] Move the rate-limit store to Redis so it works across instances.
- [ ] Write Supertest integration tests for register → login → me → refresh → logout.
- [ ] Add audit logging (timestamp, IP, user agent) for login/reset events.

---

## Common Errors & Fixes

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| App exits at boot: *"Missing required environment variables"* | `.env` not created or a key is blank | `cp .env.example .env` and fill `MONGODB_URI`, `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET` |
| `MongooseServerSelectionError` / `ECONNREFUSED 127.0.0.1:27017` | MongoDB isn't running or wrong URI | Start `mongod`, or fix the Atlas URI (and whitelist your IP in Atlas) |
| `npm install bcrypt` fails to compile | Missing native build tools | Use `bcryptjs` instead (drop-in: `const bcrypt = require('bcryptjs')`) |
| Login always returns *"Incorrect email or password"* even when correct | Forgot `.select('+password')`, so `user.password` is `undefined` and `bcrypt.compare` fails | Ensure `User.findOne({ email }).select('+password')` in the login service |
| `req.user` is `undefined` in `restrictTo` | `restrictTo` ran before `protect`, or `protect` was omitted | Order middleware as `protect, restrictTo('admin')` |
| `req.cookies` is `undefined` | `cookie-parser` not registered, or registered after the route | `app.use(cookieParser())` before routes in `app.js` |
| Refresh cookie not sent back by the browser/curl | Cookie `path`/`sameSite`/`secure` mismatch; in dev over HTTP `secure:true` blocks it | Keep `secure: config.isProd` (false in dev); use `-b/-c` cookie jar in curl; match `path` on clear |
| `JsonWebTokenError: invalid signature` after restart | Different `JWT_ACCESS_SECRET` than the token was signed with | Keep secrets stable; old tokens are invalid after a secret change (by design) |
| `TokenExpiredError` on `/me` | Access token expired (15 min) | Call `POST /auth/refresh` to get a new access token |
| Duplicate email returns a 500 instead of 400 | Running in dev shows the raw `11000` error; the prod handler maps it | Expected in dev (full error). In prod it becomes a clean 400 via `handleDuplicateKey` |
| `E11000 duplicate key` won't go away even with unique emails | The unique index was built while duplicates existed | Drop the collection/index, ensure no dups, restart so the index rebuilds |
| Error responses are plain HTML, not JSON | `errorHandler` declared with 3 args, so Express doesn't treat it as an error handler | Keep all four params `(err, req, res, next)` |
| `429 Too Many Requests` during testing | Hit the auth rate limit (10/15min) | Wait out the window, or temporarily raise `max` in `rateLimiter.js` for local testing |
| Behind a proxy everyone gets rate-limited together | `trust proxy` not set, so all requests share the proxy's IP | `app.set('trust proxy', 1)` (already in `app.js`) |
| Reset link gives *"Token is invalid or has expired"* immediately | You pasted the **hashed** token, or it expired (10 min), or it was already used | Use the raw token from the email link, request a fresh one if needed |

> BEST PRACTICE: When something "doesn't work," check it **layer by layer** in request order: did the body parse (`req.body`)? did cookies parse (`req.cookies`)? did `protect` set `req.user`? did the service return what you expect? This top-down trace finds 90% of auth bugs in minutes.

---

You now have a complete, production-shaped authentication and authorization system — and, more importantly, you can explain every security decision in it. Build the extensions next: refresh-token rotation and account lockout are exactly the kind of nuance that turns a good interview into a job offer. When you're ready, Project 4 (Real-Time Chat) reuses this exact `protect` middleware to authenticate WebSocket connections.
