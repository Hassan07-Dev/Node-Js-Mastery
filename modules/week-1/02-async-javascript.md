# Asynchronous JavaScript Deep Dive

Backend development is, at its heart, the art of waiting well. A web server spends most of its life waiting — for a database query to come back, for a file to be read off disk, for another HTTP API to respond, for a network socket to deliver bytes. The single most important skill that separates a competent Node.js developer from a struggling one is a precise mental model of **how JavaScript handles waiting without freezing**.

You already know JavaScript the language. This module is not about syntax you've seen a thousand times — it's about the *execution model* underneath it. By the end you will understand, from first principles, why Node can serve thousands of concurrent connections on a single thread, what a Promise actually *is* (not just how to type `.then`), and how to write async code that is correct, fast, and free of the subtle bugs that plague even experienced developers.

## What You'll Learn

- The difference between **synchronous** and **asynchronous** execution, and how the **call stack** drives both.
- **Callbacks**, the **error-first callback** convention that defines Node's standard library, and why deeply nested callbacks create "callback hell."
- **Promises** in depth: their three states, the `then` / `catch` / `finally` methods, chaining, and how errors propagate down a chain.
- How to **create your own promises** with `new Promise(resolve, reject)` and when you actually need to.
- The four **Promise combinators** — `Promise.all`, `Promise.allSettled`, `Promise.race`, `Promise.any` — with concrete use cases and the differences between them.
- **async/await** syntax: how it relates to Promises, error handling with `try/catch`, and returning values.
- **Sequential vs. parallel** async execution — the single most common performance mistake in Node code.
- Converting old **callback-based APIs to Promises** with `util.promisify`.
- The **common async mistakes** that bite everyone: forgetting `await`, unhandled rejections, and `await` inside `forEach`.
- A preview of **microtasks vs. macrotasks** that sets up the deeper Event Loop module.

---

## 1. Synchronous vs. Asynchronous Execution and the Call Stack

### The single thread and the call stack

JavaScript executes your code on **one thread**. There is exactly one place where your code runs at any instant. That place is governed by a data structure called the **call stack**: a stack (last-in, first-out) of *execution contexts*, one per function call.

When you call a function, a frame is **pushed** onto the stack. When the function returns, its frame is **popped**. The function at the top of the stack is the one currently running. Because there is only one stack and one thread, **only one thing runs at a time**.

```js
// call-stack.js
function multiply(a, b) {
  return a * b; // 3) runs, returns, frame popped
}

function square(n) {
  return multiply(n, n); // 2) pushes multiply onto the stack
}

function printSquare(n) {
  const result = square(n); // 1) pushes square onto the stack
  console.log(result);      // 4) runs after square returns
}

printSquare(5); // pushes printSquare; the whole chain runs to completion
```

The stack for the call above grows and shrinks like this:

```
printSquare        printSquare        printSquare        printSquare
                   square             square
                                      multiply
   (step 1)          (step 2)          (step 3)           (step 4)
```

### What "synchronous" means

**Synchronous** code runs top to bottom, each statement fully completing before the next begins. Each call blocks the stack until it returns. This is intuitive and easy to reason about — but it has a fatal flaw for servers: **if a synchronous operation takes a long time, nothing else can run**, because the single thread is stuck.

```js
// blocking.js — DO NOT do this in a server
const fs = require('fs');

console.log('start');
// readFileSync BLOCKS the entire thread until the file is fully read.
const data = fs.readFileSync('./big-file.txt', 'utf8');
console.log('file length:', data.length);
console.log('end');
// Output is strictly: start -> file length -> end
```

While `readFileSync` runs, the call stack is occupied. Your Node process **cannot** accept a new HTTP request, run a timer, or do anything else. On a server handling many users, a single slow synchronous call freezes *everyone*.

> WARNING — Pitfall: The `*Sync` variants in Node's `fs` module, `JSON.parse` on huge payloads, and tight CPU loops are all **blocking**. In request-handling code paths they destroy concurrency. Reserve synchronous calls for startup/config loading, CLI scripts, or one-off tooling.

### What "asynchronous" means

**Asynchronous** operations let you *start* a task now and get notified *later* when it finishes, without blocking the thread in the meantime. The classic example is a timer:

```js
// async-basics.js
console.log('1: start');

setTimeout(() => {
  console.log('3: timer fired (later)'); // runs after the stack is empty
}, 0);

console.log('2: end');

// Output:
// 1: start
// 2: end
// 3: timer fired (later)   <-- even with a delay of 0ms!
```

Notice the order. Even with a `0` ms delay, the callback runs **after** the synchronous code finishes. Here's why, in one sentence you should memorize:

> **Asynchronous callbacks never run while the call stack is non-empty. They wait until the current synchronous code finishes and the stack is clear.**

### How async actually works: the runtime, not the language

Here is the crucial insight that confuses most people. The JavaScript *language* has no concept of timers, files, or networks. Those are provided by the **host environment** — the browser, or in our case **Node.js (built on libuj and the V8 engine)**. When you call `setTimeout` or `fs.readFile`, you are handing the work off to the runtime, which performs it *outside* the JS thread (using OS timers, a thread pool, or the OS kernel). When the work completes, the runtime places your callback into a **queue**. The **event loop** picks callbacks off that queue and pushes them onto the call stack — but only when the stack is empty.

```
   ┌─────────────────┐
   │   Call Stack    │  <- your JS runs here, one frame at a time
   └────────▲────────┘
            │ event loop pushes a ready callback
            │ ONLY when the stack is empty
   ┌────────┴────────┐         ┌──────────────────────────┐
   │  Callback Queue │ <─────  │  Node runtime / libuv:   │
   │  (macrotasks)   │         │  timers, fs threadpool,  │
   └─────────────────┘         │  network, etc.           │
                               └──────────────────────────┘
```

This is **non-blocking I/O**: the thread stays free to run other code while the OS handles the slow part. It is the entire reason a single-threaded runtime can serve thousands of simultaneous connections. We'll dissect the event loop, its phases, and the queues in the dedicated Event Loop module; for now, hold onto the model above.

