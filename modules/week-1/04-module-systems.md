# Node.js Module Systems (CommonJS & ES Modules)

You already know JavaScript. But in the browser, you were used to dropping a `<script>` tag on a page and having every variable live in one shared global soup (or, more recently, using a bundler that hid the messy details from you). On the server, Node.js gives you something far more disciplined: a real **module system** where every file is its own isolated unit of code with explicit imports and exports.

In fact, Node.js gives you *two* module systems — the original **CommonJS** (CJS) and the standardized **ECMAScript Modules** (ESM). Understanding both, how they differ, how they interoperate, and when to choose each, is foundational backend knowledge. It's also one of the most common sources of confusion (and bugs) for developers new to Node, and a frequent interview topic. This module makes you fluent in all of it.

## What You'll Learn

- What modules are, why modular code matters, and the problem modules solve on the server
- CommonJS in depth: `require()`, `module.exports` vs `exports`, and the hidden **module wrapper function**
- The full **require resolution algorithm** — how Node finds a module (core modules, files, `node_modules`, folders, `index.js`)
- **Module caching** and how it gives you the singleton pattern for free
- **Circular dependencies**: why they happen, what Node returns, and how to handle them safely
- ES Modules: `import`/`export`, **default vs named** exports, and **dynamic `import()`**
- How to **enable ESM**: `.mjs` files, `"type": "module"` in `package.json`, and **top-level `await`**
- Recreating `__dirname` / `__filename` in ESM with `import.meta.url`, and the **CJS ↔ ESM interop** gotchas
- The `package.json` fields that govern module behavior: `main`, `type`, `exports`, and `imports`
- A practical framework for **choosing CommonJS vs ESM** in real projects

---

## 1. What Are Modules and Why Modular Code Matters

A **module** is a self-contained unit of code — typically a single file — that encapsulates related functionality and explicitly declares what it shares with the outside world (its **exports**) and what it needs from elsewhere (its **imports/requires**).

### The problem modules solve

Imagine building a server without modules. Every function, every variable, every helper lives in one enormous file or in many files that all share the same global scope. This causes real, painful problems:

- **Name collisions.** Two files both define a `config` variable, and one silently clobbers the other.
- **No encapsulation.** Any code can touch any variable. There's no notion of "private" implementation detail.
- **Unclear dependencies.** You can't tell what a piece of code needs to run. Load order becomes a fragile, hand-managed nightmare.
- **No reusability.** Copy-pasting logic between projects instead of importing a package.
- **Untestable code.** You can't isolate a unit to test it if everything is entangled.

Modules fix every one of these:

| Benefit | What it means in practice |
| --- | --- |
| **Encapsulation** | Each file has its own scope. Variables are private unless explicitly exported. |
| **Explicit dependencies** | `require`/`import` statements document exactly what a file needs. |
| **Reusability** | A well-defined module can be reused across files and published as a package. |
| **Maintainability** | Small, focused files with single responsibilities are easier to reason about. |
| **Testability** | Isolated units can be imported and tested independently, with dependencies mocked. |
| **Namespacing** | No global pollution — each module controls its own surface area. |

### A quick mental model

Think of a module as a small factory. Raw materials come in through imports. Inside, there's machinery (private functions and state) the outside world never sees. Finished goods leave through a single, well-labeled shipping door (the exports). The rest of your application only deals in finished goods — it never reaches inside the factory.

> BEST PRACTICE: Aim for modules with a **single responsibility** and a **small, intentional public surface**. Export only what callers genuinely need. Everything else stays private, which keeps you free to refactor internals without breaking consumers.

---

## 2. CommonJS: `require()`, `module.exports` vs `exports`

CommonJS (CJS) is the **original** module system Node.js shipped with, back in 2009, long before JavaScript had a standardized module syntax. For years it was *the* way to write Node code, and it's still ubiquitous — the majority of packages on npm historically were CJS, and you will read and write CJS constantly. It is **synchronous** and built around two globals provided to every file: `require` and `module`.

### Exporting with `module.exports`

Every module has an object called `module`, and `module.exports` is the value that gets returned when another file `require`s it. Initially `module.exports` is an empty object `{}`.

```js
// math.js — a CommonJS module

// A private helper — NOT exported, invisible to the outside world.
function assertNumber(n) {
  if (typeof n !== 'number') throw new TypeError('Expected a number');
}

function add(a, b) {
  assertNumber(a);
  assertNumber(b);
  return a + b;
}

function subtract(a, b) {
  return a - b;
}

// Attach the public API to module.exports.
module.exports = {
  add,
  subtract,
};
```

```js
// app.js — consuming the module

// require() returns whatever math.js assigned to module.exports.
const math = require('./math'); // ".js" extension is optional for files

console.log(math.add(2, 3));      // 5
console.log(math.subtract(7, 4)); // 3

// assertNumber is private — this is undefined:
console.log(math.assertNumber);   // undefined
```

You can also export a single value — a function, a class, a string, anything:

```js
// logger.js — exporting one function as the whole module
module.exports = function log(message) {
  console.log(`[${new Date().toISOString()}] ${message}`);
};
```

```js
// app.js
const log = require('./logger'); // the export IS the function
log('Server started'); // [2026-06-19T...] Server started
```

```js
// User.js — exporting a class
class User {
  constructor(name) {
    this.name = name;
  }
  greet() {
    return `Hi, I'm ${this.name}`;
  }
}
module.exports = User;
```

```js
// app.js
const User = require('./User');
const u = new User('Ada');
console.log(u.greet()); // Hi, I'm Ada
```

### `module.exports` vs `exports` — the classic gotcha

Node gives you a convenience shorthand: a local variable named `exports` that **initially points to the same object** as `module.exports`. So at the start of every module:

```js
exports === module.exports; // true — they reference the same object
```

This means you can *add properties* to `exports`, and because both names point to the same object, it works:

```js
// utils.js — using the exports shorthand correctly
exports.double = (n) => n * 2;       // adds to the shared object — works
exports.square = (n) => n * n;       // adds to the shared object — works
```

```js
const utils = require('./utils');
console.log(utils.double(5)); // 10
console.log(utils.square(4)); // 16
```

**But here's the trap.** If you *reassign* `exports` to a brand-new value, you break the link. `exports` now points somewhere else, but Node still returns `module.exports` — which you never changed.

```js
// broken.js
exports = function () {        // ❌ reassigns the LOCAL variable only
  return 'this will not work';
};
// module.exports is STILL the original empty {} — your function is lost.
```

```js
const broken = require('./broken');
console.log(broken); // {} — an empty object, NOT the function!
```

The fix is to assign to `module.exports` directly when you want to replace the entire export:

```js
// fixed.js
module.exports = function () { // ✅ replaces the actual returned value
  return 'this works';
};
```

Here's the relationship visualized:

```
                Before any assignment:

   module.exports ──┐
                    ├──►  { }   (the same empty object)
   exports ─────────┘


   After:  exports = somethingNew;   ❌

   module.exports ──►  { }          (unchanged — this is what require returns)
   exports ─────────►  somethingNew (orphaned local variable)


   After:  module.exports = somethingNew;   ✅

   module.exports ──►  somethingNew (this is what require returns)
   exports ─────────►  { }          (now stale, but you stopped using it)
