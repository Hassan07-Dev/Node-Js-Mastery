# Advanced Node: Threads, Processes & Scaling

You already know that Node.js is "single-threaded and non-blocking." That phrase is repeated so often that it has become a slogan rather than an understanding. In this module we are going to take it apart and rebuild it precisely, because the moment you ship a real production service, the comfortable single-threaded model develops cracks: a CPU-heavy request freezes every other request, a single Node process leaves 15 of your 16 CPU cores idle, and a crash takes down your entire service.

This is the module where you learn to make Node use *all* of a machine's cores, run genuinely parallel CPU work without blocking the event loop, spawn and coordinate other programs, and scale a stateless service across many machines behind a load balancer. These are exactly the topics that separate a mid-level Node developer from a senior one, and they are heavily represented in interviews because they reveal whether you actually understand the runtime or just memorized "async/await."

By the end you will be able to choose correctly between `child_process`, `worker_threads`, and `cluster`; offload CPU-bound work to a worker pool; run a production app in PM2 cluster mode with zero-downtime reloads; and reason clearly about horizontal scaling, statelessness, and sticky sessions.

## What You'll Learn

- **A precise recap of Node concurrency**: the single JS thread, the libuv thread pool, what actually runs in parallel, and how to tune `UV_THREADPOOL_SIZE`.
- **`child_process`**: the differences between `spawn`, `exec`, `execFile`, and `fork`; streaming a child's output; and full duplex IPC with `fork`.
- **`worker_threads`**: when you need *real* threads, how to create them, every way to pass data (`workerData`, `postMessage`, transferables, `SharedArrayBuffer`), and how to build a reusable worker pool (the idea behind `piscina`).
- **The `cluster` module**: forking one worker per CPU core, the primary/worker model, how a port is shared, and how load is balanced.
- **A decision framework** for choosing among `cluster`, `worker_threads`, and `child_process`.
- **Handling CPU-bound tasks** without blocking the event loop — chunking, offloading, and measuring with `perf_hooks`.
- **PM2 in cluster mode** with graceful zero-downtime reloads.
- **Horizontal scaling**: statelessness, externalizing state, and sticky sessions for WebSockets.
- **Pitfalls, best practices, interview answers, and progressively harder exercises** to lock it all in.

> All examples target **Node.js current LTS (the v20/v22 line)** and use CommonJS (`require`) by default. Where ES Modules differ, that is called out. The APIs shown are current and non-deprecated as of this writing.

---

## Part 1: Recap — How Node Concurrency Actually Works

Before we add threads and processes, we need an honest mental model of what a single Node process is doing. Most concurrency bugs and most wrong scaling decisions come from a fuzzy version of this model.

### The single thread is the *JavaScript* thread

When we say Node is single-threaded, we mean precisely this: **your JavaScript executes on exactly one thread, the main thread, driven by one event loop.** Two pieces of your JS code never run at literally the same instant within one process. This is why you almost never need locks or mutexes in ordinary Node code — there is no preemption between your statements.

What makes Node fast despite the single JS thread is that **I/O does not happen on that thread.** When you call `fs.readFile` or make a network request, Node hands the work off to the operating system or to a background thread pool, and your JS thread is immediately free to do other things. When the I/O finishes, a callback is queued and the event loop runs it. This is the "non-blocking I/O" half of the slogan.

### The two kinds of "parallelism" hiding inside one process

There are actually two separate mechanisms doing background work, and conflating them is a classic mistake:

1. **The OS kernel via the event loop (epoll/kqueue/IOCP).** Most network I/O — TCP sockets, HTTP — is handled by the kernel's event notification system. libuv asks the OS "tell me when this socket is readable" and gets on with life. This scales to tens of thousands of connections **without** using extra threads. This is the heart of why Node handles high concurrency on a single thread.

2. **The libuv thread pool.** Some operations cannot be done with non-blocking OS primitives, so libuv runs them on a small pool of background threads. By default this pool has **4 threads**. The categories that use it:
   - **File system operations** (`fs.*` async methods) — there is no portable non-blocking file I/O, so these go to the pool.
   - **DNS lookups** via `dns.lookup` (which uses the blocking `getaddrinfo`). Note: `dns.resolve*` uses the network and does **not** use the pool.
   - **Some crypto** (`crypto.pbkdf2`, `crypto.scrypt`, `crypto.randomBytes` when async, key generation).
   - **`zlib`** (gzip/deflate) async operations.

> WARNING — Pitfall: People assume *all* async work is unlimited and parallel. It is not. If you fire 100 concurrent `fs.readFile` calls, only 4 are truly in flight at once (default pool size); the rest queue. Pure network requests, by contrast, are not limited by the pool. Knowing which operations hit the pool is a senior-level distinction.

### Visualizing the model

```text
                ┌─────────────────────────────────────────────┐
                │            ONE NODE PROCESS                   │
                │                                               │
   your JS ───▶ │   Main Thread (V8 + Event Loop, single)      │
                │        │                                       │
                │        ├── network I/O ──▶ OS kernel (epoll)  │ ← scales huge, no threads
                │        │                                       │
                │        └── fs / dns / crypto / zlib ──▶        │
                │                    libuv thread pool           │ ← default 4 threads
                │                    [T1][T2][T3][T4]            │
                └─────────────────────────────────────────────┘
```

### Tuning the pool: `UV_THREADPOOL_SIZE`

The pool size is fixed when the process starts and is controlled by the environment variable `UV_THREADPOOL_SIZE` (max 1024). You set it **before** the process launches, not at runtime.

```bash
# Increase the libuv thread pool to 8 background threads
UV_THREADPOOL_SIZE=8 node server.js
```

You can also set it from inside your code, but **only if you do it before any pool work happens** — practically, at the very top of your entry file:

```js
// Must be set before libuv lazily initializes the pool (i.e., before any fs/crypto/dns/zlib call).
process.env.UV_THREADPOOL_SIZE = 8;

const fs = require('node:fs');
// ...rest of app
```

Here is a demonstration that the pool size actually gates concurrency. Each `pbkdf2` call occupies one pool thread for a while:

```js
// threadpool-demo.js
// Run with: UV_THREADPOOL_SIZE=2 node threadpool-demo.js   (then try =4, =8)
const crypto = require('node:crypto');

const start = Date.now();
const elapsed = () => `${Date.now() - start}ms`;

// Fire 4 CPU-heavy crypto operations that each occupy one pool thread.
for (let i = 1; i <= 4; i++) {
  crypto.pbkdf2('password', 'salt', 200000, 64, 'sha512', () => {
    console.log(`pbkdf2 #${i} finished at ${elapsed()}`);
  });
}
```

With `UV_THREADPOOL_SIZE=2`, you will see two finish, then the next two finish much later (they waited for a free thread). With `=4`, all four finish at roughly the same time. This is the pool gating concurrency in front of your eyes.

> TIP: A reasonable rule for the pool is to size it near the number of CPU cores if your workload is fs/crypto/zlib heavy, but **measure**. Oversizing wastes memory and causes context-switch churn; undersizing serializes work that could be parallel.

### What the pool does NOT help with

The libuv pool offloads *specific libuv-implemented* operations. It does **nothing** for your own synchronous JavaScript. If you write a `for` loop that crunches numbers for 500ms, that runs on the **main JS thread** and blocks the event loop for 500ms — no requests are served, no timers fire, nothing. That is the CPU-bound problem, and solving it requires `worker_threads` or a separate process. We will return to this repeatedly.

---

## Part 2: `child_process` — Running Other Programs

The `child_process` module lets one Node process create and control **separate OS processes**. Each child has its own memory, its own V8 instance (if it is Node), and its own PID. This is the right tool when you want to:

- Run an external command-line program (`ffmpeg`, `git`, `python`, ImageMagick).
- Isolate risky or crash-prone work in its own process so it cannot take down the parent.
- Run a separate Node script and talk to it over a message channel.

There are four creation functions. Choosing the right one is a common interview question, so let's be precise.

### The four functions at a glance

| Function | Spawns | Shell? | Buffers output? | Returns | Best for |
| --- | --- | --- | --- | --- | --- |
| `spawn` | any command | No (by default) | No — streams it | `ChildProcess` (streams) | Long-running / large output, streaming |
| `exec` | any command | **Yes** (`/bin/sh`) | Yes — into memory | `ChildProcess` + callback | Quick commands, small output, shell features |
| `execFile` | a file directly | No | Yes — into memory | `ChildProcess` + callback | Running a binary without a shell |
| `fork` | a **Node** script | No | No | `ChildProcess` + IPC channel | Child Node processes with message passing |

### `spawn` — stream-oriented, no buffering

`spawn` launches a command and gives you back streams for `stdout`, `stderr`, and `stdin`. Output is **not** collected into memory; you consume it as it flows. This makes `spawn` the right choice for commands that produce large or continuous output.

```js
// spawn-demo.js
const { spawn } = require('node:child_process');