| Aspect | Synchronous | Asynchronous |
| --- | --- | --- |
| Blocks the thread? | Yes, until it returns | No — hands work off, continues |
| Result available | Immediately (return value) | Later (callback / Promise) |
| Concurrency in a server | Destroys it | Enables it |
| Reasoning difficulty | Easy (linear) | Harder (deferred) |
| Typical Node APIs | `fs.readFileSync`, `JSON.parse` | `fs.readFile`, `fetch`, DB drivers |

---

## 2. Callbacks and the Error-First Convention

### What a callback is

A **callback** is simply a function you pass to another function, to be called *back* later — when an async operation completes. Before Promises existed, this was the *only* way to handle async results in JavaScript.

```js
// callback-intro.js
const fs = require('fs');

// fs.readFile is asynchronous: it returns immediately, and Node calls
// our callback once the file has been read (or an error occurred).
fs.readFile('./config.json', 'utf8', (err, contents) => {
  // This runs LATER, when the file is ready.
  if (err) {
    console.error('Failed to read file:', err.message);
    return; // bail out — do NOT continue as if it succeeded
  }
  console.log('File contents:', contents);
});

console.log('This logs first — readFile did not block.');
```

### The error-first convention

Node's entire standard library follows one rule for async callbacks, often called the **error-first** or **Node-style** callback convention:

1. The callback's **first parameter is the error** (`err`). It is `null` or `undefined` if the operation succeeded.
2. The **remaining parameters are the successful result(s)**.

```js
// error-first.js
function readUserFromDb(id, callback) {
  // Simulate an async DB lookup.
  setTimeout(() => {
    if (id <= 0) {
      // Convention: pass an Error as the FIRST argument on failure.
      return callback(new Error(`Invalid id: ${id}`));
    }
    // Convention: first arg null means "no error", result follows.
    callback(null, { id, name: 'Ada Lovelace' });
  }, 50);
}

readUserFromDb(1, (err, user) => {
  if (err) return console.error('Error:', err.message);
  console.log('Got user:', user);
});

readUserFromDb(-5, (err, user) => {
  if (err) return console.error('Error:', err.message); // this branch runs
  console.log('Got user:', user);
});
```

> BEST PRACTICE: Always check `err` first and `return` early when it is set. Forgetting the early return is a classic bug — execution falls through and you operate on `undefined` data.

### Callback hell / the pyramid of doom

Callbacks work fine for a single operation. The trouble starts when async operations **depend on each other** and must run in sequence. Each step nests inside the previous step's callback, and the code marches diagonally to the right — the **pyramid of doom**.

```js
// callback-hell.js — the problem we are about to solve
const fs = require('fs');

fs.readFile('./order.json', 'utf8', (err, orderRaw) => {
  if (err) return console.error(err);
  const order = JSON.parse(orderRaw);

  readUserFromDb(order.userId, (err, user) => {
    if (err) return console.error(err);

    chargeCard(user, order.total, (err, receipt) => {
      if (err) return console.error(err);

      fs.writeFile('./receipt.json', JSON.stringify(receipt), (err) => {
        if (err) return console.error(err);

        sendEmail(user.email, receipt, (err) => {
          if (err) return console.error(err);
          console.log('Order complete!'); // 5 levels deep
        });
      });
    });
  });
});
```

The problems with this style are not cosmetic:

- **Error handling is repeated** at every level (`if (err) return ...`) and easy to forget.
- **Control flow is hard to follow** — the logical sequence is buried in nesting.
- **Composition is painful** — running some steps in parallel, adding retries, or sharing variables across levels gets ugly fast.
- **No single place to `catch`** — one missed `if (err)` swallows a failure silently.

These pains are exactly what **Promises** were invented to fix.

---

## 3. Promises

### What a Promise is

A **Promise** is an object that represents the eventual result of an asynchronous operation. Instead of passing a callback *into* a function, the function **returns a Promise**, and you attach handlers *to* it. This flips control around and, crucially, makes async results **first-class values** you can pass around, return, and compose.

### The three states

A Promise is always in exactly one of three states:

| State | Meaning | Can it change? |
| --- | --- | --- |
| **pending** | The operation hasn't finished yet | Yes — to fulfilled or rejected |
| **fulfilled** | The operation succeeded; the Promise has a *value* | No — it is now **settled** |
| **rejected** | The operation failed; the Promise has a *reason* (usually an Error) | No — it is now **settled** |

Two terms worth knowing for interviews:

- A Promise is **settled** once it is either fulfilled or rejected. A settled Promise is **immutable** — its state and value never change again.
- "**Resolved**" is subtly different from "fulfilled": a Promise is *resolved* when it has been locked onto its final fate, which might be another pending Promise. For everyday use, treat fulfilled and "resolved successfully" as the same thing.

### Consuming a Promise: then / catch / finally

```js
// promise-consume.js
const fs = require('fs/promises'); // the Promise-based fs API

// readFile here returns a Promise instead of taking a callback.
fs.readFile('./config.json', 'utf8')
  .then((contents) => {
    // Runs when the Promise FULFILLS. `contents` is the value.
    console.log('Got contents:', contents);
  })
  .catch((err) => {
    // Runs when the Promise REJECTS. `err` is the reason.
    console.error('Failed:', err.message);
  })
  .finally(() => {
    // Runs after EITHER outcome — great for cleanup.
    console.log('Done attempting to read the file.');
  });
```

- `.then(onFulfilled, onRejected)` — `onFulfilled` receives the value; the optional second argument receives the rejection reason.
- `.catch(onRejected)` — shorthand for `.then(undefined, onRejected)`. Prefer it for readability.
- `.finally(onFinally)` — runs on settle regardless of outcome, receives no argument, and passes the value/reason through unchanged. Ideal for closing connections, hiding spinners, etc.

### Chaining and error propagation

