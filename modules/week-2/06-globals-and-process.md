# Node Globals & the process Object

When you write JavaScript in a browser, an invisible host environment surrounds your code. You never declared `window`, `document`, `fetch`, or `localStorage`, yet there they are — the browser injected them. Node.js does exactly the same thing, but for a *server* environment. Instead of a DOM and a window, Node gives you a runtime built for talking to the operating system: reading files, listening on sockets, spawning child processes, and inspecting the running process itself.

This module is about that injected layer. We'll cover the **global scope** in Node (and how it differs from the browser), the everyday **globals** you'll reach for constantly (`console`, `__dirname`, `Buffer`, the timer family, and the module-system identifiers), and then the crown jewel of server-side Node: the **`process` object**. The `process` object is your program's window into its own existence — its arguments, its environment, its memory, its lifecycle, and the OS signals that can end it. Mastering `process` is the difference between writing scripts that "work on my machine" and writing services that start cleanly, read configuration correctly, and shut down gracefully under load.

## What You'll Learn

- How Node's **global object** works, why `globalThis` exists, and how the global scope differs from the browser's `window`.
- The most important **built-in globals**: `console`, `__dirname`, `__filename`, `require`, `module`, `exports`, `Buffer`, the timer functions (`setTimeout`, `setInterval`, `setImmediate`), and `queueMicrotask`.
- The **`process` object** in depth: `argv`, `env`, `cwd()`, `pid`, `platform`, `arch`, `version`/`versions`, `memoryUsage()`, `uptime()`, and `hrtime()`.
- How to **end a process correctly** with `process.exit()`, exit codes, and the safer `process.exitCode`.
- The **process lifecycle events** — `exit`, `beforeExit`, `uncaughtException`, `unhandledRejection` — and the OS signals `SIGINT` and `SIGTERM`.
- The **graceful shutdown pattern** that every production service needs.
- How to manage **environment variables** and an introduction to **`dotenv`**.
- Working with the **standard streams** (`process.stdin`, `stdout`, `stderr`), reading command-line arguments, and building a simple argument parser.

---

## 1. The Global Object: Node vs. the Browser

### The concept and the WHY

Every JavaScript runtime has a single top-level object that holds the values available everywhere without an import. In the browser that object is `window` (and its aliases `self` and `frames`). In Node.js, that object is literally named `global`.

```js
// In Node — `global` is the top-level object.
console.log(typeof global); // "object"
global.myAppName = "shop-api"; // attach something to it
console.log(myAppName); // "shop-api" — readable without the `global.` prefix
```

Why does Node call it `global` instead of `window`? Because `window` is a browser concept — it represents a browser window/tab with a DOM, a screen, a location bar, and a render loop. Node has none of those. There is no DOM, no `document`, no `localStorage`, no `alert`. Naming the object `window` would be a lie about what's available. Node names it `global` to describe what it *is*: the container for globally available bindings.

### The key difference: top-level `var` does NOT become global in Node

This trips up nearly every developer coming from the browser. In a browser script, `var x = 1` at the top level creates `window.x`. **In Node, it does not.**

```js
// browser-script.js (loaded via <script>)
var fruit = "apple";
console.log(window.fruit); // "apple" — top-level var leaks onto window
```

```js
// node-script.js (run with `node node-script.js`)
var fruit = "apple";
console.log(global.fruit); // undefined — NOT attached to global!
```

The reason is fundamental to how Node loads files. **Every Node module (file) is wrapped in a function before execution.** Node literally takes your file's source and wraps it like this:

```js
// What Node does internally to every CommonJS file:
(function (exports, require, module, __filename, __dirname) {
  // ... your file's code runs here ...
  var fruit = "apple"; // this `var` is a LOCAL variable of the wrapper function
});
```

Because your code runs inside a function, top-level `var`, `let`, `const`, and `function` declarations are *function-scoped locals*, not properties of `global`. This is a feature, not a bug: it means two modules can both declare `var config` without clobbering each other. The browser's habit of leaking globals was always a hazard; Node avoids it by design.

> TIP: This wrapper is also why `__dirname`, `__filename`, `require`, `module`, and `exports` "just exist" in every CommonJS file — they're the parameters of that wrapper function, not true globals. We'll return to this.

### `globalThis`: the universal name

For years, code that needed to run in both browsers and Node had to sniff the environment: `typeof window !== "undefined" ? window : global`. ES2020 fixed this with **`globalThis`** — a standardized name for the global object that works *everywhere*: browsers, Node, Deno, web workers, and Bun.

```js
// Works identically in Node, browsers, workers, Deno, Bun:
globalThis.sharedFlag = true;
console.log(globalThis.sharedFlag); // true

// In Node, `globalThis` and `global` are the same object:
console.log(globalThis === global); // true (in Node)
```

> BEST PRACTICE: Prefer `globalThis` over `global` in any code that might be shared, copied between environments, or bundled for both server and client. Reserve `global` for Node-only scripts where you want to be explicit that this is server code.

### Comparison: browser global vs. Node global

| Aspect | Browser | Node.js |
| --- | --- | --- |
| Global object name | `window` (also `self`, `frames`) | `global` |
| Standard universal name | `globalThis` | `globalThis` |
| Top-level `var x = 1` | Becomes `window.x` | Does **not** attach to `global` |
| DOM (`document`, `location`) | Yes | No |
| `fetch` | Yes (always) | Yes (global since Node 18) |
| `alert`, `localStorage` | Yes | No |
| `process` | No | Yes |
| `require`, `module`, `__dirname` | No | Yes (CommonJS) |
| `Buffer` | No (use `Uint8Array`/`Blob`) | Yes |

> WARNING — Pitfall: Polluting `global` is almost always a mistake. It defeats the module system's isolation, makes code untestable, and creates hidden coupling ("spooky action at a distance"). If you find yourself reaching for `global.something = ...` to share state, you almost certainly want to `module.exports` a value and `require` it instead.

---

## 2. Common Globals You'll Use Every Day

Node injects a set of globals that are always available without `require`. Let's go through the ones that matter most.

### 2.1 `console`

`console` is the logging interface. It's global, and it writes to the standard streams: `console.log`/`console.info` go to **stdout**, while `console.error`/`console.warn` go to **stderr**. This stdout/stderr split matters enormously in production — it lets you pipe diagnostics separately from program output.

```js
console.log("normal output");          // -> stdout
console.error("something went wrong");  // -> stderr
console.warn("deprecation notice");     // -> stderr

// Useful, lesser-known methods:
console.table([{ id: 1, name: "Ana" }, { id: 2, name: "Bo" }]); // pretty grid
console.dir({ a: { b: { c: 1 } } }, { depth: null });           // full nesting
console.time("load");                  // start a named timer
// ... do work ...
console.timeEnd("load");               // prints "load: 12.34ms"
console.assert(1 + 1 === 3, "math broke"); // logs only if assertion fails
console.count("hit");                  // "hit: 1", "hit: 2", ...
```

> TIP: `console.log` formats objects with `util.inspect`, which truncates deep objects at depth 2 by default. When you see `[Object]` or `[Array]` in your logs, switch to `console.dir(obj, { depth: null })` or `JSON.stringify(obj, null, 2)` to see everything.

> WARNING — Pitfall: `console.log` is *synchronous* when writing to a file or pipe, and can be *asynchronous* when writing to a TTY. Heavy logging inside a hot loop can become a real performance bottleneck because it blocks the event loop. For production logging, use a dedicated async logger (Pino, Winston) rather than thousands of `console.log` calls.

