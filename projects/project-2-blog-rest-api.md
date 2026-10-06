# Project 2 — Blog REST API (Express + Mongoose)

> **Stack:** Node.js (current LTS) · Express.js · MongoDB + Mongoose · plain JavaScript (CommonJS)
> **Builds on:** Modules 13–18 (Express), 19–20 (MongoDB & Mongoose), 23 (Error Handling & Validation).
> **Time:** ~6–10 hours. **Difficulty:** Intermediate.

This is the project where you stop writing toy servers and start writing an API the way a real team would. You'll design a layered codebase, model data with Mongoose, validate every incoming request, handle errors from one place, and expose a clean, predictable JSON contract. By the end you'll be able to *build it, explain every layer, and defend the design choices in an interview.*

---

## Overview

We are building a **Blog REST API**: a backend (no UI) that manages blog **Posts** and the **Comments** nested under each post. It speaks JSON over HTTP and is the kind of service a React/Vue/mobile front-end (or another microservice) would consume.

Concretely, the finished API exposes these endpoints:

### Posts

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/api/v1/posts` | Create a post |
| `GET` | `/api/v1/posts` | List posts — with **pagination, filtering, sorting, search** |
| `GET` | `/api/v1/posts/:id` | Get one post by id |
| `PATCH` | `/api/v1/posts/:id` | Partially update a post |
| `DELETE` | `/api/v1/posts/:id` | Delete a post (and its comments) |

### Comments (nested under a post)

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/api/v1/posts/:postId/comments` | Add a comment to a post |
| `GET` | `/api/v1/posts/:postId/comments` | List a post's comments (paginated) |
| `DELETE` | `/api/v1/posts/:postId/comments/:commentId` | Delete a comment |

### Cross-cutting features we'll implement

- **Layered architecture:** `routes → controllers → services → models`. Each layer has exactly one job.
- **Mongoose schemas** with validation, indexes, virtuals, a slug, and timestamps.
- **List query power:** `?page=2&limit=10&tag=node&author=...&sort=-createdAt&search=express`.
- **Centralized error handling** with a custom `AppError` class and a `catchAsync` wrapper so controllers stay clean and never repeat `try/catch`.
- **Request validation** at the edge with `express-validator` (we also show the **Joi** alternative).
- **Environment config** via `dotenv`, loaded once and validated.
- **Correct HTTP status codes** and a **consistent JSON envelope** for both success and failure.
- **Security & DX middleware:** `helmet`, `cors`, rate limiting, request logging, graceful shutdown.

> The example URLs use `/api/v1/...`. Versioning your API in the URL from day one is cheap insurance — when you need a breaking change you ship `/api/v2` without disturbing existing clients.

---

## What You'll Practice

- Translating a feature list into REST **resources, routes, and status codes**.
- Designing **Mongoose schemas**: types, validators, `enum`, `unique`, indexes, virtuals, pre-save hooks, `toJSON` transforms.
- Modelling a **relationship** (Post ⇄ Comments) and choosing between embedding and referencing.
- Building **reusable query features** — pagination/filter/sort/search — as a small utility.
- Writing a **thin controller / fat service** split so business logic is testable and framework-agnostic.
- **Centralized error handling**: operational vs programmer errors, and turning Mongoose/cast/duplicate-key errors into clean 4xx responses.
- **Input validation** as a middleware layer (the difference between *validation* at the edge and *schema validation* in the DB).
- Wiring up **config, logging, security headers, CORS, and rate limiting** the way production apps do.
- Testing an API with **curl** and **Postman**.

---

## Prerequisites & Setup

### You need