```

> WARNING — Pitfall: Never write `exports = ...` (reassigning the shorthand). Either add properties with `exports.foo = ...`, or replace the whole export with `module.exports = ...`. Mixing the two — e.g. `module.exports = {}` and then later `exports.bar = ...` — is also a bug, because after the reassignment they no longer point to the same object.

> BEST PRACTICE: Pick one convention per file. For modules that export a single thing (a class, a factory, one function), use `module.exports = X`. For modules that export a grab-bag of named utilities, either build one object literal and assign it to `module.exports`, or consistently use `exports.name = ...`. Don't mix styles in the same file.

### The Module Wrapper Function

Here's something that surprises many newcomers: **your module code does not run at the top level**. Before executing a CommonJS file, Node wraps its entire contents in a function that looks like this:

```js
(function (exports, require, module, __filename, __dirname) {
  // ── your module's code lives here ──
});
```

Node compiles your file into this function and then **invokes** it, passing in:

| Parameter | What it is |
| --- | --- |
| `exports` | A reference to `module.exports` (the shorthand). |
| `require` | The function used to import other modules. |
| `module` | The object representing the current module (has `.exports`, `.id`, `.filename`, `.loaded`, `.children`, `.parent`). |
| `__filename` | The absolute path of the current module file. |
| `__dirname` | The absolute path of the directory containing the current module. |

This wrapper is *why* those five identifiers exist in every CommonJS file even though you never declared them — they're function parameters. It's also why each module gets its **own private scope**: anything you declare with `const`/`let`/`var` or `function` at the "top level" of your file is actually a local inside this wrapper function. It does **not** leak to the global scope.

```js
// scope-demo.js
var secret = 42; // local to the wrapper function, NOT global
console.log(__filename); // absolute path to scope-demo.js
console.log(__dirname);  // absolute path to its directory

// Prove it's wrapped — `arguments` reflects the wrapper's parameters:
console.log(typeof require, typeof module, typeof exports); // function object object
```

```js
// another.js
require('./scope-demo');
console.log(typeof secret); // "undefined" — secret never escaped its module
```

> TIP: To declare a *truly* global variable in Node (rarely a good idea), you must attach it to the `globalThis` (or `global`) object explicitly, e.g. `globalThis.myGlobal = 1`. The implicit-global behavior of browser scripts does not exist for module-scoped declarations.

---

## 3. The `require` Resolution Algorithm

When you call `require(X)`, how does Node figure out *which* file or package to load? This is the **module resolution algorithm**, and knowing it cold explains a huge class of "module not found" errors. Node resolves `X` in a defined order:

### Step 1 — Core (built-in) modules

If `X` is the name of a built-in module — `fs`, `path`, `http`, `crypto`, `os`, `events`, etc. — Node returns it immediately. Core modules always win and are never looked up on disk.

```js
const fs = require('fs');     // core module — resolved first, fastest
const path = require('path'); // core module
```

Modern Node also exposes core modules under a `node:` prefix, which is unambiguous and slightly faster to resolve (and impossible to shadow with a same-named npm package):

```js
const fs = require('node:fs');     // explicit core module — recommended
const path = require('node:path'); // never confused with a local "path" package
```

> BEST PRACTICE: Prefer the `node:` prefix for built-ins (`require('node:fs')`, `import ... from 'node:path'`). It signals intent clearly and protects you from a malicious or accidental `node_modules/fs` package shadowing the core module.

### Step 2 — Relative and absolute paths (`./`, `../`, `/`)

If `X` begins with `./`, `../`, or `/`, Node treats it as a **file path** relative to the requiring module (or absolute). It then tries, in order:

1. The exact path as a **file**: `X`
2. `X` with extensions appended: `X.js`, then `X.json`, then `X.node`
3. The path as a **directory** (see Step 4 below)

```js
require('./math');       // tries ./math, ./math.js, ./math.json, ./math.node
require('./config.json'); // loads & parses JSON, returns the parsed object
require('../lib/db');     // walk up one directory, then into lib/
require('/abs/path/mod'); // absolute path
```

JSON files are special-cased: `require('./data.json')` reads the file and returns the **parsed object** directly.

### Step 3 — Bare specifiers → `node_modules` lookup

If `X` is a "bare" specifier (no path prefix and not a core module), like `require('express')` or `require('lodash/fp')`, Node searches `node_modules` directories. It starts in the requiring file's directory and **walks up the directory tree**, checking `node_modules` at each level until it finds a match or reaches the filesystem root.

For a file at `/home/me/app/src/server.js`, requiring `'express'` checks:

```
/home/me/app/src/node_modules/express
/home/me/app/node_modules/express        ← typically found here
/home/me/node_modules/express
/home/node_modules/express
/node_modules/express
```

This upward walk is why a dependency installed at the project root is available to every file in the project, and why nested `node_modules` (hoisting, peer dep edge cases) sometimes matter.

### Step 4 — Folders as modules (the package/`index.js` rule)

When a resolved path is a **directory**, Node decides which file inside it to load:

1. If the directory contains a `package.json` with a `"main"` field, Node loads the file named by `"main"`. (And if the package defines an `"exports"` field, that takes precedence — see Section 9.)
2. Otherwise, Node looks for an **`index.js`** (then `index.json`, then `index.node`) in that directory.

```
my-package/
├── package.json   →  { "main": "lib/start.js" }
└── lib/
    └── start.js    ←  loaded by require('my-package') or require('./my-package')