// Arguments are passed as an array — NOT interpolated into a shell string.
// This is safer (no shell injection) and avoids quoting headaches.
const child = spawn('ls', ['-lh', '/usr']);

// stdout/stderr are Readable streams.
child.stdout.on('data', (chunk) => {
  process.stdout.write(`stdout: ${chunk}`);
});

child.stderr.on('data', (chunk) => {
  process.stderr.write(`stderr: ${chunk}`);
});

// 'close' fires once the process has exited AND its stdio streams are closed.
child.on('close', (code) => {
  console.log(`child exited with code ${code}`);
});

// 'error' fires if the process could not be spawned at all (e.g. command not found).
child.on('error', (err) => {
  console.error('failed to start child process:', err);
});
```

> WARNING — Pitfall: There are two distinct failure events. `'error'` means Node could not even start the process (binary missing, permissions). A non-zero **exit code** on `'close'` means the process *ran* but reported failure. Handle both — beginners forget `'error'` and get unhandled exceptions.

#### `spawn` with `shell: true`

By default `spawn` does not use a shell, so features like pipes (`|`), globbing (`*`), and `&&` do not work. You can opt in with `{ shell: true }`, but then you reopen the shell-injection door:

```js
// Only do this with TRUSTED, non-user input.
const child = spawn('ls -lh /usr | grep bin', [], { shell: true });
```

> WARNING — Pitfall: Never build a shell command by concatenating untrusted input (`spawn(\`convert ${userFile} out.png\`, { shell: true })`). A filename like `; rm -rf /` becomes a real command. Prefer the array-args form with no shell, which passes arguments verbatim.

### `exec` — convenient, buffered, shell-backed

`exec` runs a command **through a shell** and buffers all stdout/stderr into memory, handing the complete result to a callback. It is convenient for short commands whose output is small.

```js
// exec-demo.js
const { exec } = require('node:child_process');

exec('node --version && npm --version', (error, stdout, stderr) => {
  if (error) {
    // 'error' here is non-null on non-zero exit OR spawn failure.
    console.error(`exec error: ${error.message}`);
    return;
  }
  console.log('stdout:\n' + stdout);
  if (stderr) console.error('stderr:\n' + stderr);
});
```

Because output is buffered, there is a `maxBuffer` limit (default 1 MB). Exceed it and the child is killed with an error.

```js
// Raise maxBuffer for commands with large output (value in bytes).
exec('cat big.log', { maxBuffer: 10 * 1024 * 1024 }, (err, stdout) => { /* ... */ });
```

> WARNING — Pitfall: `exec` buffering large output silently fails at 1 MB by default with an `ERR_CHILD_PROCESS_STDOUT_MAXBUFFER` error. For anything potentially large, use `spawn` and stream, or raise `maxBuffer` deliberately.

#### Promise-based `exec`

For modern async/await code, use the promisified version:

```js
const { promisify } = require('node:util');
const execAsync = promisify(require('node:child_process').exec);

async function gitBranch() {
  const { stdout } = await execAsync('git rev-parse --abbrev-ref HEAD');
  return stdout.trim();
}
```

### `execFile` — run a binary directly, no shell

`execFile` is like `exec` (buffered, callback) but it does **not** spawn a shell — it executes a file directly with an args array. This is both faster and safer than `exec` when you do not need shell features.

```js
// execFile-demo.js
const { execFile } = require('node:child_process');

// Runs the `node` binary directly; '--version' is an argument, not parsed by a shell.
execFile('node', ['--version'], (error, stdout) => {
  if (error) throw error;
  console.log(`Node version: ${stdout.trim()}`);
});
```

> BEST PRACTICE: Prefer `execFile` (or `spawn` with array args) over `exec` whenever you can. Avoiding the shell eliminates an entire class of injection vulnerabilities and is the default secure choice.

### `fork` — a specialized `spawn` for Node children with IPC

`fork` is a special case of `spawn` designed to launch **another Node.js script**. Its defining feature is that it automatically sets up an **IPC (inter-process communication) channel** between parent and child, so they can exchange structured messages with `child.send()` / `process.on('message')`.

> Note: `fork` here is *not* the Unix `fork()` syscall (which clones a process). Node's `fork` starts a fresh Node process running the script you name. The naming is a frequent source of confusion in interviews.

**parent.js**

```js
// parent.js
const { fork } = require('node:child_process');

// Launch worker.js as a separate Node process with an IPC channel.
const child = fork('./worker.js');

// Send a structured message (serialized automatically) to the child.
child.send({ type: 'sum', payload: [1, 2, 3, 4, 5] });

// Receive messages back from the child.
child.on('message', (msg) => {
  console.log('parent received:', msg);
  child.disconnect(); // close the IPC channel; child can then exit
});

child.on('exit', (code) => console.log(`worker exited with code ${code}`));
```

**worker.js**

```js
// worker.js
// In a forked child, process.send() and 'message' are available automatically.
process.on('message', (msg) => {
  if (msg.type === 'sum') {
    const total = msg.payload.reduce((a, b) => a + b, 0);
    process.send({ type: 'result', total });
  }
});
```

The messages are serialized for you. By default Node uses JSON-like serialization; you can opt into the **structured clone** algorithm (which supports more types, like `Map`, `Set`, `BigInt`, typed arrays) with `{ serialization: 'advanced' }`:

```js
const child = fork('./worker.js', [], { serialization: 'advanced' });
```

> WARNING — Pitfall: `fork`/`send` copy data by serialization. Sending a 200 MB object to a child means serializing and deserializing 200 MB. IPC is for *messages and coordination*, not for sharing large buffers cheaply. If you need shared memory, that is `worker_threads` + `SharedArrayBuffer` (Part 3), not `fork`.

### A practical streaming example: piping a child into the parent's stdout

```js
// stream-passthrough.js
const { spawn } = require('node:child_process');

// 'inherit' wires the child's stdio directly to the parent's — output appears live,
// with no JS-level data handling at all. Great for CLI tools.
const child = spawn('npm', ['install'], { stdio: 'inherit', shell: process.platform === 'win32' });

child.on('close', (code) => {
  console.log(`npm install finished with code ${code}`);
});
```