### 2.2 `__dirname` and `__filename`

These two give you the **absolute path** of the current module — `__filename` is the full path to the current file, and `__dirname` is the directory containing it.

```js
// Suppose this file is /home/app/src/util/paths.js
console.log(__filename); // /home/app/src/util/paths.js
console.log(__dirname);  // /home/app/src/util
```

The WHY: file paths in Node are resolved relative to the **current working directory** (`process.cwd()`), *not* relative to the file doing the work. If you write `fs.readFile("./config.json")`, Node looks for `config.json` in whatever directory the user launched `node` from — which is unpredictable. `__dirname` lets you build paths relative to *the file*, which is what you almost always want.

```js
const path = require("node:path");
const fs = require("node:fs");

// CORRECT: resolve relative to THIS file, regardless of where node was launched.
const configPath = path.join(__dirname, "config.json");
const data = fs.readFileSync(configPath, "utf8");

// FRAGILE: depends on the user's current working directory.
// const data = fs.readFileSync("./config.json", "utf8"); // may fail
```

> WARNING — Pitfall (ES Modules): `__dirname` and `__filename` **do not exist in ES Modules** (`.mjs` files or `"type": "module"` packages). In ESM you reconstruct them from `import.meta.url`:
>
> ```js
> // ESM equivalent of __dirname / __filename
> import { fileURLToPath } from "node:url";
> import path from "node:path";
>
> const __filename = fileURLToPath(import.meta.url);
> const __dirname = path.dirname(__filename);
> ```
>
> As of recent Node (v20.11+/v21.2+), ESM also exposes `import.meta.dirname` and `import.meta.filename` directly, which is cleaner:
>
> ```js
> console.log(import.meta.dirname);  // directory of the current ES module
> console.log(import.meta.filename); // full path of the current ES module
> ```

### 2.3 `require`, `module`, `exports`

These are the heart of the **CommonJS module system**. As noted in Section 1, they are not truly global — they're parameters of the per-file wrapper function — but they're available in every CommonJS module, so they behave like globals.

- **`require(id)`** — loads and returns another module's exports. Resolution: core modules (`node:fs`), then relative/absolute paths, then `node_modules`.
- **`module`** — the object representing the current module. `module.exports` is what `require` returns to callers.
- **`exports`** — a convenience reference that *starts out* pointing at the same object as `module.exports`.

```js
// math.js
function add(a, b) {
  return a + b;
}
function sub(a, b) {
  return a - b;
}

// Attach to the SAME object that `exports` and `module.exports` both reference:
exports.add = add;
exports.sub = sub;
// Equivalent: module.exports.add = add; module.exports.sub = sub;
```

```js
// app.js
const math = require("./math"); // ".js" extension is optional
console.log(math.add(2, 3));    // 5
console.log(math.sub(5, 1));    // 4
```

> WARNING — Pitfall: The single most common CommonJS mistake is **reassigning `exports`**:
>
> ```js
> // BROKEN: this rebinds the local `exports` variable; module.exports is untouched.
> exports = function () { /* ... */ };
>
> // CORRECT: to export a single value (function, class, object), assign module.exports.
> module.exports = function () { /* ... */ };
> ```
>
> The rule: use `exports.foo = ...` to add named properties, but use `module.exports = ...` when you want to replace the entire export with one value. Once you assign `module.exports = ...`, the `exports` alias is no longer useful in that file.

> TIP: `require` caches modules. The first `require("./math")` executes the file; every subsequent `require("./math")` returns the *same cached* exports object without re-running the file. This is why a module's top-level code runs exactly once — it's a natural singleton.

### 2.4 `Buffer`

A `Buffer` is a fixed-length chunk of raw binary data — Node's answer to "how do I represent bytes?" It predates the standardized `Uint8Array` and remains essential for file I/O, network sockets, cryptography, and any place where you handle bytes rather than text. Under the hood a modern `Buffer` *is* a `Uint8Array` with extra convenience methods.

```js
// Create a Buffer from a string (default encoding is UTF-8):
const buf = Buffer.from("héllo");
console.log(buf);            // <Buffer 68 c3 a9 6c 6c 6f>
console.log(buf.length);     // 6 bytes (note: "é" is 2 bytes in UTF-8, not 1!)
console.log("héllo".length); // 5 characters — bytes != characters

// Convert back to a string:
console.log(buf.toString("utf8"));   // "héllo"
console.log(buf.toString("hex"));    // "68c3a96c6c6f"
console.log(buf.toString("base64")); // "aMOpbGxv"

// Allocate a zero-filled buffer of a given size:
const zeros = Buffer.alloc(4);       // <Buffer 00 00 00 00> (safe, zeroed)
```

> WARNING — Pitfall: Never use the deprecated `new Buffer(size)` or `Buffer.allocUnsafe(size)` without immediately overwriting every byte. `Buffer.allocUnsafe` is faster because it skips zeroing, but it can hand you memory containing leftover data — potentially secrets from earlier in the process. Use `Buffer.alloc(size)` (zero-filled) unless you have measured a real need for `allocUnsafe` *and* you fully overwrite it.

> TIP: A character is not a byte. `"€".length` is 1 in JavaScript (one UTF-16 code unit) but `Buffer.from("€").length` is 3 (three UTF-8 bytes). When you slice or size data based on `string.length`, you can corrupt multi-byte characters. Size by bytes when dealing with binary boundaries.

### 2.5 Timers: `setTimeout`, `setInterval`, `setImmediate`, and `clearXxx`

Node provides the browser timer API plus a Node-specific one (`setImmediate`). These are global.

- **`setTimeout(fn, delay, ...args)`** — run `fn` once after at least `delay` ms.
- **`setInterval(fn, delay, ...args)`** — run `fn` repeatedly every `delay` ms.
- **`setImmediate(fn, ...args)`** — run `fn` on the next iteration of the event loop, after the current poll phase completes (Node-only).
- **`clearTimeout` / `clearInterval` / `clearImmediate`** — cancel a pending timer using the handle returned by the corresponding setter.

```js
const t = setTimeout(() => console.log("ran after 100ms"), 100);
clearTimeout(t); // cancels it before it fires

const id = setInterval(() => console.log("tick"), 1000);
setTimeout(() => clearInterval(id), 3500); // stop after ~3 ticks

setImmediate(() => console.log("runs ASAP, after current phase"));
```

**Ordering matters and is a classic interview question.** The general rule: microtasks (Promises, `queueMicrotask`) run before the next macrotask; within the macrotask world, `process.nextTick` callbacks run before Promise microtasks, and `setImmediate` vs `setTimeout(fn, 0)` ordering depends on context.

```js
setTimeout(() => console.log("1: setTimeout"), 0);
setImmediate(() => console.log("2: setImmediate"));
Promise.resolve().then(() => console.log("3: promise microtask"));
process.nextTick(() => console.log("4: nextTick"));
console.log("5: synchronous");

// Typical output:
// 5: synchronous       <- runs now
// 4: nextTick          <- nextTick queue drains first
// 3: promise microtask <- then the microtask queue
// 1: setTimeout        <- macrotasks (order of 1 vs 2 can vary at top level)
// 2: setImmediate
```

> WARNING — Pitfall: `setTimeout(fn, 0)` does **not** mean "run immediately." The minimum delay is clamped (historically ~1ms) and the callback is a macrotask, so it runs *after* all synchronous code, all `process.nextTick` callbacks, and all Promise microtasks. If you need "run after the current operation but before timers," reach for `setImmediate` or `queueMicrotask` depending on the semantics you want.