The real power of Promises is **chaining**. Each `.then` returns a *new* Promise, so you can line up dependent steps **flat**, not nested. And here is the rule that eliminates callback hell's repeated error checks:

> **A single `.catch` at the end of a chain handles a rejection from *any* prior step.** Errors propagate down the chain, skipping `.then` handlers until they hit a `.catch`.

```js
// promise-chain.js — the pyramid of doom, flattened
const fs = require('fs/promises');

fs.readFile('./order.json', 'utf8')
  .then((orderRaw) => {
    const order = JSON.parse(orderRaw);
    // Return a value -> next .then receives it.
    // Return a Promise -> the chain WAITS for it to settle, then unwraps it.
    return readUserFromDb(order.userId).then((user) => ({ order, user }));
  })
  .then(({ order, user }) => chargeCard(user, order.total))
  .then((receipt) => fs.writeFile('./receipt.json', JSON.stringify(receipt)))
  .then(() => console.log('Order complete!'))
  .catch((err) => {
    // ONE place handles a failure from ANY step above.
    console.error('Order failed:', err.message);
  });
```

The two chaining rules to internalize:

1. **Returning a plain value** from a `.then` makes it the fulfillment value of the next `.then`.
2. **Returning a Promise** from a `.then` makes the chain *wait* for that Promise and *unwrap* it — you get its value, not the Promise itself, in the next `.then`. This is how you flatten dependent async steps.

```js
// chaining-values-vs-promises.js
Promise.resolve(2)
  .then((n) => n * 10)            // returns a value -> 20
  .then((n) => Promise.resolve(n + 5)) // returns a Promise -> waits, unwraps -> 25
  .then((n) => console.log(n));   // 25
```

> WARNING — Pitfall: **Forgetting to `return` inside a `.then`.** If you call an async function but don't `return` its Promise, the chain does not wait for it, and its errors escape your `.catch`. Every async operation in a chain must be returned.

### Recovering from errors mid-chain

A `.catch` doesn't have to be at the very end. Placed in the middle, it can *recover* — if it returns a value (instead of re-throwing), the chain continues in the fulfilled state.

```js
// catch-recovery.js
fetchUserSettings(userId)
  .catch((err) => {
    console.warn('Settings unavailable, using defaults:', err.message);
    return DEFAULT_SETTINGS; // recover -> chain continues fulfilled
  })
  .then((settings) => applySettings(settings)) // runs even if fetch failed
  .catch((err) => console.error('Fatal:', err.message)); // truly fatal errors
```

---

## 4. Creating Promises with `new Promise(resolve, reject)`

Most of the time you **consume** Promises that libraries hand you. Occasionally you need to **create** one — almost always to wrap a callback-based or event-based API that doesn't already return a Promise.

The constructor takes one function, the **executor**, which receives two functions: `resolve` and `reject`. Call `resolve(value)` to fulfill, `reject(reason)` to reject. The executor runs **synchronously and immediately**.

```js
// new-promise.js
function delay(ms) {
  return new Promise((resolve, reject) => {
    if (ms < 0) {
      return reject(new Error('Delay cannot be negative'));
    }
    // setTimeout is callback-based; we bridge it to a Promise.
    setTimeout(() => resolve(`waited ${ms}ms`), ms);
  });
}

delay(100)
  .then((msg) => console.log(msg))   // "waited 100ms"
  .catch((err) => console.error(err.message));
```

A realistic example: wrapping a callback API by hand.

```js
// wrap-callback.js
const fs = require('fs'); // classic callback-based fs

function readFilePromise(path) {
  return new Promise((resolve, reject) => {
    fs.readFile(path, 'utf8', (err, data) => {
      if (err) return reject(err); // error-first err -> reject
      resolve(data);               // success value -> resolve
    });
  });
}
```

Key rules for the executor:

- A Promise settles on the **first** `resolve` or `reject` call. Subsequent calls are **ignored** (no error, just no effect). This makes Promises safe — a result can't be delivered twice.
- If the executor **throws** synchronously, the Promise rejects with the thrown value automatically.
- `resolve` and `reject` take **exactly one argument**. To "return" multiple values, resolve with an object or array.

Convenience static methods create already-settled Promises:

```js
// already-settled.js
Promise.resolve(42);                 // immediately fulfilled with 42
Promise.reject(new Error('nope'));   // immediately rejected
// Promise.resolve on a thenable adopts its state — handy for normalizing inputs.
```

> WARNING — Pitfall: The **explicit Promise construction antipattern** — wrapping something in `new Promise` that is *already* a Promise. If a function already returns a Promise, just return it or chain it. Reserve `new Promise` for bridging non-Promise APIs (callbacks, events, timers).

> WARNING — Pitfall: Don't swallow errors inside the executor. If you do async work in the executor and an error happens in a *nested callback*, a synchronous `try/catch` won't catch it — you must call `reject` from that callback.

---

## 5. Promise Combinators: `all`, `allSettled`, `race`, `any`

When you have **multiple** Promises, these four static methods let you coordinate them. Choosing the right one is a frequent interview question because they differ in subtle, important ways.

### `Promise.all` — wait for all, fail fast

Takes an iterable of Promises. Fulfills with an **array of all results** (in input order) once **every** Promise fulfills. **Rejects immediately** if **any** Promise rejects (the others keep running but their results are discarded). Use it when you need *all* results and any failure means the whole operation failed.

```js
// promise-all.js
const [user, orders, prefs] = await Promise.all([
  fetchUser(id),       // these run CONCURRENTLY, not one after another
  fetchOrders(id),
  fetchPreferences(id),
]);
// Reaches here only if ALL three fulfilled.
// Results are in the SAME ORDER as the input, regardless of who finished first.
console.log(user.name, orders.length, prefs.theme);
```

### `Promise.allSettled` — wait for all, never short-circuit

Waits for **every** Promise to settle (fulfill *or* reject) and **never rejects**. Fulfills with an array of result objects, each either `{ status: 'fulfilled', value }` or `{ status: 'rejected', reason }`. Use it when you want *every* outcome regardless of individual failures — e.g., firing several independent notifications and reporting which succeeded.