```

```
utils/
└── index.js        ←  require('./utils') loads this (no package.json present)
```

> TIP: The `index.js` convention lets you turn a folder into a tidy module. Put the public API in `utils/index.js` and split internals into `utils/parse.js`, `utils/format.js`, etc. Callers just `require('./utils')`.

### The resolution order, summarized

| Specifier form | Example | Resolved as |
| --- | --- | --- |
| Core module name | `require('fs')` / `require('node:fs')` | Built-in, returned immediately |
| Starts with `./`, `../`, `/` | `require('./math')` | File path → file → extensions → directory |
| `.json` path | `require('./data.json')` | Parsed JSON object |
| Bare specifier | `require('express')` | Walk up `node_modules`, then folder rules |

> WARNING — Pitfall: A very common error is `Error: Cannot find module './utils'`. Causes include: a typo in the path, a wrong relative depth (`./` vs `../`), a case-mismatch (`./Utils` vs `./utils` — fine on macOS/Windows, fatal on Linux/production), or forgetting that bare specifiers (no `./`) are looked up in `node_modules`, not relative to your file.

---

## 4. Module Caching & the Singleton Pattern

The **first** time you `require` a module, Node executes its code and stores the resulting `module.exports` in a cache (`require.cache`), keyed by the resolved absolute file path. **Every subsequent `require` of that same module returns the cached export object — the code does not run again.**

This has a profound and very useful consequence: **a module is effectively a singleton.** State you create at module load time is shared by every part of your app that requires it.

```js
// counter.js — module-level state, loaded and executed exactly once
let count = 0;

module.exports = {
  increment() { return ++count; },
  current() { return count; },
};

// This line runs ONLY on first require (proof of single execution):
console.log('counter.js evaluated');
```

```js
// a.js
const counter = require('./counter');
counter.increment();
counter.increment();
module.exports = counter.current();
```

```js
// main.js
require('./a');                       // logs "counter.js evaluated" (first load)
const counter = require('./counter'); // returns the SAME cached object — no log
console.log(counter.current());       // 2 — state shared across files
```

Notice "counter.js evaluated" prints exactly **once**, and the count carries across files. This is the idiomatic way to share a single database connection, a config object, or a logger across an entire application.

```js
// db.js — a classic singleton via module caching
const { MongoClient } = require('mongodb');

const client = new MongoClient(process.env.MONGO_URI);
// Connect once; export the same connection promise everywhere.
const connection = client.connect();

module.exports = connection; // every requirer awaits the SAME connection
```

> BEST PRACTICE: Lean on module caching for app-wide singletons (config, DB pool, logger). Define the resource at module load time and export it. You don't need a fancy singleton class — the module system gives you one for free.

### Cache keys and gotchas

- The cache key is the **resolved absolute path**. The same file required via different *relative* paths still resolves to one cache entry — Node normalizes them. But a dependency installed in two different `node_modules` locations (e.g. due to version conflicts) is **two distinct files** and thus **two separate instances** — a subtle cause of "why is my singleton not shared?" bugs.
- You can inspect the cache via `require.cache` and (rarely, mostly in tests) delete entries with `delete require.cache[require.resolve('./mod')]` to force a re-evaluation. Avoid this in production code.
- `require.resolve('./mod')` returns the absolute path Node *would* load, without loading it — handy for diagnostics.

```js
console.log(Object.keys(require.cache)); // every loaded module's absolute path
console.log(require.resolve('./counter')); // the cache key for counter.js
```

> WARNING — Pitfall: Because modules are cached as singletons, **mutating** another module's exported object affects everyone. If `config.js` exports a settings object and one file does `config.debug = true`, every other consumer sees it. That can be intentional, but accidental mutation of shared state causes spooky action-at-a-distance bugs. Freeze configs you don't want mutated: `module.exports = Object.freeze({...})`.

---

## 5. Circular Dependencies

A **circular dependency** occurs when module A requires module B, and B (directly or transitively) requires A. This forms a cycle. Node does not crash on cycles, but the result can be surprising, so you must understand exactly what happens.

### What Node does

When module A starts loading and requires B, Node begins loading B. If B then requires A, Node sees that A is **already in the process of loading** (it's in the cache but not finished). Rather than loop forever, Node returns A's **partially completed `module.exports`** — whatever has been assigned to it *so far*.

```js
// a.js
console.log('a: start');
exports.loaded = false;            // assigned BEFORE requiring b
const b = require('./b');          // triggers loading of b
console.log('a: b.value =', b.value);
exports.loaded = true;             // assigned AFTER b finishes
console.log('a: done');
```

```js
// b.js
console.log('b: start');
const a = require('./a');          // a is mid-load → gets PARTIAL exports
console.log('b: a.loaded (partial) =', a.loaded); // false — not yet true!
exports.value = 'B value';
console.log('b: done');
```

```js
// main.js
require('./a');
```

Running `node main.js` prints:

```
a: start
b: start
b: a.loaded (partial) = false   ← b sees a's exports as they were so far
b: done
a: b.value = B value            ← by the time a continues, b is complete
a: done
```

The lesson: in a cycle, the module that gets imported *while still loading* hands back an **incomplete** exports object. Whatever wasn't assigned yet is `undefined` (or its earlier value).

### The most common real-world failure

The breakage usually shows up with the "replace the whole export" pattern:

```js
// a.js
const b = require('./b');
class A {}
module.exports = A;   // ← assigned LATE; b already grabbed the old {} 
```

```js
// b.js
const A = require('./a'); // gets the partial export ({}) — NOT class A
class B extends A {}      // 💥 TypeError: Class extends value #<Object> is not a constructor
module.exports = B;
```

Because `a.js` reassigns `module.exports = A` *after* `b.js` already captured a reference to the original empty object, `b.js` ends up with `{}`, not the class. ESM has its own version of this problem (you can hit a `ReferenceError: Cannot access 'X' before initialization` due to its live bindings and temporal dead zone).

### How to handle circular dependencies

1. **Eliminate the cycle (best).** Cycles are usually a design smell. Extract the shared piece into a third module both depend on.

   ```js
   // shared.js
   class Base {}
   module.exports = Base;
   ```
   ```js
   // a.js
   const Base = require('./shared');
   class A extends Base {}
   module.exports = A;
   ```
   ```js
   // b.js
   const Base = require('./shared');
   class B extends Base {}
   module.exports = B;
   ```

2. **Require lazily, at point of use.** Move the `require` inside the function that needs it, so it runs *after* both modules finish loading.

   ```js
   // a.js
   module.exports = function doWork() {
     const b = require('./b'); // resolved at call time — cycle fully loaded by now
     return b.help();
   };
   ```

3. **Export early, mutate later.** Assign `module.exports` (or its properties) *before* requiring the other module, so the partial export already has what the peer needs.

4. **Restructure dependencies** so they flow in one direction (a layered architecture: routes → controllers → services → models, never backward).

> BEST PRACTICE: Treat a circular dependency as a code-smell alarm, not a feature to exploit. The robust fix is almost always to extract shared code into a separate module so the dependency graph is acyclic. Reach for lazy `require` only as a tactical patch.

> TIP: Set `module.exports` (or attach the properties peers depend on) *before* any `require` of a module that depends back on you. "Export first, import second" minimizes the partial-export window.

---

## 6. ES Modules: `import` / `export`, Default vs Named, Dynamic `import()`

**ECMAScript Modules** (ESM) are the *official, language-standard* module system — the same `import`/`export` syntax used in modern browsers. Node has supported ESM natively for years, and it's the forward-looking default for new code. ESM differs from CommonJS in important ways: it's **statically analyzable**, supports **asynchronous** loading, uses **live bindings**, and runs in **strict mode** automatically.

### Named exports

You can export any number of named bindings:

```js
// math.mjs
export function add(a, b) {     // inline named export
  return a + b;
}