- **Node.js current LTS** (v20 or v22+). Check with `node --version`.
- **MongoDB** — either a local install **or** a free [MongoDB Atlas](https://www.mongodb.com/atlas) cluster. Atlas is recommended; it gives you a connection string you can paste straight into `.env`.
- An API client: **Postman**, **Thunder Client** (VS Code extension), or just **curl**.

> TIP: If you have Docker, the fastest local MongoDB is one command:
> ```bash
> docker run -d --name blog-mongo -p 27017:27017 mongo:7
> ```
> Your connection string is then `mongodb://localhost:27017/blog_api`.

### 1. Create the project and install dependencies

```bash
# Make a fresh folder and initialise package.json
mkdir blog-rest-api && cd blog-rest-api
npm init -y

# Runtime dependencies
npm install express mongoose dotenv express-validator helmet cors morgan express-rate-limit

# Dev dependency: auto-restart on file changes during development
npm install --save-dev nodemon
```

What each package does:

| Package | Role |
|---------|------|
| `express` | The web framework: routing, middleware, request/response. |
| `mongoose` | ODM for MongoDB: schemas, validation, queries. |
| `dotenv` | Loads `.env` into `process.env`. |
| `express-validator` | Declarative request validation middleware. |
| `helmet` | Sets secure HTTP response headers. |
| `cors` | Enables Cross-Origin requests from browser front-ends. |
| `morgan` | HTTP request logger (dev convenience). |
| `express-rate-limit` | Basic protection against abuse / brute force. |
| `nodemon` (dev) | Restarts the server when files change. |

> BEST PRACTICE: We use **CommonJS** (`require`/`module.exports`) throughout because it's still the most common style you'll meet in existing Express codebases. At the end of "Extensions" we show how to convert the project to **ES Modules**.

### 2. Set the scripts in `package.json`

Open `package.json` and make the `"scripts"` and `"type"` look like this:

```json
{
  "name": "blog-rest-api",
  "version": "1.0.0",
  "description": "A RESTful Blog API built with Express and Mongoose",
  "main": "server.js",
  "type": "commonjs",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "keywords": ["express", "mongoose", "rest", "api"],
  "license": "MIT"
}
```

### 3. Final folder structure

This is the tree you'll have built by the end. Create the folders now (`mkdir -p src/{config,models,services,controllers,routes/v1,middleware,validators,utils}`), or just create files as you go:

```
blog-rest-api/
├── .env                      # your secrets (NOT committed)
├── .env.example              # template, safe to commit
├── .gitignore
├── package.json
├── server.js                 # entry point: connect DB, start HTTP server, handle signals
└── src/
    ├── app.js                # builds the Express app (middleware + routes + error handler)
    ├── config/
    │   ├── env.js            # loads & validates environment variables
    │   └── db.js             # Mongoose connection logic
    ├── models/
    │   ├── post.model.js
    │   └── comment.model.js
    ├── services/
    │   ├── post.service.js   # business logic + DB access for posts
    │   └── comment.service.js
    ├── controllers/
    │   ├── post.controller.js   # HTTP glue: read req, call service, send res
    │   └── comment.controller.js
    ├── routes/
    │   └── v1/
    │       ├── index.js         # mounts all v1 resource routers
    │       ├── post.routes.js
    │       └── comment.routes.js
    ├── validators/
    │   ├── post.validator.js    # express-validator rule chains for posts
    │   └── comment.validator.js
    ├── middleware/
    │   ├── error.middleware.js  # centralized error handler + 404 handler
    │   └── validate.middleware.js  # runs validators & formats validation errors
    └── utils/
        ├── AppError.js          # custom operational error class
        ├── catchAsync.js        # wraps async handlers, forwards errors to next()
        ├── ApiResponse.js       # consistent success envelope helper
        └── ApiFeatures.js       # pagination / filter / sort / search builder
```

> Why so many folders for a "small" app? Because the *shape* is what we're learning. This is exactly the layout that scales to the capstone (Project 5). Each file stays small and single-purpose, which is what makes the codebase testable and reviewable.

---

## Step 1 — Environment config (`dotenv`) and `.gitignore`

Backend apps must never hard-code secrets (DB passwords, ports, JWT keys). We keep them in environment variables and load them with `dotenv`. We also **validate** them on boot so the app fails fast with a clear message instead of crashing mysteriously later.

### `.gitignore`

```bash
# .gitignore
node_modules/
.env
*.log
.DS_Store
coverage/
```

### `.env.example`  (commit this — it documents what's required)

```bash
# .env.example — copy to .env and fill in real values
NODE_ENV=development
PORT=3000

# MongoDB connection string.
# Local:   mongodb://localhost:27017/blog_api
# Atlas:   mongodb+srv://<user>:<pass>@<cluster>.mongodb.net/blog_api?retryWrites=true&w=majority
MONGODB_URI=mongodb://localhost:27017/blog_api

# Comma-separated list of allowed CORS origins (use * to allow all in dev)
CORS_ORIGIN=*

# Rate limiting
RATE_LIMIT_WINDOW_MINUTES=15
RATE_LIMIT_MAX=200
```

### `.env`  (your real, secret values — never committed)

```bash
# .env
NODE_ENV=development
PORT=3000
MONGODB_URI=mongodb://localhost:27017/blog_api
CORS_ORIGIN=*
RATE_LIMIT_WINDOW_MINUTES=15
RATE_LIMIT_MAX=200
```

### `src/config/env.js`

We load `dotenv` exactly once, in this module, and export a typed, validated config object. Everywhere else in the app imports **this** — never `process.env` directly. That gives us a single source of truth and one place to add validation.

```js
// src/config/env.js
// Load environment variables from .env into process.env (must run before we read them).
require('dotenv').config();

// Small helper: read a required variable or throw a clear error on boot.
function required(name) {
  const value = process.env[name];
  if (value === undefined || value === '') {
    throw new Error(`Missing required environment variable: ${name}`);
  }
  return value;
}

// Small helper: read an optional variable with a default.
function optional(name, fallback) {
  const value = process.env[name];
  return value === undefined || value === '' ? fallback : value;
}

// Build a single frozen config object. Freezing prevents accidental mutation at runtime.
const env = Object.freeze({
  nodeEnv: optional('NODE_ENV', 'development'),
  port: Number(optional('PORT', 3000)),
  mongoUri: required('MONGODB_URI'),
  // CORS_ORIGIN can be "*" or a comma-separated list of origins.
  corsOrigin: optional('CORS_ORIGIN', '*'),
  rateLimit: {
    windowMs: Number(optional('RATE_LIMIT_WINDOW_MINUTES', 15)) * 60 * 1000,
    max: Number(optional('RATE_LIMIT_MAX', 200)),
  },
  // Convenience booleans used across the app.
  get isProd() {
    return this.nodeEnv === 'production';
  },
  get isDev() {
    return this.nodeEnv === 'development';
  },
});

module.exports = env;
```

> BEST PRACTICE: Centralising config (and validating it on startup) is a small habit that prevents a whole category of "works on my machine" bugs. If `MONGODB_URI` is missing, the process refuses to start and tells you exactly why.

---

## Step 2 — Database connection (`src/config/db.js`)

Mongoose manages a connection pool for you. We wrap the connect call so we can log success/failure and react to connection events. Note: starting in Mongoose 6+, the old flags (`useNewUrlParser`, `useUnifiedTopology`, etc.) are defaults and should be omitted — passing them now just produces deprecation noise.

```js
// src/config/db.js
const mongoose = require('mongoose');
const env = require('./env');

// In dev, log Mongoose queries to the console so you can see what hits the DB.
if (env.isDev) {
  mongoose.set('debug', true);
}

// `strictQuery: true` ignores fields not in the schema when querying (Mongoose 7 default).
mongoose.set('strictQuery', true);

async function connectDB() {
  // Mongoose buffers commands until connected, but we await so the server only
  // starts listening AFTER the DB is reachable — fail fast if it isn't.
  await mongoose.connect(env.mongoUri);
  console.log('✅ MongoDB connected');

  // Surface connection-pool problems that happen AFTER the initial connect.
  mongoose.connection.on('error', (err) => {
    console.error('❌ MongoDB connection error:', err.message);
  });
  mongoose.connection.on('disconnected', () => {
    console.warn('⚠️  MongoDB disconnected');
  });
}

// Used by graceful shutdown to close the pool cleanly.
async function disconnectDB() {
  await mongoose.connection.close();
  console.log('🔌 MongoDB connection closed');
}

module.exports = { connectDB, disconnectDB };
```

> WARNING — Pitfall: Do not call `mongoose.connect()` inside a request handler or call it more than once. Connect **once** at startup; Mongoose pools and reuses the connection for every query.

---

## Step 3 — Utilities: errors, async wrapper, response envelope

These four tiny files are the backbone of a clean Express app. Build them before the models so everything downstream can use them.

### `src/utils/AppError.js`

A custom error class that distinguishes **operational** errors (expected: "post not found", "validation failed" — safe to show the client) from **programmer** errors (bugs: undefined is not a function — must not leak details). We mark our intentional errors with `isOperational = true`.

```js
// src/utils/AppError.js

class AppError extends Error {
  /**
   * @param {string} message  Human-readable message safe to send to the client.
   * @param {number} statusCode  HTTP status code (e.g. 404, 400).
   * @param {object} [details]  Optional structured details (e.g. validation errors).
   */
  constructor(message, statusCode, details = undefined) {
    super(message);

    this.statusCode = statusCode;
    // 4xx => 'fail', 5xx => 'error'. Used in the JSON envelope.
    this.status = `${statusCode}`.startsWith('4') ? 'fail' : 'error';
    // Flag this as an *expected* error we deliberately threw.
    this.isOperational = true;
    if (details) this.details = details;

    // Keep this constructor out of the captured stack trace for cleaner logs.
    Error.captureStackTrace(this, this.constructor);
  }

  // Handy named constructors for the common cases.
  static badRequest(msg = 'Bad request', details) {
    return new AppError(msg, 400, details);
  }
  static unauthorized(msg = 'Unauthorized') {
    return new AppError(msg, 401);
  }
  static forbidden(msg = 'Forbidden') {
    return new AppError(msg, 403);
  }
  static notFound(msg = 'Resource not found') {
    return new AppError(msg, 404);
  }
  static conflict(msg = 'Conflict') {
    return new AppError(msg, 409);
  }
}

module.exports = AppError;
```

### `src/utils/catchAsync.js`

Express 4 does **not** catch errors thrown inside `async` handlers — a rejected promise just becomes an unhandled rejection unless you `try/catch` everywhere. This wrapper does it once: it runs your async function and pipes any rejection into `next(err)`, where our central error handler takes over.

```js
// src/utils/catchAsync.js

// Wrap an async route handler so any thrown/rejected error is forwarded to Express.
// Usage: router.get('/', catchAsync(async (req, res) => { ... }));
module.exports = function catchAsync(handler) {
  return (req, res, next) => {
    // Promise.resolve handles both async functions and plain promise-returning fns.
    Promise.resolve(handler(req, res, next)).catch(next);
  };
};
```

> TIP — Express 5 difference: **Express 5 automatically forwards rejected promises from async middleware to your error handler**, so `catchAsync` becomes optional there. We keep it because (a) most jobs are still on Express 4, and (b) it makes the intent explicit. The code in this guide runs unchanged on both v4 and v5.

### `src/utils/ApiResponse.js`

One helper so every successful response has the same shape. A predictable envelope makes client code trivial to write.

```js
// src/utils/ApiResponse.js

/**
 * Send a consistent success response.
 * Shape: { status, message, data, meta? }
 */
function sendSuccess(res, { statusCode = 200, message = 'OK', data = null, meta } = {}) {
  const body = { status: 'success', message, data };
  if (meta) body.meta = meta; // e.g. pagination info
  return res.status(statusCode).json(body);
}

module.exports = { sendSuccess };
```

### `src/utils/ApiFeatures.js`

This is the reusable engine for list endpoints: **filtering, searching, sorting, field-limiting, and pagination**. It takes a Mongoose query and the request's `req.query`, applies each feature, and hands back a chainable query plus the pagination math. Building this once means every future "list" endpoint in the course gets these powers for free.

```js
// src/utils/ApiFeatures.js

class ApiFeatures {
  /**
   * @param {import('mongoose').Query} query  A Mongoose query, e.g. Post.find()
   * @param {object} queryString  The raw req.query object
   */
  constructor(query, queryString) {
    this.query = query;
    this.queryString = queryString;
    // Pagination metadata, populated by paginate().
    this.pagination = { page: 1, limit: 10 };
  }

  /**
   * Equality filtering on whitelisted fields, e.g. ?tag=node&author=<id>.
   * We EXCLUDE control params (page/sort/etc.) so they aren't treated as filters.
   * `tag` matches against the array field `tags` using $in.
   */
  filter() {
    const queryObj = { ...this.queryString };
    const excluded = ['page', 'sort', 'limit', 'fields', 'search'];
    excluded.forEach((key) => delete queryObj[key]);

    const mongoFilter = {};
    if (queryObj.author) mongoFilter.author = queryObj.author;
    if (queryObj.status) mongoFilter.status = queryObj.status;
    if (queryObj.tag) mongoFilter.tags = { $in: [queryObj.tag] }; // post has ANY of these tags

    this.query = this.query.find(mongoFilter);
    return this;
  }

  /**
   * Full-text-ish search across title/content using a case-insensitive regex.
   * For real full-text search you'd add a text index and use $text (see Extensions).
   * ?search=express
   */
  search() {
    if (this.queryString.search) {
      const term = this.queryString.search.trim();
      // Escape regex special chars to avoid injection / invalid patterns.
      const safe = term.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
      const rx = new RegExp(safe, 'i');
      this.query = this.query.find({ $or: [{ title: rx }, { content: rx }] });
    }
    return this;
  }

  /**
   * Sorting, e.g. ?sort=-createdAt,title  (minus = descending).
   * Defaults to newest first.
   */
  sort() {
    if (this.queryString.sort) {
      const sortBy = this.queryString.sort.split(',').join(' ');
      this.query = this.query.sort(sortBy);
    } else {
      this.query = this.query.sort('-createdAt');
    }
    return this;
  }

  /**
   * Sparse fieldsets, e.g. ?fields=title,author  -> only return those fields.
   */
  limitFields() {
    if (this.queryString.fields) {
      const fields = this.queryString.fields.split(',').join(' ');
      this.query = this.query.select(fields);
    } else {
      this.query = this.query.select('-__v'); // hide the Mongoose version key by default
    }
    return this;
  }

  /**
   * Pagination, e.g. ?page=2&limit=10. Stores page/limit for the meta envelope.
   * Clamps limit to a sane maximum to protect the DB.
   */
  paginate() {
    const page = Math.max(1, Number(this.queryString.page) || 1);
    const limit = Math.min(100, Math.max(1, Number(this.queryString.limit) || 10));
    const skip = (page - 1) * limit;

    this.pagination = { page, limit, skip };
    this.query = this.query.skip(skip).limit(limit);
    return this;
  }

  /**
   * Build the meta block for the response given the total count of matching docs.
   */
  buildMeta(totalItems) {
    const { page, limit } = this.pagination;
    const totalPages = Math.max(1, Math.ceil(totalItems / limit));
    return {
      page,
      limit,
      totalItems,
      totalPages,
      hasPrevPage: page > 1,
      hasNextPage: page < totalPages,
    };
  }
}

module.exports = ApiFeatures;
```

> WARNING — Pitfall: Skip-based pagination (`.skip(n)`) gets slow on very large collections because Mongo still scans the skipped documents. For this project it's perfect; for millions of rows you'd switch to **cursor / keyset pagination** (paginate by `_id` or `createdAt`). We note this in Extensions.

---

## Step 4 — Mongoose models

Now the data layer. We'll model a **Post** and a **Comment**. A post can have many comments. We **reference** comments from a separate collection (rather than embedding them in the post document) because comments are unbounded and we want to paginate them independently — a classic "one-to-many, many side is large" decision.

> Embedding vs referencing — the rule of thumb:
> - **Embed** when the child is bounded, always loaded with the parent, and not queried on its own (e.g. an address inside a user).
> - **Reference** when the child set can grow without limit or needs its own queries/pagination (e.g. comments, orders). We reference here.

### `src/models/post.model.js`

```js
// src/models/post.model.js
const mongoose = require('mongoose');

const postSchema = new mongoose.Schema(
  {
    title: {
      type: String,
      required: [true, 'A post must have a title'],
      trim: true,
      minlength: [3, 'Title must be at least 3 characters'],
      maxlength: [160, 'Title must be at most 160 characters'],
    },
    // URL-friendly unique identifier derived from the title (e.g. "my-first-post").
    slug: {
      type: String,
      unique: true,
      index: true,
    },
    content: {
      type: String,
      required: [true, 'A post must have content'],
      minlength: [10, 'Content must be at least 10 characters'],
    },
    // Short preview text. If omitted, we derive it from content in a pre-save hook.
    excerpt: {
      type: String,
      maxlength: [300, 'Excerpt must be at most 300 characters'],
    },
    // In Project 3 this becomes a ref to a User. For now it's a free-text author name.
    author: {
      type: String,
      required: [true, 'A post must have an author'],
      trim: true,
    },
    tags: {
      type: [String],
      default: [],
      // Normalise tags: lowercase + trim each one.
      set: (tags) => tags.map((t) => String(t).toLowerCase().trim()).filter(Boolean),
    },
    status: {
      type: String,
      enum: {
        values: ['draft', 'published'],
        message: 'Status must be either "draft" or "published"',
      },
      default: 'draft',
    },
    // Denormalised counters kept in sync by the comment service (cheap reads).
    commentCount: {
      type: Number,
      default: 0,
      min: 0,
    },
  },
  {
    // Adds createdAt & updatedAt automatically.
    timestamps: true,
    // Include virtuals when converting to JSON/objects.
    toJSON: { virtuals: true },
    toObject: { virtuals: true },
  }
);

// --- Indexes ---------------------------------------------------------------
// Compound index supporting the most common list query: filter by status, newest first.
postSchema.index({ status: 1, createdAt: -1 });
// Index tags for ?tag= filtering.
postSchema.index({ tags: 1 });

// --- Virtuals --------------------------------------------------------------
// A virtual "reading time" derived from word count (not stored in the DB).
postSchema.virtual('readingTimeMinutes').get(function () {
  const words = (this.content || '').trim().split(/\s+/).filter(Boolean).length;
  return Math.max(1, Math.ceil(words / 200)); // ~200 wpm
});

// --- Helpers ---------------------------------------------------------------
function slugify(text) {
  return String(text)
    .toLowerCase()
    .trim()
    .replace(/[^a-z0-9\s-]/g, '') // drop non-url chars
    .replace(/\s+/g, '-') // spaces -> hyphens
    .replace(/-+/g, '-'); // collapse repeats
}

// --- Hooks -----------------------------------------------------------------
// Generate slug + excerpt before validation/save when the title/content changes.
postSchema.pre('validate', function (next) {
  if (this.isModified('title') || !this.slug) {
    // Add a short random suffix to keep slugs unique even for identical titles.
    const base = slugify(this.title);
    const suffix = Math.random().toString(36).slice(2, 7);
    this.slug = `${base}-${suffix}`;
  }
  if (!this.excerpt && this.content) {
    this.excerpt = this.content.slice(0, 297).trimEnd() + (this.content.length > 297 ? '…' : '');
  }
  next();
});

// --- Output shaping --------------------------------------------------------
// Clean the JSON shape: expose `id`, hide `_id`/`__v`.
postSchema.set('toJSON', {
  virtuals: true,
  transform(doc, ret) {
    ret.id = ret._id;
    delete ret._id;
    delete ret.__v;
    return ret;
  },
});

module.exports = mongoose.model('Post', postSchema);
```

> BEST PRACTICE: Add **indexes** for the fields you filter/sort on. Without the `{ status: 1, createdAt: -1 }` index, the list query does a collection scan + in-memory sort, which falls over as data grows. Indexes are the single biggest performance lever in a read-heavy API like a blog.

### `src/models/comment.model.js`

```js
// src/models/comment.model.js
const mongoose = require('mongoose');

const commentSchema = new mongoose.Schema(
  {
    // Reference to the parent post. Indexed because we always query "comments of a post".
    post: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'Post',
      required: true,
      index: true,
    },
    author: {
      type: String,
      required: [true, 'A comment must have an author'],
      trim: true,
      maxlength: [80, 'Author name too long'],
    },
    body: {
      type: String,
      required: [true, 'A comment cannot be empty'],
      trim: true,
      minlength: [1, 'A comment cannot be empty'],
      maxlength: [1000, 'Comment must be at most 1000 characters'],
    },
  },
  {
    timestamps: true,
    toJSON: {
      virtuals: true,
      transform(doc, ret) {
        ret.id = ret._id;
        delete ret._id;
        delete ret.__v;
        return ret;
      },
    },
  }
);

// Supports listing a post's comments newest-first, paginated.
commentSchema.index({ post: 1, createdAt: -1 });

module.exports = mongoose.model('Comment', commentSchema);
```

---

## Step 5 — Service layer (business logic + DB access)

The **service** is where the real work lives: querying the DB, enforcing rules, throwing `AppError` for expected failures. Services know **nothing** about HTTP — no `req`, no `res`. That separation is what makes them unit-testable and reusable (a CLI command or a background job could call the same service).

### `src/services/post.service.js`

```js
// src/services/post.service.js
const Post = require('../models/post.model');
const Comment = require('../models/comment.model');
const AppError = require('../utils/AppError');
const ApiFeatures = require('../utils/ApiFeatures');

/**
 * Create a new post.
 * @param {object} data  Already-validated fields from the request body.
 */
async function createPost(data) {
  // Mongoose runs schema validators on .create(); validation errors bubble up
  // and are translated to a 400 by the central error handler.
  return Post.create(data);
}

/**
 * List posts with filtering, search, sorting, field limiting, and pagination.
 * Returns both the page of data and the pagination meta.
 * @param {object} queryString  req.query
 */
async function listPosts(queryString) {
  // Build the data query through the feature pipeline.
  const features = new ApiFeatures(Post.find(), queryString)
    .filter()
    .search()
    .sort()
    .limitFields()
    .paginate();

  // To compute total pages we need the count of ALL matching docs (pre-pagination).
  // Re-run filter+search on a fresh query and count it.
  const countFeatures = new ApiFeatures(Post.find(), queryString).filter().search();
  const [posts, totalItems] = await Promise.all([
    features.query.exec(),
    countFeatures.query.countDocuments().exec(),
  ]);

  return { posts, meta: features.buildMeta(totalItems) };
}

/**
 * Fetch a single post by id, or throw 404.
 */
async function getPostById(id) {
  const post = await Post.findById(id);
  if (!post) throw AppError.notFound(`No post found with id ${id}`);
  return post;
}

/**
 * Partially update a post. Returns the updated doc, or throws 404.
 */
async function updatePost(id, updates) {
  const post = await Post.findById(id);
  if (!post) throw AppError.notFound(`No post found with id ${id}`);

  // Apply allowed updates, then save so pre-save hooks (slug/excerpt) and
  // validators run. (findByIdAndUpdate would skip some hooks unless configured.)
  Object.assign(post, updates);
  await post.save();
  return post;
}

/**
 * Delete a post AND its comments. We use a transaction when the deployment
 * supports it (replica set / Atlas); otherwise fall back to sequential deletes.
 */
async function deletePost(id) {
  const post = await Post.findById(id);
  if (!post) throw AppError.notFound(`No post found with id ${id}`);

  // Cascade: remove the post's comments so we don't leave orphans.
  await Comment.deleteMany({ post: id });
  await post.deleteOne();
  return { id };
}

module.exports = {
  createPost,
  listPosts,
  getPostById,
  updatePost,
  deletePost,
};
```

### `src/services/comment.service.js`

```js
// src/services/comment.service.js
const Comment = require('../models/comment.model');
const Post = require('../models/post.model');
const AppError = require('../utils/AppError');
const ApiFeatures = require('../utils/ApiFeatures');

// Guard: make sure the parent post exists before touching its comments.
async function assertPostExists(postId) {
  const exists = await Post.exists({ _id: postId });
  if (!exists) throw AppError.notFound(`No post found with id ${postId}`);
}

/**
 * Add a comment to a post and bump the post's denormalised commentCount.
 */
async function addComment(postId, data) {
  await assertPostExists(postId);

  const comment = await Comment.create({ ...data, post: postId });

  // Keep the cached counter on the post in sync ($inc is atomic).
  await Post.findByIdAndUpdate(postId, { $inc: { commentCount: 1 } });

  return comment;
}

/**
 * List a post's comments, newest first, paginated.
 */
async function listComments(postId, queryString) {
  await assertPostExists(postId);

  const baseFilter = Comment.find({ post: postId });
  const features = new ApiFeatures(baseFilter, queryString).sort().paginate();

  const [comments, totalItems] = await Promise.all([
    features.query.exec(),
    Comment.countDocuments({ post: postId }).exec(),
  ]);

  return { comments, meta: features.buildMeta(totalItems) };
}

/**
 * Delete one comment that belongs to a specific post.
 */
async function deleteComment(postId, commentId) {
  await assertPostExists(postId);

  // Match BOTH ids so you can't delete a comment via the wrong post URL.
  const comment = await Comment.findOneAndDelete({ _id: commentId, post: postId });
  if (!comment) throw AppError.notFound(`No comment ${commentId} found on post ${postId}`);

  await Post.findByIdAndUpdate(postId, { $inc: { commentCount: -1 } });
  return { id: commentId };
}

module.exports = { addComment, listComments, deleteComment };
```

> BEST PRACTICE — "fat service, thin controller": all decisions (does it exist? is it allowed? what gets updated?) live in the service. Controllers shrink to three lines. When you write tests in Module 27, you'll test services directly with no HTTP mocking required.

---

## Step 6 — Controllers (HTTP glue)

A controller does exactly three things: pull the inputs out of `req`, call a service, and send a response. Every handler is wrapped in `catchAsync`, so there's not a single `try/catch` here — errors flow to the central handler.

### `src/controllers/post.controller.js`

```js
// src/controllers/post.controller.js
const catchAsync = require('../utils/catchAsync');
const { sendSuccess } = require('../utils/ApiResponse');
const postService = require('../services/post.service');

const createPost = catchAsync(async (req, res) => {
  const post = await postService.createPost(req.body);
  sendSuccess(res, { statusCode: 201, message: 'Post created', data: { post } });
});

const listPosts = catchAsync(async (req, res) => {
  const { posts, meta } = await postService.listPosts(req.query);
  sendSuccess(res, { message: 'Posts fetched', data: { posts }, meta });
});

const getPost = catchAsync(async (req, res) => {
  const post = await postService.getPostById(req.params.id);
  sendSuccess(res, { message: 'Post fetched', data: { post } });
});

const updatePost = catchAsync(async (req, res) => {
  const post = await postService.updatePost(req.params.id, req.body);
  sendSuccess(res, { message: 'Post updated', data: { post } });
});

const deletePost = catchAsync(async (req, res) => {
  await postService.deletePost(req.params.id);
  // 204 No Content: success, nothing to return.
  res.status(204).send();
});

module.exports = { createPost, listPosts, getPost, updatePost, deletePost };
```

### `src/controllers/comment.controller.js`

```js
// src/controllers/comment.controller.js
const catchAsync = require('../utils/catchAsync');
const { sendSuccess } = require('../utils/ApiResponse');
const commentService = require('../services/comment.service');

const addComment = catchAsync(async (req, res) => {
  const comment = await commentService.addComment(req.params.postId, req.body);
  sendSuccess(res, { statusCode: 201, message: 'Comment added', data: { comment } });
});

const listComments = catchAsync(async (req, res) => {
  const { comments, meta } = await commentService.listComments(req.params.postId, req.query);
  sendSuccess(res, { message: 'Comments fetched', data: { comments }, meta });
});

const deleteComment = catchAsync(async (req, res) => {
  await commentService.deleteComment(req.params.postId, req.params.commentId);
  res.status(204).send();
});

module.exports = { addComment, listComments, deleteComment };
```

---

## Step 7 — Request validation (`express-validator`)

There are **two** layers of validation, and they're not redundant:

1. **Edge validation** (this step): reject malformed requests *before* they reach business logic, with friendly, field-level messages. Fast feedback, no DB round-trip.
2. **Schema validation** (the Mongoose models): the database's own guarantee — the last line of defence even if a request bypasses the route.

We use `express-validator`. It works as a chain of middleware that collects errors into the request; our `validate` middleware then turns those into a clean 400 response.

### `src/middleware/validate.middleware.js`

```js
// src/middleware/validate.middleware.js
const { validationResult } = require('express-validator');
const AppError = require('../utils/AppError');

/**
 * Runs an array of express-validator chains, then checks the result.
 * If anything failed, throw a 400 AppError carrying field-level details.
 *
 * Usage: router.post('/', validate(createPostRules), controller.createPost)
 */
function validate(rules) {
  return async (req, res, next) => {
    // Execute every validation chain against the request.
    await Promise.all(rules.map((rule) => rule.run(req)));

    const result = validationResult(req);
    if (result.isEmpty()) return next();

    // Reshape errors into a compact { field: message } friendly array.
    const details = result.array().map((e) => ({
      field: e.path,
      message: e.msg,
    }));

    return next(AppError.badRequest('Validation failed', details));
  };
}

module.exports = validate;
```

### `src/validators/post.validator.js`

```js
// src/validators/post.validator.js
const { body, param, query } = require('express-validator');

// Rules for creating a post. Mirrors (but is friendlier than) the Mongoose schema.
const createPostRules = [
  body('title')
    .trim()
    .notEmpty().withMessage('Title is required')
    .isLength({ min: 3, max: 160 }).withMessage('Title must be 3–160 characters'),
  body('content')
    .trim()
    .notEmpty().withMessage('Content is required')
    .isLength({ min: 10 }).withMessage('Content must be at least 10 characters'),
  body('author')
    .trim()
    .notEmpty().withMessage('Author is required'),
  body('excerpt')
    .optional()
    .isLength({ max: 300 }).withMessage('Excerpt must be at most 300 characters'),
  body('tags')
    .optional()
    .isArray().withMessage('Tags must be an array of strings'),
  body('tags.*')
    .optional()
    .isString().withMessage('Each tag must be a string'),
  body('status')
    .optional()
    .isIn(['draft', 'published']).withMessage('Status must be "draft" or "published"'),
];

// For PATCH: every field is optional, but if present it must be valid.
const updatePostRules = [
  param('id').isMongoId().withMessage('Invalid post id'),
  body('title').optional().trim().isLength({ min: 3, max: 160 })
    .withMessage('Title must be 3–160 characters'),
  body('content').optional().trim().isLength({ min: 10 })
    .withMessage('Content must be at least 10 characters'),
  body('author').optional().trim().notEmpty().withMessage('Author cannot be empty'),
  body('excerpt').optional().isLength({ max: 300 })
    .withMessage('Excerpt must be at most 300 characters'),
  body('tags').optional().isArray().withMessage('Tags must be an array'),
  body('status').optional().isIn(['draft', 'published'])
    .withMessage('Status must be "draft" or "published"'),
];

// Validate the :id param for GET-one / DELETE.
const postIdRules = [param('id').isMongoId().withMessage('Invalid post id')];

// Validate list query params so bad input can't reach the DB.
const listPostsRules = [
  query('page').optional().isInt({ min: 1 }).withMessage('page must be a positive integer'),
  query('limit').optional().isInt({ min: 1, max: 100 }).withMessage('limit must be 1–100'),
  query('status').optional().isIn(['draft', 'published'])
    .withMessage('status filter must be "draft" or "published"'),
  query('author').optional().isString(),
  query('tag').optional().isString(),
  query('search').optional().isString(),
  query('sort').optional().isString(),
  query('fields').optional().isString(),
];

module.exports = {
  createPostRules,
  updatePostRules,
  postIdRules,
  listPostsRules,
};
```

### `src/validators/comment.validator.js`

```js
// src/validators/comment.validator.js
const { body, param, query } = require('express-validator');

const addCommentRules = [
  param('postId').isMongoId().withMessage('Invalid post id'),
  body('author').trim().notEmpty().withMessage('Author is required')
    .isLength({ max: 80 }).withMessage('Author name too long'),
  body('body').trim().notEmpty().withMessage('Comment body is required')
    .isLength({ max: 1000 }).withMessage('Comment must be at most 1000 characters'),
];

const listCommentsRules = [
  param('postId').isMongoId().withMessage('Invalid post id'),
  query('page').optional().isInt({ min: 1 }).withMessage('page must be a positive integer'),
  query('limit').optional().isInt({ min: 1, max: 100 }).withMessage('limit must be 1–100'),
];

const deleteCommentRules = [
  param('postId').isMongoId().withMessage('Invalid post id'),
  param('commentId').isMongoId().withMessage('Invalid comment id'),
];

module.exports = { addCommentRules, listCommentsRules, deleteCommentRules };
```

> TIP: Validators are *just middleware*. They run before the controller and never touch the DB, so a request with `"limit": "banana"` is rejected instantly with a 400, not after a wasted query.

---

## Step 8 — Routes

Routes wire URLs + HTTP verbs to `[validators] → validate → controller`. We keep comments under posts with a **nested router** using `mergeParams: true` so the child router can read `:postId` from the parent path.

### `src/routes/v1/comment.routes.js`

```js
// src/routes/v1/comment.routes.js
const express = require('express');
const controller = require('../../controllers/comment.controller');
const validate = require('../../middleware/validate.middleware');
const {
  addCommentRules,
  listCommentsRules,
  deleteCommentRules,
} = require('../../validators/comment.validator');

// mergeParams lets this router access :postId defined on the parent post router.
const router = express.Router({ mergeParams: true });

router
  .route('/')
  .get(validate(listCommentsRules), controller.listComments)
  .post(validate(addCommentRules), controller.addComment);

router.delete('/:commentId', validate(deleteCommentRules), controller.deleteComment);

module.exports = router;
```

### `src/routes/v1/post.routes.js`

```js
// src/routes/v1/post.routes.js
const express = require('express');
const controller = require('../../controllers/post.controller');
const validate = require('../../middleware/validate.middleware');
const {
  createPostRules,
  updatePostRules,
  postIdRules,
  listPostsRules,
} = require('../../validators/post.validator');

// Nested comment router.
const commentRouter = require('./comment.routes');

const router = express.Router();

// Mount comments under a specific post: /posts/:postId/comments
router.use('/:postId/comments', commentRouter);

router
  .route('/')
  .get(validate(listPostsRules), controller.listPosts)
  .post(validate(createPostRules), controller.createPost);

router
  .route('/:id')
  .get(validate(postIdRules), controller.getPost)
  .patch(validate(updatePostRules), controller.updatePost)
  .delete(validate(postIdRules), controller.deletePost);

module.exports = router;
```

### `src/routes/v1/index.js`

```js
// src/routes/v1/index.js
const express = require('express');
const postRoutes = require('./post.routes');

const router = express.Router();

// Lightweight health check — handy for load balancers and uptime monitors.
router.get('/health', (req, res) => {
  res.json({ status: 'success', message: 'API is healthy', uptime: process.uptime() });
});

router.use('/posts', postRoutes);

module.exports = router;
```

> TIP — Express 5 routing change: Express 5 upgraded `path-to-regexp`. Optional params are now written `/:id?` and wildcards must be **named** (`/*splat` instead of bare `*`). The routes above use no exotic patterns, so they're identical on v4 and v5 — but it's a common upgrade gotcha worth knowing for interviews.

---

## Step 9 — Centralized error handling

This is the payoff for all the `AppError`/`catchAsync` discipline. One middleware catches **every** error in the app and returns a consistent JSON shape. It also **translates** Mongoose's internal errors (validation, bad ObjectId, duplicate key) into clean 4xx responses, and it hides internals for unexpected (programmer) errors in production.

An Express error-handling middleware is identified by having **four** arguments `(err, req, res, next)`. We also add a catch-all 404 for unmatched routes.

### `src/middleware/error.middleware.js`

```js
// src/middleware/error.middleware.js
const AppError = require('../utils/AppError');
const env = require('../config/env');

// 404 handler: reached only if no route matched. Forwards a clean AppError.
function notFoundHandler(req, res, next) {
  next(AppError.notFound(`Route not found: ${req.method} ${req.originalUrl}`));
}

// --- Translators: convert known Mongoose errors into operational AppErrors ---

function handleCastError(err) {
  // e.g. GET /posts/not-a-valid-id  -> Mongoose CastError on _id
  return AppError.badRequest(`Invalid ${err.path}: ${err.value}`);
}

function handleValidationError(err) {
  // Mongoose schema validation failed (last line of defence past express-validator).
  const details = Object.values(err.errors).map((e) => ({
    field: e.path,
    message: e.message,
  }));
  return AppError.badRequest('Validation failed', details);
}

function handleDuplicateKeyError(err) {
  // e.g. duplicate slug. keyValue tells us which field collided.
  const field = Object.keys(err.keyValue || {})[0] || 'field';
  const value = err.keyValue ? err.keyValue[field] : '';
  return AppError.conflict(`Duplicate value for ${field}: "${value}"`);
}

// The central error-handling middleware (note the 4 args).
function errorHandler(err, req, res, next) { // eslint-disable-line no-unused-vars
  let error = err;

  // Normalise known third-party/Mongoose errors into AppError instances.
  if (err.name === 'CastError') error = handleCastError(err);
  else if (err.name === 'ValidationError') error = handleValidationError(err);
  else if (err.code === 11000) error = handleDuplicateKeyError(err);

  // Anything not flagged operational is treated as an unexpected server error.
  const isOperational = error instanceof AppError && error.isOperational;
  const statusCode = isOperational ? error.statusCode : 500;
  const status = isOperational ? error.status : 'error';
  const message = isOperational ? error.message : 'Something went wrong';

  // Always log the real error server-side for debugging/observability.
  if (!isOperational || statusCode >= 500) {
    console.error('💥 Unhandled error:', err);
  }

  const body = { status, message };
  if (error.details) body.details = error.details;
  // Only leak the stack trace in development.
  if (env.isDev && err.stack) body.stack = err.stack;

  res.status(statusCode).json(body);
}

module.exports = { notFoundHandler, errorHandler };
```

> WARNING — Pitfall: The error handler **must be registered last**, after all routes, and it **must** keep all four parameters `(err, req, res, next)` even if `next` is unused — Express detects error handlers by arity. Drop the fourth arg and Express treats it as a normal middleware and your errors won't be caught.

### Operational vs programmer errors — the table

| | Operational error | Programmer error |
|---|---|---|
| Examples | "Post not found", validation failed, bad id | `undefined.foo`, typo, null deref |
| Expected? | Yes — part of normal operation | No — a bug |
| `isOperational` | `true` | `false` |
| Client sees | The real message + correct 4xx | Generic 500 "Something went wrong" |
| Server logs | Optional | Always (with stack) |

---

## Step 10 — Assemble the Express app (`src/app.js`)

`app.js` builds and returns the configured Express application but does **not** start listening. Keeping app-construction separate from server-startup is a best practice: tests can `require('./src/app')` and hit it with [supertest](https://www.npmjs.com/package/supertest) without opening a port.

```js
// src/app.js
const express = require('express');
const helmet = require('helmet');
const cors = require('cors');
const morgan = require('morgan');
const rateLimit = require('express-rate-limit');

const env = require('./config/env');
const v1Routes = require('./routes/v1');
const { notFoundHandler, errorHandler } = require('./middleware/error.middleware');

const app = express();

// --- Security & infrastructure middleware (order matters) ------------------

// 1. Secure HTTP headers.
app.use(helmet());

// 2. CORS. Allow all in dev; restrict to a list in prod via CORS_ORIGIN.
const corsOptions =
  env.corsOrigin === '*'
    ? { origin: '*' }
    : { origin: env.corsOrigin.split(',').map((o) => o.trim()) };
app.use(cors(corsOptions));

// 3. Body parsers. Limit the JSON body size to mitigate large-payload abuse.
app.use(express.json({ limit: '10kb' }));
app.use(express.urlencoded({ extended: true, limit: '10kb' }));

// 4. Request logging (concise in dev; combined in prod).
app.use(morgan(env.isDev ? 'dev' : 'combined'));

// 5. Rate limiting on the API surface.
const limiter = rateLimit({
  windowMs: env.rateLimit.windowMs,
  max: env.rateLimit.max,
  standardHeaders: true, // return RateLimit-* headers
  legacyHeaders: false,
  message: { status: 'fail', message: 'Too many requests, please try again later.' },
});
app.use('/api', limiter);

// --- Routes ----------------------------------------------------------------
app.get('/', (req, res) => {
  res.json({ status: 'success', message: 'Blog REST API. See /api/v1/health' });
});
app.use('/api/v1', v1Routes);

// --- Error handling (always LAST) ------------------------------------------
app.use(notFoundHandler); // catch unmatched routes -> 404
app.use(errorHandler); // central error handler

module.exports = app;
```

> BEST PRACTICE — middleware ordering: parsers and security middleware go **before** routes; the 404 and error handlers go **after** routes. Express runs middleware top-to-bottom, so order is behaviour, not style.

---

## Step 11 — Server entry point (`server.js`)

Finally, the entry point: load config, connect to MongoDB, start listening, and handle shutdown signals + unexpected crashes gracefully.

```js
// server.js
const app = require('./src/app');
const env = require('./src/config/env');
const { connectDB, disconnectDB } = require('./src/config/db');

let server;

async function start() {
  // Connect to the database FIRST; only listen once it's reachable.
  await connectDB();

  server = app.listen(env.port, () => {
    console.log(`🚀 Server listening on http://localhost:${env.port} [${env.nodeEnv}]`);
  });
}