```js
// promise-all-settled.js
const results = await Promise.allSettled([
  notifyEmail(user),
  notifySms(user),
  notifyPush(user),
]);

for (const r of results) {
  if (r.status === 'fulfilled') console.log('Sent:', r.value);
  else console.warn('Failed:', r.reason.message); // one failure doesn't stop the rest
}
```

### `Promise.race` — first to *settle* wins

Settles as soon as the **first** input Promise settles — whether it fulfills *or* rejects — adopting that outcome. Use it for **timeouts** and "first to respond, success or failure."

```js
// promise-race.js
function withTimeout(promise, ms) {
  const timeout = new Promise((_, reject) =>
    setTimeout(() => reject(new Error(`Timed out after ${ms}ms`)), ms)
  );
  // Whichever settles first wins: the real work, or the timeout rejection.
  return Promise.race([promise, timeout]);
}

await withTimeout(fetchSlowApi(), 3000); // rejects if fetch takes > 3s
```

### `Promise.any` — first to *fulfill* wins

Settles with the **first fulfillment**, ignoring rejections. Only if **all** inputs reject does it reject — with an `AggregateError` whose `.errors` array holds every reason. Use it for redundancy: query several mirrors, take the first that succeeds.

```js
// promise-any.js
try {
  const fastest = await Promise.any([
    fetch('https://mirror-1.example.com/data'),
    fetch('https://mirror-2.example.com/data'),
    fetch('https://mirror-3.example.com/data'),
  ]);
  console.log('First successful mirror:', fastest.url);
} catch (err) {
  // err is an AggregateError ONLY when EVERY mirror failed.
  console.error('All mirrors failed:', err.errors.map((e) => e.message));
}
```

### Comparison

| Combinator | Settles when… | Fulfills with | Rejects when | Short-circuits? | Classic use |
| --- | --- | --- | --- | --- | --- |
| `Promise.all` | all fulfill, or one rejects | array of all values | any one rejects | Yes (on first reject) | Fetch independent resources you all need |
| `Promise.allSettled` | all settle | array of `{status, value/reason}` | **never** | No | Independent tasks; report each outcome |
| `Promise.race` | first settles | first settler's value | first settler rejects | Yes (on first settle) | Timeouts; fastest-response wins |
| `Promise.any` | first fulfills, or all reject | first fulfillment value | **all** reject (`AggregateError`) | Yes (on first fulfill) | Redundant sources; first success |

> TIP: A memory hook — **all** = "all or nothing," **allSettled** = "tell me everything," **race** = "first across the line, win or lose," **any** = "first *winner*."

---

## 6. async / await

### What it is

`async`/`await` is **syntactic sugar over Promises**. It lets you write asynchronous code that *reads* like synchronous code while keeping the non-blocking behavior. It does not replace Promises — it consumes them more ergonomically.

- An `async` function **always returns a Promise**. A `return value` inside it fulfills that Promise with `value`; a `throw` rejects it.
- `await` **pauses the async function** until the awaited Promise settles, then evaluates to its fulfillment value (or throws its rejection reason). It does **not** block the thread — the function yields control back to the event loop, and other code runs while it waits.

```js
// async-basics.js
async function getUserName(id) {
  const user = await fetchUser(id); // pause until fetchUser's Promise settles
  return user.name;                 // fulfills the returned Promise with the name
}

// Because getUserName returns a Promise, you consume it like any other:
getUserName(1).then((name) => console.log(name));
// ...or await it inside another async function.
```

The chain from Section 3, rewritten with async/await, reads like a straightforward script:

```js
// async-chain.js
const fs = require('fs/promises');

async function processOrder() {
  const orderRaw = await fs.readFile('./order.json', 'utf8');
  const order = JSON.parse(orderRaw);
  const user = await readUserFromDb(order.userId);
  const receipt = await chargeCard(user, order.total);
  await fs.writeFile('./receipt.json', JSON.stringify(receipt));
  console.log('Order complete!');
}
```

### Error handling with try/catch

Because `await` *throws* on rejection, you handle async errors with ordinary `try/catch` — the same construct you use for synchronous errors. This is one of async/await's biggest wins.

```js
// async-try-catch.js
async function processOrder() {
  try {
    const order = JSON.parse(await fs.readFile('./order.json', 'utf8'));
    const user = await readUserFromDb(order.userId);
    const receipt = await chargeCard(user, order.total);
    await fs.writeFile('./receipt.json', JSON.stringify(receipt));
    return receipt; // fulfillment value of the returned Promise
  } catch (err) {
    // Catches a rejection from ANY await above, AND synchronous throws
    // like JSON.parse failing. One handler, like the chain's .catch.
    console.error('Order failed:', err.message);
    throw err; // re-throw so the CALLER knows it failed (don't swallow silently)
  } finally {
    console.log('Order attempt finished.'); // runs on success or failure
  }
}
```

> BEST PRACTICE: Don't wrap *every single* `await` in its own `try/catch` — that recreates callback hell's noise. Group related awaits under one `try/catch` at the level where you can meaningfully handle (or report) the error. Let errors propagate to a caller that knows what to do.

### Returning values and the await trap

A subtle but important detail: `return await somePromise` vs `return somePromise`.

```js
// return-await.js
async function a() {
  return fetchData();        // returns the Promise; fine in most cases
}

async function b() {
  return await fetchData();  // awaits, then returns the value
}
```

Functionally these usually behave the same to the caller (who awaits either way). The meaningful difference: inside a **`try/catch`**, you need `return await` for the local `catch` to handle a rejection. With a bare `return promise`, the function returns before the Promise settles, so a rejection escapes the local `try/catch` and surfaces in the caller instead.

```js
// return-await-trycatch.js
async function safe() {
  try {
    return await fetchData(); // ✅ a rejection is caught HERE
  } catch (err) {
    return fallbackValue;     // local recovery works
  }
}
```

