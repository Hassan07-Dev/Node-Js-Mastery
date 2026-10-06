# Express Middleware (The Heart of Express)

If you understand middleware, you understand Express. Almost everything Express *does* — parsing JSON bodies, serving static files, logging, authentication, compression, CORS, sessions, error handling — is middleware. Express itself is sometimes described, only half-jokingly, as "a routing and middleware web framework that has minimal functionality of its own." The framework's real power is that it gives you a clean, composable pipeline that a request flows through, and lets *you* (and the npm ecosystem) plug functions into that pipeline.

This module takes you from the mental model of "what is a middleware function" all the way to the production patterns and subtle ordering bugs that trip up even experienced developers. By the end you will be able to read any Express codebase and immediately understand *why* each `app.use(...)` line is where it is — and you will be able to debug the class of bugs ("why is `req.body` undefined?", "why does my request hang?") that are *all* really middleware-ordering or `next()` bugs.

## What You'll Learn

By the end of this module you will be able to:

- Explain what **middleware** is from first principles and describe the **`(req, res, next)`** signature precisely.
- Trace the full **request–response cycle** and the **middleware chain** a request flows through.
- Distinguish **`next()`** vs **`next(err)`** vs **not calling `next()`**, and explain exactly how a request *hangs*.
- Identify and use every **type** of middleware: application-level, router-level, built-in (`express.json`, `express.urlencoded`, `express.static`), third-party (`morgan`, `cors`, `helmet`), and **error-handling** middleware (the special 4-argument form).
- Explain **why middleware order matters** and recognize the classic ordering bugs.
- Write robust **custom middleware**: a request logger, an auth/authorization check, and a request timer.
- **Share data down the chain** correctly using `req` and `res.locals`, and know the trade-offs.
- Apply **conditional** and **route-specific** middleware, including `app.use` mounting and per-route arrays.
- Recognize and avoid the **real-world pitfalls** that cause hanging requests, double responses, and silent failures.
- Answer the **interview questions** that gatekeep Express knowledge.

> Note: Express **v5** became the default `express` on npm in 2024. This module targets v5 but flags every place where v4 and v5 behave differently (the biggest one — automatic error handling for async middleware — is in §4 and §6). The middleware *model* is identical in both versions.

---

## 1. What Middleware Actually Is

### The one-sentence definition

A **middleware function** is a function that has access to the **request object (`req`)**, the **response object (`res`)**, and the **`next` function** in the application's request–response cycle, and that can:

1. Execute any code (logging, timing, DB lookups, etc.).
2. Make changes to the `req` and `res` objects.
3. End the request–response cycle (by sending a response).
4. Call the **next** middleware in the stack via `next()`.

That's the whole definition, taken almost verbatim from the Express docs. Everything else in this module is a consequence of it.

### The mental model: an assembly line

Picture an incoming HTTP request as an item moving down a **conveyor belt**. Each middleware is a **station** on the belt. As the item arrives at a station, the worker (the middleware function) can:

- Inspect or stamp the item (read/modify `req`).
- Attach a note to it for later stations (`req.user = ...`, `res.locals.x = ...`).
- **Box it up and ship it out** — end the cycle by sending a response (`res.send(...)`). The item leaves the belt; downstream stations never see it.
- **Pass it along** to the next station by calling `next()`.

The crucial insight: **a middleware function must either end the cycle or pass control on.** If it does neither, the item just sits on the belt forever — the client waits, and eventually times out. This single rule explains the most common Express bug there is.

### Why a framework needs this

