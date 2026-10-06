# What is Node.js & How It Works (Architecture)

You already write JavaScript. You know closures, promises, `async/await`, the DOM. But everything you've done so far has run inside a *browser* — a sandbox someone else built, that hands you `fetch`, `localStorage`, and a rendering engine. Node.js takes that same JavaScript language and runs it on a *server*: a long-lived program that opens files, listens on network ports, talks to databases, and serves thousands of users at once.

This module is the conceptual foundation for the rest of the course. By the end you will understand — from first principles — *why* Node is built the way it is, *how* it achieves high concurrency on a single thread, and *when* you should (and should not) reach for it. We won't write a web server yet (that's Week 2-3); here we build the mental model that makes every later module click.

> TIP: Keep a terminal open. Run every snippet as you read. The event loop is the single concept students most often "understand" on paper and then get wrong in an interview — executing the code is what makes it stick.

---

## What You'll Learn

- What Node.js actually *is* (a runtime, not a language or framework), its history, and why running JavaScript on the server changed backend development
- How the **V8 engine** compiles JavaScript to machine code with JIT compilation, and how Node *embeds* V8
- What **libuv** is, and how the **event loop** and **thread pool** give Node its concurrency
- A "diagram in words" of Node's internals: your JS, the C++ bindings, V8, and libuv — and how they fit together
- The **single-threaded, event-driven, non-blocking I/O** model explained from scratch, with no hand-waving
- The crucial difference between **blocking and non-blocking** code, and between **CPU-bound and I/O-bound** work — with runnable demonstrations
- How **Node.js differs from browser JavaScript**: globals, module systems, and available APIs
- How to **install Node**, manage multiple versions with **nvm**, and what **LTS vs Current** means
- How to use the **REPL**, run scripts, and the most useful **`node` CLI flags**
- **When Node is a great fit** and **when it is the wrong tool** — so you can defend your architecture choices

---

## 1. What Node.js Is (and What It Is Not)

Let's kill the most common beginner misconception immediately:

> **Node.js is not a programming language. It is not a framework. Node.js is a *runtime* — a program that lets you execute JavaScript outside the browser.**

A useful analogy: JavaScript is a language, like English. The browser is one place you can "speak" it. Node.js is another place. The grammar (the language) is the same; the *environment* and the *vocabulary of built-in capabilities* differ.

More precisely, Node.js is:

- **Google's V8 JavaScript engine** (the same one inside Chrome) — extracted and embedded into a standalone C++ program.
- Plus **libuv**, a C library that provides the event loop, asynchronous I/O, and a thread pool.
- Plus a set of **built-in modules** (`fs`, `http`, `crypto`, `path`, `stream`, …) that expose operating-system capabilities to your JavaScript.
- Plus a few **C++ bindings** and a bootstrap layer written in JavaScript that wires it all together.

| Term | What it actually is | Example |
|------|--------------------|---------|
| **Language** | The syntax & semantics you write | JavaScript (ECMAScript) |
| **Engine** | Compiles/executes the language | V8, SpiderMonkey, JavaScriptCore |
| **Runtime** | Engine + APIs + event loop; an environment to run code in | Node.js, Deno, Bun, the browser |
| **Framework** | A library that structures *your* app | Express, Fastify, NestJS |

So when someone says "I built it in Node," they mean: *I wrote JavaScript and ran it with the Node runtime.* Express, which you'll learn in Week 3, is a framework that runs **on top of** Node.

### Why this matters

Because the language is identical, you can use one language across your entire stack — browser front-end and server back-end. That single fact (sometimes called "isomorphic" or "full-stack JavaScript") is a large part of why Node became dominant: one mental model, one set of tooling (`npm`), shared validation/utility code between client and server, and a single hiring pool.

---

## 2. A Short History (and Why It Was a Big Deal)

Understanding *why* Node was created tells you what problem it is good at.

- **2009** — Ryan Dahl presented Node.js at JSConf EU. His motivation: existing web servers (notably Apache) handled concurrency by spawning a **thread or process per connection**. Each idle connection (a slow client, a long-polling request) still consumed memory and a thread. Dahl wanted a server that could hold **thousands of concurrent connections** cheaply by *not blocking* a thread while waiting for I/O.
- He chose JavaScript deliberately: the language already had **no built-in I/O** (so no legacy blocking I/O API to fight), first-class functions (perfect for callbacks), and a single-threaded execution model that pairs naturally with an event loop.
- **2010** — `npm` (Node Package Manager) launched, giving Node a package ecosystem. This, more than the runtime itself, drove adoption.
- **2014-2015** — A governance dispute led to a fork, **io.js**, which moved faster and tracked newer V8 versions. The communities reconciled and merged under the **Node.js Foundation** (now part of the **OpenJS Foundation**). Node **v4** was the first post-merger release, which is why Node jumped from 0.12 to 4.0.
- **Since then** — Predictable release cadence, semantic versioning, and a formal **LTS (Long Term Support)** schedule (more on this in §11). Modern Node has gained native `fetch`, Web Streams, a built-in test runner, a stable `--watch` mode, native `.env` loading, and ES Modules support.

> TIP: The historical "thread-per-connection vs event loop" framing is a classic interview opener. If asked "why was Node created?", the answer is: *to handle many concurrent I/O-bound connections efficiently without dedicating a thread to each one.*

---

## 3. The V8 Engine and JIT Compilation

### What an engine does

A JavaScript engine takes your human-readable source text and turns it into instructions the CPU can run. **V8** is Google's open-source engine, written in C++, that powers Chrome — and Node.

JavaScript is *not* purely interpreted line-by-line in any modern engine. V8 uses **Just-In-Time (JIT) compilation**: it compiles JavaScript to native machine code *while the program runs*, optimizing based on what it observes.

### How V8 processes your code (the pipeline)

1. **Parsing** — The source text is tokenized and parsed into an **Abstract Syntax Tree (AST)**, a structured representation of your code.
2. **Ignition (the interpreter)** — V8 compiles the AST to compact **bytecode** and starts executing it immediately. This gives a fast startup with low memory use.
3. **Profiling** — While Ignition runs the bytecode, V8 watches which functions run often ("hot") and what *types* they actually receive.
4. **TurboFan (the optimizing compiler)** — Hot functions are recompiled to highly optimized **machine code**, using the observed type information to make assumptions (e.g., "this function always gets two numbers").
5. **Deoptimization** — If a runtime assumption is later violated (you suddenly pass a string where numbers were expected), V8 throws away the optimized code and falls back to bytecode. This is why **stable, predictable types make JS faster.**

