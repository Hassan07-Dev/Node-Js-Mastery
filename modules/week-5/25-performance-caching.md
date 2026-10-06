# Performance, Caching & Optimization

You have spent four weeks learning to *build* things in Node and Express. This module is about making those things **fast, cheap, and survivable under load** — the difference between an app that demos well on your laptop and one that serves ten thousand concurrent users on a single modest box without falling over.

Performance work in Node is unusual because Node's runtime is unusual. A Node process is, at its core, **a single thread running a single event loop**. That one architectural fact dictates almost everything that follows: why a single slow function can freeze your *entire* server, why caching matters more here than in a thread-per-request stack, why memory leaks are so insidious, and why the most important optimization you will ever make is *not doing work twice*.

This module is deliberately practical. Every technique comes with the *why*, the *how to measure it* (because optimizing without measuring is just guessing), and runnable code. We move from the foundational rule (never block the loop), through profiling and memory leaks, into the highest-leverage win in web engineering (caching), then HTTP-level optimizations, database performance, pagination, and finally load testing so you can *prove* your changes worked.

> BEST PRACTICE: The cardinal rule of performance work is **measure first, optimize second, measure again**. Intuition about what is slow is wrong more often than it is right. Every section here pairs a technique with a way to measure its effect.

## What You'll Learn

By the end of this module you will be able to:

- Explain **why blocking the event loop is catastrophic** in Node, and detect blocking with `perf_hooks`, `--prof`, and `blocked-at`.
- **Profile a running Node process** using `node --prof`, the V8 inspector / Chrome DevTools, `clinic.js`, and flamegraphs — and read what they tell you.
- **Find and fix memory leaks** using heap snapshots and `--inspect`, and recognize the classic leak sources: globals, closures, timers, and un-removed event listeners.
- Choose the right **caching strategy and layer** — in-process vs. distributed — and reason about cache invalidation.
- Use **Redis** as a cache from Node (`ioredis` / `node-redis`): the **cache-aside** pattern, **TTLs**, and invalidation strategies.
- Apply **HTTP caching** correctly: `Cache-Control`, `ETag`, and conditional requests (`304 Not Modified`).
- Add **response compression** (gzip/Brotli) and know when *not* to.
- Tune **database performance**: indexes, `lean()`, projection, avoiding N+1 with `populate`, and connection pooling.
- Implement **pagination** for large datasets and explain **offset vs. cursor** trade-offs.
- **Load test** with `autocannon` and read latency percentiles, throughput, and errors correctly.

> Note: Examples target Node.js current LTS (22.x) and Express (v5 is now the default `express` on npm; differences are flagged where relevant). MongoDB examples use Mongoose. Code uses CommonJS `require` by default; ES Module variants are shown where it matters.

---

## 1. The Golden Rule: Never Block the Event Loop

### Why this is the rule above all rules

Recall the architecture from Week 1: Node runs your JavaScript on **one thread**. Asynchronous I/O (file reads, network calls, DB queries) is handed off to libuv's thread pool or the OS and does *not* block that thread — that is the whole point of Node. But **any synchronous CPU work runs on the main thread**, and while it runs, **nothing else can happen**: no new connections accepted, no responses sent, no timers fired, no callbacks invoked.

In a thread-per-request server (classic PHP/Apache, Java servlets), one slow request ties up *one* thread; other requests get other threads. In Node, **one slow synchronous operation freezes the server for every connected client simultaneously.** This is the single most important mental model in Node performance.

Consider:

```js
// server-blocking.js — a deliberately broken endpoint
const express = require('express');
const app = express();

// A CPU-bound, SYNCHRONOUS handler. While this loops, the ENTIRE
// process is frozen — every other request waits.
app.get('/slow', (req, res) => {
  const start = Date.now();
  // Busy-wait: pure CPU, no yielding to the event loop.
  while (Date.now() - start < 5000) {
    // burn 5 seconds of CPU
  }
  res.send('done after blocking for 5s');
});

app.get('/fast', (req, res) => res.send('I should be instant'));

app.listen(3000, () => console.log('http://localhost:3000'));
```

Hit `/slow` in one tab, then immediately hit `/fast` in another. The "instant" endpoint will hang until `/slow` finishes. You did not make `/fast` slow — you froze the loop, and `/fast` simply could not be processed until the loop was free again.

### What "blocking" actually means

Anything that does meaningful CPU work *synchronously* before returning control to the event loop is blocking. Common offenders:

| Blocking operation | Non-blocking alternative |
| --- | --- |
| `fs.readFileSync(path)` | `fs.promises.readFile(path)` / `fs.readFile` |
| `JSON.parse` on a huge payload | Stream-parse, or offload to a worker |
| `crypto.pbkdf2Sync` / `bcrypt.hashSync` | `crypto.pbkdf2` / `bcrypt.hash` (async versions) |
| Synchronous `zlib.gzipSync` on big data | `zlib.gzip` (async) |
| Giant `for` loop / heavy `JSON.stringify` | Chunk the work, or use a Worker thread |
| `Array.prototype.sort` on a 5M-element array | Paginate / index in the DB instead |
| Catastrophic regex backtracking (ReDoS) | Fix the regex; validate input length |

> WARNING — Pitfall: The `*Sync` family of APIs (`readFileSync`, `existsSync`, `execSync`, `hashSync`) is fine at **startup** (loading config once before `listen`) but is a landmine **inside request handlers**. A `readFileSync` in a hot path is one of the most common Node performance bugs in real codebases.

### Detecting blocking #1: measure event-loop lag with `perf_hooks`

The cleanest built-in way to quantify how "stuck" your loop is uses `monitorEventLoopDelay` from `perf_hooks`. It samples how late the loop's timers fire relative to when they *should* — that lateness is your blocking signal.

```js
// loop-lag.js
const { monitorEventLoopDelay } = require('node:perf_hooks');

// Sample loop delay every 20ms.
const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();

setInterval(() => {
  // All values are in NANOSECONDS — divide by 1e6 for ms.
  console.log(
    `loop delay  mean=${(h.mean / 1e6).toFixed(2)}ms` +
    `  p99=${(h.percentile(99) / 1e6).toFixed(2)}ms` +
    `  max=${(h.max / 1e6).toFixed(2)}ms`
  );
  h.reset();
}, 1000);

// Simulate periodic blocking so you can watch the numbers spike.
setInterval(() => {
  const start = Date.now();
  while (Date.now() - start < 200) {} // block 200ms
}, 3000);
```

On a healthy event loop, `mean` delay should be sub-millisecond and `p99` in the low single-digit milliseconds. If you see `p99` of tens or hundreds of milliseconds, something is blocking. This metric is so valuable that you should **export it to your monitoring system in production** (Prometheus, Datadog, etc.) — rising loop lag is a leading indicator of trouble before users even notice.

### Detecting blocking #2: `blocked-at` to find *where*

Loop lag tells you *that* you are blocking; the `blocked-at` package tells you *where* by capturing a stack trace each time the loop is blocked beyond a threshold.

```bash
npm install blocked-at
```

```js
// Put this at the very top of your entry file, BEFORE other requires.
const blocked = require('blocked-at');

blocked((time, stack) => {
  // `time` = how long (ms) the loop was blocked; `stack` = where it happened.
  console.warn(`Event loop blocked for ${time.toFixed(1)}ms, at:\n${stack.join('\n')}`);
}, { threshold: 100 }); // only report blocks longer than 100ms
```

> TIP: `blocked-at` adds overhead (it instruments async hooks) so use it in development or staging to hunt a specific problem, not as an always-on production agent. For production, prefer the lightweight `monitorEventLoopDelay` metric.

### Fixing blocking: offload to a Worker thread

When you genuinely have CPU-bound work (image processing, cryptography, big aggregations you cannot push to the DB), move it off the main thread with the built-in `worker_threads` module. Workers run real JS in parallel on separate threads with their own event loop and memory.

```js
// fib-worker.js — the worker: runs in its own thread
const { parentPort, workerData } = require('node:worker_threads');

function fib(n) {
  return n < 2 ? n : fib(n - 1) + fib(n - 2); // intentionally CPU-heavy
}

parentPort.postMessage(fib(workerData.n));
```

```js
// server-with-worker.js — the main thread stays responsive
const express = require('express');
const { Worker } = require('node:worker_threads');
const app = express();

function runFib(n) {
  return new Promise((resolve, reject) => {
    const worker = new Worker('./fib-worker.js', { workerData: { n } });
    worker.once('message', resolve);
    worker.once('error', reject);
    worker.once('exit', (code) => {
      if (code !== 0) reject(new Error(`Worker stopped with code ${code}`));
    });
  });
}

app.get('/fib/:n', async (req, res, next) => {
  try {
    // Heavy work happens on a separate thread; the loop stays free.
    const result = await runFib(Number(req.params.n));
    res.json({ result });
  } catch (err) {
    next(err);
  }
});

app.get('/fast', (req, res) => res.send('still instant, even during /fib/45'));

app.listen(3000);
```

> BEST PRACTICE: For a real app, do not spawn a fresh `Worker` per request (creation has cost). Use a **worker pool** — the `piscina` library is the de-facto standard and gives you a clean `pool.run(data)` Promise API over a reusable pool of threads.

For workloads that are *both* CPU-heavy *and* high-throughput, the other lever is **horizontal scaling**: run multiple Node processes (via the `cluster` module or a process manager like PM2, or simply multiple containers behind a load balancer) so blocking in one process does not affect requests routed to another. Workers and clustering are complementary, not alternatives.

---

## 2. Profiling Node: Finding the Hot Path