Without middleware, every route handler would have to repeat the same cross-cutting work: parse the body, check the auth token, log the request, set CORS headers, catch errors. That is duplication and a maintenance nightmare. Middleware lets you **factor cross-cutting concerns into reusable, ordered, composable functions** that run *before* (or around) your route logic. It is the [chain-of-responsibility](https://en.wikipedia.org/wiki/Chain-of-responsibility_pattern) pattern applied to HTTP.

```js
// A trivially simple middleware. It logs and passes control on.
function logger(req, res, next) {
  console.log(`${req.method} ${req.url}`); // do some work
  next(); // hand off to the next middleware/handler
}
```

That is a real, complete, usable middleware function. There is nothing magic about it — it is a plain function with a specific signature that Express agrees to call in a specific way.

### "Route handlers are middleware too"

This is one of the most clarifying facts in all of Express: **your route handlers are just middleware.** When you write `app.get('/users', (req, res) => res.json(users))`, that arrow function *is* a middleware function — it simply happens to be the one that ends the cycle. There is no separate "handler" category in Express's internals; there is only the middleware stack and the functions in it. Once you internalize this, the framework collapses into a single, simple idea.

---

## 2. The `(req, res, next)` Signature

Express decides what *kind* of function it is looking at by counting its declared parameters (its `.length`, the arity). This is why the signature matters so much — it is not just convention, it is how Express dispatches.

| Arity | Shape | Express treats it as |
| --- | --- | --- |
| 2 | `(req, res)` | A terminal handler (you just won't have `next`) |
| 3 | `(req, res, next)` | **Regular** middleware / route handler |
| 4 | `(err, req, res, next)` | **Error-handling** middleware (note `err` is first!) |

> WARNING — Pitfall: Express uses the function's **arity** (number of declared parameters) to detect error-handling middleware. If you write `(err, req, res)` with only three parameters, Express thinks it's *regular* middleware and will **never** route errors to it. Error middleware **must** declare all four parameters, even if you don't use `next`. (You can name the unused one `next` and ignore it.)

### The three parameters in detail

- **`req`** — the [request object](https://expressjs.com/en/api.html#req). An enhanced version of Node's `http.IncomingMessage`. Carries `req.method`, `req.url`, `req.headers`, `req.params`, `req.query`, and (after a body parser runs) `req.body`. You can attach your own properties to it.
- **`res`** — the [response object](https://expressjs.com/en/api.html#res). An enhanced `http.ServerResponse`. You end the cycle through it: `res.send()`, `res.json()`, `res.end()`, `res.redirect()`, `res.render()`, `res.sendFile()`. It also carries `res.locals`, a per-request scratchpad (see §8).
- **`next`** — a function Express hands you. **Calling it advances the chain.** Calling it with an argument signals an error (see §4). It is *not* a return-from-handler mechanism — calling `next()` does not stop the rest of your current function from running.

```js
function example(req, res, next) {
  // 1. Read from req
  const auth = req.headers.authorization;

  // 2. Decide what to do
  if (!auth) {
    // 3a. End the cycle right here.
    return res.status(401).send('Unauthorized');
    // NOTE the `return`: without it, execution continues to next() below,
    // and you'd both send a response AND advance the chain -> bug.
  }

  // 3b. Or attach data and pass control on.
  req.user = decode(auth);
  next();
}
```

> BEST PRACTICE: Always write `return res.send(...)` (or `return next()`) when you mean "and we're done here." The `return` doesn't change what `res.send` does, but it prevents the rest of the function from running and causing a double-send or an accidental `next()` call. This single habit eliminates a whole category of bugs.

---

## 3. The Request–Response Cycle and the Middleware Chain

### The cycle, in words

Here is the life of a single request, narrated as a diagram-in-words:

```
   ┌──────────┐
   │  CLIENT  │  sends:  GET /users/42   Authorization: Bearer ...
   └────┬─────┘
        │  HTTP request
        ▼
┌───────────────────────────────────────────────────────────────┐
│                       EXPRESS APP                               │
│                                                                 │
│  Request enters the MIDDLEWARE STACK, top to bottom:            │
│                                                                 │
│   [1] helmet()            -> sets security headers, next()      │
│   [2] cors()              -> sets CORS headers, next()          │
│   [3] morgan('dev')       -> starts a log timer, next()         │
│   [4] express.json()      -> parses body into req.body, next()  │
│   [5] requestId()         -> req.id = uuid(), next()            │
│   [6] authenticate()      -> req.user = ..., next()             │
│                                                                 │
│   ── Router matching: does the METHOD + PATH match a route? ──  │
│                                                                 │
│   [7] GET /users/:id  ->  handler reads req.params.id,          │
│                           queries DB, res.json(user)  ◄── ENDS  │
│                                                                 │
│   (If a handler calls next(err) anywhere above, control         │
│    jumps DOWN to the first ERROR-handling middleware:)          │
│                                                                 │
│   [E] (err, req, res, next) -> res.status(500).json({error})    │
│                                                                 │
└───────────────────────────────────────────────────────────────┘
        │  HTTP response
        ▼
   ┌──────────┐
   │  CLIENT  │  receives:  200 OK   { "id": 42, "name": "Ada" }
   └──────────┘
```

The key properties of this flow:

1. **Order is top-to-bottom.** Middleware runs in the exact order you register it (`app.use`, `app.get`, etc.). This is why §5 (order) is so important.
2. **The chain is sequential and cooperative.** Each link runs to the point where it calls `next()` (or sends a response). Nothing in the chain runs unless the previous link explicitly passed control.
3. **It is a one-way trip toward a response... usually.** But `next()` can be called *after* awaiting async work, so the "trip" can pause at any station while I/O happens, then resume.
4. **Exactly one response should be sent.** The cycle ends the moment a middleware sends headers + body. Trying to send again throws `ERR_HTTP_HEADERS_SENT`.

### What "the stack" really is

Internally, Express keeps an ordered list (a stack/array) of **layers**. Each layer pairs a path pattern with a middleware function. When a request comes in, Express walks the list from the top, and for each layer that *matches* the request's path (and method, for route layers), it invokes the function. `next()` is literally the mechanism that tells Express "advance to the next matching layer." That's it — there's no hidden machinery.

> TIP: You can think of `next` as a closure Express creates fresh for each layer, capturing "where we are" in the stack. Calling it pops you to the next layer. This is why calling `next()` twice runs two layers (a bug — see §9).

---

## 4. `next()` vs `next(err)` vs Not Calling `next()`

This is the conceptual core of middleware. Get these three behaviors right and you've avoided most middleware bugs.

### 4.1 `next()` — advance to the next middleware

Calling `next()` with **no arguments** tells Express: "I'm done; run the next matching middleware/handler in the stack."

```js
app.use((req, res, next) => {
  console.log('A: runs first');
  next(); // -> control goes to B
});

app.use((req, res, next) => {
  console.log('B: runs second');
  res.send('done'); // ends the cycle; nothing after this runs
});
```

There's one special string argument: **`next('route')`** skips the remaining middleware *of the current route* and jumps to the next *matching route*. (And in v5, `next('router')` skips out of the entire current router.) These are advanced escape hatches; you'll rarely need them, but interviewers like to ask.

```js
app.get('/user/:id',
  (req, res, next) => {
    // If id is 0, skip the normal handler below and try the next route.
    if (req.params.id === '0') return next('route');
    next();
  },
  (req, res) => res.send('regular user') // skipped when id === '0'
);

app.get('/user/:id', (req, res) => res.send('special user zero'));
```

### 4.2 `next(err)` — jump to error-handling middleware

Calling `next()` with **any argument other than the strings `'route'`/`'router'`** signals an **error**. Express **skips all remaining regular middleware** and jumps straight to the first **error-handling** middleware (the 4-argument kind, §6).

```js
app.get('/users/:id', (req, res, next) => {
  const user = findUser(req.params.id);
  if (!user) {
    // Create an error and forward it. Regular middleware below is skipped.
    const err = new Error('User not found');
    err.status = 404;
    return next(err); // -> jumps to the error handler
  }
  res.json(user);
});

// Error-handling middleware (must be registered AFTER routes; see §6)
app.use((err, req, res, next) => {
  res.status(err.status || 500).json({ error: err.message });
});
```

This is the single most important pattern for clean error handling: **route handlers don't send error responses themselves; they `next(err)` and let one centralized error handler format the response.**

#### Synchronous errors are caught automatically

If a *synchronous* middleware **throws**, Express catches it and routes it to your error handler as if you'd called `next(err)`:

```js
app.get('/boom', (req, res) => {
  throw new Error('sync boom'); // Express catches this automatically
});
```

#### Async errors: the v4 vs v5 difference

This is the most important version difference in the whole module.

- **Express v4:** Express does **NOT** catch errors thrown inside `async` functions or rejected promises. An unhandled rejection in an async handler will *not* reach your error middleware — the request hangs and you get an `UnhandledPromiseRejection`. You must `try/catch` and call `next(err)` yourself (or wrap handlers).
- **Express v5:** A returned promise that **rejects** (including an `async` function that throws) is **automatically** forwarded to the error handler. This is a major quality-of-life improvement.

```js
// ----- Express v4: MUST handle async errors manually -----
app.get('/v4', async (req, res, next) => {
  try {
    const data = await db.query('...'); // might reject
    res.json(data);
  } catch (err) {
    next(err); // required in v4
  }
});

// ----- Express v5: a rejected promise is auto-forwarded -----
app.get('/v5', async (req, res) => {
  const data = await db.query('...'); // if this rejects, Express calls next(err) for you
  res.json(data);
});
```

> WARNING — Pitfall (v4): A *thrown* error inside an `async` function becomes a **rejected promise**, which Express v4 silently ignores. The classic symptom is "the request just hangs and nothing logs." If you're on v4, either upgrade, use `try/catch`, or use a wrapper (see the `asyncHandler` pattern in §10 and the exercises).

### 4.3 Not calling `next()` (and not responding) — the hanging request

If a middleware **neither sends a response nor calls `next()`**, the request **hangs**. Express has no idea you're done — it's still waiting for you to either advance the chain or end it. The client sits there until it (or a proxy) times out.

```js
// THE classic bug. This middleware does nothing to advance or end the cycle.
app.use((req, res, next) => {
  console.log('I forgot to call next()!');
  // <- no next(), no res.send() ... request hangs forever
});

app.get('/', (req, res) => res.send('You will never see me'));
```

> WARNING — Pitfall: Forgetting `next()` in a non-terminal middleware is the #1 cause of "my server accepts the request but never responds." If a request hangs, audit every middleware *before* the route for a missing `next()` or a code path (an `if` branch, a caught error) that returns without calling `next()` or responding.

### Summary table

| You do... | Express does... | When to use |
| --- | --- | --- |
| `next()` | Runs the next matching middleware | Default: pass control on |
| `next('route')` | Skips rest of *this route's* middleware, tries next matching route | Conditional route bypass |
| `next('router')` (v5) | Skips out of the current router instance | Mounting/bypass logic |
| `next(err)` | Skips to the first **error** middleware | Forward an error centrally |
| `res.send()` / `res.json()` / `res.end()` (no `next`) | Ends the cycle | You're the terminal handler |
| Nothing (no `next`, no response) | **Hangs** the request | Never — this is a bug |
| `throw` in sync middleware | Auto-routes to error middleware | Convenient sync error path |
| `throw` in async middleware | v5: auto-routes; **v4: ignored** | Know your version! |

---

## 5. Types of Middleware

Express's docs categorize middleware into five kinds. They're not different *technologies* — they're the same `(req, res, next)` functions used in different places. The categories are just a vocabulary.

### 5.1 Application-level middleware

Bound to an instance of the `app` via `app.use()` or `app.METHOD()` (`app.get`, `app.post`, …). Runs for matching requests across the whole app.

```js
const express = require('express');
const app = express();

// Runs for EVERY request (no path = matches all paths).
app.use((req, res, next) => {
  req.requestTime = Date.now();
  next();
});

// Runs only for requests whose path starts with /api (mount path).
app.use('/api', (req, res, next) => {
  console.log('An API request');
  next();
});

// Method + path specific (this is also "application-level"):
app.get('/users', (req, res) => res.json([]));
```

A subtle but important detail: **`app.use('/api', fn)` matches `/api` and everything under it (`/api/users`, `/api/v1/...`)** — it's a *prefix*. By contrast, `app.get('/api', fn)` matches only the exact path `/api` for GET requests. Mount paths are stripped from `req.url` inside the mounted middleware (but `req.originalUrl` keeps the full path).

### 5.2 Router-level middleware

Identical to application-level middleware, but bound to an instance of `express.Router()`. Routers let you **modularize** your app into mountable mini-applications — the cornerstone of organizing a real codebase.

```js
// routes/users.js
const express = require('express');
const router = express.Router();

// Router-level middleware: runs for every route IN THIS ROUTER.
router.use((req, res, next) => {
  console.log('Users router hit:', req.method, req.originalUrl);
  next();
});

router.get('/', (req, res) => res.json([{ id: 1 }]));
router.get('/:id', (req, res) => res.json({ id: req.params.id }));

module.exports = router;
```

```js
// app.js
const usersRouter = require('./routes/users');
// Mount the router under /users. Now router '/' is really '/users',
// and router '/:id' is really '/users/:id'.
app.use('/users', usersRouter);
```

> TIP: Routers are the answer to "how do I keep `app.js` from becoming 2,000 lines?" Group related routes (users, products, auth) into their own router files, attach router-level middleware where it belongs, and mount them with one `app.use()` each.

### 5.3 Built-in middleware

Express bundles a small set of middleware. (Historically — pre-v4.16 — these lived in the separate `body-parser` package; since 4.16 they're built into Express itself.)

| Built-in | What it does | Sets/affects |
| --- | --- | --- |
| `express.json([opts])` | Parses incoming requests with a JSON body | Populates `req.body` |
| `express.urlencoded([opts])` | Parses URL-encoded bodies (HTML form posts) | Populates `req.body` |
| `express.static(root[, opts])` | Serves static files (HTML, CSS, images) from a folder | Sends file responses |
| `express.raw([opts])` | Parses body into a `Buffer` | `req.body` as Buffer |
| `express.text([opts])` | Parses body as a string | `req.body` as string |

```js
const path = require('path');

// Parse JSON request bodies (Content-Type: application/json).
// `limit` guards against huge payloads (DoS protection).
app.use(express.json({ limit: '1mb' }));

// Parse HTML form submissions (Content-Type: application/x-www-form-urlencoded).
// `extended: true` uses the `qs` library so you can encode nested objects.
app.use(express.urlencoded({ extended: true }));

// Serve everything in ./public as static files at the site root.
// A GET /logo.png will return ./public/logo.png if it exists.
app.use(express.static(path.join(__dirname, 'public')));
```

> WARNING — Pitfall: If you don't register `express.json()`, then `req.body` is **`undefined`** for JSON requests — accessing `req.body.email` throws `TypeError: Cannot read properties of undefined`. The fix is almost always "you forgot the body parser, or it's registered *after* the route." (See §5.7 for the ordering nuance.)

> TIP: `express.json()` only parses when the request's `Content-Type` is `application/json` (configurable via the `type` option). A request with no/wrong content type leaves `req.body` as `{}` (or undefined) even with the parser registered — a common "why is my body empty?" gotcha. Make sure the client sends the correct `Content-Type` header.

### 5.4 Third-party middleware

The npm ecosystem provides middleware for nearly everything. You install the package and `app.use()` it. The three you'll meet first:

```bash
npm install morgan cors helmet
```

```js
const morgan = require('morgan');   // HTTP request logging
const cors = require('cors');       // Cross-Origin Resource Sharing headers
const helmet = require('helmet');   // Sets a bundle of security headers

app.use(helmet());                  // secure headers first
app.use(cors());                    // allow cross-origin requests
app.use(morgan('dev'));             // concise colored request logs
```

| Package | Purpose | Why you want it |
| --- | --- | --- |
| **morgan** | Logs each HTTP request (method, URL, status, response time) | Visibility into traffic & latency; invaluable in dev and prod |
| **cors** | Adds `Access-Control-Allow-*` headers; handles preflight `OPTIONS` | Lets browsers on other origins call your API |
| **helmet** | Sets ~15 security-related HTTP headers (CSP, HSTS, X-Frame-Options, etc.) | One line of defense-in-depth against common web attacks |

Other staples you'll encounter: `compression` (gzip responses), `express-session` (sessions), `cookie-parser` (parse cookies into `req.cookies`), `express-rate-limit` (throttling), `multer` (file uploads / multipart bodies).

> BEST PRACTICE: Configure third-party middleware explicitly in production. `cors()` with no options allows *all* origins — fine for a public API, dangerous for a credentialed one. Pass an `origin` allowlist: `cors({ origin: ['https://app.example.com'], credentials: true })`.

### 5.5 Error-handling middleware

A special category defined entirely by its **four-argument** signature `(err, req, res, next)`. Covered in depth in §6.

### 5.6 Built-in vs third-party vs custom — at a glance

| Category | Bound to | Examples | You write it? |
| --- | --- | --- | --- |
| Application-level | `app` | `app.use(fn)`, `app.get(...)` | Yes |
| Router-level | `express.Router()` | `router.use(fn)` | Yes |
| Built-in | Express core | `express.json`, `express.static` | No (configure) |
| Third-party | npm packages | `morgan`, `cors`, `helmet` | No (configure) |
| Error-handling | `app`/router | `(err, req, res, next)` | Yes |

### 5.7 Mounting forms recap

The ways to register middleware, all of which place a function (or several) into the stack in order:

```js
app.use(fn);                       // all paths, all methods
app.use('/path', fn);              // all methods, paths under /path
app.get('/path', fn);              // GET, exact path (Express route)
app.get('/path', mw1, mw2, handler); // multiple, run in order, route-specific
app.use(express.Router());         // mount a router
app.use(fn1, fn2, fn3);            // multiple app-level middleware at once
```

---

## 6. Error-Handling Middleware (The 4-Argument Form)

Error-handling middleware is how Express centralizes failure. It looks like normal middleware but takes **four** parameters, and Express only routes to it when an error was forwarded via `next(err)` (or thrown synchronously, or — in v5 — from a rejected promise).

### The rules

1. It **must** have exactly **four** parameters: `(err, req, res, next)`. The arity is how Express recognizes it.
2. It must be registered **after** all the routes and other middleware it should catch errors from. Express walks the stack top-to-bottom; an error handler can only catch errors from middleware *above* it.
3. You can have **multiple** error handlers; calling `next(err)` inside one passes the error to the next error handler.
4. If you don't write one, Express uses its **default error handler** (sends the stack trace in development, a generic 500 in production).

```js
// ---- All your normal middleware and routes go ABOVE ----
app.use(express.json());
app.get('/users/:id', getUserHandler);
// ... more routes ...

// 1) A catch-all 404 handler (NOT an error handler — it's regular middleware
//    that runs when no route matched). It must come AFTER all real routes.
app.use((req, res, next) => {
  res.status(404).json({ error: `Not found: ${req.originalUrl}` });
});

// 2) THE centralized error handler. Four args. Registered LAST.
app.use((err, req, res, next) => {
  // Log the full error server-side (never leak stack traces to clients in prod).
  console.error(err.stack);

  // If headers already sent, you must delegate to Express's default handler.
  if (res.headersSent) {
    return next(err);
  }

  const status = err.status || err.statusCode || 500;
  res.status(status).json({
    error: err.message || 'Internal Server Error',
    // Only include the stack in development.
    ...(process.env.NODE_ENV !== 'production' && { stack: err.stack }),
  });
});
```

> WARNING — Pitfall: If you register the error handler **before** your routes, it will never fire — there are no errors yet when the request reaches it, and by the time an error happens, control is past it. Error handlers go **last**.

> WARNING — Pitfall: A 404 handler is **not** an error handler. A request for a non-existent route doesn't *throw* — it simply matches nothing, so it "falls through" to the bottom of the stack. Catch it with a *regular* 3-arg middleware placed after your routes, not with the 4-arg error handler.

> BEST PRACTICE: Always check `res.headersSent` at the top of your error handler. If an error occurs *after* a response started streaming, you cannot send a new status/body; you must delegate to `next(err)` so Express closes the connection cleanly. Re-sending throws `ERR_HTTP_HEADERS_SENT`.

### A reusable typed error class

Centralized error handling pairs beautifully with a small custom error class so your handlers can `next()` rich errors:

```js
class HttpError extends Error {
  constructor(status, message) {
    super(message);
    this.status = status;
    this.name = 'HttpError';
  }
}

app.get('/users/:id', async (req, res, next) => {
  const user = await db.findUser(req.params.id);
  if (!user) return next(new HttpError(404, 'User not found')); // clean!
  res.json(user);
});
```

---

## 7. Why Order Matters (and the Bugs That Prove It)

Middleware runs in **registration order**, top to bottom. Because each middleware can read/modify `req`/`res` and decide whether to advance, **the order you call `app.use()` is part of your program's logic.** Reordering two lines can silently change behavior. Here are the canonical ordering bugs.

### Bug 1: Body parser registered after the route

```js
// ❌ BROKEN: route registered before the body parser
app.post('/users', (req, res) => {
  // req.body is undefined here -> TypeError on req.body.name
  res.json({ created: req.body.name });
});
app.use(express.json()); // too late! it's below the route

// ✅ FIXED: body parser first
app.use(express.json());
app.post('/users', (req, res) => {
  res.json({ created: req.body.name }); // req.body is populated
});
```

Why: when the POST arrives, Express walks the stack and hits the route handler *before* it ever reaches `express.json()`, so the body was never parsed. The handler ends the cycle, and the parser below never runs.

### Bug 2: Auth middleware registered after the protected route

```js
// ❌ BROKEN: the secret route is registered before the auth gate
app.get('/secret', (req, res) => res.send('TOP SECRET')); // unprotected!
app.use(requireAuth); // runs only for routes registered after it

// ✅ FIXED: gate first, then protected routes
app.use(requireAuth);
app.get('/secret', (req, res) => res.send('TOP SECRET'));
```

Why: `app.use(requireAuth)` only protects routes that come *after* it in the stack. This is a real security hole — the kind that ships to production.

### Bug 3: A catch-all/404 handler registered too early

```js
// ❌ BROKEN: the 404 swallows everything because it's first
app.use((req, res) => res.status(404).send('Not found')); // matches ALL paths!
app.get('/', (req, res) => res.send('home')); // never reached

// ✅ FIXED: real routes first, 404 last
app.get('/', (req, res) => res.send('home'));
app.use((req, res) => res.status(404).send('Not found'));
```

Why: `app.use(fn)` with no path matches *every* request. Placed first, it ends every cycle with a 404 before any real route can match.

### Bug 4: `express.static` shadowing or being shadowed

```js
// If a file ./public/api exists, static could intercept /api requests
// before your API router runs. Order static relative to routes deliberately.
app.use(express.static('public')); // serves files if they exist
app.use('/api', apiRouter);        // API routes for everything else
```

Why: static middleware sends a response (ending the cycle) when a matching file exists. Whichever of "static" and "routes" you register first wins for overlapping paths.

### The reliable ordering recipe

A battle-tested top-to-bottom order for a typical API:

```js
app.use(helmet());                 // 1. security headers
app.use(cors(corsOptions));        // 2. CORS
app.use(morgan('combined'));       // 3. logging (early, so it logs everything)
app.use(express.json());           // 4. body parsing
app.use(express.urlencoded({ extended: true }));
app.use(express.static('public')); // 5. static assets
app.use(attachRequestId);          // 6. custom context
app.use('/api', authenticate, apiRouter); // 7. auth + routes
app.use(notFoundHandler);          // 8. 404 (after all routes)
app.use(errorHandler);             // 9. error handler (LAST, 4 args)
```

> BEST PRACTICE: Memorize the shape: **security → cross-cutting (CORS/logging) → parsing → static → context → auth+routes → 404 → errors.** When something behaves oddly, the first question is always "is this middleware in the right place in the order?"

---

## 8. Sharing Data Down the Chain: `req` vs `res.locals`

Middleware frequently computes something (the authenticated user, a request ID, a parsed token) that *later* middleware or the route handler needs. There are two idiomatic places to stash it.

### 8.1 Attaching to `req`

You can add properties to `req`. They live for the duration of the request and are visible to all downstream middleware and handlers. This is the standard place for **request-scoped server-side data** the rest of your code needs.

```js
app.use((req, res, next) => {
  req.requestId = crypto.randomUUID(); // attach context
  req.requestTime = Date.now();
  next();
});

app.use(async (req, res, next) => {
  req.user = await authenticate(req); // downstream handlers read req.user
  next();
});

app.get('/me', (req, res) => {
  res.json({ id: req.requestId, user: req.user }); // both available
});
```

### 8.2 Using `res.locals`

`res.locals` is an object scoped to the request, **specifically intended for data passed to views/templates** (and, by convention, for response-related context). When you render a template with `res.render()`, everything on `res.locals` is automatically available to the template — without you passing it explicitly.

```js
app.use((req, res, next) => {
  // Available to every template rendered during THIS request.
  res.locals.currentYear = new Date().getFullYear();
  res.locals.user = req.user || null;
  next();
});

app.get('/dashboard', (req, res) => {
  // The template can use `user` and `currentYear` without us re-passing them.
  res.render('dashboard', { title: 'Dashboard' });
});
```

### 8.3 Which one should you use?

| | `req.something` | `res.locals.something` |
| --- | --- | --- |
| Scope | Per-request | Per-request |
| Intended for | Server-side data the request pipeline needs | Data destined for the rendered view/response |
| Auto-exposed to templates? | No | **Yes** (in `res.render`) |
| Typical contents | `req.user`, `req.id`, parsed tokens | `currentUser`, flash messages, view flags |

> BEST PRACTICE: Use `req.x` for pipeline/business data (the user object you looked up, the request ID for logging). Use `res.locals.x` for things the *view layer* needs. For a JSON API with no templates, you'll mostly use `req`. For a server-rendered app, `res.locals` keeps templates clean.

> WARNING — Pitfall: Don't overwrite existing Express properties on `req`/`res` (e.g., don't assign `req.params`, `req.query`, or `req.body` to something incompatible, and avoid shadowing methods like `req.get`). Namespace custom data clearly (`req.user`, `req.context = {}`) to avoid collisions. Also note: data on `req`/`res` is **per-request** — never stash request-specific data on the `app` object or a module-level variable, or you'll leak one user's data into another's request.

---

## 9. Custom Middleware: Three Worked Examples

Writing your own middleware is just writing `(req, res, next)` functions. Here are three you'll genuinely use, built up to production quality.

### 9.1 A request logger

```js
// A minimal but useful logger. (In production you'd use morgan/pino, but
// writing this yourself cements the model.)
function logger(req, res, next) {
  const start = process.hrtime.bigint(); // high-resolution start time

  // 'finish' fires on res when the response has been fully sent.
  res.on('finish', () => {
    const durationMs = Number(process.hrtime.bigint() - start) / 1e6;
    console.log(
      `${req.method} ${req.originalUrl} ${res.statusCode} ${durationMs.toFixed(1)}ms`
    );
  });

  next(); // important: advance the chain so the request is actually handled
}

app.use(logger);
```

Note the technique: we attach a one-time `res.on('finish', ...)` listener and call `next()` immediately. The log line is written *after* the response completes, capturing the real status code and total duration — without blocking the request. This `res.on('finish')` pattern is the canonical way to do work "after the response is sent."

### 9.2 An authentication / authorization check

```js
const jwt = require('jsonwebtoken');

// Authentication: verifies WHO the user is, attaches req.user.
function authenticate(req, res, next) {
  const header = req.headers.authorization || '';
  const token = header.startsWith('Bearer ') ? header.slice(7) : null;

  if (!token) {
    return res.status(401).json({ error: 'Missing token' }); // end cycle
  }

  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET); // attach context
    next(); // authenticated -> proceed
  } catch (err) {
    return res.status(401).json({ error: 'Invalid token' });
  }
}

// Authorization: a middleware FACTORY — returns middleware for a given role.
// This is the standard pattern for parameterized middleware.
function requireRole(role) {
  return (req, res, next) => {
    if (!req.user || req.user.role !== role) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    next();
  };
}

// Usage: authenticate first, then authorize, then handle.
app.get('/admin', authenticate, requireRole('admin'), (req, res) => {
  res.json({ secret: 'admin data' });
});
```

The **middleware factory** (`requireRole('admin')`) is a pattern worth internalizing: a function that *returns* a `(req, res, next)` function, closing over configuration. Express itself uses this (`express.json()` returns a middleware), as do `cors()`, `helmet()`, `morgan('dev')`, etc.

### 9.3 A request timer that adds a response header

```js
function responseTimer(req, res, next) {
  const start = process.hrtime.bigint();

  // We want to set a header BEFORE headers are sent, so hook into the
  // moment Express is about to write the head. res.on('finish') is too late
  // to add headers, so we wrap res.end (or use a library). Simpler: set a
  // header right before sending by overriding when the route responds.
  // Cleanest approach: use the 'X-Response-Time' via overriding res.end.
  const originalEnd = res.end;
  res.end = function (...args) {
    const ms = Number(process.hrtime.bigint() - start) / 1e6;
    // Only set if headers not yet sent (defensive).
    if (!res.headersSent) res.setHeader('X-Response-Time', `${ms.toFixed(1)}ms`);
    return originalEnd.apply(this, args);
  };

  next();
}

app.use(responseTimer);
```

> TIP: Setting a header that depends on the *final* timing is subtle, because by the time the response finishes, headers are already flushed. The `res.end` wrapper above sets the header in the narrow window right before the response is written. In practice, reach for the maintained [`response-time`](https://www.npmjs.com/package/response-time) package, which handles the edge cases — but understanding the wrapper teaches you how middleware can intercept the response, not just the request.

---

## 10. Conditional and Route-Specific Middleware

Not all middleware should run for all requests. Express gives you several ways to scope it.

### 10.1 Route-specific middleware

Pass middleware as arguments before the handler. They run in order, only for that route.

```js
// `validateBody` and `authenticate` run only for POST /users.
app.post('/users', authenticate, validateBody, (req, res) => {
  res.status(201).json({ created: true });
});

// You can also pass an array (handy for reusable middleware groups).
const protect = [authenticate, requireRole('user')];
app.get('/profile', protect, (req, res) => res.json(req.user));
```

### 10.2 Mount-path (prefix) middleware

`app.use('/api', mw)` runs `mw` only for paths under `/api`. Combine with routers to apply middleware to a whole sub-tree.

```js
// Every /admin/* request must pass through authenticate + requireRole('admin').
app.use('/admin', authenticate, requireRole('admin'), adminRouter);
```

### 10.3 Conditional middleware (run-it-or-skip-it)

Sometimes you want a single registration point but conditional execution. Wrap the condition inside the middleware and call `next()` to skip:

```js
function onlyInProduction(middleware) {
  return (req, res, next) => {
    if (process.env.NODE_ENV === 'production') {
      return middleware(req, res, next); // run it
    }
    next(); // skip it in dev
  };
}

app.use(onlyInProduction(helmet()));
```

Or branch on the request itself:

```js
function skipForHealthCheck(req, res, next) {
  if (req.path === '/health') return next(); // skip heavy logging for health pings
  return morgan('combined')(req, res, next);
}
app.use(skipForHealthCheck);
```

> TIP: The pattern "a middleware that conditionally delegates to another middleware" is extremely common. Note that `someMiddleware(req, res, next)` is just a function call — you can invoke any middleware manually whenever you decide it should run.

### 10.4 The `asyncHandler` wrapper (essential for v4, still tidy in v5)

To avoid `try/catch` in every async route (mandatory in v4, optional but clean in v5), wrap handlers so rejected promises are forwarded to the error middleware:

```js
// Wraps an async handler so any rejection is sent to next(err).
const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);

app.get('/users/:id', asyncHandler(async (req, res) => {
  const user = await db.findUser(req.params.id); // if this rejects...
  if (!user) throw new HttpError(404, 'Not found'); // ...or this throws...
  res.json(user);                                   // ...next(err) is called.
}));
```

The popular [`express-async-errors`](https://www.npmjs.com/package/express-async-errors) package patches this behavior globally for v4. In v5 it's built in, so the wrapper becomes optional sugar.

---

## 11. Common Pitfalls & Gotchas

- **Forgetting `next()`** — A non-terminal middleware that neither responds nor calls `next()` hangs the request forever. The #1 middleware bug.
- **Forgetting `return` before `res.send`/`next()`** — Without `return`, execution falls through and you call `next()` *and* send a response (or send two responses), causing `Cannot set headers after they are sent`.
- **Wrong order** — Body parser after the route (`req.body` undefined), auth after the protected route (security hole), 404/catch-all too early (swallows real routes). Order is logic.
- **Error handler with 3 args** — `(err, req, res)` is treated as *regular* middleware; errors never reach it. Error handlers need all **four** parameters.
- **Error handler placed before routes** — It must be **last**; it can only catch errors from middleware above it.
- **Async errors in v4** — Thrown errors / rejected promises in `async` handlers are *not* auto-caught in v4. Use `try/catch`, `asyncHandler`, or upgrade to v5.
- **Calling `next()` more than once** — Runs the rest of the chain twice; often surfaces as `ERR_HTTP_HEADERS_SENT` or duplicated side effects. Guard branches with `return next()`.
- **Sending a response then calling `next()`** — Same double-handling problem. End the cycle *or* advance, never both.
- **`req.body` empty despite `express.json()`** — Usually the client didn't send `Content-Type: application/json`, or the parser is registered after the route.
- **Treating 404 as an error** — Unmatched routes don't throw; catch them with a trailing *regular* middleware, not the error handler.
- **Putting request data on `app` or module scope** — Leaks one request's data into another. Request-scoped data belongs on `req`/`res.locals`.
- **`res.headersSent` ignored in the error handler** — If an error occurs mid-stream, you must delegate via `next(err)` rather than trying to send again.
- **Mount-path confusion** — `app.use('/api', fn)` is a *prefix* match (matches `/api/anything`); `app.get('/api', fn)` is an *exact* match. They are not interchangeable.
- **`cors()` with no options in a credentialed API** — Allows all origins; lock down `origin` and set `credentials` deliberately.

---

## 12. Best Practices

- **Register cross-cutting middleware early, in a deliberate order:** security (`helmet`) → CORS → logging → body parsing → static → custom context → auth + routes → 404 → error handler.
- **Centralize error handling** in one (or a few) 4-arg error middleware at the bottom. Route handlers should `next(err)` (or throw, in v5), not format error responses themselves.
- **Always `return` when you respond or branch** (`return res.send(...)`, `return next(err)`). It is the cheapest insurance against double-handling bugs.
- **Use middleware factories** for configurable middleware (`requireRole('admin')`, `rateLimit({ max: 100 })`). It keeps call sites declarative.
- **Keep each middleware single-purpose.** A function that logs *and* authenticates *and* validates is hard to reorder and test. Compose small pieces.
- **Use routers to modularize.** One router per resource, mounted with `app.use('/resource', router)`. Attach resource-specific middleware to the router, not globally.
- **Prefer maintained libraries for hard problems** (logging: `pino`/`morgan`; security: `helmet`; uploads: `multer`; rate limiting: `express-rate-limit`). Write custom middleware for *your* business logic, not for solved problems.
- **Guard payload size** (`express.json({ limit: '1mb' })`) to mitigate denial-of-service via huge bodies.
- **Never leak stack traces to clients in production.** Log server-side; return a generic message and a safe error shape.
- **Make middleware async-safe.** On v4, wrap async handlers (`asyncHandler`) or use `express-async-errors`. On v5, returned rejected promises are auto-forwarded — still keep handlers clean.
- **Check `res.headersSent`** in your error handler and delegate to `next(err)` if a response already started.

---

## 13. Key Interview Points

- **"What is middleware in Express?"** — A function with access to `req`, `res`, and `next` that runs during the request–response cycle. It can run code, modify `req`/`res`, end the cycle, or call `next()` to pass control to the next middleware. Route handlers are themselves middleware.
- **"What does `next()` do?"** — Advances control to the next matching middleware in the stack. `next(err)` (any non-`'route'`/`'router'` argument) skips to the error-handling middleware. `next('route')` skips the rest of the current route's middleware.
- **"What happens if you don't call `next()` and don't send a response?"** — The request hangs; the client waits until it times out. A middleware must either end the cycle or pass control on.
- **"How does Express know a function is error-handling middleware?"** — By its arity: it has **four** parameters `(err, req, res, next)`. Three-parameter functions are treated as regular middleware.
- **"Where must error-handling middleware be placed?"** — Last, after all routes and other middleware, so it can catch errors propagated from above.
- **"Does Express catch errors thrown in async functions?"** — In **v4**, no — you must `try/catch` and `next(err)` (or use a wrapper). In **v5**, yes — a rejected promise from a handler is automatically forwarded to the error handler.
- **"Why does middleware order matter? Give an example."** — Middleware runs in registration order. E.g., a body parser registered after a route leaves `req.body` undefined; an auth gate after a protected route fails to protect it.
- **"Difference between `app.use('/path', fn)` and `app.get('/path', fn)`?"** — `app.use` matches the path as a **prefix** for **all** HTTP methods; `app.get` matches the **exact** path for **GET** only.
- **"What's the difference between application-level and router-level middleware?"** — Same function shape; application-level is bound to the `app`, router-level to an `express.Router()` instance for modular mounting.
- **"`req` vs `res.locals` for sharing data?"** — Both are per-request. Use `req.x` for pipeline/business data; `res.locals.x` for data automatically exposed to rendered views.
- **"What are `express.json()` / `express.urlencoded()` / `express.static()`?"** — Built-in middleware: parse JSON bodies into `req.body`, parse form-encoded bodies into `req.body`, and serve static files from a directory, respectively.
- **"What do `morgan`, `cors`, and `helmet` do?"** — Request logging, cross-origin headers/preflight handling, and a bundle of security headers, respectively.
- **"How do you avoid `try/catch` in every async route?"** — Wrap handlers with an `asyncHandler` (`Promise.resolve(fn(...)).catch(next)`), use `express-async-errors` (v4), or rely on v5's built-in promise forwarding.
- **"What causes `Cannot set headers after they are sent`?"** — Sending more than one response: e.g., calling `res.send()` then `next()`, calling `next()` twice, or missing a `return` before a response.

---

## 14. Practice Exercises

Work these in order; each builds on the last. Set up a project with `npm init -y && npm install express morgan` and create `server.js`.

1. **Hello middleware.** Write an app-level middleware that logs `METHOD URL` for every request and calls `next()`. Add a `GET /` route that responds `"home"`. Verify the log appears for every request, including unmatched ones.

2. **The hanging request.** Intentionally write a middleware that does *not* call `next()` and does *not* respond. Add a route after it. Confirm the request hangs (the browser/curl spins). Then fix it by adding `next()`. This is to *feel* the bug.

3. **Request timer header.** Write middleware that measures how long the request took and sets an `X-Response-Time` response header (in ms). Verify it with `curl -i localhost:3000/`. (Hint: you'll need to intercept the response — wrap `res.end`, or use the `res.on('finish')` pattern for logging the time even if you can't set the header.)

4. **Role-based auth via a factory.** Write `authenticate` (reads an `x-user-role` header and sets `req.user = { role }`, or 401s if missing) and a `requireRole(role)` *factory*. Protect `GET /admin` so only `x-user-role: admin` succeeds (others get 403). Demonstrate with three curls: no header, `user`, `admin`.

5. **Centralized error handling.** Create an `HttpError` class. Add a route `GET /users/:id` that calls `next(new HttpError(404, 'User not found'))` when `id !== '1'`. Add a 404 catch-all and a 4-arg error handler that returns JSON `{ error }` with the right status. Verify `/users/1`, `/users/2`, and `/nope` behave correctly.

6. **Async safety + ordering.** Write an `asyncHandler` wrapper. Add an async route that `await`s a function which rejects, wrapped in `asyncHandler`, and confirm the rejection reaches your error handler. Then deliberately move `express.json()` *below* a `POST /echo` route that echoes `req.body`, observe the bug (`req.body` undefined), and fix the order.

### Solutions

#### Solution 1 — Hello middleware

```js
const express = require('express');
const app = express();

// App-level middleware: runs for EVERY request because there's no path arg.
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next(); // advance the chain
});

app.get('/', (req, res) => res.send('home'));

app.listen(3000, () => console.log('http://localhost:3000'));
// `curl localhost:3000/` logs "GET /". `curl localhost:3000/nope` logs
// "GET /nope" too (the logger runs before routing/404 fall-through).
```

#### Solution 2 — The hanging request

```js
const express = require('express');
const app = express();

// ❌ This middleware neither responds nor calls next() -> request hangs.
app.use((req, res, next) => {
  console.log('Reached the broken middleware');
  // (no next(), no res.* ) -- comment the next line to see the hang
  // next();
});

app.get('/', (req, res) => res.send('you will not see me'));

app.listen(3000);
// `curl localhost:3000/` will hang. Uncomment `next()` above to fix it:
// then the request flows to the route and returns "you will not see me".
```

#### Solution 3 — Request timer header

```js
const express = require('express');
const app = express();

function responseTimer(req, res, next) {
  const start = process.hrtime.bigint();

  // Wrap res.end so we can set the header in the narrow window before flush.
  const originalEnd = res.end;
  res.end = function (...args) {
    const ms = Number(process.hrtime.bigint() - start) / 1e6;
    if (!res.headersSent) res.setHeader('X-Response-Time', `${ms.toFixed(2)}ms`);
    return originalEnd.apply(this, args);
  };

  next();
}

app.use(responseTimer);
app.get('/', (req, res) => res.send('ok'));
app.listen(3000);
// `curl -i localhost:3000/` shows a header like: X-Response-Time: 0.42ms
```

#### Solution 4 — Role-based auth via a factory

```js
const express = require('express');
const app = express();

function authenticate(req, res, next) {
  const role = req.headers['x-user-role'];
  if (!role) return res.status(401).json({ error: 'Missing x-user-role' });
  req.user = { role };
  next();
}

// Middleware factory: returns a middleware closed over `role`.
function requireRole(role) {
  return (req, res, next) => {
    if (req.user?.role !== role) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    next();
  };
}

app.get('/admin', authenticate, requireRole('admin'), (req, res) => {
  res.json({ secret: 'admin data' });
});

app.listen(3000);
// curl -i localhost:3000/admin                      -> 401 (no header)
// curl -i -H "x-user-role: user"  localhost:3000/admin -> 403
// curl -i -H "x-user-role: admin" localhost:3000/admin -> 200 + secret
```

#### Solution 5 — Centralized error handling

```js
const express = require('express');
const app = express();

class HttpError extends Error {
  constructor(status, message) {
    super(message);
    this.status = status;
    this.name = 'HttpError';
  }
}

app.get('/users/:id', (req, res, next) => {
  if (req.params.id !== '1') {
    return next(new HttpError(404, 'User not found')); // forward the error
  }
  res.json({ id: 1, name: 'Ada' });
});

// 404 catch-all: a REGULAR (3-arg) middleware after all routes.
app.use((req, res) => {
  res.status(404).json({ error: `Not found: ${req.originalUrl}` });
});

// Error handler: FOUR args, registered LAST.
app.use((err, req, res, next) => {
  console.error(err.stack);
  if (res.headersSent) return next(err);
  res.status(err.status || 500).json({ error: err.message });
});

app.listen(3000);
// /users/1 -> 200 {id:1,...} | /users/2 -> 404 "User not found"
// /nope    -> 404 "Not found: /nope"
```

#### Solution 6 — Async safety + ordering

```js
const express = require('express');
const app = express();

// Wrap async handlers so rejections become next(err).
const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);

// Simulated async work that rejects.
const failingQuery = async () => { throw new Error('DB exploded'); };

app.get('/async', asyncHandler(async (req, res) => {
  await failingQuery();      // rejects -> asyncHandler forwards to next(err)
  res.send('unreachable');
}));

// ----- Ordering demo -----
// ✅ Correct: body parser BEFORE the route that needs req.body.
app.use(express.json());
app.post('/echo', (req, res) => res.json({ youSent: req.body }));

// (If you moved app.use(express.json()) to AFTER this POST route,
//  req.body would be undefined and req.body access would throw — which,
//  conveniently, the error handler below would then catch.)

// Error handler last.
app.use((err, req, res, next) => {
  res.status(err.status || 500).json({ error: err.message });
});

app.listen(3000);
// curl localhost:3000/async -> 500 {"error":"DB exploded"}  (reached the handler!)
// curl -X POST localhost:3000/echo -H "Content-Type: application/json" \
//      -d '{"a":1}'  -> {"youSent":{"a":1}}
```

---

## 15. Summary / Cheat Sheet

| Concept | The essential fact |
| --- | --- |
| Middleware | A `(req, res, next)` function in the request pipeline; can run code, modify `req`/`res`, end the cycle, or call `next()` |
| Route handlers | Are themselves middleware (the terminal one) |
| `next()` | Advance to the next matching middleware |
| `next(err)` | Skip to the first 4-arg error-handling middleware |
| `next('route')` | Skip the rest of this route's middleware; try the next route |
| No `next()` + no response | **Request hangs** — the cardinal bug |
| Error middleware | **4 args** `(err, req, res, next)`; placed **last**; detected by arity |
| Sync `throw` | Auto-routed to error middleware |
| Async `throw`/reject | v4: **not** caught (use `try/catch`/`asyncHandler`); v5: auto-forwarded |
| `express.json()` | Parses JSON body into `req.body` |
| `express.urlencoded()` | Parses form-encoded body into `req.body` |
| `express.static(dir)` | Serves static files from `dir` |
| `morgan` / `cors` / `helmet` | Logging / cross-origin headers / security headers |
| Application-level | Bound to `app` via `app.use`/`app.METHOD` |
| Router-level | Bound to `express.Router()`; modularizes the app |
| `app.use('/p', fn)` | **Prefix** match, all methods |
| `app.get('/p', fn)` | **Exact** match, GET only |
| Share data | `req.x` (pipeline data) / `res.locals.x` (view data) |
| Order rule | Runs top-to-bottom in registration order; **order is logic** |
| Recommended order | security → CORS → logging → parsing → static → context → auth+routes → 404 → error handler |
| Golden habit | `return` before every `res.send` / `next` |

> The whole framework in one line: **a request walks down an ordered list of `(req, res, next)` functions until one ends the cycle — middleware is everything in that list, and you control the list and its order.**