> TIP: A `setInterval` callback that takes longer than its interval will *not* overlap — Node waits for the event loop to be free — but the calls will bunch up and drift. For precise periodic work that depends on the previous run finishing, prefer a self-rescheduling `setTimeout`:
>
> ```js
> async function poll() {
>   await doWork();
>   setTimeout(poll, 1000); // schedule the next run only after this one finishes
> }
> poll();
> ```

### 2.6 `queueMicrotask`

`queueMicrotask(fn)` schedules `fn` to run on the **microtask queue** — the same queue Promise `.then` callbacks use. It runs *after* the current synchronous code finishes but *before* any timer or I/O callback (any macrotask). It's the standardized, framework-agnostic way to defer work to "right after the current task, before yielding to the event loop."

```js
console.log("start");
queueMicrotask(() => console.log("microtask"));
console.log("end");
// Output: start, end, microtask
```

Why use it instead of `Promise.resolve().then(fn)`? It's clearer in intent, slightly cheaper (no Promise allocation), and standardized across environments. Use it when you want to defer a side effect until the current call stack unwinds — for example, to ensure callbacks always fire asynchronously even when data is already available, avoiding the "sometimes sync, sometimes async" footgun (a.k.a. "releasing Zalgo").

> TIP: There's also `process.nextTick(fn)`, which is Node-specific and runs on a queue that drains *before* the microtask queue — even before Promises. It's lower-level and easy to abuse. Prefer `queueMicrotask` for general "defer to microtask" needs; reserve `process.nextTick` for library code that must run before any Promise reactions.

| Mechanism | Queue | When it runs | Standard / Node-only |
| --- | --- | --- | --- |
| `process.nextTick(fn)` | nextTick queue | Before microtasks, before next phase | Node-only |
| `queueMicrotask(fn)` | microtask queue | After sync code, before macrotasks | Standard |
| `Promise.then(fn)` | microtask queue | Same queue as `queueMicrotask` | Standard |
| `setTimeout(fn, 0)` | timers (macrotask) | Next event-loop timers phase | Standard |
| `setImmediate(fn)` | check (macrotask) | After poll phase completes | Node-only |

---

## 3. The `process` Object — Your Window into the Running Program

`process` is a global object (technically an `EventEmitter`) that represents the currently running Node.js process. It exposes information about the process and lets you control its lifecycle. You never `require` it — it's always there.

### 3.1 `process.argv` — command-line arguments

`process.argv` is an array of the command-line arguments. **The first two entries are always fixed:**

- `argv[0]` — the absolute path to the `node` executable.
- `argv[1]` — the absolute path to the script being run.
- `argv[2]` and beyond — the actual arguments the user passed.

```js
// greet.js
console.log(process.argv);
```

```bash
$ node greet.js Ana --loud --count=3
[
  '/usr/local/bin/node',          // argv[0] - the node binary
  '/home/app/greet.js',           // argv[1] - this script
  'Ana',                          // argv[2]
  '--loud',                       // argv[3]
  '--count=3'                     // argv[4]
]
```

To get just the user-supplied arguments, slice off the first two:

```js
const userArgs = process.argv.slice(2); // ["Ana", "--loud", "--count=3"]
```

> TIP: Node also has `process.argv0`, the original `argv[0]` value as the process was launched. It's rarely needed but exists for edge cases where `argv[0]` gets rewritten.

### 3.2 `process.env` — environment variables

`process.env` is an object containing the environment variables of the process. Every value is a **string** (or `undefined` if not set) — there are no numbers or booleans.

```js
console.log(process.env.HOME);      // "/home/app" (on Unix)
console.log(process.env.PATH);      // the system PATH
console.log(process.env.NODE_ENV);  // "production" | "development" | undefined

// Setting a variable affects only THIS process (and children it spawns):
process.env.MY_FLAG = "on";
console.log(process.env.MY_FLAG);   // "on"
```

```bash
# Set an env var for a single run (Unix/macOS):
$ NODE_ENV=production node server.js

# Multiple vars:
$ PORT=8080 LOG_LEVEL=debug node server.js
```

The WHY: environment variables are the standard, language-agnostic way to inject **configuration** and **secrets** into a process without hard-coding them. Database URLs, API keys, ports, and feature flags belong in the environment — never in committed source code. We'll cover `dotenv` for managing them in Section 7.

> WARNING — Pitfall: Everything in `process.env` is a string. `process.env.PORT` is `"8080"`, not `8080`. `process.env.DEBUG === "false"` is comparing strings — and `Boolean("false")` is `true`! Always coerce explicitly:
>
> ```js
> const port = Number(process.env.PORT) || 3000;          // string -> number with fallback
> const debug = process.env.DEBUG === "true";             // string -> proper boolean
> ```

### 3.3 `process.cwd()` — current working directory

Returns the directory from which the Node process was launched. This is the base for resolving *relative* file paths and is independent of where your script file lives.

```js
console.log(process.cwd()); // e.g. "/home/app" — wherever you ran `node` from
```

```bash
$ cd /tmp && node /home/app/script.js
# Inside script.js: process.cwd() === "/tmp"  (where you launched from)
#                   __dirname        === "/home/app" (where the file is)
```

This contrast — `cwd()` vs `__dirname` — is exactly why we use `__dirname` for loading bundled assets and `cwd()` (or relative paths) for operating on user-provided locations.

> TIP: `process.chdir(directory)` changes the working directory at runtime, but use it sparingly — it's global mutable state that affects every relative path in your program and is a frequent source of confusing bugs in concurrent code.

### 3.4 Process identity and platform info

```js
console.log(process.pid);      // e.g. 48213 — the OS process ID (useful in logs)
console.log(process.ppid);     // parent process ID
console.log(process.platform); // "darwin" | "linux" | "win32" | "freebsd" | ...
console.log(process.arch);     // "x64" | "arm64" | "ia32" | ...
console.log(process.title);    // process name shown in `ps`/Activity Monitor
```

`process.platform` is how you write cross-platform code:

```js
// Pick the right shell command per OS:
const clearCmd = process.platform === "win32" ? "cls" : "clear";

// Windows uses "win32" even on 64-bit Windows — a famous gotcha.
const isWindows = process.platform === "win32";
```

> WARNING — Pitfall: `process.platform` returns `"win32"` on **all** Windows installations, including 64-bit ones. It's a historical name, not an architecture. Use `process.arch` to distinguish 32- vs 64-bit (`"ia32"` vs `"x64"`), and `process.platform === "win32"` simply to mean "is Windows."

### 3.5 `process.version` and `process.versions`

```js
console.log(process.version);  // "v22.22.2" — the Node.js version string (note the leading 'v')

console.log(process.versions);
// {
//   node: '22.22.2',          // Node version (no leading 'v' here)
//   v8: '12.4.254.21-node.31',// the V8 engine
//   uv: '1.48.0',             // libuv (the event-loop / async I/O library)
//   openssl: '3.0.13+quic',   // crypto/TLS
//   ... napi, modules, zlib, ares, etc.
// }
```

`process.version` is the single Node version string (with the `v` prefix). `process.versions` is an object breaking down every bundled component — invaluable for diagnosing "works on my machine" issues where the V8 or OpenSSL version actually differs.

