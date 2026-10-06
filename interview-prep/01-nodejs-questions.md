# Node.js Interview Questions & Answers

A curated bank of interview questions covering Node.js core internals: the runtime architecture, the event loop, modules, streams, the `process` object, error handling, concurrency primitives, and the npm ecosystem. The questions are grouped by difficulty and written to reflect what interviewers actually ask — and what separates a "knows the buzzwords" answer from a "genuinely understands the runtime" answer.

## How to use this bank

- **First pass (learning):** Read each question, attempt your own answer out loud, then compare with the model answer. The goal is not memorization but understanding *why* each answer is true.
- **Second pass (drilling):** Cover the answers and quiz yourself. If you cannot explain a concept from first principles, revisit the relevant course module.
- **Last-minute revision:** Jump straight to the [Rapid-Fire Quick Answers](#rapid-fire-quick-answers) table at the bottom.
- **Output-prediction questions:** These are marked and are interviewer favorites. Trace the code by hand before reading the answer — getting the *reasoning* right matters more than the final line.

> TIP: In a real interview, a strong candidate does not just answer the literal question — they briefly explain the underlying mechanism and call out a relevant pitfall or trade-off. The model answers below are written in that spirit. Trim them to ~60 seconds when speaking; the written depth is for your understanding.

---

## Beginner

### 1. **What is Node.js, and how is it different from JavaScript running in a browser?**

Node.js is a **runtime** that lets you execute JavaScript outside the browser, primarily on servers. It embeds Google's **V8** engine (the same engine that powers Chrome) to compile and run JS, and adds a set of C/C++ libraries — most importantly **libuv** — to provide capabilities the browser deliberately withholds: filesystem access, raw TCP/UDP sockets, child processes, OS-level information, and more.

The key differences from browser JavaScript:

| Aspect | Browser JS | Node.js |
| --- | --- | --- |
| Global object | `window` / `self` | `global` (or `globalThis` in both) |
| DOM / `document` | Available | Not available (no DOM) |
| APIs | `fetch`, `localStorage`, `alert` | `fs`, `http`, `process`, `Buffer`, `crypto` |
| Module system | ES Modules (`<script type="module">`) | CommonJS by default, ESM supported |
| Security model | Sandboxed per-origin | Full access to the host machine |

The conceptual point an interviewer wants: Node.js is **not a language and not a framework** — it is a runtime/host environment for the JavaScript language, analogous to how the browser is also a host environment.

> Follow-up: "So is Node.js single-threaded?" (see Q4) and "What is V8's job vs libuv's job?" (see Q12).

---

### 2. **Why is Node.js a good fit for I/O-bound applications but a poor fit for CPU-bound ones?**

Node.js uses a **single-threaded, event-driven, non-blocking I/O** model. When your code initiates an I/O operation (reading a file, querying a database, making a network call), Node hands that work off to the operating system or to libuv's thread pool and **immediately continues executing other JavaScript** instead of blocking. When the I/O completes, a callback is queued and run later. This means a single Node process can juggle thousands of concurrent connections that are mostly *waiting*, with very little memory per connection.

For **CPU-bound** work (image processing, cryptographic hashing of huge inputs, big JSON parsing, complex computation), the story flips. That work runs on the **single main thread** that also runs the event loop. While it churns, no other JavaScript — including incoming requests — can be processed. One expensive synchronous computation blocks the entire server.

```js
// I/O-bound: scales beautifully. Thousands of these can be in flight at once.
const data = await db.query("SELECT * FROM users WHERE id = $1", [id]);

// CPU-bound: this blocks the event loop. Every other request waits.
function fib(n) {
  return n < 2 ? n : fib(n - 1) + fib(n - 2);
}
const result = fib(45); // synchronous, ~seconds of pure CPU — server is frozen
```

> BEST PRACTICE: Offload CPU-bound work to `worker_threads`, a `child_process`, a job queue, or a separate service so the event loop stays responsive. (See Q22.)

> Follow-up: "How would you fix a CPU-bound endpoint without rewriting in another language?"

---

### 3. **What is the difference between blocking and non-blocking code in Node.js?**

**Blocking** code halts execution of the JavaScript thread until the operation completes — nothing else runs in the meantime. **Non-blocking** code initiates the operation and returns control immediately, delivering the result later via a callback, promise, or event.

In Node.js the cleanest illustration is the `fs` module, which exposes both styles:

```js
const fs = require("node:fs");

// Blocking (synchronous): the thread stops here until the file is fully read.
const dataSync = fs.readFileSync("./big.txt", "utf8");
console.log("1: have the data");
console.log("2: this runs only after the read finishes");

// Non-blocking (asynchronous): returns immediately; callback runs later.
fs.readFile("./big.txt", "utf8", (err, data) => {
  if (err) throw err;
  console.log("B: have the data");
});
console.log("A: this runs BEFORE the file is read");
// Output order: "A" then "B"
```

The critical interview nuance: "non-blocking" does **not** mean "multithreaded in your JS." Your JavaScript still runs on one thread. Non-blocking means the *waiting* is delegated, so the one thread is never stuck idling.

> WARNING — Pitfall: Synchronous (`...Sync`) methods are fine in CLI scripts and at startup, but using them inside a request handler will block *every* concurrent request. Avoid `fs.readFileSync`, `crypto.pbkdf2Sync`, etc., on the hot path of a server.

---

### 4. **Is Node.js single-threaded or multi-threaded? Explain carefully.**

This is a classic trap. The honest answer is: **"It depends on what you mean."**

- **Your JavaScript runs on a single thread** — the *main thread*, which hosts the event loop. There is only ever one piece of your JS executing at a time. This is what people mean by "Node is single-threaded."
- **Under the hood, Node is multi-threaded.** libuv maintains a **thread pool** (default size 4) used for certain operations: filesystem I/O, DNS lookups (`dns.lookup`), and CPU-bound crypto/zlib functions. Network I/O does *not* use this pool — it uses the OS's async mechanisms (epoll/kqueue/IOCP). The V8 engine also spawns threads for garbage collection and background compilation.
- Additionally, **you can opt into more threads** explicitly via the `worker_threads` module, or more processes via `cluster` / `child_process`.

So a precise answer: *"Node executes application JavaScript on a single main thread, but uses a libuv thread pool and OS-level async I/O internally, and offers `worker_threads`/`cluster` for explicit parallelism."*

> Follow-up: "Which operations use the libuv thread pool?" (see Q23) and "How big is the thread pool and how do you change it?" (`UV_THREADPOOL_SIZE`, max 1024).

---

### 5. **What is the difference between CommonJS and ES Modules in Node.js?**

These are the two module systems Node supports.

| Feature | CommonJS (CJS) | ES Modules (ESM) |
| --- | --- | --- |
| Import syntax | `const x = require("x")` | `import x from "x"` |
| Export syntax | `module.exports = ...` / `exports.foo = ...` | `export default ...` / `export const foo` |
| Loading | Synchronous | Asynchronous (supports top-level `await`) |
| Resolution timing | Runtime (dynamic, can be conditional) | Static (parsed before execution) |
| Bindings | Value **copy** at time of require | **Live bindings** (reference to the export) |
| File extension | `.js` (default), `.cjs` | `.mjs`, or `.js` with `"type": "module"` |
| `__dirname` / `__filename` | Available | Not available; use `import.meta.url` / `import.meta.dirname` |
| `this` at top level | `module.exports` | `undefined` |

```js
// CommonJS
const fs = require("node:fs");
module.exports = { greet: () => "hi" };

// ES Modules
import fs from "node:fs";
export const greet = () => "hi";

// Reconstructing __dirname in ESM:
import { fileURLToPath } from "node:url";
import path from "node:path";
const __dirname = path.dirname(fileURLToPath(import.meta.url));
// Node 20.11+ also exposes import.meta.dirname directly.
```

How Node decides which loader to use: a file is treated as ESM if it has a `.mjs` extension, or if the nearest `package.json` has `"type": "module"`. It is CJS if it is `.cjs`, or `.js` with `"type": "commonjs"` (or no `type` field). You can use `import()` (dynamic import) from CJS to load ESM, and as of recent Node versions you can `require()` a synchronous ESM module too, but historically `require` of ESM threw `ERR_REQUIRE_ESM`.

> Follow-up: "What does 'live bindings' actually mean?" — In ESM, if a module re-assigns an exported variable, importers see the new value; with `require` you captured a snapshot of the value at require time.

---

### 6. **What is `module.exports` vs `exports`, and why can reassigning `exports` break your module?** (output prediction)

`exports` starts out as a **reference to the same object** as `module.exports`. When you `require()` a module, Node returns whatever `module.exports` points to — it does **not** look at the local `exports` variable. Adding properties to `exports` works because you are mutating the shared object. But **reassigning** `exports` just repoints the local variable and severs the link.

```js
// utils.js — WORKS: mutating the shared object
exports.add = (a, b) => a + b;        // module.exports.add is set too
module.exports.sub = (a, b) => a - b; // also fine

// other.js — BROKEN: reassigning the local variable only
exports = { multiply: (a, b) => a * b }; // module.exports is untouched!
```

```js
const utils = require("./utils");
console.log(utils.add(2, 3)); // 5  ✅
console.log(utils.sub(5, 1)); // 4  ✅

const other = require("./other");
console.log(other.multiply); // undefined ❌ — module.exports was still {}
```

The rule: to export a **single value** (a class, a function, one object), assign to `module.exports`. To export **multiple named things**, attach them to `exports` (or to `module.exports`). Never reassign `exports` itself.

> BEST PRACTICE: Pick one style per file. If in doubt, always use `module.exports` — it is never wrong.

---

### 7. **What are some important Node.js global objects/variables that are not in the browser?**

Node provides several "globals" — some are truly global, some are module-scoped variables that *feel* global.

- **`global`** — the global object (like `window`). Prefer `globalThis` for portability.
- **`process`** — information about and control over the current Node process (env vars, argv, exit, streams). See Q8.
- **`Buffer`** — fixed-length raw binary data. See Q17.
- **`console`** — writes to `stdout`/`stderr`.
- **Timers** — `setTimeout`, `setInterval`, `setImmediate`, `queueMicrotask`.
- **`__dirname` / `__filename`** — directory and full path of the current module (CommonJS only; module-scoped, not truly global).
- **`require` / `module` / `exports`** — the CommonJS module machinery (also module-scoped).
- **`structuredClone`, `fetch`, `URL`, `TextEncoder`** — Web-standard APIs now available globally in modern Node.

```js
console.log(process.version);        // e.g. v22.x
console.log(process.platform);       // 'darwin' | 'linux' | 'win32'
console.log(__dirname);              // absolute dir of this file (CJS)
console.log(globalThis === global);  // true in Node
```

> WARNING — Pitfall: Adding properties to `global` to share state across modules is an anti-pattern — it creates hidden coupling and is hard to test. Export values from modules instead.

---

### 8. **What is the `process` object and what are some of its most useful properties and events?**

`process` is a global object that represents the running Node.js process and is your bridge to the operating system and runtime.

Commonly used members:

- **`process.argv`** — array of command-line arguments (`[node, scriptPath, ...args]`).
- **`process.env`** — environment variables (`process.env.NODE_ENV`, `process.env.PORT`).
- **`process.cwd()`** — current working directory.
- **`process.platform` / `process.arch`** — OS and CPU architecture.
- **`process.pid`** — process id.
- **`process.exit([code])`** — terminate the process (use sparingly; it can cut off pending I/O).
- **`process.nextTick(cb)`** — schedule a callback before the next event-loop phase (see Q14).
- **`process.stdin` / `process.stdout` / `process.stderr`** — standard streams.
- **`process.memoryUsage()` / `process.uptime()`** — diagnostics.

Important events:

- **`exit`** — fired when the process is about to exit; only synchronous work runs here.
- **`uncaughtException`** — a thrown error reached the top of the stack. (See Q21.)
- **`unhandledRejection`** — a rejected promise had no `.catch`.
- **`SIGINT` / `SIGTERM`** — OS signals, used for graceful shutdown.

```js
// Graceful shutdown skeleton
process.on("SIGTERM", async () => {
  console.log("Shutting down...");
  await server.close();     // stop accepting new connections
  await db.disconnect();    // close DB pool
  process.exit(0);
});
```

> Follow-up: "How do you read configuration safely from `process.env`?" — Validate at startup, never trust types (everything is a string), and never log secrets.

---

### 9. **What is the `EventEmitter` and how do you use it?**

`EventEmitter` (from the `node:events` module) is the foundation of Node's event-driven architecture. Many core objects — HTTP servers, streams, the `process` object — are emitters. It implements the **publish/subscribe (observer)** pattern: you register listeners with `.on(eventName, listener)` and trigger them with `.emit(eventName, ...args)`.

```js
const EventEmitter = require("node:events");

class OrderService extends EventEmitter {
  placeOrder(order) {
    // ... persist order ...
    this.emit("order:placed", order); // synchronously calls listeners
  }
}

const orders = new OrderService();

orders.on("order:placed", (order) => {
  console.log(`Sending confirmation email for ${order.id}`);
});
orders.once("order:placed", (order) => {
  console.log("This runs only for the FIRST order");
});

orders.placeOrder({ id: 42 });
```

Key behaviors and gotchas:

- **Listeners are called synchronously**, in registration order, when `emit` is called.
- **`.once()`** auto-removes the listener after one invocation.
- A special **`'error'` event**: if you `emit('error', ...)` and there is no listener, Node **throws** and can crash the process. Always attach an error listener to emitters that can fail.
- By default an emitter warns at **more than 10 listeners** for one event (a leak heuristic); raise it with `emitter.setMaxListeners(n)` only if intentional.
- Remove listeners with `.off()` / `.removeListener()` to prevent memory leaks in long-lived emitters.

> Follow-up: "Is `emit` sync or async?" — Synchronous. To defer, wrap the emit in `setImmediate` or `process.nextTick`.

---

### 10. **What is the difference between `setTimeout(fn, 0)`, `setImmediate(fn)`, and `process.nextTick(fn)` at a high level?**

All three defer work, but they run at different times relative to the event loop:

- **`process.nextTick(fn)`** — runs **before the event loop continues**, after the current operation completes. It is processed *between* every phase, ahead of promises. The highest-priority deferral.
- **`Promise.then` / `queueMicrotask`** — microtasks, run right after `nextTick` callbacks, also between phases.
- **`setTimeout(fn, 0)`** — runs in the **timers** phase on a *future* loop iteration (minimum ~1ms in practice).
- **`setImmediate(fn)`** — runs in the **check** phase, designed to fire right after the **poll** phase of the current iteration.

```js
console.log("start");
setTimeout(() => console.log("timeout"), 0);
setImmediate(() => console.log("immediate"));
process.nextTick(() => console.log("nextTick"));
Promise.resolve().then(() => console.log("promise"));
console.log("end");

// Output:
// start
// end
// nextTick      <- microtask-priority queue, before promises
// promise       <- microtasks
// timeout / immediate  <- order between these two is non-deterministic here
```

The relative order of `timeout` vs `immediate` in the top-level (main module) context is **not guaranteed** because it depends on how fast the loop reaches the timers phase. See Q15 for the case where the order *is* deterministic.

> Follow-up: "When is `setImmediate` guaranteed to run before `setTimeout(fn,0)`?" (see Q15).

---

### 11. **What does `require()` actually do? Walk through what happens when you call it.**

`require()` is the CommonJS loader. When you call `require("./foo")`, Node performs roughly these steps:

1. **Resolve** the specifier to an absolute file path using the module resolution algorithm (see Q13). This may add extensions (`.js`, `.json`, `.node`), look for `index.js`, or consult `package.json` `main`/`exports`.
2. **Check the cache** (`require.cache`, keyed by resolved absolute path). If the module was already loaded, return the cached `module.exports` immediately — the module body does **not** run again.
3. **Create a `module` object** (`{ id, exports: {}, loaded: false, ... }`) and insert it into the cache *before* executing (this is what enables circular dependencies to partially work — see Q19).
4. **Wrap and execute** the module's source code inside a function wrapper that injects the local variables:

   ```js
   (function (exports, require, module, __filename, __dirname) {
     // ...your module code runs here...
   });
   ```

5. **Return `module.exports`** — whatever the module assigned to it.

This wrapping is why `exports`, `require`, `module`, `__filename`, and `__dirname` appear "magically" available in every CommonJS file: they are function parameters, not true globals.

> Follow-up: "What's in `require.cache` and when would you clear it?" — It is an object keyed by file path; deleting an entry forces a re-require (used in some hot-reload setups, but fragile in production).

---

## Intermediate

### 12. **Describe the high-level architecture of Node.js. What are V8 and libuv responsible for?**

Node.js is a layered system. From top to bottom:

1. **Your JavaScript** — application and npm packages.
2. **Node.js core API** (`fs`, `http`, `stream`, ...) — partly JS, partly C++ bindings.
3. **V8** — Google's JavaScript engine. It parses, compiles (JIT), and executes JS, and manages the JS heap and garbage collection. V8 knows *nothing* about files, sockets, or timers — it is purely a language engine.
4. **libuv** — a C library that provides the **event loop**, the **thread pool**, and cross-platform asynchronous I/O (filesystem, networking, timers, signals). This is what gives Node its non-blocking superpowers and its platform abstraction (epoll on Linux, kqueue on macOS, IOCP on Windows).
5. **Other C/C++ libraries** — OpenSSL (crypto/TLS), c-ares (async DNS), zlib (compression), llhttp (HTTP parsing).

The interplay: V8 runs your JS; when your JS calls `fs.readFile`, the Node C++ binding hands the request to libuv; libuv performs the async work (via thread pool or OS), and when done, places your callback on a queue; the event loop (also libuv) eventually invokes it back in V8.

```
┌──────────────────────────────────────────┐
│        Your JS + npm packages             │
├──────────────────────────────────────────┤
│   Node core API (fs, http, stream...)     │
├───────────────┬──────────────────────────┤
│      V8       │   libuv (event loop,      │
│  (JS engine)  │   thread pool, async I/O) │
├───────────────┴──────────────────────────┤
│  OpenSSL · c-ares · zlib · llhttp · OS    │
└──────────────────────────────────────────┘
```

> Follow-up: "Where does the event loop live — in V8 or libuv?" — libuv. This is a common gotcha; V8 has a microtask queue, but the macro event loop is libuv's.

---

### 13. **How does Node.js resolve a module specifier in `require()`?**

Node distinguishes three kinds of specifiers and resolves each differently:

1. **Core modules** — e.g. `require("fs")` or `require("node:fs")`. These short-circuit immediately; nothing is read from disk. The `node:` prefix makes it explicit and unshadowable.
2. **Relative / absolute paths** — `require("./util")`, `require("../lib/db")`, `require("/abs/path")`. Node tries, in order:
   - the exact file,
   - the file with `.js`, `.json`, `.node` appended,
   - if it is a directory: the `main` field (or `exports`) of its `package.json`, else `index.js` / `index.json` / `index.node`.
3. **Bare specifiers / packages** — `require("express")`. Node walks **`node_modules`** directories upward from the current file's location: `./node_modules`, then `../node_modules`, then `../../node_modules`, and so on up to the filesystem root. The first match wins. Inside a package, the entry point is determined by `exports` (preferred, modern) or `main` (legacy), defaulting to `index.js`.

```js
require("node:path");        // core module, no disk lookup
require("./helpers");        // ./helpers.js -> ./helpers/index.js -> ...
require("lodash");           // ./node_modules/lodash -> walk up if missing
```

> Follow-up: "What does the `exports` field in package.json do vs `main`?" — `exports` defines an *encapsulated* public API and can map subpaths and conditions (`import` vs `require`, `node` vs `browser`); it blocks deep imports into internal files, unlike `main`.

---

### 14. **Explain the Node.js event loop and its phases in order.**

The event loop is libuv's mechanism for executing callbacks. Each full iteration ("tick") proceeds through several **phases**, each with its own FIFO callback queue:

1. **Timers** — executes callbacks scheduled by `setTimeout` and `setInterval` whose threshold has elapsed.
2. **Pending callbacks** — executes certain system-level callbacks deferred from the previous loop (e.g. some TCP errors).
3. **Idle / prepare** — internal use only.
4. **Poll** — the heart of the loop: retrieves new I/O events and executes their callbacks (incoming connections, data, completed file reads). If there are no timers due, the loop may **block here waiting for I/O**. This is where most of your callbacks run.
5. **Check** — executes `setImmediate` callbacks.
6. **Close callbacks** — executes `'close'` event callbacks (e.g. `socket.on('close')`).

**Crucially**, between each phase (and after each individual callback), Node drains two higher-priority queues:

- the **`process.nextTick` queue**, then
- the **microtask queue** (resolved Promises / `queueMicrotask`).

These are *not* phases of the loop — they run *between* and *within* phases, which is why they always "jump ahead" of timers and `setImmediate`.

```
   ┌───────────────────────────┐
┌─>│           timers          │   setTimeout / setInterval
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │   (internal)
│  └─────────────┬─────────────┘   ┌───────────────┐
│  ┌─────────────┴─────────────┐   │   incoming:   │
│  │           poll            │<──┤  connections, │
│  └─────────────┬─────────────┘   │   data, etc.  │
│  ┌─────────────┴─────────────┐   └───────────────┘
│  │           check           │   setImmediate
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │   socket.on('close')
   └───────────────────────────┘

   (nextTick + microtask queues drain between EVERY step)
```

> Follow-up: "What happens if a `nextTick` callback recursively schedules another `nextTick`?" — You **starve the event loop**: the nextTick queue is drained completely before the loop advances, so an infinite chain prevents I/O from ever running. (See Q14b/Q16.)

---

### 15. **(Output prediction) Why does `setImmediate` vs `setTimeout(0)` ordering become deterministic inside an I/O callback?**

At the top level (main module), the order of `setTimeout(fn, 0)` and `setImmediate(fn)` is **non-deterministic** — it depends on how quickly the process reaches the timers phase versus the loop starting. But inside an **I/O callback** (which runs in the *poll* phase), the order is **guaranteed**: `setImmediate` always fires first.

```js
const fs = require("node:fs");

// Top level: order is NOT guaranteed
setTimeout(() => console.log("timeout"), 0);
setImmediate(() => console.log("immediate"));

// Inside an I/O callback: order IS guaranteed
fs.readFile(__filename, () => {
  setTimeout(() => console.log("io -> timeout"), 0);
  setImmediate(() => console.log("io -> immediate"));
});

// Guaranteed portion of output:
// io -> immediate
// io -> timeout
```

The reason: the I/O callback runs in the **poll** phase. The very next phase is **check** (where `setImmediate` lives), so the immediate callback runs almost instantly. The timer callback must wait for the loop to come back around to the **timers** phase on the *next* iteration. Hence `setImmediate` wins reliably from within any poll-phase (I/O) callback.

> Follow-up: "So when should I use `setImmediate` vs `setTimeout(0)`?" — Use `setImmediate` to yield back to the loop after I/O ("run this right after the current poll work"); it expresses intent better and avoids the minimum-1ms timer clamp.

---

### 16. **What is `process.nextTick` exactly, when should you use it, and what is the danger?**

`process.nextTick(cb)` queues `cb` to run **immediately after the currently executing operation completes, before the event loop proceeds to its next phase** — and before any Promise microtask. It is the highest-priority deferral mechanism in Node.

Legitimate uses:

- **Making an API consistently asynchronous.** If a function sometimes returns synchronously and sometimes asynchronously, callers face subtle bugs. Defer the synchronous path with `nextTick` so the callback *always* fires after the current stack unwinds.
- **Allowing the caller to attach listeners before an event fires.** e.g., emit an error on `nextTick` so a freshly created emitter's `.on('error')` is wired up first.

```js
function readConfig(useCache, cb) {
  if (useCache && cache) {
    // Don't call cb synchronously — keep behavior consistent.
    return process.nextTick(cb, null, cache);
  }
  fs.readFile("config.json", cb); // already async
}
```

The danger — **event loop starvation**:

```js
// ❌ This NEVER lets I/O or timers run. The nextTick queue is fully
//    drained each time before the loop advances, and it never empties.
function loop() {
  process.nextTick(loop);
}
loop();
```

Because the entire nextTick queue is processed before the loop continues, an ever-growing or recursive nextTick chain prevents I/O callbacks from ever executing. `setImmediate` does **not** have this problem because immediates are processed one phase per loop iteration, leaving room for I/O.

> WARNING — Pitfall: Prefer `setImmediate` for "run soon, but don't starve the loop." Reserve `process.nextTick` for the narrow consistency/listener-timing cases above. The names are historically backwards — `nextTick` fires *before* `setImmediate`.

---

### 17. **What is a Buffer and how does it relate to streams and binary data?**

A **`Buffer`** is Node's representation of a fixed-length sequence of bytes — raw binary data outside the V8 heap. It predates JavaScript's `Uint8Array`/`TypedArray`; today `Buffer` is a subclass of `Uint8Array` with extra convenience methods. Buffers are essential because the network and filesystem deal in bytes, not strings, and not every byte sequence is valid text in a given encoding.

```js
// Create
const a = Buffer.from("héllo", "utf8");      // from a string
const b = Buffer.alloc(8);                    // 8 zero-filled bytes (safe)
const c = Buffer.allocUnsafe(8);              // faster, but contains old memory!

console.log(a.length);          // 6 (é is two bytes in UTF-8, not 5!)
console.log(a.toString("hex")); // 68c3a96c6c6f
console.log(a.toString("base64"));

// Concatenate chunks (e.g. from a stream)
const whole = Buffer.concat([Buffer.from("foo"), Buffer.from("bar")]);
console.log(whole.toString()); // "foobar"
```

Key points for interviews:

- **`Buffer.alloc(n)`** zero-fills (safe). **`Buffer.allocUnsafe(n)`** is faster but may expose **leftover memory** — always overwrite it fully before use, or you risk leaking sensitive data.
- A `Buffer`'s byte length can differ from a string's character length because of multi-byte encodings (UTF-8).
- Streams emit `Buffer` chunks by default (unless you set an encoding or use object mode). Reassembling text from a stream requires care: a multi-byte character may be **split across two chunks**, so naively calling `.toString()` per chunk can corrupt it. Use `StringDecoder` or set the stream's encoding.

> WARNING — Pitfall: `chunk.toString()` per chunk can break multi-byte UTF-8 across chunk boundaries. Use `new (require('string_decoder').StringDecoder)('utf8')` or accumulate Buffers and decode once.

---

### 18. **What are the four types of streams in Node.js, and why use streams at all?**

Streams process data **piece by piece** instead of loading everything into memory at once. This keeps memory usage flat and constant regardless of data size, lets you start processing before all data arrives, and composes naturally via piping.

The four types:

| Type | Description | Example |
| --- | --- | --- |
| **Readable** | Source you read from | `fs.createReadStream`, HTTP request (server), `process.stdin` |
| **Writable** | Destination you write to | `fs.createWriteStream`, HTTP response, `process.stdout` |
| **Duplex** | Both readable and writable, independent | TCP socket (`net.Socket`) |
| **Transform** | Duplex where output is computed from input | `zlib.createGzip`, `crypto.createCipheriv` |

Contrast with reading a whole file: `fs.readFile("10gb.log")` tries to hold 10 GB in memory and will crash; a stream reads it in ~64 KB chunks.

```js
const fs = require("node:fs");
const zlib = require("node:zlib");

// Read -> gzip (transform) -> write, all chunked, constant memory:
fs.createReadStream("big.log")
  .pipe(zlib.createGzip())            // Transform stream
  .pipe(fs.createWriteStream("big.log.gz"));
```

Streams operate in two modes: **flowing** (data is pushed via `'data'` events) and **paused** (you `.read()` explicitly). Piping puts the readable into flowing mode and handles the plumbing for you.

> Follow-up: "What's the difference between `pipe` and `pipeline`?" (see Q26) and "What is backpressure?" (see Q25).

---

### 19. **(Output prediction) How do circular dependencies behave in CommonJS?**

Circular dependencies do not crash in CommonJS, but they can yield **partially populated** exports. The reason: Node inserts a module into the cache *before* its body finishes executing. If module A requires B, and B requires A while A is still mid-execution, B receives A's exports **as they exist at that moment** — possibly incomplete.

```js
// a.js
console.log("a: start");
exports.loaded = false;
const b = require("./b");          // triggers b.js
console.log("a: b.value =", b.value);
exports.loaded = true;
console.log("a: done");

// b.js
console.log("b: start");
const a = require("./a");          // A is still running -> partial exports
console.log("b: a.loaded =", a.loaded); // sees the snapshot so far: false
exports.value = 42;
console.log("b: done");

// main.js
require("./a");
```

Output:

```
a: start
b: start
b: a.loaded = false   // <-- a.js hadn't set loaded = true yet
b: done
a: b.value = 42
a: done
```

The takeaway: B saw `a.loaded === false` because A had not reached that line yet. ESM handles cycles more gracefully thanks to **live bindings** (the binding exists even before initialization, and accessing it before assignment can throw a `ReferenceError` due to the temporal dead zone, which surfaces bugs earlier).

> BEST PRACTICE: Treat circular dependencies as a design smell. Refactor shared logic into a third module, move `require` calls to the point of use (lazy require), or restructure to break the cycle.

---

### 20. **What is the module cache, and how can it surprise you?**

When a module is `require()`d, Node caches its `module.exports` keyed by the **resolved absolute path** in `require.cache`. Subsequent `require()`s of the same path return the cached object **without re-running the module body**. This makes modules effectively **singletons** within a process.

Consequences and surprises:

```js
// counter.js
let count = 0;
module.exports = {
  inc: () => ++count,
  get: () => count,
};
```

```js
const a = require("./counter");
const b = require("./counter");
a.inc();
console.log(b.get()); // 1  <- same instance! a and b are identical objects
console.log(a === b); // true
```

Edge cases that defeat the cache (and create *separate* instances unexpectedly):

- The **same package installed at two different paths** (e.g., different versions nested in `node_modules`) resolves to two different absolute paths → two cached copies → two singletons. This causes classic "instanceof fails across versions" bugs.
- On case-insensitive filesystems, casing differences can or cannot collide depending on platform.
- **CJS and ESM caches are separate**, so requiring vs importing the same file can yield distinct instances.

> WARNING — Pitfall: Deleting from `require.cache` to "reload" a module in production is fragile — old references already captured by other modules will not update. Use it only for dev tooling.

---

### 21. **How do you handle errors in Node.js across callbacks, promises, async/await, and EventEmitters?**

Error handling differs by async style, and a strong answer covers all four plus the global safety net.

**1. Error-first callbacks** — the Node convention: the first callback argument is the error (or `null`).

```js
fs.readFile("x.txt", (err, data) => {
  if (err) return handle(err); // always check err first
  use(data);
});
```

**2. Promises** — attach `.catch`, or errors propagate down the chain.

```js
doThing().then(use).catch(handle);
```

**3. async/await** — use `try/catch`. An unhandled rejection here is the most common production bug source.

```js
async function run() {
  try {
    const data = await doThing();
    use(data);
  } catch (err) {
    handle(err);
  }
}
```

**4. EventEmitters / streams** — listen for the `'error'` event. An emitted `'error'` with no listener **crashes the process**.

```js
stream.on("error", handle); // mandatory for any stream/emitter that can fail
```

**5. Global safety nets** — last resort, for logging and graceful shutdown, not for resuming:

```js
process.on("uncaughtException", (err) => { log(err); process.exit(1); });
process.on("unhandledRejection", (reason) => { throw reason; }); // crash loudly
```

> BEST PRACTICE: After an `uncaughtException`, the process is in an undefined state — log, flush, and exit. Do not try to keep running. Let a process manager (PM2, systemd, Kubernetes) restart it. Also: throw `Error` objects, never strings, so you keep a stack trace.

> WARNING — Pitfall: Forgetting `return` after `if (err)` in a callback — the success path then runs anyway with bad data. And never mix `await` with a forgotten `.catch` on a fire-and-forget promise.

---

### 22. **Compare `child_process`, `worker_threads`, and `cluster`. When would you use each?**

All three add concurrency beyond the single main thread, but they differ in isolation, memory model, and use case.

| | `child_process` | `worker_threads` | `cluster` |
| --- | --- | --- | --- |
| Unit | Separate OS process | Thread in same process | Multiple processes (forks) |
| Memory | Fully isolated | Shared heap possible via `SharedArrayBuffer`; otherwise isolated | Fully isolated |
| Communication | IPC (serialized messages), stdio | `MessagePort` (structured clone), shared memory | IPC |
| Startup cost | Highest | Lower than process | Per-process |
| Best for | Running other programs / scripts; isolation | CPU-bound JS computation | Scaling a server across CPU cores |

- **`child_process`** (`spawn`, `exec`, `execFile`, `fork`): run external commands (e.g. `ffmpeg`) or fork another Node script. `spawn` streams output (good for large/long output); `exec` buffers it (convenient but risky for big output); `fork` is a specialized spawn for Node scripts with an IPC channel.

  ```js
  const { spawn } = require("node:child_process");
  const p = spawn("ls", ["-lh", "/usr"]);
  p.stdout.on("data", (d) => console.log(`${d}`));
  ```

- **`worker_threads`**: run CPU-bound JavaScript in parallel without spawning whole processes. Lighter than child processes and can share memory. The right tool for offloading heavy computation while keeping the event loop responsive.

  ```js
  const { Worker } = require("node:worker_threads");
  const worker = new Worker("./heavy-calc.js", { workerData: { n: 45 } });
  worker.on("message", (result) => console.log("got", result));
  ```

- **`cluster`**: fork the *same* server across CPU cores; the master shares a listening socket so the OS load-balances connections. This scales **I/O throughput** across cores. (In production many teams instead run multiple processes behind a load balancer / PM2, which is conceptually similar.)

The decision rule: **CPU-bound, same logic → `worker_threads`**; **scale a network server across cores → `cluster`**; **run/another program or need hard process isolation → `child_process`**.

> Follow-up: "Why not just always use cluster?" — Cluster gives you N copies of your app (more I/O concurrency) but does not speed up a single CPU-heavy request; worker_threads parallelizes the computation itself.

---

### 23. **Which operations use the libuv thread pool, and how do you change its size?**

Most people assume "all async I/O uses the thread pool," which is **wrong**. Network I/O (TCP/UDP/HTTP) uses the OS's native async facilities (epoll/kqueue/IOCP) and does **not** consume thread-pool slots. The thread pool (default size **4**) is used for operations that have no portable async OS primitive:

- **Filesystem operations** (`fs.*`, except a few)
- **DNS resolution via `dns.lookup`** (which uses `getaddrinfo` under the hood) — but **not** `dns.resolve*`, which uses async c-ares
- **CPU-bound crypto** like `crypto.pbkdf2`, `crypto.scrypt`, `crypto.randomBytes` (async forms)
- **`zlib`** compression/decompression (async forms)

Because the pool defaults to 4 threads, issuing more than 4 of these operations at once queues the rest — a hidden source of latency.

```bash
# Increase the libuv thread pool (must be set before the process starts).
# Max is 1024. Set based on cores and workload.
UV_THREADPOOL_SIZE=8 node server.js
```

```js
// Inside Node you can also set it before any async work touches the pool:
process.env.UV_THREADPOOL_SIZE = "8"; // only effective if set early enough
```

> WARNING — Pitfall: A burst of `crypto.pbkdf2` (e.g. password hashing on login) plus heavy `fs` access can saturate the 4 default threads, serializing requests. Profile and tune `UV_THREADPOOL_SIZE` for crypto/fs-heavy services.

> Follow-up: "Does `dns.lookup` differ from `dns.resolve`?" — Yes: `lookup` uses the thread pool (getaddrinfo, respects /etc/hosts); `resolve` queries DNS servers via async c-ares and does not use the pool.

---

## Advanced

### 24. **(Output prediction) Predict the exact output and explain why.**

```js
console.log("1");

setTimeout(() => console.log("2"), 0);

Promise.resolve().then(() => {
  console.log("3");
  process.nextTick(() => console.log("4"));
});

process.nextTick(() => {
  console.log("5");
  Promise.resolve().then(() => console.log("6"));
});

setImmediate(() => console.log("7"));

console.log("8");
```

**Output:**

```
1
8
5
3
6
4
2   (2 and 7 order: 2 then 7 OR 7 then 2 is environment-dependent at top level;
7    typically 7 then 2 here — see note)
```

Reasoning, step by step:

1. **Synchronous code first:** `1`, then `8` print immediately.
2. The current operation (the main module) finishes. Node now drains the **`nextTick` queue before microtasks**. The only nextTick queued is the one printing `5`. So `5` prints. Inside it, a Promise `.then` (→ `6`) is queued as a microtask.
3. Node drains the **microtask queue**. At this point it contains: the original `.then` (→ `3`), and the just-added `.then` (→ `6`). The `3` callback runs first → prints `3`, and schedules a **new nextTick** (→ `4`). Then `6` runs → prints `6`.
4. After microtasks, before moving to the next phase, Node again checks the **nextTick queue**, which now has `4` → prints `4`. (Microtasks drain again after — empty.)
5. Now the event loop advances to phases: **timers** (`2`) and **check** (`7`). At the top level the relative order of a 0ms timer and a `setImmediate` is **not guaranteed**, though `setImmediate` (`7`) commonly precedes the timer (`2`).

The core lessons: **nextTick beats Promises**, **microtasks fully drain between phases**, and **callbacks scheduled during draining are processed in the same draining window** (which is how `4` runs before the loop advances).

> Follow-up: "If `5`'s callback scheduled another `nextTick`, when would it run?" — Still before the event loop advances, because the entire nextTick queue (including newly added entries) drains first.

---

### 25. **What is backpressure in streams, and how do you handle it correctly?**

**Backpressure** is what happens when a fast producer (readable) overwhelms a slow consumer (writable) — for example, reading from a fast disk and writing to a slow network socket. Without handling it, data buffers in memory unbounded until the process runs out of memory.

The mechanism: `writable.write(chunk)` returns a **boolean**. `false` means the internal buffer has exceeded its `highWaterMark` and you should **stop writing** until the writable emits a `'drain'` event. Ignoring this return value is the classic backpressure bug.

```js
// ❌ WRONG: ignores write()'s return value -> unbounded memory growth
readable.on("data", (chunk) => {
  writable.write(chunk); // may return false; we never pause -> backpressure ignored
});

// ✅ Manual backpressure handling
readable.on("data", (chunk) => {
  const ok = writable.write(chunk);
  if (!ok) {
    readable.pause();                       // stop reading
    writable.once("drain", () => readable.resume()); // resume when buffer clears
  }
});
```

The good news: **`pipe()` and `pipeline()` handle backpressure for you automatically.** That is the single biggest reason to prefer them over manual `data`/`write` loops.

```js
// ✅ Backpressure handled automatically:
readable.pipe(writable);
```

> BEST PRACTICE: Never hand-roll a `data` → `write` loop unless you have a specific reason. Use `pipe`/`pipeline`, which respect `highWaterMark` and `drain` correctly.

> Follow-up: "What is `highWaterMark`?" — The buffer-size threshold (default ~64 KB for byte streams, 16 objects for object mode) at which `write()` starts returning `false` and a readable stops reading ahead.

---

### 26. **What is the difference between `stream.pipe()` and `stream.pipeline()`? Why does it matter?**

Both connect streams together, but they differ critically in **error handling and cleanup**.

**`pipe()`** wires a readable into a writable and handles data flow + backpressure — but it does **not** forward errors or clean up the whole chain on failure. If a stream in the middle errors, the others are **not** automatically destroyed, leading to **leaked file descriptors / hanging sockets**. You must attach `'error'` handlers to *every* stream yourself.

```js
// ❌ Fragile: an error in gzip won't destroy src/dest -> potential leak
src.pipe(gzip).pipe(dest);
src.on("error", cleanup);   // you must add these to EVERY stream
gzip.on("error", cleanup);  // easy to forget one
dest.on("error", cleanup);
```

**`pipeline()`** (from `node:stream`, also `stream/promises`) pipes a series of streams, **propagates errors**, **destroys all streams** in the chain on any failure, and gives you a single completion callback/promise. It is the modern, correct way.

```js
const { pipeline } = require("node:stream/promises");
const fs = require("node:fs");
const zlib = require("node:zlib");

async function gzipFile() {
  try {
    await pipeline(
      fs.createReadStream("input.log"),
      zlib.createGzip(),
      fs.createWriteStream("input.log.gz")
    );
    console.log("Done — all streams cleaned up on success or error.");
  } catch (err) {
    console.error("Pipeline failed:", err); // any stream's error lands here
  }
}
```

| | `pipe()` | `pipeline()` |
| --- | --- | --- |
| Backpressure | Handled | Handled |
| Error propagation | No (per-stream `'error'` needed) | Yes (single error path) |
| Cleanup on failure | No (manual `destroy`) | Yes (auto-destroys all) |
| Completion signal | None built-in | Callback or Promise |

> BEST PRACTICE: Use `pipeline` (the promise form) for any multi-stream chain in production. Reserve bare `pipe` for trivial, short-lived, single-hop cases.

---

### 27. **How would you detect and diagnose event-loop blocking / a slow Node service?**

A blocked event loop manifests as rising latency, timeouts, and unresponsiveness even though CPU may look idle-ish or pegged on one core. Diagnosis approach:

1. **Measure event-loop lag / utilization.**
   - `perf_hooks.monitorEventLoopDelay()` gives a histogram of loop delay.
   - `perf_hooks.performance.eventLoopUtilization()` (ELU) reports the fraction of time the loop was busy vs idle — excellent for spotting CPU saturation.

   ```js
   const { monitorEventLoopDelay } = require("node:perf_hooks");
   const h = monitorEventLoopDelay({ resolution: 20 });
   h.enable();
   setInterval(() => {
     console.log("loop delay p99 (ms):", (h.percentile(99) / 1e6).toFixed(2));
   }, 5000);
   ```

2. **Profile CPU.** Use `node --prof` (V8 tick processor) or `--cpu-prof` to produce a `.cpuprofile`, or attach Chrome DevTools via `node --inspect`. Flame graphs reveal which synchronous function is hogging the loop.

3. **Check the usual culprits:** large synchronous `JSON.parse`/`stringify`, synchronous `fs.*Sync`, unbounded loops, regular-expression catastrophic backtracking (ReDoS), synchronous crypto, and saturating the libuv thread pool (Q23).

4. **Fix:** move CPU work to `worker_threads`, chunk large work across `setImmediate`, stream large payloads instead of buffering, add caching, and tune `UV_THREADPOOL_SIZE`.

> TIP: A quick gut check in dev — schedule `setInterval` for every 100ms and log if the actual gap is much larger; the drift equals event-loop lag.

> Follow-up: "What is event loop utilization vs lag?" — Lag measures how late timers fire (delay); ELU measures the *proportion* of time the loop was actively executing JS vs waiting — a high ELU near 1.0 means the loop is CPU-bound.

---

### 28. **Explain `npm`, `package.json`, the difference between `dependencies` and `devDependencies`, and `package-lock.json`.**

**npm** is Node's default package manager and registry client. **`package.json`** is the manifest describing your project: its name, version, entry point, scripts, and dependency declarations.

Dependency categories:

| Field | Installed in production? | Examples |
| --- | --- | --- |
| `dependencies` | Yes (`npm install`) | `express`, `mongoose` — needed at runtime |
| `devDependencies` | No with `--omit=dev` / `NODE_ENV=production` | `jest`, `eslint`, `nodemon` — build/test only |
| `peerDependencies` | Not auto-installed; expected to be provided by host | a plugin requiring a specific `react` version |
| `optionalDependencies` | Installed if possible; failure is non-fatal | platform-specific native addons |

**`package-lock.json`** records the **exact resolved version, integrity hash, and full dependency tree** that was installed. It ensures **reproducible installs** across machines and CI — without it, semver ranges could resolve to different versions over time. Commit it.

- `npm install` may update the lockfile to satisfy `package.json` ranges.
- **`npm ci`** installs **strictly from the lockfile** (deleting `node_modules` first), failing if `package.json` and the lock are out of sync. Use `npm ci` in CI/CD for deterministic builds.

```jsonc
{
  "name": "my-api",
  "version": "1.2.0",
  "type": "commonjs",
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js",
    "dev": "nodemon src/index.js",
    "test": "jest"
  },
  "dependencies":    { "express": "^4.19.2" },
  "devDependencies": { "nodemon": "^3.1.0" }
}
```

> Follow-up: "Difference between `npm install` and `npm ci`?" — `ci` is faster, deletes `node_modules`, installs exactly from the lockfile, and never writes to `package.json`; `install` resolves ranges and may mutate the lockfile.

---

### 29. **Explain semantic versioning (semver) and the meaning of `^`, `~`, and exact pins.**

**Semantic Versioning** uses `MAJOR.MINOR.PATCH` (e.g. `4.19.2`):

- **MAJOR** — incompatible/breaking API changes.
- **MINOR** — new functionality, **backward-compatible**.
- **PATCH** — backward-compatible bug fixes.

npm version **ranges** in `package.json` control which updates an `npm install` will accept:

| Range | Meaning | `4.19.2` allows up to (exclusive) |
| --- | --- | --- |
| `^4.19.2` | Compatible: allow MINOR + PATCH, lock MAJOR | `< 5.0.0` |
| `~4.19.2` | Approximately: allow PATCH only | `< 4.20.0` |
| `4.19.2` | Exact pin | only `4.19.2` |
| `*` / `latest` | Anything | (dangerous) |
| `>=4.0 <5.0` | Explicit range | as written |

```bash
npm install express          # adds "^4.x.y" by default (caret)
npm install express@4.18.2   # specific version
npm install express --save-exact  # pins exactly, no caret
```

A special rule worth knowing: for **`0.x.y`** versions, `^` behaves like `~` — `^0.3.1` allows only `>=0.3.1 <0.4.0`, because pre-1.0 minor bumps are treated as potentially breaking.

> WARNING — Pitfall: Caret ranges mean two developers running `npm install` weeks apart can get different transitive versions. The **`package-lock.json`** is what actually guarantees everyone gets the same tree — semver ranges only define what is *allowed*.

> Follow-up: "How do you safely update dependencies?" — `npm outdated` to see drift, `npm update` for in-range bumps, and tools/audits (`npm audit`) plus tests before bumping a MAJOR.

---

### 30. **What happens when an HTTP request hits an Express server — trace the journey through the event loop.**

This synthesizes the architecture into a concrete flow.

1. **Connection accepted (poll phase).** The OS notifies libuv that a socket has incoming data; the event loop, in the **poll** phase, runs the HTTP server's connection/data callback. Network I/O here is OS-async, not thread-pool based.
2. **HTTP parsing.** Node's `llhttp` parser turns the raw bytes (a `Buffer`) into a request line, headers, and body. The `http.IncomingMessage` is a **Readable stream**; the `http.ServerResponse` is a **Writable stream**.
3. **Express dispatch.** Express runs its **middleware chain** synchronously per `next()` call: body parsing, auth, routing, your handler. Each middleware runs as JavaScript on the main thread.
4. **Your handler does async work.** A DB query (`await db.query(...)`) is dispatched to the driver, which uses a socket (OS-async). Your handler **yields**; the event loop is free to serve other requests in the meantime.
5. **Result returns.** When the DB responds, its callback/promise resolution is queued (microtask after the I/O callback), your handler resumes, and you call `res.json(...)`, which **writes to the response stream** (with backpressure handled).
6. **Response flushed and socket closed/kept alive.** `'finish'`/`'close'` events fire in the close-callbacks phase.

The crucial insight: between steps 4 and 5, **the single thread is not blocked** — it processes other requests. This is exactly why one Node process serves thousands of concurrent connections for I/O-bound workloads, and exactly why a synchronous CPU-heavy handler (step 3/4 done wrong) would freeze *all* of them.

> Follow-up: "Where would a synchronous `bcrypt.hashSync` in step 4 hurt you?" — It blocks the main thread, stalling every concurrent request; use the async form, which offloads to the libuv thread pool (mind the pool size, Q23).

---

### 31. **What is the difference between `exec`, `execFile`, `spawn`, and `fork` in `child_process`?** (bonus)

All create child processes but differ in how they invoke programs and handle output.

| Method | Shell? | Output handling | Typical use |
| --- | --- | --- | --- |
| `spawn` | No (by default) | **Streams** stdout/stderr | Long-running / large output (`ffmpeg`, `git log`) |
| `exec` | Yes (runs in a shell) | **Buffers** all output, then callback | Short commands where you want the full string |
| `execFile` | No shell | Buffers output | Run an executable directly (safer than `exec`) |
| `fork` | No | IPC channel + stdio | Spawn another **Node** script with messaging |

```js
const { spawn, exec, fork } = require("node:child_process");

// spawn: stream output, no shell -> safer, scales to big output
const ls = spawn("ls", ["-lh"]);
ls.stdout.on("data", (d) => process.stdout.write(d));

// exec: convenient but buffers (maxBuffer) and uses a shell -> injection risk
exec("git rev-parse HEAD", (err, stdout) => console.log(stdout.trim()));

// fork: child is a Node module; communicate via messages
const child = fork("./worker.js");
child.send({ task: "process" });
child.on("message", (msg) => console.log("from child:", msg));
```

> WARNING — Pitfall: `exec` (and `spawn` with `{ shell: true }`) interpolate through a shell, enabling **command injection** if you concatenate user input. Prefer `spawn`/`execFile` with an args array and never build command strings from untrusted input.

---

## Rapid-Fire Quick Answers

| Question | One-line answer |
| --- | --- |
| Is Node single-threaded? | Your JS runs on one main thread; libuv uses a thread pool + OS async I/O underneath. |
| What runs JS in Node? | The V8 engine. |
| What provides the event loop and async I/O? | libuv. |
| `nextTick` vs `Promise.then`? | `nextTick` callbacks run first, before the microtask (promise) queue. |
| `setImmediate` vs `setTimeout(0)`? | Inside an I/O callback, `setImmediate` always runs first; at top level it's non-deterministic. |
| Default libuv thread pool size? | 4 (set via `UV_THREADPOOL_SIZE`, max 1024). |
| Does network I/O use the thread pool? | No — it uses OS async (epoll/kqueue/IOCP); fs/dns.lookup/crypto/zlib use the pool. |
| CommonJS vs ESM import? | `require()` (sync, value copy) vs `import` (async, static, live bindings). |
| `module.exports` vs `exports`? | They reference the same object; reassign only `module.exports`, never `exports`. |
| Are modules cached? | Yes — keyed by resolved absolute path; modules act as singletons per process. |
| Four stream types? | Readable, Writable, Duplex, Transform. |
| What is backpressure? | A slow writable signaling (via `write()` returning false) to pause a fast readable. |
| `pipe` vs `pipeline`? | `pipeline` propagates errors and auto-destroys all streams on failure; `pipe` does not. |
| `Buffer.alloc` vs `allocUnsafe`? | `alloc` zero-fills (safe); `allocUnsafe` is faster but may expose old memory. |
| What crashes on an emitter? | Emitting `'error'` with no listener throws and can crash the process. |
| `npm install` vs `npm ci`? | `ci` installs strictly from the lockfile (deletes node_modules), ideal for CI. |
| What does `^1.2.3` allow? | Minor + patch updates, locked to major 1 (`>=1.2.3 <2.0.0`). |
| worker_threads vs cluster? | worker_threads = parallel CPU-bound JS; cluster = scale a server across cores. |
| Where do you handle async errors? | `try/catch` for await, `.catch` for promises, `err`-first callbacks, `'error'` events. |