The `stdio` option is powerful: `'inherit'` shares the parent's streams; `'pipe'` (default) gives you stream objects; `'ignore'` discards; and you can pass an array to configure stdin/stdout/stderr independently, e.g. `stdio: ['ignore', 'pipe', 'inherit']`.

---

## Part 3: `worker_threads` — Real Threads for CPU Work

`child_process` creates whole new processes. That is heavy (each is a full V8 + Node runtime, ~tens of MB) and isolated (no shared memory). When you need **parallel CPU computation inside one application**, with the option of *shared memory* and cheaper creation, the right tool is `worker_threads`.

### The problem worker_threads solves

Recall: your JS runs on one thread. A CPU-bound computation blocks that thread, freezing the whole server. Consider this disaster of a route:

```js
// BAD: blocks the event loop for the duration of the loop.
app.get('/fib/:n', (req, res) => {
  const n = Number(req.params.n);
  function fib(k) { return k < 2 ? k : fib(k - 1) + fib(k - 2); }
  const result = fib(n); // for n=45 this takes seconds — ALL other requests wait
  res.json({ n, result });
});
```

While `fib(45)` churns, your health check times out, other users get no response, and your service looks dead. `worker_threads` lets you run that computation **on a separate thread**, keeping the event loop free.

### Threads vs. processes

| Aspect | `worker_threads` | `child_process` / `cluster` |
| --- | --- | --- |
| Unit | Thread (within one process) | Separate OS process |
| Memory | Can **share** (`SharedArrayBuffer`) | Isolated; copy via IPC |
| Startup cost | Lower (shares the process) | Higher (new runtime) |
| Crash isolation | A crashing worker can affect the process | Full isolation; one crash ≠ all crash |
| V8 instance | Separate per worker (own heap) | Separate per process |
| Best for | CPU-bound work, shared data | Scaling across cores, running other programs |

> Note: each Worker still gets its **own V8 isolate and event loop**. They do not share JS objects automatically — only `SharedArrayBuffer`-backed memory is truly shared. Everything else is copied or transferred.

### Creating a worker and passing initial data

**main.js**

```js
// main.js
const { Worker } = require('node:worker_threads');

function runFib(n) {
  return new Promise((resolve, reject) => {
    // Spawn a new thread running fib-worker.js.
    // workerData is the one-time initial payload, copied (structured clone) into the worker.
    const worker = new Worker('./fib-worker.js', { workerData: { n } });

    worker.on('message', resolve);          // result from the worker
    worker.on('error', reject);             // uncaught error inside the worker
    worker.on('exit', (code) => {
      if (code !== 0) reject(new Error(`Worker stopped with exit code ${code}`));
    });
  });
}

(async () => {
  console.log('main thread is responsive while this computes...');
  const result = await runFib(42);
  console.log('fib(42) =', result);
})();
```

**fib-worker.js**

```js
// fib-worker.js
const { workerData, parentPort } = require('node:worker_threads');

function fib(k) { return k < 2 ? k : fib(k - 1) + fib(k - 2); }

const result = fib(workerData.n); // heavy CPU work — but on THIS thread, not main
parentPort.postMessage(result);   // send the result back to the parent
```

The main event loop stays free the entire time the worker computes. That is the whole point.

### Two-way communication with `postMessage`

`workerData` is a one-shot initial payload. For ongoing back-and-forth, use the `MessagePort` exposed as `parentPort` (in the worker) and the `Worker` object (in the parent). Both are `EventEmitter`s with `postMessage` / `'message'`.

**main.js**

```js
const { Worker } = require('node:worker_threads');
const worker = new Worker('./echo-worker.js');

worker.on('message', (msg) => console.log('main got:', msg));

worker.postMessage({ task: 'square', value: 7 });
worker.postMessage({ task: 'square', value: 9 });
worker.postMessage({ task: 'exit' });
```

**echo-worker.js**

```js
const { parentPort } = require('node:worker_threads');

parentPort.on('message', (msg) => {
  if (msg.task === 'square') {
    parentPort.postMessage({ input: msg.value, result: msg.value ** 2 });
  } else if (msg.task === 'exit') {
    process.exit(0);
  }
});
```

### Transferring vs. copying: avoiding the serialization tax

By default `postMessage` **copies** data (via structured clone). For large `ArrayBuffer`s, copying is wasteful. You can instead **transfer** ownership: the buffer is moved (zero-copy) and becomes unusable in the sender.

```js
const big = new Uint8Array(50 * 1024 * 1024); // 50 MB
// The second argument is the transfer list: move `big.buffer` instead of copying it.
worker.postMessage(big, [big.buffer]);
// After transfer, `big` is detached in the sender — accessing it throws / is empty.
```

### Truly shared memory: `SharedArrayBuffer`

A `SharedArrayBuffer` is genuinely shared across threads — no copy, no transfer, both threads see the same bytes. This is the *only* way to share mutable memory in Node. Because two threads can now touch the same memory simultaneously, you need `Atomics` for coordination (locks, counters, waiting).

**main.js**

```js
// main.js
const { Worker } = require('node:worker_threads');

// Allocate shared memory: a 4-byte region viewed as one Int32.
const shared = new SharedArrayBuffer(4);
const counter = new Int32Array(shared);

const worker = new Worker('./counter-worker.js', { workerData: { shared } });

worker.on('exit', () => {
  // Both threads incremented the SAME memory — no message passing for the data itself.
  console.log('final counter value:', Atomics.load(counter, 0));
});

// Main thread also increments, atomically, to avoid lost updates.
for (let i = 0; i < 100000; i++) Atomics.add(counter, 0, 1);
```

**counter-worker.js**

```js
// counter-worker.js
const { workerData } = require('node:worker_threads');
const counter = new Int32Array(workerData.shared);

// Atomics.add is an atomic read-modify-write: safe across threads.
for (let i = 0; i < 100000; i++) Atomics.add(counter, 0, 1);
```

The final value is reliably 200000 because `Atomics.add` prevents the classic lost-update race. If you used plain `counter[0]++`, two threads could read the same value and both write back the same increment, losing counts.

> WARNING — Pitfall: `SharedArrayBuffer` only shares *binary* memory (numbers, bytes). You cannot share JS objects, strings, Maps, or class instances across threads — only `ArrayBuffer`-backed numeric data. Coordinate non-numeric state via messages.

> TIP: `Atomics.wait` / `Atomics.notify` let a worker block efficiently until a value changes — the building block for cross-thread signaling without busy-waiting. You generally only reach for these when building low-level primitives like a worker pool.

### Worker pools — the production pattern (and `piscina`)

Spawning a brand-new worker for every task is wasteful: thread creation has overhead, and unbounded workers will exhaust memory under load. The production pattern is a **worker pool**: a fixed set of long-lived workers that pull tasks from a queue. This caps concurrency and amortizes startup cost.

Here is a compact, working pool to demystify the concept:

```js
// pool.js — a minimal fixed-size worker pool
const { Worker } = require('node:worker_threads');
const os = require('node:os');
const EventEmitter = require('node:events');

class WorkerPool extends EventEmitter {
  constructor(workerFile, size = os.cpus().length) {
    super();
    this.workerFile = workerFile;
    this.idle = [];        // free workers
    this.queue = [];       // pending {task, resolve, reject}
    this.workers = new Map(); // worker -> current job

    for (let i = 0; i < size; i++) this._addWorker();
  }

  _addWorker() {
    const worker = new Worker(this.workerFile);

    worker.on('message', (result) => {
      const job = this.workers.get(worker);
      job.resolve(result);
      this.workers.set(worker, null);
      this.idle.push(worker);
      this._drain(); // pick up the next queued task
    });

    worker.on('error', (err) => {
      const job = this.workers.get(worker);
      if (job) job.reject(err);
      // Replace the dead worker so the pool keeps its size.
      this.workers.delete(worker);
      this._addWorker();
    });

    this.workers.set(worker, null);
    this.idle.push(worker);
  }

  _drain() {
    if (this.queue.length === 0 || this.idle.length === 0) return;
    const worker = this.idle.pop();
    const job = this.queue.shift();
    this.workers.set(worker, job);
    worker.postMessage(job.task);
  }

  run(task) {
    return new Promise((resolve, reject) => {
      this.queue.push({ task, resolve, reject });
      this._drain();
    });
  }
}

module.exports = WorkerPool;
```

```js
// use-pool.js
const WorkerPool = require('./pool');
const pool = new WorkerPool('./fib-pool-worker.js', 4);

(async () => {
  // 20 tasks, but only 4 run at once — the pool throttles concurrency.
  const jobs = Array.from({ length: 20 }, (_, i) => pool.run({ n: 35 + (i % 5) }));
  const results = await Promise.all(jobs);
  console.log(results);
})();
```

```js
// fib-pool-worker.js
const { parentPort } = require('node:worker_threads');
function fib(k) { return k < 2 ? k : fib(k - 1) + fib(k - 2); }
parentPort.on('message', ({ n }) => parentPort.postMessage(fib(n)));
```

In production, you usually do not hand-roll this. **`piscina`** is the de-facto worker pool library: it manages a pool, queues tasks, supports transfers and abort signals, resizes dynamically, and recycles workers. Conceptually it is exactly the pool above, hardened.

```js
// piscina-example.js  (npm i piscina)
const Piscina = require('piscina');
const path = require('node:path');

const pool = new Piscina({ filename: path.resolve(__dirname, 'fib-pool-worker.js') });

(async () => {
  // Piscina turns each call into a queued task on the pool.
  const result = await pool.run({ n: 40 });
  console.log(result);
})();
```

> BEST PRACTICE: Always **bound** your parallelism with a pool sized to roughly the number of CPU cores for CPU-bound work. Unbounded worker creation under load is a memory and scheduling disaster.

---

## Part 4: The `cluster` Module — One Process Per Core

`worker_threads` parallelize CPU work *within* one process. But a single Node process, no matter how many worker threads, still has one HTTP server accepting connections, and there is a ceiling to how much one process can push. On a 16-core machine, a single Node process leaves most cores idle for serving requests.

The **`cluster`** module solves this by forking **multiple Node processes (workers)** that all share the same listening port. Incoming connections are distributed across the workers, so you use all the cores for handling traffic.

### The primary/worker model

- The **primary** (formerly "master") process does not handle requests itself. It forks workers and supervises them (restarts on crash).
- Each **worker** is a full, separate Node process running your server code.
- They share a server port through magic in the cluster module (explained below).

### How do multiple processes share one port?

This is the favorite cluster interview question. Two mechanisms exist:

1. **Default on most platforms (round-robin):** The **primary** process creates the listening socket and accepts connections, then distributes them to workers in a round-robin fashion (with some load-awareness). Workers never call `listen()` on the real socket themselves; the cluster module intercepts `listen` and routes through the primary.
2. **`SO_REUSEPORT`-style / "none" scheduling:** The primary passes the socket handle to workers and lets the OS kernel decide which worker accepts each connection. This was the old default on some platforms and can produce uneven distribution. Round-robin (`cluster.SCHED_RR`) is the default everywhere except Windows.

You can set the policy explicitly:

```js
const cluster = require('node:cluster');
cluster.schedulingPolicy = cluster.SCHED_RR;   // round-robin (default on Linux/macOS)
// cluster.schedulingPolicy = cluster.SCHED_NONE; // let the OS decide
```

### A complete cluster server

```js
// cluster-server.js
const cluster = require('node:cluster');
const http = require('node:http');
const os = require('node:os');

const numCPUs = os.availableParallelism(); // modern API; falls back to os.cpus().length

if (cluster.isPrimary) {
  console.log(`Primary ${process.pid} starting ${numCPUs} workers`);

  // Fork one worker per core.
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  // Supervise: if a worker dies, replace it. This is your crash resilience.
  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died (${signal || code}). Restarting.`);
    cluster.fork();
  });
} else {
  // Worker processes share the SAME TCP port — the cluster module makes this work.
  http.createServer((req, res) => {
    // Show which worker handled the request to see load balancing.
    res.end(`Handled by worker ${process.pid}\n`);
  }).listen(3000);

  console.log(`Worker ${process.pid} listening on :3000`);
}
```

Run it, then hammer it (`while true; do curl -s localhost:3000; done`) and you will see different PIDs handling requests — that is the primary distributing connections.

> Note on terminology: `cluster.isPrimary`/`cluster.isWorker` are the current properties. The old `cluster.isMaster` is deprecated but still works. Use `isPrimary` in new code.

### Communicating with workers

The primary can talk to each worker over IPC (cluster is built on `fork`):

```js
if (cluster.isPrimary) {
  const worker = cluster.fork();
  worker.send({ cmd: 'configure', settings: { verbose: true } });
  worker.on('message', (msg) => console.log('from worker:', msg));
} else {
  process.on('message', (msg) => {
    if (msg.cmd === 'configure') process.send({ ack: true });
  });
}
```

> WARNING — Pitfall: Cluster workers do **not** share memory. Each has its own variables, its own caches, its own in-memory sessions. A counter incremented in one worker is invisible to the others. This is the single biggest gotcha when moving from one process to a cluster — and it forces the statelessness discipline we cover in Part 8.

### Graceful shutdown of workers

To avoid dropping in-flight requests on deploy, stop accepting new connections, let existing ones finish, then exit:

```js
// inside the worker branch
const server = http.createServer(handler).listen(3000);

process.on('SIGTERM', () => {
  console.log(`Worker ${process.pid} shutting down gracefully`);
  server.close(() => process.exit(0)); // stop accepting; exit when connections drain
  // Safety net: force-exit if connections hang.
  setTimeout(() => process.exit(1), 10000).unref();
});
```

---

## Part 5: Choosing Between `cluster`, `worker_threads`, and `child_process`

This decision is asked constantly and is genuinely important. Here is the framework.

| Goal | Use | Why |
| --- | --- | --- |
| Serve more HTTP traffic using all CPU cores | **`cluster`** (or PM2) | Multiple processes share the port; OS-level isolation per worker |
| Run a heavy CPU computation without blocking the event loop, possibly sharing memory | **`worker_threads`** | Cheaper than processes; `SharedArrayBuffer` enables zero-copy sharing |
| Run an external program (ffmpeg, git, python) | **`child_process`** (`spawn`/`execFile`) | That is literally what it is for |
| Run a separate Node task with message passing and full crash isolation | **`child_process.fork`** | Built-in IPC; isolated process protects the parent |
| Scale across **multiple machines** | Horizontal scaling + load balancer | None of these cross machine boundaries |

Mental shortcuts:

- **"I/O-bound and want more throughput on one box"** → `cluster` (replicate the whole server).
- **"CPU-bound work inside a request"** → `worker_threads` (offload the computation).
- **"I need to shell out to another program"** → `child_process`.
- **"I need many machines"** → that is not a Node module at all; it is infrastructure (Part 8).

> TIP: These are not mutually exclusive. A common production shape is **cluster (one process per core) + a worker-thread pool inside each process for CPU offload**, all behind a load balancer across **multiple machines**. Each layer solves a different bottleneck.

### Why not just use cluster for CPU work too?

You *can* offload CPU work by forking processes, and before `worker_threads` existed (pre-Node 10/12) that was common. But threads are lighter, start faster, and can share memory via `SharedArrayBuffer`, avoiding the serialization cost of IPC for large data. For pure in-app CPU computation, `worker_threads` is now the idiomatic choice. Use processes when you specifically want **isolation** (a crash or memory leak should not affect the parent) or when running **non-Node** code.

---

## Part 6: Handling CPU-Bound Tasks Without Blocking

This deserves its own section because it is where Node developers most often fail in production, and "what blocks the event loop?" is a top interview topic.

### Recognizing event-loop blocking

Anything synchronous and slow on the main thread blocks everything: tight loops, big `JSON.parse`/`JSON.stringify`, synchronous crypto (`pbkdf2Sync`), regex catastrophic backtracking, large `Array.sort`, template rendering, image processing in JS. While it runs, no requests are accepted, no callbacks fire, timers drift.

You can *measure* blocking with `perf_hooks`:

```js
// measure-lag.js
const { monitorEventLoopDelay } = require('node:perf_hooks');