You cannot optimize what you cannot see. Profiling answers the question *"where is my CPU time actually going?"* — and the answer is almost always surprising.

### 2.1 The built-in V8 profiler: `node --prof`

V8 ships with a sampling profiler. It periodically records the call stack and tallies which functions are "on top" most often — those are your hot functions.

```bash
# 1. Run your app with the profiler. It writes an isolate-*.log file.
node --prof server.js

# 2. Generate load against it (see the autocannon section), then stop the app (Ctrl-C).

# 3. The raw log is unreadable. Process it into a human report:
node --prof-process isolate-0xNNNNNNN-v8.log > processed.txt
```

The processed report has several sections. The most important is the **[Summary]** (ticks split across JavaScript / C++ / GC) and the **[Bottom up (heavy) profile]**, which lists the functions where the most time was spent, with their callers.

```
 [Summary]:
   ticks  total  nonlib   name
   1842   61.4%   72.1%  JavaScript
    402   13.4%   15.7%  C++
    188    6.3%    7.3%  GC          <- high GC% hints at allocation churn / leaks
```

> TIP: A high **GC** percentage in the summary is a strong hint that you are allocating too many objects (creating garbage) — often a sign you should reuse buffers/objects or that you have a leak (see §3). A high **JavaScript** percentage with one function dominating the bottom-up profile is your hot path to optimize.

### 2.2 The inspector + Chrome DevTools (the friendly way)

The raw `--prof` log is powerful but spartan. For an interactive, visual experience, use Node's built-in **inspector**, which speaks the Chrome DevTools Protocol.

```bash
# Start your app with the inspector listening.
node --inspect server.js
# For breaking before the first line (useful to profile startup):
node --inspect-brk server.js
```

Then open `chrome://inspect` in Chrome, click **"Open dedicated DevTools for Node"**, and use the **Performance** tab (record a CPU profile while you generate load) or the **Memory** tab (heap snapshots — see §3). The flamechart it produces is far easier to read than text.

> WARNING — Pitfall: `--inspect` opens a debugging port (default `127.0.0.1:9229`). **Never expose it on a public interface in production** (`--inspect=0.0.0.0:9229` is dangerous) — anyone who can reach that port can execute arbitrary code in your process.

You can also capture a CPU profile programmatically with the `inspector` module and save a `.cpuprofile` file you can later load into DevTools — useful for profiling in environments where you cannot attach a browser:

```js
// programmatic-cpu-profile.js
const inspector = require('node:inspector');
const fs = require('node:fs');
const session = new inspector.Session();
session.connect();

session.post('Profiler.enable', () => {
  session.post('Profiler.start', () => {
    // ... run the workload you want to profile here ...
    setTimeout(() => {
      session.post('Profiler.stop', (err, { profile }) => {
        // Drag this .cpuprofile into Chrome DevTools > Performance to inspect.
        fs.writeFileSync('./cpu.cpuprofile', JSON.stringify(profile));
        session.disconnect();
      });
    }, 10000); // profile for 10 seconds
  });
});
```

### 2.3 clinic.js: opinionated, automated diagnostics