export const PI = 3.14159;       // inline named export

function subtract(a, b) {        // declared privately...
  return a - b;
}
export { subtract };             // ...then exported by name in a list
```

```js
// app.mjs
import { add, PI } from './math.mjs';        // import specific names
console.log(add(2, 3), PI);                  // 5 3.14159

import { subtract as minus } from './math.mjs'; // rename on import
console.log(minus(9, 4));                    // 5

import * as math from './math.mjs';          // namespace import — all named exports
console.log(math.add(1, 1), math.PI);        // 2 3.14159
```

> WARNING — Pitfall: In ESM, **relative import paths usually require the file extension** (`./math.mjs`, `./util.js`). Unlike CommonJS, Node's default ESM resolver does **not** auto-append `.js` or auto-resolve `index.js` for relative imports. Bare specifiers resolved through `node_modules` still use the package's configuration, but for your own files, write the extension. (Tooling/bundlers may relax this, but the Node runtime does not.)

### Default exports

A module can have **one** default export, useful when a module's primary purpose is a single thing:

```js
// User.mjs
export default class User {       // the default export
  constructor(name) { this.name = name; }
}

export const ROLE = 'member';     // you can ALSO have named exports alongside
```

```js
// app.mjs
import User from './User.mjs';            // no braces → the default export
import User, { ROLE } from './User.mjs';  // default + named in one line
const u = new User('Grace');
console.log(u.name, ROLE);                // Grace member
```

| Aspect | Named exports | Default export |
| --- | --- | --- |
| How many per module | Unlimited | Exactly one |
| Import syntax | `import { foo } from '...'` | `import anything from '...'` |
| Name fixed by module? | Yes (rename with `as`) | No — importer chooses any name |
| Tree-shaking friendliness | Excellent | Good, but coarser |
| Best for | Utility collections, multiple exports | A module's single primary export |

> BEST PRACTICE: Prefer **named exports** in most application code. They give consistent names across the codebase (better for grep, refactoring, and editor auto-import) and tree-shake more precisely. Reserve `default` for modules whose entire identity is one thing (a React component, a single class).

### Re-exporting (barrel files)

ESM has clean syntax for aggregating exports into one entry point — a "barrel":

```js
// index.mjs — re-export from submodules
export { add, subtract } from './math.mjs'; // re-export specific names
export { default as User } from './User.mjs'; // re-export a default as a name
export * from './validators.mjs';            // re-export all named exports
```

### Dynamic `import()`

Static `import` statements must sit at the top level and are resolved before the module runs. But sometimes you need to load a module **conditionally** or **lazily** (e.g. only load a heavy dependency when a feature is actually used). That's what the **dynamic `import()`** expression is for. It returns a **Promise** that resolves to the module's namespace object, and — crucially — it works in **both** CommonJS and ES Modules.

```js
// Lazy/conditional loading — works in .mjs AND in .cjs/.js (CommonJS)
async function maybeLoadHeavyThing(flag) {
  if (!flag) return null;
  const mod = await import('./heavy.mjs'); // returns a Promise → namespace object
  return mod.default ?? mod;               // default export lives on `.default`
}
```

```js
// Choosing a module at runtime
const lang = process.env.LANG_PACK || 'en';
const { default: translations } = await import(`./i18n/${lang}.mjs`);
console.log(translations.greeting);
```

This is also the **only** way to load an ES Module *from* a CommonJS file (see Section 8 on interop):

```js
// loader.cjs — a CommonJS file loading an ESM dependency
async function run() {
  const { default: chalk } = await import('chalk'); // ESM-only package
  console.log(chalk.green('Loaded an ESM package from CJS!'));
}
run();
```

> TIP: Use dynamic `import()` to defer the cost of loading rarely used or large modules until they're actually needed, and to bridge from CommonJS into ESM-only packages.

### A note on live bindings

Static `import` does not copy values — it creates a **live, read-only binding** to the exporter's variable. If the exporting module later changes an exported `let`, importers see the new value. (CommonJS `require` copies the reference at the moment of require; primitives captured are snapshots.)

```js
// counter.mjs
export let count = 0;
export function bump() { count++; } // mutates the exported binding
```

```js
// app.mjs
import { count, bump } from './counter.mjs';
console.log(count); // 0
bump();
console.log(count); // 1 — the imported binding is LIVE, reflects the change
// count = 5;       // ❌ TypeError: imports are read-only bindings
```

---

## 7. Enabling ESM: `.mjs`, `"type": "module"`, and Top-Level `await`

Node needs to know whether to parse a given file as CommonJS or as an ES Module. There are two mechanisms, and exactly one wins per file.

### Mechanism 1 — File extension

| Extension | Always treated as |
| --- | --- |
| `.mjs` | ES Module |
| `.cjs` | CommonJS |
| `.js` | Depends on the nearest `package.json` `"type"` (see below) |

`.mjs` and `.cjs` are unambiguous regardless of any `package.json` setting. Use them when you want to be explicit or mix systems in one project.

### Mechanism 2 — `"type"` in `package.json`

For `.js` files, Node looks up the nearest parent `package.json` and reads its `"type"` field:

- `"type": "module"` → `.js` files are treated as **ES Modules**.
- `"type": "commonjs"` (or the field is absent) → `.js` files are treated as **CommonJS** (this is the historical default).

```json
{
  "name": "my-app",
  "version": "1.0.0",
  "type": "module"
}
```

With `"type": "module"`, this `.js` file is ESM:

```js
// server.js (in a package with "type": "module")
import http from 'node:http';

const server = http.createServer((req, res) => res.end('Hello ESM'));
server.listen(3000, () => console.log('Listening on :3000'));
```

> TIP: Even in a `"type": "module"` project, you can still write a one-off CommonJS file by naming it `*.cjs`. And in a CommonJS project you can drop in an ESM file as `*.mjs`. The extension always overrides the `"type"` default for that specific file.

### Top-Level `await`

One of ESM's best features: you can use `await` **at the top level** of a module, outside any `async` function. This is impossible in CommonJS. It's perfect for setup that must complete before the module finishes loading — like connecting to a database or fetching configuration.

```js
// db.mjs — top-level await: the module isn't "done" until the DB connects
import { MongoClient } from 'mongodb';