const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();

setInterval(() => {
  // Mean/max event loop delay in milliseconds. Healthy is low single digits.
  console.log('event loop delay mean(ms):', (h.mean / 1e6).toFixed(2),
              'max(ms):', (h.max / 1e6).toFixed(2));
  h.reset();
}, 1000).unref();

// Simulate a blocking operation to watch the numbers spike:
setTimeout(() => {
  const end = Date.now() + 500;
  while (Date.now() < end) {} // blocks for 500ms
}, 2000);
```

### Strategy 1: Offload to a worker (preferred for real CPU work)

For genuine CPU-bound work, move it to a worker thread (Part 3) or a worker pool. This is the cleanest solution and keeps the event loop fully responsive.

```js
// express + worker pool: never block the event loop
const express = require('express');
const Piscina = require('piscina');
const path = require('node:path');

const app = express();
const pool = new Piscina({ filename: path.resolve(__dirname, 'fib-pool-worker.js') });

app.get('/fib/:n', async (req, res) => {
  try {
    // The heavy compute happens on a pool thread; this handler awaits without blocking.
    const result = await pool.run({ n: Number(req.params.n) });
    res.json({ n: Number(req.params.n), result });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.listen(3000);
```

### Strategy 2: Chunking / yielding (for work that can be split)

If the task can be broken into pieces, you can process a chunk, then yield to the event loop with `setImmediate`, then continue. This keeps the server responsive even though the work still runs on the main thread (just interleaved with other work). It is slower overall but simple and dependency-free.

```js
// chunked-sum.js — process a huge array without freezing the loop
function sumInChunks(arr, chunkSize = 10000) {
  return new Promise((resolve) => {
    let i = 0;
    let total = 0;
    function next() {
      const end = Math.min(i + chunkSize, arr.length);
      for (; i < end; i++) total += arr[i];
      if (i < arr.length) {
        // Yield to the event loop so other callbacks (requests, timers) can run.
        setImmediate(next);
      } else {
        resolve(total);
      }
    }
    next();
  });
}
```

> TIP: Use chunking when the work is naturally iterable and you want to avoid the complexity of threads. Use workers when the work is one indivisible heavy computation (parsing, hashing, image transforms) or when you want true parallelism across cores.

### Strategy 3: Use async/native variants

Many "CPU-ish" tasks have async, off-thread native implementations. Prefer `crypto.pbkdf2` (async, uses the pool) over `pbkdf2Sync`; prefer streaming `zlib` over sync. Native modules like `sharp` do image work off the JS thread. Reaching for the async/native variant is often the easiest fix.

> WARNING — Pitfall: `JSON.parse`/`JSON.stringify` on multi-megabyte payloads is a hidden event-loop blocker with no async core variant. For very large JSON, use a streaming parser, or offload the parse to a worker.

---

## Part 7: PM2 — Cluster Mode and Zero-Downtime Reloads

Writing your own `cluster` supervisor is educational, but in production most teams use a process manager. **PM2** is the most common. It runs your app in cluster mode (so you do not write cluster code yourself), restarts crashed processes, manages logs, and — crucially — does **zero-downtime reloads**.

### Running in cluster mode

```bash
npm install -g pm2

# Start app.js with one process per CPU core ("max"), in cluster mode.
pm2 start app.js -i max

# Or a fixed count:
pm2 start app.js -i 4

pm2 list          # see all processes
pm2 logs          # tail logs
pm2 monit         # live dashboard
```

> Note: With PM2 in cluster mode, your `app.js` should **not** contain its own `cluster` logic. PM2 *is* the cluster primary; it forks your app as workers. Just write a normal single-process server and let PM2 replicate it. (PM2 uses Node's cluster module under the hood, so the same shared-port mechanics apply.)

### The ecosystem config file

For anything non-trivial, use a config file instead of long CLI flags:

```js
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'api',
    script: './app.js',
    instances: 'max',        // one worker per core
    exec_mode: 'cluster',    // enable cluster mode (vs 'fork')
    max_memory_restart: '500M', // restart a worker if it exceeds 500MB (leak guard)
    env: { NODE_ENV: 'development' },
    env_production: { NODE_ENV: 'production' },
  }],
};
```

```bash
pm2 start ecosystem.config.js --env production
```

> WARNING — Pitfall: PM2's `exec_mode: 'fork'` runs a **single** process (no port sharing, no clustering); `'cluster'` is what gives you multi-core scaling. Setting `instances` without `exec_mode: 'cluster'` does not do what you expect. Mixing these up is a classic deployment bug.

### Zero-downtime reloads

A naive `pm2 restart` kills and restarts all workers at once — there is a window where no process is serving, dropping requests. `pm2 reload` does a **rolling restart**: it restarts workers **one at a time**, waiting for each new worker to come up (and optionally signal readiness) before killing the next old one. At every instant, some workers are serving.

```bash
pm2 reload api        # zero-downtime rolling reload
pm2 reload ecosystem.config.js --env production
```

For this to be truly zero-downtime, your app must shut down gracefully and tell PM2 when it is ready:

```js
// graceful + readiness signaling for PM2 reload
const server = require('http').createServer(handler).listen(3000, () => {
  // Tell PM2 this worker is ready to receive traffic (requires wait_ready: true).
  if (process.send) process.send('ready');
});