> TIP: When the awaited Promise sits inside a `try` and you want the local `catch` to handle it, use `return await`. Outside a `try/catch`, prefer a bare `return promise` to avoid an unnecessary microtask tick.

### Top-level await

In **ES Modules** (`.mjs` files, or `"type": "module"` in `package.json`) you can use `await` at the top level of a module — no wrapping `async` function needed. In **CommonJS** (`require`) you cannot; wrap your logic in an `async` IIFE.

```js
// top-level-await.mjs (ES Module)
const data = await fetch('https://api.example.com').then((r) => r.json());
console.log(data);
```

```js
// commonjs-iife.js (CommonJS)
(async () => {
  const data = await fetchData();
  console.log(data);
})();
```

---

## 7. Sequential vs. Parallel Async Execution

This is the **#1 performance mistake** in real Node codebases, and a guaranteed interview probe. `await` pauses until *this* Promise settles before moving on. If you `await` operations one after another that **don't depend on each other**, you serialize work that could have overlapped.

### The slow way: accidental sequential execution

```js
// sequential-slow.js — each await waits for the previous to finish
async function loadDashboardSlow(id) {
  const user = await fetchUser(id);     // ~100ms
  const orders = await fetchOrders(id); // ~100ms — starts only AFTER user resolves
  const stats = await fetchStats(id);   // ~100ms — starts only AFTER orders resolves
  return { user, orders, stats };       // total ≈ 300ms
}
```

These three fetches are independent — `fetchOrders` doesn't need `user`. Yet they run back-to-back, taking the **sum** of their durations.

### The fast way: parallel execution with `Promise.all`

Start all the operations *first* (which kicks off the I/O concurrently), then await them together.

```js
// parallel-fast.js — all three run concurrently
async function loadDashboardFast(id) {
  // Calling the functions starts the work immediately and returns Promises.
  const userP = fetchUser(id);
  const ordersP = fetchOrders(id);
  const statsP = fetchStats(id);

  // Now await them together. Total ≈ max(100, 100, 100) = ~100ms.
  const [user, orders, stats] = await Promise.all([userP, ordersP, statsP]);
  return { user, orders, stats };
}
```

The mental rule:

> **If async operations are independent, run them in parallel with `Promise.all`. Only `await` sequentially when a later step genuinely needs an earlier step's result.**

### `await` in a loop: the hidden serializer

A `for` or `for...of` loop with `await` inside runs iterations **one at a time**. That's correct when each iteration depends on the previous one (e.g., paginated cursors), but it's a performance trap when the items are independent.

```js
// loop-sequential.js — processes one URL at a time (slow if independent)
async function fetchAllSequential(urls) {
  const results = [];
  for (const url of urls) {
    results.push(await fetch(url)); // waits for each before starting the next
  }
  return results;
}

// loop-parallel.js — kicks off all requests, then waits once
async function fetchAllParallel(urls) {
  // .map returns an array of Promises (work already started).
  return Promise.all(urls.map((url) => fetch(url)));
}
```

> WARNING — Pitfall: Unbounded parallelism. `Promise.all(thousands.map(...))` fires *thousands* of simultaneous requests — exhausting sockets, hitting rate limits, or melting the database. For large collections, use a **concurrency limit** (e.g., the `p-limit` library, or process in fixed-size batches).

```js
// bounded-batches.js — cap concurrency to BATCH_SIZE at a time
async function processInBatches(items, batchSize, worker) {
  const results = [];
  for (let i = 0; i < items.length; i += batchSize) {
    const batch = items.slice(i, i + batchSize);
    // Each batch runs in parallel; batches run sequentially.
    results.push(...(await Promise.all(batch.map(worker))));
  }
  return results;
}
```

---

## 8. Converting Callback APIs to Promises with `util.promisify`

Lots of older Node APIs and third-party libraries still use error-first callbacks. Rather than hand-wrapping each in `new Promise`, Node provides `util.promisify`, which converts an **error-first callback function** into one that **returns a Promise**.

```js
// promisify.js
const util = require('util');
const fs = require('fs');

// Turn fs.readFile (callback-based) into a Promise-returning function.
const readFileAsync = util.promisify(fs.readFile);

async function main() {
  const data = await readFileAsync('./config.json', 'utf8');
  console.log(data);
}
main().catch(console.error);
```

`util.promisify` works on **any function that follows the error-first convention**: the callback is the last argument, and its first parameter is the error.

```js
// promisify-custom.js
const util = require('util');

function loadById(id, callback) {
  setTimeout(() => {
    if (!id) return callback(new Error('id required'));
    callback(null, { id, ok: true }); // error-first: (err, result)
  }, 20);
}

const loadByIdAsync = util.promisify(loadById);
// await loadByIdAsync(7) -> { id: 7, ok: true }
```

> TIP: Many core modules now ship **Promise-native** variants — prefer those over promisifying. Use `require('fs/promises')` instead of promisifying `fs`, `require('dns/promises')`, `require('timers/promises')` (e.g., `setTimeout` that returns a Promise), and so on. Reach for `util.promisify` only when no Promise-based version exists.

```js
// native-promise-apis.js
const { setTimeout: sleep } = require('timers/promises');
await sleep(1000); // a Promise-based delay, no wrapper needed
```

There's also the inverse, `util.callbackify`, for the rarer case of exposing an async function to callback-expecting consumers. And for functions whose callback returns *multiple* success values, `util.promisify` supports a `[util.promisify.custom]` symbol to customize the wrapping — useful but uncommon.

---

## 9. Common Pitfalls & Gotchas

### Forgetting `await`

If you call an async function but forget `await`, you get the **Promise object itself**, not its value — and any rejection becomes an unhandled rejection.

```js
// forgot-await.js
async function bad() {
  const user = getUser();         // ❌ user is a Promise, not a user
  console.log(user.name);         // undefined — Promises have no .name
}

async function good() {
  const user = await getUser();   // ✅ user is the resolved value
  console.log(user.name);
}
```

