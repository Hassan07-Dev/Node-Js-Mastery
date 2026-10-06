# Express.js Interview Questions & Answers

A focused question bank for the **Express.js** portion of your backend interviews. It assumes you already know JavaScript and the basics of Node.js (the `http` module, the event loop, modules). Every answer is written at the depth a strong mid-to-senior candidate would actually give out loud, with runnable, commented code where it helps.

## How to Use This Bank

1. **Read the question, then cover the answer and try to respond out loud.** Interviews are spoken; rehearse speaking, not just reading.
2. **Predict the output** for the code-prediction questions before reading on. Getting these wrong is how you learn the edge cases interviewers love.
3. **Work through the sections in order.** Beginner builds the vocabulary, Intermediate is where most interviews live, Advanced is what separates a senior hire.
4. **Use the "Follow-up:" lines as a self-quiz.** If you can't answer the follow-up, you don't fully own the topic yet.
5. **Drill the Rapid-Fire table the morning of the interview.**

> TIP: When you answer, say the *why* before the *how*. "Middleware exists so you can compose cross-cutting concerns... and the signature is `(req, res, next)`" lands far better than reciting the signature alone.

Assumptions for all code: **Node.js current LTS**, **Express 4 unless a question is explicitly about Express 5**, **CommonJS (`require`) by default**. Differences between Express 4 and 5 are called out where they matter.

---

## Beginner

### 1. What is Express.js, and what problem does it solve?

**Express is a minimal, unopinionated web framework that sits on top of Node's built-in `http` module.** Node's `http` module can already create a server, but it gives you one giant request handler and nothing else — no routing, no convenient request/response helpers, no middleware pipeline. You'd hand-write URL parsing, method checks, body buffering, and content-type headers for every endpoint.

Express adds a thin, ergonomic layer on top of that:

- **Routing** — map an HTTP method + URL pattern to a handler.
- **Middleware** — a composable pipeline for cross-cutting concerns (logging, auth, parsing, error handling).
- **Request/response helpers** — `res.json()`, `res.status()`, `req.params`, `req.query`, etc.

Crucially, Express does **not** replace `http` — it *wraps* it. `app.listen()` is essentially `http.createServer(app).listen()` under the hood, and the Express `app` is itself a function with the signature `(req, res)` that you could hand directly to `http.createServer`.

```js
const express = require("express");
const http = require("http");

const app = express();
app.get("/", (req, res) => res.send("Hello"));

// These two lines are equivalent:
app.listen(3000);                       // convenience method
http.createServer(app).listen(3000);    // what it does under the hood
```

> TIP: "Unopinionated" is a key word interviewers listen for — Express doesn't force a folder structure, ORM, or template engine on you, unlike a full framework such as NestJS or Rails.

**Follow-up:** *If Express is just a wrapper, why not use the `http` module directly?* Because you'd reinvent routing, middleware, and helpers — Express is the battle-tested version of code you'd otherwise write yourself.

---

### 2. How does an Express app relate to the Node `http` module?

The Express `app` object returned by `express()` is a **request handler function** plus a bag of methods (`app.get`, `app.use`, etc.). Because it's a function with the `(req, res)` shape, it can be passed straight to `http.createServer(app)`.

Internally:

- `app.listen(port)` creates an HTTP server with the app as the handler and starts listening.
- Express **augments** Node's native `req` (an `http.IncomingMessage`) and `res` (an `http.ServerResponse`) by setting their prototypes, so you keep all native methods (`res.end`, `req.headers`) *and* gain Express helpers (`res.json`, `req.params`).

```js
const app = express();
// req is still an http.IncomingMessage; res is still an http.ServerResponse.
// Express layers req.params/req.query/res.json on top of them.
```

This is why you can mix native and Express APIs in the same handler — they're the same objects with extra methods.

**Follow-up:** *Can you run two Express apps on the same server?* Yes — mount one as middleware on the other (`app.use("/admin", adminApp)`), since an Express app is itself valid middleware.

---

### 3. What is middleware? Give the definition and the function signature.

**Middleware is a function that runs during the request/response cycle and has access to the request object (`req`), the response object (`res`), and the `next` function.** It can: execute any code, modify `req`/`res`, end the cycle by sending a response, or call `next()` to pass control to the next middleware.

The standard signature has **three** parameters:

```js
// (req, res, next)
function logger(req, res, next) {
  console.log(`${req.method} ${req.url}`); // do work
  next();                                  // hand off to the next middleware
}
app.use(logger);
```

Error-handling middleware is the special case with **four** parameters (`err, req, res, next`) — Express identifies it by arity (the number of declared arguments).

The mental model: a request flows through a **pipeline** of middleware in the order they were registered. Each one either responds (ending the chain) or calls `next()` to continue.

> WARNING — Pitfall: If a middleware neither sends a response nor calls `next()`, the request **hangs** forever. The client eventually times out. This is the single most common Express bug.

**Follow-up:** *How does Express tell a normal middleware from an error handler?* By the function's arity — four declared parameters means error handler.

---

### 4. What are the different types of middleware in Express?

Express documents five categories:

| Type | Bound with | Example |
|------|-----------|---------|
| **Application-level** | `app.use()` / `app.METHOD()` | Logging on every request |
| **Router-level** | `router.use()` / `router.METHOD()` | Auth check on a sub-router |
| **Error-handling** | `app.use((err, req, res, next) => …)` | Centralized error responder |
| **Built-in** | `express.json()`, `express.static()`, `express.urlencoded()` | Body parsing, static files |
| **Third-party** | `app.use(cors())`, `app.use(helmet())` | Installed from npm |

```js
app.use(express.json());        // built-in
app.use(require("cors")());     // third-party
app.use((req, res, next) => {   // application-level (custom)
  req.requestTime = Date.now();
  next();
});
```

The categories overlap in mechanism — they're *all* just `(req, res, next)` functions. The labels describe **where** they're attached and **what** they do, not different APIs.

**Follow-up:** *Is `express.json()` middleware or configuration?* It's a middleware *factory* — calling `express.json()` returns a middleware function you then register with `app.use()`.

---

### 5. What does `next()` do, and what happens if you forget to call it?

`next()` **passes control to the next matching middleware or route handler** in the stack. Express maintains an ordered list of layers and calls them one at a time; each layer is responsible for either ending the response or calling `next()` to advance.

- `next()` — go to the next normal middleware/route.
- `next(err)` — **skip all remaining normal middleware** and jump to the next error-handling middleware.
- `next("route")` — skip the rest of the handlers **for the current route** and move to the next route (only meaningful inside route handlers chained with `app.get(path, h1, h2)`).
- `next("router")` — skip the rest of the current router's middleware and exit the router.

If you forget to call `next()` and don't send a response, the request **hangs**:

```js
app.use((req, res, next) => {
  console.log("logged"); // runs...
  // ...but no next() and no res.send() => request never completes. BUG.
});
```

> BEST PRACTICE: Every middleware should have exactly one terminal action on every code path — either send a response or call `next()`/`next(err)`. Audit your `if/else` branches for paths that do neither.

**Follow-up:** *What's the difference between `next()` and `next(err)`?* `next()` continues the normal chain; `next(err)` jumps to error-handling middleware. Passing *any* truthy argument (except the strings `"route"`/`"router"`) is treated as an error.

---

### 6. What is the difference between `app.use()` and `app.get()` (and the other `app.METHOD` functions)?