// On reload, PM2 sends SIGINT (configurable). Drain in-flight requests first.
process.on('SIGINT', () => {
  server.close(() => process.exit(0));
  setTimeout(() => process.exit(1), 8000).unref();
});
```

Enable readiness waiting in the config:

```js
// ecosystem.config.js (excerpt)
module.exports = {
  apps: [{
    name: 'api',
    script: './app.js',
    instances: 'max',
    exec_mode: 'cluster',
    wait_ready: true,       // wait for the process.send('ready') signal
    listen_timeout: 5000,   // how long to wait for 'ready' before assuming up
    kill_timeout: 8000,     // grace period before SIGKILL on shutdown
  }],
};
```

> BEST PRACTICE: Always pair zero-downtime reloads with **graceful shutdown** (drain connections) and **readiness signaling** (`wait_ready` + `process.send('ready')`). Without both, "zero-downtime" reload still drops requests during the swap.

### A note on `pm2 startup` and persistence

```bash
pm2 startup            # generate an OS init script so PM2 restarts on reboot
pm2 save               # snapshot the current process list to restore on boot
```

---

## Part 8: Horizontal Scaling, Statelessness, and Sticky Sessions

`cluster` and PM2 scale you across the cores of **one machine** (vertical-ish scaling). At some point one machine is not enough, and you scale **horizontally**: run your app on many machines behind a **load balancer** (nginx, HAProxy, an ALB, a Kubernetes Service). This is how real systems reach high scale, but it imposes a discipline: your app must become **stateless**.

### Why statelessness is mandatory

With a load balancer in front of N instances, consecutive requests from the same user may hit **different instances**. Any state you keep in a single process's memory is invisible to the others. Concretely, these break the moment you scale out:

- **In-memory sessions** (`express-session` default `MemoryStore`): user logs in on instance A, next request hits instance B, which has no session → logged out.
- **In-memory caches**: each instance has a different, inconsistent cache.
- **In-memory rate-limit counters**: limits are per-instance, so the real limit is N× what you intended.
- **Local file uploads on disk**: a later request to a different instance cannot find the file.
- **In-memory WebSocket connection registries**: instance A cannot push to a socket connected to instance B.

> BEST PRACTICE: A scalable Node service keeps **no important state in process memory**. Push state to shared, external stores: sessions and caches → **Redis**; uploads → **S3 / object storage**; data → your database. Treat each instance as disposable and interchangeable.

### Externalizing session state (the canonical fix)

```js
// stateless sessions backed by Redis — shared across all instances
const express = require('express');
const session = require('express-session');
const { RedisStore } = require('connect-redis'); // v7+ named export
const { createClient } = require('redis');

const redisClient = createClient({ url: process.env.REDIS_URL });
redisClient.connect();