[`clinic`](https://clinicjs.org/) (by NearForm) wraps these primitives into three tools that produce beautiful HTML reports and even *suggest* what is wrong.

```bash
npm install -g clinic
# or run ad-hoc with npx

# Doctor: the first thing to run. Detects WHICH category of problem you have
# (event-loop blocking, GC/memory pressure, I/O, or CPU) and tells you.
clinic doctor --on-port 'autocannon localhost:$PORT' -- node server.js

# Flame: produces a flamegraph of CPU time (best for "what's slow?").
clinic flame --on-port 'autocannon localhost:$PORT' -- node server.js

# Bubbleprof: visualizes async operations and where time is spent waiting.
clinic bubbleprof --on-port 'autocannon localhost:$PORT' -- node server.js
```

The `--on-port` flag is a lovely convenience: clinic boots your server, waits until it is listening, runs the load command (with `$PORT` substituted), then stops everything and opens the report. **Start with `clinic doctor`** — it triages the *kind* of problem so you know which other tool to reach for next.

### 2.4 Flamegraphs: how to read them

A **flamegraph** visualizes stack samples. Read it like this:

- The **x-axis is NOT time** — it is the population of samples, sorted alphabetically. **Width = total time spent in that function (including its children).** Wider = more CPU.
- The **y-axis is stack depth.** A box sitting on top of another means "was called by." The bottom is the entry point; the top boxes are the functions actually executing when sampled.
- **Look for wide plateaus near the top.** A wide box high up means a function (or its inlined work) is *itself* burning CPU — that is your optimization target.

You generate them with `clinic flame`, or with `node --prof` + a converter, or the classic `0x` tool:

```bash
npx 0x server.js   # generates an interactive flamegraph HTML when you stop the process
```

> TIP: When reading a flamegraph, ignore narrow spikes (they are noise) and hunt for the *widest* boxes at the *top* of the stack. That combination — wide and high — is "lots of time, in this function's own code." That is where optimization pays off.

---

## 3. Finding and Fixing Memory Leaks

### What a leak is in a garbage-collected language

You do not `free()` in JavaScript — V8's garbage collector reclaims objects that are no longer **reachable** from a "root" (globals, the current call stack, etc.). A **memory leak** in Node is therefore not "forgot to free"; it is **keeping a reference to something you no longer need**, so the GC *cannot* collect it. Memory grows, GC works harder (raising CPU and pause times), and eventually the process hits its heap limit and crashes with:

```
FATAL ERROR: Reached heap limit Allocation failed - JavaScript heap out of memory
```

### Recognizing a leak: the sawtooth that climbs

Healthy memory under load looks like a **sawtooth**: it rises as you allocate, then drops sharply when GC runs — and the *bottoms* of the teeth stay roughly flat. A leak looks like a sawtooth whose **bottoms creep steadily upward** over time. The fast way to watch this:

```js
// mem-monitor.js — log heap usage every 5s
setInterval(() => {
  const { heapUsed, heapTotal, rss, external } = process.memoryUsage();
  const mb = (n) => (n / 1024 / 1024).toFixed(1);
  console.log(
    `rss=${mb(rss)}MB heapUsed=${mb(heapUsed)}MB ` +
    `heapTotal=${mb(heapTotal)}MB external=${mb(external)}MB`
  );
}, 5000);
```

- **`heapUsed`** — live JS objects. A continuously rising `heapUsed` *after* GC is the classic leak signal.
- **`rss`** (Resident Set Size) — total memory the process holds from the OS (heap + stacks + code + Buffers). Rising `rss` with flat `heapUsed` often means a **native/Buffer leak** (`external` memory).
- Force-trigger GC to see the true baseline: run with `node --expose-gc` and call `global.gc()` before sampling.

### The classic leak sources

#### 1. Accidental globals and ever-growing module-level state

```js
// LEAK: a module-level cache with no eviction grows forever.
const cache = {};
app.get('/user/:id', async (req, res) => {
  if (!cache[req.params.id]) {
    cache[req.params.id] = await db.findUser(req.params.id);
  }
  res.json(cache[req.params.id]); // unbounded: every unique id stays forever
});
```

The fix is a **bounded** cache with eviction (an LRU). Never use a plain object or `Map` as a long-lived cache without a size limit or TTL. (`lru-cache` is the standard library; see §4.)

#### 2. Closures that capture more than they need

```js
// LEAK: each call retains the huge `bigData` array via the closure,
// even though the returned function only needs `bigData.length`.
function makeCounter() {
  const bigData = new Array(1e6).fill('*'); // ~ large
  return () => bigData.length;              // closure keeps bigData alive
}
```

Closures keep their *entire* enclosing scope reachable. If a long-lived function closes over a large object, that object can never be collected. Extract only what you need: `const len = bigData.length; return () => len;`.

#### 3. Timers and intervals you never clear

```js
// LEAK: setInterval keeps `payload` (and the whole closure) alive forever
// unless you clearInterval. Common in code that starts a poller per request.
function startPolling(payload) {
  return setInterval(() => doSomething(payload), 1000);
}
// FIX: store the id and clearInterval when done.
```

#### 4. Event listeners that are added but never removed — the #1 real-world leak

```js
// LEAK: every request adds a listener to a long-lived emitter and never removes it.
const bus = require('./eventBus'); // a singleton EventEmitter

app.get('/subscribe', (req, res) => {
  function onTick(data) { /* ... */ }
  bus.on('tick', onTick);   // added on every request...
  res.send('subscribed');   // ...but never removed -> listeners pile up
});
```

Node even warns you about this: `MaxListenersExceededWarning: Possible EventEmitter memory leak detected. 11 tick listeners added.` That warning is a *feature*, not noise — it usually means a real leak. The fix is to remove listeners when they are no longer needed (`bus.off('tick', onTick)` on `res` close), or use `bus.once(...)` when one-shot.

```js
app.get('/subscribe', (req, res) => {
  function onTick(data) { res.write(`data: ${JSON.stringify(data)}\n\n`); }
  bus.on('tick', onTick);
  req.on('close', () => bus.off('tick', onTick)); // clean up when the client leaves
});
```

> WARNING — Pitfall: Detached DOM-style leaks have a Node analogue: holding references in a `Map` keyed by request/connection and never deleting the entry when the connection closes. If you key a cache by something unbounded (user id, session id, socket), you **must** delete on disconnect or use a `WeakMap`/TTL.

### Capturing and reading heap snapshots

A **heap snapshot** is a complete dump of every object in the JS heap and the references between them. Diffing two snapshots taken at different times shows you *what grew* — and that is how you find the leaking object type.

There are three ways to capture one:

```bash
# Way A: from Chrome DevTools (chrome://inspect) -> Memory tab -> "Take heap snapshot".
node --inspect server.js
```

```js
// Way B: programmatically, on demand (e.g. triggered by a signal in production).
const v8 = require('node:v8');
process.on('SIGUSR2', () => {
  const file = v8.writeHeapSnapshot(); // writes Heap.<timestamp>.heapsnapshot
  console.log(`Heap snapshot written to ${file}`);
});
// Trigger with:  kill -USR2 <pid>   then load the file into DevTools.
```

```bash
# Way C: automatically when the process is about to OOM (great for post-mortems).
node --heapsnapshot-near-heap-limit=2 server.js
```

**The workflow to find a leak:**

1. Start the app, take a **baseline** snapshot.
2. Drive the suspected leaky behavior hard (e.g., 10,000 requests to the endpoint) with `autocannon`.
3. Force GC if you can, then take a **second** snapshot.
4. In DevTools, select the second snapshot and choose the **"Comparison"** view against the baseline. Sort by **"# Delta"** (count) or **"Size Delta"**.
5. The object type with a large positive delta that *should* have been collected is your leak. Click it, expand **"Retainers"** (the chain of references keeping it alive) — the retainer path points straight at the offending code (the `Map`, the listener array, the closure).

> TIP: In the DevTools Memory comparison, the column to trust is the **retained size** and the **retainers tree**. "Retained size" is how much memory would be freed if this object were collected; the retainers tree tells you *who* is preventing that. Following the retainer chain to a variable in your code is the moment the leak is solved.

---

## 4. Caching: The Highest-Leverage Optimization

### The core idea and the *why*

A cache stores the result of an expensive operation so you can return it cheaply next time instead of redoing the work. The expensive operation might be a database query, an external API call, a heavy computation, or a rendered template. **Caching is the single highest-leverage performance technique in web engineering** because it attacks the root cause: the fastest work is the work you never do.

The trade-off is captured by Phil Karlton's famous quip:

> "There are only two hard things in Computer Science: cache invalidation and naming things."

Caching is easy to *add* and hard to *keep correct*. The danger is **staleness**: serving data that has changed. Every caching decision is a deliberate trade between freshness and speed.

### Where to cache: the layers

Caching can happen at many layers, from the client outward to your data store. Each layer that absorbs a request is a layer the inner ones never see.

```
Browser cache  ->  CDN  ->  Reverse proxy (nginx/Varnish)  ->  App in-process cache  ->  Distributed cache (Redis)  ->  Database
   (§6)             (§6)        (HTTP, §6)                        (§4.2)                      (§5)                      (the slow thing)
```

This module focuses on the two layers *you* control in Node: the **in-process** cache and the **distributed** (Redis) cache. HTTP caching (browser/CDN/proxy) is §6.

### 4.1 In-process vs. distributed cache

| Aspect | In-process (in-memory, this Node process) | Distributed (Redis, Memcached) |
| --- | --- | --- |
| Latency | **Fastest** — nanoseconds, no network | Fast — sub-millisecond, but a network hop |
| Capacity | Limited by the process heap; competes with your app | Large, independent, can be huge |
| Shared across instances? | **No** — each process has its own copy | **Yes** — one source of truth for all instances |
| Survives restart? | No — gone on deploy/crash | Yes — survives app restarts |
| Consistency across the fleet | Poor — N processes, N possibly-different copies | Good — single shared state |
| Invalidation | Hard to coordinate across instances | Easy — `DEL` once, all instances see it |
| Adds infrastructure? | No | Yes — you must run/operate Redis |

The rule of thumb:

- **In-process** is perfect for small, hot, read-mostly data that is the same for every process and where slight staleness across instances is acceptable (config, feature flags, reference lookups). It is also great as an **L1 cache in front of Redis** to absorb the most-frequent keys.
- **Distributed (Redis)** is required when you run **more than one instance** and need a *shared, consistent* cache, when the data set is too big for the heap, or when you need invalidation to take effect everywhere at once.

> WARNING — Pitfall: An in-process cache and horizontal scaling are in tension. If you run 4 instances behind a load balancer, an in-process cache means **4 separate caches** that drift apart and that you cannot invalidate together. The moment you scale past one process, reach for Redis for anything that must be consistent.

### 4.2 In-process caching with `lru-cache`

Never roll your own unbounded object cache (that is leak source #1). Use a bounded LRU (Least Recently Used) cache, which evicts the least-recently-used entries when it hits a size or memory limit.

```bash
npm install lru-cache
```

```js
// cache.js — a bounded in-process cache
const { LRUCache } = require('lru-cache');

const cache = new LRUCache({
  max: 500,                 // keep at most 500 entries (then evict LRU)
  ttl: 1000 * 60 * 5,       // entries expire after 5 minutes
  ttlAutopurge: false,      // expired entries are purged lazily on access (cheaper)
  // Optionally bound by size instead of/in addition to count:
  // maxSize: 50 * 1024 * 1024,
  // sizeCalculation: (value) => Buffer.byteLength(JSON.stringify(value)),
});

module.exports = cache;
```

```js
// Using it with the cache-aside pattern (see §4.4) in a route.
const cache = require('./cache');

app.get('/products/:id', async (req, res, next) => {
  try {
    const key = `product:${req.params.id}`;
    let product = cache.get(key);     // 1. look in cache
    if (!product) {                   // 2. miss -> load from source
      product = await Product.findById(req.params.id).lean();
      if (product) cache.set(key, product); // 3. populate cache
    }
    res.json(product);
  } catch (err) {
    next(err);
  }
});
```

### 4.3 Memoizing pure, expensive computations

Caching is not only for I/O. A **pure** function (same input -> same output, no side effects) is a perfect memoization candidate.

```js
// A bounded memoizer using LRU as the backing store.
const { LRUCache } = require('lru-cache');

function memoize(fn, { max = 1000 } = {}) {
  const store = new LRUCache({ max });
  return function (...args) {
    const key = JSON.stringify(args); // assumes JSON-serializable args
    if (store.has(key)) return store.get(key);
    const result = fn.apply(this, args);
    store.set(key, result);
    return result;
  };
}

const slowSquareSum = (n) => {
  let s = 0;
  for (let i = 0; i < n; i++) s += Math.sqrt(i);
  return s;
};
const fastSquareSum = memoize(slowSquareSum);
```

> TIP: Only memoize **pure** functions, and prefer a **bounded** store. Memoizing an impure function caches stale/wrong results; memoizing into an unbounded `Map` reintroduces leak source #1.

---

## 5. Redis as a Cache

Redis is an in-memory key-value data store, accessed over the network, shared by all your app instances. It is the default choice for a distributed cache (and much more — queues, rate limiting, pub/sub, sessions). For caching, the features that matter are: blazing speed, **TTL/expiry** per key, atomic operations, and simple `GET`/`SET`/`DEL`.

### 5.1 Clients: `ioredis` vs `node-redis`

There are two mainstream Node clients. Both are excellent and Promise-based today.

| | `ioredis` | `node-redis` (the official `redis` package, v4+) |
| --- | --- | --- |
| API style | Methods mirror Redis commands (`redis.set`) | Methods mirror Redis commands (`client.set`); camelCase variants too |
| Cluster / Sentinel | First-class, batteries-included | Supported |
| Promises | Native | Native (v4+) |
| Popularity | Very widely used in production | Official, widely used |

Both are fine. The examples below use `ioredis` (its connection ergonomics are slightly simpler), with `node-redis` shown for the basics.

```bash
npm install ioredis
# or: npm install redis    (node-redis)
```

```js
// redisClient.js — ioredis
const Redis = require('ioredis');

// Reconnects automatically; reads connection from env in real apps.
const redis = new Redis(process.env.REDIS_URL || 'redis://127.0.0.1:6379', {
  maxRetriesPerRequest: 2,            // fail fast rather than queueing forever
  enableReadyCheck: true,
});

redis.on('error', (err) => console.error('[redis] error:', err.message));
redis.on('connect', () => console.log('[redis] connected'));

module.exports = redis;
```

```js
// node-redis (the official `redis` package) equivalent — note explicit connect()
const { createClient } = require('redis');
const client = createClient({ url: process.env.REDIS_URL });
client.on('error', (err) => console.error('[redis] error', err));
await client.connect(); // node-redis v4 requires an explicit connect
// usage: await client.set('k', 'v'); await client.get('k');
```

### 5.2 Basic operations and TTL

```js
const redis = require('./redisClient');

// SET with an expiry. 'EX' = seconds; 'PX' = milliseconds.
await redis.set('greeting', 'hello', 'EX', 60); // expires in 60s

const value = await redis.get('greeting'); // 'hello' (or null after expiry/miss)

await redis.del('greeting');               // delete a key
const ttl = await redis.ttl('greeting');   // seconds left (-2 = no key, -1 = no expiry)
await redis.expire('greeting', 30);        // (re)set TTL on an existing key
```

Redis values are strings (and bytes), so objects must be serialized — typically with JSON:

```js
async function cacheSetJSON(key, obj, ttlSeconds) {
  await redis.set(key, JSON.stringify(obj), 'EX', ttlSeconds);
}
async function cacheGetJSON(key) {
  const raw = await redis.get(key);
  return raw ? JSON.parse(raw) : null;
}
```

> TIP: **Always set a TTL on cache entries** unless you have a specific reason not to. A TTL is your safety net: even if your invalidation logic has a bug, stale data self-heals when it expires. Cache entries without TTLs are how a cache slowly fills with garbage and serves stale data forever.

### 5.3 The cache-aside (lazy-loading) pattern

Cache-aside is the workhorse caching pattern. The *application* is responsible for the cache; the cache does not know about the database.

The flow:

1. **Read** request arrives.
2. Look in the cache (`GET`). On a **hit**, return it — done.
3. On a **miss**, read from the database.
4. Write the result into the cache (`SET` with TTL).
5. Return the result.

```js
// cacheAside.js — a reusable helper
const redis = require('./redisClient');

/**
 * Cache-aside: return cached value if present, else compute via loader,
 * cache it with a TTL, and return it.
 */
async function getOrSet(key, ttlSeconds, loader) {
  const cached = await redis.get(key);
  if (cached !== null) {
    return JSON.parse(cached);                 // cache HIT
  }
  const fresh = await loader();                // cache MISS -> load from source
  // Only cache non-null results (caching `null` can be useful too; see below).
  if (fresh !== undefined && fresh !== null) {
    await redis.set(key, JSON.stringify(fresh), 'EX', ttlSeconds);
  }
  return fresh;
}

module.exports = { getOrSet };
```

```js
// Using it in an Express route.
const { getOrSet } = require('./cacheAside');
const Product = require('./models/Product');

app.get('/products/:id', async (req, res, next) => {
  try {
    const product = await getOrSet(
      `product:${req.params.id}`,
      300, // 5-minute TTL
      () => Product.findById(req.params.id).lean() // the expensive loader
    );
    if (!product) return res.status(404).json({ error: 'Not found' });
    res.json(product);
  } catch (err) {
    next(err);
  }
});
```

### 5.4 Invalidation: keeping the cache honest

The hard part. When the underlying data changes, the cache must be updated or removed, or you serve stale data. Strategies, roughly from simplest to most sophisticated:

1. **TTL-only (passive)** — let entries expire. Simplest; accepts staleness up to the TTL. Great for data that tolerates being a few minutes old (product listings, public profiles).
2. **Write-through / explicit invalidation (active)** — when you write to the DB, also `DEL` (or update) the cache key in the same operation. Keeps the cache fresh but you must remember it on *every* write path.

```js
// On update, write to DB then invalidate the cache key.
app.put('/products/:id', async (req, res, next) => {
  try {
    const updated = await Product.findByIdAndUpdate(
      req.params.id, req.body, { new: true }
    ).lean();
    await redis.del(`product:${req.params.id}`); // invalidate -> next read repopulates
    res.json(updated);
  } catch (err) {
    next(err);
  }
});
```

3. **Tag / pattern invalidation** — when one change affects many keys (e.g., "all listings for category X"). Avoid `KEYS pattern*` in production (it is O(N) and blocks Redis); maintain a **set of keys per tag** and delete them together, or use `SCAN` (cursor-based, non-blocking) if you must pattern-match.

```js
// Maintain a tag -> keys set, so you can invalidate a whole group at once.
async function setWithTag(tag, key, value, ttl) {
  await redis.set(key, JSON.stringify(value), 'EX', ttl);
  await redis.sadd(`tag:${tag}`, key); // remember this key belongs to the tag
}
async function invalidateTag(tag) {
  const keys = await redis.smembers(`tag:${tag}`);
  if (keys.length) await redis.del(...keys);
  await redis.del(`tag:${tag}`);
}
```

### 5.5 Two production hazards: stampedes and negative caching

**Cache stampede (thundering herd):** when a popular key expires, *many* concurrent requests all miss simultaneously and all hit the database at once — a self-inflicted spike that can knock the DB over. Mitigations: a short **lock** so only one request recomputes while others wait, **stale-while-revalidate** (serve the stale value while one worker refreshes), or **jittered TTLs** so keys do not all expire at the same instant.

```js
// Simple per-key lock to prevent stampedes (one recomputes, others briefly back off).
async function getOrSetLocked(key, ttl, loader) {
  const cached = await redis.get(key);
  if (cached !== null) return JSON.parse(cached);

  const lockKey = `lock:${key}`;
  // SET NX: acquire the lock only if it does not exist; auto-release after 10s.
  const gotLock = await redis.set(lockKey, '1', 'EX', 10, 'NX');
  if (!gotLock) {
    // Someone else is computing it; wait briefly and retry the cache.
    await new Promise((r) => setTimeout(r, 50));
    return getOrSetLocked(key, ttl, loader);
  }
  try {
    const fresh = await loader();
    await redis.set(key, JSON.stringify(fresh), 'EX', ttl);
    return fresh;
  } finally {
    await redis.del(lockKey);
  }
}
```

**Negative caching:** if a key reliably misses (a nonexistent id queried repeatedly), cache the "not found" result with a *short* TTL so you stop hammering the DB for something that does not exist. This blunts **cache penetration** attacks where an attacker requests random nonexistent ids.

> BEST PRACTICE: Treat Redis as a cache, not a system of record — design so your app **still works (slower) if Redis is down**. Wrap cache reads in try/catch and fall back to the database on error. A caching layer should never be a single point of failure for correctness.

---

## 6. HTTP Caching: Let the Network Do the Work

Everything so far cached *inside* your infrastructure. HTTP caching pushes caching all the way out to **browsers, CDNs, and reverse proxies** — meaning many requests never reach your server at all. It is governed by response headers your app sets.

### 6.1 `Cache-Control`: the master switch

`Cache-Control` tells caches whether, where, and for how long a response may be stored.

| Directive | Meaning |
| --- | --- |
| `max-age=N` | Fresh for N seconds (from when fetched). |
| `s-maxage=N` | Like `max-age` but for *shared* caches (CDN/proxy) only; overrides `max-age` there. |
| `public` | Any cache (including shared/CDN) may store it. |
| `private` | Only the end-user's browser may store it (not CDNs) — for per-user data. |
| `no-cache` | May store, but **must revalidate** with the origin before reuse (conditional request). |
| `no-store` | Do not store at all — for sensitive data. |
| `must-revalidate` | Once stale, must revalidate before serving. |
| `immutable` | The content will never change for this URL (great for fingerprinted assets). |

```js
// Static, fingerprinted asset (e.g. app.8f3a2c.js) — cache hard, forever.
app.get('/assets/:file', (req, res) => {
  res.set('Cache-Control', 'public, max-age=31536000, immutable'); // 1 year
  res.sendFile(/* ... */);
});

// Per-user, semi-dynamic data — let the browser cache briefly but privately.
app.get('/me/dashboard', (req, res) => {
  res.set('Cache-Control', 'private, max-age=30');
  res.json(/* ... */);
});

// Sensitive data — never store.
app.get('/me/billing', (req, res) => {
  res.set('Cache-Control', 'no-store');
  res.json(/* ... */);
});
```

> BEST PRACTICE: The standard pattern for front-end assets is **"cache forever + fingerprint the filename."** Because the filename changes whenever the content changes (`app.8f3a2c.js` -> `app.b91e07.js`), you can safely set a 1-year `immutable` cache; a new deploy simply references a new URL. HTML, by contrast, is usually `no-cache` so clients always pick up the latest asset references.

### 6.2 ETags and conditional requests: the `304` dance

What about data that *might* change but usually has not? You do not want to re-download it, but you do want correctness. Enter **validators**: `ETag` (an opaque fingerprint of the response body, like a hash) and `Last-Modified` (a timestamp).

The conditional-request flow:

1. Server sends a response with `ETag: "abc123"`.
2. Browser caches the body *and* the ETag.
3. Next time, the browser sends `If-None-Match: "abc123"`.
4. The server compares: if the current ETag still matches, it replies **`304 Not Modified`** with **no body**. The browser reuses its cached copy.
5. If it changed, the server replies `200` with the new body and a new ETag.

The win: even when the client must check, a `304` saves transferring (and serializing) the entire body — just a tiny round-trip of headers.

**Express handles ETags for you automatically.** Built-in static serving and `res.send`/`res.json` add a (weak) ETag by default, and Express compares incoming `If-None-Match` to send `304` automatically.

```js
// Express's default ETag is on. You can configure it:
app.set('etag', 'strong'); // 'weak' (default) | 'strong' | false | a custom function

app.get('/config', (req, res) => {
  // res.json computes an ETag from the body; on a matching If-None-Match,
  // Express sends 304 with no body automatically.
  res.json({ theme: 'dark', features: ['a', 'b'] });
});
```

You can also set validators explicitly when you have a natural version (e.g., a document's `updatedAt` or version number), which is cheaper than hashing the whole body:

```js
app.get('/articles/:id', async (req, res, next) => {
  try {
    const article = await Article.findById(req.params.id).lean();
    if (!article) return res.sendStatus(404);

    const etag = `"${article._id}-${article.__v}"`; // cheap, version-based ETag
    res.set('ETag', etag);
    res.set('Cache-Control', 'no-cache'); // store, but revalidate each time

    if (req.headers['if-none-match'] === etag) {
      return res.status(304).end(); // unchanged -> no body, big savings
    }
    res.json(article);
  } catch (err) {
    next(err);
  }
});
```

> WARNING — Pitfall: `no-cache` does **not** mean "do not cache." It means "cache, but always revalidate before using." To actually prevent storage, use `no-store`. Mixing these up is a classic cause of either leaking sensitive data into caches (`no-cache` when you meant `no-store`) or killing performance (`no-store` when `no-cache` would do).

> WARNING — Pitfall: Express's **weak** default ETag is computed from the response body, which means the body is fully generated *before* the `304` decision — you save **bandwidth** but not server compute. For expensive responses, set a cheap **explicit** ETag from a version/`updatedAt` and short-circuit *before* doing the expensive work, as in the example above.

---

## 7. Response Compression

### Why and how

Text-based responses (JSON, HTML, CSS, JS) compress dramatically — often **70–90% smaller** with gzip or Brotli. Smaller responses mean less bandwidth and faster delivery, especially on mobile networks where latency and throughput are the bottleneck, not your server's CPU. The mechanism is content negotiation: the client advertises `Accept-Encoding: gzip, br`, and the server compresses the body and sets `Content-Encoding: gzip`.

In Express, the `compression` middleware does this transparently:

```bash
npm install compression
```

```js
const compression = require('compression');
const app = require('express')();

app.use(compression({
  threshold: 1024,    // only compress responses larger than 1KB (small ones aren't worth it)
  level: 6,           // zlib level 0-9; 6 is a good speed/ratio default
  // Custom filter: skip compression when a request opts out.
  filter: (req, res) => {
    if (req.headers['x-no-compression']) return false;
    return compression.filter(req, res); // default: compress text-like content types
  },
}));
```

That single `app.use` compresses every eligible response. Order matters: put it **early** in the middleware stack so it wraps the responses that follow.

### When NOT to compress, and where to compress

| Situation | Guidance |
| --- | --- |
| Already-compressed payloads (JPEG, PNG, MP4, `.zip`, `.gz`) | **Do not** re-compress — wastes CPU for ~0 gain (sometimes grows the file). |
| Tiny responses (< ~1KB) | Skip — the overhead/headers cost more than you save (hence `threshold`). |
| High-traffic production behind nginx/CDN | Prefer compressing at the **proxy/CDN**, not in Node — frees your event loop. |
| TLS + compressing secret-bearing responses | Beware **BREACH/CRIME**-class attacks when reflecting user input into compressed, secret-bearing responses. |

> BEST PRACTICE: Compression is **CPU work on your single thread**. At low/medium traffic, the `compression` middleware is a great, easy win. At high traffic, **offload compression to nginx, a CDN, or a load balancer** so it does not compete with your request handling. Either way, do not compress what is already compressed.

> TIP: Brotli (`br`) typically beats gzip on text, especially for static assets where you can pre-compress at build time and serve the `.br` file directly. Modern reverse proxies and CDNs negotiate Brotli automatically; the `compression` middleware focuses on gzip/deflate.

---

## 8. Database Performance

For most web apps, **the database is the bottleneck**, not Node. A slow query blocks nothing in Node (it is async), but it makes the *request* slow, holds a connection, and limits throughput. These are the highest-impact database optimizations, with Mongoose examples.

### 8.1 Indexes: the single biggest lever

Without an index, a query must scan **every document** in the collection (a "collection scan", `COLLSCAN`) to find matches — O(N). An index is a sorted data structure (a B-tree) that lets the database jump straight to matching documents — roughly O(log N). On a million-document collection, this is the difference between milliseconds and seconds.

```js
// In a Mongoose schema, declare indexes on fields you query/sort/filter by.
const userSchema = new mongoose.Schema({
  email: { type: String, required: true, unique: true }, // unique => unique index
  status: String,
  createdAt: Date,
});

// Single-field index for filtering by status.
userSchema.index({ status: 1 });

// Compound index: supports queries filtering by status AND sorting by createdAt.
// Order matters and follows the "ESR" rule: Equality, then Sort, then Range.
userSchema.index({ status: 1, createdAt: -1 });
```

Prove an index is used with `.explain()`:

```js
const plan = await User
  .find({ status: 'active' })
  .sort({ createdAt: -1 })
  .explain('executionStats');

// Look at plan.executionStats:
//   executionStages.stage === 'IXSCAN'   => good, using the index
//   executionStages.stage === 'COLLSCAN' => bad, full scan; you need an index
//   totalDocsExamined ~= nReturned       => good (examined ~ what you returned)
//   totalDocsExamined >> nReturned       => bad (scanning far more than needed)
```

> WARNING — Pitfall: In **development**, Mongoose calls `createIndex` automatically for schema-declared indexes (`autoIndex` defaults to on). In **production with large collections, turn `autoIndex` off** (`mongoose.connect(uri, { autoIndex: false })`) and create indexes deliberately via a migration — building an index on a live, huge collection can lock/strain the database at the worst possible moment.

> BEST PRACTICE: Indexes are not free — they consume memory and **slow down writes** (every insert/update must maintain every index). Index the fields you actually filter, sort, and join on; do not blindly index everything. Periodically review unused indexes and drop them.

### 8.2 `lean()`: skip the Mongoose overhead

By default, a Mongoose query returns full **Mongoose documents** — rich objects with getters, setters, virtuals, change tracking, and a `.save()` method. That hydration is *expensive* and pointless when you are only going to read the data and send it as JSON. `.lean()` returns **plain JavaScript objects** instead, often **several times faster** and far lighter on memory.

```js
// Heavy: full Mongoose documents (use when you'll modify and .save()).
const users = await User.find({ status: 'active' });

// Light & fast: plain objects (use for read-only / API responses).
const usersLean = await User.find({ status: 'active' }).lean();
```

> BEST PRACTICE: Use `.lean()` for **any read-only query**, especially list/index endpoints that return many documents. The only time you must *not* use it is when you need document methods, virtuals, or to call `.save()` on the result. This is one of the easiest, highest-impact Mongoose optimizations.

### 8.3 Projection: fetch only the fields you need

Pulling back entire documents when you need three fields wastes database I/O, network transfer, and memory. **Projection** restricts which fields are returned.

```js
// Return only name and email; exclude everything else. ('1' include / '0' exclude)
const users = await User.find({ status: 'active' }, 'name email').lean();

// Equivalent with .select(). Prefix with '-' to exclude a heavy field.
const posts = await Post.find().select('title author -body').lean();
```

This matters most for documents with large fields (article bodies, embedded arrays, blobs). Combine projection with `lean()` for the leanest possible read.

### 8.4 The N+1 problem and `populate`

The **N+1 query problem** is the most common database performance bug in application code. You run **1** query to fetch a list, then loop and run **N** more queries — one per item — to fetch each item's related data. 1 + N queries where 2 would do.

```js
// N+1 ANTI-PATTERN: 1 query for posts, then N queries (one per post) for authors.
const posts = await Post.find().lean();           // 1 query
for (const post of posts) {
  post.author = await User.findById(post.author).lean(); // N queries!
}
```

With 100 posts that is 101 round-trips to the database. The fix in Mongoose is `populate`, which gathers the related ids and fetches them in **one** additional query (using `$in` under the hood):

```js
// Better: 1 query for posts + 1 query for all referenced authors = 2 total.
const posts = await Post.find()
  .populate('author', 'name email') // project the joined fields too
  .lean();                          // still works with populate
```

For more complex joins or aggregations, MongoDB's `$lookup` aggregation stage does a server-side join in a single round-trip. Whatever the tool, the principle is the same: **batch related lookups; never query inside a loop.**

> TIP: To catch N+1 bugs, watch your query count under load (Mongoose's `mongoose.set('debug', true)` logs every query). If a single API request fires dozens of nearly-identical queries, you have an N+1.

### 8.5 Connection pooling

Opening a database connection is expensive (handshake, auth, TLS). You do not want a new connection per request, nor a single connection serializing all requests. A **connection pool** maintains a set of reusable open connections; each query borrows one and returns it. The MongoDB driver (and thus Mongoose) pools automatically — you just tune the size.

```js
await mongoose.connect(process.env.MONGO_URL, {
  maxPoolSize: 20,   // max simultaneous connections (default 100; tune to your DB)
  minPoolSize: 5,    // keep some warm to avoid cold-start latency
  serverSelectionTimeoutMS: 5000, // fail fast if the DB is unreachable
  socketTimeoutMS: 45000,
});
```

> WARNING — Pitfall: Pool sizing is a balance. **Too small** and requests queue waiting for a free connection (latency climbs under load). **Too large** and you can overwhelm the database with more concurrent connections than it can handle (every connection costs the DB memory). Right-size against your DB's connection limit, accounting for *all* app instances sharing it: `maxPoolSize × number_of_instances` must stay under the DB's max.

> BEST PRACTICE: Connect **once at startup**, reuse the pooled connection for the process's lifetime, and never call `mongoose.connect` per request. In serverless (where instances are ephemeral), cache the connection across invocations and use a small pool to avoid connection storms.

---

## 9. Pagination for Large Datasets

Never return an unbounded list. A `GET /posts` that returns 2 million rows will blow up memory, freeze serialization, and time out the client. **Always paginate.** There are two fundamental approaches.

### 9.1 Offset (skip/limit) pagination

The familiar `?page=3&limit=20` approach: skip `(page-1)*limit` documents and take `limit`.

```js
app.get('/posts', async (req, res, next) => {
  try {
    const page = Math.max(1, parseInt(req.query.page) || 1);
    const limit = Math.min(100, parseInt(req.query.limit) || 20); // cap the limit!
    const skip = (page - 1) * limit;

    const [items, total] = await Promise.all([
      Post.find().sort({ createdAt: -1 }).skip(skip).limit(limit).lean(),
      Post.countDocuments(), // for total pages (can be expensive on huge collections)
    ]);

    res.json({ page, limit, total, totalPages: Math.ceil(total / limit), items });
  } catch (err) {
    next(err);
  }
});
```

**Pros:** simple; supports "jump to page N" and showing total page counts. **Cons:** two serious problems at scale:

1. **`skip` is O(N)** — the database still walks past all skipped documents. `skip(1_000_000)` is slow; deep pages get progressively slower.
2. **Drift/instability** — if rows are inserted or deleted between requests, items shift across page boundaries, so a user can see the same item twice or miss one while paging.

### 9.2 Cursor (keyset / seek) pagination

Instead of "skip N," you remember the **last item you saw** and ask for "the next page *after* this cursor." The cursor is a value from a unique, ordered field (e.g., `_id`, or `createdAt` + `_id` as a tiebreaker).

```js
app.get('/feed', async (req, res, next) => {
  try {
    const limit = Math.min(100, parseInt(req.query.limit) || 20);
    const { cursor } = req.query; // the _id of the last item from the previous page

    // Sorting by _id descending; "after this cursor" means _id < cursor.
    const query = cursor ? { _id: { $lt: cursor } } : {};

    // Fetch one extra to know whether there is a next page.
    const docs = await Post.find(query)
      .sort({ _id: -1 })
      .limit(limit + 1)
      .lean();

    const hasMore = docs.length > limit;
    const items = hasMore ? docs.slice(0, limit) : docs;
    const nextCursor = hasMore ? items[items.length - 1]._id : null;

    res.json({ items, nextCursor, hasMore });
  } catch (err) {
    next(err);
  }
});
```

**Pros:** **O(log N)** via the index regardless of depth — page 10,000 is as fast as page 1; **stable** under inserts/deletes (you anchor to a real value, not a position). This is how infinite-scroll feeds (Twitter, Slack) work. **Cons:** no "jump to page N" and no easy total count; the sort field must be unique and indexed.

| | Offset (skip/limit) | Cursor (keyset) |
| --- | --- | --- |
| Deep-page performance | Degrades — `skip` is O(N) | Constant — O(log N) via index |
| Stable under writes? | No — items shift across pages | Yes — anchored to a real value |
| "Jump to page N" / total count | Yes | No (only next/previous) |
| Best for | Admin tables, small datasets, page numbers | Feeds, infinite scroll, large/active datasets |

> BEST PRACTICE: Use **offset** pagination for admin UIs and small datasets where page numbers matter. Use **cursor** pagination for large, frequently-changing datasets and infinite-scroll feeds. In both cases, **cap the `limit`** server-side (e.g., max 100) so a client cannot request a million rows in one page, and ensure the sort field is indexed.

---

## 10. Load Testing with `autocannon`

You have optimized. Now **prove it.** Load testing fires many concurrent requests at your server and reports how it behaves under pressure — throughput, latency distribution, and errors. `autocannon` (Node-based, fast, easy) is the go-to.

```bash
npm install -g autocannon   # or use npx autocannon
```

### 10.1 Running a test

```bash
# 100 concurrent connections, for 30 seconds, against /products.
autocannon -c 100 -d 30 http://localhost:3000/products

# POST with a JSON body and headers:
autocannon -c 50 -d 20 \
  -m POST \
  -H 'Content-Type=application/json' \
  -b '{"name":"widget","price":9.99}' \
  http://localhost:3000/products

# Pipeline requests (send N before reading responses) to push throughput limits:
autocannon -c 100 -p 10 -d 20 http://localhost:3000/
```

Key flags: `-c` connections (concurrency), `-d` duration (seconds), `-p` pipelining, `-m` method, `-H` header, `-b` body, `-w` worker threads (to generate more load than one client thread can).

### 10.2 Reading the results — and what each number means

```
Running 30s test @ http://localhost:3000/products
100 connections

┌─────────┬──────┬──────┬───────┬──────┬─────────┬─────────┬───────┐
│ Stat    │ 2.5% │ 50%  │ 97.5% │ 99%  │ Avg     │ Stdev   │ Max   │
├─────────┼──────┼──────┼───────┼──────┼─────────┼─────────┼───────┤
│ Latency │ 3 ms │ 8 ms │ 41 ms │ 70ms │ 11.2 ms │ 12.1 ms │ 210ms │
└─────────┴──────┴──────┴───────┴──────┴─────────┴─────────┴───────┘
┌───────────┬─────────┬─────────┬─────────┬─────────┬─────────┐
│ Stat      │ 1%      │ 2.5%    │ 50%     │ Avg     │ Stdev   │
├───────────┼─────────┼─────────┼─────────┼─────────┼─────────┤
│ Req/Sec   │ 6,100   │ 6,400   │ 8,200   │ 8,050   │ 540     │
│ Bytes/Sec │ 2.1 MB  │ 2.2 MB  │ 2.9 MB  │ 2.8 MB  │ 190 kB  │
└───────────┴─────────┴─────────┴─────────┴─────────┴─────────┘

241k requests in 30.0s, 84 MB read
0 errors (0 timeouts)
```

How to read it — and what actually matters:

- **Req/Sec (throughput)** — requests served per second. Higher is better. This is your headline capacity number.
- **Latency percentiles — focus on the tail, not the average.** `50%` (the median) is the typical experience. But **`p97.5`, `p99` (the "tail")** are what real users feel under load and what your SLOs are written against. An 8ms median with a 700ms p99 means *some* users have a terrible time — averages hide this, which is exactly why you read percentiles.
- **Avg vs. Stdev** — a large `Stdev` relative to `Avg` means inconsistent latency (often a sign of GC pauses or intermittent event-loop blocking). Consistency matters as much as speed.
- **Errors / timeouts / non-2xx** — these must be **near zero**. Throughput numbers are meaningless if the server is shedding requests. A test that reports huge req/sec but 30% non-2xx is a *failing* server, not a fast one. Always check this line.

### 10.3 The right way to use load testing

1. **Establish a baseline.** Test the endpoint *before* optimizing. Write the numbers down (p50, p99, req/sec, errors).
2. **Change one thing.** Add the cache, the index, compression — one variable.
3. **Re-test identically** (same `-c`, `-d`, same machine, same data). Compare to the baseline.
4. **Watch the server while testing.** Run your event-loop-lag monitor (§1) and memory monitor (§3) during the test. If p99 latency climbs steadily over 30s, you may be leaking or accumulating; if loop lag spikes, you are blocking.

```bash
# A clean before/after comparison ritual:
autocannon -c 100 -d 30 http://localhost:3000/products > before.txt   # baseline
# ... add Redis cache-aside to the endpoint ...
autocannon -c 100 -d 30 http://localhost:3000/products > after.txt    # re-test
diff <(grep -E 'Req/Sec|99|Latency' before.txt) \
     <(grep -E 'Req/Sec|99|Latency' after.txt)
```

> WARNING — Pitfall: Load-testing on the **same machine** that runs the server means the test client and the server compete for CPU — your numbers will be pessimistic and noisy. For meaningful absolute numbers, run the load generator on a **separate machine** on a fast network. For *relative* before/after comparisons on your laptop, same-machine is fine as long as conditions are identical.

> TIP: `autocannon` is for **HTTP throughput/latency** benchmarking. For complex multi-step user scenarios (login -> browse -> checkout) with think-time and assertions, reach for `k6` or `artillery` instead. Use the right tool: autocannon answers "how fast is this endpoint?", scenario tools answer "how does the system behave under a realistic user journey?"

---

## Common Pitfalls & Gotchas

- **Blocking the event loop with `*Sync` calls or heavy loops inside request handlers.** Freezes the *whole* server for *all* clients. Offload CPU work to Worker threads; keep `*Sync` to startup only.
- **Premature optimization without measuring.** Optimizing the wrong thing wastes effort and adds complexity for no gain. Profile first; let the flamegraph pick the target.
- **Unbounded in-memory caches / `Map`s** that grow forever — a guaranteed memory leak. Always bound with `lru-cache` (size + TTL).
- **Forgetting to set TTLs on Redis keys.** Without expiry, your cache fills with stale junk and serves outdated data indefinitely. TTL is your safety net.
- **Cache invalidation gaps.** Updating the DB but not the cache serves stale data; invalidate on every write path, and lean on TTLs as backup.
- **Cache stampedes** when a hot key expires and a flood of misses hits the DB at once. Use locking, stale-while-revalidate, or jittered TTLs.
- **Confusing `no-cache` with `no-store`.** `no-cache` = "store but revalidate"; `no-store` = "never store." Mixing them leaks data or kills performance.
- **Compressing already-compressed assets** (images, video, archives) — pure CPU waste. And compressing in Node at high traffic when the proxy/CDN should do it.
- **Missing or wrong indexes** -> collection scans. Verify with `.explain()`; watch for `COLLSCAN` and `totalDocsExamined >> nReturned`.
- **Not using `.lean()` for read-only queries** — paying for full Mongoose hydration you never use.
- **N+1 queries** — querying inside a loop. Use `populate` / `$lookup` / batching to collapse N queries into one.
- **`autoIndex: true` in production** on big collections — index builds can lock/strain the DB at deploy time.
- **Offset pagination on deep pages** — `skip` is O(N) and unstable under writes. Use cursor pagination for large/active datasets.
- **Unbounded list endpoints** — never return all rows; always paginate and cap the page size server-side.
- **Trusting averages in load tests** — averages hide tail latency. Read p99 and the error count, not just the mean.
- **Exposing `--inspect` publicly** — a remote code execution hole. Bind to localhost only.

## Best Practices

- **Measure, change one thing, measure again.** Every optimization needs a before/after with the same load profile.
- **Export event-loop lag and memory metrics to production monitoring.** Rising loop lag and creeping post-GC heap are leading indicators of incidents.
- **Keep CPU work off the main thread.** Worker threads (via a `piscina` pool) for genuine CPU work; horizontal scaling (cluster/PM2/containers) for throughput.
- **Cache as close to the user as correctness allows** — HTTP/CDN first (never hits your server), then in-process L1, then Redis L2, then the DB.
- **Always bound caches and always set TTLs.** Bounded LRU in-process; TTL on every Redis key as a staleness safety net.
- **Make the cache optional for correctness** — fall back to the DB if Redis is down. The cache speeds things up; it must not be required.
- **Use `lean()` + projection for reads**, indexes for filters/sorts, and `populate`/`$lookup` to avoid N+1. Verify with `.explain()` and `mongoose.set('debug', true)`.
- **Right-size the connection pool** against your DB's limit across all instances; connect once at startup.
- **Paginate everything; cap page size.** Cursor pagination for large/active datasets, offset for admin/page-number UIs.
- **Compress text responses** (offload to the proxy/CDN at scale); never compress already-compressed content.
- **Fingerprint static assets and cache them `immutable` for a year**; keep HTML `no-cache`.
- **Load test before you ship**, watch the tail (p99) and the error count, and run the generator off-box for absolute numbers.

## Key Interview Points

- **"What does it mean to block the event loop, and why is it so bad in Node?"** — Node runs JS on one thread; synchronous CPU work on that thread stops *all* request processing for *every* client until it finishes. In a thread-per-request server one slow request affects one thread; in Node it freezes the whole process. Detect with `monitorEventLoopDelay`; fix by offloading to Worker threads or scaling out.
- **"How do you profile a Node app and find the hot path?"** — `node --prof` + `--prof-process` for a tick profile; `--inspect` + Chrome DevTools or `clinic flame` for a flamegraph. Read the flamegraph: width = CPU time, height = stack depth; the widest boxes near the top are your targets. Start with `clinic doctor` to triage the *kind* of problem.
- **"What is a memory leak in a GC language, and how do you find one?"** — Keeping a reference to something you no longer need so the GC cannot collect it. Symptoms: post-GC heap creeps upward (sawtooth that climbs). Find it by diffing two heap snapshots, sorting by delta, and following the *retainers* chain to the code. Classic sources: unbounded caches/globals, large closures, uncleared timers, un-removed event listeners.
- **"In-process vs. distributed cache — when each?"** — In-process: fastest, but per-process (drifts/can't be invalidated across instances), lost on restart — good for small hot data on a single instance or as an L1. Distributed (Redis): shared, consistent across instances, survives restarts, easy global invalidation — required once you run more than one instance.
- **"Explain the cache-aside pattern."** — App checks cache; on hit returns it; on miss reads the DB, writes the result to the cache with a TTL, returns it. The cache is passive; the app owns it. Pair with explicit invalidation on writes and TTLs as a safety net.
- **"How do you keep a cache from going stale?"** — TTLs (passive expiry), explicit `DEL`/update on every write path (active), and tag-based invalidation for groups. Always combine active invalidation with a TTL backstop.
- **"What's a cache stampede and how do you prevent it?"** — When a hot key expires, many concurrent misses hammer the DB at once. Prevent with a lock so only one request recomputes, stale-while-revalidate, or jittered TTLs.
- **"`Cache-Control: no-cache` vs `no-store`?"** — `no-cache` = may store but must revalidate before use (enables 304s); `no-store` = never store (for sensitive data). They are not synonyms.
- **"How do ETags / conditional requests work?"** — Server sends `ETag`; client sends it back as `If-None-Match`; if unchanged, server returns `304 Not Modified` with no body, saving bandwidth. Express does this automatically; use a cheap version-based ETag to also skip server compute.
- **"What is the N+1 query problem and how do you fix it?"** — 1 query for a list + N queries (one per item) for related data. Fix by batching: Mongoose `populate` (one `$in` query) or `$lookup`. Never query inside a loop.
- **"`lean()` — what and why?"** — Returns plain JS objects instead of full Mongoose documents, skipping expensive hydration/change-tracking. Several times faster and lighter for read-only queries; don't use it when you need document methods or `.save()`.
- **"Why and how do you index?"** — Without an index, queries do a full collection scan (O(N)); an index (B-tree) gives ~O(log N). Index fields you filter/sort by; compound indexes follow Equality-Sort-Range order. Verify with `.explain()` (`IXSCAN` good, `COLLSCAN` bad). Trade-off: indexes cost memory and slow writes.
- **"Offset vs. cursor pagination?"** — Offset (skip/limit): simple, supports page numbers/totals, but `skip` is O(N) and unstable under writes. Cursor (keyset): anchors to the last seen value, O(log N) at any depth and stable, but no jump-to-page. Use cursor for large/active feeds.
- **"What does connection pooling do and how do you size it?"** — Reuses a set of open DB connections instead of opening one per request. Size it so `maxPoolSize × instances` stays under the DB's connection limit: too small queues requests, too large overwhelms the DB.
- **"How do you read load-test results?"** — Throughput (req/sec) for capacity, latency **percentiles** (p99/tail, not the average) for user experience, and errors/timeouts (must be ~0). Always baseline, change one thing, and re-test identically.

## Practice Exercises

1. **(Warm-up — detect blocking)** Build an Express server with two routes: `/block` that busy-waits for 3 seconds, and `/ping` that returns instantly. Add a `monitorEventLoopDelay`-based logger that prints mean and p99 loop delay every second. Hit `/block` and observe `/ping` hang and the loop-delay numbers spike. Then refactor `/block` to do its work in a Worker thread and confirm `/ping` stays responsive and loop delay stays low.

2. **(Cache-aside with Redis)** Given a `Product` Mongoose model, implement `GET /products/:id` using the cache-aside pattern against Redis with a 60-second TTL. Implement `PUT /products/:id` so it updates the DB *and* invalidates the cache key. Log whether each GET was a HIT or a MISS, and verify with `redis-cli` that the key exists with a TTL after a GET and is gone after a PUT.

3. **(HTTP caching / 304)** Implement `GET /articles/:id` that sets a cheap, version-based `ETag` (from the document's `_id` and `__v`) and `Cache-Control: no-cache`. When the incoming `If-None-Match` matches, short-circuit with `304` **before** querying the full document. Use `curl -i` to confirm you get a `200` first and a `304` (no body) on the second request, then a fresh `200` after you update the article.

4. **(Database — kill an N+1 and prove it)** Seed 100 `Post` documents each referencing a `User` author. Write the naive N+1 version (loop calling `findById` per post) and the `populate` version. Turn on `mongoose.set('debug', true)`, hit both endpoints, and count the queries logged. Add the missing index and use `.explain('executionStats')` to confirm an `IXSCAN`.

5. **(Cursor pagination)** Implement `GET /feed` with cursor pagination over `Post` sorted by `_id` descending, returning `{ items, nextCursor, hasMore }`. Fetch `limit + 1` to compute `hasMore`. Then write a small client loop that follows `nextCursor` until `hasMore` is false, and confirm it returns every document exactly once even if you insert new posts mid-pagination.

6. **(Prove it with load testing)** Take the endpoint from Exercise 2. Baseline it with `autocannon -c 100 -d 20` and record p50, p99, and req/sec. Then ensure the cache is warm and re-test. Compare the numbers and explain *why* p99 improved more than p50. Add `compression` middleware and re-test; report the change in bytes/sec and req/sec.

### Solutions

#### Solution 1 — detect and fix blocking

```js
// solution1.js
const express = require('express');
const { Worker } = require('node:worker_threads');
const { monitorEventLoopDelay } = require('node:perf_hooks');
const app = express();

// --- loop-lag monitor ---
const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();
setInterval(() => {
  console.log(
    `loop mean=${(h.mean / 1e6).toFixed(2)}ms p99=${(h.percentile(99) / 1e6).toFixed(2)}ms`
  );
  h.reset();
}, 1000);

// --- BAD: blocks the loop ---
app.get('/block', (req, res) => {
  const start = Date.now();
  while (Date.now() - start < 3000) {} // 3s of pure CPU -> freezes everything
  res.send('blocked 3s');
});

// --- GOOD: same work, off-thread ---
app.get('/block-fixed', (req, res, next) => {
  // Inline worker via eval string keeps the example self-contained.
  const worker = new Worker(
    `const { parentPort } = require('worker_threads');
     const start = Date.now(); while (Date.now() - start < 3000) {}
     parentPort.postMessage('done');`,
    { eval: true }
  );
  worker.once('message', (m) => res.send(m));
  worker.once('error', next);
});

app.get('/ping', (req, res) => res.send('pong')); // stays instant for /block-fixed

app.listen(3000, () => console.log('http://localhost:3000'));
// Test: hit /block then /ping (ping hangs, loop p99 spikes).
//       hit /block-fixed then /ping (ping stays instant, loop p99 stays low).
```

The key takeaway: the *work* is identical; moving it off the main thread is the entire fix. The loop-delay monitor makes the difference visible and quantifiable.

#### Solution 2 — cache-aside with Redis

```js
// solution2.js
const express = require('express');
const Redis = require('ioredis');
const Product = require('./models/Product'); // assume connected Mongoose model
const redis = new Redis(process.env.REDIS_URL || 'redis://127.0.0.1:6379');
const app = express();
app.use(express.json());

const TTL = 60; // seconds
const keyFor = (id) => `product:${id}`;

app.get('/products/:id', async (req, res, next) => {
  try {
    const key = keyFor(req.params.id);
    const cached = await redis.get(key);
    if (cached !== null) {
      console.log(`HIT  ${key}`);
      return res.json(JSON.parse(cached));
    }
    console.log(`MISS ${key}`);
    const product = await Product.findById(req.params.id).lean();
    if (!product) return res.status(404).json({ error: 'Not found' });
    await redis.set(key, JSON.stringify(product), 'EX', TTL); // populate with TTL
    res.json(product);
  } catch (err) { next(err); }
});

app.put('/products/:id', async (req, res, next) => {
  try {
    const updated = await Product
      .findByIdAndUpdate(req.params.id, req.body, { new: true })
      .lean();
    if (!updated) return res.status(404).json({ error: 'Not found' });
    await redis.del(keyFor(req.params.id)); // invalidate so next GET repopulates
    res.json(updated);
  } catch (err) { next(err); }
});

app.listen(3000);
// Verify:  redis-cli TTL product:<id>  -> ~60 right after a GET; -2 (gone) after a PUT.
```

First GET logs `MISS` and sets the key; subsequent GETs within 60s log `HIT`. The PUT deletes the key so the next GET is a `MISS` that repopulates with fresh data — that is the invalidation loop closed correctly, with the TTL as a backstop.

#### Solution 3 — ETag / 304 short-circuit

```js
// solution3.js
const express = require('express');
const Article = require('./models/Article');
const app = express();

app.get('/articles/:id', async (req, res, next) => {
  try {
    // Fetch only the version fields first — cheap — to build the ETag.
    const meta = await Article.findById(req.params.id).select('_id __v').lean();
    if (!meta) return res.sendStatus(404);

    const etag = `"${meta._id}-${meta.__v}"`;
    res.set('ETag', etag);
    res.set('Cache-Control', 'no-cache'); // store, but revalidate each time

    // Short-circuit BEFORE the expensive full read.
    if (req.headers['if-none-match'] === etag) {
      return res.status(304).end(); // unchanged -> no body, and we skipped the big query
    }

    const article = await Article.findById(req.params.id).lean(); // full doc only on miss
    res.json(article);
  } catch (err) { next(err); }
});

app.listen(3000);
// curl -i localhost:3000/articles/<id>                 -> 200 + ETag
// curl -i -H 'If-None-Match: "<id>-0"' .../articles/<id> -> 304, no body
// (update the article so __v/content changes) then repeat -> fresh 200 + new ETag
```

The win is twofold: the `304` saves transferring the body, *and* fetching only `_id __v` for the validator lets us skip the full document read entirely when nothing changed.

#### Solution 4 — kill the N+1

```js
// solution4.js (route handlers + verification notes)
const mongoose = require('mongoose');
mongoose.set('debug', true); // log every query so we can COUNT them

// --- N+1 version: 1 + N queries ---
app.get('/posts-naive', async (req, res, next) => {
  try {
    const posts = await Post.find().lean();          // 1 query
    for (const p of posts) {
      p.author = await User.findById(p.author).lean(); // N queries (one per post!)
    }
    res.json(posts);
  } catch (err) { next(err); }
});

// --- Fixed version: 2 queries total ---
app.get('/posts-fast', async (req, res, next) => {
  try {
    const posts = await Post.find()
      .populate('author', 'name email') // 1 extra $in query for ALL authors
      .lean();
    res.json(posts);
  } catch (err) { next(err); }
});

// Ensure the join field is indexed (Post.author references User._id, which is indexed,
// but index any field you filter posts by, e.g. createdAt):
postSchema.index({ createdAt: -1 });

// Verify the index is used:
//   const plan = await Post.find().sort({ createdAt: -1 }).explain('executionStats');
//   plan.executionStats.executionStages.stage === 'IXSCAN'  // good
```

With 100 posts, `/posts-naive` logs **101** queries in the debug output; `/posts-fast` logs **2**. That count, dropping from 1+N to 2, is the entire point — and `mongoose.set('debug', true)` is how you *see* it rather than guess.

#### Solution 5 — cursor pagination

```js
// solution5.js
app.get('/feed', async (req, res, next) => {
  try {
    const limit = Math.min(100, parseInt(req.query.limit) || 20);
    const { cursor } = req.query;
    const query = cursor ? { _id: { $lt: cursor } } : {}; // "after" the last seen _id

    const docs = await Post.find(query)
      .sort({ _id: -1 })
      .limit(limit + 1)   // fetch one extra to detect a next page
      .lean();

    const hasMore = docs.length > limit;
    const items = hasMore ? docs.slice(0, limit) : docs;
    const nextCursor = hasMore ? String(items[items.length - 1]._id) : null;
    res.json({ items, nextCursor, hasMore });
  } catch (err) { next(err); }
});
```

```js
// client loop that walks the whole feed
async function drainFeed(base) {
  let cursor = null, all = [];
  do {
    const url = new URL(`${base}/feed`);
    url.searchParams.set('limit', '20');
    if (cursor) url.searchParams.set('cursor', cursor);
    const { items, nextCursor, hasMore } = await (await fetch(url)).json();
    all.push(...items);
    cursor = nextCursor;
    if (!hasMore) break;
  } while (cursor);
  // Each _id appears exactly once even if new posts (larger _ids) are inserted
  // mid-pagination, because we always page DOWN from the last seen _id.
  return all;
}
```

Because each page anchors to a real, monotonic `_id` rather than a positional offset, inserting new posts during pagination cannot cause skips or duplicates — newly inserted documents have larger `_id`s and simply fall outside the `_id < cursor` window we are walking. This is exactly the stability that offset pagination lacks.

#### Solution 6 — prove it with load testing

```bash
# 1. Baseline (cold cache / or before caching existed).
autocannon -c 100 -d 20 http://localhost:3000/products/123 > before.txt

# 2. Warm the cache, then re-test with the cache-aside endpoint hot.
curl -s http://localhost:3000/products/123 > /dev/null   # warm it
autocannon -c 100 -d 20 http://localhost:3000/products/123 > after.txt

# 3. Add `app.use(require('compression')())` to the server, restart, re-test.
autocannon -c 100 -d 20 http://localhost:3000/products/123 > after-gzip.txt
```

**Expected results and the explanation:**

- **req/sec rises sharply** once the cache is warm: each request now returns from Redis (sub-ms) instead of hitting MongoDB, so the per-request cost collapses and throughput climbs.
- **p99 improves more than p50.** The median request was probably already reasonably fast; the *tail* was dominated by slow DB queries, lock contention, or GC under load. Caching removes that variable DB work from the hot path, so the worst-case requests (the tail) tighten dramatically — which is exactly why you read p99, not the average.
- **After `compression`:** `bytes/sec` drops substantially for the JSON payload (the body is 70–90% smaller on the wire), while req/sec stays similar or dips slightly on a same-box test (compression is CPU on the main thread). The lesson: compression is a *bandwidth* win, not a CPU win — and at scale you offload it to the proxy/CDN so it does not compete with request handling.

## Summary / Cheat Sheet

| Area | Do this | Tool / API |
| --- | --- | --- |
| Event loop | Never block it; offload CPU work | Worker threads, `piscina`, cluster/PM2 |
| Detect blocking | Watch loop lag; locate blocks | `perf_hooks.monitorEventLoopDelay`, `blocked-at` |
| Profile CPU | Find the hot path via flamegraph | `node --prof` + `--prof-process`, `--inspect` + DevTools, `clinic flame`, `0x` |
| Triage | Identify the *kind* of problem first | `clinic doctor` |
| Memory leaks | Diff heap snapshots; follow retainers | `--inspect` Memory tab, `v8.writeHeapSnapshot`, `--heapsnapshot-near-heap-limit` |
| Leak sources | Bound caches, slim closures, clear timers, remove listeners | `lru-cache`, `emitter.off`, `clearInterval` |
| In-process cache | Small hot data / L1; always bounded | `lru-cache` (max + ttl) |
| Distributed cache | Shared, consistent across instances | Redis via `ioredis` / `node-redis` |
| Cache pattern | Cache-aside: get -> miss -> load -> set(TTL) | `redis.get` / `redis.set ... 'EX'` |
| Invalidation | TTL backstop + `DEL` on writes; tags for groups | `redis.del`, key-set tags, `SCAN` (not `KEYS`) |
| Stampede | Lock / stale-while-revalidate / jittered TTL | `SET ... NX EX` |
| HTTP caching | `immutable` for fingerprinted assets; `no-cache` HTML | `Cache-Control`, `ETag`, `304` |
| `no-cache` vs `no-store` | Revalidate vs. never store | — |
| Compression | Compress text; offload at scale; skip compressed | `compression` middleware, nginx/CDN |
| Indexes | Index filters/sorts (ESR order); verify | `schema.index()`, `.explain('executionStats')` |
| Reads | Plain objects + only needed fields | `.lean()`, `.select()` / projection |
| N+1 | Batch related lookups | `populate`, `$lookup`, `mongoose.set('debug', true)` |
| Pooling | Connect once; size vs. DB limit × instances | `maxPoolSize`, `minPoolSize` |
| Pagination | Cap limit; cursor for big/active, offset for admin | `_id < cursor` + `sort` + `limit+1` |
| Load testing | Baseline -> change one thing -> re-test; read p99 + errors | `autocannon -c -d`, `k6`/`artillery` for scenarios |

**The one-paragraph distillation:** Node is single-threaded, so the prime directive is *never block the event loop* — offload CPU work and detect blocking with loop-lag metrics. Beyond that, the biggest wins come from *not doing work twice* (cache aside with Redis, HTTP caching, in-process LRUs — all with TTLs and invalidation) and from *not making the database work harder than it must* (indexes, `lean()`, projection, killing N+1, pooling, pagination). And the meta-rule binding it all together: **profile and load-test to find the real bottleneck and to prove your fix worked — measure, change one thing, measure again.**