```js
// Common pattern: gate a feature on the Node major version.
const major = Number(process.versions.node.split(".")[0]);
if (major < 18) {
  throw new Error(`Node 18+ required, found ${process.version}`);
}
```

### 3.6 `process.memoryUsage()` — memory diagnostics

Returns an object describing the process's memory footprint, in **bytes**.

```js
console.log(process.memoryUsage());
// {
//   rss: 39444480,        // Resident Set Size: total RAM held by the process
//   heapTotal: 7159808,   // V8 heap allocated
//   heapUsed: 5022040,    // V8 heap actually in use (JS objects)
//   external: 1089863,    // memory used by C++ objects bound to JS (e.g. Buffers)
//   arrayBuffers: 26818   // memory for ArrayBuffers/Buffers (subset of external)
// }
```

The most useful figures:
- **`rss`** (Resident Set Size) — total memory the OS has given the process, including code, stack, and heap. This is the number that matters for "is my container about to be OOM-killed?"
- **`heapUsed`** — JavaScript objects currently alive. Watch this climb steadily across requests to spot a **memory leak**.

```js
// Quick human-readable memory snapshot helper:
function memReport() {
  const m = process.memoryUsage();
  const mb = (n) => (n / 1024 / 1024).toFixed(1) + " MB";
  return `rss=${mb(m.rss)} heapUsed=${mb(m.heapUsed)} external=${mb(m.external)}`;
}
console.log(memReport()); // rss=37.6 MB heapUsed=4.8 MB external=1.0 MB
```

### 3.7 `process.uptime()` — how long the process has run

Returns the number of **seconds** (a float, sub-second precise) the current process has been running. Great for health-check endpoints.

```js
console.log(process.uptime()); // e.g. 12.473 (seconds since start)

// Common use: a /health endpoint reporting uptime.
// app.get("/health", (req, res) => res.json({ uptime: process.uptime() }));
```

Don't confuse it with `os.uptime()`, which is how long the *machine* has been booted.

### 3.8 `process.hrtime()` and `process.hrtime.bigint()` — high-resolution timing

For measuring durations precisely, `Date.now()` is too coarse and can jump (NTP clock adjustments, daylight saving). `process.hrtime` gives a **monotonic, nanosecond-resolution** clock that only moves forward — perfect for benchmarks.

The modern, recommended form is `process.hrtime.bigint()`, which returns nanoseconds as a `BigInt`:

```js
const start = process.hrtime.bigint();    // BigInt nanoseconds
doExpensiveWork();
const end = process.hrtime.bigint();
const ms = Number(end - start) / 1e6;     // convert ns -> ms
console.log(`Took ${ms.toFixed(3)} ms`);
```

The older form, `process.hrtime([previous])`, returns a `[seconds, nanoseconds]` tuple:

```js
const start = process.hrtime();           // [seconds, nanoseconds]
doExpensiveWork();
const [s, ns] = process.hrtime(start);    // diff against the start tuple
console.log(`Took ${s}s and ${ns}ns`);
```

> BEST PRACTICE: Use `process.hrtime.bigint()` (or the standardized `performance.now()` from `node:perf_hooks`, which is also global as `performance`) for measuring elapsed time. Never use `Date.now()` for benchmarking — wall-clock time can move backward.

---

## 4. Ending a Process: `exit()`, Exit Codes, and `exitCode`

### How a Node process ends naturally

A Node process exits on its own when the event loop has **nothing left to do** — no pending timers, no open sockets, no outstanding I/O. You don't normally call any exit function; when `server.js` is just `console.log("hi")`, it prints and exits with code `0`.

### `process.exit([code])` — force an exit

`process.exit(code)` terminates the process *immediately*. The optional `code` is the exit status returned to the OS.

```js
if (criticalConfigMissing) {
  console.error("FATAL: DATABASE_URL is not set");
  process.exit(1); // non-zero = failure
}
```

### Exit codes

Exit codes are how a process tells its parent (a shell, an orchestrator, a CI runner) whether it succeeded.

| Code | Meaning (convention) |
| --- | --- |
| `0` | Success |
| `1` | Generic/uncaught failure |
| `2` | Misuse of shell builtins (bash convention) |
| `>128` | Terminated by a signal; code is `128 + signal number` (e.g. `130` = SIGINT/Ctrl-C, `143` = SIGTERM) |

```bash
$ node script.js
$ echo $?     # prints the exit code of the last command (Unix)
```

Node also defines specific internal codes (e.g. `9` for invalid CLI arg) — see the docs — but for your own programs, stick to `0` for success and a small set of non-zero codes for documented failure modes.

### `process.exit()` is dangerous: prefer `process.exitCode`

Here's the critical pitfall. **`process.exit()` exits immediately, abandoning any pending asynchronous work** — including buffered writes to stdout/stderr. If you `console.log(...)` and then `process.exit()` on the next line, the log may never actually appear, because the write hadn't flushed yet.

```js
// RISKY: the log may be lost because exit() doesn't wait for the write to flush.
console.log("Goodbye!");
process.exit(0);
```

The safer pattern is to set **`process.exitCode`** and let the process exit *naturally* once the event loop drains. This flushes output and lets cleanup finish.

```js
// SAFE: signal the intended exit code, then let the process wind down on its own.
function fail(message) {
  console.error(message); // gets flushed because we don't kill the process
  process.exitCode = 1;   // process will exit with 1 once the loop is empty
  // do NOT call process.exit() — just return and stop scheduling work
}
```

> BEST PRACTICE: Set `process.exitCode = N` and let the program terminate naturally. Reserve `process.exit()` for the rare case where you genuinely must kill the process *right now* and can accept losing pending I/O — and even then, prefer to flush first or use it only after a graceful-shutdown timeout fires.

| | `process.exit(code)` | `process.exitCode = code` |
| --- | --- | --- |
| Timing | Immediate, abrupt | Process exits when event loop drains |
| Pending I/O / logs | May be lost | Flushed normally |
| Pending async work | Abandoned | Allowed to finish |
| Use when | Hard kill needed (last resort) | Almost always — the safe default |

---

## 5. Process Lifecycle Events & OS Signals

`process` is an `EventEmitter`, so you subscribe with `process.on(eventName, handler)`. These events let you hook into the program's lifecycle — for cleanup, error handling, and graceful shutdown.

### 5.1 `'exit'`

Fires when the process is about to exit, for any reason. The handler runs **synchronously only** — the event loop is already shutting down, so *async work here is ignored*. Use it for last-moment synchronous cleanup.

```js
process.on("exit", (code) => {
  // Only synchronous code works here. No setTimeout, no awaits, no I/O callbacks.
  console.log(`Process exiting with code ${code}`);
});
```

### 5.2 `'beforeExit'`

Fires when the event loop empties and Node is *about* to exit — but **only** when the exit is "natural" (the loop ran out of work). It does **not** fire when you call `process.exit()` explicitly or when the process is killed by a signal. Crucially, you *can* schedule more async work here (which would prevent exit). It's useful for flushing buffers or doing async teardown when a script finishes its work.

```js
process.on("beforeExit", async (code) => {
  // Async work IS allowed here (unlike 'exit').
  console.log("Nothing left to do; about to exit with", code);
  // e.g. await flushMetrics(); — but beware of creating an infinite loop.
});
```

### 5.3 `'uncaughtException'` — the synchronous safety net

Fires when a synchronous error (or one thrown in a callback) propagates all the way to the top without being caught. By default, an uncaught exception **crashes the process with code 1** — which is correct! The handler exists so you can *log* the fatal error and clean up, *not* to keep running.