const app = express();
app.use(session({
  store: new RedisStore({ client: redisClient }), // shared store, not MemoryStore
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
}));
```

Now any instance can read any user's session, so the load balancer can route freely and statelessness holds. The same pattern (use Redis/a DB instead of process memory) fixes caches and rate limiters.

### The WebSocket problem and sticky sessions

WebSockets complicate horizontal scaling in two ways:

1. **The handshake + long-lived connection.** A WebSocket starts as an HTTP request that upgrades and then stays open. Some setups — notably Socket.IO with HTTP long-polling fallback — make **multiple** HTTP requests during the handshake that must all reach the **same** instance, or the handshake fails. The fix is **sticky sessions**: configure the load balancer to consistently route a given client to the same instance (by IP hash or a cookie).

```nginx
# nginx sticky sessions via IP hash — same client IP → same upstream instance
upstream node_app {
    ip_hash;                 # route by client IP so WS handshake stays on one server
    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
}
server {
    location / {
        proxy_pass http://node_app;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;     # required for WebSocket upgrade
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
```

2. **Broadcasting across instances.** Even with sticky sessions, user X (connected to instance A) and user Y (connected to instance B) need to exchange real-time messages. Instance A cannot directly reach Y's socket on B. The solution is a **pub/sub backplane** (an "adapter"): instances publish events to Redis (or NATS/Kafka), and every instance delivers to its own local sockets. For Socket.IO this is the **Redis adapter**:

```js
// Socket.IO across multiple instances using the Redis adapter
const { Server } = require('socket.io');
const { createAdapter } = require('@socket.io/redis-adapter');
const { createClient } = require('redis');

const io = new Server(httpServer);

const pubClient = createClient({ url: process.env.REDIS_URL });
const subClient = pubClient.duplicate();

(async () => {
  await Promise.all([pubClient.connect(), subClient.connect()]);
  // Now io.emit() on ANY instance reaches sockets on ALL instances.
  io.adapter(createAdapter(pubClient, subClient));
})();
```

> TIP: Sticky sessions and a pub/sub adapter solve **different** problems. Sticky sessions keep one client pinned to one instance (needed for the handshake/long-polling). The adapter lets instances broadcast to each other (needed for cross-instance messaging). Real-time apps at scale usually need **both**.

> WARNING — Pitfall: Sticky sessions undermine even load distribution and resilience — if an instance dies, all its pinned clients drop and must reconnect. Prefer pure WebSocket transport (Socket.IO can skip long-polling) to *minimize* the need for stickiness, and never use stickiness as a substitute for externalizing real state.

### The cloud-native scaling picture

The full production stack typically layers all of this:

```text
                       Internet
                          │
                  ┌───────────────┐
                  │ Load Balancer │  (sticky for WS; round-robin for HTTP)
                  └───────────────┘
              ┌───────────┼───────────┐
        ┌─────────┐  ┌─────────┐  ┌─────────┐
        │Machine 1│  │Machine 2│  │Machine 3│   ← horizontal scaling
        │ PM2     │  │ PM2     │  │ PM2     │
        │ cluster │  │ cluster │  │ cluster │   ← one process per core
        │ [w][w]  │  │ [w][w]  │  │ [w][w]  │
        │ +pool   │  │ +pool   │  │ +pool   │   ← worker_threads for CPU work
        └─────────┘  └─────────┘  └─────────┘
              └───────────┼───────────┘
                  ┌───────────────┐
                  │ Redis / DB /  │  ← shared external state
                  │ S3 (state)    │
                  └───────────────┘
```

In container/Kubernetes setups, you often run **one Node process per container** and let the orchestrator handle replication (horizontal pod autoscaling) instead of using `cluster`/PM2 inside the container — the cluster layer "moves up" into the orchestrator. Either way, the statelessness discipline is identical.

---

## Common Pitfalls & Gotchas

- **Thinking all async is unlimited and parallel.** Only network I/O scales freely via the kernel. `fs`, `dns.lookup`, `crypto`, and `zlib` share the **4-thread libuv pool** by default; beyond that they queue.
- **Setting `UV_THREADPOOL_SIZE` too late.** It must be set before the pool initializes (before the first fs/crypto/dns/zlib call). Setting it mid-run has no effect.
- **Forgetting the `'error'` event on child processes and workers.** `'error'` (failed to start / uncaught) is separate from a non-zero exit code. Handle both or you get crashes.
- **Shell injection via `exec` / `spawn({shell:true})`.** Never interpolate untrusted input into a shell command. Use `execFile` or array-arg `spawn` with no shell.
- **`exec` `maxBuffer` overflow.** Large output silently fails at 1 MB by default; stream with `spawn` or raise `maxBuffer`.
- **Using IPC (`fork`/`send`) to move large data.** It serializes everything — expensive. For large shared data use `worker_threads` + `SharedArrayBuffer` (or transfer an `ArrayBuffer`).
- **Confusing `child_process.fork` with the Unix `fork()` syscall.** Node's `fork` starts a brand-new Node process running a script; it does not clone memory.
- **Assuming cluster workers share memory.** They do not. Each worker has its own variables, caches, and sessions. State must be externalized.
- **CPU work on the main thread.** A tight loop, big `JSON.parse`, or `pbkdf2Sync` blocks the entire event loop. Offload to a worker or chunk it.
- **Unbounded worker creation.** Spawning a worker per request exhausts memory. Use a fixed-size pool (or `piscina`).
- **PM2 `fork` vs `cluster` mode.** `exec_mode: 'fork'` is a single process; you need `'cluster'` for multi-core scaling.
- **`pm2 restart` instead of `pm2 reload`.** `restart` drops requests; `reload` does a rolling, zero-downtime swap — but only if your app shuts down gracefully and signals readiness.
- **In-memory sessions behind a load balancer.** The default `MemoryStore` breaks the instant you scale out. Use Redis (or another shared store).
- **WebSockets without sticky sessions and/or a pub/sub adapter.** Handshakes fail without stickiness; cross-instance broadcasts fail without an adapter. These are two different problems.

## Best Practices

- **Keep the event loop free.** Treat any synchronous operation over a few milliseconds as a bug. Measure with `perf_hooks` `monitorEventLoopDelay` and alert on high p99 loop delay.
- **Offload CPU-bound work to a bounded worker pool** (`piscina` or a hand-rolled pool sized ~= CPU cores). Never block the main thread; never spawn unbounded workers.
- **Prefer `execFile` / array-arg `spawn` over `exec`.** No shell means no shell injection; stream large output instead of buffering it.
- **Scale across cores with `cluster` or (more commonly) PM2 cluster mode**, and across machines with a load balancer. Let an orchestrator (Kubernetes) own replication when you containerize.
- **Be ruthlessly stateless.** Sessions, caches, rate limiters, uploads, and real-time registries all go to external stores (Redis, S3, DB). Treat instances as disposable.
- **Implement graceful shutdown everywhere.** On `SIGTERM`/`SIGINT`, stop accepting connections, drain in-flight requests, then exit — with a hard-timeout safety net.
- **Pair zero-downtime reloads with readiness signaling.** `wait_ready` + `process.send('ready')` so the LB/PM2 only routes to ready workers.
- **For real-time at scale, use both** sticky sessions (for the handshake) **and** a pub/sub backplane (for cross-instance broadcast); minimize stickiness by preferring pure WebSocket transport.
- **Guard against leaks in long-lived processes** with `max_memory_restart` and good observability, since a single worker can otherwise grow unbounded.
- **Size `UV_THREADPOOL_SIZE` deliberately** for fs/crypto/zlib-heavy workloads, set early, and verify by measuring.

## Key Interview Points

- **"Is Node single-threaded?"** — The JavaScript executes on a single thread (one event loop), but Node uses the OS kernel for network I/O and a libuv thread pool (default 4 threads) for fs/dns/crypto/zlib. So the *runtime* is multi-threaded; *your JS* is not.
- **"What is `UV_THREADPOOL_SIZE`?"** — The size of the libuv background thread pool (default 4, max 1024). It gates concurrency of fs/crypto/zlib/`dns.lookup`. Set it via env var before the process starts.
- **"`spawn` vs `exec` vs `execFile` vs `fork`?"** — `spawn`: streams output, no shell, best for large/long output. `exec`: buffers output, runs through a shell, best for small commands. `execFile`: buffers, no shell, runs a binary directly (safer than `exec`). `fork`: spawns a Node script with a built-in IPC channel.
- **"`worker_threads` vs `cluster` vs `child_process`?"** — `worker_threads`: parallel CPU work in one process, can share memory (`SharedArrayBuffer`). `cluster`: multiple processes sharing a port to use all cores for traffic. `child_process`: run separate programs / isolated Node tasks with IPC.
- **"How do cluster workers share a port?"** — The primary creates the listening socket and distributes connections (round-robin by default on Linux/macOS); workers don't actually own the socket. Alternatively the OS decides (`SCHED_NONE`).
- **"Do cluster workers share memory?"** — No. Each is a separate process with its own heap. State must be externalized (Redis/DB).
- **"How do you handle CPU-bound work without blocking?"** — Offload to a worker thread / pool (`piscina`), chunk the work and yield with `setImmediate`, or use async native variants. Measure loop delay with `perf_hooks`.
- **"`SharedArrayBuffer` and `Atomics` — why?"** — `SharedArrayBuffer` is the only truly shared memory across threads; `Atomics` provides safe, race-free read-modify-write and cross-thread signaling (`wait`/`notify`).
- **"What is zero-downtime reload?"** — A rolling restart (`pm2 reload`) that replaces workers one at a time, each new one coming up (and signaling ready) before the next old one is killed — so traffic is always served. Requires graceful shutdown.
- **"What does it mean for a service to be stateless, and why does scaling require it?"** — No important state in process memory; behind a load balancer requests may hit any instance, so sessions/caches/state must live in shared external stores. Enables free routing and disposable instances.
- **"Why do WebSockets need sticky sessions?"** — The handshake (and any long-polling fallback) must reach the same instance; without stickiness it fails. Sticky sessions pin a client to one instance; a Redis pub/sub adapter then lets instances broadcast to each other.
- **"`fork` (child_process) vs Unix `fork()`?"** — Node's `fork` launches a new Node process running a given script with an IPC channel; it does not clone the parent's memory like the syscall does.

## Practice Exercises

Work these in order — each builds on the last. Try them before reading the solutions.

1. **Pool-size proof.** Write a script that runs 8 async `crypto.pbkdf2` operations and logs when each finishes with a timestamp. Run it with `UV_THREADPOOL_SIZE=2`, then `=4`, then `=8`, and explain the timing pattern you see.

2. **Stream a child process.** Use `spawn` to run a command that produces continuous output (e.g. `ping` a few times, or `node -e` printing in a loop) and print each line as it arrives, prefixed with `[child]`. Handle both the `'error'` event and a non-zero exit code distinctly.

3. **Non-blocking CPU endpoint.** Build an Express server with two routes: `GET /block/:n` that computes `fib(n)` synchronously on the main thread, and `GET /work/:n` that computes the same in a worker thread. While a slow request to each runs, hit `GET /health` (returns `ok`) and observe which version keeps `/health` responsive.

4. **Mini worker pool.** Without using `piscina`, implement a fixed-size worker pool (say 4 workers) that processes an array of 20 CPU tasks, ensuring only 4 run concurrently and all results return in order of submission.

5. **Cluster with graceful restart.** Write a `cluster` HTTP server (one worker per core) where each response includes the handling worker's PID. Add a route `GET /crash` that throws, and verify the primary restarts the dead worker. Implement graceful `SIGTERM` shutdown in the workers.

6. **Stateless sessions (design + code sketch).** Take a single-instance Express app using the default in-memory session store and convert it to be cluster/horizontal-scaling safe. Explain in one paragraph what breaks under scale-out and show the minimal code change that fixes it.

### Solutions

#### Solution 1 — Pool-size proof

```js
// sol1-pool.js
// Run: UV_THREADPOOL_SIZE=2 node sol1-pool.js   (then 4, then 8)
const crypto = require('node:crypto');
const start = Date.now();

for (let i = 1; i <= 8; i++) {
  crypto.pbkdf2('pw', 'salt', 200000, 64, 'sha512', () => {
    console.log(`#${i} done at ${Date.now() - start}ms`);
  });
}
```

**Explanation:** Each `pbkdf2` occupies one libuv pool thread for its duration. With `=2`, you see operations complete in **waves of 2** — two finish, then the next two (which had to wait for a free thread), and so on. With `=4`, you get waves of 4. With `=8`, all eight run concurrently and finish at roughly the same time. This proves the pool size, not the number of calls, gates true concurrency for pool-backed operations.

#### Solution 2 — Stream a child process

```js
// sol2-stream.js
const { spawn } = require('node:child_process');
const readline = require('node:readline');

// A portable command that emits a few lines over time.
const child = spawn('node', ['-e',
  'let i=0;const t=setInterval(()=>{console.log("tick "+(++i));if(i>=5){clearInterval(t)}} ,300)']);

// Read stdout line-by-line as it streams.
readline.createInterface({ input: child.stdout }).on('line', (line) => {
  console.log(`[child] ${line}`);
});

child.stderr.on('data', (d) => console.error(`[child:err] ${d}`));

// Distinct handling: 'error' = could not start; exit code = ran but maybe failed.
child.on('error', (err) => console.error('FAILED TO START:', err.message));
child.on('close', (code) => {
  console.log(code === 0 ? '[child] finished OK' : `[child] exited with code ${code}`);
});
```

**Key point:** `spawn` streams, so lines print as they arrive (not all at the end like `exec`). The `'error'` event and the exit `code` are handled separately — the crux of robust child-process code.

#### Solution 3 — Non-blocking CPU endpoint

```js
// sol3-server.js
const express = require('express');
const { Worker } = require('node:worker_threads');
const app = express();

function fib(k) { return k < 2 ? k : fib(k - 1) + fib(k - 2); }

// BLOCKING route — freezes the event loop, so /health stalls during this.
app.get('/block/:n', (req, res) => {
  const result = fib(Number(req.params.n));
  res.json({ result });
});

// NON-BLOCKING route — offloads to a worker; /health stays responsive.
app.get('/work/:n', (req, res) => {
  const worker = new Worker('./sol3-worker.js', { workerData: { n: Number(req.params.n) } });
  worker.on('message', (result) => res.json({ result }));
  worker.on('error', (err) => res.status(500).json({ error: err.message }));
});

app.get('/health', (_req, res) => res.send('ok'));
app.listen(3000, () => console.log('http://localhost:3000'));
```

```js
// sol3-worker.js
const { workerData, parentPort } = require('node:worker_threads');
function fib(k) { return k < 2 ? k : fib(k - 1) + fib(k - 2); }
parentPort.postMessage(fib(workerData.n));
```

**Observation:** Request `GET /block/45` and simultaneously `GET /health` — health hangs until the fib finishes (event loop blocked). Do the same with `GET /work/45` — health responds instantly because the computation runs on another thread. (For a production version, replace per-request workers with a pool from Solution 4.)

#### Solution 4 — Mini worker pool

Use the `WorkerPool` class from Part 3 with the `fib-pool-worker.js` worker. The driver:

```js
// sol4-driver.js
const WorkerPool = require('./pool');       // the class from Part 3
const pool = new WorkerPool('./fib-pool-worker.js', 4);

(async () => {
  const inputs = Array.from({ length: 20 }, (_, i) => ({ n: 34 + (i % 6) }));
  // Promise.all preserves order: results[i] corresponds to inputs[i],
  // even though only 4 tasks run concurrently inside the pool.
  const results = await Promise.all(inputs.map((task) => pool.run(task)));
  console.log(results);
})();
```

**Why it satisfies the requirements:** The pool holds exactly 4 workers; `run()` queues tasks and `_drain()` only dispatches to idle workers, so concurrency is capped at 4. `Promise.all` returns results positionally, preserving submission order regardless of completion order. The `'error'` handler replaces a dead worker so the pool self-heals.

#### Solution 5 — Cluster with graceful restart

```js
// sol5-cluster.js
const cluster = require('node:cluster');
const http = require('node:http');
const os = require('node:os');

if (cluster.isPrimary) {
  const n = os.availableParallelism();
  for (let i = 0; i < n; i++) cluster.fork();

  // Supervisor: replace any worker that dies (including from /crash).
  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died (${signal || code}); restarting.`);
    cluster.fork();
  });
} else {
  const server = http.createServer((req, res) => {
    if (req.url === '/crash') {
      // Throw asynchronously so it crashes the worker process.
      setImmediate(() => { throw new Error('boom'); });
      return;
    }
    res.end(`Handled by worker ${process.pid}\n`);
  }).listen(3000, () => console.log(`Worker ${process.pid} up`));

  // Graceful shutdown: stop accepting, drain, exit (with a hard timeout).
  process.on('SIGTERM', () => {
    server.close(() => process.exit(0));
    setTimeout(() => process.exit(1), 10000).unref();
  });
}
```

**Verification:** `curl localhost:3000` shows rotating PIDs (round-robin). `curl localhost:3000/crash` kills a worker; the primary's `'exit'` handler logs the death and forks a replacement, so total worker count stays constant and the service keeps serving on the surviving workers. `kill -TERM <worker-pid>` drains rather than dropping connections.

#### Solution 6 — Stateless sessions

**What breaks:** With the default `MemoryStore`, each cluster worker / machine has its own in-memory session map. After login on worker A, the next request may be routed to worker B, which has no record of the session — the user appears logged out, intermittently and confusingly. `MemoryStore` is explicitly not for production/multi-instance use.

**Minimal fix — swap the store for a shared Redis-backed one:**

```js
// sol6-sessions.js
const express = require('express');
const session = require('express-session');
const { RedisStore } = require('connect-redis');
const { createClient } = require('redis');

const redisClient = createClient({ url: process.env.REDIS_URL || 'redis://localhost:6379' });
redisClient.connect();

const app = express();
app.use(session({
  store: new RedisStore({ client: redisClient }), // <-- the only real change
  secret: process.env.SESSION_SECRET || 'dev-secret',
  resave: false,
  saveUninitialized: false,
  cookie: { httpOnly: true, secure: process.env.NODE_ENV === 'production' },
}));

app.get('/login', (req, res) => { req.session.user = 'alice'; res.send('logged in'); });
app.get('/me', (req, res) => res.send(req.session.user || 'anonymous'));

app.listen(3000);
```

Now every instance reads/writes the same session data in Redis, so any worker or machine can serve any request. The app becomes stateless with respect to sessions, which is the prerequisite for safe horizontal scaling. The identical principle applies to caches and rate limiters.

---

## Summary / Cheat Sheet

| Concept | Key takeaway |
| --- | --- |
| JS thread | One per process; your JS never runs two statements in parallel |
| Network I/O | Handled by the OS kernel via the event loop; scales without threads |
| libuv pool | Default **4** threads for fs / `dns.lookup` / crypto / zlib; tune with `UV_THREADPOOL_SIZE` |
| `spawn` | Streams output, no shell — large/long output |
| `exec` | Buffers output (1 MB default), runs a shell — small commands |
| `execFile` | Buffers, no shell, runs a binary — safer than `exec` |
| `fork` | Spawns a Node script + automatic IPC (`send`/`message`) |
| `worker_threads` | Real threads for **CPU-bound** work; own V8/event loop per worker |
| `workerData` | One-shot initial data copied into a worker |
| `postMessage` | Two-way messaging; copies (structured clone) or transfers (`[buffer]`) |
| `SharedArrayBuffer` | Only truly shared memory across threads; use `Atomics` for safety |
| Worker pool | Fixed-size, reusable workers; **`piscina`** in production |
| `cluster` | One process per core sharing a port; primary supervises & restarts |
| Port sharing | Primary distributes connections (round-robin default) |
| Cluster memory | **Not shared** — externalize all state |
| CPU-bound fix | Offload to worker/pool, or chunk with `setImmediate`, or async native |
| `perf_hooks` | `monitorEventLoopDelay` to measure event-loop lag |
| PM2 | `-i max`, `exec_mode: 'cluster'`; `pm2 reload` for zero-downtime |
| Zero-downtime | Rolling restart + graceful shutdown + `wait_ready`/`process.send('ready')` |
| Statelessness | No state in process memory; use Redis/S3/DB |
| Sticky sessions | Pin a client to one instance (WS handshake/long-polling) |
| Pub/sub adapter | Cross-instance broadcast for WebSockets (e.g. Socket.IO Redis adapter) |

**Decision in one line:** *More HTTP throughput on one box → `cluster`/PM2. CPU work without blocking → `worker_threads` pool. Run another program → `child_process`. More machines → load balancer + stateless app (+ sticky sessions and a pub/sub backplane for WebSockets).*