const client = new MongoClient(process.env.MONGO_URI);
await client.connect();              // top-level await — no async wrapper needed
console.log('Connected to MongoDB');

export const db = client.db('app');  // exported only after connection succeeds
```

```js
// app.mjs
import { db } from './db.mjs'; // importing this awaits db.mjs's top-level await first
const users = await db.collection('users').find().toArray();
console.log(users);
```

> WARNING — Pitfall: Top-level `await` makes the importing graph wait for the awaited work. A slow or hanging top-level `await` (e.g. a network call that never resolves) will **block** any module that imports it from finishing initialization. Use it for genuinely required setup, and make sure those operations have timeouts/error handling.

---

## 8. `__dirname` / `__filename` in ESM, and CJS ↔ ESM Interop Gotchas

### `__dirname` and `__filename` don't exist in ESM

In CommonJS, `__dirname` and `__filename` are handed to you by the module wrapper. **In ES Modules they are not defined** — referencing them throws `ReferenceError`. Instead, ESM exposes `import.meta.url`, the file's URL (a `file://` string), from which you derive paths.

```js
// paths.mjs — recreating __dirname / __filename in ESM
import { fileURLToPath } from 'node:url';
import { dirname } from 'node:path';

const __filename = fileURLToPath(import.meta.url);  // absolute file path
const __dirname = dirname(__filename);              // its directory

console.log(__filename); // /abs/path/to/paths.mjs
console.log(__dirname);  // /abs/path/to
```

On modern Node (v20.11+ / v21.2+), there's a shortcut: `import.meta.dirname` and `import.meta.filename`:

```js
// Modern Node — no boilerplate needed
console.log(import.meta.dirname);  // /abs/path/to
console.log(import.meta.filename); // /abs/path/to/paths.mjs
```

> TIP: Use `import.meta.dirname` / `import.meta.filename` on Node 20.11+. They're the cleanest equivalents to CJS `__dirname` / `__filename`. For libraries that must support older runtimes, fall back to the `fileURLToPath(import.meta.url)` pattern.

### Importing CommonJS from ESM — works, with a catch

ESM can `import` a CommonJS module. Node treats the CJS module's `module.exports` as the **default export**. Named imports may also work for top-level statically-detectable names (Node uses a heuristic, "cjs-module-lexer"), but the reliable approach is to import the default and destructure:

```js
// CJS package (e.g. lodash, an older library) imported from ESM
import _ from 'lodash';              // CJS module.exports → default export
const { cloneDeep } = _;            // destructure what you need (always reliable)

// Named imports MAY work when Node can statically detect the names:
import { cloneDeep as cd } from 'lodash'; // sometimes works, sometimes not
```

> WARNING — Pitfall: Named imports from a CommonJS module are not guaranteed. They depend on Node's static analysis of the CJS file's exports and can fail for dynamically constructed exports. The bulletproof pattern is `import pkg from 'cjs-pkg'; const { thing } = pkg;`.

### Importing ESM from CommonJS — only via dynamic `import()`

You **cannot** use a static `require()` on a pure ES Module from CommonJS. (`require()` is synchronous; ESM loading is asynchronous, and historically the two were incompatible.) The portable way is dynamic `import()`:

```js
// app.cjs — load an ESM-only package from CommonJS
async function main() {
  const { default: chalk } = await import('chalk'); // ESM-only since v5
  console.log(chalk.blue('Hello from CJS calling ESM'));
}
main();
```

> TIP: Newer Node versions (v22+/v23+) have introduced **`require(esm)`** — synchronously requiring an ES Module — under certain conditions (the ESM graph must have no unsettled top-level `await`). It's increasingly available by default, but for code that must run on a range of versions, the dynamic `import()` approach remains the safe, universal bridge.

### Other ESM-vs-CJS differences to remember

| Feature | CommonJS | ES Modules |
| --- | --- | --- |
| Import syntax | `require()` (function, sync) | `import` (statement, async graph) / `import()` (dynamic) |
| Export syntax | `module.exports` / `exports.x` | `export` / `export default` |
| File extension | `.cjs` (or `.js` w/o `"type":"module"`) | `.mjs` (or `.js` w/ `"type":"module"`) |
| `__dirname` / `__filename` | Provided | Not available — use `import.meta.url` / `import.meta.dirname` |
| `require`, `module`, `exports` | Available | Not available |
| Top-level `await` | Not allowed | Allowed |
| Strict mode | Opt-in (`'use strict'`) | Always on |
| Loading | Synchronous | Asynchronous |
| Bindings | Value/reference copy at require time | Live, read-only bindings |
| `this` at top level | `module.exports` | `undefined` |
| Resolution of own files | Extension optional, `index.js` auto | Extension generally required, no auto-index |
| JSON import | `require('./x.json')` directly | `import data from './x.json' with { type: 'json' }` |
| Conditional/lazy load | `require()` inline | `await import()` |

> WARNING — Pitfall: At the top level of an ES Module, `this` is `undefined` (not `module.exports`, not the global object). Code copied from a CJS file that relied on top-level `this` will break under ESM.

---

## 9. `package.json` Fields That Affect Modules: `main`, `type`, `exports`, `imports`

The `package.json` file is the control panel for how Node treats a package's modules. Four fields matter here.

### `main` — the legacy entry point

`"main"` names the file loaded when someone does `require('your-package')` or `import 'your-package'` without a subpath. If absent, Node falls back to `index.js`.

```json
{
  "name": "my-lib",
  "main": "./lib/index.js"
}
```

### `type` — default module system for `.js`

As covered in Section 7: `"module"` makes `.js` files ESM; `"commonjs"` (or omitted) makes them CJS.

```json
{ "type": "module" }
```

### `exports` — the modern entry points (and encapsulation)

`"exports"` is the modern, more powerful replacement for `"main"`. It does three big things:

1. **Defines official entry points** (including subpaths) — anything not listed becomes **inaccessible** from outside the package. This *encapsulates* internals: consumers can't reach into `your-package/lib/secret.js`.
2. **Conditional exports** — serve different files to ESM (`import`) vs CommonJS (`require`) consumers (a "dual package"), or for `node` vs `browser`, `development` vs `production`, etc.
3. **Subpath exports** — expose curated entry points like `my-lib/utils`.

```json
{
  "name": "my-lib",
  "type": "module",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",   // used by `import 'my-lib'`
      "require": "./dist/index.cjs",  // used by `require('my-lib')`
      "types": "./dist/index.d.ts"    // (TS consumers; informational here)
    },
    "./utils": {                       // a public subpath: `import 'my-lib/utils'`
      "import": "./dist/utils.mjs",
      "require": "./dist/utils.cjs"
    },
    "./package.json": "./package.json" // explicitly allow reading package.json
  }
}
```