```js
process.on("uncaughtException", (err, origin) => {
  console.error(`UNCAUGHT EXCEPTION (${origin}):`, err);
  // The process is now in an UNKNOWN, possibly corrupt state.
  // Log, attempt minimal cleanup, then exit. Do NOT resume normal operation.
  process.exit(1);
});

// Somewhere later, a bug throws synchronously with no try/catch:
JSON.parse("{ not valid json");
```

> WARNING — Pitfall: Using `uncaughtException` to swallow errors and "keep the server alive" is a serious anti-pattern. After an uncaught exception, your application state is undefined — half-finished writes, leaked connections, corrupt data structures. Log and exit; let a process manager (PM2, systemd, Kubernetes) restart you cleanly. Treat this handler as a black-box flight recorder, not an error-recovery mechanism.

### 5.4 `'unhandledRejection'` — the async safety net

The Promise-world counterpart. Fires when a rejected Promise has no `.catch()` handler. In modern Node (v15+), an unhandled rejection **crashes the process by default**, just like an uncaught exception — so handle your Promise rejections!

```js
process.on("unhandledRejection", (reason, promise) => {
  console.error("UNHANDLED REJECTION at:", promise, "reason:", reason);
  // Best practice: log it, then exit (let your supervisor restart).
  process.exit(1);
});

// This Promise rejects and nobody catches it:
Promise.reject(new Error("forgot to .catch()"));
```

> BEST PRACTICE: Don't rely on `unhandledRejection` for normal control flow — fix the missing `.catch()`/`try-catch` instead. Keep the handler purely as a logging-and-exit safety net so that a stray rejection produces a clear log line rather than a silent crash.

### 5.5 OS signals: `SIGINT` and `SIGTERM`

Signals are messages the operating system (or another process) sends to your process. Node turns the catchable ones into events on `process`.

| Signal | Trigger | Default behavior | Catchable? |
| --- | --- | --- | --- |
| `SIGINT` | Ctrl-C in the terminal | Terminate | Yes |
| `SIGTERM` | `kill <pid>`, orchestrator "please stop" (e.g. K8s) | Terminate | Yes |
| `SIGKILL` | `kill -9` | Terminate immediately | **No** (cannot be caught) |
| `SIGHUP` | Terminal closed / reload convention | Terminate | Yes |

```js
process.on("SIGINT", () => {
  console.log("\nReceived SIGINT (Ctrl-C). Cleaning up...");
  // Close servers, flush logs, then exit.
  process.exit(0);
});

process.on("SIGTERM", () => {
  console.log("Received SIGTERM. Shutting down gracefully...");
  process.exit(0);
});
```

> WARNING — Pitfall: `SIGKILL` (signal 9) and `SIGSTOP` **cannot be caught, blocked, or handled** — the OS kills the process instantly. This is *intentional*: it guarantees a runaway process can always be stopped. Never design cleanup logic that depends on catching `SIGKILL`. Orchestrators like Kubernetes send `SIGTERM` first, wait a grace period, *then* send `SIGKILL` — which is exactly why your graceful-shutdown handler must finish within that grace window.

---

## 6. The Graceful Shutdown Pattern

### The WHY

Imagine a web server handling 50 in-flight requests when the orchestrator sends `SIGTERM` to deploy a new version. If you call `process.exit()` immediately, those 50 users get dropped connections and possibly corrupted writes. **Graceful shutdown** means: stop accepting *new* work, finish the *in-flight* work, release resources (DB pools, sockets), then exit — all within the grace period before `SIGKILL` arrives.

The pattern has four parts:
1. Listen for `SIGTERM` and `SIGINT`.
2. Stop accepting new connections (`server.close()`).
3. Close downstream resources (database, message queue).
4. Set a **failsafe timeout** so a hung shutdown still terminates.

### Full example: graceful shutdown of an HTTP server

```js
// server.js
const http = require("node:http");

const server = http.createServer((req, res) => {
  // Simulate a slow request to demonstrate in-flight handling.
  setTimeout(() => {
    res.writeHead(200, { "Content-Type": "text/plain" });
    res.end("Hello!\n");
  }, 2000);
});

server.listen(3000, () => {
  console.log(`Server up on :3000 (pid ${process.pid})`);
});

// --- Graceful shutdown wiring ---

let shuttingDown = false;

async function shutdown(signal) {
  if (shuttingDown) return;     // guard against double signals
  shuttingDown = true;
  console.log(`\n${signal} received. Starting graceful shutdown...`);

  // 1) Stop accepting NEW connections; the callback fires once all
  //    in-flight requests have completed and the server is fully closed.
  server.close((err) => {
    if (err) {
      console.error("Error during server.close():", err);
      process.exitCode = 1;
      return;
    }
    console.log("All connections drained. Closing resources...");

    // 2) Close other resources here (DB pool, redis, queues), e.g.:
    //    await db.end();
    console.log("Cleanup complete. Bye.");
    // Let the process exit naturally now that nothing is pending.
  });

  // 3) Failsafe: if shutdown hangs beyond the grace period, force-exit.
  //    unref() so this timer itself does not keep the process alive.
  setTimeout(() => {
    console.error("Graceful shutdown timed out. Forcing exit.");
    process.exit(1);
  }, 10_000).unref();
}

// Handle the two signals you'll actually receive in production/dev.
process.on("SIGTERM", () => shutdown("SIGTERM")); // orchestrators, `kill`
process.on("SIGINT", () => shutdown("SIGINT"));   // Ctrl-C in dev
```

> TIP: The `.unref()` on the failsafe timer is subtle but important. A normal timer keeps the event loop alive (and the process running) until it fires. Calling `.unref()` tells Node "don't count this timer when deciding whether to exit." So if cleanup finishes early, the process exits naturally; the failsafe only matters if cleanup hangs.

> BEST PRACTICE: Always include the failsafe timeout. Without it, a single stuck database connection or a request that never ends will hang your shutdown forever — and the orchestrator will eventually `SIGKILL` you anyway, but uncleanly. The timeout converts an indefinite hang into a controlled, logged force-exit.

> WARNING — Pitfall: Register signal handlers **once**, near startup. Re-registering them inside request handlers leaks listeners and triggers Node's `MaxListenersExceededWarning`. Also remember handlers won't fire for `SIGKILL` — design for `SIGTERM`/`SIGINT` only.

---

## 7. Environment Variables & an Intro to `dotenv`

### Why environment variables

