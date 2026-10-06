# Error Handling & Input Validation

Almost every production incident you will ever debug is, at root, one of two things: an error that nobody handled, or input that nobody validated. The two topics are joined at the hip, which is why this module covers them together. Bad input *causes* errors; unhandled errors *leak* details about your system and crash your process. Get these two things right and your API becomes the thing every team wishes theirs was — predictable, debuggable, and safe.

This is also the module that most clearly separates a junior backend developer from a senior one. Juniors sprinkle `try/catch` everywhere, return whatever shape feels convenient at the moment, and let stack traces escape to the client. Seniors build a small, deliberate *error-handling architecture*: a single place where errors are translated into clean HTTP responses, a single class that describes "expected" errors, a validation layer that rejects garbage at the door, and process-level safety nets for the truly unexpected. By the end of this module you will be able to design exactly that architecture from memory, and explain every decision in an interview.

We target **Node.js current LTS** and **Express.js v5** (the default `express` on npm since 2024), with **MongoDB + Mongoose** for the database-error examples. Every place where Express v4 and v5 differ — and async error handling is the *big* one — is flagged explicitly.

## What You'll Learn

By the end of this module you will be able to:

- Distinguish **operational errors** from **programmer errors** and explain why that distinction drives your entire error-handling strategy.
- Use **`try/catch` with `async/await`** correctly, and explain precisely why an **unhandled promise rejection** is dangerous (and what Node does about it now).
- Build a **centralized Express error-handling middleware** using the special **4-argument signature**, and explain how errors reach it.
- Design a **custom `AppError` class** with `statusCode` and `isOperational`, and throw meaningful, intentional errors throughout your code.
- Eliminate repetitive `try/catch` with an **async error wrapper** (`catchAsync`) or the **`express-async-errors`** package, and know when each is needed (v4 vs v5).
- Install **process-level safety nets** for `unhandledRejection` and `uncaughtException`, and shut down **gracefully** instead of leaving a zombie process.
- Add a proper **404 not-found handler** that funnels into your central error handler.
- Produce **environment-aware error responses**: verbose and developer-friendly in dev, safe and minimal in production.
- Validate input with **express-validator** (validation chains + `validationResult`), **Joi**, and **Zod**, and articulate where each one fits.
- Build **reusable validation middleware** so a route's contract is declared in one line.
- **Map** validation errors, Mongoose errors (`CastError`, `ValidationError`, duplicate-key `11000`), and JWT errors into clean, consistent client responses.

> Note: This module assumes you have already met **middleware** (the `(req, res, next)` signature and the chain) and the **request–response cycle**. If `next(err)` and "error-handling middleware has four parameters" are not yet second nature, revisit the middleware module first — error handling *is* a middleware topic.

---

## 1. The Philosophy of Error Handling

Before a single line of code, you need a model of *what an error even is* in a server context. The single most important idea in this entire module is this distinction:

### Operational errors vs programmer errors