Symptoms to recognize: `[object Promise]` in output, `undefined` where data should be, or a value that "isn't there yet." Many linters (ESLint's `no-floating-promises` via typescript-eslint, or `require-await`) catch these.

### Unhandled promise rejections

A rejected Promise with no `.catch` (and no surrounding `try/catch` on its `await`) is an **unhandled rejection**. In modern Node this **crashes the process by default** (`--unhandled-rejections=throw`).

```js
// unhandled-rejection.js
doRiskyThing(); // ❌ returns a Promise that might reject — nobody is catching

// Fix: handle it.
doRiskyThing().catch((err) => console.error('Handled:', err.message));
```

> BEST PRACTICE: Install a last-resort safety net, but treat any hit as a bug to fix at the source, not a substitute for proper handling:

```js
// safety-net.js
process.on('unhandledRejection', (reason) => {
  console.error('UNHANDLED REJECTION:', reason);
  process.exit(1); // fail fast; let your process manager restart cleanly
});
```

### `await` inside `forEach` (it does nothing)

`Array.prototype.forEach` **ignores** the return value of its callback — so it ignores the Promise an `async` callback returns. The loop does **not** wait, and any errors are lost.

```js
// foreach-await-broken.js
async function broken(urls) {
  urls.forEach(async (url) => {
    const data = await fetch(url); // these all fire at once...
    save(data);                    // ...and forEach does NOT wait for any of them
  });
  console.log('done'); // ❌ logs BEFORE any save completes
}
```

Fixes, depending on intent:

```js
// foreach-await-fixed.js
// Sequential (wait for each):
async function sequential(urls) {
  for (const url of urls) {
    const data = await fetch(url);
    save(data);
  }
  console.log('done'); // ✅ truly after all saves
}

// Parallel (start all, wait once):
async function parallel(urls) {
  await Promise.all(urls.map(async (url) => save(await fetch(url))));
  console.log('done'); // ✅ after all complete
}
```

> WARNING — Pitfall: `map`, `filter`, `reduce`, and `sort` are also not async-aware in the way you'd hope. `.map(async ...)` yields an **array of Promises** (wrap in `Promise.all`); async predicates in `.filter` don't work as written; `.sort` cannot use an async comparator at all.

### Mixing callbacks and Promises / double-settling

Don't call both a callback and resolve a Promise for the same result, and never `resolve` then `reject` the same Promise expecting both to matter — only the first settle counts.

### Creating Promises but not chaining their errors

Returning early without `return`-ing an inner Promise (see Section 3) detaches it from the chain, so its rejection bypasses your `catch`. Always `return` async work you want awaited.

### Swallowing errors with empty catches

`.catch(() => {})` or a `try/catch` that does nothing hides failures and makes debugging miserable. At minimum, log; ideally, handle or re-throw.

---

## 10. Best Practices

- **Prefer async/await** for readability; drop to raw `.then` chains only for short, fluent transformations or when you need a combinator's array semantics directly.
- **Run independent work in parallel** with `Promise.all`; reserve sequential `await` for genuine data dependencies.
- **Always handle rejections** — a `try/catch`, a `.catch`, or propagation to a caller who has one. Never leave a Promise floating.
- **Don't block the event loop.** Avoid `*Sync` calls and heavy CPU work in request paths; offload CPU-bound work to worker threads or a queue.
- **Bound your concurrency.** Cap parallel I/O against external systems to respect rate limits and connection pools.
- **Use native Promise APIs** (`fs/promises`, `timers/promises`, `dns/promises`) before reaching for `util.promisify`.
- **Return Promises** from functions that do async work — don't bury results in callbacks. Returnable async values compose; callbacks don't.
- **Re-throw after logging** when a function can't fully recover, so the caller can decide. Don't silently swallow.
- **Avoid the `new Promise` antipattern** — only construct Promises to wrap non-Promise (callback/event) APIs.
- **Add timeouts** to external calls with `Promise.race` (or `AbortController`/`AbortSignal.timeout()`), so a hung dependency can't hang your request forever.
- **Use `Promise.allSettled`** when you need every outcome and partial failure is acceptable; use `Promise.all` when any failure should abort.
- **Install a global `unhandledRejection` handler** as a safety net that fails fast — but fix the root cause every time it fires.

---

## 11. Key Interview Points

- **Why is Node single-threaded yet able to handle high concurrency?** Because I/O is non-blocking: slow operations are delegated to the OS/libuv (kernel async I/O plus a thread pool), freeing the single JS thread to run other code; the event loop schedules callbacks when work completes.
- **What are the three states of a Promise?** Pending, fulfilled, rejected. Once **settled** (fulfilled or rejected) it is immutable — its value/reason never changes.
- **What does `await` actually do — does it block the thread?** No. It pauses the *current async function* and yields control to the event loop, resuming when the awaited Promise settles. Other code runs meanwhile; the thread is not blocked.
- **`Promise.all` vs `allSettled` vs `race` vs `any`?** `all` = all fulfill or reject on first failure; `allSettled` = wait for all, never rejects, returns per-item status; `race` = first to *settle* (win or lose); `any` = first to *fulfill*, rejects with `AggregateError` only if all fail.
- **What's the difference between sequential and parallel awaits, and when does each matter?** Sequential `await`s sum their durations and are correct only for dependent steps; independent operations should run in parallel via `Promise.all` to overlap their wait times.
- **Why doesn't `await` work inside `forEach`?** `forEach` ignores its callback's return value, so it never awaits the returned Promise. Use a `for...of` loop (sequential) or `Promise.all(map(...))` (parallel).
- **What is an unhandled rejection and what happens in modern Node?** A rejected Promise with no handler; by default it terminates the process. Handle every rejection and add a `process.on('unhandledRejection')` safety net.
- **What is `util.promisify` for?** Converting an error-first-callback function into one returning a Promise. Prefer built-in `*/promises` modules where they exist.
- **How does error propagation differ between callbacks and Promises?** Callbacks require manual `if (err)` checks at every level; Promise chains propagate errors down to the nearest `.catch`, and async/await routes them to `try/catch`.
- **Microtask vs macrotask — which runs first?** Microtasks (Promise reactions, `queueMicrotask`) drain *completely* after each macrotask and before the next one, so a `.then` callback runs before a `setTimeout(…, 0)` callback queued at the same time.
- **What does an `async` function return, and how do `return`/`throw` behave inside it?** Always a Promise; `return value` fulfills it, `throw`/rejected `await` rejects it.
- **When do you need `return await` vs `return`?** Inside a `try/catch` you need `return await` for the local `catch` to handle the rejection; otherwise a bare `return promise` avoids an extra tick.

---

## 12. Microtasks vs. Macrotasks (Event Loop Preview)

This is a preview that the **Event Loop** module expands on, but it directly affects async ordering, so you need the essentials now.

The runtime maintains two kinds of pending-work queues:

- **Macrotask queue** (a.k.a. the task queue): timers (`setTimeout`, `setInterval`), I/O callbacks, `setImmediate`. Each event-loop iteration runs **one** macrotask.
- **Microtask queue**: **Promise** reactions (`.then` / `.catch` / `.finally` callbacks and resumed `await`s), `queueMicrotask`, and `process.nextTick` (which in Node has its *own* even-higher-priority queue). After a macrotask finishes — and after the currently running synchronous code finishes — the engine **drains the entire microtask queue** before taking the next macrotask.

The rule that surprises people:

> **All available microtasks run before the next macrotask.** A `Promise.then` callback queued at the same moment as a `setTimeout(…, 0)` callback runs *first*.

```js
// micro-vs-macro.js
console.log('1: sync');

setTimeout(() => console.log('4: setTimeout (macrotask)'), 0);

Promise.resolve().then(() => console.log('3: promise (microtask)'));

console.log('2: sync');

// Output:
// 1: sync
// 2: sync
// 3: promise (microtask)   <- microtasks drain before the macrotask
// 4: setTimeout (macrotask)
```

In Node specifically, `process.nextTick` callbacks run before *other* microtasks (Promises), which run before macrotasks. Starving the loop is possible: an endlessly recursive `process.nextTick` (or a self-queuing microtask) can prevent timers and I/O from ever running. The full ordering — Node's event loop phases (timers, pending callbacks, poll, check, close), where `setImmediate` fits, and how this differs from the browser — is covered next.

---

## 13. Practice Exercises

Work these in order; each builds on the last. Try them before peeking at the solutions.

1. **Promisify by hand.** Write `delay(ms)` returning a Promise that fulfills with the string `"done"` after `ms` milliseconds, and rejects with an `Error` if `ms` is negative. Demonstrate it with async/await.

2. **Flatten callback hell.** You're given three error-first callback functions: `getUser(id, cb)`, `getPostsForUser(user, cb)`, and `getCommentsForPosts(posts, cb)`. Rewrite a nested-callback pipeline that fetches a user, then their posts, then comments on those posts — using async/await with a single `try/catch`. (Stub the three functions with `setTimeout`.)

3. **Sequential vs parallel timing.** Given an async `fetchPrice(symbol)` that takes ~200ms, fetch prices for `['AAPL', 'GOOG', 'MSFT']` two ways — sequentially and in parallel — and log the elapsed time for each. Confirm parallel is roughly 3× faster.

4. **Pick the right combinator.** Implement `fetchWithTimeout(promise, ms)` using `Promise.race` so it rejects with a timeout error if `promise` doesn't settle in time. Then implement `firstSuccessful(promises)` that returns the first fulfillment and only rejects (with all reasons) if every input rejects.

5. **Bounded concurrency.** Implement `mapWithConcurrency(items, limit, worker)` that runs `worker(item)` over all items but never more than `limit` at once, preserving result order. (Don't use an external library.)

6. **Microtask ordering.** Without running it, predict the exact output of the snippet in the solution, then run it to verify. Explain *why* each line appears where it does.

### Solutions

**Solution 1 — Promisify by hand**

```js
// sol1.js
function delay(ms) {
  return new Promise((resolve, reject) => {
    if (ms < 0) return reject(new Error('ms must be >= 0'));
    setTimeout(() => resolve('done'), ms);
  });
}

(async () => {
  try {
    console.log(await delay(100)); // "done" after ~100ms
    await delay(-1);               // throws -> caught below
  } catch (err) {
    console.error('Caught:', err.message); // "ms must be >= 0"
  }
})();
```

**Solution 2 — Flatten callback hell**

```js
// sol2.js
const util = require('util');

// Error-first callback stubs.
const getUser = (id, cb) =>
  setTimeout(() => cb(null, { id, name: 'Grace' }), 30);
const getPostsForUser = (user, cb) =>
  setTimeout(() => cb(null, [{ id: 1, by: user.id }, { id: 2, by: user.id }]), 30);
const getCommentsForPosts = (posts, cb) =>
  setTimeout(() => cb(null, posts.flatMap((p) => [`c on ${p.id}`])), 30);

// Promisify the callback APIs.
const getUserAsync = util.promisify(getUser);
const getPostsAsync = util.promisify(getPostsForUser);
const getCommentsAsync = util.promisify(getCommentsForPosts);

async function pipeline(id) {
  try {
    const user = await getUserAsync(id);
    const posts = await getPostsAsync(user);     // depends on user
    const comments = await getCommentsAsync(posts); // depends on posts
    return { user, posts, comments };
  } catch (err) {
    console.error('Pipeline failed:', err.message);
    throw err;
  }
}

pipeline(7).then((r) => console.log(JSON.stringify(r, null, 2)));
// Note: these steps are genuinely dependent, so sequential await is correct here.
```

**Solution 3 — Sequential vs parallel timing**

```js
// sol3.js
function fetchPrice(symbol) {
  return new Promise((resolve) =>
    setTimeout(() => resolve({ symbol, price: Math.random() * 100 }), 200)
  );
}

async function sequential(symbols) {
  const t = Date.now();
  const out = [];
  for (const s of symbols) out.push(await fetchPrice(s)); // ~200ms each
  console.log('sequential:', Date.now() - t, 'ms');       // ≈ 600ms
  return out;
}

async function parallel(symbols) {
  const t = Date.now();
  const out = await Promise.all(symbols.map(fetchPrice));  // all at once
  console.log('parallel:', Date.now() - t, 'ms');         // ≈ 200ms
  return out;
}

(async () => {
  const symbols = ['AAPL', 'GOOG', 'MSFT'];
  await sequential(symbols); // ~600ms
  await parallel(symbols);   // ~200ms — roughly 3x faster
})();
```

**Solution 4 — Pick the right combinator**

```js
// sol4.js
function fetchWithTimeout(promise, ms) {
  const timeout = new Promise((_, reject) =>
    setTimeout(() => reject(new Error(`Timeout after ${ms}ms`)), ms)
  );
  // race: whichever settles first (the real work or the timeout) wins.
  return Promise.race([promise, timeout]);
}

function firstSuccessful(promises) {
  // Promise.any: first fulfillment wins; rejects only if ALL reject.
  return Promise.any(promises);
}

// Demo
(async () => {
  const slow = new Promise((res) => setTimeout(() => res('slow done'), 1000));
  try {
    await fetchWithTimeout(slow, 300);
  } catch (err) {
    console.log(err.message); // "Timeout after 300ms"
  }

  try {
    const winner = await firstSuccessful([
      Promise.reject(new Error('mirror A down')),
      new Promise((res) => setTimeout(() => res('mirror B ok'), 50)),
      Promise.reject(new Error('mirror C down')),
    ]);
    console.log(winner); // "mirror B ok"
  } catch (err) {
    // Only if EVERY input rejected:
    console.log('all failed:', err.errors.map((e) => e.message));
  }
})();
```

**Solution 5 — Bounded concurrency**

```js
// sol5.js
async function mapWithConcurrency(items, limit, worker) {
  const results = new Array(items.length); // preserve order by index
  let nextIndex = 0;

  // Each "runner" pulls the next index until items are exhausted.
  async function runner() {
    while (nextIndex < items.length) {
      const current = nextIndex++;           // claim an index
      results[current] = await worker(items[current], current);
    }
  }

  // Launch up to `limit` runners; they share the index counter.
  const runners = Array.from({ length: Math.min(limit, items.length) }, runner);
  await Promise.all(runners);
  return results;
}

// Demo: process 7 items, max 2 at a time.
(async () => {
  const work = (n) =>
    new Promise((res) => setTimeout(() => res(n * n), 100));
  const out = await mapWithConcurrency([1, 2, 3, 4, 5, 6, 7], 2, work);
  console.log(out); // [1, 4, 9, 16, 25, 36, 49] — order preserved
})();
```

**Solution 6 — Microtask ordering**

```js
// sol6.js
console.log('A');

setTimeout(() => console.log('B (timeout)'), 0);

Promise.resolve().then(() => {
  console.log('C (then 1)');
  Promise.resolve().then(() => console.log('D (then nested)'));
});

queueMicrotask(() => console.log('E (queueMicrotask)'));

console.log('F');

// Predicted output:
// A           <- synchronous
// F           <- synchronous
// C (then 1)  <- microtask, drained after sync code
// E (queueMicrotask) <- microtask, queued after C, runs after C
// D (then nested)    <- microtask queued DURING C's run; still drained
//                       in this same microtask flush, before any macrotask
// B (timeout) <- macrotask, runs only after ALL microtasks drain
```

Explanation: synchronous `console.log`s (`A`, `F`) run first. When the stack clears, the microtask queue drains *entirely* — including microtasks (`D`) that get added *while* draining — before the event loop ever reaches the macrotask (`B`). `C` was queued before `E`, so it runs first; `D` is queued during `C` and joins the same flush after `E`.

---

## 14. Summary / Cheat Sheet

| Concept | One-line takeaway |
| --- | --- |
| Call stack | One thread, one stack; only one frame runs at a time. |
| Synchronous | Blocks the thread until it returns — bad in request paths. |
| Asynchronous | Hands work to the runtime; callback runs later when the stack is clear. |
| Error-first callback | `(err, result)` — check `err` first and `return` early. |
| Callback hell | Nested callbacks for dependent async steps; Promises fix it. |
| Promise states | pending → fulfilled / rejected; settled is immutable. |
| `.then` / `.catch` / `.finally` | value handler / error handler / cleanup-on-settle. |
| Chaining | Return a value or a Promise from `.then`; one `.catch` handles the whole chain. |
| `new Promise` | Only to wrap callback/event APIs; settles on first `resolve`/`reject`. |
| `Promise.all` | All values, or reject on first failure. |
| `Promise.allSettled` | Every outcome; never rejects. |
| `Promise.race` | First to settle (win or lose). |
| `Promise.any` | First to fulfill; `AggregateError` if all reject. |
| `async` function | Always returns a Promise; `return` fulfills, `throw` rejects. |
| `await` | Pauses the function (not the thread) until the Promise settles. |
| try/catch | Handles `await` rejections and sync throws together. |
| Sequential vs parallel | Independent work → `Promise.all`; dependent work → sequential `await`. |
| `await` in `forEach` | Broken — use `for...of` (sequential) or `Promise.all(map)` (parallel). |
| `util.promisify` | Callback API → Promise; prefer native `*/promises` modules. |
| Forgot `await` | You get a Promise, not a value; rejection goes unhandled. |
| Unhandled rejection | Crashes Node by default — always handle or propagate. |
| Microtasks vs macrotasks | All microtasks (Promises) drain before the next macrotask (timers). |

> BEST PRACTICE: If you remember just three things from this module — (1) `await` pauses the function but not the thread, (2) run independent async work with `Promise.all` instead of sequential `await`, and (3) every Promise must have an owner who handles its rejection — you'll already write better async Node code than most.
