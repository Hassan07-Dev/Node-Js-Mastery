# Async, Event Loop & Output-Prediction Questions

This is an interview preparation question bank focused on **asynchronous JavaScript and the Node.js event loop** — the single most common area where backend interviews separate confident candidates from shaky ones. If you can reliably predict the output of an ordering puzzle and *explain why*, you signal genuine understanding of how Node achieves concurrency on one thread.

### How to use this bank

1. **Cover the answer first.** For every output-prediction question, write down your predicted output on paper *before* reading the model answer. Reading the answer is not learning — predicting and then checking is.
2. **Verify everything yourself.** Every code example here was run on **Node.js v22 LTS**. Paste any snippet into a file and run `node file.js`. Seeing the output appear in your own terminal builds the intuition that re-reading cannot.
3. **Learn the mental model, not the trivia.** Memorizing "nextTick beats Promise.then" is fragile. Understanding *why* (nextTick has its own queue drained before the microtask queue) means you can derive the answer for any variation.
4. **Practice explaining out loud.** In a real interview you must narrate your reasoning. Each model answer is written the way a strong candidate would actually talk.

> TIP: When asked to predict output, always verbalize the same checklist: (1) run all synchronous code top to bottom, (2) drain `process.nextTick` queue, (3) drain the microtask (Promise) queue, (4) move to the next event-loop phase (timers, then I/O, then `setImmediate`), and after **each** macrotask callback, drain nextTick + microtasks again.

---

## Beginner

### 1. **What does it mean that Node.js is "single-threaded but non-blocking"? How can one thread handle thousands of concurrent connections?**

Node runs your JavaScript on a **single main thread** with a single call stack — only one line of *your* code executes at a time. There is no parallel execution of JS. What makes it scale is that Node never *blocks* that thread waiting for slow operations.

When you start an I/O operation (reading a file, querying a database, awaiting an HTTP response), Node hands the work off to the operating system or to **libuv's worker thread pool**, then immediately returns control to the event loop so it can run other code. When the slow operation finishes, its callback is queued and eventually run on the main thread.

The key insight: most server work is **I/O-bound**, not CPU-bound. A web server spends almost all its time *waiting* — for the database, the disk, the network. A traditional thread-per-request model parks an entire OS thread (with its ~1MB stack and context-switch cost) on each wait. Node instead parks a tiny callback/closure and reuses the one thread for whatever is ready next. That is how a single thread juggles thousands of connections: it is never idle waiting; it is always working on whatever is ready.

```js
// This does NOT block the thread for 5 seconds.
// The timer is registered with libuv and the thread moves on.
setTimeout(() => console.log("done waiting"), 5000);
console.log("this prints first, immediately"); // runs while the timer counts down
```

> WARNING — Pitfall: "Non-blocking" only applies to async I/O APIs. A CPU-heavy synchronous loop (e.g. hashing in a `for` loop, `JSON.parse` on a huge string, `fs.readFileSync`) **does** block the single thread and freezes *every* connection. Offload CPU-bound work to `worker_threads` or a separate process.

**Follow-up:** "If JS is single-threaded, who actually does the file read?" → libuv's thread pool (default 4 threads, tunable via `UV_THREADPOOL_SIZE`) for file system and some crypto/DNS work; network I/O typically uses the OS kernel's async mechanisms (epoll/kqueue/IOCP) directly, not the pool.

---

### 2. **Explain callbacks, Promises, and async/await. Why did each one get introduced?**

They are three generations of solving the same problem: *running code after an async operation completes.*

**Callbacks** are the original mechanism — you pass a function to be called when the work finishes. Node's convention is **error-first callbacks**: `(err, result) => {}`.

```js
const fs = require("fs");

fs.readFile("data.txt", "utf8", (err, data) => {
  if (err) return console.error("failed:", err); // error is the FIRST argument
  console.log(data);
});
```

The problem: nesting dependent async steps produces **"callback hell"** (the pyramid of doom) — deeply indented, hard to read, and error handling must be repeated at every level.

**Promises** (ES2015) are objects representing a future value, in one of three states: `pending`, `fulfilled`, or `rejected`. They let you *chain* with `.then()` and centralize error handling with a single `.catch()`, flattening the pyramid.

```js
const fs = require("fs/promises");

fs.readFile("data.txt", "utf8")
  .then((data) => JSON.parse(data))   // each .then returns a new promise
  .then((obj) => console.log(obj.name))
  .catch((err) => console.error("any step failed:", err)); // one handler for all
```

**async/await** (ES2017) is **syntactic sugar over Promises**. An `async` function always returns a Promise; `await` pauses the function until the awaited Promise settles, then resumes with its value. It lets you write asynchronous code that *reads* synchronously, using normal `try/catch` for errors.

```js
async function load() {
  try {
    const data = await fs.readFile("data.txt", "utf8"); // looks synchronous, isn't
    const obj = JSON.parse(data);
    console.log(obj.name);
  } catch (err) {
    console.error("any step failed:", err);
  }
}
```

| Aspect | Callbacks | Promises | async/await |
|---|---|---|---|
| Introduced | Original | ES2015 | ES2017 |
| Composition | Nesting (pyramid) | `.then()` chaining | sequential `await` |
| Error handling | manual, per-callback | `.catch()` | `try/catch` |
| Underlying mechanism | function reference | Promise object | Promises (sugar) |
| Readability at scale | poor | good | excellent |

> BEST PRACTICE: Reach for async/await by default in new code. Drop to raw Promises when you need combinators (`Promise.all`, `Promise.race`) or to keep operations running concurrently without awaiting each in turn.

**Follow-up:** "Is async/await slower than Promises?" → No meaningful difference; it compiles down to the same Promise machinery and state-machine transitions.

---

### 3. **What are the three states of a Promise? Can a Promise change state more than once?**

A Promise is in exactly one of:

- **pending** — initial state, not yet settled.
- **fulfilled** — completed successfully, holds a *value*.
- **rejected** — failed, holds a *reason* (usually an Error).

Once a Promise transitions from `pending` to either `fulfilled` or `rejected`, it is **settled** and **immutable** — it can never change state again, and its value/reason is fixed forever. Calling `resolve()` and then `reject()` (or `resolve()` twice) is a no-op after the first call.

```js
const p = new Promise((resolve, reject) => {
  resolve("first");   // settles the promise as fulfilled with "first"
  reject("second");   // ignored — already settled
  resolve("third");   // ignored — already settled
});

p.then((v) => console.log(v)); // logs "first"
```

> TIP: "Settled" = fulfilled OR rejected (i.e. no longer pending). "Resolved" is a subtler term: a resolved promise is one that has been *locked in* to follow another promise/thenable — it may still be pending if that inner promise is pending. For interviews, "settled = fulfilled or rejected" is the answer they want.

**Follow-up:** "What happens if you `.then()` an already-settled Promise?" → The callback still runs, but asynchronously, on the next microtask tick — never synchronously.

---

### 4. **Predict the output:**

```js
console.log("1");
setTimeout(() => console.log("2"), 0);
Promise.resolve().then(() => console.log("3"));
console.log("4");
```

**Output:**

```text
1
4
3
2
```

**Why, step by step:**

1. `console.log("1")` is synchronous → prints `1`.
2. `setTimeout(..., 0)` registers a **macrotask** in the timers phase. It does *not* run now.
3. `Promise.resolve().then(...)` schedules a **microtask**. It does *not* run now.
4. `console.log("4")` is synchronous → prints `4`.
5. The synchronous script finishes and the call stack empties. Before moving to the next event-loop phase, Node **drains the entire microtask queue** → prints `3`.
6. The event loop proceeds to the timers phase and runs the expired `setTimeout` callback → prints `2`.