| | **Operational errors** | **Programmer errors** |
| --- | --- | --- |
| What they are | Expected runtime problems in a correct program | Bugs — mistakes in your code |
| Examples | Invalid user input, failed DB connection, 404, request timeout, file not found, "email already taken", expired JWT | `undefined is not a function`, reading a property of `undefined`, calling an async fn without `await`, passing wrong argument types, typos |
| Whose "fault" | The world's (network, user, external service) | Yours (the developer's) |
| Can you anticipate them? | Yes — they *will* happen, plan for them | No — if you anticipated it you'd have fixed it |
| Correct response | **Handle gracefully**: send a clean HTTP error, retry, log | **Crash and restart**: the process is in an unknown state |
| HTTP status | Usually 4xx (client's fault) or specific 5xx | Usually 500 (generic "something broke") |

This table is the philosophical core of the module. Re-read it. Everything else is implementation detail in service of it.

Why does the distinction matter so much? Because **operational and programmer errors call for opposite reactions**:

- **Operational errors are part of normal operation.** A user submitting a malformed email is not a crisis — it's Tuesday. You catch it, return `400 Bad Request` with a helpful message, and move on. The process keeps serving other requests.
- **Programmer errors mean your program is now in an unknown, untrustworthy state.** If you hit `Cannot read properties of undefined`, some assumption your code relied on is false. Memory might be corrupted; a half-finished operation might have left data inconsistent. The *only* safe thing to do is **crash the process and let a process manager restart it clean.** Trying to "recover" from a bug by catching it and continuing is how you get silent data corruption.

> BEST PRACTICE: Treat operational errors as data you handle, and programmer errors as a reason to crash and restart. The `isOperational` flag on our custom error class (§5) exists precisely to mark "this one is expected — handle it, don't crash."

### The goals of an error-handling strategy

A good strategy achieves five things at once:

1. **The process stays alive** through operational errors (one bad request must never take down the server for everyone).
2. **The process restarts cleanly** after programmer errors (a process manager like `pm2`, `systemd`, or a container orchestrator brings it back).
3. **The client gets a consistent, useful, *safe* response** — same JSON shape every time, helpful in dev, leak-free in prod.
4. **You get a great log** — full stack trace, request context — so you can actually fix the bug.
5. **There is one place** that owns turning errors into responses, so you are not repeating yourself in 80 route handlers.

Keep these five goals in mind; each section below advances one or more of them.

---

## 2. `try/catch` with `async/await`, and the Danger of Unhandled Rejections

### `try/catch` only catches *synchronous* throws (and awaited rejections)

A plain `try/catch` block catches an exception only if it is thrown **synchronously within the `try` block**. This is the trap that bites everyone moving from sync to async code:

```js
// ❌ This does NOT catch the error. The callback throws LATER,
// long after the try/catch has finished executing.
try {
  setTimeout(() => {
    throw new Error('boom'); // thrown on a future tick — try/catch is long gone
  }, 100);
} catch (err) {
  console.log('caught:', err.message); // never runs
}
// Result: an uncaught exception that can crash the process.
```

The fix in modern Node is `async/await`, because `await` *re-synchronizes* a promise rejection into a throw that `try/catch` can see:

```js
// ✅ await turns a rejected promise into a throw at this exact line,
// so the surrounding try/catch catches it.
async function loadUser(id) {
  try {
    const user = await db.users.findById(id); // if this rejects...
    return user;
  } catch (err) {
    // ...we land here, with err = the rejection reason.
    console.error('DB lookup failed:', err.message);
    throw err; // re-throw so the caller knows (don't swallow!)
  }
}
```

The mental model: **`await somePromise` is sugar for "wait for this promise; if it resolves, give me the value; if it rejects, *throw* the reason right here."** That "throw right here" is what makes `try/catch` work with async code.

### Why an unhandled rejection is dangerous

A **promise rejection** is "unhandled" when a promise rejects and there is no `.catch()` (and no surrounding `try/catch` around an `await`) to deal with it. Consider:

```js
// ❌ The classic mistake: calling an async function without awaiting
// it and without a .catch(). If it rejects, nobody is listening.
function handler(req, res) {
  saveToDatabase(req.body); // returns a promise we ignore — rejection floats away
  res.json({ ok: true });   // we even respond success before the save finished!
}
```

If `saveToDatabase` rejects, that rejection has nowhere to go. Historically this produced a warning; **as of recent Node.js versions the default behavior is to terminate the process** (`--unhandled-rejections=throw`). So an unhandled rejection is dangerous for two reasons:

1. **It can crash your process** — and unlike a deliberate crash on a programmer error, this one happens at an arbitrary time with no graceful shutdown, dropping in-flight requests.
2. **Even when it doesn't crash, it hides bugs.** A save silently failed, the client was told "ok," and your data is now inconsistent — with no error in your logs.

> WARNING — Pitfall: The most common source of unhandled rejections is **calling an async function and forgetting to `await` it (or attach `.catch`)**. This is sometimes called a "floating promise." Linters (`eslint-plugin-promise`, `@typescript-eslint/no-floating-promises` for TS) exist specifically to catch these. In plain JS, build the habit: *every* promise must be awaited, returned, or `.catch()`-ed.

### The rule of thumb

```js
// ✅ Three legitimate things to do with a promise — never a fourth ("ignore it"):
await doThing();                 // 1. await it (inside an async function)
return doThing();                // 2. return it (let the caller handle it)
doThing().catch(logError);       // 3. attach a catch handler
// doThing();                    // 4. ❌ floating promise — forbidden
```

We will see in §6 that for *Express route handlers specifically*, there are cleaner patterns than manually wrapping every handler in `try/catch`. But the underlying principle never changes: a rejection must always reach a handler.

---

## 3. Centralized Express Error-Handling Middleware (the 4-Argument Signature)

This is the keystone. Express has a built-in concept of an **error-handling middleware**: a middleware function that declares **four** parameters — `(err, req, res, next)` — instead of the usual three. Express detects it *by arity* (the number of declared parameters) and only sends errors to it.

```js
// The centralized error handler. Note FOUR parameters — that is what
// tells Express "this is an error handler," even if you don't use `next`.
function errorHandler(err, req, res, next) {
  console.error(err.stack);                 // log for the developer
  res.status(err.statusCode || 500).json({  // respond to the client
    status: 'error',
    message: err.message || 'Internal Server Error',
  });
}
```

### How an error reaches it

There are exactly two ways an error gets routed to your error-handling middleware:

1. **You call `next(err)`** with a truthy argument anywhere in the chain. Express skips all remaining *normal* middleware and routes/jumps straight to the error handler.
2. **In Express v5, a thrown error (or rejected promise) in a route handler is automatically passed to `next(err)` for you.** (In v4 this is true only for *synchronous* throws; async rejections must be forwarded manually — see §6.)

```js
// Two equivalent ways to trigger the error handler from a route:
app.get('/a', (req, res, next) => {
  next(new Error('forwarded explicitly')); // way #1
});

app.get('/b', (req, res) => {
  throw new Error('thrown — Express v5 forwards this for you'); // way #2 (sync)
});
```

### The rules of error-handling middleware

- It **must declare all four parameters**, in order `(err, req, res, next)`. Even if you never use `next`, you must list it — otherwise Express sees a 3-arity function and treats it as *normal* middleware.
- It **must be registered LAST**, after all routes and other middleware. Express tries middleware in registration order; the error handler can only catch errors from things registered *before* it.
- You can have **more than one** error-handling middleware (e.g., one that logs, then one that responds), chaining via `next(err)`. In practice, one well-structured handler is plenty.

> WARNING — Pitfall: If you write `(err, req, res)` with only three parameters, Express will treat it as a *regular* middleware and **never** send errors to it. Conversely, a normal middleware accidentally written with four parameters becomes an error handler that never runs on the happy path. Arity is everything here. When in doubt, count the parameters.

### Where it goes in the file

```js
const express = require('express');
const app = express();

app.use(express.json());          // body parsing
// ... all your routes ...
app.use('/api/users', userRouter);

// 404 handler (see §7) — registered AFTER all routes
app.use((req, res, next) => {
  next(new AppError(`Cannot find ${req.originalUrl} on this server`, 404));
});

// THE central error handler — registered DEAD LAST
app.use(errorHandler);

module.exports = app;
```

This ordering is non-negotiable: parsers → routes → 404 → error handler. We will flesh out `AppError` and `errorHandler` over the next few sections until they are production-grade.

---

## 4. A Custom `AppError` Class

The built-in `Error` carries a `message` and a `stack`, but it knows nothing about HTTP. We want an error that *also* carries an HTTP status code and a flag marking it as "operational" (expected) versus a programmer bug. Enter `AppError`.

```js
// utils/AppError.js
class AppError extends Error {
  /**
   * @param {string} message  Human-readable, client-safe message.
   * @param {number} statusCode  HTTP status code (e.g. 400, 401, 404).
   */
  constructor(message, statusCode) {
    super(message);                 // sets this.message and captures stack

    this.statusCode = statusCode;
    // Convention: 4xx = 'fail' (client's fault), 5xx = 'error' (server's fault)
    this.status = `${statusCode}`.startsWith('4') ? 'fail' : 'error';

    // The crucial flag: errors WE create on purpose are operational/expected.
    // Programmer errors (unexpected bugs) will NOT have this flag, so the
    // central handler can treat them differently (hide details in prod).
    this.isOperational = true;

    // Keep this constructor out of the stack trace so the trace points at
    // the line that actually threw, not at AppError itself.
    Error.captureStackTrace(this, this.constructor);
  }
}

module.exports = AppError;
```

### Why each piece matters

- **`extends Error`** — so `err instanceof Error` is true, `try/catch` works normally, and the stack trace is real.
- **`statusCode` and `status`** — let the central handler pick the right HTTP code and a `'fail'`/`'error'` label without re-deriving it.
- **`isOperational = true`** — this is the heart of the philosophy from §1 made concrete. *We* set this only on errors we deliberately create. A `TypeError` from a bug won't have it. In production, the central handler reveals messages for operational errors but hides details for non-operational ones (they might leak internals). See §8.
- **`Error.captureStackTrace(this, this.constructor)`** — a V8 feature that omits the `AppError` constructor frame from the stack, so the trace begins where you actually `throw`. Nice-to-have, not essential.

### Throwing meaningful errors

Now your business logic reads like a clear contract. Instead of vague failures, you throw intentional, well-typed errors:

```js
const AppError = require('../utils/AppError');

async function getUser(id) {
  const user = await User.findById(id);
  if (!user) {
    // Intentional, operational, with the right status code.
    throw new AppError(`No user found with id ${id}`, 404);
  }
  return user;
}

function transferFunds(account, amount) {
  if (amount <= 0) {
    throw new AppError('Transfer amount must be positive', 400);
  }
  if (amount > account.balance) {
    throw new AppError('Insufficient funds', 422); // 422 Unprocessable Entity
  }
  // ... perform transfer ...
}
```

> BEST PRACTICE: Throw `AppError` for every *expected* failure with a deliberate status code. Reserve plain `throw new Error(...)` (or letting native errors bubble) for situations you treat as bugs. This way `isOperational` cleanly partitions your errors, and your central handler can do the right thing automatically.

> TIP: You can add static factory methods for common cases to make call sites even tidier, e.g. `AppError.notFound('User')`, `AppError.badRequest(msg)`, `AppError.unauthorized()`. It is optional sugar but reads beautifully in route code.

---

## 5. An Async Error Wrapper (`catchAsync` / `express-async-errors`)

### The problem: try/catch in every handler

Without help, an async route handler in Express v4 must wrap everything in `try/catch` and forward via `next(err)`, or the rejection floats away (§2):

```js
// 😩 Repetitive and easy to forget. Every async handler needs this boilerplate.
app.get('/users/:id', async (req, res, next) => {
  try {
    const user = await getUser(req.params.id);
    res.json({ status: 'success', data: { user } });
  } catch (err) {
    next(err); // forward to the central handler
  }
});
```

Multiply that by every route and it becomes noise that hides the actual logic — and one forgotten `try/catch` is a floating promise waiting to crash the process.

### Solution A: a `catchAsync` higher-order function

We write a tiny wrapper once. It takes an async handler and returns a new handler that automatically forwards any rejection to `next`:

```js
// utils/catchAsync.js
// Takes an async (req,res,next) function and returns a function that runs it
// and routes any rejection to Express's error pipeline via next(err).
module.exports = (fn) => (req, res, next) => {
  // fn(...) returns a promise; if it rejects, .catch(next) forwards the error.
  // Equivalent to .catch(err => next(err)).
  Promise.resolve(fn(req, res, next)).catch(next);
};
```

Now handlers are clean — no `try/catch`, no `next(err)`, and impossible to forget:

```js
const catchAsync = require('../utils/catchAsync');

// ✅ Clean: just the logic. Rejections are forwarded automatically.
app.get('/users/:id', catchAsync(async (req, res) => {
  const user = await getUser(req.params.id); // throws AppError(404) if missing
  res.json({ status: 'success', data: { user } });
}));
```

`Promise.resolve(fn(...))` is defensive: it works whether `fn` returns a promise (async) or not (sync that throws would still be caught by Express directly, but this keeps behavior uniform).

### Solution B: the `express-async-errors` package

`express-async-errors` is a one-line import that *monkey-patches* Express's router so that **any** async handler's rejection is automatically forwarded — no per-route wrapper needed:

```js
// Import for side effects ONCE, at the very top of your entry file,
// before you create the app. It patches Express globally.
require('express-async-errors');

const express = require('express');
const app = express();

// Now even bare async handlers are safe — rejections auto-forward.
app.get('/users/:id', async (req, res) => {
  const user = await getUser(req.params.id); // rejection -> central handler
  res.json({ status: 'success', data: { user } });
});
```

### Which approach, and the v4-vs-v5 picture

| Approach | Express v4 | Express v5 | Notes |
| --- | --- | --- | --- |
| Manual `try/catch` + `next(err)` | Works | Works | Verbose; error-prone |
| `catchAsync` wrapper | **Recommended** | Optional (still tidy) | Explicit, zero deps, you control it |
| `express-async-errors` | Common | Largely redundant | Global monkey-patch; one import |
| Nothing (bare async handler) | ❌ floats/crashes | ✅ **auto-forwarded** | v5's built-in async support |

The headline difference: **Express v5 natively forwards rejected promises from async route handlers to your error-handling middleware.** That means in v5 you can often skip both the wrapper and the package. Many teams still use `catchAsync` even on v5 because it is explicit, dependency-free, and makes the intent obvious at every call site.

> WARNING — Pitfall: In Express **v4**, a bare `async` handler that rejects will *not* be caught by your error middleware — the rejection becomes an unhandled rejection. If you are on v4 and ever see "my error handler never runs for async routes," this is why. Use `catchAsync` or `express-async-errors`, or upgrade to v5.

> TIP: `catchAsync` only covers your async *route handlers*. Promises that reject *outside* the request lifecycle (a `setInterval` job, an event listener) still need their own `.catch()` and are backstopped by the process-level handlers in §6.

---

## 6. Process-Level Safety Nets: `unhandledRejection` and `uncaughtException`

Even with `catchAsync` everywhere, some errors escape the request cycle entirely — a background job, a third-party library emitting on a stream, a bug in code that runs at startup. Node gives the `process` object two events as last-resort safety nets. **These are nets, not strategies** — if they fire, something slipped through, and the right move is to log loudly and shut down gracefully.

### `unhandledRejection` — a promise rejected with no handler

```js
// At the top of your entry file (e.g. server.js).
const server = app.listen(process.env.PORT || 3000);

process.on('unhandledRejection', (reason, promise) => {
  console.error('UNHANDLED REJECTION! Shutting down...');
  console.error(reason); // log the actual rejection reason (often an Error)

  // Graceful shutdown: stop accepting new connections, finish in-flight ones,
  // THEN exit. server.close() invokes its callback once existing requests drain.
  server.close(() => {
    process.exit(1); // exit with a non-zero (failure) code
  });
});
```

### `uncaughtException` — a synchronous throw nobody caught

```js
// Register this as EARLY as possible, ideally before anything else runs.
process.on('uncaughtException', (err) => {
  console.error('UNCAUGHT EXCEPTION! Shutting down...');
  console.error(err.name, err.message, err.stack);

  // After an uncaught exception the process is in an undefined state.
  // Best practice is to exit IMMEDIATELY rather than try to drain connections,
  // because nothing about the program's state can be trusted now.
  process.exit(1);
});
```

### Why crash instead of recover?

This trips up beginners: "If I have a handler, why not keep running?" Because an `uncaughtException` (or an unhandled rejection that reflects a bug) means **an error reached the top of the stack with nobody handling it — your program's invariants are broken.** Continuing risks corrupted state, memory leaks, and hung connections. The official Node guidance is blunt: after an uncaught exception, **the only safe action is to restart the process.**

The pattern, then, is: **log → (optionally drain) → `process.exit(1)` → let a process manager restart you.** In production you *always* run behind something that restarts a crashed process:

- A container orchestrator (Kubernetes restarts the pod; Docker `--restart`).
- A process manager (`pm2`, `systemd`, `forever`).

A clean crash + automatic restart beats a zombie process serving corrupted responses every single time.

### Difference between the two events

| | `uncaughtException` | `unhandledRejection` |
| --- | --- | --- |
| Triggered by | A synchronous `throw` not caught anywhere | A rejected promise with no `.catch`/`await` handler |
| Typical cause | A bug in sync code, or thrown outside any try/catch | A floating promise (§2) |
| State after it fires | Undefined/untrustworthy | Untrustworthy if it's a bug; sometimes benign |
| Recommended action | Exit **immediately** | Drain then exit (slightly safer to finish requests) |

> BEST PRACTICE: Register `uncaughtException` *first*, before you `require` your app, so it can catch errors during startup. Register `unhandledRejection` after you have a reference to `server` so you can drain connections. Both should ultimately `process.exit(1)`.

> WARNING — Pitfall: Do **not** treat these handlers as a way to keep a buggy app alive. People who `console.log` in `uncaughtException` and then *don't* exit end up with a process that limps along leaking memory and serving garbage. Log, then exit. Always.

> TIP: There is also `process.on('SIGTERM', ...)` / `'SIGINT'` for *graceful shutdown on deploy* (when your orchestrator sends a stop signal). That is the same `server.close()` drain pattern but triggered intentionally, not by an error. Worth wiring up alongside these handlers.

---

## 7. A 404 Not-Found Handler

If a request matches no route, Express's default is a plain-text `Cannot GET /whatever` with a 404. That is inconsistent with your nice JSON errors. So we add a **catch-all middleware** *after* all real routes that manufactures an operational `AppError` and forwards it to the central handler — keeping one consistent response shape for *every* error, including "not found."

```js
const AppError = require('./utils/AppError');

// ... all real routes registered above this line ...

// Catch-all 404. In Express v5, an unmatched path that flows past every
// route reaches this middleware. Forward an AppError so the SAME central
// handler formats the response.
app.use((req, res, next) => {
  next(new AppError(`Cannot ${req.method} ${req.originalUrl} on this server`, 404));
});

// Central error handler comes LAST.
app.use(errorHandler);
```

> WARNING — Pitfall (v4 vs v5 routing): In Express **v4** you might see `app.all('*', ...)` for the 404 catch-all. In Express **v5** the path-matching library changed and a bare `'*'` string is no longer a valid path pattern — you must use a *named* wildcard like `app.all('/*splat', ...)` or, more simply, a parameter-less `app.use(...)` as shown above (which matches everything that reached it). The `app.use` form is the most version-robust. If you copy a v4 tutorial's `app.get('*', ...)` into v5, it will throw a path-to-regexp error at startup.

The beauty of routing 404s through `AppError` + the central handler: there is now exactly **one** code path that turns *any* failure (missing route, validation error, DB error, thrown bug) into a response. That uniformity is the whole point.

---

## 8. Consistent, Environment-Aware Error Responses

Now we make the central error handler production-grade. Two requirements pull in opposite directions:

- **In development**, you want *everything*: the message, the stack trace, the full error object — so you can debug fast.
- **In production**, you want the *opposite* for non-operational errors: never leak a stack trace, internal paths, SQL/Mongo details, or library internals to a client. Leaking these is a real security issue (it hands attackers a map of your stack). For operational errors you *do* send the message (it's client-safe by design); for everything else you send a generic "Something went wrong."

Here is the full handler, environment-aware and built on `AppError`:

```js
// controllers/errorController.js

// --- DEV: be loud and complete ---
function sendErrorDev(err, res) {
  res.status(err.statusCode || 500).json({
    status: err.status || 'error',
    error: err,                 // the whole error object
    message: err.message,
    stack: err.stack,           // full stack trace for the developer
  });
}

// --- PROD: be safe ---
function sendErrorProd(err, res) {
  if (err.isOperational) {
    // Expected error we created on purpose: the message is client-safe.
    res.status(err.statusCode).json({
      status: err.status,
      message: err.message,
    });
  } else {
    // Programmer error or unknown library error: DON'T leak details.
    console.error('UNEXPECTED ERROR 💥', err); // but DO log it for ourselves
    res.status(500).json({
      status: 'error',
      message: 'Something went wrong',
    });
  }
}

// The central handler. Four params => Express treats it as an error handler.
module.exports = (err, req, res, next) => {
  err.statusCode = err.statusCode || 500;
  err.status = err.status || 'error';

  if (process.env.NODE_ENV === 'development') {
    sendErrorDev(err, res);
  } else {
    // In prod we may also "translate" certain native errors into operational
    // AppErrors first (see §9), so their messages become safe to send.
    let error = mapKnownErrors(err);   // defined in §9
    sendErrorProd(error, res);
  }
};
```

### The consistent response shape

Pick one JSON envelope and use it *everywhere* — success and error alike. A widely used shape:

```json
// Error response (production, operational error)
{
  "status": "fail",
  "message": "No user found with id 64f0c2..."
}
```

```json
// Error response (development) — same top fields PLUS debug info
{
  "status": "fail",
  "message": "No user found with id 64f0c2...",
  "error": { "statusCode": 404, "status": "fail", "isOperational": true },
  "stack": "AppError: No user found...\n    at getUser (/app/..."
}
```

> BEST PRACTICE: Define the success envelope to mirror the error one, e.g. `{ "status": "success", "data": { ... } }`. When success and error responses share a shape, frontends can branch on `status` and never have to guess. Consistency is a *feature*.

> TIP: `NODE_ENV` should be set to `'production'` in prod (most platforms do this for you) and `'development'` locally. Never trust the *default* — explicitly check, and consider defaulting to the *safe* (prod) branch if `NODE_ENV` is unset, so a misconfiguration fails closed, not open.

---

## 9. Mapping Mongoose & JWT Errors to Clean Responses

Native errors from Mongoose, the MongoDB driver, and `jsonwebtoken` are *not* operational `AppError`s — they have cryptic messages and the wrong status codes. In production they would hit the "Something went wrong / 500" branch, which is unhelpful for what are really **client mistakes (400/401)**. The fix: a translation layer that converts known native errors into clean, operational `AppError`s *before* the prod responder runs.

```js
const AppError = require('../utils/AppError');

// Mongoose: bad ObjectId / type mismatch -> 400
function handleCastErrorDB(err) {
  const message = `Invalid ${err.path}: ${err.value}`;
  return new AppError(message, 400);
}

// MongoDB driver: duplicate unique key (code 11000) -> 400
function handleDuplicateFieldsDB(err) {
  // err.keyValue is { email: 'taken@x.com' } on modern drivers.
  const field = Object.keys(err.keyValue || {})[0];
  const value = err.keyValue ? err.keyValue[field] : 'value';
  const message = `Duplicate field ${field}: "${value}". Please use another value.`;
  return new AppError(message, 400);
}

// Mongoose: schema validation failed -> 400, with all messages joined.
function handleValidationErrorDB(err) {
  const messages = Object.values(err.errors).map((e) => e.message);
  const message = `Invalid input data. ${messages.join('. ')}`;
  return new AppError(message, 400);
}

// jsonwebtoken: invalid signature / malformed token -> 401
function handleJWTError() {
  return new AppError('Invalid token. Please log in again.', 401);
}

// jsonwebtoken: token expired -> 401
function handleJWTExpiredError() {
  return new AppError('Your token has expired. Please log in again.', 401);
}

// The translator: takes any error, returns an AppError if we recognize it,
// otherwise returns the original error unchanged.
function mapKnownErrors(err) {
  let error = err;
  if (err.name === 'CastError')        error = handleCastErrorDB(err);
  if (err.code === 11000)              error = handleDuplicateFieldsDB(err);
  if (err.name === 'ValidationError')  error = handleValidationErrorDB(err);
  if (err.name === 'JsonWebTokenError') error = handleJWTError();
  if (err.name === 'TokenExpiredError') error = handleJWTExpiredError();
  return error;
}

module.exports = { mapKnownErrors };
```

### The native errors you must know

| Source | `err.name` / property | Means | Map to |
| --- | --- | --- | --- |
| Mongoose | `CastError` | Couldn't cast a value (e.g. `/users/abc` where id must be ObjectId) | `400` |
| MongoDB driver | `err.code === 11000` | Unique index violated (duplicate email, etc.) | `400` (or `409 Conflict`) |
| Mongoose | `ValidationError` | A `required`/`min`/`enum` schema rule failed | `400` |
| `jsonwebtoken` | `JsonWebTokenError` | Signature invalid / token malformed | `401` |
| `jsonwebtoken` | `TokenExpiredError` | `exp` claim is in the past | `401` |

> WARNING — Pitfall: A Mongoose `CastError` is *extremely* common — it fires whenever someone hits `/api/users/not-a-valid-id`. Without mapping, your prod handler returns a confusing 500 for what is plainly a client error. Always map `CastError` to 400.

> TIP: Some teams prefer `409 Conflict` over `400` for duplicate-key errors, since the request was well-formed but conflicts with existing state. Either is defensible — pick one and be consistent. The point is: *don't* leak `E11000 duplicate key error collection: app.users index: email_1 dup key`.

With this layer in place, `sendErrorProd` from §8 now reveals helpful, safe messages for these (because they became `isOperational`), while still hiding *genuinely* unexpected errors behind a generic 500.

---

## 10. Input Validation: Why, and the Three Tools

Validation is the *prevention* half of this module: most operational errors are bad input, so rejecting bad input at the door eliminates whole categories of downstream failures. The golden rule of backend development:

> **Never trust the client.** Validate and sanitize *every* piece of input — body, query string, route params, headers — on the server. Client-side validation is a UX nicety; server-side validation is the real security boundary.

What "validation" covers:

- **Presence/required** — is the field there?
- **Type/format** — is `email` actually an email, `age` actually a number, `id` a valid ObjectId?
- **Constraints** — length, range, allowed values (enums), regex patterns.
- **Sanitization** — trimming whitespace, normalizing emails, escaping HTML to prevent XSS, stripping unknown keys (mass-assignment protection).

There are three dominant tools in the Node ecosystem. They overlap but fit different styles:

| Tool | Style | Best fit | Notes |
| --- | --- | --- | --- |
| **express-validator** | Middleware *chains* on `req` | Express apps that want validation co-located with routes | Built *for* Express; validates + sanitizes; uses `validator.js` under the hood |
| **Joi** | Standalone *schema* object | Validating any object (config, messages, requests) outside or inside Express | Mature, hugely featureful, framework-agnostic |
| **Zod** | Standalone *schema* with type inference | Modern apps, esp. where one schema should drive validation *and* types | TS-first but works great in plain JS; composable; great DX |

We will build the same "create user" validation in all three so you can compare them directly.

---

## 11. Validation with `express-validator`

`express-validator` provides **validation chains** — middleware you attach to a route. Each chain targets one field (`body('email')`, `param('id')`, `query('page')`) and you fluently chain validators and sanitizers. After the chains run, `validationResult(req)` collects any failures.

```bash
npm install express-validator
```

```js
// validators/userValidators.js
const { body, param, validationResult } = require('express-validator');
const AppError = require('../utils/AppError');

// An array of validation-chain middlewares for "create user".
const createUserRules = [
  body('email')
    .trim()
    .isEmail().withMessage('A valid email is required')
    .normalizeEmail(),                       // sanitizer: lowercases, etc.
  body('password')
    .isLength({ min: 8 }).withMessage('Password must be at least 8 characters')
    .matches(/\d/).withMessage('Password must contain a number'),
  body('age')
    .optional()                              // only validate if present
    .isInt({ min: 0, max: 120 }).withMessage('Age must be 0–120')
    .toInt(),                                // sanitizer: cast to Number
  body('role')
    .optional()
    .isIn(['user', 'admin']).withMessage('Role must be user or admin'),
];

// One reusable middleware that checks the accumulated results and, if there
// are errors, forwards a single clean AppError to the central handler.
function validate(req, res, next) {
  const result = validationResult(req);
  if (result.isEmpty()) return next(); // no errors — carry on

  // Collect all messages into one operational error.
  const messages = result.array().map((e) => `${e.path}: ${e.msg}`);
  return next(new AppError(messages.join('. '), 400));
}

module.exports = { createUserRules, validate };
```

```js
// Wiring it into a route: rules first, then the validate gate, then handler.
const { createUserRules, validate } = require('../validators/userValidators');
const catchAsync = require('../utils/catchAsync');

router.post(
  '/users',
  createUserRules,               // run all the validation chains
  validate,                      // gate: bail with 400 if any failed
  catchAsync(async (req, res) => {
    const user = await User.create(req.body); // input is now trusted
    res.status(201).json({ status: 'success', data: { user } });
  })
);
```

### How the chain works

Each chain (`body('email')...`) is itself a middleware: when the request arrives, it inspects `req.body.email`, runs validators/sanitizers, and *records* any error onto the request (it does **not** stop the request itself). Your `validate` gate then reads all recorded errors via `validationResult(req)` and decides whether to proceed or reject. This separation — chains record, gate decides — is the key mental model.

> TIP: Sanitizers like `.trim()`, `.normalizeEmail()`, and `.toInt()` *mutate* `req.body`, so by the time your handler runs, the data is cleaned and coerced. Put sanitizers in a sensible order (e.g. `.trim()` before `.isEmail()`).

> BEST PRACTICE: Always end a chain that can fail with `.withMessage(...)` so the client gets a human-readable reason rather than express-validator's terse default.

---

## 12. Schema Validation with Joi

Joi takes a different approach: you describe the *entire* expected object as one **schema**, then validate data against it. It is framework-agnostic — equally good for validating an Express body, a config file, or a message off a queue.

```bash
npm install joi
```

```js
// validators/userSchema.js (Joi)
const Joi = require('joi');

const createUserSchema = Joi.object({
  email: Joi.string().email().required().messages({
    'string.email': 'A valid email is required',
    'any.required': 'Email is required',
  }),
  password: Joi.string().min(8).pattern(/\d/).required().messages({
    'string.min': 'Password must be at least 8 characters',
    'string.pattern.base': 'Password must contain a number',
  }),
  age: Joi.number().integer().min(0).max(120).optional(),
  role: Joi.string().valid('user', 'admin').default('user'),
}); // unknown keys rejected by default -> mass-assignment protection

module.exports = { createUserSchema };
```

### A reusable Joi validation middleware

The real power move is a *generic* middleware factory: pass it a schema and which part of the request to validate, and it returns a ready-to-use middleware.

```js
// middleware/validateJoi.js
const AppError = require('../utils/AppError');

// Factory: returns middleware that validates req[property] against `schema`.
const validateJoi = (schema, property = 'body') => (req, res, next) => {
  const { error, value } = schema.validate(req[property], {
    abortEarly: false,    // collect ALL errors, not just the first
    stripUnknown: true,   // remove keys not in the schema (security)
    convert: true,        // coerce types (e.g. "5" -> 5) where sensible
  });

  if (error) {
    const message = error.details.map((d) => d.message).join('. ');
    return next(new AppError(message, 400));
  }

  req[property] = value;  // use the validated+coerced+stripped value downstream
  next();
};

module.exports = validateJoi;
```

```js
// Usage — one expressive line per route.
const validateJoi = require('../middleware/validateJoi');
const { createUserSchema } = require('../validators/userSchema');

router.post('/users', validateJoi(createUserSchema), catchAsync(async (req, res) => {
  const user = await User.create(req.body); // req.body is validated & stripped
  res.status(201).json({ status: 'success', data: { user } });
}));
```

> BEST PRACTICE: Always pass `abortEarly: false` so clients see *every* problem at once (better UX than fixing one error, resubmitting, finding the next). And `stripUnknown: true` is quiet but important security: it prevents a client from sneaking in extra fields (e.g. `"role": "admin"` on signup) — the classic **mass-assignment** vulnerability.

> TIP: Joi shines for **non-request** validation too — validate `process.env` at startup with a Joi schema so the app refuses to boot with missing/invalid config. Fail fast at startup beats failing mysteriously at runtime.

---

## 13. Schema Validation with Zod

Zod is the modern favorite. Like Joi it is schema-first and framework-agnostic, but its API is exceptionally composable and its real superpower (in TypeScript projects) is **inferring static types from the schema** — one source of truth for both validation and types. In plain JS you still get a clean, ergonomic validator.

```bash
npm install zod
```

```js
// validators/userSchema.zod.js
const { z } = require('zod');

const createUserSchema = z.object({
  email: z.string().email('A valid email is required'),
  password: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .regex(/\d/, 'Password must contain a number'),
  age: z.number().int().min(0).max(120).optional(),
  role: z.enum(['user', 'admin']).default('user'),
}).strict(); // .strict() = reject unknown keys (mass-assignment protection)

module.exports = { createUserSchema };
```

### A reusable Zod validation middleware

```js
// middleware/validateZod.js
const AppError = require('../utils/AppError');

// Factory mirroring the Joi one. Uses safeParse so we never throw here.
const validateZod = (schema, property = 'body') => (req, res, next) => {
  const result = schema.safeParse(req[property]);

  if (!result.success) {
    // result.error.issues is an array of { path, message, ... }
    const message = result.error.issues
      .map((i) => `${i.path.join('.')}: ${i.message}`)
      .join('. ');
    return next(new AppError(message, 400));
  }

  req[property] = result.data; // parsed/coerced/defaulted value
  next();
};

module.exports = validateZod;
```

```js
// Usage — identical ergonomics to the Joi version.
const validateZod = require('../middleware/validateZod');
const { createUserSchema } = require('../validators/userSchema.zod');

router.post('/users', validateZod(createUserSchema), catchAsync(async (req, res) => {
  const user = await User.create(req.body);
  res.status(201).json({ status: 'success', data: { user } });
}));
```

### `parse` vs `safeParse`

- **`schema.parse(data)`** *throws* a `ZodError` on failure. Convenient if you have a global error handler that knows how to format `ZodError` (you could add a `handleZodError` to the §9 translator).
- **`schema.safeParse(data)`** returns `{ success, data | error }` and never throws. Preferred in middleware because you control the response explicitly without relying on throw/catch.

### Where each tool fits — the honest comparison

| Concern | express-validator | Joi | Zod |
| --- | --- | --- | --- |
| Coupling to Express | Tight (by design) | None | None |
| Mental model | Per-field middleware chains | One schema object | One schema object |
| Type inference (TS) | No | Limited | **Excellent** (`z.infer`) |
| Sanitization built in | **Yes** (trim, escape, normalize) | Coercion/transform | `.transform()` / coercion |
| Reuse outside HTTP | Awkward | **Great** | **Great** |
| Ecosystem momentum | Steady | Mature | **Growing fast** |
| Good default choice for… | Existing Express app, want sanitizers | Config + complex object validation | New projects, TS, shared schemas |

> BEST PRACTICE: Don't mix three validators in one codebase. **Pick one** and standardize. For a brand-new plain-JS Express API, Zod or Joi behind a reusable middleware factory gives the cleanest routes; express-validator is excellent when you specifically want its rich built-in sanitizers and like co-locating rules with routes.

---

## 14. Putting It All Together: A Complete Skeleton

Here is how every piece composes into a coherent app. This is the reference architecture worth memorizing.

```js
// server.js — process bootstrap & safety nets
process.on('uncaughtException', (err) => {
  console.error('UNCAUGHT EXCEPTION! Shutting down...');
  console.error(err.name, err.message);
  process.exit(1); // can't drain — state is untrustworthy
});

require('dotenv').config();
const mongoose = require('mongoose');
const app = require('./app');

mongoose.connect(process.env.DB_URI).then(() => console.log('DB connected'));

const server = app.listen(process.env.PORT || 3000, () =>
  console.log(`Listening on ${process.env.PORT || 3000}`)
);

process.on('unhandledRejection', (reason) => {
  console.error('UNHANDLED REJECTION! Shutting down...');
  console.error(reason);
  server.close(() => process.exit(1)); // drain in-flight requests, then exit
});

process.on('SIGTERM', () => {
  console.log('SIGTERM received. Graceful shutdown.');
  server.close(() => console.log('Process terminated.'));
});
```

```js
// app.js — the Express pipeline
const express = require('express');
const AppError = require('./utils/AppError');
const globalErrorHandler = require('./controllers/errorController');
const userRouter = require('./routes/userRoutes');

const app = express();

app.use(express.json({ limit: '10kb' })); // parse JSON, cap body size
app.use('/api/v1/users', userRouter);     // feature routers

// 404 — anything that fell through becomes an operational error.
app.use((req, res, next) => {
  next(new AppError(`Cannot ${req.method} ${req.originalUrl}`, 404));
});

// Central error handler — registered LAST.
app.use(globalErrorHandler);

module.exports = app;
```

The flow for *any* failure: a route throws (or `next(err)`) → `catchAsync` forwards async rejections → 404 handler manufactures errors for unmatched routes → `globalErrorHandler` maps native errors to `AppError`s, then responds verbosely (dev) or safely (prod). One pipeline, every error.

---

## Common Pitfalls & Gotchas

- **Error middleware with three parameters.** `(err, req, res)` is *not* recognized as an error handler — Express needs all four params. Errors silently never reach it.
- **Registering the error handler too early.** It must come *after* all routes; it only catches errors from middleware registered before it.
- **Forgetting `return` before `res` / `next`.** `res.status(400).json(...)` without `return` lets execution continue, causing double responses ("Cannot set headers after they are sent") or stray `next()` calls.
- **Bare `async` handlers on Express v4.** Their rejections become unhandled rejections and never reach your error handler. Use `catchAsync` / `express-async-errors`, or move to v5.
- **Floating promises.** Calling an async function without `await`/`.catch()` — the rejection escapes silently and can crash the process.
- **Swallowing errors.** `catch (err) {}` (empty) or `catch (err) { console.log(err) }` *without* re-throwing or responding hides bugs. Either handle (respond) or re-throw.
- **Leaking internals in prod.** Sending `err.stack` or raw Mongo/SQL errors to clients hands attackers a map. Gate verbose output behind `NODE_ENV`.
- **Trusting `NODE_ENV` defaults.** If unset, your "dev" branch (verbose) may run in prod. Fail closed — default to the safe branch.
- **Mongoose `CastError` → 500.** Without mapping, `/users/not-an-id` returns a confusing 500 instead of a clean 400.
- **Mass assignment.** Not stripping unknown keys lets a client send `"role": "admin"` on signup. Use `stripUnknown`/`.strict()` and/or whitelist fields.
- **`app.get('*', ...)` on Express v5.** The bare `'*'` path throws at startup in v5. Use `app.use(...)` or a named wildcard `/*splat`.
- **Treating process-level handlers as recovery.** Logging in `uncaughtException` and *not* exiting leaves a zombie process. Log, then `process.exit(1)`.
- **`abortEarly: true` by default (Joi).** You only see the first validation error. Set `abortEarly: false` to report all of them.

---

## Best Practices

- **One central error handler.** Exactly one place turns errors into HTTP responses. No `res.status(500)` scattered through controllers.
- **One error class.** Throw `AppError(message, statusCode)` for every expected failure; mark it `isOperational`.
- **One response envelope.** `{ status, message, [data] }` for success *and* error, so frontends branch predictably.
- **Crash on programmer errors; handle operational ones.** Use `isOperational` to decide. Restart via a process manager.
- **Always validate at the boundary.** Validate body, query, params, and headers with a reusable middleware before business logic runs. Never trust the client.
- **Sanitize and strip unknown keys.** Prevent XSS (escape), normalize input, and block mass assignment.
- **Use a `catchAsync` wrapper** (or rely on Express v5's auto-forwarding) so no async handler can leak a rejection.
- **Map native errors** (Mongoose, JWT) to clean operational `AppError`s with correct status codes.
- **Be environment-aware.** Verbose in dev, minimal and leak-free in prod; gate on `NODE_ENV` and fail closed.
- **Log richly, respond minimally.** Full stack + request context to your logs; a clean message to the client.
- **Validate config at startup.** Refuse to boot on missing/invalid env vars (fail fast).
- **Graceful shutdown.** Handle `SIGTERM`/`SIGINT` with `server.close()` so deploys don't drop in-flight requests.

---

## Key Interview Points

- **What's the difference between operational and programmer errors?** Operational errors are expected runtime problems in a correct program (bad input, DB down, 404) — handle them gracefully. Programmer errors are bugs (`undefined is not a function`) that leave the process in an unknown state — log and crash/restart. The distinction dictates your whole strategy.
- **How does an error reach Express's error-handling middleware?** Either you call `next(err)` with a truthy value, or (Express v5) a route handler throws / its promise rejects and Express forwards it automatically. The handler is identified by its **four** parameters `(err, req, res, next)` and must be registered last.
- **Why must error middleware have four parameters?** Express detects error handlers by *arity*. With three params it's treated as normal middleware and never receives errors.
- **Why are unhandled promise rejections dangerous?** They have no handler, so the rejection can crash the process (modern Node terminates by default) and, even when it doesn't, it hides a silent failure (e.g., a save that failed while the client was told "ok").
- **How do you avoid try/catch in every async route?** A `catchAsync` higher-order wrapper (`fn => (req,res,next) => Promise.resolve(fn(...)).catch(next)`), the `express-async-errors` package, or Express v5's native async forwarding.
- **What's the difference between `uncaughtException` and `unhandledRejection`?** The former is a synchronous throw nobody caught; the latter is a rejected promise with no handler. Both are last-resort nets — log and `process.exit(1)`; rely on a process manager to restart.
- **Why crash instead of recovering after an uncaught exception?** The program's invariants may be broken; continuing risks corrupted state and leaks. A clean restart is the only safe option.
- **How do you keep error responses safe in production?** Gate on `NODE_ENV`: send stack traces and full errors only in dev; in prod, reveal messages for `isOperational` errors and a generic 500 message for everything else. Never leak internals.
- **What's `AppError` and why `isOperational`?** A custom `Error` subclass carrying `statusCode`, `status`, and `isOperational`. The flag marks errors you created on purpose so the central handler can reveal their message in prod while hiding unexpected ones.
- **How do you handle a Mongoose `CastError` / duplicate key / `ValidationError`?** Translate them into operational `AppError`s with status 400 *before* responding — `CastError` (bad ObjectId) → 400, code `11000` (duplicate) → 400/409, `ValidationError` → 400 with joined messages.
- **express-validator vs Joi vs Zod — when each?** express-validator: Express-coupled chains with rich built-in sanitizers. Joi: mature, framework-agnostic schema validation, great for config too. Zod: schema-first with excellent TS type inference and composability. Pick one per codebase.
- **What is mass assignment and how do you prevent it?** A client sending unexpected fields (e.g. `role: 'admin'`) that get persisted. Prevent it by stripping unknown keys (`stripUnknown` / `.strict()`) and/or whitelisting fields before writing to the DB.
- **Why validate on the server if the client already validates?** Client validation is UX only and trivially bypassed (curl, Postman, devtools). The server is the real security boundary; never trust the client.

---

## Practice Exercises

Work these in order — each builds on the last. Solutions follow.

1. **`AppError` from scratch.** Implement the `AppError` class with `statusCode`, a derived `status` (`'fail'` for 4xx, `'error'` for 5xx), `isOperational = true`, and a clean stack trace. Write a function `getProduct(id)` that throws `AppError('Product not found', 404)` when a lookup returns null.

2. **`catchAsync`.** Implement `catchAsync(fn)` so that an async route handler's rejection is forwarded to `next`. Demonstrate it on a route that `await`s a function which rejects, and confirm (in words) how the error reaches the central handler.

3. **Central error handler, environment-aware.** Write a 4-arg `globalErrorHandler` that responds verbosely (message + stack) when `NODE_ENV === 'development'` and safely (operational message, else generic 500) otherwise. Default to the *safe* branch if `NODE_ENV` is unset.

4. **404 + wiring.** Wire a minimal Express v5 app: `express.json()`, one `GET /api/items/:id` route (use `catchAsync`), a catch-all 404 that forwards an `AppError`, and the central handler last. Make `/api/items/123` work and `/nope` return your JSON 404.

5. **Reusable validation middleware (Zod *or* Joi).** Build a `validate(schema)` factory that validates `req.body`, collects *all* errors, strips unknown keys, and forwards a single `AppError(400)` on failure. Apply it to a "create user" route requiring `email` (valid email) and `password` (≥8 chars, contains a digit).

6. **Map native errors.** Extend the central handler with a `mapKnownErrors(err)` step that converts a Mongoose `CastError` → 400 and a Mongo duplicate-key (`code === 11000`) → 400, producing clean messages. Show the prod response for each.

### Solutions

**Solution 1 — `AppError`**

```js
// AppError.js
class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.statusCode = statusCode;
    this.status = `${statusCode}`.startsWith('4') ? 'fail' : 'error';
    this.isOperational = true;
    Error.captureStackTrace(this, this.constructor); // omit ctor frame
  }
}

// Simulated lookup that throws a meaningful, operational error.
async function getProduct(id) {
  const product = await db.products.findById(id); // pretend DB call
  if (!product) throw new AppError(`Product ${id} not found`, 404);
  return product;
}

module.exports = { AppError, getProduct };
```

The thrown error is an `Error` (so `try/catch` and `instanceof` work), carries the right HTTP code, and is marked operational so the central handler will reveal its message in production.

**Solution 2 — `catchAsync`**

```js
// catchAsync.js
const catchAsync = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next); // rejection -> next(err)

// Demo route
app.get('/risky', catchAsync(async (req, res) => {
  await Promise.reject(new AppError('It rejected', 502)); // forwarded
  res.json({ ok: true }); // never reached
}));
```

How the error reaches the handler: `fn(...)` returns a rejected promise; `.catch(next)` calls `next(err)`; Express skips remaining normal middleware and routes the error to the 4-arg `globalErrorHandler`. No `try/catch` needed at the call site, and the rejection can never float away.

**Solution 3 — environment-aware central handler**

```js
// errorController.js
module.exports = (err, req, res, next) => {
  err.statusCode = err.statusCode || 500;
  err.status = err.status || 'error';

  // Fail closed: only verbose when EXPLICITLY in development.
  const isDev = process.env.NODE_ENV === 'development';

  if (isDev) {
    return res.status(err.statusCode).json({
      status: err.status,
      message: err.message,
      error: err,
      stack: err.stack,
    });
  }

  // Production branch (also the default when NODE_ENV is unset).
  if (err.isOperational) {
    return res.status(err.statusCode).json({
      status: err.status,
      message: err.message,
    });
  }
  console.error('UNEXPECTED ERROR 💥', err); // log for us
  return res.status(500).json({ status: 'error', message: 'Something went wrong' });
};
```

Because the dev branch requires `NODE_ENV === 'development'` *exactly*, any other value (including unset) falls through to the safe prod branch — failing closed.

**Solution 4 — minimal Express v5 app**

```js
// app.js
const express = require('express');
const { AppError } = require('./AppError');
const catchAsync = require('./catchAsync');
const globalErrorHandler = require('./errorController');

const app = express();
app.use(express.json());

const items = { '123': { id: '123', name: 'Widget' } };

app.get('/api/items/:id', catchAsync(async (req, res) => {
  const item = items[req.params.id];
  if (!item) throw new AppError(`No item with id ${req.params.id}`, 404);
  res.json({ status: 'success', data: { item } });
}));

// Catch-all 404 (v5-safe form). Anything unmatched lands here.
app.use((req, res, next) => {
  next(new AppError(`Cannot ${req.method} ${req.originalUrl}`, 404));
});

app.use(globalErrorHandler); // LAST
module.exports = app;
```

`GET /api/items/123` → `200 { status: 'success', data: { item } }`. `GET /nope` → the 404 middleware forwards an `AppError(404)`, the central handler formats it: `404 { status: 'fail', message: 'Cannot GET /nope' }`. Same shape as every other error.

**Solution 5 — reusable validation middleware (Zod)**

```js
// validate.js
const { z } = require('zod');
const { AppError } = require('./AppError');

const validate = (schema, property = 'body') => (req, res, next) => {
  const result = schema.safeParse(req[property]);
  if (!result.success) {
    const message = result.error.issues
      .map((i) => `${i.path.join('.')}: ${i.message}`)
      .join('. ');
    return next(new AppError(message, 400)); // all errors, one response
  }
  req[property] = result.data; // stripped + coerced
  next();
};

const createUserSchema = z.object({
  email: z.string().email('A valid email is required'),
  password: z.string().min(8, 'Min 8 chars').regex(/\d/, 'Needs a digit'),
}).strict(); // reject unknown keys

// Route
app.post('/api/users', validate(createUserSchema), catchAsync(async (req, res) => {
  const user = await User.create(req.body); // trusted, stripped input
  res.status(201).json({ status: 'success', data: { user } });
}));
```

A request missing the password and sending `role: 'admin'` would fail twice: the missing/short password *and* `.strict()` rejecting the unknown `role` key — both reported in one `400` message.

**Solution 6 — mapping native errors**

```js
// In errorController.js, before the prod responder:
const { AppError } = require('./AppError');

function mapKnownErrors(err) {
  let error = err;
  if (err.name === 'CastError') {
    error = new AppError(`Invalid ${err.path}: ${err.value}`, 400);
  }
  if (err.code === 11000) {
    const field = Object.keys(err.keyValue)[0];
    error = new AppError(`Duplicate ${field}: "${err.keyValue[field]}"`, 400);
  }
  return error;
}

// ...inside the prod branch:
//   const error = mapKnownErrors(err);
//   if (error.isOperational) { res.status(error.statusCode).json({...}) }
```

Prod responses:
- `GET /users/not-an-id` → Mongoose throws `CastError` → mapped → `400 { status: 'fail', message: 'Invalid _id: not-an-id' }`.
- `POST /users` with a taken email → Mongo `11000` → mapped → `400 { status: 'fail', message: 'Duplicate email: "taken@x.com"' }`.

Both are now operational (so their messages are safe to send), instead of confusing 500s that leak driver internals.

---

## Summary / Cheat Sheet

| Concept | Key takeaway |
| --- | --- |
| **Operational vs programmer error** | Expected (handle gracefully) vs bug (log + crash + restart). Drives everything. |
| **`try/catch` + `async/await`** | `await` turns rejection into a throw `try/catch` can see. Only catches the awaited line. |
| **Unhandled rejection** | Floating promise with no handler → can crash process / hides silent failures. Always await/return/`.catch`. |
| **Error middleware** | 4 params `(err, req, res, next)`; detected by arity; registered **last**. |
| **Reaching the handler** | `next(err)`, or (Express v5) a thrown/rejected handler auto-forwards. |
| **`AppError`** | `extends Error` + `statusCode`, `status`, `isOperational`. Throw for every expected failure. |
| **`catchAsync`** | `fn => (req,res,next) => Promise.resolve(fn(...)).catch(next)`. Kills repetitive try/catch. |
| **`express-async-errors`** | One-import global patch; mostly redundant on v5. |
| **Process nets** | `uncaughtException` → exit now; `unhandledRejection` → drain + exit. Restart via pm2/k8s. |
| **404 handler** | Catch-all `app.use(...)` after routes → forward `AppError(404)`. Use `app.use`, not `'*'`, on v5. |
| **Env-aware responses** | Dev: message + stack. Prod: operational message, else generic 500. Fail closed on `NODE_ENV`. |
| **Native error mapping** | `CastError`→400, `11000`→400/409, `ValidationError`→400, `JsonWebTokenError`/`TokenExpiredError`→401. |
| **express-validator** | `body()/param()/query()` chains + `validationResult(req)`; great sanitizers; Express-coupled. |
| **Joi** | Schema object; framework-agnostic; `abortEarly:false` + `stripUnknown:true`. Great for config. |
| **Zod** | Schema-first; `safeParse`; `.strict()`; best TS type inference; modern DX. |
| **Reusable validation** | `validate(schema, property)` factory → 1-line route gate forwarding `AppError(400)`. |
| **Golden rule** | Never trust the client. Validate + sanitize every input on the server. |

**The architecture in one breath:** validate input at the door → throw `AppError` for expected failures → `catchAsync` forwards async rejections → 404 handler manufactures errors for unmatched routes → one central 4-arg handler maps native errors and responds (verbose in dev, safe in prod) → process-level nets catch the rest and crash cleanly for a manager to restart. One pipeline, every error, every time.
