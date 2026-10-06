# The Event Loop, Timers & EventEmitter

The event loop is the single most important concept in Node.js. It is *the* mechanism that lets a single-threaded JavaScript runtime handle thousands of concurrent connections without blocking. If you understand the event loop deeply, almost every "weird" Node.js behavior — surprising log ordering, mysterious latency spikes, callbacks that "never fire", `process.nextTick` recursion crashing your server — stops being mysterious and becomes predictable.

This module builds the model from first principles. By the end you should be able to read any snippet that mixes `setTimeout`, `setImmediate`, `process.nextTick`, and Promises and predict the exact output order — and explain *why* — which is one of the most common Node.js interview exercises.

## What You'll Learn

- What the event loop actually is, and why Node needs one to deliver concurrency on a single thread.
- The six phases of the libuv event loop in order — **timers → pending callbacks → idle/prepare → poll → check → close callbacks** — and what runs in each.
- The two microtask queues — `process.nextTick` and the Promise job queue — and their relative priority.
- The precise semantics and ordering rules of `setTimeout`, `setInterval`, `setImmediate`, and `process.nextTick`.
- How to cancel timers (`clearTimeout`, `clearInterval`, `clearImmediate`) and how `ref()` / `unref()` control whether a timer keeps the process alive.
- The `EventEmitter` class — `on`, `once`, `emit`, `off`, error handling, memory-leak warnings — as the backbone of event-driven Node.
- How to confidently solve classic output-prediction puzzles, step by step.

---

## Part 1 — Why an Event Loop at All?

### The blocking problem

Imagine a naive server that reads a file for every request:

```js
// Pseudo-blocking server (NOT how Node works) — illustration only
const data = readFileSync('big.json'); // CPU sits idle for ~10ms waiting on disk
respond(data);
```

A disk read might take 10 milliseconds. During those 10ms the CPU does *nothing useful* — it just waits. If your process is single-threaded and you call this synchronously, a second request that arrives during those 10ms has to wait for the first to finish. With blocking I/O, the only way to serve many clients at once is to spawn a thread (or process) per client. That model — used by traditional servers like Apache's prefork mode — costs memory (each thread needs its own stack) and CPU (context switching between thousands of threads is expensive).

### Node's answer: non-blocking I/O + an event loop

Node flips the model. Instead of *waiting* for I/O, it *registers a callback* and immediately moves on to other work. The operating system (or a background thread pool) does the slow part; when the result is ready, the OS notifies Node, and Node runs your callback.

```js
const fs = require('node:fs');

// Non-blocking: returns immediately, callback runs later when data is ready
fs.readFile('big.json', (err, data) => {
  if (err) throw err;
  respond(data); // runs ~10ms later, but we never blocked the thread
});

console.log('I run FIRST, before the file is read'); // proves we did not block
```

The **event loop** is the orchestrator of this scheme. It is an infinite loop that, on each iteration ("tick"), asks: *"Is there any completed I/O whose callback I should run? Any expired timers? Any `setImmediate` callbacks?"* It picks up ready work, runs the associated JavaScript, and loops again. When there is genuinely nothing left to do — no pending timers, no open sockets, no scheduled callbacks — the loop exits and the process ends.

> TIP: "Single-threaded" refers to **your JavaScript**. Node itself is multi-threaded under the hood. libuv maintains a thread pool (default 4 threads, set via `UV_THREADPOOL_SIZE`) for operations that have no async OS primitive — file system calls, DNS lookups via `dns.lookup`, and CPU-bound crypto like `crypto.pbkdf2`. Network I/O does *not* use the thread pool; it uses efficient OS event notification (epoll on Linux, kqueue on macOS, IOCP on Windows).

### How concurrency emerges on one thread

Concurrency here means *making progress on many tasks during overlapping time windows*, not *running code simultaneously*. Your JS callbacks never run in parallel — exactly one runs at a time, to completion, with no preemption. But while one request's callback is parked waiting on the database, the event loop is free to run another request's callback. The waiting happens off-thread (in the OS or thread pool); your single JS thread is only ever busy doing actual computation.

> WARNING — Pitfall: Because there is no preemption, **a single long-running synchronous function blocks the entire process.** A `while` loop that runs for 2 seconds, a synchronous `JSON.parse` of a huge payload, or a `bcrypt.hashSync` call freezes *every* connection for that whole duration. The event loop cannot interrupt running JS. This is why CPU-bound work belongs in worker threads or a separate service — covered later in the course.

---

## Part 2 — The Phases of the Event Loop

The event loop is implemented by **libuv** (the C library underpinning Node). Each full iteration of the loop ("tick") moves through a fixed sequence of **phases**. Each phase has a FIFO queue of callbacks. When the loop enters a phase, it runs callbacks in that phase's queue until the queue is empty *or* a system-dependent maximum is hit, then moves to the next phase.

Here are the six phases, **in order**:

```
   ┌───────────────────────────┐
┌─>│           timers          │  setTimeout / setInterval callbacks whose
│  └─────────────┬─────────────┘  threshold has elapsed
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  some system-level callbacks deferred from a
│  └─────────────┬─────────────┘  previous loop (e.g. certain TCP errors)
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  internal to libuv/Node only
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐      ┌───────────────┐
│  │           poll            │<─────┤  incoming:    │  retrieve new I/O events;
│  └─────────────┬─────────────┘      │  connections, │  run I/O callbacks; may
│  ┌─────────────┴─────────────┐      │  data, etc.   │  BLOCK here waiting for I/O
│  │           check           │      └───────────────┘
│  └─────────────┬─────────────┘  setImmediate callbacks
│  ┌─────────────┴─────────────┐
└──┤      close callbacks       │  'close' events: socket.on('close', ...) etc.
   └───────────────────────────┘
```