The [Twelve-Factor App](https://12factor.net/config) methodology says configuration that varies between deployments — credentials, hostnames, ports, toggles — should live in the **environment**, not in code. Benefits:

- **Security**: secrets never enter version control.
- **Portability**: the same build artifact runs in dev, staging, and prod with different env values.
- **Standardization**: every language and platform understands env vars.

### Reading config safely

```js
// config.js — centralize and validate config in ONE place.
function required(name) {
  const value = process.env[name];
  if (value === undefined || value === "") {
    throw new Error(`Missing required environment variable: ${name}`);
  }
  return value;
}

module.exports = {
  port: Number(process.env.PORT) || 3000,          // number, with default
  nodeEnv: process.env.NODE_ENV || "development",  // string, with default
  isProd: process.env.NODE_ENV === "production",   // derived boolean
  databaseUrl: required("DATABASE_URL"),           // no default — must be set
  logLevel: process.env.LOG_LEVEL || "info",
};
```

> BEST PRACTICE: Read `process.env` in exactly one module (`config.js`) at startup, validate everything, and export a typed-ish config object. The rest of your app imports `config`, never touching `process.env` directly. This makes missing config fail loudly at boot, centralizes coercion, and keeps env-var names from being scattered across the codebase.

### `dotenv`: loading a `.env` file in development

In development it's tedious to type env vars on every run. The **`dotenv`** package reads a `.env` file and loads its keys into `process.env`.

```bash
$ npm install dotenv
```

```bash
# .env  (in the project root — and ADD THIS TO .gitignore!)
PORT=8080
DATABASE_URL=postgres://localhost:5432/shop
LOG_LEVEL=debug
NODE_ENV=development
```

```js
// Load .env as the VERY FIRST thing, before any module reads process.env.
require("dotenv").config();

// Now process.env is populated from .env:
console.log(process.env.PORT);        // "8080"
console.log(process.env.DATABASE_URL); // "postgres://localhost:5432/shop"
```

For ES Modules, import the side-effecting entry first:

```js
import "dotenv/config"; // must come before other imports that read process.env
import { startServer } from "./server.js";
```

**Node's built-in alternative (no package needed):** modern Node (v20.6+) can load a `.env` file natively via the `--env-file` flag:

```bash
# No dotenv dependency required:
$ node --env-file=.env server.js
```

> WARNING — Pitfall #1: **Never commit your `.env` file.** It contains secrets. Add `.env` to `.gitignore` and commit a `.env.example` (with placeholder values and no real secrets) so teammates know which variables to set.

> WARNING — Pitfall #2: `dotenv` does **not overwrite** variables already present in `process.env` by default. A value set in the real environment (e.g. by your CI/orchestrator) wins over the `.env` file — which is the correct precedence (real environment > local file). Don't be surprised when your `.env` change "doesn't take" because a shell export is shadowing it.

> WARNING — Pitfall #3: Load `dotenv` *before* any code that reads `process.env`. If `config.js` runs at import time and you `require("dotenv").config()` *after* importing `config.js`, the config will read undefined values. Put the dotenv call at the literal top of your entry file (or use `--env-file`).

---

## 8. Standard Streams & Reading Command-Line Arguments

### 8.1 `process.stdin`, `process.stdout`, `process.stderr`

Every process has three standard streams, and Node exposes them on `process`:

- **`process.stdout`** — a *Writable* stream for normal output. `console.log` writes here.
- **`process.stderr`** — a *Writable* stream for errors/diagnostics. `console.error` writes here.
- **`process.stdin`** — a *Readable* stream for input piped or typed into the program.

The stdout/stderr split lets users redirect them independently:

```bash
$ node app.js > out.log 2> err.log   # stdout -> out.log, stderr -> err.log
$ node app.js 2>/dev/null            # discard errors, keep normal output
$ cat data.txt | node process.js     # pipe a file into process.stdin
```

Writing directly (no automatic newline, unlike `console.log`):

```js
process.stdout.write("Progress: ");
for (let i = 0; i <= 100; i += 25) {
  process.stdout.write(i + "% "); // stays on one line, no newline added
}
process.stdout.write("\n");
process.stderr.write("a diagnostic message\n");
```

### 8.2 Reading from stdin

```js
// echo-stdin.js — read everything piped in, then echo it back uppercased.
let input = "";

process.stdin.setEncoding("utf8");           // treat chunks as strings, not Buffers
process.stdin.on("data", (chunk) => {
  input += chunk;                            // accumulate as data arrives
});
process.stdin.on("end", () => {              // fires when input is exhausted (EOF)
  process.stdout.write(input.toUpperCase());
});
```

```bash
$ echo "hello world" | node echo-stdin.js
HELLO WORLD
```

For interactive line-by-line input, the `node:readline` module is cleaner:

```js
// ask-name.js — prompt the user interactively.
const readline = require("node:readline");

const rl = readline.createInterface({
  input: process.stdin,
  output: process.stdout,
});

rl.question("What is your name? ", (name) => {
  console.log(`Hello, ${name}!`);
  rl.close(); // closing releases stdin so the process can exit
});
```

### 8.3 Building a simple argument parser

`process.argv.slice(2)` gives raw strings. Real CLIs need to interpret flags (`--verbose`), options with values (`--count 3` or `--count=3`), and positional arguments. Here's a compact, dependency-free parser:

```js
// parseArgs.js — minimal flag/option/positional parser.
function parseArgs(argv = process.argv.slice(2)) {
  const result = { _: [] }; // `_` holds positional (non-flag) arguments
  for (let i = 0; i < argv.length; i++) {
    const token = argv[i];
    if (token.startsWith("--")) {
      const body = token.slice(2);          // strip the leading "--"
      if (body.includes("=")) {
        // Form: --key=value
        const [key, value] = body.split(/=(.*)/s); // split on FIRST "="
        result[key] = value;
      } else {
        // Form: --key value   OR   --flag (boolean)
        const next = argv[i + 1];
        if (next !== undefined && !next.startsWith("--")) {
          result[body] = next;              // consume the following token as the value
          i++;                              // skip it so it's not re-parsed
        } else {
          result[body] = true;              // a bare flag becomes boolean true
        }
      }
    } else {
      result._.push(token);                 // positional argument
    }
  }
  return result;
}

console.log(parseArgs(["build", "--out=dist", "--minify", "--level", "3"]));
// { _: [ 'build' ], out: 'dist', minify: true, level: '3' }

module.exports = { parseArgs };
```

> BEST PRACTICE: For anything beyond a toy, use Node's **built-in `util.parseArgs`** (stable since v18.3+/v20) instead of writing your own. It handles short flags, value coercion config, and is well tested:
>
> ```js
> const { parseArgs } = require("node:util");
> const { values, positionals } = parseArgs({
>   options: {
>     out: { type: "string", short: "o" },     // --out / -o <value>
>     minify: { type: "boolean", short: "m" },  // --minify / -m
>   },
>   allowPositionals: true,
> });
> // $ node build.js src --out dist -m
> // values     -> { out: 'dist', minify: true }
> // positionals -> [ 'src' ]
> ```
>
> For full-featured CLIs (subcommands, help text, validation), reach for a library like `commander` or `yargs`.

---

## Common Pitfalls & Gotchas

- **Expecting top-level `var` to land on `global`.** It doesn't — the module wrapper makes it a local. Use `globalThis.x = ...` explicitly if you truly need a global (you rarely do).
- **Using `__dirname`/`__filename` in ES Modules.** They're undefined in ESM. Use `import.meta.dirname`/`import.meta.filename` (Node 20.11+) or derive from `import.meta.url`.
- **Reassigning `exports = ...`.** Breaks the export. Use `module.exports = ...` to replace, `exports.name = ...` to add.
- **Treating `process.env` values as non-strings.** Everything is a string. `process.env.FLAG === "false"` is comparing strings; `Boolean(process.env.FLAG)` is `true` for any non-empty string. Coerce explicitly.
- **Calling `process.exit()` right after `console.log`.** Output can be truncated because exit abandons pending writes. Prefer `process.exitCode` and a natural exit.
- **Using `uncaughtException`/`unhandledRejection` to keep running.** After such an error your state is corrupt. Log and exit; let a supervisor restart you.
- **Forgetting the failsafe timeout in graceful shutdown.** A hung resource will block shutdown forever. Always add an `.unref()`'d timeout.
- **Relying on catching `SIGKILL`.** Impossible by design. Build cleanup around `SIGTERM`/`SIGINT`.
- **Confusing `process.cwd()` with `__dirname`.** `cwd()` is where node was launched; `__dirname` is where the file lives. Use `__dirname` for bundled assets.
- **`setTimeout(fn, 0)` is not "immediate."** It's a macrotask that runs after all sync code, `nextTick`, and microtasks.
- **Committing `.env`.** It holds secrets. Gitignore it; ship a `.env.example`.
- **Loading `dotenv` after reading config.** Call `require("dotenv").config()` (or use `--env-file`) before anything reads `process.env`.
- **`Buffer.allocUnsafe` leaking memory contents.** Use `Buffer.alloc` unless you measure a need and fully overwrite.

## Best Practices

- **Use `globalThis`** for portable code; avoid mutating the global object entirely when you can `require`/`import` instead.
- **Centralize configuration** in one `config.js` that reads, validates, and coerces `process.env` at startup, then export a config object the rest of the app consumes.
- **Validate required env vars at boot** so misconfiguration fails fast and loudly, not at 3 a.m. on the first request that needs `DATABASE_URL`.
- **Prefer `process.exitCode` over `process.exit()`**; let the event loop drain so output flushes and cleanup runs.
- **Implement graceful shutdown** for every long-running service: stop new work, drain in-flight work, close resources, failsafe-timeout, exit.
- **Treat the global error handlers as flight recorders** — log richly, then exit; rely on a process manager for restarts.
- **Use `process.hrtime.bigint()` or `performance.now()`** for timing, never `Date.now()`.
- **Prefer `node:` prefixes** for core modules (`require("node:fs")`) to make intent explicit and avoid shadowing by an npm package named `fs`.
- **Use `util.parseArgs`** (built in) or a mature CLI library instead of hand-rolling arg parsing for real tools.
- **Use `--env-file`** (built-in) for simple cases and `dotenv` when you need its richer features; in both cases keep secrets out of source control.

## Key Interview Points

- **Q: What is the global object in Node, and how does it differ from the browser's `window`?**
  A: It's `global` (aliased by the universal `globalThis`). Unlike the browser there's no DOM, and crucially top-level `var`/`let`/`const`/`function` declarations do **not** attach to `global` because every module is wrapped in a function — they become local variables of that wrapper.

- **Q: Why do `__dirname`, `require`, `module`, and `exports` exist in every file if they're not truly global?**
  A: Node wraps each CommonJS module in a function `(exports, require, module, __filename, __dirname) => {...}` before running it. Those identifiers are the wrapper's parameters, so they're per-module, not on the global object.

- **Q: What's the difference between `module.exports` and `exports`?**
  A: `exports` starts as a reference to `module.exports`. Adding properties (`exports.foo = ...`) works because both point to the same object. But reassigning `exports = ...` only rebinds the local variable and breaks the export — to replace the whole export you must assign `module.exports = ...`.

- **Q: What's the difference between `process.exit()` and `process.exitCode`?**
  A: `process.exit(code)` terminates immediately and can lose buffered I/O and abandon async work. Setting `process.exitCode = code` lets the process exit naturally when the event loop drains, flushing output and finishing cleanup. Prefer `exitCode`.

- **Q: What happens on an uncaught exception or unhandled rejection, and how should you handle them?**
  A: Both crash the process by default in modern Node (rejections since v15). Register `process.on("uncaughtException"|"unhandledRejection")` only to log and exit — not to keep running, because application state is unreliable afterward. Let a supervisor restart the process.

- **Q: Which signals can't you catch, and why does that matter for shutdown?**
  A: `SIGKILL` and `SIGSTOP` cannot be caught/handled — the OS acts immediately, by design, so runaway processes can always be stopped. Graceful shutdown must therefore hang off `SIGTERM`/`SIGINT` and complete within the orchestrator's grace period before `SIGKILL` arrives.

- **Q: Describe the graceful shutdown pattern.**
  A: On `SIGTERM`/`SIGINT`: stop accepting new connections (`server.close()`), finish in-flight requests, close downstream resources (DB pools, queues), and include an `.unref()`'d failsafe timeout that force-exits if cleanup hangs.

- **Q: Why are `process.env` values tricky?**
  A: They're always strings or `undefined`. `"false"` is truthy, and `process.env.PORT` is a string, so you must explicitly coerce (`Number(...)`, `=== "true"`) and provide defaults.

- **Q: `process.cwd()` vs `__dirname`?**
  A: `cwd()` is the directory from which `node` was launched (base for relative paths); `__dirname` is the directory of the current file. Use `__dirname` to load assets bundled with your code reliably.

- **Q: What does `process.argv` contain?**
  A: An array where `[0]` is the node binary path, `[1]` is the script path, and `[2..]` are the user arguments. Use `process.argv.slice(2)` to get just the user args.

- **Q: `setImmediate` vs `setTimeout(fn, 0)` vs `process.nextTick` vs `queueMicrotask`?**
  A: `nextTick` drains first (before microtasks); `queueMicrotask`/Promise callbacks run next (microtask queue); then macrotasks — `setTimeout(fn,0)` (timers phase) and `setImmediate` (check phase). Inside an I/O callback, `setImmediate` reliably runs before `setTimeout(fn,0)`.

- **Q: Why use `process.hrtime.bigint()` instead of `Date.now()` for timing?**
  A: It's a monotonic, nanosecond-resolution clock that never goes backward, whereas `Date.now()` is wall-clock and can jump due to NTP/DST adjustments, producing bogus or negative durations.

## Practice Exercises

1. **Environment snapshot.** Write `info.js` that prints the Node version, platform, architecture, PID, current working directory, and process uptime in a clean multi-line format.

2. **Safe config loader.** Write `config.js` that reads `PORT` (default `3000`, coerced to a number), `NODE_ENV` (default `"development"`), and `API_KEY` (required — throw a clear error if missing). Export the resulting config object and demonstrate it failing when `API_KEY` is unset and succeeding when it's set.

3. **Mini arg parser CLI.** Write `cli.js` that accepts `--name <value>` (string), `--shout` (boolean flag), and `--repeat <n>` (number). It should print the name, repeated `n` times, uppercased if `--shout` is present. Use Node's built-in `util.parseArgs`.

4. **Benchmark helper.** Write a function `time(label, fn)` that runs `fn`, measures its duration with `process.hrtime.bigint()`, and logs `"<label>: X.XXX ms"`. Use it to compare summing an array with a `for` loop versus `Array.prototype.reduce` over 10 million numbers.

5. **Graceful shutdown server.** Write an HTTP server on port `3000` that responds after a 3-second delay. Wire up `SIGINT`/`SIGTERM` to stop accepting new connections, log when all in-flight requests have drained, and force-exit after a 5-second failsafe. Test it: start a slow request, then press Ctrl-C, and confirm the in-flight request still completes.

6. **(Stretch) stdin line counter.** Write `wc.js` that reads all of stdin and prints the number of lines, words, and characters (like Unix `wc`). Test with `cat somefile.txt | node wc.js`.

### Solutions

**Solution 1 — Environment snapshot**

```js
// info.js
const lines = [
  `Node version : ${process.version}`,
  `Platform     : ${process.platform}`,
  `Architecture : ${process.arch}`,
  `PID          : ${process.pid}`,
  `CWD          : ${process.cwd()}`,
  `Uptime       : ${process.uptime().toFixed(3)}s`,
  `V8 version   : ${process.versions.v8}`,
];
console.log(lines.join("\n"));
```

```bash
$ node info.js
# Node version : v22.22.2
# Platform     : darwin
# ... etc.
```

**Solution 2 — Safe config loader**

```js
// config.js
function required(name) {
  const value = process.env[name];
  if (value === undefined || value === "") {
    throw new Error(`Missing required environment variable: ${name}`);
  }
  return value;
}

const config = {
  port: Number(process.env.PORT) || 3000,         // coerce to number with default
  nodeEnv: process.env.NODE_ENV || "development", // default string
  apiKey: required("API_KEY"),                    // required — throws if absent
};

module.exports = config;

// Demo (only runs when this file is executed directly, not when required):
if (require.main === module) {
  console.log("Loaded config:", config);
}
```

```bash
$ node config.js
# Error: Missing required environment variable: API_KEY

$ API_KEY=secret123 PORT=8080 node config.js
# Loaded config: { port: 8080, nodeEnv: 'development', apiKey: 'secret123' }
```

> The `require.main === module` check is the idiomatic "is this file being run directly vs imported?" guard — like Python's `if __name__ == "__main__"`.

**Solution 3 — Mini arg parser CLI**

```js
// cli.js
const { parseArgs } = require("node:util");

const { values } = parseArgs({
  options: {
    name: { type: "string" },                       // --name <value>
    shout: { type: "boolean", default: false },     // --shout
    repeat: { type: "string", default: "1" },       // --repeat <n> (strings only)
  },
});

const name = values.name ?? "world";
const repeat = Number(values.repeat) || 1;          // coerce: parseArgs gives strings
let message = `Hello, ${name}!`;
if (values.shout) message = message.toUpperCase();

for (let i = 0; i < repeat; i++) {
  console.log(message);
}
```

```bash
$ node cli.js --name Ana --shout --repeat 3
# HELLO, ANA!
# HELLO, ANA!
# HELLO, ANA!
```

> Note: `util.parseArgs` does not have a native number type — numeric options arrive as strings, so we coerce `repeat` with `Number(...)`.

**Solution 4 — Benchmark helper**

```js
// bench.js
function time(label, fn) {
  const start = process.hrtime.bigint();      // monotonic ns clock
  const result = fn();
  const end = process.hrtime.bigint();
  const ms = Number(end - start) / 1e6;        // ns -> ms
  console.log(`${label}: ${ms.toFixed(3)} ms`);
  return result;
}

const N = 10_000_000;
const arr = Array.from({ length: N }, (_, i) => i);

time("for loop", () => {
  let sum = 0;
  for (let i = 0; i < arr.length; i++) sum += arr[i];
  return sum;
});

time("reduce", () => arr.reduce((acc, n) => acc + n, 0));
```

```bash
$ node bench.js
# for loop: 11.204 ms
# reduce: 78.913 ms   (reduce is typically slower due to per-element callbacks)
```

**Solution 5 — Graceful shutdown server**

```js
// graceful-server.js
const http = require("node:http");

const server = http.createServer((req, res) => {
  setTimeout(() => {
    res.writeHead(200, { "Content-Type": "text/plain" });
    res.end("done\n");
  }, 3000); // 3s slow response to demonstrate draining
});

server.listen(3000, () => console.log(`Listening on :3000 (pid ${process.pid})`));

let shuttingDown = false;
function shutdown(signal) {
  if (shuttingDown) return;
  shuttingDown = true;
  console.log(`\n${signal} received — draining...`);

  server.close((err) => {
    if (err) {
      console.error("close error:", err);
      process.exitCode = 1;
      return;
    }
    console.log("All requests drained. Exiting cleanly.");
    // Natural exit now that nothing is pending.
  });

  setTimeout(() => {
    console.error("Failsafe timeout hit — forcing exit.");
    process.exit(1);
  }, 5000).unref(); // .unref() so it doesn't keep the process alive
}

process.on("SIGINT", () => shutdown("SIGINT"));
process.on("SIGTERM", () => shutdown("SIGTERM"));
```

```bash
# Terminal A:
$ node graceful-server.js
# Terminal B (start a slow request):
$ curl localhost:3000 &
# Back in Terminal A, press Ctrl-C:
# ^C
# SIGINT received — draining...
#   (curl still receives "done" after its 3s)
# All requests drained. Exiting cleanly.
```

**Solution 6 — stdin line/word/char counter**

```js
// wc.js
let input = "";

process.stdin.setEncoding("utf8");
process.stdin.on("data", (chunk) => { input += chunk; });
process.stdin.on("end", () => {
  const chars = input.length;
  // Words: split on any run of whitespace; filter out empty strings.
  const words = input.split(/\s+/).filter(Boolean).length;
  // Lines: count newline characters (matches `wc -l` behavior).
  const lines = (input.match(/\n/g) || []).length;
  process.stdout.write(`${lines}\t${words}\t${chars}\n`);
});
```

```bash
$ printf "hello world\nfoo bar baz\n" | node wc.js
# 2	5	24
```

---

## Summary / Cheat Sheet

| Topic | Key takeaway |
| --- | --- |
| Global object | `global` in Node; use `globalThis` for portability. Top-level `var` does **not** attach to it. |
| Module wrapper | Each file is wrapped in `(exports, require, module, __filename, __dirname) => {...}` — that's why those "globals" exist per-file. |
| `__dirname` / `__filename` | Absolute dir/file of the current module. Absent in ESM → use `import.meta.dirname`/`filename`. |
| `module.exports` vs `exports` | Add props via `exports.x`; replace the whole export via `module.exports = ...`. Never reassign `exports`. |
| `Buffer` | Raw bytes; `Buffer.from(str)`. Bytes ≠ characters. Prefer `Buffer.alloc` over `allocUnsafe`. |
| Timers | `setTimeout` (delay), `setInterval` (repeat), `setImmediate` (next loop), `clearXxx` to cancel. `setTimeout(fn,0)` ≠ immediate. |
| Microtasks | `queueMicrotask` (standard) and Promises run before macrotasks; `process.nextTick` runs before microtasks. |
| `process.argv` | `[node, script, ...userArgs]`; use `.slice(2)`. |
| `process.env` | Always strings/`undefined`; coerce explicitly; centralize & validate in `config.js`. |
| `process.cwd()` | Launch directory (relative-path base) — distinct from `__dirname`. |
| Identity/platform | `pid`, `ppid`, `platform` (`win32` on all Windows), `arch`, `version`, `versions`. |
| Diagnostics | `memoryUsage()` (`rss`, `heapUsed`), `uptime()` (seconds), `hrtime.bigint()` (monotonic ns timing). |
| Exiting | Prefer `process.exitCode = N` + natural exit over `process.exit()` (which can drop I/O). `0`=success. |
| Lifecycle events | `exit` (sync only), `beforeExit` (natural exit, async ok), `uncaughtException`, `unhandledRejection` → log & exit. |
| Signals | Handle `SIGTERM`/`SIGINT`; `SIGKILL`/`SIGSTOP` can't be caught. |
| Graceful shutdown | Stop new work → drain in-flight → close resources → `.unref()`'d failsafe timeout → exit. |
| Env config | Keep secrets out of git; load via `dotenv` or built-in `--env-file`; load before reading `process.env`. |
| Streams | `stdout` (log), `stderr` (errors), `stdin` (input). Redirect independently in the shell. |
| Arg parsing | Use built-in `util.parseArgs` (or `commander`/`yargs`) over hand-rolled parsers for real CLIs. |
