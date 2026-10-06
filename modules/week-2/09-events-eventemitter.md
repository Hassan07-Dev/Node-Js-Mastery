# Events & EventEmitter (In Depth)

Node.js is, at its core, an **event-driven** runtime. If you understand events deeply, you understand the philosophy that everything else in Node — streams, HTTP servers, sockets, file watchers, child processes — is built upon. This module takes you from the conceptual foundations of event-driven programming all the way to advanced patterns like async iterators over event streams and building decoupled pub/sub systems.

## What You'll Learn

By the end of this module you will be able to:

- Explain **event-driven programming** from first principles and identify where Node uses it internally.
- Use the full **`EventEmitter` API**: `on`/`addListener`, `once`, `emit` (with arguments), and `prependListener`/`prependOnceListener`.
- **Remove listeners** correctly with `removeListener`/`off` and `removeAllListeners`, and understand the subtle gotchas.
- **Introspect** an emitter with `listenerCount`, `eventNames`, `rawListeners`, `setMaxListeners`/`getMaxListeners`.
- Understand the special **`'error'` event** and why an unhandled one crashes the process.
- Diagnose and fix the **`MaxListenersExceededWarning`** memory-leak warning.
- **Extend `EventEmitter`** in your own classes to build emitting components.
- Handle **asynchronous listeners** safely, and use `events.once()` and `events.on()` as async iterators.
- Choose correctly between **events, callbacks, and promises**.
- Apply **real-world patterns**: publish/subscribe, decoupling modules, and observable lifecycles.

---

## 1. Event-Driven Programming: The Concept and the WHY

### What "event-driven" actually means

In a traditional, synchronous, top-to-bottom program, control flows like a recipe: do step 1, then step 2, then step 3. The program asks "what should I do next?" and the answer is always "the next line."

In an **event-driven** program, the model is inverted. Instead of asking "what do I do next?", the program declares "**when** *this* happens, run *that*." It then sits in a loop waiting for things to happen, and reacts. The "things that happen" are **events**, and the functions that react to them are **listeners** (also called **handlers**).

This is the same mental model you already know from the browser:

```js
// Browser: you don't poll the button asking "are you clicked yet?"
// You register interest in the 'click' event and react when it fires.
button.addEventListener('click', () => {
  console.log('The user clicked the button');
});
```

Node uses the *exact same idea* on the server. The difference is that the browser's event target is the DOM, while Node's general-purpose event mechanism is the **`EventEmitter`** class from the built-in `events` module.

### Why Node is built this way

Node runs JavaScript on a **single main thread** with a **non-blocking, asynchronous I/O** model. The runtime cannot afford to sit and wait for a disk read or a network response — that would freeze the entire process for every other request. Instead, Node kicks off the I/O operation, registers a callback/listener, and continues. When the operating system signals that the I/O is complete, Node fires the corresponding event and runs the listener.

The event-driven model is therefore not a stylistic choice; it is the **natural fit** for an asynchronous, single-threaded server. It lets one thread juggle thousands of concurrent connections by reacting to events as they complete, rather than blocking on each one.

> TIP: Think of the **event loop** as the engine and **events** as the fuel. The event loop continuously pulls completed operations off its queues and dispatches their listeners. The `EventEmitter` class is the user-facing API for participating in this same emit/listen pattern with your *own* events.

### Where Node uses events internally

Almost every core abstraction in Node is an `EventEmitter` (or wraps one). A few you will meet constantly:

| Node API | Notable events | What it represents |
| --- | --- | --- |
| `http.Server` | `'request'`, `'connection'`, `'close'` | A new HTTP request arrived, a socket connected, server closing |
| `net.Socket` / streams | `'data'`, `'end'`, `'error'`, `'close'`, `'drain'` | Bytes flowing through a TCP/file/stream pipe |
| `process` | `'exit'`, `'uncaughtException'`, `'SIGINT'`, `'beforeExit'` | The lifecycle of the running process itself |
| `fs.FSWatcher` | `'change'`, `'rename'` | A watched file/directory changed |
| `child_process.ChildProcess` | `'exit'`, `'message'`, `'close'` | A spawned subprocess's lifecycle |
| Streams (`Readable`/`Writable`) | `'data'`, `'readable'`, `'finish'`, `'pipe'` | The backbone of files, HTTP bodies, compression, etc. |

When you write `server.on('request', handler)`, you are using the *same* `EventEmitter` API you are about to master. Learning `EventEmitter` is therefore learning the lingua franca of the entire Node ecosystem.

```js
// A classic example: http.Server IS an EventEmitter.
const http = require('node:http');

const server = http.createServer(); // returns an EventEmitter

// 'request' fires every time a client hits the server.
server.on('request', (req, res) => {
  res.end('Hello from an event listener!');
});

server.listen(3000, () => console.log('Listening on http://localhost:3000'));
```

> Note: Modern Node lets you import core modules with the `node:` prefix (`require('node:events')`). The prefix is recommended — it disambiguates from npm packages and is unambiguous to readers. Both `require('events')` and `require('node:events')` work.

---

## 2. The `EventEmitter` Class

The `events` module exports the `EventEmitter` class. Every emitter instance maintains an internal registry mapping **event names** (strings or symbols) to **arrays of listener functions**.

```js
const EventEmitter = require('node:events');

const emitter = new EventEmitter();
```