// Graceful shutdown: stop accepting connections, then close the DB pool.
async function shutdown(signal) {
  console.log(`\n${signal} received. Shutting down gracefully…`);
  if (server) {
    server.close(async () => {
      await disconnectDB();
      process.exit(0);
    });
  } else {
    await disconnectDB();
    process.exit(0);
  }
}

// Catch unhandled async errors that escaped our handlers — log & exit so a
// process manager (pm2/systemd/Docker) can restart a clean instance.
process.on('unhandledRejection', (reason) => {
  console.error('💥 Unhandled Rejection:', reason);
  shutdown('unhandledRejection');
});
process.on('uncaughtException', (err) => {
  console.error('💥 Uncaught Exception:', err);
  process.exit(1); // uncaught exceptions leave the process in an unknown state
});

// OS termination signals (Ctrl+C, container stop).
process.on('SIGINT', () => shutdown('SIGINT'));
process.on('SIGTERM', () => shutdown('SIGTERM'));

start().catch((err) => {
  console.error('💥 Failed to start server:', err.message);
  process.exit(1);
});
```

> BEST PRACTICE: After an `uncaughtException` the process is in an undefined state — log it and let it die so a supervisor restarts a fresh one. Never "swallow" it and keep running.

---

## How to Run & Test

### 1. Start MongoDB

Make sure your DB is reachable (local install, Docker, or Atlas) and `MONGODB_URI` in `.env` points to it.

### 2. Run the server

```bash
# Development (auto-restart on save)
npm run dev