The headline rule: **all synchronous code first, then all microtasks, then the next macrotask.** Microtasks always beat `setTimeout`.

**Follow-up:** "What if there were two `.then()` callbacks?" → Both microtasks drain before the timer, in registration order.

---

### 5. **What is the difference between `setTimeout(fn, 0)` and `setImmediate(fn)`?**

Both schedule a callback to run "soon, asynchronously," but they live in **different phases** of the event loop:

- `setTimeout(fn, 0)` schedules in the **timers** phase (the first phase of each loop iteration). Note `0` is clamped to a minimum of `1ms` internally.
- `setImmediate(fn)` schedules in the **check** phase, which runs *after* the poll (I/O) phase.

When called from the **top level** of the main module, their relative order is **non-deterministic** — it depends on how long process startup took relative to the timer's 1ms floor.

```js
// Top-level: order is NOT guaranteed — could be either way on different runs/machines.
setTimeout(() => console.log("timeout"), 0);
setImmediate(() => console.log("immediate"));
```

But **inside an I/O callback**, the order is **deterministic**: `setImmediate` always runs first. That is because an I/O callback executes in the poll phase, and the very next phase is `check` (setImmediate), whereas timers are not revisited until the next full loop iteration.

```js
const fs = require("fs");
fs.readFile(__filename, () => {
  setTimeout(() => console.log("timeout"), 0);
  setImmediate(() => console.log("immediate"));
});
// ALWAYS prints:
// immediate
// timeout
```

> TIP: If an interviewer shows you the top-level version and asks "what's the output?", the correct senior answer is: *"It's non-deterministic at the top level. If you want a guaranteed answer, put them inside an I/O callback, where `setImmediate` always wins."*

**Follow-up:** "Why use `setImmediate` over `setTimeout(fn, 0)`?" → To run a callback right after the current I/O completes without the 1ms timer floor, and to yield to I/O between heavy chunks of work.

---

### 6. **What is `process.nextTick()` and how does it differ from `setImmediate()`? (Despite the misleading names.)**

Their names are essentially backwards from what they do:

- **`process.nextTick(fn)`** runs `fn` **before the event loop continues** — immediately after the current operation completes, before any I/O or timers. It does *not* wait for the next loop tick; it runs at the *end of the current one*.
- **`setImmediate(fn)`** runs `fn` on the **next** iteration of the event loop, in the check phase — i.e. *later* than nextTick.

So `process.nextTick` actually fires sooner. The `nextTick` queue is a **separate queue** that is drained **completely** after each operation and **before** the regular Promise microtask queue.

```js
console.log("start");
setImmediate(() => console.log("setImmediate"));
process.nextTick(() => console.log("nextTick"));
Promise.resolve().then(() => console.log("promise"));
console.log("end");

// Output:
// start
// end
// nextTick        <- nextTick queue drains first
// promise         <- then the microtask (Promise) queue
// setImmediate    <- macrotask, next loop iteration
```

> WARNING — Pitfall: Recursively calling `process.nextTick` **starves the event loop** — the loop can never advance to I/O or timers because the nextTick queue keeps refilling and must be fully drained first. The same recursion with `setImmediate` does *not* starve I/O, because setImmediate yields to the loop between iterations.

```js
// DANGER: this freezes all I/O forever — the loop never reaches the poll phase.
function loop() {
  process.nextTick(loop);
}
loop();
```

**Follow-up:** "Which queue has higher priority, nextTick or Promise microtasks?" → `process.nextTick` — its queue is drained fully before the Promise microtask queue on each pass.

---

### 7. **What is an "unhandled promise rejection" and why does it matter?**

It is a rejected Promise that has **no `.catch()` handler** (or no surrounding `try/catch` for an `await`). The rejection has nowhere to go.

```js
Promise.reject(new Error("boom")); // no .catch — unhandled rejection
```

This matters because in **modern Node (v15+)**, an unhandled rejection **crashes the process by default** (it behaves like an uncaught exception). In older Node it merely printed a warning. Either way, it signals a real bug: an error is being silently lost.

```js
// Always attach a handler, or await inside try/catch.
Promise.reject(new Error("boom")).catch((err) => {
  console.error("handled:", err.message);
});

// Last-resort safety net (log + decide whether to exit gracefully):
process.on("unhandledRejection", (reason, promise) => {
  console.error("Unhandled Rejection:", reason);
  // In production: log, alert, then do a graceful shutdown.
});
```

> BEST PRACTICE: Treat `unhandledRejection` and `uncaughtException` handlers as logging/observability and *graceful shutdown* hooks — not as a way to keep limping along. After an uncaught error the process is in an unknown state; log it and exit, letting a process manager restart you.

**Follow-up:** "What's the difference between `unhandledRejection` and `uncaughtException`?" → The former is for rejected promises with no handler; the latter for synchronous (or callback) errors thrown with no `try/catch`.

---

### 8. **Predict the output:**

```js
async function f() {
  console.log("A");
  await null; // awaiting a non-promise still yields to the microtask queue
  console.log("B");
}
console.log("C");
f();
console.log("D");
```

**Output:**

```text
C
A
D
B
```

**Why:**

1. `console.log("C")` → prints `C`.
2. `f()` is called. Code *before* the first `await` runs **synchronously** → prints `A`.
3. `await null` — even though `null` isn't a Promise, `await` wraps it (`Promise.resolve(null)`) and **suspends** `f`, scheduling the continuation (everything after the `await`) as a **microtask**. Control returns to the caller.
4. `console.log("D")` → prints `D`.
5. Synchronous script ends; the microtask queue is drained → the continuation of `f` resumes → prints `B`.

Key takeaway: **everything before the first `await` in an async function runs synchronously.** Only the part after `await` is deferred. And `await` *always* yields, even when awaiting a non-Promise value.

**Follow-up:** "Does `await` on a resolved promise still take a tick?" → Yes. `await Promise.resolve(x)` still schedules the continuation as a microtask; it never resumes synchronously.

---

### 9. **What does `Promise.all()` do, and how does it fail?**

`Promise.all(iterable)` takes an array of promises and returns a single promise that:

- **fulfills** with an array of all resolved values **once every input promise fulfills**, preserving input order (not completion order); or
- **rejects immediately** as soon as **any one** input promise rejects — this is "fail-fast." The other promises are *not* cancelled (JS promises can't be cancelled), but their results are ignored.

```js
const results = await Promise.all([
  fetch("/users"),
  fetch("/posts"),
  fetch("/comments"),
]);
// results is [usersResp, postsResp, commentsResp] in INPUT order.
// If ANY fetch rejects, the whole await throws and you lose the others' values.
```

It runs the operations **concurrently** — they all start immediately when you create the promises, and `Promise.all` just waits for them collectively. That makes it far faster than awaiting each in sequence when the operations are independent.

> WARNING — Pitfall: Because `Promise.all` rejects on the first failure, a single failed request makes you lose the values of the successful ones. When you need *all* results regardless of individual failures, use `Promise.allSettled` instead.

**Follow-up:** "How would you get results from the rest even if one fails?" → `Promise.allSettled`, covered below.

---

### 10. **Predict the output:**

```js
console.log("start");

setTimeout(() => console.log("timeout"), 0);

Promise.resolve()
  .then(() => console.log("promise 1"))
  .then(() => console.log("promise 2"));

console.log("end");
```

**Output:**

```text
start
end
promise 1
promise 2
timeout
```

**Why:**

1. `start` prints synchronously.
2. `setTimeout` registers a macrotask.
3. The first `.then` is queued as a microtask. The **second `.then` is not queued yet** — it can only be scheduled *after* the first one runs and returns.
4. `end` prints synchronously.
5. Microtask queue drains: `promise 1` runs. Returning from it resolves the chained promise, which **now** queues `promise 2` as a new microtask. The microtask queue is drained until *empty*, so `promise 2` runs in the same draining pass → prints `promise 2`.
6. Only after the microtask queue is fully empty does the loop reach the timers phase → `timeout`.

The lesson: a chained `.then` is scheduled lazily, but because the entire microtask queue (including newly-added microtasks) drains before any macrotask, the whole chain completes before `setTimeout`.

**Follow-up:** "Could a long Promise chain delay timers indefinitely?" → Yes, in principle — continuously adding microtasks starves macrotasks, the same way recursive nextTick does.

---

## Intermediate

### 11. **Walk through the phases of the Node.js event loop in order. Where do microtasks fit in?**

libuv's event loop runs in repeating iterations ("ticks"), each progressing through ordered **phases**, each with its own callback queue:

1. **timers** — executes callbacks scheduled by `setTimeout` and `setInterval` whose time has elapsed.
2. **pending callbacks** — executes certain deferred system/OS callbacks (e.g. some TCP errors).
3. **idle, prepare** — internal use only.
4. **poll** — the heart of the loop: retrieves new I/O events and executes their callbacks (file reads, incoming connections, etc.); may *block* here waiting for I/O if there's nothing else to do.
5. **check** — executes `setImmediate` callbacks.
6. **close callbacks** — executes close events, e.g. `socket.on("close", ...)`.

**Microtasks are not a phase.** The `process.nextTick` queue and the Promise microtask queue are drained **between every callback** and **between every phase transition** — essentially, whenever the call stack empties. On each opportunity, the **nextTick queue is drained first**, then the **Promise microtask queue**, and this repeats until both are empty before the loop advances.

```text
   ┌───────────────────────────┐
┌─>│           timers          │  setTimeout / setInterval
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  (internal)
│  └─────────────┬─────────────┘      ┌───────────────┐
│  ┌─────────────┴─────────────┐      │   incoming:   │
│  │           poll            │<─────┤  connections, │  I/O callbacks
│  └─────────────┬─────────────┘      │   data, etc.  │
│  ┌─────────────┴─────────────┐      └───────────────┘
│  │           check           │  setImmediate
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │  'close' events
   └───────────────────────────┘

   Between EVERY callback: drain nextTick queue, then microtask (Promise) queue.
```

**Follow-up:** "When does the poll phase block?" → When the poll queue is empty *and* there are no `setImmediate` callbacks pending and no timers about to fire, it blocks waiting for I/O (with a computed timeout so timers still fire on time).

---

### 12. **Predict the output:**

```js
console.log("script start");

setTimeout(() => console.log("setTimeout"), 0);

Promise.resolve()
  .then(() => console.log("promise1"))
  .then(() => console.log("promise2"));

process.nextTick(() => console.log("nextTick"));

console.log("script end");
```

**Output:**

```text
script start
script end
nextTick
promise1
promise2
setTimeout
```

**Why:**

1. Synchronous code runs: `script start`, then (after scheduling the async items) `script end`.
2. The call stack empties. **nextTick queue drains first** → `nextTick`.
3. **Microtask (Promise) queue drains next.** `promise1` runs; its return schedules `promise2`; the queue keeps draining → `promise2`.
4. Loop advances to the timers phase → `setTimeout`.

This is the canonical priority puzzle: **sync → nextTick → microtasks → macrotasks (timers).**

**Follow-up:** "Add a `process.nextTick` *inside* `promise1`. Where does it print?" → Right after `promise1`, before `promise2`. After `promise1` runs, the engine drains the nextTick queue before continuing the microtask queue.

---

### 13. **Predict the output (the classic async/await ordering puzzle):**

```js
async function async1() {
  console.log("async1 start");
  await async2();
  console.log("async1 end");
}
async function async2() {
  console.log("async2");
}

console.log("script start");

setTimeout(() => console.log("setTimeout"), 0);

async1();

new Promise((resolve) => {
  console.log("promise1 executor");
  resolve();
}).then(() => console.log("promise2 then"));

console.log("script end");
```

**Output:**

```text
script start
async1 start
async2
promise1 executor
script end
async1 end
promise2 then
setTimeout
```

**Why, step by step:**

1. `console.log("script start")` → `script start`.
2. `setTimeout` registers a macrotask.
3. `async1()` is called. `console.log("async1 start")` runs synchronously → `async1 start`. Then `await async2()` — `async2()` runs synchronously up to its end, printing `async2`. `async2` returns a resolved promise. The `await` **suspends `async1`**, scheduling `console.log("async1 end")` as a microtask. Control returns to the caller.
4. The `new Promise` executor runs **synchronously** → `promise1 executor`. `resolve()` settles it, so its `.then` is queued as a microtask.
5. `console.log("script end")` → `script end`.
6. Synchronous script done. Drain microtasks **in the order they were queued**: first the `async1` continuation → `async1 end`; then the `.then` → `promise2 then`.
7. Loop advances to timers → `setTimeout`.

> TIP: The trap here is the **executor function runs synchronously**. `new Promise((resolve) => {...})` invokes the executor immediately — only the `.then` callback is async. Many candidates wrongly delay `promise1 executor`.

**Follow-up:** "Why does `async1 end` print before `promise2 then`?" → Because the `await async2()` continuation was queued *before* the `.then` (async1() is called before the new Promise line), and microtasks run in FIFO order.

---

### 14. **What is the difference between sequential and parallel (concurrent) execution with async/await? Show both.**

The mistake is awaiting independent operations one after another, forcing them to run **sequentially** when they could overlap.

```js
// SEQUENTIAL — total time ≈ a + b + c (each await blocks the next from starting)
async function sequential() {
  const a = await fetchA(); // starts, waits for A to finish...
  const b = await fetchB(); // ...only then starts B
  const c = await fetchC(); // ...only then starts C
  return [a, b, c];
}

// CONCURRENT — total time ≈ max(a, b, c) (all start before any await)
async function concurrent() {
  const pa = fetchA(); // start all three immediately — DON'T await yet
  const pb = fetchB();
  const pc = fetchC();
  return [await pa, await pb, await pc]; // now await the already-running promises
}

// Cleaner concurrent form with Promise.all:
async function concurrentAll() {
  return Promise.all([fetchA(), fetchB(), fetchC()]); // start all, wait collectively
}
```

The crucial point: a Promise starts executing **the moment it is created**, not when it is awaited. To run things concurrently, *create all the promises first*, then await them (or use `Promise.all`).

> BEST PRACTICE: Use sequential `await` only when a later step genuinely depends on an earlier step's result. For independent operations, always use `Promise.all` — it can turn a 3-second endpoint into a 1-second one with a one-line change.

**Follow-up:** "What's a downside of `Promise.all` here vs the manual `await pa` version?" → `Promise.all` rejects fast on first failure; if you `await pa; await pb` and `pa` rejects, you may get an unhandled rejection on `pb`. Use `allSettled` or attach `.catch` to each when you need every result.

---

### 15. **Compare `Promise.all`, `Promise.allSettled`, `Promise.race`, and `Promise.any`.**

All four are static combinators that take an iterable of promises and return a single promise, but they differ in *when* they settle and *what* they give you:

| Combinator | Fulfills when | Rejects when | Resolves with | Use case |
|---|---|---|---|---|
| `Promise.all` | **all** fulfill | **any** rejects (fast-fail) | array of values (input order) | All must succeed |
| `Promise.allSettled` | **all** settle (never rejects) | never | array of `{status, value/reason}` | Want every outcome, failures OK |
| `Promise.race` | first to **settle** (fulfill *or* reject) | first settles as reject | value/reason of the first settler | Timeouts; first responder |
| `Promise.any` | first to **fulfill** | **all** reject → `AggregateError` | value of first fulfiller | First success; ignore failures |

```js
// allSettled — inspect each result individually, nothing is lost
const results = await Promise.allSettled([fetchA(), fetchB()]);
for (const r of results) {
  if (r.status === "fulfilled") console.log("ok:", r.value);
  else console.log("failed:", r.reason);
}

// race — implement a timeout
function withTimeout(promise, ms) {
  const timeout = new Promise((_, reject) =>
    setTimeout(() => reject(new Error("timed out")), ms)
  );
  return Promise.race([promise, timeout]); // whichever settles first wins
}

// any — return the fastest SUCCESSFUL response, ignoring individual failures
const fastest = await Promise.any([mirror1(), mirror2(), mirror3()]);
```

Memory hooks: **all** = all must win. **allSettled** = report card for everyone. **race** = first across the line, win *or* lose. **any** = first *winner* (losers ignored unless they all lose).

> WARNING — Pitfall: `Promise.race` settles on the first promise to *reject* too, not just fulfill. If you want "first success," that's `Promise.any`, not `race`.

**Follow-up:** "What does `Promise.any` reject with if all reject?" → An `AggregateError` whose `.errors` array contains every individual rejection reason.

---

### 16. **Predict the output:**

```js
const p1 = Promise.reject("err1");
Promise.any([p1, Promise.resolve("ok"), Promise.reject("err2")])
  .then((v) => console.log("any:", v))
  .catch((e) => console.log("any caught:", e));

Promise.race([Promise.reject("race-err"), Promise.resolve("race-ok")])
  .then((v) => console.log("race:", v))
  .catch((e) => console.log("race caught:", e));
```

**Output:**

```text
race caught: race-err
any: ok
```

**Why:**

- **`Promise.any`** ignores rejections and resolves with the first *fulfillment*. `Promise.resolve("ok")` is fulfilled, so `any` resolves with `"ok"`. The two rejections are ignored → `any: ok`.
- **`Promise.race`** settles with the *first to settle*, regardless of fulfill/reject. The inputs are all already-settled promises; the **first in the array** wins the race when they're queued in order, and that's the rejection `"race-err"` → `race caught: race-err`.

As for ordering between the two lines: both `race` inputs are already settled, so `race` resolves in an earlier microtask round than `any` (which must also evaluate already-settled inputs but resolves with the second array element). In practice `race caught` prints before `any:`. The reliable interview point is the *values*: `race` rejects with `"race-err"`, `any` fulfills with `"ok"`.

**Follow-up:** "What if you swap the first two race elements?" → `race` would resolve with `"race-ok"` instead, since the first-positioned settled promise wins.

---

### 17. **Predict the output and explain the bug:**

```js
const items = [1, 2, 3];

items.forEach(async (item) => {
  await delay(100);          // pretend this is an async DB write
  console.log("processed", item);
});

console.log("all done");

function delay(ms) {
  return new Promise((resolve) => setTimeout(resolve, ms));
}
```

**Output:**

```text
all done
processed 1
processed 2
processed 3
```

**Why — and why it's a bug:**

`Array.prototype.forEach` is **not promise-aware**. It calls the async callback for each element, which returns a promise — but `forEach` **ignores those returned promises** and moves on synchronously. So all three callbacks start, each scheduling its `await delay` continuation, and `forEach` returns immediately. `console.log("all done")` runs *before any item is processed.*

The bug: **`forEach` does not wait.** If your intent was "log 'all done' *after* processing everything," this is broken. The processing also runs concurrently (all three `delay`s start together), which may or may not be what you want.

**Fixes:**

```js
// Fix 1 — sequential, awaited: use a for...of loop (await works as expected)
for (const item of items) {
  await delay(100);
  console.log("processed", item);
}
console.log("all done"); // truly after all items

// Fix 2 — concurrent, awaited: map to promises, then Promise.all
await Promise.all(
  items.map(async (item) => {
    await delay(100);
    console.log("processed", item);
  })
);
console.log("all done"); // after all items, but they ran concurrently
```

> WARNING — Pitfall: `await` inside `forEach`/`map`/`filter` callbacks is one of the most common async bugs. `forEach` silently drops the promises (errors become unhandled rejections!). Use `for...of` for sequential awaited iteration, or `Promise.all(arr.map(...))` for concurrent.

**Follow-up:** "What happens to errors thrown inside the `forEach` async callback?" → They reject the ignored promise → **unhandled rejection** → can crash the process. Another reason to avoid this pattern.

---

### 18. **Predict the output:**

```js
async function main() {
  console.log("1");
  setTimeout(() => console.log("2"), 0);
  await Promise.resolve();
  console.log("3");
  await Promise.resolve();
  console.log("4");
}
main();
console.log("5");
```

**Output:**

```text
1
5
3
4
2
```

**Why:**

1. `main()` runs synchronously until the first `await`: prints `1`, registers the `setTimeout` macrotask, then `await Promise.resolve()` suspends `main` and queues its continuation as a microtask.
2. Control returns to top level → prints `5`.
3. Sync code done; drain microtasks. The first continuation resumes → prints `3`, hits the second `await`, suspends again, queuing another microtask.
4. Microtask queue still has work: the second continuation resumes → prints `4`. `main`'s promise resolves; queue is now empty.
5. Loop reaches timers → prints `2`.

Each `await` costs one microtask "tick," but **all** of them resolve before the `setTimeout` macrotask, because the microtask queue is fully drained between phases.

**Follow-up:** "How many microtask ticks does this take before `setTimeout` runs?" → Two (one per `await`), all completing before the timer.

---

### 19. **How do you properly handle errors in async/await vs Promise chains?**

**Promise chains** — use `.catch()`. A single `.catch` at the end handles a rejection from *any* preceding step, because a rejection propagates down the chain until it hits a handler.

```js
fetchUser(id)
  .then((user) => fetchOrders(user.id))
  .then((orders) => render(orders))
  .catch((err) => {
    // catches rejection from fetchUser, fetchOrders, OR a throw in render
    console.error("pipeline failed:", err);
  });
```

**async/await** — use `try/catch`. `await` converts a rejection into a *thrown* error, so normal synchronous-looking error handling works.

```js
async function load(id) {
  try {
    const user = await fetchUser(id);
    const orders = await fetchOrders(user.id);
    return render(orders);
  } catch (err) {
    console.error("pipeline failed:", err); // catches any awaited rejection or throw
  }
}
```

Important subtleties:

- A `try/catch` **only catches awaited rejections**. If you call an async function *without* `await` inside the `try`, its rejection escapes the `try/catch`:

```js
async function bug() {
  try {
    doAsyncThing(); // NOT awaited → rejection is NOT caught here → unhandled
  } catch (e) {
    console.log("never reached for the missing await");
  }
}
```

- Use `finally` for cleanup that must run on both success and failure (closing connections, clearing timers).

> BEST PRACTICE: In Express route handlers, wrap async logic so rejections reach the error-handling middleware. In Express 5 this is automatic (a rejected async handler is forwarded to `next(err)`); in Express 4 you must wrap handlers yourself (e.g. an `asyncHandler` higher-order function or `express-async-errors`).

```js
// Express 4 — manual wrapper so async rejections reach the error middleware
const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);

app.get("/users/:id", asyncHandler(async (req, res) => {
  const user = await User.findById(req.params.id); // a rejection now goes to next()
  res.json(user);
}));
```

**Follow-up:** "Express 4 vs 5 difference for async errors?" → Express 5 automatically forwards rejected promises from async route handlers to `next(err)`; Express 4 does not, so you must wrap handlers or the rejection becomes unhandled.

---

### 20. **Predict the output:**

```js
console.log("start");

setTimeout(() => {
  console.log("timeout1");
  Promise.resolve().then(() => console.log("promise in timeout1"));
}, 0);

setTimeout(() => {
  console.log("timeout2");
}, 0);

Promise.resolve().then(() => console.log("promise1"));

console.log("end");
```

**Output:**

```text
start
end
promise1
timeout1
promise in timeout1
timeout2
```

**Why:**

1. Synchronous: `start`, `end`.
2. Drain microtasks before any macrotask → `promise1`.
3. Timers phase, first callback: `timeout1` prints; it schedules a microtask (`promise in timeout1`) but does *not* run it yet.
4. **Critical rule:** after each macrotask callback completes, Node drains the **entire microtask queue before running the next macrotask.** So `promise in timeout1` runs now → prints.
5. Then the second timer callback → `timeout2`.

The headline: **microtasks are drained between every individual macrotask**, not just once per loop iteration. This is the difference that trips people up most.

**Follow-up:** "Is browser behavior the same?" → Yes for this rule — browsers also drain the microtask queue after each macrotask. (Node's *phase* distinctions like nextTick and setImmediate are Node-specific.)

---

### 21. **What is "promisifying" a callback-based API? Show how.**

It means wrapping an error-first callback function so it returns a Promise, letting you use it with `await` and combinators.

```js
const fs = require("fs");

// Manual promisify
function readFilePromise(path) {
  return new Promise((resolve, reject) => {
    fs.readFile(path, "utf8", (err, data) => {
      if (err) reject(err);   // error-first: reject on err
      else resolve(data);     // otherwise resolve with the value
    });
  });
}

// Built-in helper — does exactly this for any error-first callback API:
const { promisify } = require("util");
const readFile = promisify(fs.readFile);

async function main() {
  const data = await readFile("data.txt", "utf8");
  console.log(data);
}
```

Many core modules already ship promise versions, so you often don't need to promisify at all:

```js
const fs = require("fs/promises"); // promise-based fs, no wrapping needed
const data = await fs.readFile("data.txt", "utf8");
```

> TIP: `util.promisify` assumes the **error-first callback convention** (callback is the *last* argument, `(err, result)`). For APIs that don't follow it (e.g. multiple result args, or callback-first), you must wrap manually or set the `util.promisify.custom` symbol.

**Follow-up:** "Can you promisify something with multiple success values, like `dns.lookup` returning `(err, address, family)`?" → `util.promisify` resolves with only the first value by default; use the `[util.promisify.custom]` hook on the function, or wrap manually to resolve with `{address, family}`.

---

## Advanced

### 22. **Predict the output of this multi-queue puzzle:**

```js
console.log("1");

setTimeout(() => console.log("2"), 0);

setImmediate(() => console.log("3"));

Promise.resolve().then(() => {
  console.log("4");
  process.nextTick(() => console.log("5"));
});

process.nextTick(() => {
  console.log("6");
  Promise.resolve().then(() => console.log("7"));
});

console.log("8");
```

**Output:**

```text
1
8
6
4
7
5
3   (or 2 — top-level setImmediate vs setTimeout order is non-deterministic)
2
```

**Why:**

1. Synchronous: `1`, `8`.
2. Call stack empty → **drain nextTick queue first.** The nextTick callback runs: prints `6`, and schedules a *microtask* (`7`).
3. nextTick queue is empty → **drain microtask queue.** The first `.then` runs: prints `4`, and schedules a *new nextTick* (`5`).
4. The microtask queue still has `7` (queued in step 2) → prints `7`.
5. Microtask queue now empty. But a nextTick (`5`) was added in step 3. The engine re-checks: drain nextTick again → prints `5`. Then re-check microtasks (empty).
6. All micro-queues drained → advance the loop. `setTimeout` (timers) and `setImmediate` (check) are both pending macrotasks; at the **top level** their relative order is **not guaranteed**, but commonly `3` then `2` on many runs.

The reliable, defensible answer for the micro-portion is: `1, 8, 6, 4, 7, 5`, followed by the two macrotasks (`2` and `3`) in non-deterministic order. The interplay to articulate: **after every microtask, the engine re-checks the higher-priority nextTick queue.**

**Follow-up:** "Why does `5` print last among the micro items even though it was scheduled in `4`?" → Because `5` is a nextTick scheduled *during* microtask draining; the engine finishes the current microtask drain, then re-checks nextTick, finding `5` after `7` has already run.

---

### 23. **Predict the output (timers vs immediate, with I/O):**

```js
const fs = require("fs");

setTimeout(() => console.log("timeout"), 0);
setImmediate(() => console.log("immediate"));

fs.readFile(__filename, () => {
  console.log("read file done");
  setTimeout(() => console.log("timeout inside IO"), 0);
  setImmediate(() => console.log("immediate inside IO"));
  process.nextTick(() => console.log("nextTick inside IO"));
});
```

**Output:**

```text
timeout            (or immediate first — top-level order non-deterministic)
immediate
read file done
nextTick inside IO
immediate inside IO
timeout inside IO
```

**Why:**

- The first two (`timeout`, `immediate`) are scheduled at the top level; their order is **non-deterministic** relative to each other. They both run before the file read completes, since reading a file is slower than the first loop iteration.
- When `fs.readFile` completes, its callback runs in the **poll** phase → prints `read file done`.
- Inside that callback we schedule three things. When the callback returns, the **nextTick queue drains first** → `nextTick inside IO`.
- The loop is now in the poll phase; the **very next phase is `check`** (setImmediate) → `immediate inside IO`.
- The timers phase isn't revisited until the next full loop iteration → `timeout inside IO` runs last.

This is the deterministic case for setImmediate vs setTimeout: **inside an I/O callback, `setImmediate` always beats `setTimeout`.**

**Follow-up:** "Why is the *inside-I/O* order deterministic but the top-level order isn't?" → Inside the poll phase, `check` is the immediate next phase, so setImmediate is guaranteed next. At the top level, whether the first loop iteration reaches the timers phase before the 1ms floor elapses is a race dependent on startup time.

---

### 24. **Predict the output — async function return value vs await:**

```js
async function getValue() {
  return 42; // returning a plain value from async fn
}

async function getPromise() {
  return Promise.resolve(99); // returning a promise from async fn
}

console.log("start");
getValue().then((v) => console.log("getValue:", v));
getPromise().then((v) => console.log("getPromise:", v));
Promise.resolve().then(() => console.log("plain microtask"));
console.log("end");
```

**Output:**

```text
start
end
getValue: 42
plain microtask
getPromise: 99
```

**Why this exact order:**

- `getValue()` returns a promise that fulfills with `42`. Its `.then` is queued as a microtask after one tick.
- `getPromise()` returns `Promise.resolve(99)`. Here's the subtlety: when an async function returns a *thenable*, the runtime must **adopt** it, which costs **extra microtask ticks** (the returned promise has to "unwrap" the inner promise). So `getPromise`'s `.then` resolves **later** than `getValue`'s.
- The `plain microtask` was queued after both async calls but resolves in a single tick, so it lands between `getValue` (1 tick) and `getPromise` (multiple ticks).

The precise tick counts vary by engine version, but the observable ordering on V8/Node is `getValue` → `plain microtask` → `getPromise`. The key teaching point: **returning a promise from an async function adds extra unwrapping ticks** compared to returning a plain value.

> TIP: This is why `return await somePromise` and `return somePromise` differ subtly in tick timing (and in stack traces inside try/catch). Inside a `try`, prefer `return await` so a rejection is caught locally.

**Follow-up:** "Does `return await x` vs `return x` matter for correctness?" → For the resolved value, no. But inside a `try/catch`, `return await x` lets the local `catch` handle a rejection; `return x` would forward the rejection to the caller instead.

---

### 25. **Predict the output — error propagation through a chain:**

```js
Promise.resolve()
  .then(() => {
    console.log("A");
    throw new Error("boom");
  })
  .then(() => {
    console.log("B"); // skipped — previous step rejected
  })
  .catch((err) => {
    console.log("caught:", err.message);
    return "recovered";
  })
  .then((val) => {
    console.log("C:", val);
  });
```

**Output:**

```text
A
caught: boom
C: recovered
```

**Why:**

1. First `.then` runs → prints `A`, then throws. A throw inside a `.then` callback **rejects** the promise that `.then` returns.
2. The next `.then` (logging `B`) has only an `onFulfilled` handler, so the rejection **skips it entirely** and propagates down.
3. `.catch` catches the rejection → prints `caught: boom`. It returns `"recovered"`, which means the promise `.catch` returns is **fulfilled** with `"recovered"` — the chain *recovers*.
4. The final `.then` runs with the recovered value → `C: recovered`.

Two key lessons: (a) a rejection skips downstream `onFulfilled` handlers until it finds a `.catch`/rejection handler; (b) a `.catch` that returns a value (and doesn't re-throw) **recovers** the chain, so subsequent `.then`s run normally.

**Follow-up:** "What if `.catch` itself threw, or re-threw?" → The chain would reject again, and the final `.then` would be skipped (its returned promise rejected) — you'd need another `.catch`.

---

### 26. **Predict the output — `Promise.all` rejection timing:**

```js
function delay(ms, value, shouldReject = false) {
  return new Promise((resolve, reject) =>
    setTimeout(() => (shouldReject ? reject(value) : resolve(value)), ms)
  );
}

console.log("start");

Promise.all([
  delay(100, "A"),
  delay(50, "B-fail", true),  // rejects at 50ms
  delay(10, "C"),
])
  .then((vals) => console.log("all:", vals))
  .catch((err) => console.log("all rejected:", err));

console.log("end");
```

**Output:**

```text
start
end
all rejected: B-fail
```

(Printed at roughly the 50ms mark.)

**Why:**

- `Promise.all` is **fail-fast**: it rejects the instant *any* input rejects. Here `B-fail` rejects at 50ms, so `Promise.all` rejects at 50ms with `"B-fail"` — even though `A` (100ms) hasn't settled and `C` (10ms) already fulfilled.
- The fulfilled value of `C` and the eventual value of `A` are **discarded** — `Promise.all` gives you nothing once it has rejected.
- Note `A` still *runs to completion* in the background (promises can't be cancelled); its resolution is simply ignored.

> WARNING — Pitfall: If those background promises could *reject* after `Promise.all` already rejected, and they have no other handler, you can get **unhandled rejection** warnings. When using `Promise.all` with operations that may fail independently, consider `allSettled` or attach a no-op `.catch` to each.

**Follow-up:** "How would you collect all successes and report failures separately?" → `Promise.allSettled`, then partition results by `status`.

---

### 27. **Predict the output — `for...of` with `await` vs `Promise.all` with `map`:**

```js
function task(n) {
  return new Promise((resolve) =>
    setTimeout(() => {
      console.log("done", n);
      resolve(n);
    }, n * 100)
  );
}

async function run() {
  console.log("--- for...of ---");
  for (const n of [3, 1, 2]) {
    await task(n); // sequential: waits for each
  }

  console.log("--- Promise.all(map) ---");
  await Promise.all([3, 1, 2].map((n) => task(n))); // concurrent
}

run();
```

**Output:**

```text
--- for...of ---
done 3
done 1
done 2
--- Promise.all(map) ---
done 1
done 2
done 3
```

**Why:**

- In the **`for...of` loop**, each `await task(n)` fully completes before the next iteration starts. So they run **strictly sequentially in array order** regardless of duration: `3` (300ms), then `1` (100ms), then `2` (200ms). Total ≈ 600ms.
- In **`Promise.all([...].map(...))`**, all three promises are created *at once*, so all three timers start together. They complete in **duration order**: `1` (100ms), `2` (200ms), `3` (300ms). Total ≈ 300ms.

This crystallizes sequential vs concurrent: same tasks, different total time and different completion order.

> BEST PRACTICE: Use `for...of` + `await` when order matters or steps depend on each other (or you must rate-limit). Use `Promise.all(map)` for independent work you want to overlap. For *bounded* concurrency (e.g. "max 5 at a time"), use a pool/limiter like `p-limit` rather than firing thousands of promises at once.

**Follow-up:** "How do you run with a concurrency limit of 5?" → Chunk the array, or use a semaphore/`p-limit`-style limiter that only starts a new task when a slot frees up.

---

### 28. **Predict the output — the closure-in-a-loop classic, async edition:**

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log("var:", i), 0);
}

for (let j = 0; j < 3; j++) {
  setTimeout(() => console.log("let:", j), 0);
}
```

**Output:**

```text
var: 3
var: 3
var: 3
let: 0
let: 1
let: 2
```

**Why:**

- **`var`** is **function-scoped**: there is one single `i` shared by all three closures. The loop completes synchronously (incrementing `i` to `3`) *before* any `setTimeout` callback runs. When the callbacks finally fire, they all read the same `i`, now `3` → `3, 3, 3`.
- **`let`** is **block-scoped**: each loop iteration gets a **fresh binding** of `j`. Each closure captures its own `j` with the value at that iteration → `0, 1, 2`.

All six timers are macrotasks; the three `var` ones were registered first, so they fire first, then the `let` ones. Within each group the order matches registration.

> TIP: This is the textbook demonstration of why `let`/`const` replaced `var` for loops. With `var`, the pre-ES6 fix was an IIFE per iteration to capture the value: `(function(captured) { setTimeout(() => console.log(captured), 0); })(i);`

**Follow-up:** "Fix the `var` version to print 0,1,2 without changing `var` to `let`." → Wrap the body in an IIFE that takes `i` as a parameter, capturing its current value per iteration.

---

### 29. **Predict the output — interleaving two async functions:**

```js
async function foo() {
  console.log("foo 1");
  await Promise.resolve();
  console.log("foo 2");
  await Promise.resolve();
  console.log("foo 3");
}

async function bar() {
  console.log("bar 1");
  await Promise.resolve();
  console.log("bar 2");
  await Promise.resolve();
  console.log("bar 3");
}

foo();
bar();
console.log("sync");
```

**Output:**

```text
foo 1
bar 1
sync
foo 2
bar 2
foo 3
bar 3
```

**Why:**

1. `foo()` runs synchronously until its first `await` → `foo 1`, then suspends, queuing continuation #1.
2. `bar()` runs synchronously until its first `await` → `bar 1`, then suspends, queuing continuation #2.
3. Top-level sync code → `sync`.
4. Drain microtasks in FIFO order: continuation #1 (`foo 2`), then continuation #2 (`bar 2`). Each hits another `await` and re-queues.
5. The newly queued continuations run in order: `foo 3`, then `bar 3`.

The two async functions **interleave** because each `await` yields control, and their continuations are processed round-robin from the single microtask queue. This is cooperative concurrency on one thread — no parallelism, but overlapping progress.

**Follow-up:** "Is this parallel execution?" → No. It's concurrency via interleaving on one thread; only one statement runs at any instant. True parallelism requires `worker_threads` or multiple processes.

---

### 30. **Predict the output — `await` in a `try/catch` with a non-awaited rejection:**

```js
async function risky() {
  throw new Error("inner failure");
}

async function caller() {
  try {
    risky(); // BUG: not awaited
    console.log("after risky call");
  } catch (e) {
    console.log("caught:", e.message);
  }
}

caller();
console.log("end");
```

**Output:**

```text
after risky call
end
<UnhandledPromiseRejection> Error: inner failure   (and in modern Node, the process exits non-zero)
```

**Why:**

- `risky()` is called **without `await`**. It returns a rejected promise, but because nothing awaits it, the `try/catch` cannot intercept it — `try/catch` only catches **synchronous throws and awaited rejections**.
- Execution continues normally: `after risky call`, then `end`.
- The rejected promise from `risky()` has no handler → **unhandled rejection**. In Node 15+ this terminates the process by default after the current synchronous work and microtasks flush.

**Fix:** add the missing `await`:

```js
async function caller() {
  try {
    await risky(); // now the rejection becomes a throw caught here
    console.log("after risky call"); // skipped on rejection
  } catch (e) {
    console.log("caught:", e.message); // prints "caught: inner failure"
  }
}
```

> WARNING — Pitfall: "Forgetting `await`" is a top production-bug source. The code *looks* correct, runs without immediate error, and silently drops failures — or crashes the process later via unhandled rejection. Linters (`require-await`, `no-floating-promises` via typescript-eslint, or `eslint-plugin-promise`) catch many of these.

**Follow-up:** "When is it legitimate to *not* await a promise?" → Fire-and-forget where you genuinely don't need the result and you've attached a `.catch` (e.g. best-effort logging/metrics). Even then, attach a `.catch` so a failure can't become an unhandled rejection.

---

### 31. **Predict the output — mixed nextTick, microtask, and async chain:**

```js
async function async1() {
  console.log("async1 start");
  await async2();
  console.log("async1 end");
  process.nextTick(() => console.log("nextTick in async1"));
}
async function async2() {
  console.log("async2");
}

console.log("script start");
async1();
process.nextTick(() => console.log("nextTick top"));
Promise.resolve().then(() => console.log("promise top"));
console.log("script end");
```

**Output:**

```text
script start
async1 start
async2
script end
nextTick top
async1 end
promise top
nextTick in async1
```

**Why:**

1. Sync: `script start`. `async1()` runs sync to its first `await` → `async1 start`, `async2`. The continuation (`async1 end` onward) is queued as a microtask. Then `process.nextTick(...)` queues `nextTick top`. `Promise.resolve().then` queues `promise top`. Sync: `script end`.
2. Call stack empty → **nextTick queue drains first** → `nextTick top`.
3. **Microtask queue drains**, FIFO. First is the `async1` continuation → `async1 end`, which schedules a *new nextTick* (`nextTick in async1`). Then the next microtask → `promise top`.
4. Microtask queue empty, but a nextTick was added in step 3 → re-check nextTick queue → `nextTick in async1`.

Why does `promise top` beat `nextTick in async1`? Because `nextTick in async1` is only *scheduled* while the `async1` continuation runs; the engine finishes draining the *current* microtask batch (which already contained `promise top`) before re-checking the nextTick queue.

**Follow-up:** "State the priority rule precisely." → After the stack empties: drain all nextTicks → drain microtasks one at a time; if a microtask schedules a nextTick, it's processed only after the current microtask drain completes (then nextTick is rechecked, then microtasks again), repeating until both queues are empty.

---

### 32. **What is the difference between concurrency and parallelism in Node, and how does each map to Node's tooling?**

- **Concurrency** is *dealing with* many things at once by **interleaving** them on a single thread — making progress on multiple tasks by switching between them whenever one is waiting. This is what the event loop gives you for free: thousands of in-flight I/O operations, one thread.
- **Parallelism** is *doing* many things at the literal same instant on **multiple cores/threads**. JavaScript execution in Node is *not* parallel by default.

Mapping to Node's tools:

| Need | Tool | Mechanism |
|---|---|---|
| Many concurrent I/O operations | Event loop (built-in) | Single thread, non-blocking I/O, callbacks/promises |
| CPU-bound work (parsing, hashing, image processing) | `worker_threads` | Real OS threads, parallel JS execution, message passing |
| Scale across CPU cores for a server | `cluster` / multiple processes | Fork N processes, share a listening socket |
| Internal async file/crypto/DNS | libuv thread pool | 4 threads by default (`UV_THREADPOOL_SIZE`) |

```js
// CPU-bound work blocks the event loop — move it to a worker thread for parallelism.
const { Worker } = require("worker_threads");

function runHeavy(data) {
  return new Promise((resolve, reject) => {
    const worker = new Worker("./heavy-task.js", { workerData: data });
    worker.on("message", resolve); // result computed on another core, in parallel
    worker.on("error", reject);
  });
}
```

> TIP: Pithy interview line: *"Node gives you concurrency for I/O for free via the event loop; for parallelism (CPU work across cores) you opt in with worker_threads or cluster."*

**Follow-up:** "How do you know if your bottleneck is CPU-bound or I/O-bound?" → If the single thread is pegged (event loop lag rises, requests queue even though external services are fast), it's CPU-bound → workers. If the thread is mostly idle waiting on DB/network, it's I/O-bound → concurrency already helps; scale processes.

---

### 33. **Predict the output — `setImmediate` vs `Promise` inside an I/O callback:**

```js
const fs = require("fs");

fs.readFile(__filename, () => {
  setImmediate(() => console.log("immediate"));
  Promise.resolve().then(() => console.log("promise"));
  process.nextTick(() => console.log("nextTick"));
  console.log("io callback sync");
});
```

**Output:**

```text
io callback sync
nextTick
promise
immediate
```

**Why:**

1. The I/O callback body runs synchronously → `io callback sync`. It schedules a `setImmediate` (check phase macrotask), a Promise microtask, and a nextTick.
2. When the callback returns, the stack empties. **Micro-queues drain before the loop advances to the next phase**: nextTick first → `nextTick`; then microtasks → `promise`.
3. Only after the micro-queues are empty does the loop move from poll to the **check** phase → `immediate`.

Even though we're inside the loop's I/O phase, the micro-queues (nextTick, then Promise) still take priority over advancing to the next macrotask phase.

**Follow-up:** "Would `setTimeout(fn, 0)` here run before or after `immediate`?" → After. From the poll phase, `check` (setImmediate) is next; timers aren't revisited until the next loop iteration.

---

### 34. **What problems does the event loop create for CPU-bound work, and how do you diagnose and fix a "blocked" event loop?**

**The problem:** because all your JS runs on one thread, any long *synchronous* computation blocks the event loop entirely — no other callbacks run, no requests are served, no timers fire, until it finishes. A single 2-second `JSON.parse` or password hashing loop freezes the whole server for every connected client.

```js
// BAD — synchronous hash blocks every other request for the duration.
const crypto = require("crypto");
app.get("/hash", (req, res) => {
  const hash = crypto.pbkdf2Sync(req.query.pw, "salt", 1_000_000, 64, "sha512");
  res.send(hash.toString("hex"));
});
```

**Diagnosis:**

- Rising **event loop lag** (measure with `perf_hooks.monitorEventLoopDelay()` or libraries like `@nestjs/terminus`/`event-loop-lag`). Lag climbing under load points at CPU blocking.
- Requests timing out even though downstream services (DB, APIs) are healthy and fast.
- CPU pinned near 100% on one core while others idle.

**Fixes:**

1. **Use the async variant** if one exists — `crypto.pbkdf2` (callback) offloads to the libuv pool instead of blocking.
2. **Offload to `worker_threads`** for pure CPU work (parsing, compression, ML inference).
3. **Chunk and yield** long loops with `setImmediate` so the loop can service I/O between chunks.
4. **Move work out of process** — a job queue (BullMQ/Redis) + dedicated workers.

```js
// GOOD — async pbkdf2 runs on the libuv thread pool; the event loop stays free.
app.get("/hash", (req, res, next) => {
  crypto.pbkdf2(req.query.pw, "salt", 1_000_000, 64, "sha512", (err, derived) => {
    if (err) return next(err);
    res.send(derived.toString("hex"));
  });
});
```

> BEST PRACTICE: Keep per-tick synchronous work small (rule of thumb: well under ~10ms). Treat the event loop as a shared resource — any time you hog it, *every* client waits.

**Follow-up:** "Does `worker_threads` share memory with the main thread?" → They have separate V8 isolates/heaps, but can share raw binary memory via `SharedArrayBuffer`; everything else is passed by structured-clone message passing.

---

### 35. **Predict the output — a hard combined puzzle:**

```js
console.log("1");

setTimeout(() => {
  console.log("2");
  Promise.resolve().then(() => console.log("3"));
  process.nextTick(() => console.log("4"));
}, 0);

Promise.resolve().then(() => {
  console.log("5");
  setTimeout(() => console.log("6"), 0);
});

process.nextTick(() => console.log("7"));

console.log("8");
```

**Output:**

```text
1
8
7
5
2
4
3
6
```

**Why, carefully:**

1. **Sync pass:** `console.log("1")` → `1`. Register `setTimeout` (call it T1). Queue `.then` microtask (M1). Queue nextTick (N1). `console.log("8")` → `8`.
2. **Stack empty → drain nextTick:** N1 → `7`.
3. **Drain microtasks:** M1 runs → `5`, and registers `setTimeout` T2 (a macrotask). Microtask queue now empty.
4. **Timers phase:** T1 runs → `2`. Inside it: queue a microtask (M2 → `3`) and a nextTick (N2 → `4`).
5. **After T1 (a macrotask) completes, drain micro-queues before the next macrotask:** nextTick first → N2 → `4`; then microtasks → M2 → `3`.
6. **Next timer:** T2 (registered in step 3) runs → `6`.

So the order is `1, 8, 7, 5, 2, 4, 3, 6`. The two trickiest points: (a) after `setTimeout` T1's callback, the micro-queues drain (nextTick `4` before Promise `3`) *before* T2 runs; (b) T2 was only scheduled during microtask draining in step 3, so it runs in a later timers phase, last.

**Follow-up:** "Why does `4` come before `3` but `7` came before `5` for the same reason?" → Identical rule: the `process.nextTick` queue is always drained before the Promise microtask queue at each draining opportunity.

---

### 36. **Implement a `promiseAllSettled` polyfill and explain why it never rejects.**

```js
function promiseAllSettled(promises) {
  return Promise.all(
    promises.map((p) =>
      // Normalize each input to a promise, then convert BOTH outcomes
      // (fulfill and reject) into a fulfilled result object.
      Promise.resolve(p).then(
        (value) => ({ status: "fulfilled", value }),
        (reason) => ({ status: "rejected", reason })
      )
    )
  );
}

// Usage
promiseAllSettled([
  Promise.resolve(1),
  Promise.reject(new Error("nope")),
  Promise.resolve(3),
]).then((results) => console.log(results));
// [
//   { status: 'fulfilled', value: 1 },
//   { status: 'rejected', reason: Error: nope },
//   { status: 'fulfilled', value: 3 },
// ]
```

**Why it never rejects:** the trick is the **two-argument `.then(onFulfilled, onRejected)`**. Each input promise's rejection is *handled* by the `onRejected` callback, which **returns a value** (a result object). Returning a value from a rejection handler produces a **fulfilled** promise. So every mapped promise fulfills — never rejects — which means the wrapping `Promise.all` always fulfills with the full array of result descriptors.

> TIP: `.then(onFulfilled, onRejected)` differs from `.then(onFulfilled).catch(onRejected)`: the two-arg form does **not** catch errors thrown *inside* `onFulfilled`, whereas the chained `.catch` does. Here the two-arg form is exactly right because we want to convert the *input's* outcome, not catch our own mapper's errors.

**Follow-up:** "Implement `Promise.any` similarly." → Invert it: resolve on first fulfillment; collect rejections and, if *all* reject, reject with an `AggregateError` containing them.

---

## Rapid-Fire Quick Answers

| Question | One-line answer |
|---|---|
| Is Node single-threaded? | Your JS runs on one thread; I/O is offloaded to libuv/OS, so it's non-blocking. |
| nextTick vs setImmediate — which runs first? | `process.nextTick` — it drains before the loop continues; `setImmediate` waits for the check phase. |
| Microtasks vs macrotasks priority? | All microtasks (nextTick first, then Promises) drain before the next macrotask. |
| setTimeout(fn,0) vs setImmediate at top level? | Non-deterministic; inside an I/O callback `setImmediate` always wins. |
| Does code before the first `await` run sync? | Yes — only the part after `await` is deferred to a microtask. |
| Does `await Promise.resolve(x)` cost a tick? | Yes — the continuation always resumes on the next microtask, never synchronously. |
| Promise.all failure behavior? | Fail-fast: rejects on the first rejection, discarding other results. |
| Promise.allSettled — can it reject? | No — it always fulfills with a `{status, value/reason}` array. |
| Promise.race vs Promise.any? | `race` = first to *settle* (win or lose); `any` = first to *fulfill* (ignores rejections). |
| What does Promise.any reject with? | An `AggregateError` (only if *all* inputs reject). |
| Why is `await` in `forEach` a bug? | `forEach` ignores returned promises — it doesn't wait and can drop errors. Use `for...of` or `Promise.all(map)`. |
| Sequential vs concurrent awaits? | Sequential = await each in turn; concurrent = create all promises first, then `Promise.all`. |
| What crashes on an unhandled rejection? | In Node 15+, the process exits by default. Always `.catch` or `try/catch`. |
| Can a Promise change state twice? | No — once settled (fulfilled/rejected) it's immutable. |
| How to run CPU-bound work without blocking? | Offload to `worker_threads` (or chunk with `setImmediate`); the event loop only helps with I/O. |
| Order of event-loop phases? | timers → pending → idle/prepare → poll → check → close; micro-queues drain between each. |
| Express 4 vs 5 async route errors? | v5 auto-forwards rejected async handlers to `next(err)`; v4 needs a manual wrapper. |
| Why doesn't recursive `process.nextTick` ever yield? | Its queue must fully drain each pass, so refilling it starves I/O/timers (setImmediate doesn't). |