> Note: In modern Node, `require('node:events')` returns the `EventEmitter` class directly (it is the module's default export), and it is *also* available as `require('node:events').EventEmitter`. Both of these are the same class:
>
> ```js
> const EventEmitter = require('node:events');            // the class itself
> const { EventEmitter } = require('node:events');        // also the class
> ```

### 2.1 `on` / `addListener` — register a listener

`emitter.on(eventName, listener)` registers a listener that runs **every time** the named event is emitted. `addListener` is a pure alias — identical behavior, kept for historical/DOM-familiarity reasons.

```js
const EventEmitter = require('node:events');
const emitter = new EventEmitter();

// Register a listener for the 'greet' event.
emitter.on('greet', () => {
  console.log('Hello!');
});

emitter.emit('greet'); // -> "Hello!"
emitter.emit('greet'); // -> "Hello!"  (fires again every time)
```

Key facts:

- Listeners fire **synchronously**, in the order they were registered, in the same tick as `emit()` (more on this in §9).
- `on` returns the emitter itself, so calls are **chainable**.
- The **same** function can be added multiple times; it will then be called multiple times per emit.

```js
const emitter = new EventEmitter();

emitter
  .on('tick', () => console.log('first'))
  .on('tick', () => console.log('second'))   // chaining
  .on('tick', () => console.log('third'));

emitter.emit('tick');
// first
// second
// third   (registration order is preserved)
```

### 2.2 `emit` — fire an event, with arguments

`emitter.emit(eventName, ...args)` synchronously calls every listener registered for `eventName`, passing along any extra arguments. It returns `true` if the event had at least one listener, and `false` otherwise.

```js
const EventEmitter = require('node:events');
const emitter = new EventEmitter();

emitter.on('order', (id, amount) => {
  console.log(`Order ${id} for $${amount}`);
});

const hadListeners = emitter.emit('order', 42, 99.95);
// -> "Order 42 for $99.95"
console.log(hadListeners); // -> true

console.log(emitter.emit('nonexistent')); // -> false (no listener)
```

#### `this` inside a listener

When you use a **regular function** (not an arrow function) as a listener, `this` is bound to the emitter instance. This is occasionally handy for chaining or introspection:

```js
const emitter = new EventEmitter();

emitter.on('ping', function () {
  // `this` === emitter (only with a regular function, NOT an arrow fn)
  console.log(this === emitter); // -> true
});

emitter.emit('ping');
```

> WARNING — Pitfall: With an **arrow function**, `this` is lexically captured from the surrounding scope, *not* the emitter. If you rely on `this` being the emitter, use a regular function. In practice, prefer closures over relying on `this` — it is clearer.

### 2.3 `once` — register a one-time listener

`emitter.once(eventName, listener)` registers a listener that fires **at most once**. After it fires, it is automatically removed.

```js
const EventEmitter = require('node:events');
const server = new EventEmitter();

server.once('ready', () => {
  console.log('Server initialized — this runs exactly once.');
});

server.emit('ready'); // -> runs the listener, then auto-removes it
server.emit('ready'); // -> nothing happens
```

`once` is perfect for one-shot lifecycle moments: a connection becoming ready, a process having started, a stream finishing. Internally, `once` wraps your listener in a function that removes itself before calling your code, so you do not have to manage cleanup.

### 2.4 `prependListener` / `prependOnceListener` — register at the front

By default, `on`/`once` **append** the listener to the end of the array, so it runs *after* previously-registered listeners. Sometimes you need a listener to run **first** — for example, a framework injecting a logging or auth hook ahead of user listeners.

- `prependListener(eventName, listener)` adds to the **front** of the listener array.
- `prependOnceListener(eventName, listener)` does the same but as a one-time listener.

```js
const EventEmitter = require('node:events');
const emitter = new EventEmitter();

emitter.on('boot', () => console.log('2: normal listener'));
emitter.prependListener('boot', () => console.log('1: prepended listener'));

emitter.emit('boot');
// 1: prepended listener   <-- ran first despite being registered second
// 2: normal listener
```

> TIP: You rarely need prepend in application code, but it is invaluable when **building libraries/frameworks** where you must guarantee your hook runs before any consumer's listeners.

### 2.5 The `'newListener'` and `'removeListener'` meta-events

`EventEmitter` emits two special meta-events about *itself*:

- `'newListener'` fires **before** a listener is added, with `(eventName, listener)`.
- `'removeListener'` fires **after** a listener is removed.

These let you build emitters that react to subscription changes (e.g., lazily start producing data only when someone is listening).

```js
const EventEmitter = require('node:events');
const emitter = new EventEmitter();

emitter.on('newListener', (event, listener) => {
  console.log(`A listener was added for "${event}"`);
});

emitter.on('data', () => {}); // -> "A listener was added for "data""
```

> WARNING — Pitfall: If a `'newListener'` handler itself adds a listener for the *same* event name being registered, you can create infinite recursion. Use `'newListener'` carefully and avoid re-registering the same event inside it.

---

## 3. Removing Listeners

Listeners that are never removed are the #1 source of memory leaks in long-lived Node processes. Knowing how to remove them precisely is essential.

### 3.1 `removeListener` / `off` — remove a specific listener

`emitter.off(eventName, listener)` (alias `removeListener`) removes **one** listener — and it must be the **same function reference** you registered. `off` was added in Node 10 as a shorter, DOM-aligned alias; they are identical.

```js
const EventEmitter = require('node:events');
const emitter = new EventEmitter();

function onData(chunk) {
  console.log('Got:', chunk);
}

emitter.on('data', onData);
emitter.emit('data', 'A'); // -> "Got: A"

emitter.off('data', onData); // remove by reference
emitter.emit('data', 'B'); // -> (nothing)
```

> WARNING — Pitfall: You **cannot** remove an anonymous or inline arrow function, because you have no reference to it:
>
> ```js
> emitter.on('data', (c) => console.log(c)); // no reference kept
> emitter.off('data', (c) => console.log(c)); // DIFFERENT function — removes nothing!
> ```
>
> If you ever intend to remove a listener, store it in a named variable or property first.

#### Removing the *most recently added* matching listener

If the same function is registered multiple times, a single `off`/`removeListener` removes **only the most recently added instance**, not all of them. Call it repeatedly to remove more.

```js
const emitter = new EventEmitter();
function f() {}
emitter.on('e', f);
emitter.on('e', f);

console.log(emitter.listenerCount('e')); // -> 2
emitter.off('e', f);
console.log(emitter.listenerCount('e')); // -> 1  (removed only one)
```

#### Removing during emit

Removing a listener while an event is being emitted does **not** affect listeners already scheduled in the *current* emit. The change takes effect on the *next* emit. This prevents surprising "the array shifted under me" bugs.

```js
const emitter = new EventEmitter();

const a = () => {
  console.log('A');
  emitter.off('go', b); // remove B mid-emit
};
const b = () => console.log('B');

emitter.on('go', a);
emitter.on('go', b);

emitter.emit('go');
// A
// B            <-- B still runs this emit; it was already scheduled
emitter.emit('go');
// A            <-- B is gone now
```

### 3.2 `removeAllListeners` — bulk removal

`emitter.removeAllListeners([eventName])` removes every listener for a specific event, or — if called with **no argument** — *all* listeners for *all* events.

```js
const EventEmitter = require('node:events');
const emitter = new EventEmitter();

emitter.on('a', () => {});
emitter.on('b', () => {});

emitter.removeAllListeners('a'); // remove only 'a' listeners
emitter.removeAllListeners();    // remove EVERYTHING
```

> WARNING — Pitfall: Calling `removeAllListeners()` with no arguments is a blunt instrument. On an emitter you do not fully own (e.g., a shared `process` object, an HTTP server you got from a framework), this can remove listeners that *other code* depends on, causing baffling breakage. Prefer removing specific listeners you registered.

> BEST PRACTICE: For cleanup, store an **`AbortController`** and use the `signal` option (see §9.3) — it lets you remove a *group* of related listeners atomically without tracking each function reference.

---

## 4. Introspection: Inspecting an Emitter

Sometimes you need to ask an emitter about its own state — how many listeners it has, what events it knows about, etc. This is invaluable for debugging leaks and for building diagnostics.

### 4.1 `listenerCount`

`emitter.listenerCount(eventName[, listener])` returns the number of listeners for a given event. Passing the optional `listener` (Node 19.8+) counts only occurrences of that specific function.

```js
const EventEmitter = require('node:events');
const emitter = new EventEmitter();

emitter.on('data', () => {});
emitter.on('data', () => {});
emitter.on('end', () => {});

console.log(emitter.listenerCount('data')); // -> 2
console.log(emitter.listenerCount('end'));  // -> 1
console.log(emitter.listenerCount('none')); // -> 0
```

### 4.2 `eventNames`

`emitter.eventNames()` returns an array of all event names (strings and symbols) that currently have at least one registered listener.

```js
const emitter = new EventEmitter();
emitter.on('connect', () => {});
emitter.once('close', () => {});

console.log(emitter.eventNames()); // -> [ 'connect', 'close' ]
```

### 4.3 `listeners` and `rawListeners`

- `emitter.listeners(eventName)` returns a **copy** of the listener array for an event.
- `emitter.rawListeners(eventName)` is similar, but for `once` listeners it returns the **wrapper** function (which carries a `.listener` property pointing at your original function). This lets you inspect or re-register a `once` listener without it being "used up."

```js
const EventEmitter = require('node:events');
const emitter = new EventEmitter();

emitter.once('log', () => console.log('once!'));

// rawListeners gives the once-wrapper; .listener is your original fn.
const [wrapper] = emitter.rawListeners('log');
console.log(typeof wrapper);          // -> 'function'
console.log(typeof wrapper.listener); // -> 'function' (the original)

// Calling the raw wrapper directly does NOT consume the once registration
// in the emitter's list — useful for advanced introspection.
```

### 4.4 `setMaxListeners` / `getMaxListeners`

Each emitter has a maximum listener count *per event* (default **10**) used purely as a **leak-detection heuristic**. Exceeding it logs a `MaxListenersExceededWarning` (see §6). These methods read and adjust that threshold for a single emitter.

```js
const emitter = new EventEmitter();

console.log(emitter.getMaxListeners()); // -> 10 (default)

emitter.setMaxListeners(25); // raise the threshold for THIS emitter
console.log(emitter.getMaxListeners()); // -> 25

emitter.setMaxListeners(0); // 0 means "unlimited" (disables the warning)
```

You can change the **global default** for all emitters via the static property:

```js
const EventEmitter = require('node:events');
EventEmitter.defaultMaxListeners = 20; // affects emitters that didn't set their own
```

> WARNING — Pitfall: Bumping `setMaxListeners` to silence the warning is often the *wrong* fix. The warning usually means you are adding listeners in a loop and never removing them. Raise the limit only when you genuinely, intentionally need many listeners (e.g., a high-fan-out event bus).

---

## 5. The Special `'error'` Event

This is one of the most interview-asked and bug-prone parts of `EventEmitter`. Read it carefully.

### Why `'error'` is special

`EventEmitter` gives the event name `'error'` unique semantics. If an emitter emits an `'error'` event and there is **no listener registered for it**, the `EventEmitter` will **throw** the error. If that throw is not caught, it becomes an **uncaught exception** and **crashes the Node process**.

```js
const EventEmitter = require('node:events');
const emitter = new EventEmitter();

// No 'error' listener registered...
emitter.emit('error', new Error('Boom!'));
// Throws -> "Uncaught Error: Boom!" -> process exits with a non-zero code.
```

This is intentional and *good*: errors must never be silently swallowed. A network socket that errors, a stream that fails — if you have not explicitly decided how to handle it, the safest behavior is a loud crash rather than a silent, corrupted state.

### Handling it correctly

Always register an `'error'` listener on any emitter that might emit one (sockets, streams, child processes, your own emitters that can fail):

```js
const emitter = new EventEmitter();

emitter.on('error', (err) => {
  // Decide how to handle/log/recover.
  console.error('Handled error:', err.message);
});

emitter.emit('error', new Error('Boom!'));
// -> "Handled error: Boom!"   (no crash; the listener consumed it)
```

> BEST PRACTICE: For **every** stream, socket, or long-lived emitter you create or receive, attach an `'error'` handler *before* you start using it. A missing `'error'` listener is the most common cause of mysterious production crashes in Node.

### The `errorMonitor` symbol

Sometimes you want to **observe** errors (for logging/metrics) **without** counting as the handler that prevents the throw. The special `events.errorMonitor` symbol does exactly this: it lets you monitor errors while still requiring a *real* `'error'` listener.

```js
const EventEmitter = require('node:events');
const { errorMonitor } = require('node:events');

const emitter = new EventEmitter();

// Observe-only: this runs but does NOT prevent the throw.
emitter.on(errorMonitor, (err) => {
  console.log('[monitor] saw error:', err.message);
});

// Still no normal 'error' listener -> Node will throw after the monitor runs.
emitter.emit('error', new Error('Still fatal'));
// [monitor] saw error: Still fatal
// ...then throws and crashes (because no regular 'error' listener exists)
```

> TIP: Use `errorMonitor` for centralized error metrics/logging while keeping the strict "must handle or crash" contract intact. It is the right tool when a framework wants to *see* all errors but not *swallow* them.

### `'error'` and `process`-level safety nets

As an absolute last resort, `process.on('uncaughtException', ...)` catches errors that nothing else handled. This is a safety net for *logging and graceful shutdown*, **not** a way to keep running normally — after an uncaught exception the process is in an undefined state and you should exit.

```js
process.on('uncaughtException', (err) => {
  console.error('Last-resort handler:', err);
  // Log, flush, then EXIT. Do not pretend nothing happened.
  process.exit(1);
});
```

---

## 6. `MaxListenersExceededWarning` and Memory Leaks

### What the warning is

When you register **more than `maxListeners` (default 10)** listeners for the **same event name** on the **same emitter**, Node prints a warning to stderr:

```text
(node:12345) MaxListenersExceededWarning: Possible EventEmitter memory leak detected.
11 data listeners added to [EventEmitter]. Use emitter.setMaxListeners() to increase limit
```

This is **not an error** and does not stop your program. It is a heuristic alarm: "you have a suspiciously large number of listeners — did you forget to remove old ones?"

### Why it usually means a leak

The classic leak pattern: you add a listener inside a function that runs repeatedly (per request, per connection, per loop iteration) but never remove it. Each iteration adds another listener; they accumulate forever, holding references that the garbage collector cannot free.

```js
const EventEmitter = require('node:events');
const bus = new EventEmitter();

// BUG: every call adds another permanent listener to the shared bus.
function handleRequest(req) {
  bus.on('shutdown', () => cleanup(req)); // never removed!
}

// After 11 requests -> MaxListenersExceededWarning, and growing memory.
for (let i = 0; i < 50; i++) handleRequest({ id: i });
```

### How to fix it (in order of preference)

1. **Remove listeners when done.** Use `once` for one-shot listeners, or `off` to remove when the work completes.

   ```js
   function handleRequest(req) {
     const onShutdown = () => cleanup(req);
     bus.on('shutdown', onShutdown);
     req.on('end', () => bus.off('shutdown', onShutdown)); // cleanup!
   }
   ```

2. **Use an `AbortController`** to remove related listeners as a group (see §9.3).

3. **Don't share one emitter for per-entity logic** when each entity should own its own emitter. Sometimes the right fix is one emitter per connection, not one global emitter with N listeners.

4. **Raise the limit *only* when the high count is intentional.**

   ```js
   bus.setMaxListeners(100); // legitimate only if you truly need 100 listeners
   ```

> WARNING — Pitfall: The warning counts listeners **per event name on one emitter**, not total across the process. Ten `'data'` listeners is fine; eleven triggers it. Five `'data'` + five `'end'` on the same emitter does *not* trigger it.

> BEST PRACTICE: Treat `MaxListenersExceededWarning` as a bug until proven otherwise. Reach for `setMaxListeners` last, after confirming the high count is intentional and bounded.

---

## 7. Extending `EventEmitter` in Your Own Classes

The real power of `EventEmitter` is making your *own* objects emit events. You do this by subclassing it. This is how you build components that decouple "something happened" from "who cares."

### Class syntax (recommended)

```js
const EventEmitter = require('node:events');

// A job runner that announces its lifecycle via events.
class Job extends EventEmitter {
  constructor(name) {
    super();          // MUST call super() to initialize EventEmitter internals
    this.name = name;
  }

  async run() {
    this.emit('start', this.name);
    try {
      const result = await this.#doWork();
      this.emit('progress', 100);
      this.emit('done', result);   // success notification
      return result;
    } catch (err) {
      this.emit('error', err);     // emit 'error' on failure (see §5)
      throw err;
    }
  }

  #doWork() {
    return new Promise((resolve) =>
      setTimeout(() => resolve(`${this.name} complete`), 50)
    );
  }
}

// Consumers subscribe without the Job knowing who they are = decoupling.
const job = new Job('build');
job.on('start', (name) => console.log(`Started: ${name}`));
job.on('done', (res) => console.log(`Result: ${res}`));
job.on('error', (err) => console.error('Job failed:', err.message));

job.run();
// Started: build
// Result: build complete
```

> WARNING — Pitfall: If you forget `super()` in the constructor, the `EventEmitter` internals are never set up, and the first `this.on(...)` or `this.emit(...)` will throw. Always call `super()` first.

### When emitting `'error'` from your own class

If your class can fail, decide deliberately: do you emit `'error'` (and require consumers to handle it, crashing if they don't), or do you return a rejected promise? Streaming/long-lived components conventionally emit `'error'`; one-shot operations are often better as promises (see §10).

### Older prototype/`util.inherits` style (legacy — recognize it)

Before ES2015 classes, inheritance was wired up with `util.inherits`. You will still see it in older codebases:

```js
const EventEmitter = require('node:events');
const util = require('node:util');

function Job(name) {
  EventEmitter.call(this); // initialize EventEmitter on `this`
  this.name = name;
}
util.inherits(Job, EventEmitter); // set up the prototype chain

Job.prototype.run = function () {
  this.emit('start', this.name);
};
```

> BEST PRACTICE: For new code, always use `class extends EventEmitter`. `util.inherits` is legacy and clumsier; know it only to read old code.

---

## 8. Composition vs. Inheritance

You do not *have* to subclass `EventEmitter`. Sometimes it is cleaner to **own** an emitter as a private field and expose only the subscribe methods you want. This avoids leaking the full emitter API (like `emit`) to consumers, who should not be allowed to fire your internal events.

```js
const EventEmitter = require('node:events');

class Thermostat {
  #events = new EventEmitter(); // private, encapsulated emitter
  #temp = 20;

  // Expose a narrow, intentional subscription API.
  onChange(listener) {
    this.#events.on('change', listener);
    return this; // chainable
  }

  set(temp) {
    const previous = this.#temp;
    this.#temp = temp;
    // Only the Thermostat can emit — consumers cannot.
    this.#events.emit('change', { previous, current: temp });
  }
}

const t = new Thermostat();
t.onChange(({ previous, current }) =>
  console.log(`Temp changed ${previous} -> ${current}`)
);
t.set(23); // -> "Temp changed 20 -> 23"
```

| Approach | Pros | Cons |
| --- | --- | --- |
| **`extends EventEmitter`** | Familiar; consumers use standard `on`/`once`/`off`; integrates with `events.once`/`on` helpers | Exposes the entire emitter API (`emit`, `removeAllListeners`) to consumers |
| **Compose (private emitter)** | Strong encapsulation; expose only intended API; can't be misused | More boilerplate; consumers don't get standard `EventEmitter` introspection unless you forward it |

> BEST PRACTICE: Use **inheritance** for stream-like or framework components meant to be used with the full emitter idiom. Use **composition** when you want a tight, intentional public API and to prevent outsiders from emitting your events.

---

## 9. Asynchronous Listeners and the Async Helpers

### 9.1 Listeners fire synchronously — but can do async work

`emit()` calls listeners **synchronously and in order**. The emitter does *not* await your async listeners or care about their return values.

```js
const EventEmitter = require('node:events');
const emitter = new EventEmitter();

emitter.on('go', async () => {
  console.log('A: before await');
  await new Promise((r) => setTimeout(r, 10));
  console.log('C: after await');
});
emitter.on('go', () => console.log('B: second listener'));

emitter.emit('go');
console.log('D: after emit returns');
// A: before await
// B: second listener     <-- runs before the first listener's await resolves
// D: after emit returns
// C: after await         <-- resolves later, on a future tick
```

Two crucial consequences:

1. **`emit` does not wait** for async listeners. Code after `emit()` runs as soon as all listeners have *started* (returned their — possibly pending — promise).
2. **Rejections in async listeners are unhandled.** If an `async` listener throws/rejects, the emitter does not catch it — it becomes an **unhandled promise rejection**.

```js
emitter.on('risky', async () => {
  throw new Error('async boom'); // becomes an UnhandledPromiseRejection!
});
emitter.emit('risky');
```

> WARNING — Pitfall: Never let an async listener reject silently. Either wrap its body in `try/catch` and emit `'error'` yourself, or use the `captureRejections` option below.

### 9.2 `captureRejections` — routing async-listener rejections to `'error'`

Modern Node can automatically funnel rejected promises from async listeners into the `'error'` event. Enable it per-emitter or globally.

```js
const EventEmitter = require('node:events');

// Per-emitter:
const emitter = new EventEmitter({ captureRejections: true });

emitter.on('task', async () => {
  throw new Error('it failed');
});
emitter.on('error', (err) => {
  console.log('Captured rejection as error:', err.message);
});

emitter.emit('task');
// -> "Captured rejection as error: it failed"

// Global default for all emitters:
EventEmitter.captureRejections = true;
```

> BEST PRACTICE: For emitters whose listeners are commonly `async`, enable `captureRejections` and always provide an `'error'` listener. This turns silent unhandled rejections into a single, handleable error channel.

### 9.3 `events.once()` — await a single event as a promise

`events.once(emitter, eventName[, options])` returns a **promise** that resolves with an array of the emitted arguments the next time the event fires, or rejects if the emitter emits `'error'` first. This bridges the event world into `async/await`.

```js
const EventEmitter = require('node:events');
const { once } = require('node:events');

async function waitForReady() {
  const emitter = new EventEmitter();

  // Fire 'ready' shortly after.
  setTimeout(() => emitter.emit('ready', { port: 3000 }), 20);

  // Await it like a promise. Resolves with the emit arguments as an array.
  const [info] = await once(emitter, 'ready');
  console.log('Ready on port', info.port); // -> "Ready on port 3000"
}

waitForReady();
```

`once` automatically rejects on `'error'` (unless you are awaiting the `'error'` event itself):

```js
const { once } = require('node:events');
const EventEmitter = require('node:events');

const emitter = new EventEmitter();
setTimeout(() => emitter.emit('error', new Error('failed to start')), 20);

try {
  await once(emitter, 'ready'); // rejects because 'error' fired first
} catch (err) {
  console.error('Could not get ready:', err.message);
}
```

It also supports an `AbortSignal` for cancellation/timeouts:

```js
const { once } = require('node:events');
const EventEmitter = require('node:events');

const emitter = new EventEmitter();
const ac = new AbortController();
setTimeout(() => ac.abort(), 50); // give up after 50ms

try {
  await once(emitter, 'ready', { signal: ac.signal });
} catch (err) {
  console.error(err.name); // -> 'AbortError'
}
```

### 9.4 `events.on()` — consume a stream of events with `for await`

`events.on(emitter, eventName[, options])` returns an **async iterator** that yields once per emit, letting you process a *stream* of events with a `for await...of` loop. This is the cleanest way to consume repeated events sequentially.

```js
const EventEmitter = require('node:events');
const { on } = require('node:events');

async function consume() {
  const emitter = new EventEmitter();

  // Produce a few events, then signal completion via an AbortController.
  const ac = new AbortController();
  let n = 0;
  const timer = setInterval(() => {
    emitter.emit('tick', ++n);
    if (n === 3) {
      clearInterval(timer);
      ac.abort(); // ends the for-await loop
    }
  }, 10);

  try {
    // Each iteration yields the emit args as an array.
    for await (const [value] of on(emitter, 'tick', { signal: ac.signal })) {
      console.log('Tick:', value);
    }
  } catch (err) {
    if (err.name !== 'AbortError') throw err; // AbortError is the clean exit
  }
  console.log('Done consuming.');
}

consume();
// Tick: 1
// Tick: 2
// Tick: 3
// Done consuming.
```

> WARNING — Pitfall: `events.on()` **buffers** events that arrive faster than your loop body consumes them. If the producer is much faster than the consumer, the internal buffer grows unbounded — effectively a memory leak. Use it when the consumer can keep up, or apply backpressure / the `highWaterMark` option (Node 22+) to cap the buffer.

| Helper | Use when | Returns | Ends when |
| --- | --- | --- | --- |
| `events.once(emitter, name)` | You need **one** future occurrence | A promise of `[...args]` | Event fires once, or `'error'` rejects it |
| `events.on(emitter, name)` | You need to consume **many** occurrences | An async iterator | The `AbortSignal` aborts, or `'error'` throws into the loop |

---

## 10. Events vs. Callbacks vs. Promises

Three async patterns coexist in Node. Choosing the right one is a frequent interview topic and a real design skill.

### Quick comparison

| Pattern | Cardinality | Shape | Best for | Error handling |
| --- | --- | --- | --- | --- |
| **Callback** (error-first) | One result | `fn(args, (err, data) => {})` | A single async operation with one outcome | First `err` argument |
| **Promise / async-await** | One result | `await fn(args)` | A single async operation; composable, awaitable | `try/catch` / `.catch()` |
| **Event** (EventEmitter) | **Many** over time | `emitter.on('x', handler)` | Repeated/streaming occurrences; multiple subscribers | A dedicated `'error'` event |

### The decisive question: how many times will it happen?

- **Exactly once?** Use a **promise** (modern default) or an error-first **callback** (older APIs). Example: "read this file," "make this HTTP request."
- **Many times, or unknown count over time?** Use **events**. Example: "every chunk of incoming data," "every new client connection," "every progress update."
- **Many subscribers need to know?** Use **events** — multiple listeners can react to one emit, which a single callback/promise cannot express.

```js
// ONE outcome -> promise:
const data = await fs.promises.readFile('config.json', 'utf8');

// MANY outcomes over time -> events:
const stream = fs.createReadStream('huge.log');
stream.on('data', (chunk) => process(chunk)); // fires repeatedly
stream.on('end', () => console.log('done'));
stream.on('error', (err) => console.error(err));
```

### Why not just use one everywhere?

- A **promise resolves once** — it physically cannot represent a stream of values. Forcing a stream into a promise loses data or requires awkward buffering.
- **Events have no built-in "done"** signal and don't compose with `await` directly — which is exactly why `events.once`/`events.on` exist, to bridge the two worlds when you need to.
- **Callbacks** are still pervasive in older core APIs and some libraries; `util.promisify` converts an error-first callback API into a promise-returning one.

```js
const util = require('node:util');
const fs = require('node:fs');

// Convert a callback API to a promise API.
const readFile = util.promisify(fs.readFile);
const text = await readFile('notes.txt', 'utf8');
```

> BEST PRACTICE: Use promises for one-shot operations, events for streams/multi-subscriber notifications, and the `events.once`/`events.on` bridges to move between them. Reach for raw callbacks mainly when interfacing with older APIs.

---

## 11. Real-World Patterns

### 11.1 Publish/Subscribe (an in-process event bus)

A **pub/sub** (publish/subscribe) bus lets independent parts of an app communicate without referencing each other. Publishers emit named events; subscribers react. Neither side knows the other exists — they only share the event name contract.

```js
// event-bus.js — a shared, app-wide bus (a singleton module).
const EventEmitter = require('node:events');

class EventBus extends EventEmitter {}
module.exports = new EventBus(); // export a single shared instance
```

```js
// user-service.js — PUBLISHER
const bus = require('./event-bus');

function registerUser(user) {
  // ... save to DB ...
  bus.emit('user:registered', user); // announce; doesn't know who listens
}
module.exports = { registerUser };
```

```js
// email-service.js — SUBSCRIBER
const bus = require('./event-bus');

bus.on('user:registered', (user) => {
  console.log(`Sending welcome email to ${user.email}`);
});
```

```js
// analytics-service.js — ANOTHER independent SUBSCRIBER
const bus = require('./event-bus');

bus.on('user:registered', (user) => {
  console.log(`Tracking signup for ${user.id}`);
});
```

The user service has **zero knowledge** of email or analytics. You can add, remove, or change subscribers without touching the publisher — that is **decoupling**.

> WARNING — Pitfall: A global event bus is powerful but can become a "spooky action at a distance" maze if overused. Keep event names namespaced (`'user:registered'`), document the payload shape, and resist routing *everything* through the bus. For cross-process/distributed pub/sub, use a message broker (Redis Pub/Sub, RabbitMQ, Kafka) — `EventEmitter` is **in-process only**.

### 11.2 Decoupling modules and inverting dependencies

Events let a low-level module notify higher-level modules without depending on them. A database layer can emit `'slow-query'`; a monitoring module subscribes. The DB layer never imports the monitor — the arrow of dependency is inverted.

```js
// db.js
const EventEmitter = require('node:events');
class Database extends EventEmitter {
  async query(sql) {
    const start = Date.now();
    const rows = await this.#execute(sql);
    const ms = Date.now() - start;
    if (ms > 100) this.emit('slow-query', { sql, ms }); // notify, don't depend
    return rows;
  }
  #execute(sql) { /* ... */ return Promise.resolve([]); }
}
module.exports = new Database();
```

```js
// monitoring.js — subscribes; db.js has no idea this file exists
const db = require('./db');
db.on('slow-query', ({ sql, ms }) =>
  console.warn(`SLOW (${ms}ms): ${sql}`)
);
```

### 11.3 Lifecycle/observable objects

Long-lived objects (connections, jobs, file watchers) naturally model their lifecycle as events: `'connect'`, `'ready'`, `'data'`, `'drain'`, `'close'`. Consumers subscribe to the phases they care about. This is exactly how Node's own streams and sockets are designed — so your custom components feel idiomatic when they follow the same pattern.

---

## Common Pitfalls & Gotchas

> WARNING — Pitfall: **Forgetting an `'error'` listener.** An emitted `'error'` with no listener throws and crashes the process. Attach an `'error'` handler to every stream/socket/emitter that can fail.

> WARNING — Pitfall: **Trying to remove an anonymous/arrow listener.** `off`/`removeListener` require the *exact same function reference*. Inline functions can never be removed — store a reference if you'll need to remove it.

> WARNING — Pitfall: **Adding listeners in a loop/per-request without removing them.** This triggers `MaxListenersExceededWarning` and leaks memory. Use `once`, `off`, or an `AbortController` signal to clean up.

> WARNING — Pitfall: **Assuming `emit` awaits async listeners.** It does not — `emit` is synchronous and ignores returned promises. Code after `emit` runs immediately; async listener rejections become unhandled unless you use `captureRejections` or `try/catch`.

> WARNING — Pitfall: **Silencing `MaxListenersExceededWarning` with `setMaxListeners`** instead of fixing the leak. The warning is usually a real bug.

> WARNING — Pitfall: **Calling `removeAllListeners()` on a shared emitter** (like `process` or a framework's server) removes *other* code's listeners too. Remove only your own.

> WARNING — Pitfall: **Relying on `this` in an arrow-function listener.** Arrows capture lexical `this`, not the emitter. Use a regular function if you need `this === emitter`.

> WARNING — Pitfall: **Order assumptions.** Listeners run in registration order. If you depend on running first, use `prependListener` — don't rely on luck.

> WARNING — Pitfall: **Forgetting `super()`** when extending `EventEmitter`. The internals won't initialize and `emit`/`on` will throw.

> WARNING — Pitfall: **Unbounded buffering with `events.on()`.** A fast producer + slow consumer grows the internal buffer without limit. Cap it or ensure the consumer keeps up.

---

## Best Practices

> BEST PRACTICE: Attach an `'error'` listener to every fallible emitter *before* using it. Treat a missing one as a production-crash waiting to happen.

> BEST PRACTICE: Namespace event names (`'user:registered'`, `'order:shipped'`) and document each event's payload shape. Events are a contract; treat the name and payload as a stable API.

> BEST PRACTICE: Prefer `once` for one-shot lifecycle events so cleanup is automatic. Prefer `events.once()` when you want to `await` a single event.

> BEST PRACTICE: Always pair a listener registration with a removal strategy in long-lived processes — `off`, `once`, or an `AbortController` `signal`.

> BEST PRACTICE: Enable `{ captureRejections: true }` (or `EventEmitter.captureRejections = true`) on emitters with async listeners, and always provide an `'error'` handler to receive routed rejections.

> BEST PRACTICE: Use **composition** (a private emitter) when you want consumers to subscribe but never emit; use **inheritance** for stream-like components meant to be used with the full emitter idiom.

> BEST PRACTICE: Use `EventEmitter` for **in-process** decoupling only. For cross-process or distributed messaging, use a real broker (Redis/RabbitMQ/Kafka).

> BEST PRACTICE: Keep listeners small and fast. If a listener does heavy async work, hand off to a queue/promise and make sure errors are captured — don't let `emit` callers depend on listener completion.

---

## Key Interview Points

- **What is `EventEmitter` and why is it central to Node?**
  It is the built-in implementation of the observer/pub-sub pattern in the `events` module. Node's async, single-threaded I/O model is event-driven, and most core APIs (HTTP servers, streams, sockets, `process`) are `EventEmitter`s. Mastering it means understanding Node's core idiom.

- **`on` vs `once`?**
  `on` runs every time the event fires; `once` runs at most once and auto-removes itself afterward. Use `once` for one-shot lifecycle moments.

- **Is `emit` synchronous or asynchronous?**
  Synchronous. It calls all listeners in registration order in the same tick and returns `true`/`false` for whether any listener existed. It does **not** await async listeners.

- **What happens on an `'error'` event with no listener?**
  The emitter throws the error; if uncaught, it becomes an uncaught exception and crashes the process. Always register an `'error'` listener on fallible emitters. `events.errorMonitor` lets you observe errors without satisfying this requirement.

- **What causes `MaxListenersExceededWarning`?**
  Registering more than `maxListeners` (default 10) listeners for the same event on one emitter — typically because listeners are added repeatedly and never removed. Fix the leak (remove listeners) rather than just raising the limit.

- **Why can't you remove an arrow/anonymous listener?**
  `removeListener`/`off` matches by exact function reference; an inline function has no stored reference, so there's nothing to match. Store named references for removable listeners.

- **`removeListener` vs `off` vs `removeAllListeners`?**
  `off` is an alias of `removeListener` (removes one specific listener by reference). `removeAllListeners([name])` removes all listeners for an event, or every listener if called with no argument.

- **How do you `await` an event?**
  `events.once(emitter, name)` returns a promise resolving with the emit arguments (rejecting on `'error'`). `events.on(emitter, name)` returns an async iterator for consuming many events with `for await`.

- **Events vs callbacks vs promises — when to use which?**
  Promises/callbacks for a **single** outcome; events for **repeated/streaming** occurrences or **multiple subscribers**. A promise resolves only once and can't model a stream.

- **How do you build your own emitter?**
  `class MyThing extends EventEmitter`, call `super()` in the constructor, then `this.emit(...)`. Or compose a private `EventEmitter` field for stronger encapsulation.

- **What is `captureRejections`?**
  An option that routes rejected promises from async listeners into the `'error'` event instead of leaving them as unhandled rejections.

- **What's the difference between `listeners()` and `rawListeners()`?**
  `listeners()` returns the user functions; `rawListeners()` returns the internal wrappers (for `once`, the wrapper with a `.listener` back-reference), useful for advanced introspection.

---

## Practice Exercises

Try each before reading the solution. They build from basic to advanced.

1. **Basic emitter.** Create an `EventEmitter`, register a listener for a `'greeting'` event that prints a name passed as an argument, and emit it twice with different names.

2. **Once + count.** Create an emitter with a `'connect'` listener registered via `once`. Emit `'connect'` twice. Before and after each emit, print `listenerCount('connect')` to demonstrate the auto-removal.

3. **Safe error handling.** Build a `Downloader` class extending `EventEmitter` with a `start(url)` method that, after a short delay, emits `'done'` for valid URLs and `'error'` for URLs containing the word `"bad"`. Wire up listeners so a bad URL is handled gracefully (no crash).

4. **Fix the leak.** The code below logs a `MaxListenersExceededWarning`. Identify why and fix it so each request's listener is cleaned up.

   ```js
   const EventEmitter = require('node:events');
   const bus = new EventEmitter();
   function handle(id) {
     bus.on('shutdown', () => console.log(`cleanup ${id}`));
   }
   for (let i = 0; i < 15; i++) handle(i);
   ```

5. **Await an event.** Using `events.once`, write an async function `waitForLogin(emitter)` that resolves with the user object when an `'login'` event fires, but rejects (with a clear message) if it doesn't happen within 100ms. Use an `AbortController`.

6. **Async iterator pub/sub.** Build a small `MessageQueue` (extending `EventEmitter`) with a `publish(msg)` method emitting `'message'`. Using `events.on`, consume exactly 3 messages with a `for await` loop, then stop cleanly via an `AbortController`. Publish messages on an interval.

### Solutions

#### Solution 1 — Basic emitter

```js
const EventEmitter = require('node:events');
const emitter = new EventEmitter();

// Listener receives whatever args emit() passes.
emitter.on('greeting', (name) => {
  console.log(`Hello, ${name}!`);
});

emitter.emit('greeting', 'Ada');   // -> "Hello, Ada!"
emitter.emit('greeting', 'Linus'); // -> "Hello, Linus!"
```

#### Solution 2 — Once + count

```js
const EventEmitter = require('node:events');
const emitter = new EventEmitter();

emitter.once('connect', () => console.log('connected!'));

console.log(emitter.listenerCount('connect')); // -> 1 (registered, not yet fired)
emitter.emit('connect');                        // -> "connected!"
console.log(emitter.listenerCount('connect')); // -> 0 (auto-removed by `once`)
emitter.emit('connect');                        // -> (nothing; no listeners left)
console.log(emitter.listenerCount('connect')); // -> 0
```

#### Solution 3 — Safe error handling

```js
const EventEmitter = require('node:events');

class Downloader extends EventEmitter {
  start(url) {
    setTimeout(() => {
      if (url.includes('bad')) {
        // Emit 'error' on failure. With a listener attached, this is safe.
        this.emit('error', new Error(`Failed to download: ${url}`));
      } else {
        this.emit('done', { url, bytes: 1024 });
      }
    }, 20);
    return this; // allow chaining
  }
}

const dl = new Downloader();

dl.on('done', ({ url, bytes }) => console.log(`Downloaded ${url} (${bytes}B)`));
// Without this listener, the 'error' emit below would CRASH the process.
dl.on('error', (err) => console.error('Handled:', err.message));

dl.start('http://example.com/file.zip'); // -> "Downloaded ... (1024B)"
dl.start('http://example.com/bad.zip');   // -> "Handled: Failed to download: ..."
```

#### Solution 4 — Fix the leak

The bug: every `handle(id)` call adds a **new, permanent** `'shutdown'` listener to the shared `bus`, and none are ever removed. After 11 they exceed the default `maxListeners` and the warning fires; they also leak.

Two clean fixes:

```js
// Fix A: store the reference and remove it once shutdown has happened.
const EventEmitter = require('node:events');
const bus = new EventEmitter();

function handle(id) {
  // `once` auto-removes after the first 'shutdown', so listeners don't pile up
  // across the process lifetime if shutdown is a one-time event.
  bus.once('shutdown', () => console.log(`cleanup ${id}`));
}
for (let i = 0; i < 15; i++) handle(i);
```

But `once` still registers 15 listeners *before* shutdown, so it only helps if you also need them gone after firing. If the real issue is that these listeners should be tied to a request's lifetime, remove them when the request ends:

```js
// Fix B: tie each listener to its own lifecycle and remove it when done.
const EventEmitter = require('node:events');
const bus = new EventEmitter();

function handle(id) {
  const onShutdown = () => console.log(`cleanup ${id}`);
  bus.on('shutdown', onShutdown);

  // Simulate the request finishing -> remove its listener.
  return () => bus.off('shutdown', onShutdown); // call to detach
}

const detachers = [];
for (let i = 0; i < 15; i++) detachers.push(handle(i));
// When a request completes, detach its listener so it can't accumulate:
detachers.forEach((detach) => detach());
console.log(bus.listenerCount('shutdown')); // -> 0, no warning, no leak
```

#### Solution 5 — Await an event with a timeout

```js
const EventEmitter = require('node:events');
const { once } = require('node:events');

async function waitForLogin(emitter) {
  const ac = new AbortController();
  const timer = setTimeout(() => ac.abort(), 100); // give up after 100ms

  try {
    const [user] = await once(emitter, 'login', { signal: ac.signal });
    return user;
  } catch (err) {
    if (err.name === 'AbortError') {
      throw new Error('Login timed out after 100ms');
    }
    throw err; // re-throw anything unexpected (e.g., an 'error' event)
  } finally {
    clearTimeout(timer); // always clean up the timer
  }
}

// --- Demo ---
const auth = new EventEmitter();
setTimeout(() => auth.emit('login', { id: 7, name: 'Grace' }), 30);

waitForLogin(auth)
  .then((user) => console.log('Logged in:', user.name)) // -> "Logged in: Grace"
  .catch((err) => console.error(err.message));

// Try changing the emit delay to 200ms to see the timeout path fire.
```

#### Solution 6 — Async iterator pub/sub

```js
const EventEmitter = require('node:events');
const { on } = require('node:events');

class MessageQueue extends EventEmitter {
  publish(msg) {
    this.emit('message', msg); // broadcast to async iterator consumers
  }
}

async function main() {
  const mq = new MessageQueue();
  const ac = new AbortController();

  // Producer: publish a message every 10ms.
  let n = 0;
  const timer = setInterval(() => mq.publish(`msg-${++n}`), 10);

  let received = 0;
  try {
    // Consume the stream of 'message' events one at a time.
    for await (const [msg] of on(mq, 'message', { signal: ac.signal })) {
      console.log('Received:', msg);
      if (++received === 3) {
        clearInterval(timer);
        ac.abort(); // stop the for-await loop cleanly
      }
    }
  } catch (err) {
    if (err.name !== 'AbortError') throw err; // AbortError = our clean exit
  }

  console.log(`Consumed ${received} messages, queue drained.`);
}

main();
// Received: msg-1
// Received: msg-2
// Received: msg-3
// Consumed 3 messages, queue drained.
```

---

## Summary / Cheat Sheet

| Method / Concept | Purpose | Key note |
| --- | --- | --- |
| `on(name, fn)` / `addListener` | Register a recurring listener | Fires every emit, in registration order |
| `once(name, fn)` | One-time listener | Auto-removes after firing |
| `emit(name, ...args)` | Fire an event synchronously | Returns `true` if any listener; doesn't await async listeners |
| `prependListener` / `prependOnceListener` | Register at the **front** | Runs before earlier-registered listeners |
| `off(name, fn)` / `removeListener` | Remove a specific listener | Must pass the exact same function reference |
| `removeAllListeners([name])` | Bulk remove | No arg = remove everything (dangerous on shared emitters) |
| `listenerCount(name[, fn])` | How many listeners | Per event name |
| `eventNames()` | All events with listeners | Includes symbols |
| `listeners(name)` / `rawListeners(name)` | Inspect listeners | `raw` exposes `once` wrappers (`.listener`) |
| `setMaxListeners(n)` / `getMaxListeners()` | Per-emitter leak threshold | Default 10; `0` = unlimited |
| `EventEmitter.defaultMaxListeners` | Global threshold | Affects emitters that didn't set their own |
| `'error'` event | Special: throws if unhandled | Always attach a listener to fallible emitters |
| `events.errorMonitor` | Observe errors without handling | Doesn't prevent the throw |
| `{ captureRejections: true }` | Route async-listener rejections to `'error'` | Per-emitter or global |
| `events.once(emitter, name)` | `await` a single event | Resolves `[...args]`; rejects on `'error'` |
| `events.on(emitter, name)` | `for await` over many events | Buffers; cap it / keep consumer fast |
| `class X extends EventEmitter` | Make your own emitter | Must call `super()` |

**Decision rule:** one outcome -> **promise/callback**; many outcomes or many subscribers -> **events**; bridge them with **`events.once`** (single) and **`events.on`** (stream).

**Three rules to never forget:**
1. Always handle `'error'` — an unhandled one crashes the process.
2. Always have a removal strategy — leaked listeners trigger `MaxListenersExceededWarning` and grow memory.
3. `emit` is synchronous and ignores async listener results — use `captureRejections` or `try/catch` for async safety.