- `app.use([path], ...handlers)` registers **middleware** that runs for **all HTTP methods** and matches the path as a **prefix**.
- `app.get(path, ...handlers)` (and `app.post`, `app.put`, etc.) registers a **route handler** for a **specific HTTP method** and matches the path **exactly** (per the route's pattern).

```js
app.use("/api", mw);        // runs for GET/POST/... on /api, /api/users, /api/x/y ...
app.get("/api", handler);   // runs ONLY for GET, ONLY on exactly /api
```

Key distinctions:

| Aspect | `app.use("/api", …)` | `app.get("/api", …)` |
|--------|----------------------|----------------------|
| HTTP methods | All | GET only |
| Path matching | Prefix (`/api/anything`) | The route pattern (here, exact `/api`) |
| Typical use | Middleware, mounting routers | Endpoint handlers |
| `req.params` from path | No (unless router with params) | Yes |

`app.all(path, handler)` is the in-between: a specific path but all methods.

**Follow-up:** *Does `app.use("/users", h)` match `/users-list`?* No — `use` matches on path *segments*, so it matches `/users` and `/users/...` but not `/users-list`.

---

### 7. Why does middleware order matter? Give an example where order breaks things.

Express executes middleware **in the exact order it is registered**, top to bottom. A request walks down the stack until something sends a response. So a middleware can only affect handlers registered **after** it.

Classic bug — body parser registered too late:

```js
const app = express();

app.post("/users", (req, res) => {
  res.json(req.body); // req.body is undefined here — parser hasn't run yet!
});

app.use(express.json()); // registered AFTER the route => never applies to it
```

Fix: register parsing/auth/logging middleware **before** the routes that need them.

```js
app.use(express.json()); // FIRST
app.post("/users", (req, res) => res.json(req.body)); // now req.body is populated
```

The same logic explains why your **404 handler and error handler must be registered last** — they're the fallthrough cases.

> BEST PRACTICE: Order your `app.use()` calls intentionally: security headers → CORS → body parsers → logging → routes → 404 handler → error handler.

**Follow-up:** *Where should `express.static` go relative to routes?* Usually before your dynamic routes if static assets should win, but be aware a static file matching a route path will be served instead of your handler.

---

### 8. How do you read route parameters versus query string parameters?

- **Route (path) parameters** are named segments in the URL pattern, read from `req.params`.
- **Query string parameters** come after `?` in the URL, read from `req.query`.

```js
// GET /users/42/posts?sort=desc&limit=10
app.get("/users/:userId/posts", (req, res) => {
  req.params.userId; // "42"        <- from the path pattern :userId
  req.query.sort;    // "desc"      <- from ?sort=desc
  req.query.limit;   // "10"        <- always a STRING (or array/object)
  res.json({ params: req.params, query: req.query });
});
```

Important gotchas:

- **Both are always strings** (or arrays/objects for query). `req.query.limit` is `"10"`, not `10` — convert with `Number()` and validate.
- `req.params` keys are defined by the route pattern; `req.query` keys are whatever the client sends.
- Query parsing supports nested/array syntax: `?tags=a&tags=b` → `["a", "b"]`, and `?filter[role]=admin` → `{ filter: { role: "admin" } }` (depending on the query parser setting).

> WARNING — Pitfall: Never trust the *type* of query/param values. `?limit=10` and `?limit=abc` both arrive as strings; coerce and validate before using them in queries or arithmetic.

**Follow-up:** *Where do form/JSON body fields live?* In `req.body`, which only exists after a body-parsing middleware runs.

---

### 9. What is `express.Router` and why would you use it?

`express.Router()` creates a **mini, mountable Express application** — an isolated stack of middleware and routes you can attach to a path with `app.use()`. It's the primary tool for **modularizing** routes so a large app isn't one giant file.

```js
// routes/users.js
const router = require("express").Router();

router.get("/", listUsers);        // GET  /users
router.post("/", createUser);      // POST /users
router.get("/:id", getUser);       // GET  /users/:id

module.exports = router;
```

```js
// app.js
const usersRouter = require("./routes/users");
app.use("/users", usersRouter); // mount the router under /users
```

Benefits:

- **Encapsulation** — each resource gets its own file with its own middleware.
- **Prefix once** — the mount path (`/users`) is written in one place; routes inside use relative paths (`/`, `/:id`).
- **Router-level middleware** — `router.use(auth)` applies to just that router's routes.

> TIP: A Router behaves like an app, so you can nest routers inside routers for deep resource hierarchies (e.g., `/users/:id/posts`).

**Follow-up:** *How do you access a param from the mount path inside a router?* Create the router with `express.Router({ mergeParams: true })` so parent route params (like `:userId`) are merged into the child router's `req.params`.

---

### 10. How do you send different kinds of responses? Compare `res.send`, `res.json`, and `res.end`.

| Method | Use for | Content-Type set | Notes |
|--------|---------|------------------|-------|
| `res.send(body)` | Strings, Buffers, objects, arrays | Auto-detected | Objects/arrays are JSON-serialized; sets `Content-Length` |
| `res.json(obj)` | JSON specifically | `application/json` | Runs through `JSON.stringify`; respects `app.set("json spaces")` |
| `res.end([data])` | Ending without a body, or streaming | None added by Express | The raw Node method; no helpers |

```js
res.send("plain text");           // text/html
res.send({ ok: true });           // application/json (auto)
res.json({ ok: true });           // application/json (explicit, preferred for APIs)
res.status(204).end();            // No Content, no body
```

For APIs, **prefer `res.json()`** even for objects — it's explicit and unambiguous about intent. Use `res.end()` when you're streaming or deliberately sending an empty body.

Other common responders: `res.sendFile(path)`, `res.redirect(url)`, `res.render(view, data)` (with a template engine), `res.sendStatus(code)` (sets status *and* sends the status text as body).

> WARNING — Pitfall: You can only send **one** response per request. Calling `res.send()` twice throws `Error [ERR_HTTP_HEADERS_SENT]: Cannot set headers after they are sent to the client`.

**Follow-up:** *What does `res.send` do with a number?* In Express 4, `res.send(404)` was treated as a *status code* (deprecated); always use `res.status(404).send(...)` or `res.sendStatus(404)` instead.

---

### 11. How do you set the HTTP status code of a response?

Use **`res.status(code)`**, which is chainable, then call a responder:

```js
res.status(201).json({ id: 1 });   // 201 Created with a JSON body
res.status(404).send("Not Found"); // 404 with a body
res.sendStatus(204);               // sets status AND sends "No Content" as body text
```

`res.status()` only **sets** the status — it does not send the response. You must follow it with `.send()`, `.json()`, `.end()`, etc. `res.sendStatus(code)` is a shortcut that sets the status and sends the standard status message as the body.

Common codes you should know cold:

| Code | Meaning | Typical use |
|------|---------|-------------|
| 200 | OK | Successful GET/PUT |
| 201 | Created | Successful POST that creates a resource |
| 204 | No Content | Successful DELETE / empty body |
| 400 | Bad Request | Validation failure / malformed input |
| 401 | Unauthorized | Missing/invalid authentication |
| 403 | Forbidden | Authenticated but not allowed |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate / version conflict |
| 422 | Unprocessable Entity | Semantically invalid input |
| 500 | Internal Server Error | Unhandled server error |

> TIP: 401 is about *authentication* ("who are you?"); 403 is about *authorization* ("you're known, but not allowed"). Interviewers love this distinction.

**Follow-up:** *What's the default status code if you never set one?* 200, as soon as you send any body.

---

### 12. What is body parsing and how do you enable it in modern Express?

When a client sends a request body (JSON, a form submission, etc.), it arrives as a **stream of bytes** on `req`. Body parsing is the step that reads that stream, decodes it according to the `Content-Type`, and populates **`req.body`** with a usable JavaScript value.

Express does **not** parse bodies by default — you must add the middleware:

```js
// Built into Express (since 4.16) — no separate body-parser install needed.
app.use(express.json());                          // Content-Type: application/json
app.use(express.urlencoded({ extended: true }));  // application/x-www-form-urlencoded (HTML forms)
```

- `express.json()` parses JSON bodies into objects/arrays.
- `express.urlencoded({ extended: true })` parses URL-encoded form bodies; `extended: true` uses the `qs` library to allow rich/nested objects, `false` uses the simpler `querystring`.
- For file uploads (`multipart/form-data`), use a dedicated library like **multer** — the built-ins don't handle multipart.

```js
app.post("/login", (req, res) => {
  // With express.json() registered, a JSON request body is now an object:
  const { email, password } = req.body;
  res.json({ received: email });
});
```

> WARNING — Pitfall: If `req.body` is `undefined`, you almost certainly forgot the parser, registered it *after* the route, or the client sent the wrong `Content-Type`.

**Follow-up:** *Why was `body-parser` a separate package historically?* It was extracted out, then the core parsers were re-incorporated into Express as `express.json()`/`express.urlencoded()` in 4.16. The standalone `body-parser` still works but is no longer needed.

---

### 13. How do you serve static files (CSS, images, client JS) in Express?

Use the built-in **`express.static`** middleware, pointing it at a directory:

```js
const path = require("path");

// Files in ./public are served from the URL root.
app.use(express.static(path.join(__dirname, "public")));
// public/style.css  ->  http://host/style.css
// public/img/a.png  ->  http://host/img/a.png
```

To serve under a URL prefix, give `app.use` a mount path:

```js
app.use("/static", express.static(path.join(__dirname, "public")));
// public/style.css  ->  http://host/static/style.css
```

Notes:

- Always use `path.join(__dirname, ...)` so the path is **absolute** and works regardless of the process's current working directory.
- `express.static` automatically sets `Content-Type`, `Content-Length`, `Last-Modified`, and supports conditional/range requests and caching headers (`maxAge` option).
- You can register multiple static directories; Express checks them in order.

```js
app.use(express.static("public", { maxAge: "1d" })); // cache for a day
```

> BEST PRACTICE: In production, prefer serving static assets via a CDN or reverse proxy (Nginx) for performance; `express.static` is fine for development and small apps.

**Follow-up:** *What happens if a static file and a route share a path?* Whichever middleware is registered first wins. If `express.static` runs first and finds the file, your route handler never executes.

---

### 14. What is the difference between `req.params`, `req.query`, and `req.body`?

These are the three places user-supplied data arrives, each from a different part of the request:

| Property | Source | Example URL/body | Requires middleware? |
|----------|--------|------------------|----------------------|
| `req.params` | Named segments in the route pattern | `/users/:id` → `/users/7` | No |
| `req.query` | The query string after `?` | `/search?q=node` | No (built-in query parser) |
| `req.body` | The request body (JSON, form, etc.) | `{ "name": "Ann" }` | **Yes** (`express.json()` etc.) |

```js
// PUT /users/7?notify=true   body: { "name": "Ann" }
app.put("/users/:id", (req, res) => {
  req.params.id;     // "7"
  req.query.notify;  // "true"
  req.body.name;     // "Ann" (only if express.json() ran)
  res.sendStatus(204);
});
```

The big distinction: `params` and `query` are read from the **URL** and need no setup; `body` needs a **body-parsing middleware** registered before the route.

**Follow-up:** *Which of these is safe to trust?* None — all three are client-controlled input. Validate and sanitize all of them.

---

## Intermediate

### 15. Explain how Express matches routes. What determines which handler runs?

Express matching is governed by **three** things, checked in order:

1. **Registration order** — Express walks its layer stack top-to-bottom. The **first** layer that matches both method and path wins.
2. **HTTP method** — `app.get` only matches GET; `app.use`/`app.all` match any method.
3. **Path pattern** — the URL must match the route's path (converted internally to a RegExp by `path-to-regexp`).

So routes are matched **first-registered-first-served**, not most-specific-first. This means a broad route registered early can "shadow" a specific one registered later:

```js
app.get("/users/:id", getUser);     // matches /users/new too!  :id = "new"
app.get("/users/new", showNewForm); // unreachable — the line above already matched
```

Fix by ordering specific routes **before** dynamic ones:

```js
app.get("/users/new", showNewForm); // specific first
app.get("/users/:id", getUser);     // dynamic second
```

A handler that calls `next()` (without responding) lets matching continue to the next layer that also matches — useful for falling through to a more general handler.

> WARNING — Pitfall: Putting a 404/catch-all handler in the middle of your stack will swallow every request below it. Catch-alls go **last**.

**Follow-up:** *Does Express stop at the first match?* It stops when a handler *responds*. If a matching handler calls `next()`, Express continues looking for the next matching layer.

---

### 16. How does error-handling middleware work, and how is it different from regular middleware?

Error-handling middleware is a middleware function declared with **four** parameters: `(err, req, res, next)`. Express recognizes it **by arity** (four declared args) and routes errors to it.

```js
// Regular middleware: 3 args
app.use((req, res, next) => { /* ... */ });

// Error-handling middleware: 4 args (MUST declare all four, even if unused)
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(err.status || 500).json({ error: err.message });
});
```

How errors reach it:

- You call `next(err)` with a truthy argument, **or**
- (Express 5 only) an async route handler **rejects** — Express auto-forwards the rejection to the error handler.
- (Express 4) a **synchronous** throw inside a handler is caught automatically; an **async** rejection is *not* — you must catch and `next(err)` it yourself.

Key rules:

- Error handlers must be registered **after** all routes and other middleware.
- You can have multiple error handlers; calling `next(err)` inside one passes to the next error handler.
- If you don't provide one, Express's **default error handler** responds with the stack trace (in development) or a generic 500.

> WARNING — Pitfall: Even if you don't use `next`, you must declare all four parameters. `(err, req, res) => {}` (three args) is treated as a *regular* middleware and will never catch errors.

**Follow-up:** *How do you handle async errors in Express 4?* Wrap handlers in a helper like `const asyncHandler = fn => (req, res, next) => Promise.resolve(fn(req, res, next)).catch(next);`.

---

### 17. Predict the output: in what order do these run, and what does the client receive?

```js
const express = require("express");
const app = express();

app.use((req, res, next) => { console.log("A"); next(); });
app.get("/", (req, res, next) => { console.log("B"); next(); });
app.use((req, res, next) => { console.log("C"); res.send("done"); });
app.use((req, res) => { console.log("D"); res.send("late"); });

app.listen(3000);
```

**A GET request to `/` logs `A`, `B`, `C` (in that order) and the client receives `"done"`.** `D` never runs.

Walkthrough:

1. `A` — application-level middleware, matches everything, calls `next()`.
2. `B` — the `GET /` route handler, calls `next()` (so matching continues to subsequent middleware).
3. `C` — the next middleware matches `/`, sends `"done"`, and **ends the cycle**.
4. `D` — never reached, because `C` already responded and didn't call `next()`.

The lesson: route handlers and middleware live in **one ordered stack**; calling `next()` from a route handler continues into later middleware. The first one to respond wins.

> TIP: If `C` had called `next()` instead of responding, `D` would run and send `"late"` — and you'd be fine, because nothing responded before it.

**Follow-up:** *What if `C` called both `res.send("done")` and `next()`?* `D` would run, try to `res.send("late")`, and throw `ERR_HTTP_HEADERS_SENT`.

---

### 18. How do you write a 404 handler and a centralized error handler? Where do they go?

Both go at the **very end** of the middleware stack, after all routes. The 404 handler is a normal middleware (no path) that runs only when no route matched. The error handler is the four-arg version.

```js
// ... all your routes above this point ...

// 404 — runs when nothing above matched the request.
app.use((req, res, next) => {
  res.status(404).json({ error: `Not found: ${req.method} ${req.originalUrl}` });
});

// Centralized error handler — runs when any layer calls next(err).
app.use((err, req, res, next) => {
  console.error(err);                       // log full error server-side
  const status = err.status || err.statusCode || 500;
  res.status(status).json({
    error: status === 500 ? "Internal Server Error" : err.message, // don't leak internals on 500
  });
});
```

Why this order works: a request that matches no route "falls through" to the 404 middleware. If any handler throws or calls `next(err)`, Express skips straight past the 404 (and all normal middleware) to the error handler.

> BEST PRACTICE: Have one error handler that maps known error types to status codes (e.g., a `ValidationError` → 400, a `CastError` → 404) and defaults everything else to 500. Never send raw stack traces to clients in production.

**Follow-up:** *Can the 404 handler trigger the error handler?* Yes — call `next(createError(404))` instead of responding directly, so all "not found" logic flows through the central error handler. The `http-errors` package is the common way to do this.

---

### 19. What are `app.set()`, `app.locals`, and `res.locals`? When do you use each?

These are three distinct stores with different lifetimes and purposes:

| Mechanism | Scope / lifetime | Read with | Typical use |
|-----------|------------------|-----------|-------------|
| `app.set(key, val)` | App-wide settings | `app.get(key)` / `req.app.get(key)` | Configuration: `view engine`, `trust proxy`, custom config |
| `app.locals` | App-wide variables | `app.locals.x` | App-level constants available in all views (site name, helpers) |
| `res.locals` | **Per-request** | `res.locals.x` | Request-scoped data for views/handlers (current user, flash messages) |

```js
// app.set — configuration (some keys have special meaning to Express)
app.set("view engine", "ejs");
app.set("trust proxy", 1);
app.set("config", { region: "us-east-1" });   // custom setting
app.get("config");                             // -> { region: "us-east-1" }

// app.locals — shared across all requests, available in templates
app.locals.siteName = "My App";

// res.locals — fresh for each request; great for "current user"
app.use((req, res, next) => {
  res.locals.user = req.user; // available to templates and later middleware for THIS request
  next();
});
```

The critical distinction: **`res.locals` is per-request** (created and destroyed with each response), while **`app.locals` lives for the lifetime of the app** (shared across all requests — never store request-specific data there, or you'll leak one user's data to another).

> WARNING — Pitfall: Putting user-specific data in `app.locals` is a serious bug — it's shared globally, so request B can see request A's data. Use `res.locals` for anything request-scoped.

**Follow-up:** *Name a special `app.set` key.* `view engine`, `views`, `trust proxy`, `env`, `json spaces`, `case sensitive routing`, `strict routing`, `query parser`.

---

### 20. Walk through building a RESTful CRUD resource in Express. What are the conventions?

REST maps **HTTP methods to actions** on a **noun-based resource URL**. For a `users` resource:

| Method | Path | Action | Success status |
|--------|------|--------|----------------|
| GET | `/users` | List all | 200 |
| GET | `/users/:id` | Read one | 200 (or 404) |
| POST | `/users` | Create | 201 (+ `Location` header) |
| PUT | `/users/:id` | Replace fully | 200 / 204 |
| PATCH | `/users/:id` | Update partially | 200 |
| DELETE | `/users/:id` | Remove | 204 |

```js
const router = require("express").Router();

router.get("/", async (req, res) => {
  const users = await User.find();
  res.status(200).json(users);
});

router.get("/:id", async (req, res, next) => {
  const user = await User.findById(req.params.id);
  if (!user) return res.status(404).json({ error: "User not found" });
  res.json(user);
});

router.post("/", async (req, res) => {
  const user = await User.create(req.body);
  res.status(201).location(`/users/${user.id}`).json(user); // 201 + Location
});

router.patch("/:id", async (req, res, next) => {
  const user = await User.findByIdAndUpdate(req.params.id, req.body, {
    new: true,           // return the updated doc
    runValidators: true, // enforce schema validation on update
  });
  if (!user) return res.status(404).json({ error: "Not found" });
  res.json(user);
});

router.delete("/:id", async (req, res) => {
  await User.findByIdAndDelete(req.params.id);
  res.sendStatus(204); // No Content
});

module.exports = router; // mount with app.use("/users", router)
```

REST principles to mention: **nouns not verbs** in URLs (`/users`, not `/getUsers`), **plural resource names**, **statelessness** (each request carries its own auth), correct **status codes**, and **proper use of methods** (GET is safe/idempotent, PUT/DELETE are idempotent, POST is not).

> BEST PRACTICE: Return `201` with a `Location` header on create, `204` (empty) on delete, and a `404` (not a 200 with `null`) when a resource doesn't exist.

**Follow-up:** *PUT vs PATCH?* PUT replaces the entire resource (idempotent); PATCH applies a partial update.

---

### 21. What does `mergeParams` do, and when do you need it?

By default, an `express.Router()` has its **own** `req.params` and does **not** see params captured by a parent router's mount path. `mergeParams: true` merges the parent's params into the child router's `req.params`.

This matters for **nested resources**:

```js
// app.js — parent captures :userId in the mount path
const postsRouter = require("./routes/posts");
app.use("/users/:userId/posts", postsRouter);
```

```js
// routes/posts.js
const router = require("express").Router({ mergeParams: true }); // <- needed!

router.get("/:postId", (req, res) => {
  // Without mergeParams, req.params.userId would be undefined here.
  res.json({ userId: req.params.userId, postId: req.params.postId });
});

module.exports = router;
```

Without `mergeParams: true`, `req.params.userId` would be `undefined` inside the child router, even though it was captured by the mount path.

> WARNING — Pitfall: If parent and child use the **same** param name, the child's value wins after merge. Use distinct names to avoid surprises.

**Follow-up:** *Does `mergeParams` affect `req.query`?* No — query is global to the request and always fully available; `mergeParams` only concerns path params across nested routers.

---

### 22. How do you handle asynchronous errors in route handlers?

The danger: in **Express 4**, an `async` handler that throws/rejects produces an **unhandled promise rejection** — Express doesn't catch it, the request hangs, and your error handler never runs.

```js
// BROKEN in Express 4: rejection is not forwarded to the error handler.
app.get("/users/:id", async (req, res) => {
  const user = await User.findById(req.params.id); // if this throws -> request hangs
  res.json(user);
});
```

Three solutions:

**1. Manual try/catch + next(err):**

```js
app.get("/users/:id", async (req, res, next) => {
  try {
    const user = await User.findById(req.params.id);
    res.json(user);
  } catch (err) {
    next(err); // forward to the central error handler
  }
});
```

**2. An `asyncHandler` wrapper (DRY):**

```js
const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next); // catch -> next(err)

app.get("/users/:id", asyncHandler(async (req, res) => {
  const user = await User.findById(req.params.id);
  res.json(user);
}));
```

**3. Upgrade to Express 5**, which **automatically** forwards rejected promises from async middleware/handlers to the error handler — no wrapper needed. (Or use a library like `express-async-errors` in Express 4 to patch the same behavior.)

> BEST PRACTICE: Standardize on one approach. In Express 4, an `asyncHandler` wrapper applied consistently is the cleanest. In Express 5, you mostly don't need it.

**Follow-up:** *Does Express 4 catch synchronous throws?* Yes — a synchronous `throw` inside a handler is caught and routed to the error handler automatically. Only async rejections need the wrapper.

---

### 23. What are some key `req` and `res` methods/properties every Express dev should know?

A strong candidate can rattle these off. Group them mentally.

**`req` (incoming):**

| Member | What it gives you |
|--------|-------------------|
| `req.params` / `req.query` / `req.body` | Route params / query string / parsed body |
| `req.headers` / `req.get(name)` | Request headers (case-insensitive `get`) |
| `req.method` / `req.path` / `req.originalUrl` | Verb / path / full original URL (path survives router mounting) |
| `req.cookies` / `req.signedCookies` | Parsed cookies (needs `cookie-parser`) |
| `req.ip` / `req.ips` | Client IP (respects `trust proxy`) |
| `req.protocol` / `req.secure` / `req.hostname` | `http`/`https`, HTTPS boolean, host |
| `req.accepts(type)` / `req.is(type)` | Content negotiation / body content-type check |

**`res` (outgoing):**

| Member | What it does |
|--------|--------------|
| `res.status(code)` | Set status (chainable) |
| `res.json(obj)` / `res.send(body)` | Send JSON / auto-typed body |
| `res.sendStatus(code)` | Set status + send status text |
| `res.set(name, val)` / `res.get(name)` | Set/read response headers |
| `res.cookie(name, val, opts)` / `res.clearCookie()` | Manage cookies |
| `res.redirect([status,] url)` | Redirect (default 302) |
| `res.sendFile(path)` / `res.download(path)` | Send a file / prompt download |
| `res.render(view, data)` | Render a template |
| `res.location(url)` | Set the `Location` header |
| `res.type(t)` / `res.format({...})` | Set Content-Type / content negotiation |

```js
res.status(201)
   .set("X-Custom", "yes")     // chainable header
   .location("/users/1")       // Location header
   .json({ id: 1 });           // body + terminates
```

> TIP: `req.path` vs `req.originalUrl` is a favorite: inside a mounted router, `req.path` is *relative* to the mount point, while `req.originalUrl` is the full URL the client requested.

**Follow-up:** *Why use `req.get("Content-Type")` instead of `req.headers["content-type"]`?* `req.get` is case-insensitive and is the documented API; `req.headers` is the raw object with lowercased keys.

---

### 24. How do you structure a large Express application?

The goal is **separation of concerns** so the codebase stays navigable as it grows. The common pattern is a **layered / MVC-inspired** structure:

```
src/
├── app.js              # builds the Express app: middleware + mounts routers (no listen)
├── server.js           # imports app, connects DB, calls app.listen()
├── config/             # env loading, db connection, constants
├── routes/             # express.Router() per resource — URL -> controller mapping only
│   ├── users.routes.js
│   └── posts.routes.js
├── controllers/        # HTTP layer: read req, call service, shape res (no business logic)
│   └── users.controller.js
├── services/           # business logic, orchestration, transactions
│   └── users.service.js
├── models/             # Mongoose schemas/models (data layer)
│   └── user.model.js
├── middleware/         # auth, validation, error handler, rate limiting
│   └── error.middleware.js
└── utils/              # pure helpers (asyncHandler, AppError, logger)
```

The layered flow of a request: **route → middleware (auth/validation) → controller → service → model → DB**, then back out.

```js
// routes/users.routes.js — only wiring
const router = require("express").Router();
const ctrl = require("../controllers/users.controller");
router.get("/:id", ctrl.getUser);
module.exports = router;

// controllers/users.controller.js — HTTP concerns only
exports.getUser = asyncHandler(async (req, res) => {
  const user = await userService.findById(req.params.id); // delegate logic
  res.json(user);
});

// services/users.service.js — business logic, no req/res
exports.findById = (id) => User.findById(id).orFail(new AppError("User not found", 404));
```

Key principles:

- **Separate `app.js` from `server.js`** so you can import the app into tests without starting a listener.
- **Controllers stay thin** — no business logic, no direct DB calls; they translate HTTP ↔ service calls.
- **Services hold business logic** and are framework-agnostic (no `req`/`res`), so they're reusable and testable.
- **Centralize error handling and config.**

> BEST PRACTICE: The single most valuable split is `app.js` (exports the configured app) vs `server.js` (starts it). It makes integration testing with Supertest trivial.

**Follow-up:** *Why keep services free of `req`/`res`?* So business logic is testable in isolation and reusable from non-HTTP contexts (CLI scripts, cron jobs, queues).

---

### 25. What is the difference between `req.path`, `req.url`, `req.originalUrl`, and `req.baseUrl`?

These four describe the URL but differ in scope, especially once a request enters a **mounted router**.

| Property | Meaning | Example (request `/api/users/7?active=1`, router mounted at `/api`) |
|----------|---------|--------------------------------------------------------------------|
| `req.originalUrl` | Full original URL incl. query — never changes | `/api/users/7?active=1` |
| `req.baseUrl` | The mount path of the current router | `/api` |
| `req.path` | Path **relative to the current router**, no query | `/users/7` |
| `req.url` | Like `path` but **includes** query; mutable, rewritten by routing | `/users/7?active=1` |

```js
app.use("/api", (req, res) => {
  // For a request to /api/users/7?active=1
  res.json({
    originalUrl: req.originalUrl, // "/api/users/7?active=1"
    baseUrl: req.baseUrl,         // "/api"
    path: req.path,               // "/users/7"
    url: req.url,                 // "/users/7?active=1"
  });
});
```

The key insight: as a request descends through mounted middleware, Express **strips the mount path** from `req.url`/`req.path` so inner handlers see relative paths. `req.originalUrl` preserves what the client actually requested — use it for logging.

**Follow-up:** *Which should you log?* `req.originalUrl`, because it reflects the real URL the client hit regardless of internal mounting.

---

### 26. How do you implement reusable middleware that takes configuration (a middleware factory)?

A **middleware factory** is a function that takes options and **returns** a middleware function. This is the pattern behind `express.json()`, `cors()`, etc. It lets you parameterize behavior.

```js
// Factory: takes a role, returns an authorization middleware.
function requireRole(role) {
  return function (req, res, next) {
    if (!req.user) return res.status(401).json({ error: "Not authenticated" });
    if (req.user.role !== role) {
      return res.status(403).json({ error: "Forbidden" });
    }
    next();
  };
}

// Use it with different configs on different routes:
app.get("/admin", requireRole("admin"), adminHandler);
app.get("/reports", requireRole("manager"), reportsHandler);
```

The outer function captures `role` in a **closure**; each call returns a fresh middleware bound to that value. This is the idiomatic way to make configurable, reusable middleware.

> TIP: If your middleware needs no config, just export the `(req, res, next)` function directly. Reach for a factory only when behavior must vary by call site.

**Follow-up:** *How would you make middleware that applies only to certain HTTP methods?* Check `req.method` inside, or attach it via `app.METHOD`/`router.METHOD` rather than `app.use`.

---

## Advanced

### 27. What are the major differences between Express 4 and Express 5?

Express 5 is the modern major release; interviewers increasingly ask about it. The headline changes:

| Area | Express 4 | Express 5 |
|------|-----------|-----------|
| **Async error handling** | Rejected promises in handlers are **not** caught (request hangs) | Rejected promises are **auto-forwarded** to the error handler |
| **Path matching / `path-to-regexp`** | Older version; `*` and regex-in-string allowed loosely | Upgraded; **named wildcards** required (`/*splat`), stricter patterns |
| **Optional params** | `/:name?` | `/:name?` still works but semantics tightened; some regex chars disallowed in strings |
| **`req.param(name)`** | Available | **Removed** — use `req.params`, `req.query`, `req.body` |
| **`app.del()`** | Available (alias) | **Removed** — use `app.delete()` |
| **`res.sendfile()` (lowercase)** | Available | **Removed** — use `res.sendFile()` |
| **`res.json(status, obj)` / `res.send(status, body)`** | Deprecated signatures still work | **Removed** — use `res.status(s).json(obj)` |
| **`res.redirect('back')`** | Supported | **Removed** — use `req.get('Referrer')` explicitly |
| **`app.router`** | Deprecated property | **Removed** |
| **Node support** | Older Node OK | Requires **Node 18+** |

The two that matter most in interviews:

1. **Automatic async error handling** — in Express 5 you can write `async` handlers that throw, and the rejection is sent to your error middleware automatically. No more `asyncHandler` wrapper.

```js
// Express 5: this is safe — the rejection reaches the error handler.
app.get("/users/:id", async (req, res) => {
  const user = await User.findById(req.params.id);
  if (!user) throw new AppError("Not found", 404); // auto-caught in Express 5
  res.json(user);
});
```

2. **Wildcard/route syntax changes.** A bare `*` is no longer a valid path; you must name it.

```js
// Express 4: app.get("*", handler)
// Express 5: name the wildcard
app.get("/*splat", handler);   // req.params.splat holds the matched segments
```

> WARNING — Pitfall: The most common Express 4 → 5 migration breakage is unnamed wildcard routes (`"*"`, `"/foo/*"`) throwing at startup. Convert them to named wildcards.

**Follow-up:** *Does upgrading to Express 5 let you delete your asyncHandler wrappers?* Yes — async rejections are forwarded automatically, though keeping a wrapper does no harm.

---

### 28. Predict the output: what does the client receive and what is logged?

```js
const express = require("express");
const app = express();

app.get("/data", (req, res, next) => {
  res.json({ stage: 1 });
  next(); // <-- called AFTER sending a response
});

app.use((req, res, next) => {
  console.log("post-handler");
  res.json({ stage: 2 }); // tries to respond again
});

app.use((err, req, res, next) => {
  console.log("ERROR:", err.message);
  res.status(500).end();
});

app.listen(3000);
```

**The client receives `{ "stage": 1 }`.** The server logs `post-handler`, then logs `ERROR: Cannot set headers after they are sent to the client`. The error handler's `res.status(500).end()` also fails silently (or throws) because headers are already sent.

Why:

1. The route handler sends `{ stage: 1 }` — that response is now committed and flushed.
2. It then calls `next()`, so the next middleware runs and logs `post-handler`.
3. That middleware calls `res.json({ stage: 2 })`, which tries to set headers on an already-sent response → throws `ERR_HTTP_HEADERS_SENT`.
4. Express catches that synchronous throw and routes it to the error handler, which logs the message — but it too can't send anything, because the response is already gone.

The lesson: **never call `next()` after sending a response** unless you deliberately know nothing downstream will try to respond.

> BEST PRACTICE: Guard with `if (res.headersSent) return next(err);` inside error handlers, and `return` after every `res.send`/`res.json` so control flow can't continue accidentally.

**Follow-up:** *How do you defensively detect this?* Check `res.headersSent` before writing, and always `return` the result of your responder calls.

---

### 29. How does Express handle a synchronous throw versus a returned rejected promise (Express 4)?

This is a precise distinction that trips up many candidates.

- **Synchronous `throw`** inside a route handler/middleware: Express **catches it** automatically and forwards to the error handler. This works because Express invokes your handler inside a `try/catch`.

```js
app.get("/sync", (req, res) => {
  throw new Error("boom"); // Express 4 catches this -> error handler runs. ✅
});
```

- **Asynchronous rejection** (an `async` function that rejects, or a promise that rejects after a tick): Express 4 does **not** catch it, because the throw happens after the synchronous `try/catch` has already returned. Result: an unhandled rejection and a hung request.

```js
app.get("/async", async (req, res) => {
  throw new Error("boom"); // Express 4: unhandled rejection, request hangs. ❌
});
```

The fix in Express 4 is to surface async errors yourself via `next(err)` (try/catch or an `asyncHandler` wrapper). **Express 5 closes this gap** by awaiting your handler and forwarding rejections automatically.

```js
// Express 4 fix
app.get("/async", async (req, res, next) => {
  try {
    throw new Error("boom");
  } catch (err) {
    next(err); // now the error handler runs
  }
});
```

> WARNING — Pitfall: Setting up a `process.on("unhandledRejection")` does not fix the hung request — it only lets you log/crash. The request that triggered it still never gets a response. You must `next(err)`.

**Follow-up:** *Why does the synchronous case work but not the async one?* The `try/catch` around the handler only spans the synchronous call. By the time an async error fires, that frame is gone.

---

### 30. How would you implement graceful shutdown and where does Express fit in?

Graceful shutdown means: stop accepting new connections, let in-flight requests finish, close resources (DB, etc.), then exit. Express itself doesn't manage this — you operate on the underlying **HTTP server** returned by `app.listen()`.

```js
const app = require("./app");
const server = app.listen(3000); // returns the Node http.Server

async function shutdown(signal) {
  console.log(`${signal} received, shutting down gracefully...`);

  // 1. Stop accepting new connections; callback fires when existing ones finish.
  server.close(async () => {
    try {
      await mongoose.connection.close(); // 2. close DB and other resources
      console.log("Cleanup complete. Exiting.");
      process.exit(0);
    } catch (err) {
      console.error("Error during cleanup", err);
      process.exit(1);
    }
  });

  // 3. Safety net: force-exit if shutdown stalls (e.g., a hung keep-alive socket).
  setTimeout(() => {
    console.error("Forced shutdown after timeout");
    process.exit(1);
  }, 10_000).unref(); // unref so this timer doesn't keep the process alive on its own
}

process.on("SIGTERM", () => shutdown("SIGTERM")); // e.g., orchestrators/Docker stop
process.on("SIGINT", () => shutdown("SIGINT"));   // Ctrl-C
```

Why it matters: in containerized/PaaS environments, the platform sends `SIGTERM` and gives you a grace period before `SIGKILL`. Without graceful shutdown you drop in-flight requests and may corrupt state.

> BEST PRACTICE: Pair `server.close()` with a forced-exit timeout. `server.close()` alone won't terminate lingering keep-alive connections, so a hung socket could block exit forever.

**Follow-up:** *Why separate `app.js` from `server.js` here?* So `app` (the configured Express app) can be imported by tests without binding a port, while `server.js` owns the listener and shutdown logic.

---

### 31. How do you secure an Express application? Name the essentials.

A senior answer covers defense in depth across several layers:

- **`helmet()`** — sets security-related HTTP headers (CSP, HSTS, `X-Content-Type-Options`, etc.) with sane defaults.
- **CORS** — configure `cors()` with an explicit allowlist of origins, not `*`, for credentialed APIs.
- **Rate limiting** — `express-rate-limit` to blunt brute-force and DoS attempts.
- **Input validation & sanitization** — validate every `req.body`/`params`/`query` (e.g., `zod`, `joi`, `express-validator`); reject unexpected fields.
- **Body size limits** — `express.json({ limit: "10kb" })` to prevent payload-based DoS.
- **NoSQL/SQL injection prevention** — sanitize inputs; with Mongo, guard against operator injection (`$gt`, etc. in query objects).
- **Secure cookies** — `httpOnly`, `secure`, `sameSite` flags on session/auth cookies.
- **HTTPS + `trust proxy`** — terminate TLS and set `app.set("trust proxy", 1)` behind a reverse proxy so `req.secure`/`req.ip` are correct.
- **Don't leak errors** — never send stack traces to clients; disable the `X-Powered-By` header (`app.disable("x-powered-by")` or via helmet).
- **Dependency hygiene** — `npm audit`, keep packages patched.

```js
const helmet = require("helmet");
const rateLimit = require("express-rate-limit");

app.disable("x-powered-by");                 // hide framework fingerprint
app.use(helmet());                           // secure headers
app.use(express.json({ limit: "10kb" }));    // cap body size
app.use("/api", rateLimit({ windowMs: 15 * 60 * 1000, max: 100 })); // throttle
app.use(require("cors")({ origin: ["https://app.example.com"], credentials: true }));
```

> WARNING — Pitfall: `cors({ origin: "*", credentials: true })` is invalid per the CORS spec and browsers will reject it — wildcard origins can't be combined with credentials.

**Follow-up:** *What's the simplest, highest-impact security middleware?* `helmet()` — it's one line and fixes a class of header-based vulnerabilities immediately.

---

### 32. How do you enable and correctly configure trust proxy, and why does it matter?

When your app runs **behind a reverse proxy or load balancer** (Nginx, a cloud LB, Heroku), the proxy is the direct TCP peer — so `req.ip` would be the *proxy's* IP and `req.protocol` would be `http` even when the client used HTTPS. The real client info is carried in `X-Forwarded-*` headers.

`app.set("trust proxy", …)` tells Express to **trust those forwarded headers** and derive the true client IP/protocol from them.

```js
app.set("trust proxy", 1); // trust the first hop (one proxy in front of us)
// Now:
//   req.ip       -> real client IP (from X-Forwarded-For)
//   req.protocol -> "https" when the client used HTTPS (from X-Forwarded-Proto)
//   req.secure   -> true accordingly
```

Configuration options:

| Value | Meaning |
|-------|---------|
| `false` (default) | Don't trust any proxy headers |
| `true` | Trust all proxies (use only if every hop is trusted) |
| `1` (a number) | Trust *n* hops closest to the app |
| `"loopback"` / a subnet / IP list | Trust specific addresses |
| A function | Custom predicate per address |

Why it matters: rate limiters, audit logs, and `secure`-cookie logic all rely on the correct client IP/protocol. Getting this wrong means everyone appears to share the proxy's IP (breaking per-IP rate limiting) or HTTPS detection fails.

> WARNING — Pitfall: Setting `trust proxy` to `true` blindly lets a malicious client spoof `X-Forwarded-For` and forge their IP. Trust only the specific number of hops you actually have.

**Follow-up:** *How does this interact with `express-rate-limit`?* The limiter keys on `req.ip`; without correct `trust proxy`, all clients share one IP and either everyone gets blocked together or limiting is ineffective.

---

### 33. What is content negotiation, and how does Express support it?

Content negotiation lets the **server respond in the format the client prefers**, based on the request's `Accept` header. Express supports it via `req.accepts()` and `res.format()`.

```js
app.get("/users/:id", (req, res) => {
  const user = { id: req.params.id, name: "Ann" };

  res.format({
    "application/json": () => res.json(user),                 // Accept: application/json
    "text/html": () => res.send(`<p>${user.name}</p>`),       // Accept: text/html
    "text/plain": () => res.send(`${user.name}`),             // Accept: text/plain
    default: () => res.status(406).send("Not Acceptable"),    // none matched
  });
});
```

- `res.format(obj)` picks the handler matching the client's `Accept` header (in quality order) and sets `Content-Type` and the `Vary: Accept` header automatically.
- `req.accepts(["json", "html"])` returns the best match or `false`.
- If nothing matches and there's no `default`, Express responds `406 Not Acceptable`.

For most JSON APIs you won't bother — you just return JSON. Content negotiation shines when one endpoint serves multiple representations (JSON + HTML + CSV).

> TIP: Mention the `Vary` header — telling caches that the response varies by `Accept` is essential for correctness behind a CDN.

**Follow-up:** *What status code signals "I can't produce any format you accept"?* 406 Not Acceptable.

---

### 34. How would you implement API versioning in Express, and what are the trade-offs?

Three common strategies:

**1. URI path versioning** (most common, most explicit):

```js
app.use("/api/v1", require("./routes/v1"));
app.use("/api/v2", require("./routes/v2")); // v2 mounts a separate router tree
```

**2. Header / `Accept` versioning** (cleaner URLs, harder to test in a browser):

```js
app.use("/api", (req, res, next) => {
  const version = req.get("Accept-Version") || "1"; // or a custom Accept media type
  req.apiVersion = version;
  next();
});
```

**3. Query parameter versioning** (`/api/users?version=2`) — simple but unconventional and easy to forget.

| Strategy | Pros | Cons |
|----------|------|------|
| URI path | Explicit, cacheable, easy to test/browse | "Pollutes" the URL; versions whole namespace |
| Header | Clean URLs, RESTful purist-approved | Invisible in browser, harder to cache/debug |
| Query param | Trivial to add | Easy to omit; weak caching semantics |

In practice, **URI path versioning wins** for most teams because it's visible, cache-friendly, and trivial to route — you mount an entirely separate router per version and can deprecate old ones cleanly.

> BEST PRACTICE: Version at the boundary you expect to break. Keep shared logic in services and let the version-specific routers/controllers adapt the request/response shape, so you don't duplicate business logic per version.

**Follow-up:** *When do you actually need a new version?* Only for **breaking** changes (removing/renaming fields, changing semantics). Additive changes (new optional fields/endpoints) don't require a version bump.

---

### 35. How do you test an Express app, and why structure it to make testing easy?

The standard tool is **Supertest**, which spins up your app on an ephemeral port (or in-memory) and lets you make real HTTP assertions. This is why separating `app.js` (exports the app) from `server.js` (calls `listen`) matters — Supertest imports the app directly, no port binding needed.

```js
// app.js exports the configured app WITHOUT calling listen()
// test/users.test.js
const request = require("supertest");
const app = require("../src/app");

describe("GET /users/:id", () => {
  it("returns 404 for an unknown id", async () => {
    const res = await request(app).get("/users/000000000000000000000000");
    expect(res.status).toBe(404);
    expect(res.body).toHaveProperty("error");
  });

  it("creates a user and returns 201", async () => {
    const res = await request(app)
      .post("/users")
      .send({ name: "Ann", email: "ann@example.com" }) // sets JSON content-type
      .expect(201);
    expect(res.body).toMatchObject({ name: "Ann" });
    expect(res.headers.location).toBeDefined(); // Location header on create
  });
});
```

Testing layers:

- **Unit tests** — services/utilities in isolation (no `req`/`res`), fast.
- **Integration tests** — Supertest hitting real routes through the full middleware stack, with a test database (e.g., `mongodb-memory-server`).
- **Contract/E2E** — the running server.

> BEST PRACTICE: Make controllers thin and put logic in services so unit tests don't need HTTP at all; reserve Supertest for verifying routing, middleware, status codes, and serialization.

**Follow-up:** *Why not call `app.listen()` in `app.js`?* Because importing it would bind a port, causing conflicts and slow, flaky tests. Export the app; listen elsewhere.

---

### 36. What happens under the hood when a request hits an Express app? Trace the lifecycle.

A senior-level trace, end to end:

1. **Node receives the request.** The `http.Server` (created by `app.listen()`) emits a `request` event with native `req`/`res`. Express is the registered handler.
2. **Express augments `req`/`res`.** Their prototypes are swapped so Express helpers (`req.params`, `res.json`, …) are available alongside native methods.
3. **The router walks its layer stack.** Express iterates registered layers (middleware + routes) **in order**. For each, it checks:
   - Does the **method** match (for `METHOD` routes)?
   - Does the **path** match the layer's compiled RegExp (`path-to-regexp`)? If so, it extracts `req.params`.
4. **Matching middleware runs one at a time.** Each either:
   - Sends a response (ending the cycle), or
   - Calls `next()` to advance to the next matching layer, or
   - Calls `next(err)` to jump to the error-handling chain.
5. **Body parsers / other middleware populate `req`.** e.g., `express.json()` reads the body stream and sets `req.body`.
6. **The route handler executes**, typically delegating to a controller/service, then calls a responder (`res.json`, etc.).
7. **If nothing matched**, control reaches the **404 middleware** at the end.
8. **If any layer errored**, control jumps to the **error-handling middleware**, which formats and sends the error response.
9. **`res` is flushed** back through Node's `http.ServerResponse` to the client.

The two engines to name: **`path-to-regexp`** (compiles route strings to RegExps and extracts params) and the **layer stack** iterated by a `next()` callback that advances an index.

> TIP: Framing the whole thing as "an ordered stack of `(req, res, next)` functions, driven by `next`" demonstrates real understanding rather than memorized API surface.

**Follow-up:** *What data structure backs routing?* An ordered array of "layers," each with a compiled path regex and a handler; `next()` advances an index through it.

---

### 37. Compare session-based and token-based (JWT) auth in the context of Express middleware.

Both are implemented as **authentication middleware** that runs before protected routes and attaches the identity to `req.user`. They differ in where state lives.

| Aspect | Session-based | Token-based (JWT) |
|--------|---------------|-------------------|
| Where state lives | Server-side store (Redis/DB), client holds a session ID cookie | Self-contained signed token, client stores it |
| Express setup | `express-session` + a store + cookie | Verify the `Authorization: Bearer` token per request |
| Scalability | Needs a shared store across instances | Stateless — any instance can verify |
| Revocation | Easy — delete the session | Hard — must use short expiries + a denylist |
| Typical client | Browser (cookies) | SPAs, mobile, service-to-service |
| Vulnerabilities | CSRF (cookie auto-sent) | XSS if stored in JS-accessible storage |

```js
// JWT verification middleware
const jwt = require("jsonwebtoken");

function authenticate(req, res, next) {
  const header = req.get("Authorization") || "";
  const token = header.startsWith("Bearer ") ? header.slice(7) : null;
  if (!token) return res.status(401).json({ error: "Missing token" });

  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET); // throws if invalid/expired
    next();
  } catch {
    res.status(401).json({ error: "Invalid or expired token" });
  }
}

app.get("/me", authenticate, (req, res) => res.json(req.user));
```

Both patterns share the same Express shape: middleware verifies identity → sets `req.user` → calls `next()` (or returns 401). Authorization then layers on top (e.g., `requireRole("admin")`).

> WARNING — Pitfall: Don't store JWTs in `localStorage` if you can avoid it (XSS-readable). For browsers, an `httpOnly` cookie with `sameSite` is safer — at which point you also need CSRF protection.

**Follow-up:** *How do you "log out" a JWT?* You can't truly invalidate it server-side without state; use short lifetimes + refresh tokens, and a denylist for forced revocation.

---

### 38. What's the difference between `app.use(path, router)` and chaining middleware on a route with `router.route()`?

Two related but distinct organizational tools.

**`app.use(path, router)`** *mounts* a whole router under a prefix:

```js
app.use("/users", usersRouter); // every route in usersRouter is prefixed with /users
```

**`router.route(path)`** groups multiple HTTP methods for the **same path** in one place, avoiding repetition:

```js
const router = require("express").Router();

router
  .route("/:id")                 // define the path once
  .get(getUser)                  // GET    /:id
  .patch(updateUser)             // PATCH  /:id
  .delete(deleteUser);           // DELETE /:id

// Equivalent without route():
// router.get("/:id", getUser);
// router.patch("/:id", updateUser);
// router.delete("/:id", deleteUser);
```

`router.route()` is purely ergonomic — it returns a single route object you attach method handlers to, keeping all verbs for one URL together and stating the path once. It pairs naturally with per-route middleware:

```js
router.route("/:id")
  .all(authenticate)            // runs for all methods on /:id
  .get(getUser)
  .delete(requireRole("admin"), deleteUser);
```

**Follow-up:** *What does `.all()` do on a route?* Registers a handler that runs for every HTTP method on that path — handy for shared per-route middleware like auth.

---

### 39. How do you handle file uploads in Express, and why can't `express.json()` do it?

File uploads use the `multipart/form-data` content type, which encodes files and fields as separate parts with boundaries — a fundamentally different wire format than JSON or URL-encoded data. `express.json()` and `express.urlencoded()` only understand their respective types, so they leave `req.body` empty for multipart and don't expose files at all.

The standard solution is **multer**, middleware that parses multipart, writes files (to disk or memory), and populates `req.file`/`req.files` plus `req.body` for text fields.

```js
const multer = require("multer");

// Store on disk with size and type limits.
const upload = multer({
  dest: "uploads/",
  limits: { fileSize: 5 * 1024 * 1024 }, // 5 MB cap (DoS protection)
  fileFilter: (req, file, cb) => {
    const ok = ["image/png", "image/jpeg"].includes(file.mimetype);
    cb(ok ? null : new Error("Only PNG/JPEG allowed"), ok);
  },
});

// `avatar` matches the form field name; single file.
app.post("/avatar", upload.single("avatar"), (req, res) => {
  res.json({ file: req.file, fields: req.body }); // req.file = upload metadata
});

// Multiple files:
app.post("/gallery", upload.array("photos", 10), (req, res) => {
  res.json({ count: req.files.length });
});
```

Considerations: always set `limits.fileSize`, validate MIME types/extensions, and for production prefer streaming directly to object storage (S3) rather than local disk.

> WARNING — Pitfall: Trusting the client-provided `file.mimetype` alone is weak — it can be spoofed. For untrusted uploads, verify the actual file signature (magic bytes) server-side.

**Follow-up:** *Disk storage vs memory storage in multer?* Disk is safe for large files; memory (`multer.memoryStorage()`) is convenient for small files you immediately forward to S3, but large files in memory risk OOM.

---

### 40. How do you configure CORS correctly in Express?

CORS (Cross-Origin Resource Sharing) is a **browser** security mechanism: by default a page can't read responses from a different origin. The server opts in by sending `Access-Control-Allow-*` headers. The `cors` middleware manages those headers, including the preflight `OPTIONS` request.

```js
const cors = require("cors");

// Explicit allowlist (recommended for anything with credentials).
const allowed = ["https://app.example.com", "https://admin.example.com"];

app.use(
  cors({
    origin: (origin, cb) => {
      // origin is undefined for same-origin / non-browser (curl) requests.
      if (!origin || allowed.includes(origin)) return cb(null, true);
      cb(new Error("Not allowed by CORS"));
    },
    credentials: true,                       // allow cookies/Authorization to be sent
    methods: ["GET", "POST", "PUT", "DELETE"],
    allowedHeaders: ["Content-Type", "Authorization"],
  })
);
```

Key points:

- A **preflight** `OPTIONS` request is sent by the browser before "non-simple" requests; the `cors` middleware answers it automatically when registered with `app.use`.
- `credentials: true` (for cookies/auth) requires a **specific** origin in `Access-Control-Allow-Origin` — `*` is invalid with credentials.
- CORS is enforced by the **browser**, not the server — it does not protect against non-browser clients (curl, server-to-server). It's not a substitute for authentication.

> WARNING — Pitfall: Slapping `app.use(cors())` with default `origin: "*"` on an authenticated API is a common mistake — fine for public read-only APIs, dangerous with credentials. Use an allowlist.

**Follow-up:** *Why does my preflight fail with a custom header?* The header must be listed in `allowedHeaders` (which becomes `Access-Control-Allow-Headers`); otherwise the browser blocks the actual request.

---

### 41. Predict the output: how do these routes resolve?

```js
const express = require("express");
const app = express();

app.get("/users/:id", (req, res) => {
  res.send(`user ${req.params.id}`);
});

app.get("/users/me", (req, res) => {
  res.send("current user");
});

app.listen(3000);
```

**A request to `GET /users/me` responds with `"user me"`, not `"current user"`.**

Because Express matches **in registration order** and `/users/:id` is registered first, `:id` happily captures the literal segment `"me"`. The more specific `/users/me` route is **never reached** — it's shadowed.

Fix: register the specific route **before** the dynamic one:

```js
app.get("/users/me", (req, res) => res.send("current user")); // specific FIRST
app.get("/users/:id", (req, res) => res.send(`user ${req.params.id}`)); // dynamic second
```

Now `/users/me` matches the literal route, and `/users/42` falls through to `:id`. This "specific-before-dynamic" ordering rule is one of the most practically important things to internalize about Express routing.

> BEST PRACTICE: Order routes from most specific to most general within a resource, and consider validating `:id` format (e.g., a numeric or ObjectId regex constraint) so literal aliases can't be misinterpreted.

**Follow-up:** *Can you constrain `:id` to digits so `me` won't match it?* In Express 4 you could use a regex in the param (`/users/:id(\\d+)`); Express 5 changed/tightened regex-in-path support, so prefer validating inside the handler or via middleware.

---

### 42. How do you avoid blocking the event loop in an Express handler, and why does it matter?

Node (and therefore Express) is **single-threaded for JavaScript**: one event loop processes all requests. A synchronous, CPU-heavy operation in any handler **blocks every other request** until it finishes — the server appears frozen for all clients, not just the one making the expensive call.

```js
// BAD: synchronous CPU work blocks the whole server for every other request.
app.get("/report", (req, res) => {
  const result = crunchHugeDataSetSync(); // blocks the event loop for, say, 2s
  res.json(result);
});
```

Strategies to keep the loop free:

- **Use async I/O.** File, DB, and network calls should be `await`ed (non-blocking) — never use `fs.readFileSync` etc. in request handlers.
- **Offload CPU-bound work** to a **Worker Thread** (`worker_threads`), a child process, or a separate job queue/worker service (BullMQ/Redis).
- **Stream large responses** instead of buffering everything in memory.
- **Paginate** large result sets rather than returning everything.
- **Cache** expensive computations (in-memory LRU or Redis).

```js
// BETTER: push CPU-heavy work to a worker and await the result.
const { Worker } = require("worker_threads");

app.get("/report", (req, res, next) => {
  const worker = new Worker("./report-worker.js"); // runs on another thread
  worker.once("message", (result) => res.json(result));
  worker.once("error", next);
});
```

> WARNING — Pitfall: Even seemingly innocent things — a giant `JSON.parse`, a huge synchronous regex, `bcrypt.hashSync` in a request path — can block the loop. Prefer the async variants (`bcrypt.hash`) and measure.

**Follow-up:** *Does adding more Express routes add concurrency?* No — concurrency comes from non-blocking I/O and from scaling processes (the `cluster` module / PM2) or threads, not from the number of routes.

---

### 43. How do you scale an Express app across CPU cores?

A single Node process uses **one CPU core** for JavaScript. To use a multi-core machine you run **multiple processes** and put a load balancer in front.

- **`cluster` module** — fork one worker per core; the OS/cluster balances incoming connections across workers that share the listening port.
- **PM2** — a production process manager that does clustering, restarts on crash, zero-downtime reloads, and log management for you (the common real-world choice).
- **Container orchestration** — run N single-process containers behind a load balancer (Kubernetes, ECS). Often preferred over in-app clustering in cloud deployments.

```js
// Minimal cluster example: one worker per CPU core.
const cluster = require("cluster");
const os = require("os");

if (cluster.isPrimary) {
  for (let i = 0; i < os.availableParallelism(); i++) cluster.fork(); // one per core
  cluster.on("exit", (worker) => {
    console.log(`Worker ${worker.process.pid} died; restarting`);
    cluster.fork(); // keep the pool full
  });
} else {
  require("./server"); // each worker runs the Express server
}
```

Important consequence: **with multiple processes, in-memory state is not shared.** Sessions, rate-limit counters, and caches must move to a **shared store** (Redis) or each worker will have its own inconsistent copy.

> BEST PRACTICE: In cloud/container environments, prefer "one process per container, scale containers horizontally" over in-app `cluster`; let the orchestrator handle balancing and restarts.

**Follow-up:** *What breaks when you move from one process to many?* Anything relying on per-process memory: sessions (use Redis store), WebSocket rooms (use a Redis adapter), rate limiters (use a shared store), and in-memory caches.

---

## Rapid-Fire Quick Answers

| Question | One-line answer |
|----------|-----------------|
| What is Express? | A minimal, unopinionated web framework layered on Node's `http` module, adding routing, middleware, and req/res helpers. |
| Middleware signature? | `(req, res, next)` — or `(err, req, res, next)` for error handlers. |
| How does Express identify an error handler? | By arity — a middleware with **four** declared parameters. |
| `next()` vs `next(err)`? | `next()` advances the normal chain; `next(err)` jumps to error-handling middleware. |
| What if middleware never calls `next()` or responds? | The request hangs until the client times out. |
| `req.params` vs `req.query` vs `req.body`? | Path segments / query string / request body (body needs a parser). |
| Enable JSON body parsing? | `app.use(express.json())` (built in since Express 4.16). |
| Serve static files? | `app.use(express.static(path.join(__dirname, "public")))`. |
| `app.use` vs `app.get`? | `use` = all methods + prefix match (middleware); `get` = GET + route pattern. |
| Where do 404 and error handlers go? | Last in the middleware stack, after all routes. |
| `app.locals` vs `res.locals`? | App-wide (all requests) vs per-request scope. |
| Status for create / delete? | 201 (with `Location`) / 204 (No Content). |
| 401 vs 403? | 401 = not authenticated; 403 = authenticated but not allowed. |
| Express 4 async error gotcha? | Rejected promises aren't auto-caught; use try/catch or an `asyncHandler` (Express 5 fixes this). |
| Biggest Express 4 → 5 changes? | Auto async-error forwarding, named wildcards (`/*splat`), removed legacy APIs (`req.param`, `app.del`, `res.json(status, …)`). |
| `mergeParams: true`? | Lets a child router see params from the parent's mount path. |
| Why split `app.js` and `server.js`? | So tests (Supertest) import the app without binding a port. |
| Make config-driven middleware? | A factory: a function that takes options and returns a `(req, res, next)`. |
| Scale across cores? | Multiple processes via `cluster`/PM2/containers behind a load balancer; move shared state to Redis. |
| `req.path` vs `req.originalUrl`? | `path` is relative to the current router; `originalUrl` is the full URL the client requested. |