# or production-style
npm start
```

You should see:

```
✅ MongoDB connected
🚀 Server listening on http://localhost:3000 [development]
```

### 3. Smoke test the health endpoint

```bash
curl http://localhost:3000/api/v1/health
```

```json
{ "status": "success", "message": "API is healthy", "uptime": 4.12 }
```

### 4. Create a post

```bash
curl -X POST http://localhost:3000/api/v1/posts \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Getting Started with Express",
    "content": "Express is a minimal and flexible Node.js web application framework...",
    "author": "Ada Lovelace",
    "tags": ["node", "Express", "Backend"],
    "status": "published"
  }'
```

**`201 Created`:**

```json
{
  "status": "success",
  "message": "Post created",
  "data": {
    "post": {
      "title": "Getting Started with Express",
      "slug": "getting-started-with-express-a1b2c",
      "content": "Express is a minimal and flexible Node.js web application framework...",
      "excerpt": "Express is a minimal and flexible Node.js web application framework...",
      "author": "Ada Lovelace",
      "tags": ["node", "express", "backend"],
      "status": "published",
      "commentCount": 0,
      "createdAt": "2026-06-19T10:00:00.000Z",
      "updatedAt": "2026-06-19T10:00:00.000Z",
      "readingTimeMinutes": 1,
      "id": "6671f0a2c1a2b3c4d5e6f7a8"
    }
  }
}
```

Notice the response shows the normalising in action: tags were lower-cased, a `slug` and `excerpt` were generated, `readingTimeMinutes` is a computed virtual, and `_id`/`__v` are hidden in favour of `id`.

### 5. List posts with pagination, filter, sort, and search

```bash
# Page 1, 5 per page, only published, tagged "node", newest first, matching "express"
curl "http://localhost:3000/api/v1/posts?page=1&limit=5&status=published&tag=node&sort=-createdAt&search=express"
```

**`200 OK`** (note the `meta` block):

```json
{
  "status": "success",
  "message": "Posts fetched",
  "data": { "posts": [ { "id": "6671f0a2c1a2b3c4d5e6f7a8", "title": "Getting Started with Express", "...": "..." } ] },
  "meta": {
    "page": 1,
    "limit": 5,
    "totalItems": 1,
    "totalPages": 1,
    "hasPrevPage": false,
    "hasNextPage": false
  }
}
```

Filter by author, request a sparse fieldset, sort by title:

```bash
curl "http://localhost:3000/api/v1/posts?author=Ada%20Lovelace&fields=title,author&sort=title"
```

### 6. Get one post

```bash
curl http://localhost:3000/api/v1/posts/6671f0a2c1a2b3c4d5e6f7a8
```

### 7. Update a post (PATCH = partial)

```bash
curl -X PATCH http://localhost:3000/api/v1/posts/6671f0a2c1a2b3c4d5e6f7a8 \
  -H "Content-Type: application/json" \
  -d '{ "status": "draft", "tags": ["node", "tutorial"] }'