> BEST PRACTICE: Keep object "shapes" consistent. If you create objects with the same set of properties in the same order, V8 can reuse hidden classes and inline caches. Adding/deleting properties at random or mixing types in arrays forces deoptimization and slows hot paths. You rarely need to micro-optimize, but on hot code it matters.

### What V8 provides — and what it does *not*

V8 gives you the **language**: the syntax, the standard built-in objects (`Object`, `Array`, `Map`, `Promise`, `JSON`, `Math`), the garbage-collected heap, and the **call stack** + **microtask queue** that run your code.

V8 does **not** give you:

- File system access
- Networking / HTTP
- Timers (`setTimeout` is *not* part of the language — it's provided by the host)
- The event loop itself (V8 has a microtask queue, but the broader loop that schedules timers and I/O comes from the host)

Those capabilities come from the **host environment**. In the browser the host is the browser (it provides the DOM, `fetch`, `localStorage`). In Node, the host is **Node itself**, which provides them via libuv and C++ bindings. That division is the key to the whole architecture.

### How Node embeds V8

V8 is distributed as a C++ library with an **embedder API**. Node is a C++ program that:

1. Creates a V8 **Isolate** (an isolated instance of the engine with its own heap) and a **Context** (the global scope your JS runs in).
2. Injects host objects/functions into that context — `require`, `process`, `Buffer`, `setTimeout`, `console`, etc. — by binding them to native C++ implementations.
3. Hands your script to V8 to compile and run.
4. Runs the **event loop** (via libuv) so that asynchronous callbacks get scheduled back into V8 when their work completes.

You can even ask which V8 version your Node ships with:

```bash
# Print the V8 version bundled in your Node binary
node -p "process.versions.v8"

# Print all bundled component versions at once (v8, uv, openssl, zlib, ...)
node -p "JSON.stringify(process.versions, null, 2)"
```

```bash
# Example output (versions vary by Node release):
# {
#   "node": "22.x.x",
#   "v8": "12.4.x",
#   "uv": "1.48.x",
#   "openssl": "3.x.x"
# }
```

That `process.versions` object is your proof that Node = V8 + libuv (`uv`) + a stack of native libraries, all wrapped together.

---

## 4. libuv, the Event Loop, and the Thread Pool

V8 runs your JavaScript, but it has no idea how to read a file or open a socket. That job belongs to **libuv** — a C library originally written *for* Node that provides cross-platform asynchronous I/O.

libuv gives Node three things:

1. **The event loop** — the heartbeat that decides what runs next.
2. **Asynchronous I/O** — using the operating system's most efficient mechanisms (`epoll` on Linux, `kqueue` on macOS/BSD, IOCP on Windows) so that network and many I/O operations don't block.
3. **A thread pool** — a small pool of background threads for operations the OS *cannot* do asynchronously, or that are CPU-heavy.

### The event loop, in one sentence

> The event loop is a loop that repeatedly asks: *"Is there any completed asynchronous work whose callback is ready to run? If so, run those callbacks; if not, sleep until there is."*

When you call something like `fs.readFile('data.txt', cb)`, here is the high-level flow:

1. Your JS hands the request to a C++ binding, which asks libuv to read the file.
2. libuv either offloads it to the OS or to a thread-pool thread, and **your JavaScript keeps running** — it does not wait.
3. When the read finishes, libuv places your callback in the appropriate queue.
4. The event loop, on its next turn, picks up that callback and runs it (back inside V8).

The loop runs through several **phases** in order on each iteration (timers → pending callbacks → poll for I/O → check/`setImmediate` → close callbacks), draining the microtask queue (promises + `process.nextTick`) between steps. We cover the phases, their ordering, and output-prediction puzzles in depth in **Module 05 (The Event Loop, Timers & EventEmitter)** — here you only need the big picture: *one thread, many phases, callbacks scheduled as async work completes.*

### The thread pool (the part beginners miss)

People say "Node is single-threaded," and your JavaScript truly does run on one thread. But Node is **not** entirely single-threaded as a process. libuv maintains a **thread pool** (default size **4**) used for operations that have no async OS primitive, including:

- File system operations (`fs.*`) on most platforms
- DNS lookups via `dns.lookup` (which uses the blocking `getaddrinfo`)
- CPU-bound crypto like `crypto.pbkdf2`, `crypto.scrypt`, `crypto.randomBytes` (async forms)
- Compression (`zlib`)

Network I/O (TCP/UDP/HTTP sockets) usually does **not** use the thread pool — it uses the OS's event notification system directly, which scales to far more than 4 concurrent operations.

You can resize the pool with an environment variable (set it *before* the process starts):

```bash
# Increase the libuv thread pool from the default 4 to 8.
# Useful if you do lots of concurrent fs/crypto/zlib work.
UV_THREADPOOL_SIZE=8 node app.js
```

Here is a tiny demonstration that the pool is real and has a size of 4 by default:

```js
// threadpool.js
const crypto = require('node:crypto');

const start = Date.now();

// pbkdf2 is CPU-bound and runs on the libuv thread pool.
// Fire 5 of them. With a pool of 4, the first 4 finish together,
// and the 5th must wait for a free thread -> it finishes noticeably later.
function hash(i) {
  crypto.pbkdf2('secret', 'salt', 100000, 64, 'sha512', () => {
    console.log(`Hash ${i} done at +${Date.now() - start}ms`);
  });
}

for (let i = 1; i <= 5; i++) hash(i);
```

```bash
node threadpool.js
# Typical output (4 finish ~together, the 5th lands in a later "batch"):
# Hash 1 done at +210ms
# Hash 2 done at +212ms
# Hash 3 done at +214ms
# Hash 4 done at +215ms
# Hash 5 done at +410ms   <-- waited for a free thread
```

Now run it with a bigger pool and watch all five finish together:

```bash
UV_THREADPOOL_SIZE=5 node threadpool.js
# All five complete in roughly one batch.
```

> WARNING — Pitfall: The thread pool default is **4**, *not* the number of CPU cores. If you do heavy synchronous-style crypto/zlib/fs work and your throughput plateaus, the thread pool is a common bottleneck. Tune `UV_THREADPOOL_SIZE` (it can go up to 1024) — but measure first.

---

## 5. C++ Bindings and Node Internals — A Diagram in Words

Let's assemble the full picture. Imagine the architecture as layers, top (what you write) to bottom (the machine):

```text
┌─────────────────────────────────────────────────────────────┐
│  YOUR JAVASCRIPT  (app.js, your modules, npm packages)        │
├─────────────────────────────────────────────────────────────┤
│  NODE.JS STANDARD LIBRARY (written in JS):                    │
│  fs, http, path, stream, events, crypto, os, util, ...        │
├─────────────────────────────────────────────────────────────┤
│  C++ BINDINGS  (the "glue")                                   │
│  Translate JS calls <-> native C/C++ functions. Convert JS    │
│  values to C types and back, register callbacks with libuv.   │
├───────────────────────────────┬─────────────────────────────┤
│            V8 ENGINE           │            libuv             │
│  Parses, JIT-compiles & runs   │  Event loop, async I/O,      │
│  your JS. Manages the heap,    │  thread pool, timers,        │
│  call stack, microtask queue,  │  OS event notification       │
│  garbage collection.           │  (epoll/kqueue/IOCP).        │
├───────────────────────────────┴─────────────────────────────┤
│  OTHER C LIBRARIES: OpenSSL (crypto/TLS), zlib (compression), │
│  c-ares (async DNS), llhttp (HTTP parsing), ...               │
├─────────────────────────────────────────────────────────────┤
│  OPERATING SYSTEM  (system calls: open, read, socket, ...)    │
└─────────────────────────────────────────────────────────────┘
```

How a single async call travels through these layers — `fs.readFile('data.txt', cb)`:

1. **Your JS** calls `fs.readFile`. This is a JavaScript function in Node's standard library.
2. That function validates arguments and calls a **C++ binding** (the glue layer).
3. The binding asks **libuv** to perform the read. libuv hands the work to a **thread-pool thread** (filesystem reads are not async at the OS level on most platforms).
4. **Your JS keeps running.** `fs.readFile` returned immediately; the rest of your script executes. Node does not sit and wait.
5. The thread-pool thread finishes the read and notifies the **event loop**.
6. On its next appropriate phase, the event loop invokes the **C++ binding's completion handler**, which converts the raw bytes into a JS `Buffer` and **calls your `cb`** — back inside V8, on the main thread.

Two takeaways:

- **Your callback always runs on the single main JS thread.** The background thread only did the *waiting/work*; it never touches your JavaScript objects. That's why you almost never deal with locks, mutexes, or data races in normal Node code.
- The "magic" of async is just this hand-off: *start the work elsewhere, register a callback, keep going, get called back later.*

---

## 6. Single-Threaded, Event-Driven, Non-Blocking I/O — From Scratch

This is *the* defining model of Node. Let's build the intuition with an analogy, then the precise mechanics.

### The restaurant analogy

Picture a restaurant with **one waiter** (your single JS thread).

- **Blocking / thread-per-request model:** The waiter takes your order, walks to the kitchen, and *stands there waiting* until your food is cooked before serving anyone else. To serve 50 tables at once you'd need 50 waiters (50 threads) — most of them just standing in the kitchen waiting. Expensive.
- **Node's model:** The one waiter takes your order, hands it to the kitchen (the OS / thread pool), and **immediately** goes to serve other tables. When the kitchen rings a bell ("order ready!"), the waiter — whenever they're free between tasks — picks up the dish and delivers it (runs your callback). One waiter serves dozens of tables because **the waiting happens elsewhere.**

The waiter is never idle while there's work to do, and is never blocked waiting on the kitchen. That is **event-driven, non-blocking I/O on a single thread.**

The catch: if a *customer* corners the waiter and makes them do a long math problem at the table (a **CPU-bound** task), *every other table waits.* The waiter is busy and cannot ring up orders or deliver food. This is the single most important limitation of Node, and we'll demonstrate it in §7.

### The three terms, precisely

- **Single-threaded:** Your JavaScript executes on exactly one thread with one call stack. Two pieces of your JS never run literally simultaneously. (libuv's thread pool exists, but it runs C, not your JS.)
- **Event-driven:** The program is organized around *events* and the *callbacks* registered to handle them. Instead of "do A, then wait, then do B," you say "when A finishes, call this function." The event loop dispatches those callbacks.
- **Non-blocking I/O:** When you start an I/O operation, control returns to you *immediately* instead of pausing the thread until the I/O completes. The result arrives later via a callback / promise / event.

### Seeing it in action

```js
// nonblocking-order.js
console.log('1. Take order from Table A');

// Start a 2-second "cooking" task. This does NOT block the thread.
setTimeout(() => {
  console.log('4. Deliver Table A food (2s later)');
}, 2000);

console.log('2. Take order from Table B');
console.log('3. Take order from Table C');

// Output:
// 1. Take order from Table A
// 2. Take order from Table B
// 3. Take order from Table C
// 4. Deliver Table A food (2s later)
```

Notice lines 2 and 3 print **before** the Table A delivery, even though Table A was first. The thread didn't stop and wait for the timer; it kept taking orders. *That* is non-blocking, event-driven execution on a single thread.

> TIP: A great way to internalize this: any time you write a callback, a `.then()`, or `await`, you are telling Node "do this *later*, when the result is ready — don't make me wait now." The code *after* such a call (at the same level) generally runs first.

---

## 7. Blocking vs Non-Blocking Code; CPU-Bound vs I/O-Bound Work

These two distinctions decide whether Node is fast or falls on its face. Master them.

### Blocking vs non-blocking

**Blocking** code holds the single thread and prevents *anything else* from running until it finishes. **Non-blocking** code starts the work and lets the thread continue.

Most Node core APIs come in both flavors. The synchronous (`...Sync`) versions block:

```js
// blocking.js — the SYNCHRONOUS (blocking) version
const fs = require('node:fs');

console.log('Start');

// readFileSync BLOCKS the entire thread until the file is fully read.
// Nothing else — no other request, no timer — can run during this.
const data = fs.readFileSync('./big-file.txt', 'utf8');
console.log('File length:', data.length);

console.log('End');

// Output is strictly in order:
// Start
// File length: <n>
// End
```

```js
// nonblocking.js — the ASYNCHRONOUS (non-blocking) version
const fs = require('node:fs');

console.log('Start');

// readFile returns immediately; the read happens on the thread pool.
fs.readFile('./big-file.txt', 'utf8', (err, data) => {
  if (err) throw err;
  console.log('File length:', data.length); // runs LATER, via callback
});

console.log('End');

// Output:
// Start
// End
// File length: <n>   <-- arrives after the synchronous code finished
```

Why this matters on a server: imagine `readFileSync` inside an HTTP request handler. While that one request reads a large file, **every other incoming request is frozen** — they can't even be accepted by your JS — because the single thread is stuck. With the async version, the thread is free to handle other requests while the file is being read in the background.

> WARNING — Pitfall: `...Sync` APIs are fine in **CLI scripts and startup code** (e.g., reading a config file once before your server starts). They are almost always **wrong inside a request handler or any hot path** in a server, because they block *all* concurrent users. Default to the async versions on the server.

### CPU-bound vs I/O-bound work

This is a different axis. It's about *what the work actually is.*

- **I/O-bound work** spends most of its time *waiting* on something external: disk, network, database, another API. The CPU is mostly idle during the wait. **Node excels here** because the waiting happens off-thread and one thread can juggle thousands of in-flight waits.
- **CPU-bound work** spends its time *computing* on the CPU: image resizing, video encoding, cryptographic hashing, huge JSON parsing, complex math, ML inference. There is no "waiting" to offload — the CPU is the bottleneck. **This is Node's weakness**, because a long computation occupies the single thread and stalls everything else.

| | I/O-bound | CPU-bound |
|---|-----------|-----------|
| Bottleneck | Waiting on disk/network/DB | The CPU doing math |
| CPU usage during it | Mostly idle | Pegged at 100% |
| Node fit | Excellent | Poor (blocks the loop) |
| Examples | API calls, DB queries, file/stream serving, proxies | Image/video processing, hashing loops, big crypto, data crunching |
| How to handle in Node | Just use async APIs | Offload: Worker Threads, child processes, a queue, or a different service |

### Demonstration: a CPU-bound task freezes the event loop

```js
// cpu-block.js
const http = require('node:http');

// A deliberately heavy, synchronous computation (CPU-bound).
function blockCpuForMs(ms) {
  const end = Date.now() + ms;
  while (Date.now() < end) {
    // busy-wait: this loop OWNS the single thread the whole time
  }
}

const server = http.createServer((req, res) => {
  if (req.url === '/slow') {
    blockCpuForMs(5000); // 5 seconds of pure CPU work on the main thread
    res.end('done slow\n');
  } else {
    res.end('fast\n');
  }
});

server.listen(3000, () => console.log('http://localhost:3000'));
```

```bash
# Terminal 1: start the server
node cpu-block.js

# Terminal 2: hit /slow, then IMMEDIATELY hit / (fast) in another shell.
curl http://localhost:3000/slow &   # starts the 5s CPU task
curl http://localhost:3000/         # this "fast" request HANGS for ~5s
```

Even though `/` does almost no work, it waits ~5 seconds — because `/slow` monopolizes the single thread with CPU work. **No new requests, timers, or callbacks can run while the loop is blocked.** This is the canonical "don't block the event loop" lesson, and the reason CPU-heavy tasks must be offloaded (Worker Threads or a separate service — covered in **Module 24**).

> BEST PRACTICE: On a Node server, never run long synchronous computations on the main thread. For CPU-bound work, use **Worker Threads** (true parallelism within the process) or a **child process / job queue**. Keep the event loop free to do what it's best at: orchestrating I/O.

---

## 8. Node.js vs Browser JavaScript

Same language, different host. Knowing the differences prevents a lot of confusion (and is a frequent interview topic).

### Globals

| Concept | Browser | Node.js |
|---------|---------|---------|
| Global object | `window` (also `self`) | `global` (and the standard `globalThis` works in both) |
| "Current context" | `window` | The **module scope** — top-level `this` in a CommonJS module is `module.exports`, **not** `global` |
| The DOM | `document`, `window`, elements | **Absent** — there is no DOM, no `document` |
| Process info | Not available | `process` (argv, env, exit, stdin/stdout, versions) |
| Module helpers | n/a (script/`import`) | `require`, `module`, `exports`, `__dirname`, `__filename` (CJS) |
| Binary data | `Blob`, typed arrays | `Buffer` (Node-specific), plus typed arrays |
| Timers | `setTimeout` returns a number id | `setTimeout` returns a `Timeout` **object**; also `setImmediate` exists |

```js
// globals-demo.js  (run with: node globals-demo.js)
console.log(typeof window);     // "undefined"  — no browser window
console.log(typeof global);     // "object"     — Node's global
console.log(typeof globalThis); // "object"     — standard, works everywhere
console.log(typeof document);   // "undefined"  — no DOM
console.log(process.platform);  // "darwin" | "linux" | "win32"
console.log(__dirname);         // absolute path of this file's directory (CJS)
```

> WARNING — Pitfall: In a CommonJS module, top-level `this` is **`module.exports` (an empty object)**, *not* the global object. Code copied from a browser that relies on `this === window` will break. Use `globalThis` if you genuinely need the global object.

### Module systems

- **Browser:** Historically `<script>` tags (everything shared one global scope); modern browsers support **ES Modules** via `<script type="module">` and `import`/`export`.
- **Node:** Defaults to **CommonJS** (`require` / `module.exports`) but fully supports **ES Modules** (`import` / `export`) — chosen per file by extension (`.cjs` / `.mjs`) or by `"type"` in `package.json`. We dedicate **Module 04** to this entirely. In Node, each file is its own module with its own scope by default — there is no shared global scope like classic browser scripts.

### APIs

- **Browser-only:** DOM, `fetch` (long-standing), `localStorage`/`sessionStorage`, `XMLHttpRequest`, Web Audio, Geolocation, Service Workers, Canvas/WebGL.
- **Node-only:** `fs` (files), `http`/`net`/`dgram` (servers & low-level networking), `os`, `child_process`, `cluster`, `worker_threads`, `crypto` (full), `Buffer`, `process`, `stream`.
- **Shared / converging:** Both environments now offer many **Web Platform APIs**: `fetch`, `URL`, `TextEncoder`/`TextDecoder`, `structuredClone`, `AbortController`, Web Streams, `setTimeout`, `console`, `Promise`, timers. Modern Node deliberately implements these so code can be more portable.

```js
// fetch works in modern Node (built in) — no library needed
const res = await fetch('https://api.github.com/repos/nodejs/node');
const data = await res.json();
console.log(data.full_name); // "nodejs/node"
```

> TIP: When porting browser code to Node (or vice versa), the danger zones are: DOM access, `window`/`document`, and storage APIs. The pure-language parts (functions, classes, promises, array methods) move over unchanged.

---

## 9. Installing Node and Managing Versions with nvm

### The options

You can install Node via the official installer from nodejs.org, an OS package manager (Homebrew, apt, Chocolatey), or — strongly recommended for developers — a **version manager** like **nvm** (macOS/Linux) or **nvm-windows**/`fnm`/`volta` on Windows.

### Why a version manager

Different projects (and employers) pin different Node versions. A version manager lets you install many versions side-by-side and switch per project in seconds — without `sudo`, without fighting your system Node.

> BEST PRACTICE: Don't install Node with `sudo` into system paths. It leads to permission errors with global npm packages and makes upgrades painful. Use `nvm` (or `fnm`/`volta`) so Node lives in your home directory and switching is instant.

### Installing nvm (macOS / Linux)

```bash
# Install nvm (check github.com/nvm-sh/nvm for the latest version tag)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# Reload your shell config so the `nvm` command becomes available
source ~/.bashrc   # or ~/.zshrc on zsh (the default macOS shell)

# Verify
nvm --version
```

> NOTE for Windows: classic `nvm` is Unix-only. Use **nvm-windows**, **fnm**, or **volta** instead — the concepts (install, use, switch) are identical even if commands differ slightly.

### Everyday nvm commands

```bash
# Install the latest LTS ("Long Term Support") release
nvm install --lts

# Install a specific version
nvm install 22.11.0

# Install the latest Current release
nvm install node            # "node" is an alias for "latest"

# List what you have installed
nvm ls

# List everything available to install (LTS lines are labeled)
nvm ls-remote --lts

# Switch the active version for this shell session
nvm use 22

# Set a default version for new shells
nvm alias default 22

# Check which node is active and where it lives
node --version
which node
```

### Pinning a version per project

Drop a `.nvmrc` file in your project root so collaborators (and CI) use the right version:

```bash
# Create .nvmrc pinning the project to Node 22
node -v > .nvmrc        # writes e.g. "v22.11.0"
# or just: echo "22" > .nvmrc

# Then anyone in the project can run:
nvm use                 # reads .nvmrc and switches automatically
```

> TIP: Many shells can auto-run `nvm use` when you `cd` into a directory containing `.nvmrc` (via a shell hook documented in nvm's README), or you can use `fnm`/`volta`, which do this automatically.

---

## 10. The REPL, Running Scripts, and Useful `node` CLI Flags

### The REPL (Read–Eval–Print Loop)

Run `node` with no arguments to open an interactive prompt — perfect for experimenting:

```bash
node
```

```js
> 2 + 2
4
> const greet = (name) => `Hello, ${name}`;
undefined
> greet('Node')
'Hello, Node'
> const os = require('node:os')
undefined
> os.platform()
'darwin'
> _              // the special variable "_" holds the last evaluated result
'darwin'
```

Handy REPL specifics:

- `_` — the result of the **last** expression.
- `.help` — list REPL commands.
- `.editor` — multi-line editor mode (`Ctrl+D` to run, `Ctrl+C` to cancel).
- `.load file.js` / `.save file.js` — load/save a session.
- `.exit` or `Ctrl+D` — quit. (`Ctrl+C` twice also exits.)
- **Tab completion** works — type `os.` and press Tab to see available methods.

> TIP: The REPL auto-prints the value of each expression — that's the "P" (print). In a *script*, expressions are evaluated but **not** printed; you must call `console.log` yourself. This trips up beginners who copy REPL snippets into a file and wonder why nothing shows.

### Running scripts

```bash
# Run a file
node app.js

# Run a file by package main / directory (looks for index.js or package.json "main")
node .

# Evaluate an expression inline and DO print the result (-p = --print)
node -p "1 + 2"                       # prints 3
node -p "process.versions.node"       # prints your Node version

# Evaluate without printing (-e = --eval) — you print yourself
node -e "console.log(process.cwd())"  # prints current working directory

# Pipe code via STDIN
echo "console.log('hi')" | node
```

### The most useful CLI flags

| Flag | What it does | When to use |
|------|--------------|-------------|
| `--watch` | Re-runs the script when files change | Dev loop without `nodemon` (modern Node) |
| `--watch-path=dir` | Watch a specific directory | Scope what triggers restarts |
| `--env-file=.env` | Load env vars from a file into `process.env` | Local config without `dotenv` (modern Node) |
| `-p` / `--print` | Eval expression and print result | Quick one-liners, version checks |
| `-e` / `--eval` | Eval expression (no auto-print) | Inline scripts |
| `--inspect` | Start the V8 debugger (attach DevTools/VS Code) | Debugging |
| `--inspect-brk` | Same, but pause on the first line | Debug from the very start |
| `--check` / `-c` | Syntax-check a file without running it | Pre-flight / CI lint-lite |
| `--trace-warnings` | Show stack traces for process warnings | Diagnose deprecations/leaks |
| `--max-old-space-size=N` | Set V8 heap limit (MB) | Big-memory jobs / OOM tuning |
| `--prof` | Write a V8 profiling log | Performance investigation |
| `--test` | Run files with the built-in test runner | Testing (covered in Module 27) |

```bash
# Auto-restart on save (built-in; great for development)
node --watch app.js

# Load environment variables from a file (no dotenv dependency needed)
node --env-file=.env app.js

# Debug with breakpoints, paused at the first line
node --inspect-brk app.js
# then open chrome://inspect in Chrome, or attach VS Code's debugger
```

> WARNING — Pitfall: `--watch` and `--env-file` are modern Node features that were *experimental* in older releases. If a flag isn't recognized, check your Node version (`node -v`). This is another reason to stay on a current LTS.

---

## 11. LTS vs Current — and How to Choose

Node ships on a predictable schedule with two release lines:

- **Current** — the newest line, where new features land first. Cutting-edge but shorter support; good for trying new APIs, generally **not** what you run in production.
- **Active LTS (Long Term Support)** — a line promoted to long-term support, receiving bug fixes and security patches for an extended period (roughly **30 months** total), then entering **Maintenance** (critical/security fixes only) before going **End-of-Life (EOL)**.

Key rules of thumb:

- **Even-numbered** major versions (18, 20, 22, …) become **LTS**. **Odd-numbered** versions (19, 21, 23, …) never become LTS — they're shorter-lived "Current" lines.
- Each LTS line gets a friendly codename (e.g., "Iron", "Jod").
- A new even-numbered release starts as **Current** for ~6 months, then transitions to **Active LTS**.

| Line | Stability | Support length | Use it for |
|------|-----------|----------------|-----------|
| **Current** | Newest features, less battle-tested | ~6 months as Current | Experiments, library authors testing new APIs |
| **Active LTS** | Stable, recommended | ~12 months active, then maintenance | **Production and most learning** |
| **Maintenance LTS** | Frozen except fixes | Security/critical only until EOL | Legacy apps not yet upgraded |
| **EOL** | No patches | None | Upgrade *now* (security risk) |

> BEST PRACTICE: For production and for this course, run the **latest Active LTS** (an even-numbered major). Upgrade off any **EOL** version promptly — EOL means no more security patches. Use `nvm install --lts` to grab it.

```bash
# Always-current way to get the recommended line:
nvm install --lts
nvm use --lts

# Confirm you're on an even-numbered major (LTS line)
node -v   # e.g. v22.x.x  ✅   (v23.x.x would be Current, not LTS)

# See release/support details for any version in the REPL:
node -p "process.release"   # includes the LTS codename when on an LTS build
```

---

## 12. When Node Is a Great Fit — and When It Is Not

Engineering is about trade-offs. Knowing where Node shines (and where it doesn't) lets you defend architecture decisions in interviews and on the job.

### Great fit ✅

- **I/O-bound, high-concurrency network services** — REST/GraphQL APIs, BFFs (backend-for-frontend), API gateways/proxies. Most of the time is spent waiting on databases and other services; Node juggles thousands of concurrent waits cheaply.
- **Real-time apps** — chat, live dashboards, multiplayer, collaboration tools. The event-driven model and WebSockets (Module 28) are a natural fit.
- **Streaming** — proxying/transforming large files or media on the fly via Streams (Module 08), without buffering everything in memory.
- **Microservices** — small, fast-starting services; one language across the fleet.
- **Tooling & CLIs** — bundlers, linters, scaffolders, scripts. The npm ecosystem and quick startup make Node excellent here.
- **Full-stack JavaScript / SSR** — sharing validation and types between front-end and back-end; server-side rendering for React/Vue/etc.

### Poor fit ❌ (or "use with care")

- **CPU-bound workloads** — heavy image/video processing, scientific computing, ML training, big synchronous data crunching. A long computation blocks the single thread. *Mitigation:* Worker Threads, child processes, a job queue, or a service in Go/Rust/Python for the heavy lifting.
- **Hard real-time / deterministic latency** — garbage collection pauses and an event-loop model make Node a poor choice where microsecond-level, guaranteed timing is required (e.g., avionics, certain trading systems).
- **Raw single-threaded CPU throughput** — for pure number-crunching benchmarks, compiled languages (C++, Rust, Go) win. Node's strength is *concurrency of I/O*, not raw compute.
- **Memory-heavy in-process datasets** — V8's heap has limits (tunable with `--max-old-space-size`, but bounded). Holding very large data structures in memory may favor other runtimes.

> BEST PRACTICE: The honest one-line summary to give in an interview: *"Node is the right tool for I/O-bound, concurrent, event-driven workloads, and the wrong tool for CPU-bound work — for which you offload to Worker Threads, a separate process, or another service."*

---

## Common Pitfalls & Gotchas

> WARNING — Pitfall: **"Node is single-threaded, so it can't be fast."** Wrong framing. Your *JavaScript* runs on one thread, but I/O happens off-thread (OS + libuv thread pool). For I/O-bound work, one thread handling thousands of concurrent connections is *exactly* the point — and very fast.

> WARNING — Pitfall: **Blocking the event loop with CPU work.** A long synchronous loop, a huge synchronous `JSON.parse`, or `crypto.pbkdf2Sync` on the main thread freezes *all* requests. Symptom: unrelated endpoints hang or time out under load. Fix: offload CPU work (Module 24); never busy-wait.

> WARNING — Pitfall: **Using `...Sync` APIs in request handlers.** `fs.readFileSync`, `crypto.*Sync`, etc. block every concurrent user. Reserve them for startup/CLI scripts. In servers, use the async/promise versions.

> WARNING — Pitfall: **Assuming the thread pool size equals CPU cores.** It's **4** by default. Heavy concurrent `fs`/`crypto`/`zlib` work can queue behind those 4 threads. Tune `UV_THREADPOOL_SIZE` if profiling shows it's the bottleneck.

> WARNING — Pitfall: **Expecting browser APIs in Node.** `window`, `document`, `localStorage`, and the DOM do not exist. Conversely, `fs`, `process`, and `Buffer` don't exist in the browser. Check your host before using a global.

> WARNING — Pitfall: **Top-level `this` confusion.** In a CommonJS module, top-level `this` is `module.exports`, not `global`/`globalThis`. Browser snippets that assume `this === window` break.

> WARNING — Pitfall: **Running an EOL Node version.** No security patches. Check `node -v` against the official release schedule and stay on a supported LTS.

> WARNING — Pitfall: **Confusing the REPL with scripts.** The REPL auto-prints every expression; a script does not. Use `console.log` in files.

---

## Best Practices

> BEST PRACTICE: **Run the latest Active LTS** for learning and production. Install it with `nvm install --lts`.

> BEST PRACTICE: **Use a version manager (`nvm`/`fnm`/`volta`)** and commit a `.nvmrc` so your whole team and CI use the same Node version.

> BEST PRACTICE: **Default to asynchronous APIs** in any server or hot path. Treat `...Sync` as startup-only.

> BEST PRACTICE: **Keep the event loop free.** Offload CPU-bound work to Worker Threads, child processes, or a queue. Measure event-loop lag in production (e.g., via monitoring) so you catch blocking early.

> BEST PRACTICE: **Prefer the `node:` prefix for core modules** (`require('node:fs')`, `import 'node:path'`). It's unambiguous, slightly faster to resolve, and can't be shadowed by a same-named npm package.

> BEST PRACTICE: **Lean on built-ins before adding dependencies.** Modern Node has `fetch`, `--watch`, `--env-file`, a test runner, and Web Streams — you often don't need `nodemon`, `dotenv`, `node-fetch`, or extra test tooling.

> BEST PRACTICE: **Profile before tuning.** Don't bump `UV_THREADPOOL_SIZE` or `--max-old-space-size` on a hunch — confirm the bottleneck with `--prof`, the `--inspect` profiler, or metrics first.

---

## Key Interview Points

- **"What is Node.js?"** — A runtime that lets you run JavaScript outside the browser. It embeds **V8** (runs the JS) and **libuv** (event loop, async I/O, thread pool), plus core modules and C++ bindings. *Not* a language or framework.
- **"Is Node single-threaded?"** — Your JavaScript runs on a single thread (one call stack). But Node as a process uses libuv's **thread pool** (default 4) for some operations (fs, DNS, crypto, zlib), and the OS handles network I/O asynchronously. So: single-threaded *JS*, multi-threaded *runtime*.
- **"Explain the event loop."** — A loop that runs registered callbacks for completed async work, cycling through phases (timers, pending callbacks, poll, check, close) and draining microtasks (promises, `process.nextTick`) between steps. It's how one thread serves many concurrent operations. (Deep dive: Module 05.)
- **"What is non-blocking I/O?"** — Starting an I/O operation returns control immediately; the result arrives later via callback/promise/event, so the thread isn't stalled waiting.
- **"V8 — what does it do and how?"** — Google's engine; parses JS to an AST, runs bytecode in the **Ignition** interpreter, and JIT-compiles hot functions to machine code with **TurboFan**, deoptimizing when type assumptions break.
- **"What is libuv?"** — The C library providing Node's event loop, cross-platform async I/O (epoll/kqueue/IOCP), and the thread pool.
- **"CPU-bound vs I/O-bound — and Node's fit?"** — I/O-bound (waiting on disk/network/DB) is Node's sweet spot; CPU-bound (computation) blocks the single thread and should be offloaded to Worker Threads / separate processes.
- **"Node vs browser JS?"** — Same V8 language engine, different host: browser has DOM/`window`/`localStorage`; Node has `process`/`fs`/`Buffer`/`require`. Many Web APIs (`fetch`, `URL`, timers) now exist in both.
- **"Why was Node created?"** — To handle many concurrent connections efficiently with non-blocking I/O, instead of the costly thread-per-connection model.
- **"LTS vs Current?"** — Even-numbered majors become LTS (stable, ~30 months support) — use for production; odd-numbered are short-lived Current lines for new features. Never run EOL versions.
- **"What's `UV_THREADPOOL_SIZE`?"** — Env var controlling libuv's thread-pool size (default 4, max 1024); raise it for heavy concurrent fs/crypto/zlib workloads.
- **"How do you avoid blocking the event loop?"** — Use async APIs, never busy-wait, and offload CPU-heavy work (Worker Threads / child process / queue).

---

## Practice Exercises

Try each before peeking at the solutions. Type and run them — don't just read.

1. **Globals detective.** Write a script `env-report.js` that prints whether `window`, `document`, `global`, and `globalThis` are defined, plus `process.platform`, the Node version (from `process.versions`), and the libuv version. Explain in a comment why two of those globals are `undefined`.

2. **Order of execution.** Without running it, predict the exact output order of the program below, then run it to confirm. Explain *why* in one sentence each.
   ```js
   console.log('A');
   setTimeout(() => console.log('B'), 0);
   Promise.resolve().then(() => console.log('C'));
   console.log('D');
   ```

3. **Blocking vs non-blocking, felt.** Create a ~5 MB text file, then write two scripts that read it: one with `fs.readFileSync` and one with `fs.readFile`. In *both*, also schedule a `setTimeout(() => console.log('timer fired'), 0)` *before* the read. Observe and explain the difference in when `'timer fired'` prints relative to the file result.
   ```bash
   # create a ~5MB file of repeated text
   node -e "require('node:fs').writeFileSync('big.txt','x'.repeat(5_000_000))"
   ```

4. **Prove the thread pool size.** Using `crypto.pbkdf2` (async), fire 8 hashes at once and log how long each takes from start. Run it once with the default pool and once with `UV_THREADPOOL_SIZE=8`. Explain the "batching" you see in the default run.

5. **Freeze the loop (and feel it).** Write an HTTP server with two routes: `/` returns instantly, and `/block` busy-waits for 3 seconds. Start it, then time how long a request to `/` takes while `/block` is in flight. Then describe (in comments) one correct way to handle the heavy work without freezing `/`.

6. **CLI version-aware tool.** Write `node-info.js` that prints: the running Node version, whether it's an even (LTS-line) or odd (Current-line) major, the active V8 version, and the number of CPU cores (`os.cpus().length`). Bonus: print a warning if the major version is odd.

---

### Solutions

**Solution 1 — Globals detective**

```js
// env-report.js
console.log('window     :', typeof window);     // "undefined" — no browser
console.log('document   :', typeof document);   // "undefined" — no DOM
console.log('global     :', typeof global);     // "object"
console.log('globalThis :', typeof globalThis); // "object"
console.log('platform   :', process.platform);  // darwin | linux | win32
console.log('node       :', process.versions.node);
console.log('libuv (uv) :', process.versions.uv);

// WHY: `window` and `document` are part of the *browser* host environment.
// Node's host doesn't include a DOM, so they don't exist. The language-level
// global object is exposed as `global` (Node-specific) and `globalThis`
// (the standard name, available in every JS host).
```

**Solution 2 — Order of execution**

```text
A   <- synchronous, runs first
D   <- synchronous, runs before any async callback
C   <- microtask (Promise) — microtasks run after the current synchronous
       code, BEFORE the next event-loop phase (timers)
B   <- macrotask (setTimeout) — timers run in a later event-loop phase,
       after the microtask queue has been drained
```

- `A` and `D` are plain synchronous statements, so they print in source order before anything async.
- `C` is a **microtask**: the promise callback is queued and runs as soon as the current synchronous run finishes, *before* the timer phase.
- `B` is a **macrotask** (timer): it runs in the event loop's timers phase, after microtasks. Hence `A, D, C, B`. (Phase ordering is covered fully in Module 05.)

**Solution 3 — Blocking vs non-blocking, felt**

```js
// read-sync.js — BLOCKING
const fs = require('node:fs');
setTimeout(() => console.log('timer fired'), 0); // scheduled first...
console.log('before read');
const data = fs.readFileSync('big.txt', 'utf8'); // ...but this BLOCKS the thread
console.log('read done, length:', data.length);
// Output order:
// before read
// read done, length: 5000000
// timer fired               <-- the timer can't fire until the blocking read
//                               releases the thread, even though delay was 0
```

```js
// read-async.js — NON-BLOCKING
const fs = require('node:fs');
setTimeout(() => console.log('timer fired'), 0);
console.log('before read');
fs.readFile('big.txt', 'utf8', (err, data) => {
  if (err) throw err;
  console.log('read done, length:', data.length);
});
console.log('after read call'); // runs immediately; read is in background
// Typical output:
// before read
// after read call
// timer fired               <-- timer fires while the file reads in background
// read done, length: 5000000
```

The point: in the **sync** version, `readFileSync` owns the single thread, so even a `0 ms` timer can't fire until the read completes. In the **async** version the read happens on the thread pool, the thread stays free, the timer fires, and the callback runs when the read finishes.

**Solution 4 — Prove the thread pool size**

```js
// pool-proof.js
const crypto = require('node:crypto');
const start = Date.now();

for (let i = 1; i <= 8; i++) {
  crypto.pbkdf2('pw', 'salt', 100000, 64, 'sha512', () => {
    console.log(`#${i} done at +${Date.now() - start}ms`);
  });
}
```

```bash
node pool-proof.js
# Default pool = 4 -> you see TWO batches: #1-4 finish together (~Tms),
# then #5-8 finish together (~2T ms), because only 4 threads run at once.

UV_THREADPOOL_SIZE=8 node pool-proof.js
# Now all 8 run concurrently and finish in roughly ONE batch.
```

Explanation: `pbkdf2` is CPU-bound and runs on libuv's thread pool. With 4 threads, only 4 hashes compute at a time; the remaining 4 wait for a free thread, producing the visible two-batch timing. Raising the pool to 8 lets all run at once.

**Solution 5 — Freeze the loop (and feel it)**

```js
// freeze.js
const http = require('node:http');

function busyWait(ms) {
  const end = Date.now() + ms;
  while (Date.now() < end) { /* hog the thread */ }
}

http.createServer((req, res) => {
  if (req.url === '/block') {
    busyWait(3000);            // 3s of CPU on the main thread
    res.end('blocked done\n');
  } else {
    res.end('instant\n');
  }
}).listen(3000, () => console.log('http://localhost:3000'));

// Test:
//   curl http://localhost:3000/block &     # start the 3s hog
//   time curl http://localhost:3000/        # this "instant" call takes ~3s!
//
// CORRECT FIX (conceptually, full treatment in Module 24):
//   Move busyWait into a Worker Thread (worker_threads) or a child process /
//   job queue. Then /block computes off the main thread, and / stays instant
//   because the event loop is never blocked.
```

**Solution 6 — CLI version-aware tool**

```js
// node-info.js
const os = require('node:os');

const major = process.versions.node.split('.').map(Number)[0];
const isLtsLine = major % 2 === 0; // even majors become LTS, odd are Current

console.log('Node version :', process.versions.node);
console.log('Release line :', isLtsLine ? 'even (LTS line)' : 'odd (Current line)');
console.log('V8 version   :', process.versions.v8);
console.log('CPU cores    :', os.cpus().length);

if (!isLtsLine) {
  console.warn('⚠️  Odd major version detected — this is a Current line, not LTS. Prefer an even-numbered LTS for production.');
}
```

---

## Summary / Cheat Sheet

| Concept | One-line takeaway |
|---------|-------------------|
| **Node.js** | A *runtime* (V8 + libuv + core modules + C++ glue) that runs JavaScript outside the browser. Not a language or framework. |
| **V8** | Google's engine; parses → bytecode (Ignition) → JIT machine code (TurboFan); deopts on type changes. Runs your JS. |
| **libuv** | C library: provides the **event loop**, async I/O (epoll/kqueue/IOCP), and the **thread pool** (default 4). |
| **C++ bindings** | The glue translating JS calls to native C/C++ and registering callbacks with libuv. |
| **Single-threaded** | Your JS runs on one thread; the *process* uses background threads for some I/O/CPU work. |
| **Event-driven** | Organized around events + callbacks dispatched by the event loop. |
| **Non-blocking I/O** | Start I/O, get control back immediately, receive results later via callback/promise/event. |
| **Blocking code** | `...Sync` APIs and long loops hog the thread — fine in CLI/startup, *bad* in servers. |
| **I/O-bound** | Waiting on disk/network/DB → **Node's strength**. |
| **CPU-bound** | Heavy computation → **Node's weakness**; offload to Worker Threads / child process / queue. |
| **Node vs browser** | Same language; Node has `process`/`fs`/`Buffer`/`require`, no DOM/`window`/`localStorage`. Both share `fetch`, `URL`, timers, `globalThis`. |
| **`UV_THREADPOOL_SIZE`** | Env var to resize libuv's thread pool (default 4, max 1024). |
| **nvm** | Install/switch Node versions; pin per-project with `.nvmrc`. `nvm install --lts`. |
| **LTS vs Current** | Even majors = LTS (stable, prod); odd = Current (new features, short life). Never run EOL. |
| **REPL** | `node` with no args; auto-prints expressions; `_` = last result; `.editor`, `.help`, `.exit`. |
| **Key CLI flags** | `--watch`, `--env-file=.env`, `-p`/`-e`, `--inspect[-brk]`, `--check`, `--max-old-space-size`. |
| **Golden rule** | *Don't block the event loop.* Use async APIs; offload CPU work. |

**Quick command reference:**

```bash
node -v                         # Node version
node -p "process.versions"      # all bundled component versions (v8, uv, ...)
node                            # open the REPL
node app.js                     # run a script
node --watch app.js             # auto-restart on file changes
node --env-file=.env app.js     # load env vars without dotenv
nvm install --lts && nvm use --lts   # install & use the recommended Node
UV_THREADPOOL_SIZE=8 node app.js     # enlarge the libuv thread pool
```

Next up: **Module 04 — Module Systems (CommonJS & ESM)**, where we unpack `require`, `module.exports`, `import`/`export`, and how Node decides which system a file uses.