### Phase-by-phase

| Phase | What runs here | Notes |
|-------|----------------|-------|
| **timers** | Callbacks scheduled by `setTimeout` / `setInterval` whose delay has elapsed | The delay is a *minimum*, not a guarantee. The OS scheduler and other phases can push execution later. |
| **pending callbacks** | A handful of system-operation callbacks deferred from the previous tick | E.g. some types of TCP socket errors (`ECONNREFUSED`). Rarely relevant to app code. |
| **idle, prepare** | Internal bookkeeping only | You cannot schedule into these from JS. |
| **poll** | Retrieves new I/O events and runs their callbacks (file reads, network data, etc.) | The "heart" of the loop. This is the phase that can **block**, waiting for I/O if there's nothing else to do. See the poll logic below. |
| **check** | `setImmediate` callbacks | Named "check" because libuv calls it the *check* handle. `setImmediate` is specifically designed to run *right after* the poll phase. |
| **close callbacks** | `'close'` event callbacks, e.g. `socket.on('close', ...)`, abrupt destroys | If a socket/handle is closed abruptly (`socket.destroy()`), its `close` event is emitted here. |

### The crucial poll-phase logic

The poll phase has special branching logic that explains a lot of timer behavior. When the loop reaches **poll**:

1. If the poll queue has callbacks, run them synchronously (FIFO) until the queue empties or a hard limit is reached.
2. When the poll queue is empty:
   - If there are `setImmediate` callbacks scheduled (i.e. **check** has work waiting), the loop **ends the poll phase immediately** and advances to **check** to run them.
   - Otherwise, if there are **timers** due soon, the loop calculates how long until the nearest timer and **blocks here waiting for I/O for at most that long**, then wraps around to the timers phase.
   - If there is neither pending I/O nor scheduled callbacks, the loop may block indefinitely waiting for new I/O — or, if nothing can ever wake it, exit.

This is why "the loop can sleep" — when idle, Node is not burning CPU spinning; it's parked in the poll phase letting the OS wake it when a socket has data.

> TIP: A useful mental shorthand — `setTimeout`/`setInterval` run in **timers** (at the *top* of the loop), `setImmediate` runs in **check** (near the *bottom*). So within one I/O callback, `setImmediate` reliably fires before a `setTimeout(…, 0)` because you're already past the timers phase and check comes next on this same tick, whereas the timer has to wait for the *next* tick's timers phase.

### Microtasks run *between* phase callbacks, not in a phase

This is the part most explanations get fuzzy about. The two **microtask queues** — `process.nextTick` and the Promise job queue — are **not** event-loop phases. They are drained:

- After the currently executing JS callback finishes (returns to the loop), **before** the loop moves to the next callback, and
- After the top-level (main module) script finishes its initial synchronous run.

In practice: **after every single callback the loop runs (in any phase), Node fully drains the `nextTick` queue, then fully drains the Promise microtask queue, before running the next callback.** We'll make this concrete in Part 4.

---

## Part 3 — The Timer & Scheduling APIs

There are four scheduling primitives. Three are "real" timer-ish APIs; `process.nextTick` is a microtask scheduler that is named confusingly (it has nothing to do with timers).

### `setTimeout(callback, delay, ...args)`

Schedules `callback` to run after **at least** `delay` milliseconds, in the **timers** phase.

```js
// Runs after ~1000ms (a minimum — could be later under load)
const id = setTimeout((name) => {
  console.log(`Hello, ${name}`); // extra args are forwarded to the callback
}, 1000, 'Ada');

// Cancel it before it fires:
clearTimeout(id);
```

Key facts:

- `delay` is a **minimum** delay. If the event loop is busy or other phases run long, the callback fires later. There is no upper bound guarantee.
- If `delay` is `0`, omitted, larger than `2147483647` (~24.8 days), or not a number, it is coerced to `1` ms. So `setTimeout(fn, 0)` is effectively `setTimeout(fn, 1)`. This coercion detail is the reason `setTimeout(fn, 0)` vs `setImmediate(fn)` ordering at the top level is **non-deterministic** (see Part 5).
- The return value in Node is a **`Timeout` object**, not a number (unlike browsers, which return a numeric id). You can call `.ref()`, `.unref()`, and (Node 11+) `.refresh()` on it.

### `setInterval(callback, delay, ...args)`

Repeatedly runs `callback` every (at least) `delay` ms, in the timers phase, until cleared.

```js
let count = 0;
const id = setInterval(() => {
  console.log(`tick ${++count}`);
  if (count === 5) clearInterval(id); // stop after 5 ticks
}, 1000);
```

> WARNING — Pitfall: `setInterval` does **not** wait for your callback to finish before scheduling the next run, and it does not "make up" for drift. If your callback itself takes longer than the interval, runs queue up and can overlap conceptually (though they still execute one at a time). For long-running async work, prefer a self-scheduling `setTimeout` chain so each run starts only after the previous completes:

```js
// Drift-resistant, no-overlap recurring task
async function poll() {
  await doWork();                 // however long this takes...
  setTimeout(poll, 1000);         // ...we wait 1s AFTER it finishes
}
poll();
```

### `setImmediate(callback, ...args)`

Schedules `callback` to run in the **check** phase — i.e. immediately *after* the current poll phase completes, on the same or next tick.

```js
setImmediate(() => {
  console.log('runs in the check phase');
});
```

Despite the name, `setImmediate` is **not** the most immediate option — `process.nextTick` runs sooner. The name reflects its design intent: run *as soon as possible after the current poll phase*, which is the natural place to defer work without starving I/O.

> BEST PRACTICE: When you want to "yield" to let pending I/O process and then continue, use `setImmediate`, not `setTimeout(fn, 0)`. `setImmediate` is purpose-built for this, doesn't suffer the `0 → 1ms` coercion, and won't be starved by other timers.

### `process.nextTick(callback, ...args)`

Schedules `callback` to run in the **nextTick microtask queue**, which is drained **after the current operation completes but before the event loop continues to any phase or to the Promise queue.** It is *not* a timer and *not* part of the phase machinery.

```js
process.nextTick(() => {
  console.log('runs before any I/O, timer, or Promise callback');
});
```

`process.nextTick` has the **highest priority** of all the deferral mechanisms. The entire nextTick queue is emptied before Node even looks at the Promise microtask queue, and long before any event-loop phase runs.

> WARNING — Pitfall: Because the nextTick queue is fully drained before the loop proceeds, **recursively calling `process.nextTick` starves the event loop entirely** — timers never fire, I/O never completes, the process appears hung (and memory grows). This is "I/O starvation":

```js
// DO NOT DO THIS — the loop never advances; setTimeout below NEVER fires.
function loopForever() {
  process.nextTick(loopForever);
}
loopForever();
setTimeout(() => console.log('I will never run'), 100);
```

A recursive `setImmediate` does **not** have this problem, because each `setImmediate` callback runs in a *separate* check phase, with the rest of the loop (timers, poll) getting a turn in between. So for recursion or chunked work, prefer `setImmediate`.

### Comparison of the four primitives

| API | Queue / phase | Priority (when all scheduled together) | Repeats? | Use it for… |
|-----|---------------|----------------------------------------|----------|-------------|
| `process.nextTick` | nextTick microtask queue | **1 (highest)** — before everything | No | Deferring to *just after* the current function, e.g. emitting an event after a constructor returns; running cleanup before the loop continues. |
| `Promise.then` / `queueMicrotask` | Promise microtask queue | **2** — after nextTick, before phases | No | Normal async/await flow; promise continuations. |
| `setImmediate` | check phase | **3** — after the poll phase | No | Yielding to I/O then continuing; chunking CPU work without starving the loop. |
| `setTimeout(fn, 0)` | timers phase | **4 (lowest of these)** — next loop's timers phase | No | "Run later, roughly after this delay." Avoid `0` for "ASAP" — use `setImmediate`. |
| `setInterval` | timers phase | (recurring) | Yes | Periodic tasks (with the drift caveat above). |

---

## Part 4 — Microtasks in Detail: nextTick vs Promise

There are **two** microtask queues, and their ordering is a guaranteed favorite interview question.

### The two queues and their priority

1. **`process.nextTick` queue** — drained first, completely.
2. **Promise microtask queue** (also fed by `queueMicrotask()` and `async/await` continuations) — drained second, completely.

The rule Node follows after every callback (and after the initial synchronous script):

> Drain the **entire** nextTick queue. Then drain the **entire** Promise microtask queue. If draining the Promise queue scheduled new nextTicks, drain those too — keep alternating until **both** queues are empty. Only then proceed to the next loop callback / phase.

Let's see it:

```js
console.log('1: sync start');

process.nextTick(() => console.log('4: nextTick'));
Promise.resolve().then(() => console.log('5: promise'));

console.log('2: sync end');
// queueMicrotask is equivalent to Promise.resolve().then for ordering:
queueMicrotask(() => console.log('6: queueMicrotask (promise queue)'));

process.nextTick(() => console.log('3 will actually be... see below'));
```

Actual output:

```text
1: sync start
2: sync end
4: nextTick
3 will actually be... see below
5: promise
6: queueMicrotask (promise queue)
```

Reasoning: All synchronous `console.log`s run first (`1`, `2`). Then the main script finishes, so microtasks drain. **nextTick queue first**, in FIFO order: the `4` callback, then the second nextTick (`3 will actually be…`). Then the **Promise queue** in FIFO: `5`, then `6`. The labels in code are intentionally misleading to force you to reason from the rules, not the numbers.

### Interleaving: when microtasks schedule more microtasks

```js
Promise.resolve().then(() => {
  console.log('A: promise');
  process.nextTick(() => console.log('B: nextTick scheduled inside promise'));
});

process.nextTick(() => {
  console.log('C: nextTick');
  Promise.resolve().then(() => console.log('D: promise scheduled inside nextTick'));
});

setTimeout(() => console.log('E: timeout'), 0);
```

Output (verified):

```text
C: nextTick
A: promise
D: promise scheduled inside nextTick
B: nextTick scheduled inside promise
E: timeout
```

Walkthrough:

1. Main script schedules: nextTick `C`, promise `A`, timer `E`.
2. Script ends → drain nextTick queue: run `C`. Inside `C`, a *promise* `D` is queued.
3. nextTick queue now empty → drain Promise queue: run `A` (it was first), then `D`. While running `A`, a *nextTick* `B` gets queued.
4. After the Promise queue, Node re-checks: the nextTick queue is non-empty again (`B`) → drain it: run `B`.
5. Both microtask queues empty → loop proceeds to the **timers** phase: run `E`.

> TIP: The mental model: **"nextTick beats Promises; both beat the loop; and after running any microtask, re-check nextTick before continuing."** Node keeps ping-ponging between the two microtask queues until both are empty.

### `queueMicrotask()` vs `process.nextTick()`

`queueMicrotask()` is the standardized (cross-platform, also in browsers) way to schedule a microtask. It enqueues into the **Promise queue**, so it has the same priority as `Promise.resolve().then`. Prefer it over `process.nextTick` in library code unless you specifically need nextTick's higher priority — using `nextTick` unnecessarily risks the starvation pitfall and is a Node-only API.

---

## Part 5 — Ordering Rules with Worked Examples

### Top-level `setTimeout(fn, 0)` vs `setImmediate` is non-deterministic

```js
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
```

Run this repeatedly and the order may **flip between runs**. Why? At program start, the event loop must enter the timers phase to run the `setTimeout`. But whether the 1ms (coerced from 0) timer has *already elapsed* by the time the loop reaches the timers phase depends on how long process startup took — a few microseconds of jitter. If 1ms has elapsed, `timeout` wins; if not, the loop sails past timers into check and `immediate` wins. **This race makes the order unguaranteed at the top level.** Never write code that depends on it.

### Inside an I/O callback the order IS deterministic

```js
const fs = require('node:fs');

fs.readFile(__filename, () => {
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate'));
});
```

Output is **always**:

```text
immediate
timeout
```

Why deterministic now? The `fs.readFile` callback runs in the **poll** phase. From inside poll, the very next phase is **check**, so `setImmediate` runs on this same tick. The `setTimeout` must wait for the *next* loop iteration's timers phase. Hence `immediate` always precedes `timeout` here. This contrast (random at top level, fixed inside I/O) is a classic interview "gotcha."

### The grand unified example

```js
console.log('=== sync start ===');

setTimeout(() => console.log('setTimeout 0'), 0);
setImmediate(() => console.log('setImmediate'));

Promise.resolve().then(() => console.log('Promise.then'));
process.nextTick(() => console.log('process.nextTick'));

(async () => {
  console.log('async fn sync part'); // runs synchronously up to the first await
  await null;                        // everything after await is a microtask (Promise queue)
  console.log('after await');
})();

console.log('=== sync end ===');
```

Output:

```text
=== sync start ===
async fn sync part
=== sync end ===
process.nextTick
Promise.then
after await
setImmediate
setTimeout 0
```

Step by step:

1. **Synchronous run, top to bottom.** `=== sync start ===` prints. Timers/immediate/promise/nextTick get scheduled (they don't run yet). The async IIFE runs *synchronously up to* `await null`, printing `async fn sync part`; the part after `await` becomes a Promise-queue microtask. `=== sync end ===` prints.
2. **Microtasks drain.** nextTick first: `process.nextTick`. Then Promise queue (FIFO): `Promise.then` was queued before the async continuation, so `Promise.then`, then `after await`.
3. **Event loop phases.** Timers phase: is the `setTimeout` due? At top level this races with check, but in practice on a freshly started loop the immediate typically wins — and more importantly, `setImmediate` is in the very next phase. Here `setImmediate` prints, then on the next iteration `setTimeout 0`. (If you saw the timer first on some run, re-read the non-determinism note — but with `setImmediate` also present and a busy startup, immediate-then-timeout is the common, expected result.)

> TIP: When solving these by hand, always process in this exact order: **(1) all synchronous code → (2) nextTick queue → (3) Promise microtask queue → (4) timers phase → (5) check phase (`setImmediate`) → (6) close callbacks → repeat.** And remember to re-drain both microtask queues *after every single callback* in steps 4–6.

---

## Part 6 — Cancelling Timers, `ref` / `unref`

### Cancelling

Every scheduler returns a handle you can cancel:

```js
const t = setTimeout(() => console.log('nope'), 5000);
clearTimeout(t); // never fires

const iv = setInterval(() => console.log('tick'), 1000);
clearInterval(iv); // stops the repetition

const im = setImmediate(() => console.log('nope'));
clearImmediate(im); // cancels the check-phase callback

// process.nextTick CANNOT be cancelled — once scheduled, it WILL run.
```

> WARNING — Pitfall: There is no `clearNextTick`. Once you call `process.nextTick`, the callback is committed. Plan accordingly — guard with a boolean flag inside the callback if you might need to "cancel" the effect.

`clearTimeout` and `clearInterval` are interchangeable in Node (both just remove the timer), but use the matching one for readability. Calling `clear*` on an already-fired or already-cleared handle is a harmless no-op.

### `refresh()` (Node 11+)

You can restart a timer's countdown without creating a new object — handy for idle-timeout patterns:

```js
const idleTimer = setTimeout(() => console.log('idle for 30s, closing'), 30_000);

function onActivity() {
  idleTimer.refresh(); // restart the 30s countdown from now
}
```

### `ref()` and `unref()`: keeping the process alive (or not)

By default, an **active timer keeps the Node process alive** — the event loop won't exit while a timer is pending. This is usually what you want. But sometimes you have a *background* timer (a metrics flush, a watchdog) that should **not** prevent the program from exiting once the real work is done.

- `timer.unref()` — tell the loop: *"don't count this timer when deciding whether to keep running."* If it's the only thing left, the process exits and the callback never fires.
- `timer.ref()` — undo `unref()`; the timer counts again and will keep the process alive.

```js
// Watchdog that should not, by itself, hold the process open
const watchdog = setInterval(() => checkHealth(), 10_000);
watchdog.unref();

// The process will exit naturally when the main work finishes,
// even though this interval is technically still "scheduled".
```

Demonstration of the exit behavior:

```js
const t = setTimeout(() => console.log('should NOT print'), 1000);
t.unref();                          // do not keep the process alive for this timer
console.log('exiting without waiting');
// Output: only "exiting without waiting" — the process exits before 1000ms,
// so the timer callback never runs.
```

> BEST PRACTICE: Use `unref()` for any periodic background task (heartbeats, cache sweeps, metrics) so it never becomes the reason a CLI tool or short-lived job hangs instead of exiting. The same `ref`/`unref` pattern works on sockets and other libuv handles, not just timers.

> TIP: `setImmediate` handles also support `ref`/`unref`. `server.listen()` returns a server you can `unref()` too — useful when a server should only run as long as some other handle is alive.

---

## Part 7 — The `EventEmitter`: Backbone of Event-Driven Node

The event loop *delivers* events; `EventEmitter` is the standard pattern for *your code* to produce and consume named events. It's the publish/subscribe (observer) pattern, built into Node's `node:events` module. An enormous amount of core Node is built on it: HTTP servers (`server.on('request', …)`), streams (`stream.on('data', …)`, `'end'`, `'error'`), sockets, the `process` object itself (`process.on('exit', …)`), and countless libraries.

### The basics: `on`, `once`, `emit`

```js
const EventEmitter = require('node:events');

const bus = new EventEmitter();

// Subscribe: run this every time 'order' is emitted
bus.on('order', (item, qty) => {
  console.log(`Got order: ${qty} x ${item}`);
});

// Subscribe ONCE: auto-removed after the first emit
bus.once('shutdown', () => {
  console.log('Shutting down (this listener fires at most once)');
});

// Publish: synchronously invokes all 'order' listeners, in registration order
bus.emit('order', 'coffee', 2);   // → Got order: 2 x coffee
bus.emit('order', 'tea', 1);      // → Got order: 1 x tea

bus.emit('shutdown');             // → Shutting down ...
bus.emit('shutdown');             // (nothing — the once listener is gone)
```

Three things to internalize:

1. **`emit` is synchronous.** When you call `emit`, every registered listener runs *immediately and in order*, on the current call stack, before `emit` returns. Listeners are not deferred to the event loop. (This trips people up — "events" sounds async, but emission is synchronous.)
2. **Listeners run in registration order**, FIFO.
3. **`once` listeners remove themselves** after firing once.

### Building your own emitter via inheritance

The idiomatic way to give a class event capabilities is to extend `EventEmitter`:

```js
const EventEmitter = require('node:events');

class Job extends EventEmitter {
  start() {
    this.emit('start');
    // ... do work ...
    setImmediate(() => this.emit('done', { ok: true })); // emit later, asynchronously
  }
}

const job = new Job();
job.on('start', () => console.log('job started'));
job.on('done', (result) => console.log('job done:', result));
job.start();
// → job started
// → job done: { ok: true }
```

> TIP: A common idiom inside constructors or factory functions is to emit an event *after* the caller has had a chance to attach listeners. Since `emit` is synchronous, emitting directly in a constructor would fire before any `.on()` call could run. Defer it with `process.nextTick(() => this.emit('ready'))` so listeners attached on the returned instance still catch it. This is one of the few legitimate, idiomatic uses of `process.nextTick`.

### Removing listeners

```js
function onData(chunk) { /* ... */ }

bus.on('data', onData);
bus.off('data', onData);            // remove a specific listener (Node 10+ alias of removeListener)
bus.removeListener('data', onData); // older but equivalent name

bus.removeAllListeners('data');     // remove every 'data' listener
bus.removeAllListeners();           // remove ALL listeners for ALL events (use with care)
```

> WARNING — Pitfall: You can only remove a listener if you pass the **same function reference** you registered. An inline arrow function (`bus.on('x', () => …)`) can never be removed by reference — you'd have to use `removeAllListeners`. For removable listeners, store the function in a named variable.

### The special `'error'` event

This is the single most important `EventEmitter` rule for production stability:

> If an `EventEmitter` emits an `'error'` event and there is **no** listener registered for `'error'`, Node **throws** the error. If the throw is uncaught, it **crashes the process**.

```js
const ee = new EventEmitter();
ee.emit('error', new Error('boom'));
// No 'error' listener → this throws and (if unhandled) crashes the process.
```

Always attach an `'error'` listener to any emitter that might emit one — streams, sockets, child processes, etc.:

```js
ee.on('error', (err) => {
  console.error('Handled emitter error:', err.message);
});
ee.emit('error', new Error('boom')); // → Handled emitter error: boom (no crash)
```

> BEST PRACTICE: Treat `'error'` listeners as mandatory, not optional. A missing `'error'` handler on a stream or socket is one of the most common causes of mysterious production crashes.

### Other useful members

```js
bus.eventNames();              // → array of event names with active listeners
bus.listenerCount('order');    // → number of listeners for 'order'
bus.listeners('order');        // → array of the listener functions
bus.prependListener('x', fn);  // add fn to the FRONT of the listener list
bus.prependOnceListener('x', fn);

// Static helpers:
const { once, on } = require('node:events');
// once() returns a Promise that resolves when the event fires — great with async/await:
async function waitForReady(emitter) {
  const [arg] = await once(emitter, 'ready'); // resolves with emitted args as an array
  return arg;
}
```

> TIP: `events.once(emitter, eventName)` is a clean way to `await` a single event. It also rejects if the emitter fires `'error'` first, which makes async control flow much safer than manually juggling listeners.

### The `MaxListenersExceededWarning`

By default, adding **more than 10** listeners for the *same* event on the *same* emitter prints a warning to stderr:

```text
(node:12345) MaxListenersExceededWarning: Possible EventEmitter memory leak
detected. 11 order listeners added to [EventEmitter]. Use emitter.setMaxListeners()
to increase limit
```

This is a **leak-detection heuristic**, not an error. It usually means you're adding listeners in a loop or per-request without ever removing them. Investigate before silencing. If 10 is genuinely too low for your legitimate use case, raise it explicitly:

```js
bus.setMaxListeners(50);                 // per-emitter limit
require('node:events').defaultMaxListeners = 20; // global default (use sparingly)
bus.setMaxListeners(0);                   // 0 or Infinity = unlimited (disables the check)
```

> WARNING — Pitfall: Don't reflexively bump `setMaxListeners` to silence the warning. Nine times out of ten it's a real leak — e.g. attaching a `process.on('SIGINT', …)` or a `socket.on('data', …)` inside a request handler that runs thousands of times. Fix the lifecycle (remove listeners, or use `once`) instead.

---

## Common Pitfalls & Gotchas

- **Blocking the loop with synchronous CPU work.** A long `for` loop, `JSON.parse` on a huge string, `fs.readFileSync`, or `*.Sync` crypto freezes *every* connection. Offload to worker threads or chunk the work with `setImmediate`.
- **Assuming `setTimeout(fn, 0)` runs "immediately."** It runs in the *next* timers phase at the earliest, after a forced ≥1ms delay. For "ASAP," use `setImmediate` (yield to I/O) or `queueMicrotask`/`process.nextTick` (before the loop continues).
- **Relying on top-level `setTimeout(0)` vs `setImmediate` ordering.** It's a race and flips between runs. Only inside an I/O callback is the order (`setImmediate` first) guaranteed.
- **Recursive `process.nextTick` starving the loop.** Timers and I/O never get a turn. Use recursive `setImmediate` for self-scheduling work instead.
- **Forgetting `process.nextTick` and Promise jobs are different queues.** nextTick *always* wins. Mixing them up is the #1 cause of wrong answers on output-prediction questions.
- **Treating `emit` as asynchronous.** It is synchronous — all listeners run before `emit` returns. If you need async, defer the emit yourself (`setImmediate` / `process.nextTick`).
- **Emitting `'error'` with no `'error'` listener.** This *throws* and can crash the process. Always attach one.
- **Un-removable inline-arrow listeners** leaking memory because you can't `off()` them by reference.
- **`setInterval` drift / overlap** when the callback is slower than the interval. Use a self-scheduling `setTimeout` chain.
- **Background timers preventing exit.** A pending `setInterval` keeps a CLI process alive forever. `unref()` background timers.
- **`MaxListenersExceededWarning` ignored or silenced** rather than treated as the leak signal it usually is.

## Best Practices

- **Keep callbacks short and non-blocking.** The loop can only make progress when JS isn't running. Push CPU-heavy work off-thread.
- **Use `setImmediate` to yield**, not `setTimeout(0)`. It's purpose-built, avoids the `0→1ms` coercion, and won't be starved by other timers.
- **Reserve `process.nextTick` for true "after this function, before the loop" needs** — most often emitting an event so listeners attached to a just-returned object still fire. Otherwise prefer `queueMicrotask`.
- **Never recurse with `process.nextTick`.** Recurse with `setImmediate` so I/O and timers still get serviced.
- **Always handle `'error'` events** on every emitter, stream, and socket.
- **Store listener references** you might remove; prefer `once()` for one-shot events to avoid leaks.
- **`unref()` background timers** (heartbeats, watchdogs, metrics) so they don't block clean process exit.
- **Prefer a self-scheduling `setTimeout` chain over `setInterval`** for async periodic work to avoid overlap and drift.
- **Don't depend on timer precision.** Delays are minimums; never use timers for hard real-time guarantees.
- **Treat `MaxListenersExceededWarning` as a bug to investigate**, not noise to suppress.

## Key Interview Points

- **"What is the event loop?"** A single-threaded loop (implemented by libuv) that repeatedly processes phases of queued callbacks, enabling non-blocking concurrency: while one task waits on off-thread I/O, the loop runs other ready callbacks. Exactly one JS callback runs at a time, to completion, with no preemption.
- **"Name the event loop phases in order."** timers → pending callbacks → idle/prepare → poll → check → close callbacks. App-relevant ones: timers (`setTimeout`/`setInterval`), poll (I/O callbacks, can block), check (`setImmediate`), close (`'close'` events).
- **"`process.nextTick` vs Promise microtask priority?"** nextTick queue is drained *completely first*, then the Promise queue. Both fully drain (re-checking nextTick) before the loop advances to any phase.
- **"`setImmediate` vs `setTimeout(0)`?"** At the top level the order is non-deterministic (race during startup). Inside an I/O callback, `setImmediate` always runs first because the check phase immediately follows the poll phase, while the timer waits for the next tick.
- **"Is `setTimeout` precise?"** No — `delay` is a *minimum*. The callback runs in the next timers phase at or after the delay; a busy loop pushes it later. `0` is coerced to `1`ms.
- **"How does single-threaded Node do concurrency?"** By offloading slow I/O to the OS / libuv thread pool and registering callbacks, so the JS thread is only ever busy doing computation, never waiting.
- **"Why can recursive `process.nextTick` hang the server?"** The nextTick queue is fully drained before the loop proceeds; infinitely re-adding to it means the loop never reaches the I/O or timers phases. `setImmediate` recursion is safe.
- **"Is `EventEmitter.emit` synchronous or async?"** Synchronous — listeners fire in registration order on the current stack before `emit` returns.
- **"What happens if an emitter emits `'error'` with no listener?"** Node throws; if uncaught, the process crashes. Always register an `'error'` handler.
- **"What does `unref()` do?"** Removes a timer/handle from the set that keeps the process alive, so the loop can exit even if that timer is still pending.
- **"`on` vs `once`?"** `on` fires every emit; `once` fires at most once then auto-removes itself.
- **"What's the `MaxListenersExceededWarning`?"** A leak-detection warning printed when >10 listeners are added for one event on one emitter; usually signals a real lifecycle leak.

## Practice Exercises

Try to predict the output *before* running each. Then run it to check.

**Exercise 1 — Basic ordering.** Predict the exact output:

```js
console.log('A');
setTimeout(() => console.log('B'), 0);
Promise.resolve().then(() => console.log('C'));
process.nextTick(() => console.log('D'));
console.log('E');
```

**Exercise 2 — I/O phase ordering.** Predict the output and explain *why it's deterministic here* (unlike at the top level):

```js
const fs = require('node:fs');
fs.readFile(__filename, () => {
  console.log('read done');
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate'));
  process.nextTick(() => console.log('nextTick'));
});
```

**Exercise 3 — Microtask interleaving.** Predict the output:

```js
process.nextTick(() => {
  console.log('1');
  Promise.resolve().then(() => console.log('2'));
});
Promise.resolve().then(() => {
  console.log('3');
  process.nextTick(() => console.log('4'));
});
process.nextTick(() => console.log('5'));
```

**Exercise 4 — EventEmitter sync vs deferred.** Predict the output, and explain why the `ready` listener attached after construction still fires:

```js
const EventEmitter = require('node:events');
class Widget extends EventEmitter {
  constructor() {
    super();
    // Why nextTick instead of emitting directly here?
    process.nextTick(() => this.emit('ready'));
  }
}
const w = new Widget();
console.log('constructed');
w.on('ready', () => console.log('ready fired'));
console.log('after listener attached');
```

**Exercise 5 — Build a one-shot async waiter.** Without using `events.once`, write a function `waitForEvent(emitter, name)` that returns a Promise resolving with the first emitted argument when `name` fires, and rejecting if `'error'` fires first. Make sure you don't leak listeners. Then rewrite it using `events.once`.

**Exercise 6 — Drift-free poller with clean exit.** Write a `startPoller(fn, intervalMs)` that calls the async function `fn` repeatedly, waiting `intervalMs` *after each completion* (no overlap, no drift), returns a `stop()` function to cancel it, and does **not** keep the process alive on its own (the program should be able to exit between polls). Demonstrate it.

### Solutions

**Solution 1.**

```text
A
E
D
C
B
```

Synchronous first: `A`, `E`. Then microtasks: nextTick (`D`) before Promise (`C`). Then the timers phase: `B`. Order of mechanisms is always sync → nextTick → Promise → timers.

**Solution 2.**

```text
read done
nextTick
immediate
timeout
```

The `readFile` callback runs in the **poll** phase, printing `read done`. When it returns, microtasks drain → `nextTick`. Then the loop continues from poll: the **check** phase is next, so `immediate` runs *this* tick. The `setTimeout` must wait for the **next** iteration's timers phase, so `timeout` is last. It's deterministic because we're already inside the loop (in poll), where check reliably precedes the next timers phase — unlike at the top level, where the startup race makes `setImmediate` vs `setTimeout(0)` unpredictable.

**Solution 3.**

```text
1
5
3
2
4
```

Main script schedules nextTicks `1` and `5`, and Promise `3`. Drain **nextTick queue** FIFO: `1` (which schedules Promise `2`), then `5`. nextTick queue empty → drain **Promise queue** FIFO: `3` was queued before `2`, so `3` (schedules nextTick `4`), then `2`. After the Promise queue, re-check nextTick: `4` is waiting → run `4`. Final: `1, 5, 3, 2, 4`.

**Solution 4.**

```text
constructed
after listener attached
ready fired
```

The constructor runs synchronously but defers `emit('ready')` to the nextTick queue. So the constructor returns, `console.log('constructed')` runs, the `.on('ready', …)` listener is attached, `after listener attached` prints — and only *then*, once the synchronous script finishes, does the nextTick queue drain and fire `ready fired`. If the constructor had called `this.emit('ready')` *directly* (synchronously), it would fire before the caller had any chance to attach a listener, so `ready fired` would never print. Deferring the emit is the standard idiom for "emit after the caller can subscribe."

**Solution 5.**

```js
// Manual version
function waitForEvent(emitter, name) {
  return new Promise((resolve, reject) => {
    function onEvent(...args) {
      emitter.off('error', onError); // clean up the OTHER listener to avoid leaks
      resolve(args[0]);
    }
    function onError(err) {
      emitter.off(name, onEvent);    // clean up so we don't leak
      reject(err);
    }
    emitter.once(name, onEvent);     // once: auto-removes itself after firing
    emitter.once('error', onError);
  });
}

// Idiomatic version using the built-in helper:
const { once } = require('node:events');
async function waitForEventBuiltin(emitter, name) {
  const args = await once(emitter, name); // resolves with the emitted args array;
  return args[0];                          // rejects automatically if 'error' fires first
}

// Demo:
const EventEmitter = require('node:events');
const ee = new EventEmitter();
waitForEvent(ee, 'ready').then((v) => console.log('got:', v));
setTimeout(() => ee.emit('ready', 42), 10); // → got: 42
```

The key points: use `once` so the success listener self-removes, and explicitly remove the *opposite* listener in each branch so a resolved promise doesn't leave a dangling `'error'` listener (and vice versa). The built-in `events.once` handles all of this for you and is what you should reach for in real code.

**Solution 6.**

```js
function startPoller(fn, intervalMs) {
  let stopped = false;
  let timer = null;

  async function run() {
    if (stopped) return;
    try {
      await fn();                 // await so we never overlap runs
    } catch (err) {
      console.error('poll error:', err);
    }
    if (stopped) return;
    timer = setTimeout(run, intervalMs); // schedule the NEXT run only after this one finished
    timer.unref();                       // don't keep the process alive between polls
  }

  // Kick off the first run on the next tick so callers can finish setup first.
  setImmediate(run);

  return function stop() {
    stopped = true;
    if (timer) clearTimeout(timer);
  };
}

// Demo:
let n = 0;
const stop = startPoller(async () => {
  console.log('poll', ++n, new Date().toISOString());
  await new Promise((r) => setTimeout(r, 50)); // simulate async work
}, 1000);

// Stop after ~3.5s:
setTimeout(stop, 3500); // this timer is NOT unref'd, so it keeps the demo alive long enough
```

Why this satisfies the requirements: each next run is scheduled with `setTimeout` *after* `await fn()` resolves, so runs never overlap and the interval is measured from completion (no drift). `timer.unref()` means the poller's own pending timer won't hold the process open — between polls the program can exit if nothing else is keeping it alive. `stop()` flips a flag (so an in-flight run won't reschedule) and clears any pending timer. We use `setImmediate` for the first run so the caller's synchronous setup completes first.

---

## Summary / Cheat Sheet

**Event loop phases (in order):**

| Order | Phase | Runs |
|-------|-------|------|
| 1 | timers | `setTimeout` / `setInterval` (delay elapsed) |
| 2 | pending callbacks | deferred system callbacks (rare) |
| 3 | idle / prepare | internal only |
| 4 | poll | I/O callbacks; may **block** waiting for I/O |
| 5 | check | `setImmediate` |
| 6 | close callbacks | `'close'` events |

**Priority of deferral mechanisms (highest → lowest):**

1. `process.nextTick` (its own microtask queue — drained first, fully)
2. Promises / `queueMicrotask` / `await` continuations (Promise microtask queue — drained second, fully)
3. `setImmediate` (check phase)
4. `setTimeout(fn, 0)` (next timers phase; `0` → `1`ms)

**Golden rules:**

- Order to evaluate by hand: **sync code → nextTick queue → Promise queue → timers → check → close**, re-draining *both* microtask queues after *every* callback.
- Microtasks are not phases; they run *between* every loop callback.
- `setImmediate` > `setTimeout(0)` inside I/O callbacks (deterministic); non-deterministic at top level.
- Recursive `nextTick` starves the loop; recursive `setImmediate` is safe.
- `emit` is **synchronous**; listeners run in registration order.
- No `'error'` listener + emitted `'error'` = thrown/crash. Always handle it.
- `unref()` lets the process exit despite a pending timer; default timers keep it alive.
- No `clearNextTick`; `process.nextTick` can't be cancelled once scheduled.
- `>10` listeners on one event → `MaxListenersExceededWarning` (probable leak).

**Quick API reference:**

```js
const id  = setTimeout(fn, ms, ...args);   clearTimeout(id);
const iv  = setInterval(fn, ms, ...args);  clearInterval(iv);
const im  = setImmediate(fn, ...args);     clearImmediate(im);
process.nextTick(fn, ...args);             // no clear; highest priority
queueMicrotask(fn);                        // Promise-queue priority
id.unref(); id.ref(); id.refresh();        // control liveness / restart

const EventEmitter = require('node:events');
const ee = new EventEmitter();
ee.on('evt', fn); ee.once('evt', fn); ee.off('evt', fn);
ee.emit('evt', ...args);                   // synchronous, FIFO
ee.on('error', fn);                        // mandatory in practice
const [arg] = await require('node:events').once(ee, 'evt'); // await an event
```