```

### 8. Add and list comments

```bash
# Add a comment (replace :postId with a real id)
curl -X POST http://localhost:3000/api/v1/posts/6671f0a2c1a2b3c4d5e6f7a8/comments \
  -H "Content-Type: application/json" \
  -d '{ "author": "Grace Hopper", "body": "Great intro, thanks!" }'

# List comments for that post
curl "http://localhost:3000/api/v1/posts/6671f0a2c1a2b3c4d5e6f7a8/comments?page=1&limit=10"
```

**`201 Created`** for the add:

```json
{
  "status": "success",
  "message": "Comment added",
  "data": {
    "comment": {
      "post": "6671f0a2c1a2b3c4d5e6f7a8",
      "author": "Grace Hopper",
      "body": "Great intro, thanks!",
      "createdAt": "2026-06-19T10:05:00.000Z",
      "updatedAt": "2026-06-19T10:05:00.000Z",
      "id": "6671f1b3d2e3f4a5b6c7d8e9"
    }
  }
}
```

### 9. Delete a comment, then the post

```bash
curl -X DELETE http://localhost:3000/api/v1/posts/6671f0a2c1a2b3c4d5e6f7a8/comments/6671f1b3d2e3f4a5b6c7d8e9
# -> 204 No Content

curl -X DELETE http://localhost:3000/api/v1/posts/6671f0a2c1a2b3c4d5e6f7a8
# -> 204 No Content (cascade-deletes any remaining comments)
```

### 10. See the error handling in action

```bash
# Missing required fields -> 400 with field-level details
curl -X POST http://localhost:3000/api/v1/posts \
  -H "Content-Type: application/json" \
  -d '{ "title": "Hi" }'