> WARNING — Pitfall: Once you add `"exports"`, it is **strict**. Subpaths not listed are blocked — `require('my-lib/dist/secret')` throws `ERR_PACKAGE_PATH_NOT_EXPORTED` even though the file exists on disk. This is intentional encapsulation, but it surprises people upgrading older packages. If you publish a package, list every entry point consumers legitimately need.

> WARNING — Pitfall: Conditional `"exports"` ordering matters — Node matches conditions **top to bottom**. Put more specific conditions (`"node"`, `"import"`) before fallbacks (`"default"`). Also beware the **dual-package hazard**: if both the ESM and CJS builds carry independent state, a consumer can accidentally load *both*, ending up with two copies of your singletons.

### `imports` — private internal aliases (subpath imports)

`"imports"` lets a package define **internal-only** import aliases that always begin with `#`. They're private to the package (not visible to consumers) and great for avoiding brittle `../../../` relative paths and for swapping implementations conditionally.

```json
{
  "name": "my-app",
  "type": "module",
  "imports": {
    "#config": "./src/config/index.js",
    "#db": {
      "node": "./src/db/node.js",
      "default": "./src/db/stub.js"
    }
  }
}
```

```js
// anywhere inside the package — no relative path gymnastics
import config from '#config';  // resolves to ./src/config/index.js
import db from '#db';          // resolves per condition (node → node.js)
```

> BEST PRACTICE: Use `"imports"` (`#`-prefixed subpath imports) for clean internal aliases instead of long relative chains. Use `"exports"` to define and lock down your package's public API. Together they give you a crisp public/private boundary.

| Field | Audience | Purpose |
| --- | --- | --- |
| `main` | External consumers | Legacy single entry point (fallback to `index.js`) |
| `type` | This package's `.js` files | Choose CJS vs ESM parsing |
| `exports` | External consumers | Modern entry points, conditional/subpath exports, encapsulation |
| `imports` | This package internally | Private `#`-prefixed aliases, conditional internal resolution |

---

## 10. Choosing CommonJS vs ESM in Real Projects

There's no universal "right" answer, but here's a practical decision framework.

### When ESM is the better default

- **New projects.** ESM is the language standard and the long-term direction of the ecosystem. Start new apps and libraries with `"type": "module"`.
- **You want top-level `await`**, live bindings, or static analysis benefits.
- **Shared/isomorphic code** with the browser (the same `import`/`export` works in both).
- **Tree-shaking matters** (bundled front-of-server code, serverless cold-start size).
- **Modern dependencies.** A growing number of popular packages (chalk, node-fetch, many others) are now **ESM-only** — using ESM avoids the dynamic-`import()` dance.

### When CommonJS still makes sense

- **Existing CJS codebases.** Don't rewrite a working app just to switch module systems. Migrate incrementally if at all.
- **Heavy reliance on CJS-only patterns**: synchronous `require()` deep in conditional logic, `require.cache` manipulation, monkey-patching tools, certain instrumentation/APM agents that hook `require`.
- **Tooling or runtime constraints** that still assume CommonJS.
- **Maximum compatibility** for a library that must be consumed by very old Node versions or old toolchains.

### The dual-package option (for library authors)

If you publish a library and want to support *both* kinds of consumers, ship both builds and wire them up with conditional `"exports"` (the `import`/`require` conditions in Section 9). Mind the dual-package hazard — keep stateful singletons in a shared internal module rather than duplicating them across builds.

| Consideration | Lean ESM | Lean CommonJS |
| --- | --- | --- |
| Brand-new project | ✅ | |
| Large existing CJS app | | ✅ (migrate gradually) |
| Need top-level `await` | ✅ | |
| Dependency is ESM-only | ✅ | (must use dynamic `import()`) |
| Heavy `require.cache`/monkey-patch tooling | | ✅ |
| Browser-shared code | ✅ | |
| Publishing a broadly-consumed library | Dual (both) | Dual (both) |

> BEST PRACTICE: For greenfield 2026 projects, start with ESM (`"type": "module"`), use the `node:` prefix for built-ins, name occasional CJS helpers `*.cjs`, and reach for dynamic `import()` only when bridging. Keep the public API defined via `"exports"`.

---

## Common Pitfalls & Gotchas

- **Reassigning `exports`.** `exports = fn` silently fails — `require` returns the original `{}`. Use `module.exports = fn`.
- **Mixing `module.exports = {}` then `exports.x = ...`.** After reassigning `module.exports`, the `exports` shorthand is stale. Stick to one approach.
- **Case-sensitive paths.** `require('./Utils')` works on macOS/Windows but fails on Linux (and thus in production/CI). Match filename casing exactly.
- **Forgetting extensions in ESM.** Relative ESM imports need `./file.js`; there's no auto-`.js` or auto-`index.js` like in CJS.
- **`__dirname`/`__filename` in ESM.** They don't exist — use `import.meta.dirname`/`import.meta.filename` or `fileURLToPath(import.meta.url)`.
- **Trying to `require()` an ESM package.** Use `await import()` from CommonJS (or rely on `require(esm)` only when you control the Node version).
- **Top-level `this`.** It's `module.exports` in CJS but `undefined` in ESM.
- **Circular dependencies returning partial exports.** A module imported mid-load hands back an incomplete object; late `module.exports =` reassignment is the classic break.
- **Mutating shared singletons.** Cached module exports are shared; one file mutating another's exported object affects everyone. `Object.freeze` configs you mean to be immutable.
- **`"exports"` blocking deep paths.** Adding `"exports"` makes unlisted subpaths inaccessible (`ERR_PACKAGE_PATH_NOT_EXPORTED`). List all public entry points.
- **Two copies of a dependency.** Different `node_modules` locations = different files = different cache entries = a broken "singleton."
- **Blocking top-level `await`.** A hanging top-level `await` stalls every importer's initialization.

## Best Practices

- **One responsibility, small public surface.** Export only what consumers need; keep internals private.
- **Prefer named exports** for application code; use `default` only for a module's single primary export.
- **Use the `node:` prefix** for built-in modules (`node:fs`, `node:path`).
- **Choose one export convention per file** and never mix `module.exports =` with `exports.x =`.
- **Lean on module caching** for app-wide singletons (config, DB connection, logger) instead of hand-rolled singleton classes.
- **`Object.freeze`** exported config objects you don't want mutated.
- **Eliminate circular dependencies** by extracting shared code; use lazy `require`/`import()` only as a stopgap.
- **In ESM, write file extensions** on relative imports and use `import.meta.dirname`/`import.meta.filename`.
- **Define a clear public API with `"exports"`** and tidy internal aliases with `"imports"` (`#`-prefixed).
- **Start new projects in ESM**, drop to `*.cjs` for the rare CommonJS helper, and bridge to ESM-only packages with `await import()`.
- **Place imports/requires at the top** (except deliberate lazy loads), so dependencies are obvious and load order is predictable.

