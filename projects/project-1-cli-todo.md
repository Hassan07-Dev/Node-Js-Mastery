# Project 1: Command-Line Todo App (Node Core Only)

> **Week 1 Capstone** · Difficulty: Beginner-friendly (assumes solid JavaScript, brand-new to backend) · Time: 3–5 hours · Dependencies: **zero npm packages**

This is the first hands-on project in the 5-week course. You already know JavaScript. What you do *not* yet know is what it feels like to write a *program that runs on a server-like runtime* — one that talks to the operating system, reads its own command-line arguments, touches the filesystem, and persists state between runs. Every one of those skills is the bedrock of backend development. We will build all of it using **nothing but the Node.js standard library**.

---

## Overview

We are building **`todo`** — a command-line task manager. You drive it from your terminal like a real Unix tool:

```bash
node todo.js add "Buy oat milk"
node todo.js list
node todo.js done 2
node todo.js delete 1
node todo.js clear
```

Concretely, the finished program will:

- **Add** a task with a description, automatically assigning it a stable numeric ID and a creation timestamp.
- **List** all tasks in a readable table, showing ID, status (`[ ]` / `[x]`), and text.
- **Complete** a task by ID, flipping its `done` flag and recording when it was completed.
- **Delete** a single task by ID.
- **Clear** every task (with a guard so you don't nuke your list by accident).
- **Persist** everything to a JSON file (`tasks.json`) so your list survives after the process exits — this is the part that makes it feel like real software, not a toy that forgets everything the moment it stops.

It runs entirely on **Node.js core modules**: `fs` (filesystem), `path` (cross-platform path building), `process` (arguments, exit codes, environment), and `readline` (interactive confirmation prompts). No Express, no Mongoose, no `commander`, no `chalk`, no `npm install` at all. That constraint is the whole point: before you reach for a framework, you should understand the raw machinery the framework hides.

> **TIP:** Later projects in this course (the REST API, the auth system) will use Express, MongoDB, and Mongoose. This project deliberately strips all of that away so the *runtime fundamentals* — argv parsing, the filesystem, sync vs async I/O, JSON serialization, error handling, modular structure — are burned into muscle memory first. Frameworks are easier to learn once you know what they're doing for you.

### What "backend" even means here (first principles)

If your background is front-end or pure-JS, "backend" can feel abstract. Here is the mental model this project installs:

- **A program reads input, does work, produces output, and may persist state.** In the browser, input is the DOM and user events; output is the rendered page. On the backend, input is often command-line arguments, environment variables, files, network requests; output is stdout/stderr, files, or HTTP responses; persisted state lives in files or databases.
- **The process is the unit of execution.** When you type `node todo.js add "x"`, the operating system starts a *process*, hands it your arguments, runs your code, and the process *exits* with a status code. Backend services are just long-lived processes; CLI tools are short-lived ones. Same runtime, same APIs.
- **Persistence is not free.** Variables vanish when the process exits. To remember anything, you must write it somewhere durable — a file (this project) or a database (later projects). That single idea is the seed of everything from session storage to caching to databases.

---

## What You'll Practice

By the end you will be comfortable with these backend fundamentals — each is called out again in the **Code Walkthrough** and shows up in interviews:

| Skill | Where it appears | Why it matters for backend work |
| --- | --- | --- |
| Reading `process.argv` | `cli.js` argument parser | Every CLI tool, build script, and many server bootstraps parse argv. |
| Process exit codes (`process.exit`, `process.exitCode`) | Error paths everywhere | Scripts, CI pipelines, and Docker health checks read exit codes. |
| Synchronous vs. asynchronous `fs` | `storage.js` | The single most important I/O distinction in Node; gets you the event-loop conversation in interviews. |
| Safe JSON read/write | `storage.js` | Config files, API payloads, and data stores are all JSON; corrupt-JSON handling is a real production concern. |
| Cross-platform paths with `path` | `storage.js`, `config.js` | Hard-coded `/` breaks on Windows; `path.join` is the professional default. |
| Modular file structure | whole repo | Separating storage / commands / CLI is the same instinct as routes/controllers/services later. |
| Centralized, typed error handling | `errors.js` + top-level handler | Distinguishing *expected* user errors from *bugs* is core to robust services. |
| Interactive prompts with `readline` | `clear` command | Streams and the readline interface generalize to log processing and TCP servers. |
| Environment-based configuration | `config.js` | `process.env` config is how every real deployment is tuned without code changes. |

---

## Prerequisites & Setup

### Requirements

- **Node.js current LTS** (v20 or v22). Check yours:

```bash
node --version   # expect v20.x or v22.x (any current LTS is fine)
```

- A terminal (macOS/Linux shell or Windows PowerShell/WSL).
- A code editor.

### npm install commands

There are none — and that is intentional. This project uses **only Node core modules**, so there is nothing to install. We still run `npm init` to create a `package.json`, because a real Node project always has one (it defines the entry point, scripts, and module system), and because it lets us add convenient `npm run` shortcuts.

```bash
# Create the project folder and enter it
mkdir cli-todo && cd cli-todo

# Generate package.json non-interactively (the -y accepts all defaults)
npm init -y

# Confirm you have ZERO dependencies — this should print nothing under "dependencies"
npm ls
```

> **BEST PRACTICE:** Even a dependency-free project benefits from `package.json`. It documents the entry point, pins the supported Node version via the `engines` field, and gives you `npm start` / `npm run` aliases so collaborators don't have to memorize long `node` commands.

### Folder structure

We use a small but *production-shaped* layout. The mapping to a real web service is intentional and noted on the right:

```text
cli-todo/
├── package.json            # Project metadata + scripts (no deps)
├── todo.js                 # Entry point (the executable) — like server.js in a web app
├── tasks.json             # Data store (created automatically on first write) — like your DB
└── src/
    ├── config.js           # Paths & env-based settings        → like config/env.js
    ├── storage.js          # Read/write the JSON file (data layer) → like a model/repository
    ├── errors.js           # Custom error types                → like utils/AppError.js
    ├── cli.js              # Parse argv → { command, args }     → like the router
    ├── render.js           # Format output for the terminal     → like a view/serializer
    └── commands/           # One handler per command            → like controllers
        ├── index.js        # Maps command name → handler        → like routes/index.js
        ├── add.js
        ├── list.js
        ├── done.js
        ├── delete.js
        └── clear.js
```

> **TIP — the parallel is the lesson:** `cli.js` is your *router*, `commands/*` are your *controllers*, `storage.js` is your *data layer / model*, `render.js` is your *view*, and `todo.js` is the *bootstrap*. When we build the Express REST API in a later project, you will recognize this exact shape. Learning it here, without HTTP in the way, makes the later jump much smaller.

Create the directories now:

```bash
mkdir -p src/commands
```

We will write each file in the build steps below. By the end, every file shown in the tree above exists and the app runs.

---

## Step 1 — `package.json`: project manifest & scripts

`npm init -y` already created this, but we will edit it so the project is self-documenting and gives us handy scripts. Replace the generated file with the following.

```json
// package.json
{
  "name": "cli-todo",
  "version": "1.0.0",
  "description": "A zero-dependency command-line todo manager built on Node core modules.",
  "type": "commonjs",
  "main": "todo.js",
  "bin": {
    "todo": "todo.js"
  },
  "scripts": {
    "start": "node todo.js",
    "add": "node todo.js add",
    "list": "node todo.js list"
  },
  "engines": {
    "node": ">=20"
  },
  "license": "MIT"
}
```

What each field does, since manifest fields are exactly the kind of thing newcomers gloss over:

- **`"type": "commonjs"`** — declares the *module system* for `.js` files. With `commonjs`, files use `require()` / `module.exports` (Node's original system, and our default for this course). The alternative is `"type": "module"`, which switches `.js` to ES Modules (`import` / `export`). We'll show the ESM variant at the end of Step 9.
- **`"main": "todo.js"`** — the entry point if someone `require()`s this package.
- **`"bin"`** — maps a *command name* to a script. If this package were installed globally (`npm i -g .`), you could type `todo add "x"` directly. We won't rely on that, but it's how real CLI tools (like `eslint`, `prettier`) expose themselves.
- **`"scripts"`** — shortcuts runnable via `npm run <name>` (and `npm start` for the special `start` script).
- **`"engines"`** — declares the minimum Node version. npm warns if a user is below it; it's documentation that won't go stale.

> **WARNING — Pitfall:** Mixing module systems silently breaks things. If `package.json` says `"type": "commonjs"` but you write `import x from './y.js'`, Node throws `SyntaxError: Cannot use import statement outside a module`. Pick one system per project. We use CommonJS throughout, then show a clean ESM conversion at the end so you've seen both.

---

## Step 2 — `src/config.js`: paths and environment configuration

Before any file I/O, decide **where data lives** and centralize it. Hard-coding `'tasks.json'` in ten places is how bugs are born; one config module is the professional move. This is also our first real use of `path` and `process.env`.

```js
// src/config.js
// Centralized configuration. Everything path- or environment-related lives here
// so the rest of the code never hard-codes a filename or directory.

const path = require('path'); // Node core: build cross-platform file paths

// process.cwd() is the directory the user RAN the command from (the "current
// working directory"), which is NOT necessarily where this file lives.
// __dirname is the directory of THIS file. The difference matters — see the
// WARNING below — so we choose deliberately.
//
// We anchor the data file to the project root (one level up from /src) using
// __dirname, so `node todo.js` works the same no matter which folder you run it
// from. An env var (TODO_FILE) can override it — handy for tests and for users
// who want a custom location.

const PROJECT_ROOT = path.join(__dirname, '..'); // /src -> project root

// path.join handles separators correctly on every OS:
//   POSIX  -> /Users/you/cli-todo/tasks.json
//   Windows-> C:\Users\you\cli-todo\tasks.json
const DEFAULT_DATA_FILE = path.join(PROJECT_ROOT, 'tasks.json');

const config = {
  // Where tasks are stored. TODO_FILE env var wins if set (used by tests).
  dataFile: process.env.TODO_FILE || DEFAULT_DATA_FILE,

  // Name shown in usage/help text. Lets us rename the tool in one place.
  programName: 'todo',
};

module.exports = config; // CommonJS export: callers do `const config = require('./config')`
```

> **WARNING — Pitfall: `__dirname` vs `process.cwd()`.** They are different and confusing the two is a classic bug. `__dirname` is *where the file is on disk*; `process.cwd()` is *where the user launched the process*. If you anchor your data file to `process.cwd()`, then `node ../cli-todo/todo.js list` from another folder reads a `tasks.json` in *that other folder* — surprising the user. We anchor to `__dirname` so the data file is stable. (In ESM there is no `__dirname`; the Step 9 conversion shows the replacement.)

> **BEST PRACTICE:** Allow an environment-variable override for anything path-like (`TODO_FILE` here). It costs one line and makes the code testable (point tests at a temp file) and deployable (point production at a mounted volume) without edits. This is the same pattern as `DATABASE_URL` and `PORT` in the web projects to come.

---

## Step 3 — `src/errors.js`: custom error types

Robust programs distinguish two kinds of failure:

1. **Operational / expected errors** — the user did something wrong (empty task text, bad ID, deleting a task that doesn't exist). These should produce a *clean, friendly message* and a non-zero exit code. They are *not bugs*.
2. **Programmer errors / unexpected failures** — something we didn't anticipate (a permissions error, a `null` we didn't guard). These deserve a louder message and, in development, a stack trace.

We encode that distinction with a custom error class. This exact pattern (an `AppError` with a flag marking it "expected") reappears in the Express projects' centralized error middleware — learning it now pays off twice.

```js
// src/errors.js
// Custom error types let us tell EXPECTED user mistakes apart from real bugs,
// and attach a process exit code to each failure.

// Base class for errors we deliberately throw in response to bad input or
// recoverable conditions. `isOperational = true` flags "this is a known,
// user-facing situation, not a crash."
class AppError extends Error {
  constructor(message, exitCode = 1) {
    super(message);
    this.name = this.constructor.name; // e.g. "ValidationError" instead of "Error"
    this.isOperational = true;         // marks this as an expected, handled error
    this.exitCode = exitCode;          // process exit code to use when this bubbles up
    // Keep the stack trace clean (omit this constructor frame).
    if (Error.captureStackTrace) Error.captureStackTrace(this, this.constructor);
  }
}

// Thrown when the user's input is invalid (missing text, non-numeric id, etc.).
class ValidationError extends AppError {
  constructor(message) {
    super(message, 2); // exit code 2 == "misuse / bad input" by convention
  }
}

// Thrown when an operation refers to something that isn't there (no task #99).
class NotFoundError extends AppError {
  constructor(message) {
    super(message, 3);
  }
}

module.exports = { AppError, ValidationError, NotFoundError };
```

A note on exit codes, because they're invisible to GUI users but vital on the backend: by Unix convention, `0` means success and any non-zero value means failure. CI systems, shell scripts (`command && next`), Docker, and orchestrators all branch on this number. We give different error classes different codes so a calling script could distinguish "bad input" (2) from "not found" (3) if it wanted to.

> **BEST PRACTICE:** Extend a base `Error` rather than throwing strings. `throw 'bad id'` loses the stack trace and breaks `instanceof` checks. A real `Error` subclass lets the top-level handler ask `if (err instanceof ValidationError)` or `if (err.isOperational)` and respond appropriately.

---

## Step 4 — `src/storage.js`: the data layer (sync vs async fs)

This is the heart of the project and the densest learning. `storage.js` is the *only* file that touches the filesystem; everything else asks it for tasks and hands it tasks to save. That single-responsibility boundary is exactly how a model/repository works in a database app.

We'll implement it with the **asynchronous, promise-based `fs` API** (`fs.promises`) as the primary path, then discuss the synchronous alternative right after, because *understanding the difference is the single most-tested Node concept in interviews.*

```js
// src/storage.js
// The DATA LAYER. The only module that reads/writes the filesystem.
// Responsibilities:
//   - load tasks from tasks.json (handling "file missing" and "corrupt JSON")
//   - save tasks back atomically-ish and safely
// It knows nothing about commands or argv — it just moves task arrays to/from disk.

const fs = require('fs/promises'); // promise-based fs API (async/await friendly)
const fsSync = require('fs');      // sync API, used only for one targeted check
const config = require('./config');

// The shape of our data file is a single JSON object so we can evolve the
// schema later (e.g. add a "version" or "settings" key) without breaking
// readers. `tasks` is the array of task objects.
//
//   { "tasks": [ { id, text, done, createdAt, completedAt }, ... ] }
//
const EMPTY_DB = { tasks: [] };

/**
 * Load the full database object from disk.
 * Returns { tasks: [...] }. Resilient to two common failures:
 *   1. The file doesn't exist yet (first run) -> return an empty DB.
 *   2. The file exists but contains invalid JSON -> throw a clear error
 *      instead of letting a cryptic SyntaxError escape.
 */
async function load() {
  let raw;
  try {
    // 'utf8' makes readFile return a string (not a Buffer of bytes).
    raw = await fs.readFile(config.dataFile, 'utf8');
  } catch (err) {
    // ENOENT = "Error NO ENTry" = the file does not exist. That's expected on
    // first run, so we treat it as an empty list rather than an error.
    if (err.code === 'ENOENT') {
      return { ...EMPTY_DB };
    }
    // Any OTHER fs error (e.g. EACCES = permission denied) is real and unexpected.
    // Re-throw it so the top-level handler reports it loudly.
    throw err;
  }

  // An empty file is a degenerate but recoverable case (e.g. a crash mid-write).
  if (raw.trim() === '') {
    return { ...EMPTY_DB };
  }

  // Parsing is the second failure point. A truncated or hand-edited file can be
  // invalid JSON. We catch the SyntaxError and turn it into an actionable message.
  let db;
  try {
    db = JSON.parse(raw);
  } catch (err) {
    throw new Error(
      `tasks file is corrupted (invalid JSON) at ${config.dataFile}\n` +
        `Fix or delete the file and try again. Parser said: ${err.message}`
    );
  }

  // Defensive normalization: guarantee callers always get { tasks: [...] }
  // even if the file was hand-edited into a weird shape.
  if (!db || typeof db !== 'object' || !Array.isArray(db.tasks)) {
    return { ...EMPTY_DB };
  }
  return db;
}

/**
 * Persist the full database object to disk.
 * Uses a write-to-temp-then-rename strategy so a crash mid-write can't leave a
 * half-written, corrupt tasks.json. `rename` is atomic on the same filesystem.
 */
async function save(db) {
  // null, 2 -> pretty-print with 2-space indentation so the file is human-readable.
  const json = JSON.stringify(db, null, 2);

  const tmpFile = `${config.dataFile}.tmp`;
  // Write the new content to a temporary sibling file first...
  await fs.writeFile(tmpFile, json, 'utf8');
  // ...then atomically move it into place. If the process dies before this line,
  // the original tasks.json is still intact.
  await fs.rename(tmpFile, config.dataFile);
}

/**
 * Convenience: does the data file already exist? Uses the SYNC API on purpose
 * to demonstrate it — a one-shot existence check at startup is a legitimate use
 * of sync fs (see the discussion below).
 */
function dataFileExists() {
  // existsSync returns a boolean and never throws — handy, but DON'T use it to
  // "check before read" in async code (see the TOCTOU warning below).
  return fsSync.existsSync(config.dataFile);
}

module.exports = { load, save, dataFileExists, EMPTY_DB };
```

### Synchronous vs. asynchronous `fs` — the concept that matters most

Node ships every filesystem operation in three flavors. Knowing when to reach for each is fundamental backend knowledge.

| Flavor | Example | Returns | Blocks the event loop? | Use when |
| --- | --- | --- | --- | --- |
| **Sync** | `fs.readFileSync(p)` | the value, or throws | **Yes** — nothing else runs until it finishes | One-shot startup scripts, CLIs doing one thing, config load before the server starts |
| **Callback async** | `fs.readFile(p, cb)` | nothing; calls `cb(err, data)` | No | Legacy code, callback-based libraries |
| **Promise async** | `fs.promises.readFile(p)` | a `Promise` | No | Modern code; pairs with `async/await` |

What "blocks the event loop" means, from first principles: Node runs your JavaScript on **a single thread**. While a *synchronous* call like `readFileSync` is reading the disk, that thread can do *nothing else* — no other request handled, no timer fired, no other I/O progressed. In a **CLI tool that does one task and exits** (like ours), blocking is harmless and the sync API is arguably *simpler*. In a **web server handling many concurrent users**, a single `readFileSync` on the request path freezes *every* user until the disk responds — a cardinal sin.

We chose the **promise-based async** API here on purpose: not because this CLI needs concurrency, but because it's the API you'll use in the server projects, and practicing `async/await` with `fs.promises` now means no relearning later. The sync version of `load`, for comparison, is genuinely a little shorter:

```js
// Sync alternative (for comparison — NOT what we use). Note: no async/await,
// errors thrown synchronously so a plain try/catch wraps it.
const fsSync = require('fs');

function loadSync() {
  let raw;
  try {
    raw = fsSync.readFileSync(config.dataFile, 'utf8');
  } catch (err) {
    if (err.code === 'ENOENT') return { tasks: [] };
    throw err;
  }
  return raw.trim() === '' ? { tasks: [] } : JSON.parse(raw);
}
```

> **WARNING — Pitfall: the "check-then-act" (TOCTOU) race.** It is tempting to write `if (fs.existsSync(file)) { data = await fs.readFile(file) }`. Don't. Between the *check* and the *act*, the file could be deleted by another process — a classic **Time-Of-Check to Time-Of-Use** bug. The correct pattern is to *just do the operation and handle the failure*: attempt the `readFile` and catch `ENOENT`, exactly as `load()` does above. `existsSync` is fine for a non-critical "have we initialized yet?" hint, but never as a guard before a real operation.

> **BEST PRACTICE: write-temp-then-rename.** Our `save()` writes to `tasks.json.tmp` and then `rename`s it over `tasks.json`. `rename` within one filesystem is *atomic* — readers either see the old complete file or the new complete file, never a half-written one. A naive `fs.writeFile(dataFile, ...)` that's interrupted (power loss, `Ctrl-C` at the wrong moment) can truncate your data. This is the same durability instinct databases use internally.

> **WARNING — Pitfall: forgetting the encoding.** `await fs.readFile(path)` with no encoding returns a **`Buffer`** (raw bytes), and `JSON.parse(buffer)` works only because `parse` stringifies it first — but logging or string-comparing the result will surprise you. Always pass `'utf8'` when you want text.

---

## Step 5 — `src/render.js`: formatting output (the "view")

Keep presentation out of business logic. `render.js` turns task data into terminal-friendly strings. Separating it means we could later swap in JSON output or colors without touching command logic — the same reason web apps separate views/serializers from controllers.

```js
// src/render.js
// The VIEW layer: pure functions that turn task data into display strings.
// No fs, no argv, no side effects — just data in, formatted text out.

const config = require('./config');

/**
 * Render the full task list as an aligned table.
 * @param {Array} tasks
 * @returns {string}
 */
function renderList(tasks) {
  if (tasks.length === 0) {
    return 'No tasks yet. Add one with:\n' +
      `  ${config.programName} add "Your first task"`;
  }

  // Compute the width of the widest ID so the columns line up neatly.
  const idWidth = Math.max(2, ...tasks.map((t) => String(t.id).length));

  const lines = tasks.map((t) => {
    const id = String(t.id).padStart(idWidth, ' '); // right-align ids
    const box = t.done ? '[x]' : '[ ]';             // checkbox-style status
    return `${id}. ${box} ${t.text}`;
  });

  const doneCount = tasks.filter((t) => t.done).length;
  const footer = `\n${doneCount}/${tasks.length} done`;
  return lines.join('\n') + footer;
}

/** A single confirmation line after add/done/delete, for nice UX. */
function renderTask(task, verb) {
  const box = task.done ? '[x]' : '[ ]';
  return `${verb} #${task.id}: ${box} ${task.text}`;
}

/** Usage / help text shown for `help`, no command, or unknown command. */
function renderUsage() {
  const p = config.programName;
  return [
    `${p} — a command-line todo manager`,
    '',
    'USAGE:',
    `  ${p} <command> [arguments]`,
    '',
    'COMMANDS:',
    `  add <text>     Add a new task            e.g. ${p} add "Buy milk"`,
    `  list           List all tasks            e.g. ${p} list`,
    `  done <id>      Mark a task complete       e.g. ${p} done 2`,
    `  delete <id>    Delete a task by id        e.g. ${p} delete 2`,
    `  clear          Delete ALL tasks           e.g. ${p} clear`,
    `  help           Show this help`,
  ].join('\n');
}

module.exports = { renderList, renderTask, renderUsage };
```

> **TIP:** Notice these functions are *pure*: same input, same output, no I/O. Pure view functions are trivially testable (assert on the returned string) and reusable. Pushing side effects (printing, file writes) to the edges of your program is a habit that scales all the way up to large services.

---

## Step 6 — `src/commands/*`: the command handlers (controllers)

Each command is one small module that: validates input, loads the DB, mutates it, saves it, and returns a message to print. They depend on `storage` and `errors` but never touch `argv` or `console` directly — they receive parsed args and *return* text. That keeps them pure-ish and testable, exactly like a thin controller that returns data for the framework to send.

A shared idea across handlers: the **next ID**. We don't reuse the array index (deleting task 2 would renumber everything, breaking IDs the user already saw). Instead we compute `max(existing ids) + 1`, giving stable, monotonically increasing IDs.

### `src/commands/add.js`

```js
// src/commands/add.js
// Handler for: todo add "task text"

const storage = require('../storage');
const { renderTask } = require('../render');
const { ValidationError } = require('../errors');

async function add(args) {
  // args is everything after the command word. We join so that an UNQUOTED
  // multi-word task ("add buy milk") still works, while quoted text arrives as one arg.
  const text = args.join(' ').trim();

  if (!text) {
    throw new ValidationError(
      'Task text is required.\n  Example: todo add "Buy oat milk"'
    );
  }

  const db = await storage.load();

  // Stable next id: one past the current maximum (handles gaps from deletes).
  const nextId =
    db.tasks.length === 0 ? 1 : Math.max(...db.tasks.map((t) => t.id)) + 1;

  const task = {
    id: nextId,
    text,
    done: false,
    createdAt: new Date().toISOString(), // ISO 8601 string sorts & parses cleanly
    completedAt: null,
  };

  db.tasks.push(task);
  await storage.save(db);

  return renderTask(task, 'Added');
}

module.exports = add;
```

### `src/commands/list.js`

```js
// src/commands/list.js
// Handler for: todo list

const storage = require('../storage');
const { renderList } = require('../render');

async function list() {
  const db = await storage.load();
  return renderList(db.tasks);
}

module.exports = list;
```

### `src/commands/done.js`

```js
// src/commands/done.js
// Handler for: todo done <id>

const storage = require('../storage');
const { renderTask } = require('../render');
const { ValidationError, NotFoundError } = require('../errors');

async function done(args) {
  const id = parseId(args[0]); // validates and converts to a number

  const db = await storage.load();
  const task = db.tasks.find((t) => t.id === id);

  if (!task) {
    throw new NotFoundError(`No task with id ${id}. Run "todo list" to see ids.`);
  }

  if (task.done) {
    return renderTask(task, 'Already done'); // idempotent, friendly no-op
  }

  task.done = true;
  task.completedAt = new Date().toISOString();
  await storage.save(db);

  return renderTask(task, 'Completed');
}

// Shared helper: turn a raw string argument into a valid positive integer id,
// or throw a clear ValidationError. Reused by done.js and delete.js.
function parseId(raw) {
  if (raw === undefined) {
    throw new ValidationError('An id is required.  Example: todo done 2');
  }
  // Number() is strict: "2abc" -> NaN, unlike parseInt("2abc") -> 2.
  const id = Number(raw);
  if (!Number.isInteger(id) || id <= 0) {
    throw new ValidationError(`"${raw}" is not a valid task id (expected a positive whole number).`);
  }
  return id;
}

module.exports = done;
module.exports.parseId = parseId; // export the helper so delete.js can reuse it
```

> **WARNING — Pitfall: `Number` vs `parseInt`.** `parseInt("2abc", 10)` returns `2` — it stops at the first non-digit and silently accepts garbage. `Number("2abc")` returns `NaN`. For *validating* user input you want the strict behavior, so we use `Number` plus `Number.isInteger`. Reaching for `parseInt` here would let `done 2; rm -rf` -style typos slip through as a valid id.

### `src/commands/delete.js`

```js
// src/commands/delete.js
// Handler for: todo delete <id>

const storage = require('../storage');
const { renderTask } = require('../render');
const { NotFoundError } = require('../errors');
const { parseId } = require('./done'); // reuse the validated id parser

async function remove(args) {
  const id = parseId(args[0]);

  const db = await storage.load();
  const index = db.tasks.findIndex((t) => t.id === id);

  if (index === -1) {
    throw new NotFoundError(`No task with id ${id}. Run "todo list" to see ids.`);
  }

  // splice returns an array of removed items; grab the one we removed for the message.
  const [removed] = db.tasks.splice(index, 1);
  await storage.save(db);

  return renderTask(removed, 'Deleted');
}

// "delete" is a reserved word in JS, so the function is named `remove` internally
// but exported under the command's real name via the registry in index.js.
module.exports = remove;
```

### `src/commands/clear.js`

This is where `readline` earns its place: clearing everything is destructive, so we *interactively confirm* unless the user passes `--yes`/`-y`. Reading a line from stdin via the `readline` interface is your first taste of Node **streams** — `process.stdin` is a readable stream, `process.stdout` a writable one — which generalizes to log pipelines and network sockets later.

```js
// src/commands/clear.js
// Handler for: todo clear   (asks for confirmation unless --yes / -y given)

const readline = require('readline'); // Node core: line-by-line stdin reading
const storage = require('../storage');

async function clear(args) {
  const db = await storage.load();

  if (db.tasks.length === 0) {
    return 'Nothing to clear — your list is already empty.';
  }

  // Allow a non-interactive override so scripts/tests don't hang on a prompt.
  const skipConfirm = args.includes('--yes') || args.includes('-y');

  if (!skipConfirm) {
    const ok = await confirm(
      `This will permanently delete all ${db.tasks.length} task(s). Continue? (y/N) `
    );
    if (!ok) return 'Cancelled. No tasks were deleted.';
  }

  const count = db.tasks.length;
  db.tasks = []; // wipe in memory...
  await storage.save(db); // ...then persist the empty list.

  return `Cleared ${count} task(s).`;
}

/**
 * Prompt the user and resolve to true only if they type y / yes (case-insensitive).
 * Wraps the callback-based readline.question in a Promise so callers can await it.
 */
function confirm(question) {
  return new Promise((resolve) => {
    const rl = readline.createInterface({
      input: process.stdin,   // a readable stream (your keyboard)
      output: process.stdout, // a writable stream (your terminal)
    });
    rl.question(question, (answer) => {
      rl.close(); // ALWAYS close the interface or the process won't exit
      const normalized = answer.trim().toLowerCase();
      resolve(normalized === 'y' || normalized === 'yes');
    });
  });
}

module.exports = clear;
```

> **WARNING — Pitfall: a `readline` interface keeps the process alive.** If you open `readline.createInterface` and never call `rl.close()`, the process *hangs forever* after doing its work, because an open input stream is a reason for Node to keep running. Always `rl.close()` once you have your answer — note we do it inside the callback, before resolving.

> **TIP: default to the safe choice.** The prompt shows `(y/N)` with a capital **N**, signaling that just pressing Enter (empty answer) means *No*. Destructive operations should require an explicit "yes," never assume it.

### `src/commands/index.js` — the command registry (router)

This module maps a command *name* to its *handler*, plus the `help` command. The main entry point will consult this registry — that's literally what an Express router does for URL paths.

```js
// src/commands/index.js
// The command registry: maps a command name -> its handler function.
// This is the CLI equivalent of a router's route table.

const add = require('./add');
const list = require('./list');
const done = require('./done');
const remove = require('./delete');
const clear = require('./clear');
const { renderUsage } = require('../render');

// `help` is implemented inline since it has no data dependencies.
async function help() {
  return renderUsage();
}

// Keys are the words the user types. Some commands have aliases.
const commands = {
  add,
  list,
  ls: list,        // alias: `todo ls`
  done,
  complete: done,  // alias: `todo complete 2`
  delete: remove,
  del: remove,     // alias: `todo del 2`
  rm: remove,      // alias: `todo rm 2`
  clear,
  help,
};

module.exports = commands;
```

> **BEST PRACTICE:** A lookup table (`{ add, list, ... }`) beats a long `switch (command)` for dispatch: adding a command is a one-line registry entry, aliases are trivial, and you can programmatically list all valid commands (useful for "did you mean?" suggestions). This data-driven dispatch is the same idea behind route tables and plugin systems.

---

## Step 7 — `src/cli.js`: parsing `process.argv` (the router)

Now the piece that makes it a *command-line* tool: turning the raw argument vector into a structured `{ command, args }`. Understanding the *shape* of `process.argv` is essential and frequently misunderstood.

```js
// src/cli.js
// Parses Node's raw argv into a structured command + arguments.

/**
 * process.argv is an array. Its first two elements are ALWAYS:
 *   [0] -> the path to the node executable     (e.g. /usr/local/bin/node)
 *   [1] -> the path to the script being run    (e.g. /Users/you/cli-todo/todo.js)
 * Everything the USER typed starts at index 2. So for:
 *
 *   node todo.js add "Buy milk"
 *
 * process.argv === [
 *   '/usr/local/bin/node',
 *   '/Users/you/cli-todo/todo.js',
 *   'add',
 *   'Buy milk'          <- the shell already stripped the quotes
 * ]
 *
 * @param {string[]} argv - pass process.argv (injected for testability)
 * @returns {{ command: string, args: string[] }}
 */
function parse(argv) {
  const userArgs = argv.slice(2); // drop node + script paths
  const [command = 'help', ...args] = userArgs; // default to "help" if nothing typed
  return { command: command.toLowerCase(), args };
}

module.exports = { parse };
```

A few things worth internalizing:

- **The shell, not Node, handles quotes.** When you type `add "Buy milk"`, your *shell* removes the quotes and passes `Buy milk` as a single argument. Type `add Buy milk` (no quotes) and you get *two* arguments, `Buy` and `milk` — which is exactly why `add.js` does `args.join(' ')`, so both forms work.
- **We inject `argv` as a parameter** rather than reading the global `process.argv` inside `parse`. That tiny choice makes the parser unit-testable: a test can call `parse(['node', 'todo.js', 'add', 'x'])` without spawning a process. Dependency injection at the smallest scale.
- **Defaulting to `help`** means a bare `node todo.js` does something friendly instead of crashing.

> **TIP — going further on flags:** Real CLIs parse flags like `--priority high` or `-y`. For one project, manual checks (as in `clear.js`'s `args.includes('--yes')`) are fine. For anything bigger, Node 18.3+ ships a built-in `util.parseArgs()` that handles `--flag value`, booleans, and short options — still zero npm deps. It's a great "Extensions" exercise (see below).

---

## Step 8 — `todo.js`: the entry point (bootstrap + top-level error handler)

The entry point wires everything together and owns the program's lifecycle: parse argv → find the handler → run it → print the result → handle any error → set the exit code. Crucially, it is the **single place** that catches errors and decides how to report them, using the `isOperational` flag from Step 3. Centralizing error handling here is the CLI analogue of Express's error-handling middleware.

```js
// todo.js
// ENTRY POINT. Run as: node todo.js <command> [args]
// Responsibilities: dispatch the command, print its output, and centrally
// translate any thrown error into a friendly message + correct exit code.

const { parse } = require('./src/cli');
const commands = require('./src/commands');
const { renderUsage } = require('./src/render');

async function main() {
  const { command, args } = parse(process.argv);

  const handler = commands[command];

  // Unknown command: show usage and exit with a "misuse" code (2).
  if (!handler) {
    console.error(`Unknown command: "${command}"\n`);
    console.error(renderUsage());
    process.exitCode = 2; // set the code; let the process finish naturally
    return;
  }

  // Run the handler and print whatever string it returns.
  const output = await handler(args);
  if (output) console.log(output);
  // Successful runs leave process.exitCode at its default 0.
}

// Top-level invocation with a single catch-all. EVERY thrown error in the app
// funnels here, so error reporting policy lives in exactly one place.
main().catch((err) => {
  if (err && err.isOperational) {
    // Expected user-facing error: print just the message, no scary stack trace.
    console.error(`Error: ${err.message}`);
    process.exitCode = err.exitCode || 1;
  } else {
    // Unexpected error / real bug: this is louder. Show the stack in dev so we
    // can debug; in a "production" build you might log it and show a generic msg.
    console.error('Unexpected error:');
    console.error(err && err.stack ? err.stack : err);
    process.exitCode = 1;
  }
});
```

Why `process.exitCode = n` instead of `process.exit(n)`? This is a subtle but real correctness issue:

- `process.exit(n)` **terminates immediately**, potentially *before* buffered `stdout`/`stderr` writes are flushed or before async work finishes — you can lose your last line of output.
- Setting `process.exitCode = n` and letting `main()` return allows Node to drain its work and exit *naturally* with that code. Same result, no truncation risk.

> **BEST PRACTICE:** Prefer `process.exitCode = n; return;` over `process.exit(n)` in normal flows. Reserve `process.exit()` for the rare case where you must bail out *right now* (e.g. a fatal startup error before anything else runs). This is one of the most common subtle Node bugs in production scripts — output that randomly goes missing because someone called `process.exit()` too eagerly.

> **WARNING — Pitfall: an async error with no `.catch` becomes an unhandled rejection.** If `main()` is an `async` function (it is) and you call it without `.catch(...)`, a thrown error turns into an `UnhandledPromiseRejection`. On modern Node that *crashes the process with an ugly trace and a non-zero code* — but you've lost the chance to print a friendly message. The trailing `.catch` is mandatory, not decorative.

The application is now complete. Every file in the Step 0 tree exists. Let's run it.

---

## How to Run & Test

There is no HTTP server here, so we test from the **terminal** rather than with curl/Postman — the command line *is* the interface. (The REST API project later in the course is where curl and Postman come in; this section shows the CLI equivalents.)

### A full walkthrough session

Run these in order from the project root. Comments show the expected output.

```bash
# 1) No command -> friendly help, exit code 0
node todo.js
# -> prints the usage text

# 2) Add a few tasks (quotes keep multi-word text as one task)
node todo.js add "Buy oat milk"
# -> Added #1: [ ] Buy oat milk
node todo.js add "Write project 1 guide"
# -> Added #2: [ ] Write project 1 guide
node todo.js add Finish the readme        # works WITHOUT quotes too (args joined)
# -> Added #3: [ ] Finish the readme

# 3) List them
node todo.js list
# ->  1. [ ] Buy oat milk
#     2. [ ] Write project 1 guide
#     3. [ ] Finish the readme
#
#     0/3 done

# 4) Complete task 2 (alias `complete` works too)
node todo.js done 2
# -> Completed #2: [x] Write project 1 guide

node todo.js list
# ->  1. [ ] Buy oat milk
#     2. [x] Write project 1 guide
#     3. [ ] Finish the readme
#
#     1/3 done

# 5) Delete task 1 (note: ids do NOT renumber — 2 and 3 are unchanged)
node todo.js delete 1
# -> Deleted #1: [ ] Buy oat milk

node todo.js list
# ->  2. [x] Write project 1 guide
#     3. [ ] Finish the readme
#
#     1/2 done

# 6) Persistence proof: the list survived across separate process runs above.
#    Inspect the data file directly:
cat tasks.json
# -> { "tasks": [ { "id": 2, ... }, { "id": 3, ... } ] }
```

### Testing the interactive `clear` prompt

```bash
# Interactive: it will ASK before deleting everything.
node todo.js clear
# -> This will permanently delete all 2 task(s). Continue? (y/N)
#    (type n, press Enter)  -> Cancelled. No tasks were deleted.

# Non-interactive (good for scripts): skip the prompt with --yes
node todo.js clear --yes
# -> Cleared 2 task(s).
```

### Testing the error paths (this is where robustness shows)

```bash
node todo.js add ""                  # empty text
# -> Error: Task text is required. ...        (exit code 2)

node todo.js done abc                 # non-numeric id
# -> Error: "abc" is not a valid task id ...  (exit code 2)

node todo.js done 999                 # nonexistent id
# -> Error: No task with id 999. ...          (exit code 3)

node todo.js frobnicate               # unknown command
# -> Unknown command: "frobnicate"            (exit code 2)
#    (followed by usage)

# Inspect the exit code of the LAST command (POSIX shells: $?  /  PowerShell: $LASTEXITCODE)
node todo.js done 999; echo "exit code: $?"
# -> exit code: 3
```

### Testing corrupt-JSON recovery

```bash
# Deliberately break the data file, then watch the app fail GRACEFULLY.
echo "{ this is not valid json" > tasks.json
node todo.js list
# -> Error: tasks file is corrupted (invalid JSON) at .../tasks.json
#    Fix or delete the file and try again. ...

# Recover by removing the bad file; the next run starts fresh.
rm tasks.json
node todo.js list
# -> No tasks yet. Add one with: ...
```

### Using a custom data file (the env override from Step 2)

```bash
# Point at a throwaway file so you don't touch your real list — great for experiments/tests.
TODO_FILE=/tmp/test-tasks.json node todo.js add "scratch task"
TODO_FILE=/tmp/test-tasks.json node todo.js list
# Your real tasks.json is untouched.
```

> **TIP — a tiny smoke test with no test framework:** You can verify the whole happy path in one line using only the shell and exit codes — handy in CI without installing Jest:
> ```bash
> TODO_FILE=/tmp/t.json node todo.js add "a" && \
> TODO_FILE=/tmp/t.json node todo.js done 1 && \
> TODO_FILE=/tmp/t.json node todo.js list && \
> rm -f /tmp/t.json && echo "SMOKE TEST PASSED"
> ```
> Node 18+ also ships a built-in `node:test` runner (`node --test`) so you could write real unit tests for `cli.parse` and the pure `render` functions with *zero* dependencies — a natural extension.

---

## Code Walkthrough / Key Concepts Recap

A guided tour of *why* the code is shaped the way it is, mapped back to the skills table.

### 1. The request lifecycle (in CLI terms)

Trace what happens for `node todo.js done 2`:

1. The OS starts a Node **process**, populating `process.argv` with `['<node>', '<todo.js>', 'done', '2']`.
2. `todo.js`'s `main()` calls `cli.parse(process.argv)` → `{ command: 'done', args: ['2'] }`. **(argv parsing)**
3. It looks up `commands['done']` in the registry → the `done` handler. **(dispatch / routing)**
4. `done(['2'])` validates the id with `parseId` **(input validation)**, calls `storage.load()` **(async fs read)**, finds the task, mutates it, calls `storage.save()` **(safe async fs write)**, and returns a string.
5. `main()` prints the string; the process exits with code 0. **(exit codes)**

That five-step shape — *receive input → route → validate → touch data layer → respond* — is **identical** to an HTTP request flowing through Express. You've now built the skeleton by hand.

### 2. Sync vs. async `fs`, revisited

The single biggest takeaway: **Node is single-threaded for your JS, so blocking I/O blocks everything.** A CLI that does one thing can afford sync I/O; a server handling many clients cannot. We used `fs.promises` to build the `async/await` habit you'll need on the server, and we covered the **ENOENT** (file-missing) and **JSON.parse** (corrupt-file) failure points explicitly, because *real data layers must survive a missing or malformed store.*

### 3. JSON as the wire/storage format

`JSON.stringify(db, null, 2)` to write (pretty-printed for human inspection) and `JSON.parse` to read, each wrapped in error handling. JSON is the lingua franca of backend work — config files, API request/response bodies, and many databases. The habit of *never trusting that a JSON read will succeed* transfers directly to parsing untrusted API payloads.

### 4. Modular boundaries

| Module | Role | Web-app analogue |
| --- | --- | --- |
| `cli.js` | argv → `{command, args}` | router |
| `commands/*` | per-command logic | controllers |
| `storage.js` | the only fs-touching code | model / repository |
| `render.js` | data → display strings | view / serializer |
| `config.js` | paths & env | config module |
| `errors.js` | typed errors | shared error utils |
| `todo.js` | bootstrap + error policy | `server.js` + error middleware |

Each module has **one reason to change**. Want colored output? Touch only `render.js`. Want a SQLite backend instead of JSON? Rewrite only `storage.js` — its `load`/`save` interface stays the same, and nothing else cares. That decoupling is the practical payoff of the structure.

### 5. Error-handling strategy

We separated **operational errors** (`ValidationError`, `NotFoundError`, marked `isOperational`) from **unexpected errors**, and funneled *all* of them through one top-level `.catch` in `todo.js`. Operational errors print a clean message; unexpected ones print a stack trace for debugging. Each carries an appropriate **exit code**. This is the exact philosophy behind Express's centralized error middleware and the `AppError` pattern you'll reuse in the API projects.

### 6. Stable IDs and immutable-ish operations

We compute the next ID as `max(ids) + 1` rather than using array length or index, so IDs the user has already seen never change out from under them. Delete uses `splice` and returns the removed item for the confirmation message. These are small correctness details that separate a toy from a tool.

### Interview-ready soundbites

- *"`process.argv[0]` and `[1]` are the node and script paths; user arguments begin at index 2."*
- *"Sync fs blocks the single JS thread, which is fine in a CLI but fatal on a request path; that's why I default to `fs.promises` on servers."*
- *"I never check-then-act on a file (TOCTOU); I attempt the operation and handle `ENOENT`."*
- *"I set `process.exitCode` and return instead of calling `process.exit()`, so buffered output isn't truncated."*
- *"I distinguish operational errors from programmer errors with an `isOperational` flag and handle them centrally."*
- *"I write to a temp file and `rename` it into place so a crash mid-write can't corrupt the data store."*

---

## Extensions & Challenges

Each builds on the codebase without adding a single dependency. Roughly ordered easy → hard.

1. **Due dates.** Extend `add` to accept `--due 2026-07-01`. Parse it with `util.parseArgs`, validate it's a real date (`!Number.isNaN(Date.parse(value))`), store `dueDate` on the task, and have `renderList` flag overdue tasks (e.g. append `(overdue)` when `dueDate < now` and not done).

2. **Priorities.** Add `--priority high|med|low` to `add`. Store it, then add a `list --sort priority` option that sorts tasks high→low. This forces you to handle flags *and* a comparator.

3. **Search / filter.** Implement `todo search <term>` that case-insensitively filters task text, and `todo list --done` / `--pending` to filter by status. Reuse `renderList` so the display code stays untouched — proof your view layer was well-factored.

4. **Colors with zero deps.** ANSI escape codes need no library. Add a `src/colors.js`:
   ```js
   // src/colors.js — ANSI colors with no dependencies.
   // Only colorize when stdout is an interactive terminal (not a pipe/file),
   // and respect the NO_COLOR convention (https://no-color.org).
   const enabled = process.stdout.isTTY && !process.env.NO_COLOR;
   const wrap = (code) => (s) => (enabled ? `\x1b[${code}m${s}\x1b[0m` : s);
   module.exports = {
     green: wrap(32),  // completed tasks
     yellow: wrap(33), // pending
     red: wrap(31),    // overdue / errors
     dim: wrap(2),     // timestamps
   };
   ```
   Use it in `render.js`. The `isTTY` check is the professional touch: don't emit escape codes when output is redirected to a file (`todo list > out.txt`), or you'll pollute it with `\x1b[32m` junk.

5. **`edit` command.** `todo edit <id> "new text"` to change a task's text. Reuse `parseId` and the find-or-`NotFoundError` pattern from `done.js`.

6. **Undo.** Before each `save`, copy `tasks.json` to `tasks.json.bak`. Add `todo undo` that restores the backup. Introduces the idea of write-ahead history — a stepping stone to transactions.

7. **`util.parseArgs` refactor.** Replace the ad-hoc `args.includes('--yes')` checks with one `util.parseArgs({ options: { yes: { type: 'boolean', short: 'y' } } })` call. Built into Node 18.3+, still zero deps.

8. **Real unit tests with `node:test`.** Write `test/cli.test.js` and `test/render.test.js` using the built-in runner: `node --test`. Test `parse()` and the pure render functions (no fs needed), and use a `TODO_FILE` pointed at a temp file for storage tests. Zero dependencies, real coverage.

9. **Make it a global command.** With the `bin` field already in `package.json`, run `npm link` and then invoke `todo add "x"` from anywhere — no `node todo.js` prefix. Understand what `npm link` symlinks and how the shebang line works (next item).

10. **Add a shebang & make it executable.** Put `#!/usr/bin/env node` as the *very first line* of `todo.js`, then `chmod +x todo.js`, and you can run `./todo.js list` directly. This is how every real Node CLI ships.

> **TIP:** Tackle #4 (colors) and #8 (tests) even if you skip the rest — they teach `isTTY`/stream-awareness and the built-in test runner, both of which come up constantly in real Node work.

---

## Common Errors & Fixes

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `SyntaxError: Cannot use import statement outside a module` | Used `import`/`export` while `package.json` has `"type": "commonjs"`. | Use `require`/`module.exports`, **or** switch to ESM (see below) and rename files to `.mjs` or set `"type": "module"`. |
| `Error [ERR_MODULE_NOT_FOUND]` / `Cannot find module './src/cli'` | Wrong relative path, missing `.js` in ESM, or running from the wrong directory. | In CommonJS the extension is optional; in **ESM it's required** (`'./src/cli.js'`). Run from the project root. |
| `tasks file is corrupted (invalid JSON)` | The data file was hand-edited or truncated by a crash. | Fix the JSON, or `rm tasks.json` to start fresh. (This is your own friendly error working as designed.) |
| The program **hangs** and never returns after `clear` | A `readline` interface was opened but `rl.close()` was never called. | Ensure every `createInterface` is matched by `rl.close()` — we close inside the `question` callback. |
| Last line of output **missing** intermittently | Called `process.exit()` before stdout flushed. | Use `process.exitCode = n; return;` instead, as in `todo.js`. |
| `EACCES: permission denied` writing `tasks.json` | The directory isn't writable (e.g. installed system-wide). | Run from a writable dir, fix permissions, or set `TODO_FILE` to a writable path. |
| Multi-word task gets split into several tasks / only first word saved | Forgot quotes **and** the handler didn't join args. | Our `add.js` does `args.join(' ')`, so both `add "buy milk"` and `add buy milk` work. If you changed it, restore the join. |
| `done abc` succeeds with a weird id | Used `parseInt` (lenient) instead of `Number` + `Number.isInteger`. | Validate with the strict approach from `done.js`. |
| `UnhandledPromiseRejection` crash with a stack trace | An `async` function threw and nothing caught it. | Ensure the top-level `main().catch(...)` exists; never call an async function "bare." |
| On Windows, paths look wrong / file not found | Hard-coded `/` separators or string-concatenated paths. | Always build paths with `path.join` / `path.resolve` (as `config.js` does). |
| `TODO_FILE` override seems ignored | Set the env var *after* the command, or in a new shell that didn't inherit it. | Prefix it on the same line: `TODO_FILE=/tmp/t.json node todo.js list`. |

### Appendix: the ESM version of the entry point (for the curious)

The whole course defaults to CommonJS, but you'll meet ES Modules in the wild. Converting is mechanical: set `"type": "module"` in `package.json`, change every `require`/`module.exports` to `import`/`export`, and **add `.js` extensions to all relative imports** (mandatory in ESM). Because ESM has no `__dirname`, reconstruct it from `import.meta.url`:

```js
// todo.js (ES Modules variant — requires "type": "module" in package.json)
import { fileURLToPath } from 'node:url';
import path from 'node:path';
import { parse } from './src/cli.js';          // NOTE: .js extension is REQUIRED in ESM
import commands from './src/commands/index.js'; // and index.js must be spelled out

// Recreate __dirname/__filename, which don't exist in ESM:
const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

async function main() {
  const { command, args } = parse(process.argv);
  const handler = commands[command];
  if (!handler) {
    console.error(`Unknown command: "${command}"`);
    process.exitCode = 2;
    return;
  }
  const output = await handler(args);
  if (output) console.log(output);
}

main().catch((err) => {
  if (err?.isOperational) {
    console.error(`Error: ${err.message}`);
    process.exitCode = err.exitCode || 1;
  } else {
    console.error('Unexpected error:');
    console.error(err?.stack ?? err);
    process.exitCode = 1;
  }
});
```

> **BEST PRACTICE:** Use the `node:` prefix for core modules (`require('node:fs')` / `import fs from 'node:fs'`). It's explicit, future-proof, and guarantees you get the built-in module rather than a same-named package in `node_modules`. The rest of this course will adopt `node:` prefixes once you're comfortable.

---

**You've built a real, robust, dependency-free Node program** — argv parsing, a clean data layer with safe sync/async I/O, JSON persistence, modular structure, typed errors, centralized handling, and interactive prompts. Every one of these reappears, scaled up, in the Express + MongoDB projects ahead. Next project: we put this same skeleton behind an HTTP server and start serving requests.