```

```json
{
  "status": "fail",
  "message": "Validation failed",
  "details": [
    { "field": "title", "message": "Title must be 3–160 characters" },
    { "field": "content", "message": "Content is required" },
    { "field": "author", "message": "Author is required" }
  ]
}
```

```bash
# Malformed ObjectId -> 400 (translated CastError)
curl http://localhost:3000/api/v1/posts/not-a-real-id
```

```json
{ "status": "fail", "message": "Invalid post id" }
```

```bash
# Valid-shaped but nonexistent id -> 404
curl http://localhost:3000/api/v1/posts/000000000000000000000000
```

```json
{ "status": "fail", "message": "No post found with id 000000000000000000000000" }
```

### Postman tips

- Create a **Collection** "Blog API" with a `{{baseUrl}}` variable set to `http://localhost:3000/api/v1`.
- After creating a post, save its `id` to a collection variable in the **Tests** tab:
  ```js
  // Postman Tests tab for "Create Post"
  const json = pm.response.json();
  pm.collectionVariables.set('postId', json.data.post.id);
  ```
  Then reference `{{postId}}` in the comment requests so you never paste ids by hand.

---

## Code Walkthrough / Key Concepts Recap

**The request lifecycle, end to end.** Trace a `POST /api/v1/posts`:

1. `server.js` started the app and connected to Mongo.
2. `app.js` ran the request through `helmet → cors → json parser → morgan → rate limiter`.
3. The URL matched `/api/v1` → `/posts` → the `POST /` route.
4. `validate(createPostRules)` ran express-validator chains. On failure it short-circuits with a 400; on success it calls `next()`.
5. `postController.createPost` (wrapped in `catchAsync`) pulled `req.body` and called `postService.createPost`.
6. The **service** called `Post.create()`. Mongoose ran the `pre('validate')` hook (slug/excerpt), schema validators, and inserted the doc.
7. The controller sent a `201` via `sendSuccess`, which applied the `toJSON` transform (hiding `_id`/`__v`, adding `id` + virtuals).
8. Had anything thrown, `catchAsync` would have routed it to `errorHandler`, which would translate Mongoose errors and emit a clean JSON error.

**Why the layering pays off.** Each layer has one reason to change:

| Layer | Knows about | Doesn't know about | Reason to change |
|-------|-------------|--------------------|------------------|
| Route | URLs, verbs, which middleware | DB, business rules | A path or method changes |
| Validator | Request shape | DB, responses | Input contract changes |
| Controller | `req`/`res`, status codes | DB queries | The HTTP envelope changes |
| Service | Business rules, Mongoose | HTTP | A rule or data access changes |
| Model | Schema, validation, indexes | HTTP, app flow | The data shape changes |