## Key Interview Points

- **What is the difference between `module.exports` and `exports`?** `exports` is a local variable initialized to reference the same object as `module.exports`. Adding properties to `exports` works; *reassigning* `exports` breaks the link because `require` returns `module.exports`, not `exports`.
- **What is the module wrapper function?** Before execution, Node wraps each CommonJS file in `(function (exports, require, module, __filename, __dirname) { ... })` and invokes it. This is why those five identifiers exist and why each module has private scope.
- **Walk through the `require` resolution algorithm.** Core modules first; then relative/absolute paths (exact file → `.js`/`.json`/`.node` → directory); then bare specifiers walking up `node_modules`; folders resolve via `package.json` `"main"`/`"exports"` or `index.js`.
- **How does module caching work, and how does it give you a singleton?** The first `require` executes the module and caches `module.exports` by absolute path; later requires return the cached object without re-running. So module-level state is shared app-wide — a free singleton.
- **What happens with circular dependencies in Node?** Node returns the **partially completed** exports of the module that's still loading. The classic break is a late `module.exports = X` reassignment that peers captured before it was set. Fix by extracting shared code or lazy-requiring.
- **CommonJS vs ES Modules — name the key differences.** CJS: synchronous `require`, `module.exports`, `__dirname`, no top-level await, opt-in strict mode, reference copies. ESM: async, `import`/`export`, `import.meta.url`, top-level await, always strict, live bindings.
- **How do you use `__dirname` in an ES Module?** It doesn't exist; derive it with `import.meta.dirname` (Node 20.11+) or `dirname(fileURLToPath(import.meta.url))`.
- **How do CJS and ESM interoperate?** ESM can `import` CJS (its `module.exports` becomes the default export; named imports rely on static analysis). CommonJS loads ESM via `await import()` (or `require(esm)` on newer Node).
- **What is `dynamic import()` and why use it?** A Promise-returning expression that loads a module at runtime — for conditional/lazy loading and for importing ESM from CJS. Works in both module systems.
- **What does the `"type"` field do?** It sets how `.js` files are parsed: `"module"` = ESM, `"commonjs"`/absent = CJS. `.mjs`/`.cjs` override it.
- **What does `"exports"` add over `"main"`?** Multiple/subpath entry points, conditional exports (import vs require, node vs browser), and **encapsulation** — unlisted paths become inaccessible.
- **What is the dual-package hazard?** Shipping separate ESM and CJS builds can lead a consumer to load both, duplicating stateful singletons. Mitigate by keeping state in a single shared internal module.
- **Why prefer the `node:` prefix?** It's unambiguous, slightly faster to resolve, and can't be shadowed by a same-named `node_modules` package.

## Practice Exercises

Work through these in order; they build on each other. Try each before peeking at the solutions.

1. **Exercise 1 — Two export styles.** Create a CommonJS module `temperature.js` that exports two functions, `cToF` and `fToC`, using the `exports.name = ...` shorthand. Then create a second version `temperature2.js` that exports the *same* two functions by assigning a single object to `module.exports`. Require both from a `main.js` and prove they behave identically.

2. **Exercise 2 — Prove the cache is a singleton.** Create a module `id-generator.js` that keeps a private counter and exports a `nextId()` function returning an incrementing integer. From two *different* files (`a.js`, `b.js`), each require it and call `nextId()`. From `main.js`, require both `a` and `b` and then `id-generator` directly, and show that the IDs form a single continuous sequence (proving one shared instance). Add a top-level `console.log` in `id-generator.js` and confirm it prints only once.

3. **Exercise 3 — Break and fix a circular dependency.** Create `order.js` and `customer.js` where `order.js` requires `customer.js` and vice versa, with each exporting a class via `module.exports = Class`. Make `Order` reference `Customer` and `Customer` reference `Order`. Observe the failure (a partial export). Then refactor to remove the cycle.

4. **Exercise 4 — Port a module to ESM.** Take the `temperature.js` from Exercise 1 and rewrite it as an ES Module `temperature.mjs` using named exports. Write `app.mjs` that imports it. Then add a small piece of code in `app.mjs` that uses `import.meta.dirname` to print the directory, demonstrating the ESM replacement for `__dirname`.

5. **Exercise 5 — Dynamic import + interop.** In a CommonJS file `report.cjs`, load an ES Module `formatter.mjs` (which default-exports a function that uppercases a string) using dynamic `import()`. Print the formatted result. Bonus: make the load conditional on an environment variable.

6. **Exercise 6 — Design a package's public API.** Sketch a `package.json` for a library `geo-utils` that: is ESM (`"type": "module"`), exposes a root entry and a `./distance` subpath via `"exports"` with both `import` and `require` conditions, blocks all other deep paths, and defines a private internal alias `#math` via `"imports"`. Explain what `require('geo-utils/internal/secret')` would do.

### Solutions

**Solution 1 — Two export styles**

```js
// temperature.js — exports shorthand
exports.cToF = (c) => (c * 9) / 5 + 32;
exports.fToC = (f) => ((f - 32) * 5) / 9;
```

```js
// temperature2.js — single object assigned to module.exports
function cToF(c) { return (c * 9) / 5 + 32; }
function fToC(f) { return ((f - 32) * 5) / 9; }

module.exports = { cToF, fToC };
```

```js
// main.js
const t1 = require('./temperature');
const t2 = require('./temperature2');

console.log(t1.cToF(100), t2.cToF(100)); // 212 212
console.log(t1.fToC(32),  t2.fToC(32));  // 0 0
// Both styles produce an identical public object. The only rule:
// don't mix them — never do `module.exports = {}` then `exports.x = ...`.
```

**Solution 2 — Prove the cache is a singleton**

```js
// id-generator.js
console.log('id-generator.js evaluated'); // should print exactly ONCE

let current = 0; // private module-level state — the heart of the singleton
function nextId() { return ++current; }

module.exports = { nextId };
```

```js
// a.js
const { nextId } = require('./id-generator');
module.exports = nextId(); // grabs an id at load time
```

```js
// b.js
const { nextId } = require('./id-generator');
module.exports = nextId(); // grabs the next id at load time
```

```js
// main.js
const idFromA = require('./a');            // triggers id-generator (logs once) → 1
const idFromB = require('./b');            // reuses cached module           → 2
const { nextId } = require('./id-generator'); // same cached instance
const idFromMain = nextId();               //                                → 3

console.log(idFromA, idFromB, idFromMain); // 1 2 3 — one continuous sequence
// "id-generator.js evaluated" printed only once, proving a single shared instance.
```

**Solution 3 — Break and fix a circular dependency**

The broken version:

```js
// customer.js (broken)
const Order = require('./order'); // order.js is mid-load → gets partial export {}
class Customer {
  constructor(name) { this.name = name; this.orders = []; }
  placeOrder(amount) {
    const o = new Order(this, amount); // 💥 Order is {} here, not a class
    this.orders.push(o);
    return o;
  }
}
module.exports = Customer;
```

```js
// order.js (broken)
const Customer = require('./customer');
class Order {
  constructor(customer, amount) { this.customer = customer; this.amount = amount; }
}
module.exports = Order;
```

Running this throws `TypeError: Order is not a constructor` (the partial-export trap).

The fixed version — break the cycle by not having the classes import each other at module load. Here, neither class actually needs the *other module's class object at load time*; they only need instances passed in. So simply remove the unnecessary requires:

```js
// customer.js (fixed) — no require of order.js needed at all
class Customer {
  constructor(name) { this.name = name; this.orders = []; }
  // The Order class is injected/instantiated by the caller, so no import here.
  addOrder(order) { this.orders.push(order); return order; }
}
module.exports = Customer;
```

```js
// order.js (fixed) — no require of customer.js needed
class Order {
  constructor(customer, amount) { this.customer = customer; this.amount = amount; }
}
module.exports = Order;
```

```js
// main.js
const Customer = require('./customer');
const Order = require('./order');

const c = new Customer('Ada');
const o = new Order(c, 99);
c.addOrder(o);
console.log(c.orders[0].amount); // 99 — no cycle, no partial-export bug
```

If the classes genuinely must reference each other, extract the shared abstraction into a third module (`base.js`), or lazy-`require` inside methods so the require runs after both modules finish loading.

**Solution 4 — Port a module to ESM**

```js
// temperature.mjs — ES Module with named exports
export const cToF = (c) => (c * 9) / 5 + 32;
export const fToC = (f) => ((f - 32) * 5) / 9;
```

```js
// app.mjs  (in a "type": "module" project, or run as a .mjs file)
import { cToF, fToC } from './temperature.mjs'; // extension required in ESM

console.log(cToF(100)); // 212
console.log(fToC(32));  // 0

// ESM replacement for __dirname (Node 20.11+):
console.log('Running from:', import.meta.dirname);  // absolute directory path
console.log('This file:', import.meta.filename);    // absolute file path

// Portable fallback for older Node:
// import { fileURLToPath } from 'node:url';
// import { dirname } from 'node:path';
// const __dirname = dirname(fileURLToPath(import.meta.url));
```

**Solution 5 — Dynamic import + interop**

```js
// formatter.mjs — an ES Module, default-exports a function
export default function shout(str) {
  return String(str).toUpperCase() + '!';
}
```

```js
// report.cjs — CommonJS loading ESM via dynamic import()
async function main() {
  if (process.env.LOUD !== 'true') {     // bonus: conditional load
    console.log('Quiet mode; skipping formatter.');
    return;
  }
  const { default: shout } = await import('./formatter.mjs'); // ESM default → .default
  console.log(shout('deploy succeeded')); // DEPLOY SUCCEEDED!
}

main().catch((err) => {
  console.error('Failed:', err);
  process.exitCode = 1;
});
// Run with:  LOUD=true node report.cjs
```

**Solution 6 — Design a package's public API**

```json
{
  "name": "geo-utils",
  "version": "1.0.0",
  "type": "module",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    },
    "./distance": {
      "import": "./dist/distance.mjs",
      "require": "./dist/distance.cjs"
    }
  },
  "imports": {
    "#math": "./src/internal/math.js"
  }
}
```

```js
// Inside the package, internal code can do:
import { haversine } from '#math'; // private alias → ./src/internal/math.js
```

Explanation: Consumers may `import 'geo-utils'` (or `require` it) and `import 'geo-utils/distance'`. Any other deep path — including `require('geo-utils/internal/secret')` — throws **`ERR_PACKAGE_PATH_NOT_EXPORTED`**, because once `"exports"` is present it is strict and only the listed subpaths are reachable. The `#math` alias is private to the package and invisible to consumers; it just spares internal files from brittle relative paths and could be made conditional (e.g. different files for `node` vs `browser`).

## Summary / Cheat Sheet

| Topic | CommonJS | ES Modules |
| --- | --- | --- |
| Import | `const x = require('./x')` | `import x from './x.js'` |
| Export (single) | `module.exports = x` | `export default x` |
| Export (named) | `exports.x = x` | `export const x = ...` |
| Dynamic load | `require()` inline | `await import('...')` |
| File extension | `.cjs` / `.js` (no `"type":"module"`) | `.mjs` / `.js` (with `"type":"module"`) |
| Dir/file path | `__dirname` / `__filename` | `import.meta.dirname` / `import.meta.filename` |
| Top-level await | ❌ | ✅ |
| Strict mode | opt-in | always |
| Loading | synchronous | asynchronous |
| Bindings | copied at require | live, read-only |
| Top-level `this` | `module.exports` | `undefined` |

**Resolution order:** core (`node:`) → relative/absolute path (file → `.js`/`.json`/`.node` → folder) → `node_modules` walking up → folder via `"exports"`/`"main"`/`index.js`.

**`module.exports` vs `exports`:** same object initially; add properties to `exports`, but to *replace* the export assign `module.exports`. Never reassign `exports`.

**Module wrapper:** `(function (exports, require, module, __filename, __dirname) { ... })` — gives every CJS file private scope and those five identifiers.

**Caching = singleton:** first require runs & caches by absolute path; later requires reuse it. Module-level state is shared app-wide.

**Circular deps:** the still-loading module returns *partial* exports. Fix by extracting shared code or lazy-requiring; "export first, import second."

**`package.json` knobs:** `main` (legacy entry) · `type` (CJS/ESM for `.js`) · `exports` (modern entry points, conditions, encapsulation) · `imports` (`#` private aliases).

**Interop:** ESM can `import` CJS (→ default export). CJS loads ESM via `await import()` (or `require(esm)` on new Node). Named imports from CJS are best-effort.

**Default in 2026:** new projects → ESM (`"type": "module"`), `node:` prefixed built-ins, named exports, `*.cjs` for the occasional CommonJS helper, `await import()` to bridge.