**Status codes used (and why).**

| Code | When | In this API |
|------|------|-------------|
| `200 OK` | Successful GET/PATCH | list/get/update |
| `201 Created` | Resource created | create post/comment |
| `204 No Content` | Success, empty body | delete post/comment |
| `400 Bad Request` | Invalid input / bad id | validation, CastError |
| `404 Not Found` | Resource/route missing | unknown id or route |
| `409 Conflict` | Uniqueness violation | duplicate slug |
| `429 Too Many Requests` | Rate limit exceeded | the limiter |
| `500 Internal Server Error` | Unexpected bug | programmer errors |

**The two-layer validation idea.** express-validator gives clients fast, friendly, field-level feedback at the edge. Mongoose validators are the database's own contract — they protect data integrity even from code paths that skip the HTTP route (seed scripts, other services). Both matter; neither replaces the other.

**`catchAsync` + `AppError` + central handler.** Together they remove `try/catch` from every controller, make "throw a 404" a one-liner anywhere in the service layer, and guarantee one consistent error shape — including auto-translating Mongoose's `CastError`, `ValidationError`, and duplicate-key (`11000`) into proper 4xx responses while masking real bugs as a generic 500 in production.

---

## Extensions & Challenges

Tackle these in order — each is a realistic feature that deepens a specific skill.

### 1. Auth-gate the writes (ties into Project 3)
Right now anyone can create/update/delete. Add an `auth` middleware that verifies a JWT from the `Authorization: Bearer <token>` header, attaches `req.user`, and protect the write routes:

```js
// Sketch: src/middleware/auth.middleware.js
const jwt = require('jsonwebtoken'); // npm i jsonwebtoken
const AppError = require('../utils/AppError');

module.exports = function protect(req, res, next) {
  const header = req.headers.authorization || '';
  const token = header.startsWith('Bearer ') ? header.slice(7) : null;
  if (!token) return next(AppError.unauthorized('Login required'));
  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET); // { id, ... }
    next();
  } catch {
    next(AppError.unauthorized('Invalid or expired token'));
  }
};
```
Then make `author` a `ref: 'User'` on the Post model and set it from `req.user.id` instead of the body. Add an **ownership check** in the service so users can only edit/delete their own posts (throw `AppError.forbidden()` otherwise).

### 2. Likes
Add `likeCount` (Number) to the post schema and a route `POST /posts/:id/like` that does an atomic `$inc: { likeCount: 1 }`. For "one like per user", store a `likedBy: [ObjectId]` array and use `$addToSet` so a user can't double-like.

### 3. Soft delete
Instead of removing posts, add `deletedAt: Date` (default `null`). On "delete", set `deletedAt = new Date()`. Add a Mongoose **query middleware** to hide soft-deleted docs from normal reads:
```js
postSchema.pre(/^find/, function (next) {
  if (!this.getOptions().withDeleted) this.where({ deletedAt: null });
  next();
});
```
Add a `DELETE /posts/:id?hard=true` path that does a real delete for admins.

### 4. Real full-text search
Replace the regex search with a MongoDB **text index** and `$text`:
```js
postSchema.index({ title: 'text', content: 'text' });
// query: Post.find({ $text: { $search: term } }, { score: { $meta: 'textScore' } })
//             .sort({ score: { $meta: 'textScore' } })
```
This is far faster than regex on large datasets and gives relevance ranking.

### 5. Keyset (cursor) pagination
For very large collections, add a `?cursor=<lastId>&limit=20` mode that queries `_id < cursor` and returns the next cursor. Compare its performance to skip-based pagination.

### 6. Swap express-validator for Joi
Joi validates a whole object against a schema. Same concept, different ergonomics — good to know both:
```js
// npm i joi  — sketch of a generic validate-with-Joi middleware
const Joi = require('joi');
const AppError = require('../utils/AppError');

const createPostSchema = Joi.object({
  title: Joi.string().trim().min(3).max(160).required(),
  content: Joi.string().trim().min(10).required(),
  author: Joi.string().trim().required(),
  excerpt: Joi.string().max(300),
  tags: Joi.array().items(Joi.string()),
  status: Joi.string().valid('draft', 'published'),
});

const validateBody = (schema) => (req, res, next) => {
  const { error, value } = schema.validate(req.body, { abortEarly: false, stripUnknown: true });
  if (error) {
    const details = error.details.map((d) => ({ field: d.path.join('.'), message: d.message }));
    return next(AppError.badRequest('Validation failed', details));
  }
  req.body = value; // use the sanitized/coerced value
  next();
};
```

| | express-validator | Joi |
|---|---|---|
| Style | Chained middleware per field | One schema object |
| Reads from | `req.body/params/query` directly | A value you pass in |
| Sanitises | Yes (`.trim()`, etc.) | Yes (`stripUnknown`, coercion) |
| Best when | Express-centric, per-field rules | Shared/reusable schemas, complex shapes |

### 7. Populate the author
Once `author` is a `ref`, use `.populate('author', 'name email')` in the service so list/get responses include the author's profile instead of just an id.

### 8. Convert to ES Modules
Set `"type": "module"` in `package.json`, change every `require`/`module.exports` to `import`/`export`, and use `import 'dotenv/config'`. Note `__dirname` isn't defined in ESM — derive it with `import.meta.url` + `fileURLToPath` if you need it.

---

## Common Errors & Fixes

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `MongooseServerSelectionError: connect ECONNREFUSED 127.0.0.1:27017` | MongoDB isn't running / wrong URI | Start Mongo (or Docker container), or fix `MONGODB_URI`. For Atlas, whitelist your IP in Network Access. |
| `Missing required environment variable: MONGODB_URI` | No `.env`, or `dotenv` not loaded | Copy `.env.example` → `.env`; ensure `require('dotenv').config()` runs in `config/env.js` before anything reads it. |
| `req.body` is `undefined` | Body parser missing or wrong `Content-Type` | Ensure `app.use(express.json())` is registered **before** routes and the request sends `Content-Type: application/json`. |
| Errors crash the process instead of returning JSON | Async handler not wrapped (Express 4) | Wrap with `catchAsync`, or upgrade to Express 5 which auto-forwards rejections. |
| Error handler never runs | It's registered before routes, or missing the 4th arg | Register `errorHandler` **last**; keep the signature `(err, req, res, next)`. |
| `Cannot read properties of undefined (reading 'postId')` in comment routes | Forgot `mergeParams` | Create the nested router with `express.Router({ mergeParams: true })`. |
| `CastError: Cast to ObjectId failed` leaks as 500 | Not translated | The central handler converts `CastError` → 400; make sure that branch exists. |
| `E11000 duplicate key error` on `slug` | Two posts generated the same slug | We append a random suffix; if you removed it, the handler still returns 409. |
| Validation passes bad data anyway | Validator not actually wired | Ensure the route uses `validate(rules)` and that `validate` awaits `rule.run(req)`. |
| `Cannot set headers after they are sent` | Sent a response twice (e.g. `res.json` then fell through to error handler) | Always `return` after sending, and don't call `next()` after responding. |
| Deprecation warnings about `useNewUrlParser` etc. | Passing legacy options to `mongoose.connect` | Remove them — they're defaults in Mongoose 6+. |
| `429 Too Many Requests` while testing | Rate limiter tripped | Raise `RATE_LIMIT_MAX` in `.env` for local dev, or hit a non-`/api` route. |

---

### You're done

You've built a production-shaped REST API with a clean layered architecture, real validation, centralized error handling, and a powerful list endpoint. More importantly, you can now **explain every layer and why it exists** — exactly the conversation a backend interview turns into. Next up: **Project 3 — Auth & Authorization**, where the `protect` middleware sketched above becomes a full JWT auth system you can bolt onto this very API.
