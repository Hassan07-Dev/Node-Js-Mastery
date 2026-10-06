# File System (fs) & Path Modules

Working with the filesystem is one of the first things that makes Node.js feel like a *real* backend platform rather than "JavaScript in a browser." In the browser, JavaScript is sandboxed: it cannot read your hard drive, write log files, or list a directory. On the server, your code routinely reads configuration files, writes uploaded images to disk, streams large videos to clients, rotates log files, and inspects directory trees. Node exposes all of this through two tightly related core modules: **`fs`** (filesystem) and **`path`** (path string manipulation).

This module teaches both from first principles. We will treat `fs` not as a grab-bag of functions to memorize, but as a thin, well-designed wrapper over the operating system's syscalls — and once you understand *that*, the API stops feeling arbitrary. We will also spend serious time on `path`, because the single most common source of "works on my machine" filesystem bugs is naive string concatenation of paths.

## What You'll Learn

- The three API styles of the `fs` module (synchronous, callback, and promise-based) and exactly when to reach for each.
- How to read files correctly — encodings, buffers, and the difference between getting a `Buffer` and getting a `string`.
- How to write and append files, and what every file *flag* (`'w'`, `'a'`, `'r+'`, `'wx'`, …) actually means.
- How to inspect file metadata with `stat`/`lstat`, and why `fs.access` is preferred over `existsSync` for permission checks.
- How to create, read, copy, and remove directories — including recursive operations and the modern `withFileTypes` pattern.
- How to rename, move, and delete files, and how to watch the filesystem for changes with `fs.watch`.
- A practical preview of **streaming** large files with `createReadStream`/`createWriteStream` and why streams beat `readFile` for big data.
- The full `path` module toolkit: `join`, `resolve`, `basename`, `dirname`, `extname`, `parse`/`format`, `sep`, `normalize`, and `isAbsolute`.
- The crucial distinction between absolute and relative paths, why `__dirname` matters, and how to write cross-platform path code.
- Robust, production-grade error handling for filesystem operations.

---

## 1. The `fs` Module Overview: Three API Styles

`fs` is a built-in core module — you never `npm install` it, you just require it:

```js
// CommonJS (default in this course)
const fs = require('node:fs'); // synchronous + callback APIs
const fsp = require('node:fs/promises'); // promise-based API

// ES Modules equivalent
// import fs from 'node:fs';
// import fsp from 'node:fs/promises';
```

> TIP: Prefer the `node:` prefix (`require('node:fs')` over `require('fs')`). It makes it unambiguous that you mean the built-in module, prevents a same-named npm package from shadowing it, and is slightly faster to resolve. It works in all current LTS versions.

### Why three styles exist

Node is built around a **single-threaded event loop**. Filesystem operations are *I/O* — they involve the disk, which is thousands of times slower than the CPU. If your single thread sat and waited for the disk on every read, your server could not handle any other requests in the meantime. So Node offers three different ways to do filesystem work, each making a different trade-off between simplicity and concurrency.

| API style | How you call it | Returns | Blocks event loop? | Use when |
|---|---|---|---|---|
| **Synchronous** | `fs.readFileSync(path)` | the value directly (or throws) | **Yes** — blocks everything | CLI scripts, startup config, build tools, tests |
| **Callback** | `fs.readFile(path, cb)` | nothing; result via `cb(err, data)` | No | Legacy code; lowest-level / highest-performance hot paths |
| **Promise** | `await fsp.readFile(path)` | a `Promise` | No | Almost all modern application code (servers) |

A quick taste of all three reading the same file:

```js
const fs = require('node:fs');
const fsp = require('node:fs/promises');

// 1) Synchronous — simplest to read, but BLOCKS the event loop until done.
try {
  const data = fs.readFileSync('config.json', 'utf8');
  console.log('sync:', data);
} catch (err) {
  console.error('sync failed:', err.message);
}

// 2) Callback — the classic Node "error-first callback" pattern.
fs.readFile('config.json', 'utf8', (err, data) => {
  if (err) return console.error('callback failed:', err.message);
  console.log('callback:', data);
});

// 3) Promise — clean async/await, the modern default for app code.
async function readConfig() {
  try {
    const data = await fsp.readFile('config.json', 'utf8');
    console.log('promise:', data);
  } catch (err) {
    console.error('promise failed:', err.message);
  }
}
readConfig();
```

### The golden rule about synchronous APIs

> WARNING — Pitfall: Never use a `*Sync` filesystem function inside a request handler or any code that runs while your server is serving traffic. A single `readFileSync` of a slow or large file freezes the **entire** process — every concurrent user waits, health checks time out, and the event loop stalls. Synchronous calls are fine for one-time work *before* the server starts accepting connections (loading config, reading TLS certs at boot), in CLI tools, and in tests.

The callback API and the promise API are both *non-blocking*: they hand the work to Node's underlying thread pool (libuv) and let the event loop keep running other code. The only difference is ergonomics — promises compose with `async/await` and avoid "callback hell."

> BEST PRACTICE: For all new application code, default to `fs/promises` with `async/await`. Reach for `*Sync` only at process startup or in tooling. Use raw callbacks only when integrating with an old callback-based API or when profiling shows the tiny overhead of promises matters (rare).

---

## 2. Reading Files

### `readFile` family: the basics

The single most important thing to understand about reading files is **what you get back depends on whether you pass an encoding.**

```js
const fsp = require('node:fs/promises');

async function demo() {
  // No encoding → you get a Buffer (raw bytes).
  const buf = await fsp.readFile('notes.txt');
  console.log(buf);                 // <Buffer 48 65 6c 6c 6f ...>
  console.log(Buffer.isBuffer(buf)); // true
  console.log(buf.toString('utf8')); // "Hello ..." — decode bytes to text

  // With an encoding → you get a decoded string directly.
  const str = await fsp.readFile('notes.txt', 'utf8');
  console.log(typeof str);  // "string"
  console.log(str);         // "Hello ..."

  // The encoding can also be passed as an options object:
  const str2 = await fsp.readFile('notes.txt', { encoding: 'utf8' });
}
demo();
```

### Buffers vs. strings — the *why*

A file on disk is just a sequence of **bytes**. A string is a sequence of *characters*. To turn bytes into characters you need a **character encoding** that says "these bytes mean this letter." `'utf8'` is the near-universal default for text. If you do not specify an encoding, Node refuses to guess — it hands you a `Buffer`, Node's built-in type for raw binary data, and lets *you* decide how to interpret it.

This matters because not every file is text:

- A `.txt`, `.json`, `.html`, `.csv` file → text → read with `'utf8'`.
- A `.png`, `.jpg`, `.pdf`, `.zip`, `.mp4` file → binary → read **without** an encoding so you get the raw `Buffer`. Decoding binary as UTF-8 corrupts it.

```js
const fsp = require('node:fs/promises');

async function copyImageBytes() {
  // Read the raw bytes of a binary file — DO NOT pass 'utf8'.
  const imageBytes = await fsp.readFile('logo.png');
  console.log(`Image is ${imageBytes.length} bytes`);
  // We can write those exact bytes elsewhere, unchanged.
  await fsp.writeFile('logo-copy.png', imageBytes);
}
copyImageBytes();
```

> WARNING — Pitfall: Reading a binary file with `'utf8'` and writing it back will silently corrupt it. UTF-8 decoding replaces invalid byte sequences with the replacement character (U+FFFD), so the round-trip is lossy. Only decode bytes you *know* are text.

### Common encodings

| Encoding | Meaning | Typical use |
|---|---|---|
| `'utf8'` | Unicode text (variable-width) | Default for almost all text files |
| `'ascii'` | 7-bit ASCII | Legacy / strictly ASCII data |
| `'latin1'` (`'binary'`) | One byte per char, 0–255 | Some legacy protocols |
| `'base64'` | Base64 text | Encoding binary as a string (e.g., in JSON) |
| `'hex'` | Hexadecimal text | Hashes, debugging bytes |

```js
const fsp = require('node:fs/promises');

async function encodings() {
  const bytes = await fsp.readFile('logo.png');          // Buffer
  const asBase64 = bytes.toString('base64');             // safe to embed in JSON
  const asHex = bytes.toString('hex');                   // e.g. for a hash digest
  console.log(asBase64.slice(0, 40) + '...');
}
encodings();
```

### Reading and parsing JSON — the everyday case

```js
const fsp = require('node:fs/promises');

async function loadJson(path) {
  const text = await fsp.readFile(path, 'utf8'); // get a string
  return JSON.parse(text);                        // turn it into an object
}

loadJson('package.json')
  .then(pkg => console.log('Project name:', pkg.name))
  .catch(err => console.error('Could not load JSON:', err.message));
```

> TIP: There is no built-in `fs.readJson`. The idiomatic pattern is `JSON.parse(await fsp.readFile(path, 'utf8'))`. Note that `JSON.parse` throws a `SyntaxError` on malformed JSON, so a robust loader should catch both the I/O error (file missing) and the parse error.

### Synchronous and callback variants

```js
const fs = require('node:fs');

// Synchronous — returns the value or throws.
const config = JSON.parse(fs.readFileSync('config.json', 'utf8'));

// Callback — error-first.
fs.readFile('config.json', 'utf8', (err, data) => {
  if (err) throw err;
  const config = JSON.parse(data);
});
```

---

## 3. Writing & Appending Files

### `writeFile` — create or overwrite

```js
const fsp = require('node:fs/promises');

async function write() {
  // Writes the string to the file. CREATES the file if missing,
  // OVERWRITES (truncates) it completely if it already exists.
  await fsp.writeFile('greeting.txt', 'Hello, world!\n', 'utf8');

  // You can write a Buffer just as easily (no encoding needed).
  await fsp.writeFile('data.bin', Buffer.from([0xDE, 0xAD, 0xBE, 0xEF]));

  // Writing an object? Serialize it yourself. The 2-space arg pretty-prints.
  const user = { id: 1, name: 'Ada' };
  await fsp.writeFile('user.json', JSON.stringify(user, null, 2));
}
write();
```

> WARNING — Pitfall: `writeFile` **truncates** an existing file. If you meant to add to the file, you wanted `appendFile` (or the `'a'` flag). Accidentally `writeFile`-ing a log file wipes all previous logs.

### `appendFile` — add to the end

```js
const fsp = require('node:fs/promises');

async function log(message) {
  const line = `${new Date().toISOString()} ${message}\n`;
  // Appends to the end. Creates the file if it does not exist yet.
  await fsp.appendFile('app.log', line, 'utf8');
}

log('Server started');
log('Handled request /health');
```

### File flags — the precise control layer

Both `writeFile`/`appendFile` (and `open`) accept a `flag` option that maps directly to the OS `open()` flags. They control whether the file is created, truncated, or whether the operation should fail if the file already exists.

| Flag | Read? | Write? | Creates if missing? | Truncates? | Position | Fails if exists? |
|---|---|---|---|---|---|---|
| `'r'` | ✅ | ❌ | ❌ (throws if missing) | ❌ | start | — |
| `'r+'` | ✅ | ✅ | ❌ (throws if missing) | ❌ | start | — |
| `'w'` | ❌ | ✅ | ✅ | ✅ | start | ❌ |
| `'w+'` | ✅ | ✅ | ✅ | ✅ | start | ❌ |
| `'wx'` | ❌ | ✅ | ✅ | — | start | ✅ (exclusive) |
| `'a'` | ❌ | ✅ | ✅ | ❌ | **end** | ❌ |
| `'a+'` | ✅ | ✅ | ✅ | ❌ | end | ❌ |
| `'ax'` | ❌ | ✅ | ✅ | ❌ | end | ✅ (exclusive) |

The `x` suffix means **exclusive**: the call fails with `EEXIST` if the file already exists. This is the safe way to "create a file only if it doesn't already exist," avoiding a race condition.

```js
const fsp = require('node:fs/promises');

async function createOnce() {
  try {
    // 'wx' = write, but fail if the file already exists.
    await fsp.writeFile('lock.pid', String(process.pid), { flag: 'wx' });
    console.log('Lock acquired.');
  } catch (err) {
    if (err.code === 'EEXIST') {
      console.error('Lock already held by another process.');
    } else {
      throw err;
    }
  }
}
createOnce();

// Append explicitly via flag (equivalent to appendFile):
fsp.writeFile('app.log', 'line\n', { flag: 'a' });
```

> BEST PRACTICE: When you need "create if and only if it doesn't exist" semantics (lock files, idempotent initialization), use the exclusive flag `'wx'`/`'ax'` and handle `EEXIST`. Do **not** do `if (!fs.existsSync(path)) fs.writeFile(...)` — that's a classic **time-of-check-to-time-of-use (TOCTOU) race**: another process can create the file between your check and your write.

---

## 4. File Metadata: `stat`, `lstat`, `access`, and Existence

### `stat` — everything about a file

`fs.stat` returns a `Stats` object describing a path: its size, timestamps, and type (file vs. directory).

```js
const fsp = require('node:fs/promises');

async function inspect(path) {
  const stats = await fsp.stat(path);

  console.log('Is file?     ', stats.isFile());
  console.log('Is directory?', stats.isDirectory());
  console.log('Is symlink?  ', stats.isSymbolicLink()); // see lstat note below
  console.log('Size (bytes):', stats.size);
  console.log('Created:     ', stats.birthtime);
  console.log('Modified:    ', stats.mtime);  // content last modified
  console.log('Accessed:    ', stats.atime);
  console.log('Mode (perms):', (stats.mode & 0o777).toString(8)); // e.g. "644"
}
inspect('package.json');
```

Key `Stats` members you will actually use:

| Member | Meaning |
|---|---|
| `stats.isFile()` | true for regular files |
| `stats.isDirectory()` | true for directories |
| `stats.isSymbolicLink()` | true for symlinks (only meaningful with `lstat`) |
| `stats.size` | size in bytes |
| `stats.mtime` / `stats.mtimeMs` | last content modification (Date / ms number) |
| `stats.birthtime` | creation time (where the OS tracks it) |
| `stats.mode` | permission + type bits; mask with `0o777` for the rwx bits |

### `stat` vs. `lstat`

The difference is **symbolic links**:

- `stat` follows symlinks — it reports on the *target* the link points to.
- `lstat` does **not** follow symlinks — it reports on the *link itself*.

```js
const fsp = require('node:fs/promises');

async function symlinkCheck(p) {
  const s = await fsp.stat(p);   // follows links: isFile() reflects the target
  const l = await fsp.lstat(p);  // does NOT follow: isSymbolicLink() can be true
  console.log('stat says directory?', s.isDirectory());
  console.log('lstat says symlink? ', l.isSymbolicLink());
}
```

> TIP: When you walk a directory tree and want to avoid following symlinks (to prevent infinite loops or escaping a sandbox), use `lstat` and skip entries where `isSymbolicLink()` is true.

### Checking existence and permissions: `fs.access` vs. `existsSync`

There are two questions people conflate:

1. "Does this path exist?"
2. "Can I read/write/execute this path?"

`fs.existsSync(path)` answers (1) — synchronously, returning a boolean, never throwing:

```js
const fs = require('node:fs');
if (fs.existsSync('config.json')) {
  console.log('It exists.');
}
```

`existsSync` is convenient but has two problems. First, it is synchronous (blocks). Second — and more importantly — using it to *guard* a subsequent operation creates a TOCTOU race:

```js
// ANTI-PATTERN — do not do this.
if (fs.existsSync(path)) {
  const data = fs.readFileSync(path); // file may have been deleted in between!
}
```

The robust pattern is to simply **attempt the operation and handle the error**. The file might vanish a microsecond after any check, so the operation itself must handle `ENOENT` anyway.

```js
const fsp = require('node:fs/promises');

async function readIfPresent(path) {
  try {
    return await fsp.readFile(path, 'utf8');
  } catch (err) {
    if (err.code === 'ENOENT') return null; // not found → treat as "no data"
    throw err;                               // anything else is a real problem
  }
}
```

For question (2) — permissions — use `fs.access` / `fsp.access`, which throws if the requested access mode is not available:

```js
const fsp = require('node:fs/promises');
const { constants } = require('node:fs');

async function checkAccess(path) {
  try {
    // F_OK: exists; R_OK: readable; W_OK: writable; X_OK: executable
    await fsp.access(path, constants.R_OK | constants.W_OK);
    console.log('Readable AND writable.');
  } catch {
    console.log('Cannot read+write this path.');
  }
}
checkAccess('config.json');
```

> BEST PRACTICE: Prefer "try the operation, catch the error" over checking-then-acting. If you genuinely need a pre-flight check (e.g., to give a friendly CLI error message before doing a big batch), use `fs.access` rather than `existsSync` — but still handle the error at the point of the real operation, because permissions and existence can change.

---

## 5. Directories

### Creating directories: `mkdir` (and `recursive`)

```js
const fsp = require('node:fs/promises');

async function makeDirs() {
  // Create a single directory. Throws EEXIST if it already exists,
  // and ENOENT if the PARENT does not exist.
  await fsp.mkdir('uploads');

  // recursive:true creates every missing parent ("mkdir -p"),
  // and does NOT throw if the directory already exists.
  await fsp.mkdir('uploads/2026/06/images', { recursive: true });
}
makeDirs();
```

> BEST PRACTICE: For "ensure this directory exists" use `mkdir(path, { recursive: true })`. It is idempotent (safe to call repeatedly) and builds the whole chain. With `recursive: true`, the resolved value is the path of the *first* directory created, or `undefined` if nothing needed creating.

### Reading directories: `readdir` and `withFileTypes`

```js
const fsp = require('node:fs/promises');

async function listSimple() {
  // Returns an array of NAMES (strings) of the directory's entries.
  const names = await fsp.readdir('.');
  console.log(names); // ['package.json', 'src', 'README.md', ...]
}
listSimple();
```

By default you only get names — to know whether each entry is a file or directory, you would have to `stat` each one (an extra syscall per entry). The modern, efficient way is `withFileTypes: true`, which returns **`Dirent`** objects that already know their type:

```js
const fsp = require('node:fs/promises');
const path = require('node:path');

async function listTyped(dir) {
  const entries = await fsp.readdir(dir, { withFileTypes: true });
  for (const entry of entries) {
    const full = path.join(dir, entry.name);
    if (entry.isDirectory()) {
      console.log('DIR  ', full);
    } else if (entry.isFile()) {
      console.log('FILE ', full);
    }
  }
}
listTyped('.');
```

A recursive directory walk built on `withFileTypes`:

```js
const fsp = require('node:fs/promises');
const path = require('node:path');

async function walk(dir) {
  const entries = await fsp.readdir(dir, { withFileTypes: true });
  const files = [];
  for (const entry of entries) {
    const full = path.join(dir, entry.name);
    if (entry.isDirectory()) {
      files.push(...await walk(full)); // recurse into subdirectory
    } else {
      files.push(full);
    }
  }
  return files;
}

walk('src').then(all => console.log(`${all.length} files found`));
```

> TIP: Recent Node versions support `fsp.readdir(dir, { recursive: true })`, which returns every nested entry's relative path in one call — handy for quick scripts. For full control (skipping `node_modules`, following/ignoring symlinks, collecting metadata) a manual `withFileTypes` walk like the one above is still the workhorse.

### Removing directories and files: `rm` and `rmdir`

```js
const fsp = require('node:fs/promises');

async function remove() {
  // Remove an EMPTY directory. Throws ENOTEMPTY if it has contents.
  await fsp.rmdir('empty-folder');

  // The modern, flexible remover — works on files AND directories.
  await fsp.rm('build', { recursive: true, force: true });
  // recursive:true → delete a directory and all its contents ("rm -rf")
  // force:true     → do NOT error if the path doesn't exist (ignore ENOENT)

  // Remove a single file (rm works, but unlink is the classic verb):
  await fsp.rm('temp.txt', { force: true });
}
remove();
```

| Function | Removes | Recursive? | Notes |
|---|---|---|---|
| `fs.unlink` | a single file (or symlink) | n/a | classic POSIX verb for deleting a file |
| `fs.rmdir` | an **empty** directory | only via deprecated option | throws `ENOTEMPTY` on non-empty dirs |
| `fs.rm` | files **or** directories | `{ recursive: true }` | modern, unified; `force: true` ignores missing |

> WARNING — Pitfall: `fs.rmdir(path, { recursive: true })` is **deprecated** for removing non-empty directories. Use `fs.rm(path, { recursive: true, force: true })` instead. And be extremely careful: `rm` with `recursive: true` is the programmatic `rm -rf` — a bad `path` variable (e.g., an empty string resolving to `/`) can delete enormous amounts of data. Validate the path first.

### Copying: `copyFile` and `cp`

```js
const fsp = require('node:fs/promises');
const { constants } = require('node:fs');

async function copy() {
  // Copy a single file (overwrites the destination by default).
  await fsp.copyFile('report.pdf', 'backup/report.pdf');

  // Fail instead of overwriting an existing destination:
  await fsp.copyFile('a.txt', 'b.txt', constants.COPYFILE_EXCL);

  // Copy an entire directory tree (like "cp -r"):
  await fsp.cp('src', 'dist', { recursive: true });
}
copy();
```

> TIP: `fs.cp` (with `recursive: true`) is the directory-tree copier; `fs.copyFile` copies exactly one file. Note that the destination *directory* must already exist for `copyFile` (it does not create parent folders), whereas `cp` will create the target tree.

---

## 6. Renaming, Moving, Deleting, and Watching

### Rename and move: `rename`

There is no separate "move" function — moving *is* renaming. `fs.rename` changes a path's name and/or location:

```js
const fsp = require('node:fs/promises');

async function moveAround() {
  // Rename in place.
  await fsp.rename('draft.txt', 'final.txt');

  // "Move" by renaming to a different directory.
  await fsp.rename('uploads/tmp-123', 'uploads/avatar.png');
}
moveAround();
```

> WARNING — Pitfall: `fs.rename` is atomic and fast — but **only within the same filesystem/volume**. Renaming across devices (e.g., from a temp directory on one disk to a folder on another, common in Docker with mounted volumes) throws `EXDEV`. The portable "move" is: copy then delete the original. A robust mover handles `EXDEV` by falling back:

```js
const fsp = require('node:fs/promises');

async function move(src, dest) {
  try {
    await fsp.rename(src, dest);             // fast path: same volume
  } catch (err) {
    if (err.code !== 'EXDEV') throw err;     // a real error, rethrow
    // Cross-device fallback: copy then remove.
    await fsp.cp(src, dest, { recursive: true });
    await fsp.rm(src, { recursive: true, force: true });
  }
}
```

### Deleting

As covered above: `fsp.unlink(file)` for a single file, `fsp.rm(path, { recursive: true, force: true })` for directories or "delete if exists."

### Watching files and directories: `fs.watch`

`fs.watch` notifies you when a file or directory changes. It is event-based and runs until you close the watcher.

```js
const fs = require('node:fs');

// Watch a directory (recursive is supported on macOS and Windows).
const watcher = fs.watch('src', { recursive: true }, (eventType, filename) => {
  // eventType is 'rename' (created/deleted/renamed) or 'change' (content changed)
  console.log(`${eventType}: ${filename}`);
});

// Stop watching when you're done (e.g., on shutdown).
process.on('SIGINT', () => {
  watcher.close();
  process.exit(0);
});
```

There is also a promise-based async-iterator form:

```js
const fsp = require('node:fs/promises');

async function watchDir() {
  const ac = new AbortController(); // lets us cancel the watch
  setTimeout(() => ac.abort(), 30_000); // stop after 30s, for example

  try {
    const watcher = fsp.watch('src', { recursive: true, signal: ac.signal });
    for await (const event of watcher) {
      console.log(event.eventType, event.filename);
    }
  } catch (err) {
    if (err.name !== 'AbortError') throw err; // abort is expected, not an error
  }
}
watchDir();
```

> WARNING — Pitfall: `fs.watch` is famously inconsistent across platforms. It may fire the `change` event multiple times for a single save, may report `filename` as `null`, and `recursive` is not supported everywhere (notably some Linux setups historically). For anything beyond a quick dev script, use a battle-tested library like **chokidar**, which normalizes these quirks. Use `fs.watch` directly only when you understand the platform behavior or for simple tooling. (The older `fs.watchFile`, which polls `stat` on an interval, is less efficient and generally discouraged in favor of `fs.watch`.)

---

## 7. Streaming Large Files (Preview)

Everything so far loads a whole file into memory. `readFile` of a 2 GB video allocates a 2 GB Buffer — which may crash your process or, on a server, let a few large files exhaust memory. **Streams** solve this by processing data in small chunks (typically 64 KB) as it flows, keeping memory usage flat regardless of file size.

```js
const fs = require('node:fs');

// Read a large file chunk-by-chunk instead of all at once.
const readStream = fs.createReadStream('huge-input.log', { encoding: 'utf8' });

readStream.on('data', (chunk) => {
  // Called repeatedly with a piece of the file. Memory stays low.
  console.log(`Got ${chunk.length} characters`);
});
readStream.on('end', () => console.log('Done reading.'));
readStream.on('error', (err) => console.error('Read error:', err.message));
```

The real power is **piping** a readable stream into a writable one — copying a file of any size with constant memory:

```js
const fs = require('node:fs');
const { pipeline } = require('node:stream/promises');

async function copyLarge(src, dest) {
  // pipeline wires source → destination, handling backpressure and errors,
  // and resolves when the copy finishes.
  await pipeline(
    fs.createReadStream(src),
    fs.createWriteStream(dest)
  );
  console.log('Copied with constant memory.');
}
copyLarge('huge-input.mp4', 'huge-output.mp4');
```

| Approach | Memory use | Time-to-first-byte | Best for |
|---|---|---|---|
| `readFile` / `writeFile` | proportional to file size | after whole file loads | small files (config, JSON) |
| `createReadStream` / `pipeline` | constant (~64 KB chunks) | immediate (streams as it goes) | large files, HTTP responses, transforms |

> BEST PRACTICE: Always use `stream/promises` `pipeline` (or `stream.pipeline`) rather than manual `.pipe()` chains. `pipeline` propagates errors from *every* stage and automatically destroys all streams on failure, preventing file-descriptor and memory leaks. Manual `.pipe()` silently swallows source errors. Streams are covered in depth in the dedicated Streams module — this is just enough to know *when* to reach for them.

---

## 8. The `path` Module

A file path is just a string — but it is a string with rules that differ by operating system. On POSIX (macOS, Linux) the separator is `/`; on Windows it is `\` (and `C:\` style drive letters exist). Hardcoding separators or concatenating with `+` breaks cross-platform. The `path` module manipulates path strings *correctly* for the current platform — without ever touching the disk (it is pure string logic).

```js
const path = require('node:path');
```

### `path.join` — glue segments with the right separator

```js
const path = require('node:path');

// Joins segments using the platform separator and normalizes the result.
path.join('users', 'ada', 'notes.txt'); // POSIX → 'users/ada/notes.txt'
path.join('/var', 'log', '../tmp');      // → '/var/tmp' (the '..' is resolved)
path.join('a//b', 'c');                  // → 'a/b/c' (collapses double slashes)
```

### `path.resolve` — produce an absolute path

`resolve` processes segments right-to-left until it builds an absolute path, prepending the current working directory if needed. Think of it as "where would I end up if I `cd`'d through these?"

```js
const path = require('node:path');

// If CWD is /home/ada:
path.resolve('notes.txt');             // → '/home/ada/notes.txt'
path.resolve('docs', 'notes.txt');     // → '/home/ada/docs/notes.txt'
path.resolve('/etc', 'docs', 'x.txt'); // → '/etc/docs/x.txt' (absolute seg resets)
path.resolve('/a', '/b', 'c');         // → '/b/c' (last absolute wins)
```

**`join` vs. `resolve`** — the key difference: `join` just concatenates and normalizes (result can be relative); `resolve` always returns an absolute path and treats an absolute segment as a fresh start.

| Call | `path.join` | `path.resolve` (CWD = `/home/ada`) |
|---|---|---|
| `('a', 'b')` | `'a/b'` (relative) | `'/home/ada/a/b'` (absolute) |
| `('/a', 'b')` | `'/a/b'` | `'/a/b'` |
| `('a', '/b')` | `'a/b'` | `'/b'` (absolute resets) |
| `('a', '..', 'b')` | `'b'` | `'/home/ada/b'` |

### Decomposing paths: `basename`, `dirname`, `extname`

```js
const path = require('node:path');
const p = '/var/www/site/index.html';

path.basename(p);            // → 'index.html'   (the final segment)
path.basename(p, '.html');   // → 'index'        (strip a known extension)
path.dirname(p);             // → '/var/www/site'(everything but the last segment)
path.extname(p);             // → '.html'        (the extension, with the dot)
path.extname('archive.tar.gz'); // → '.gz'       (only the LAST extension)
path.extname('README');      // → ''             (no extension)
```

### `parse` and `format` — object form

`parse` explodes a path into its components; `format` rebuilds one from an object. They are inverses.

```js
const path = require('node:path');

const parsed = path.parse('/var/www/site/index.html');
// {
//   root: '/',
//   dir:  '/var/www/site',
//   base: 'index.html',
//   ext:  '.html',
//   name: 'index'
// }

// Rebuild, changing the extension to .htm:
const rebuilt = path.format({ ...parsed, base: undefined, ext: '.htm' });
// → '/var/www/site/index.htm'
// NOTE: if 'base' is present, format IGNORES 'ext' and 'name' — so we clear base.
```

> TIP: When using `format`, remember that `base` takes precedence over `name` + `ext`. To change just the extension, either set `base` explicitly to the full filename, or clear `base` (as above) and let `name` + `ext` build it.

### `sep`, `delimiter`, `normalize`, `isAbsolute`

```js
const path = require('node:path');

path.sep;        // '/' on POSIX, '\\' on Windows — the path segment separator
path.delimiter;  // ':' on POSIX, ';' on Windows — separates entries in $PATH

path.normalize('/foo/bar//baz/../qux');  // → '/foo/bar/qux'
path.normalize('a/b/../../c');           // → 'c'

path.isAbsolute('/etc/hosts');  // → true
path.isAbsolute('docs/x.txt');  // → false
path.isAbsolute('C:\\Users');   // → true on Windows
```

`path.relative(from, to)` computes how to get from one path to another — useful for display and for building links:

```js
const path = require('node:path');
path.relative('/data/app/src', '/data/app/test/unit.js');
// → '../test/unit.js'
```

> TIP: If you specifically need POSIX (forward-slash) behavior regardless of platform — for example, generating URLs or normalizing paths stored in a database — use `path.posix.*` (e.g., `path.posix.join(...)`). Likewise `path.win32.*` forces Windows semantics. The bare `path.*` functions follow whatever OS the code runs on.

---

## 9. Absolute vs. Relative Paths & Why `__dirname` Matters

A **relative** path (`./config.json`, `../data`) is interpreted relative to the **current working directory (CWD)** — `process.cwd()`. The critical, surprising fact: **CWD is wherever the user launched `node` from, NOT where your script file lives.**

Consider this layout:

```
/project
  /src
    loadConfig.js   ← uses fs.readFile('config.json')
  config.json
```

If `loadConfig.js` does `fs.readFile('config.json', ...)` and you run:

```bash
cd /project && node src/loadConfig.js   # CWD = /project → looks for /project/config.json ✅
cd /project/src && node loadConfig.js   # CWD = /project/src → looks for /project/src/config.json ❌
```

…the same code finds the file in one case and throws `ENOENT` in the other. The file didn't move — the CWD changed. This is one of the most common beginner bugs.

The fix: build paths relative to the **script's own location**, not the CWD. In CommonJS, every module gets two magic variables:

- `__dirname` — the absolute directory containing the current file.
- `__filename` — the absolute path of the current file itself.

```js
const fs = require('node:fs/promises');
const path = require('node:path');

// Resolve relative to THIS file's directory — stable no matter where node runs.
const configPath = path.join(__dirname, '..', 'config.json'); // /project/config.json

async function loadConfig() {
  const text = await fs.readFile(configPath, 'utf8');
  return JSON.parse(text);
}
```

> BEST PRACTICE: For files that ship *with your code* (templates, default config, bundled data), build the path from `__dirname`. Use the CWD (relative paths or `process.cwd()`) only for files the *user* specifies on the command line — those genuinely should be relative to where they invoked your tool.

### `__dirname` in ES Modules

ES Modules (`"type": "module"` or `.mjs`) do **not** have `__dirname` or `__filename`. Recreate them from `import.meta.url`:

```js
// ESM equivalent of __dirname / __filename
import { fileURLToPath } from 'node:url';
import path from 'node:path';

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

// Newer Node versions also expose import.meta.dirname / import.meta.filename directly:
// const __dirname = import.meta.dirname;
```

### Cross-platform path hygiene

```js
const path = require('node:path');

// ❌ BAD: hardcoded separators break on the other OS.
const bad = __dirname + '/data/' + 'file.txt';

// ✅ GOOD: path.join uses the correct separator everywhere.
const good = path.join(__dirname, 'data', 'file.txt');
```

> WARNING — Pitfall: Never build paths with string concatenation and literal `/` or `\`. Use `path.join`/`path.resolve`. Also avoid hardcoding absolute paths like `/home/me/project` — derive them from `__dirname` or configuration so the code is portable across machines and OSes.

---

## 10. Robust Error Handling for Filesystem Operations

Filesystem calls fail for many *expected* reasons: the file isn't there, you lack permission, the disk is full, a path component isn't a directory. Node surfaces these as errors with a stable **`error.code`** string (mapped from POSIX `errno`). You branch on `error.code`, not on the human-readable `message` (which is not stable and may be localized).

### The error codes you must know

| `error.code` | Meaning | Typical cause |
|---|---|---|
| `ENOENT` | "Error NO ENTry" — no such file or directory | reading/opening a missing path; missing parent on `mkdir` |
| `EEXIST` | file already exists | `mkdir` on existing dir; `'wx'`/`'ax'` flag |
| `EACCES` | permission denied | no read/write/execute right |
| `EPERM` | operation not permitted | privileged op without rights (common on Windows) |
| `EISDIR` | expected a file, got a directory | `readFile` on a directory |
| `ENOTDIR` | expected a directory, got a file | a file used as a path component |
| `ENOTEMPTY` | directory not empty | `rmdir` on a non-empty directory |
| `EMFILE` | too many open files | leaking file descriptors (not closing handles/streams) |
| `EXDEV` | cross-device link | `rename` across filesystems/volumes |
| `ENOSPC` | no space left on device | disk full while writing |

### The canonical pattern

```js
const fsp = require('node:fs/promises');

async function readUserFile(path) {
  try {
    return await fsp.readFile(path, 'utf8');
  } catch (err) {
    switch (err.code) {
      case 'ENOENT':
        // Expected: file simply doesn't exist. Decide what that means here.
        throw new Error(`Config file not found at ${path}`);
      case 'EACCES':
        throw new Error(`Permission denied reading ${path}`);
      case 'EISDIR':
        throw new Error(`Expected a file but ${path} is a directory`);
      default:
        // Unknown / unexpected: don't swallow it — rethrow with context.
        throw err;
    }
  }
}
```

### Principles

1. **Always handle errors.** With `fs/promises`, every `await` can reject — wrap in `try/catch` or attach `.catch`. With callbacks, the first argument is *always* the error; never ignore it. With `*Sync`, wrap in `try/catch`.
2. **Branch on `err.code`, never on `err.message`.** Messages change between versions and locales; codes are stable.
3. **Distinguish expected from unexpected.** `ENOENT` when reading an optional cache file is "normal" — return a default. `ENOSPC` is a genuine emergency — let it bubble up.
4. **Don't swallow errors silently.** An empty `catch {}` hides real bugs (disk full, permission misconfig). At minimum, log; usually, rethrow.
5. **Prefer try/catch over pre-checks.** As covered in §4, checking existence then acting is racy. Attempt the operation and handle failure.
6. **Add context when rethrowing.** `ENOENT` alone is cryptic in a stack trace; "Config file not found at /etc/app/config.json" is actionable. (You can attach `{ cause: err }` to preserve the original.)

> WARNING — Pitfall: A bare `catch (e) {}` around a filesystem call is one of the most damaging anti-patterns in Node. It turns "disk is full and your writes are failing" into silent data loss. If you must ignore an error, ignore *one specific code* and rethrow the rest: `if (err.code !== 'ENOENT') throw err;`.

---

## Common Pitfalls & Gotchas

- **Using `*Sync` in a server request path.** Blocks the entire event loop; every concurrent request stalls. Sync is for startup and scripts only.
- **Reading binary files as `'utf8'`.** Corrupts the bytes. Omit the encoding to get a `Buffer` for images, archives, PDFs, etc.
- **`writeFile` when you meant `appendFile`.** `writeFile` truncates; your previous content is gone.
- **Relative paths that depend on CWD.** `fs.readFile('config.json')` resolves against `process.cwd()`, not the script's folder. Use `path.join(__dirname, ...)` for bundled files.
- **String-concatenating paths with `/`.** Breaks on Windows and can produce `//` or mangled `..`. Always use `path.join`/`path.resolve`.
- **`check-then-act` races (TOCTOU).** `if (existsSync) read()` is racy. Use exclusive flags (`'wx'`) for create-if-absent and try/catch for read.
- **`fs.rmdir` on non-empty dirs / deprecated recursive `rmdir`.** Use `fs.rm(path, { recursive: true, force: true })`.
- **Catastrophic `fs.rm(badPath, { recursive: true })`.** A wrong variable can wipe huge trees. Validate paths; never `rm -rf` a value you didn't construct carefully.
- **`rename` across volumes throws `EXDEV`.** Provide a copy-then-delete fallback for cross-device moves.
- **Trusting `fs.watch` to be consistent.** Duplicate events, `null` filenames, spotty `recursive` support. Use chokidar for anything serious.
- **Leaking file descriptors.** Forgetting to close a `fs.open` handle or not destroying streams on error leads to `EMFILE`. Use `pipeline` and `try/finally` with `handle.close()`.
- **Branching on `err.message` instead of `err.code`.** Messages aren't a stable contract; codes are.
- **Forgetting `__dirname` doesn't exist in ESM.** Recreate it from `import.meta.url` (or use `import.meta.dirname`).

---

## Best Practices

- **Default to `fs/promises` + `async/await`** for application code. Use `*Sync` only at startup or in CLIs/tests; use callbacks only for legacy interop.
- **Use the `node:` prefix** when requiring core modules.
- **Always specify an encoding for text, never for binary.** `'utf8'` for text → string; no encoding for binary → Buffer.
- **Build paths with `path.join`/`path.resolve` and `__dirname`** — never concatenate, never hardcode separators or absolute machine paths.
- **Make operations idempotent and race-safe:** `mkdir(p, { recursive: true })` to ensure dirs; exclusive flags (`'wx'`) for create-once; try/catch instead of check-then-act.
- **Use `fs.rm({ recursive: true, force: true })`** for deletes; validate the path before any recursive removal.
- **Stream large files** with `createReadStream`/`createWriteStream` + `pipeline`; reserve `readFile`/`writeFile` for small files.
- **Handle errors by `err.code`,** distinguish expected (`ENOENT`) from fatal (`ENOSPC`), never silently swallow, and add context when rethrowing.
- **Close what you open.** Use `try/finally` with `FileHandle.close()`, and `pipeline` for streams so descriptors are released on error.
- **Prefer `fs.access` over `existsSync`** if you must pre-check — but still handle the failure at the real operation.
- **Reach for `chokidar`** instead of raw `fs.watch` for production file-watching.

---

## Key Interview Points

- **What are the three `fs` API styles and when do you use each?** Synchronous (`*Sync`, blocks the event loop — startup/CLI only), callback (error-first, non-blocking, legacy/low-level), and promise-based (`fs/promises`, non-blocking, the modern default for app code).
- **Why is `readFileSync` dangerous in a web server?** It blocks the single-threaded event loop, so *all* concurrent requests stall until the disk read completes. Acceptable only before the server starts accepting traffic.
- **What's the difference between getting a Buffer and a string from `readFile`?** No encoding → raw `Buffer` (bytes); pass `'utf8'` (or another encoding) → decoded string. Binary files must be read without an encoding to avoid corruption.
- **`writeFile` vs `appendFile`, and what do file flags do?** `writeFile` truncates/overwrites (`'w'`); `appendFile` adds to the end (`'a'`). Flags control create/truncate/exclusive behavior — e.g. `'wx'` fails if the file exists (race-free create-once).
- **`existsSync` vs `fs.access` — which and why?** Prefer `fs.access` (async, checks permissions) or, better, just attempt the operation and catch the error. `existsSync` invites TOCTOU races and is synchronous.
- **What is a TOCTOU race and how do you avoid it?** Time-of-check-to-time-of-use: the filesystem changes between your check and your action. Avoid by using atomic flags (`'wx'`) and try/catch rather than check-then-act.
- **`stat` vs `lstat`?** `stat` follows symlinks (reports the target); `lstat` does not (reports the link itself, so `isSymbolicLink()` works).
- **How do you remove a non-empty directory?** `fs.rm(path, { recursive: true, force: true })`. The old `fs.rmdir` recursive option is deprecated.
- **Why does `fs.rename` sometimes fail, and what's the fix?** `EXDEV` when moving across filesystems/volumes; fall back to copy-then-delete.
- **`path.join` vs `path.resolve`?** `join` concatenates + normalizes (result may be relative); `resolve` returns an absolute path and treats an absolute segment as a fresh root.
- **Why use `__dirname` instead of relative paths?** Relative paths resolve against the CWD (where `node` was launched), not the script's location, so bundled files break depending on where you run from. `__dirname` is stable. In ESM, derive it from `import.meta.url`.
- **When do you stream instead of `readFile`?** For large files: streams use constant memory (~64 KB chunks) and start immediately, while `readFile` loads the whole file into RAM.
- **How do you robustly handle fs errors?** Branch on `err.code` (`ENOENT`, `EACCES`, `EEXIST`, `EXDEV`, `ENOSPC`, …), distinguish expected from fatal, never swallow silently, add context when rethrowing.

---

## Practice Exercises

Try each before reading the solution. All use `node:fs/promises` and `node:path`. Assume current LTS Node.

1. **Safe JSON reader.** Write `async function readJsonSafe(filePath, fallback)` that returns the parsed JSON object from `filePath`. If the file doesn't exist, return `fallback` instead of throwing. If the file exists but contains invalid JSON, throw a clear error that includes the file path. Resolve `filePath` relative to `__dirname`.

2. **Logger with rotation.** Write `async function appendLog(line)` that appends `` `${ISO timestamp} ${line}\n` `` to `logs/app.log` (creating the `logs` directory if needed). Before appending, if the existing log file is larger than 1 KB, rename it to `app.log.1` (overwriting any previous `app.log.1`) and start a fresh `app.log`.

3. **Typed directory listing.** Write `async function listDir(dir)` that returns an array of `{ name, type, size }` objects for the immediate children of `dir`, where `type` is `'file'`, `'dir'`, or `'other'`, and `size` is the byte size (use `0` for directories). Use `withFileTypes` and avoid an extra `stat` for directories.

4. **Recursive `.js` finder.** Write `async function findJsFiles(root)` that returns absolute paths of every `.js` file under `root` (any depth), **skipping** any `node_modules` directory. Use a `withFileTypes` walk and `path.extname`.

5. **Atomic write.** Write `async function writeFileAtomic(filePath, contents)` that writes `contents` such that readers never see a half-written file: write to a temp file in the same directory, then `rename` it over the destination. Explain in a comment why same-directory matters.

6. **Cross-device-safe move.** Write `async function safeMove(src, dest)` that moves a file or directory, creating `dest`'s parent directory if needed, using fast `rename` when possible and falling back to copy-then-delete on `EXDEV`.

### Solutions

**1. Safe JSON reader**

```js
const fsp = require('node:fs/promises');
const path = require('node:path');

async function readJsonSafe(filePath, fallback) {
  // Resolve relative to THIS file so CWD doesn't matter.
  const abs = path.isAbsolute(filePath)
    ? filePath
    : path.join(__dirname, filePath);

  let text;
  try {
    text = await fsp.readFile(abs, 'utf8');
  } catch (err) {
    if (err.code === 'ENOENT') return fallback; // missing → fallback
    throw err;                                   // permission, etc. → propagate
  }

  try {
    return JSON.parse(text);
  } catch (err) {
    // Parse errors are separate from I/O errors — add the path for clarity.
    throw new Error(`Invalid JSON in ${abs}: ${err.message}`, { cause: err });
  }
}

// Usage:
// const cfg = await readJsonSafe('config.json', { port: 3000 });
```

**2. Logger with rotation**

```js
const fsp = require('node:fs/promises');
const path = require('node:path');

const LOG_DIR = path.join(__dirname, 'logs');
const LOG_FILE = path.join(LOG_DIR, 'app.log');
const ROTATED = path.join(LOG_DIR, 'app.log.1');
const MAX_BYTES = 1024; // 1 KB

async function appendLog(line) {
  await fsp.mkdir(LOG_DIR, { recursive: true }); // ensure dir exists (idempotent)

  // Rotate if the current log is too big.
  try {
    const stats = await fsp.stat(LOG_FILE);
    if (stats.size > MAX_BYTES) {
      // rename overwrites an existing ROTATED on the same volume.
      await fsp.rename(LOG_FILE, ROTATED);
    }
  } catch (err) {
    if (err.code !== 'ENOENT') throw err; // no log yet is fine; anything else isn't
  }

  const entry = `${new Date().toISOString()} ${line}\n`;
  await fsp.appendFile(LOG_FILE, entry, 'utf8'); // appendFile recreates if missing
}

// Usage:
// await appendLog('Server started');
```

**3. Typed directory listing**

```js
const fsp = require('node:fs/promises');
const path = require('node:path');

async function listDir(dir) {
  const entries = await fsp.readdir(dir, { withFileTypes: true });
  const results = [];

  for (const entry of entries) {
    if (entry.isDirectory()) {
      results.push({ name: entry.name, type: 'dir', size: 0 }); // no stat needed
    } else if (entry.isFile()) {
      // Only files need a stat to get a meaningful size.
      const { size } = await fsp.stat(path.join(dir, entry.name));
      results.push({ name: entry.name, type: 'file', size });
    } else {
      results.push({ name: entry.name, type: 'other', size: 0 });
    }
  }
  return results;
}

// Usage:
// console.log(await listDir('.'));
```

**4. Recursive `.js` finder (skipping node_modules)**

```js
const fsp = require('node:fs/promises');
const path = require('node:path');

async function findJsFiles(root) {
  const found = [];

  async function walk(dir) {
    const entries = await fsp.readdir(dir, { withFileTypes: true });
    for (const entry of entries) {
      const full = path.join(dir, entry.name);
      if (entry.isDirectory()) {
        if (entry.name === 'node_modules') continue; // skip dependency trees
        await walk(full);                            // recurse
      } else if (entry.isFile() && path.extname(entry.name) === '.js') {
        found.push(path.resolve(full)); // store absolute paths
      }
    }
  }

  await walk(root);
  return found;
}

// Usage:
// console.log(await findJsFiles('src'));
```

**5. Atomic write**

```js
const fsp = require('node:fs/promises');
const path = require('node:path');
const crypto = require('node:crypto');

async function writeFileAtomic(filePath, contents) {
  const dir = path.dirname(filePath);
  // Temp file in the SAME directory → guarantees the same filesystem/volume,
  // so the final rename is atomic (no EXDEV, no cross-device copy).
  const tmp = path.join(dir, `.${path.basename(filePath)}.${crypto.randomUUID()}.tmp`);

  try {
    await fsp.writeFile(tmp, contents);      // fully write the temp file
    await fsp.rename(tmp, filePath);         // atomic swap into place
  } catch (err) {
    // Clean up the temp file on failure; ignore if it isn't there.
    await fsp.rm(tmp, { force: true }).catch(() => {});
    throw err;
  }
}

// Usage:
// await writeFileAtomic('config.json', JSON.stringify(cfg, null, 2));
```

**6. Cross-device-safe move**

```js
const fsp = require('node:fs/promises');
const path = require('node:path');

async function safeMove(src, dest) {
  await fsp.mkdir(path.dirname(dest), { recursive: true }); // ensure target dir

  try {
    await fsp.rename(src, dest);               // fast path: same volume, atomic
  } catch (err) {
    if (err.code !== 'EXDEV') throw err;       // a different failure → propagate
    // Cross-device fallback: copy the tree, then remove the original.
    await fsp.cp(src, dest, { recursive: true });
    await fsp.rm(src, { recursive: true, force: true });
  }
}

// Usage:
// await safeMove('uploads/tmp-abc', 'storage/2026/file.png');
```

---

## Summary / Cheat Sheet

| Task | Modern API |
|---|---|
| Require modules | `const fsp = require('node:fs/promises')`, `const path = require('node:path')` |
| Read text | `await fsp.readFile(p, 'utf8')` |
| Read binary | `await fsp.readFile(p)` → `Buffer` |
| Read JSON | `JSON.parse(await fsp.readFile(p, 'utf8'))` |
| Write (overwrite) | `await fsp.writeFile(p, data)` |
| Append | `await fsp.appendFile(p, data)` |
| Create-once (no overwrite) | `fsp.writeFile(p, data, { flag: 'wx' })` → handle `EEXIST` |
| File info | `const s = await fsp.stat(p)` → `s.isFile()`, `s.size`, `s.mtime` |
| Symlink-aware info | `await fsp.lstat(p)` → `s.isSymbolicLink()` |
| Check permission | `await fsp.access(p, constants.R_OK)` |
| Make dir (+parents) | `await fsp.mkdir(p, { recursive: true })` |
| List dir (typed) | `await fsp.readdir(p, { withFileTypes: true })` → `Dirent` |
| Remove file/dir | `await fsp.rm(p, { recursive: true, force: true })` |
| Copy file | `await fsp.copyFile(src, dest)` |
| Copy tree | `await fsp.cp(src, dest, { recursive: true })` |
| Rename / move | `await fsp.rename(src, dest)` (fallback on `EXDEV`) |
| Watch | `fs.watch(dir, { recursive: true }, cb)` (or chokidar) |
| Stream big files | `pipeline(createReadStream(src), createWriteStream(dest))` |

**`path` quick reference**

| Call | Result (POSIX example) |
|---|---|
| `path.join('a', 'b', 'c.txt')` | `'a/b/c.txt'` |
| `path.resolve('a', 'b')` | `'<cwd>/a/b'` (absolute) |
| `path.basename('/x/y.txt')` | `'y.txt'` |
| `path.dirname('/x/y.txt')` | `'/x'` |
| `path.extname('y.tar.gz')` | `'.gz'` |
| `path.parse('/x/y.txt')` | `{ root, dir, base, name, ext }` |
| `path.sep` / `path.delimiter` | `'/'` / `':'` |
| `path.isAbsolute('/x')` | `true` |
| `path.join(__dirname, 'config.json')` | path stable across any CWD |

**Error codes to memorize:** `ENOENT` (missing), `EEXIST` (exists), `EACCES`/`EPERM` (permission), `EISDIR`/`ENOTDIR` (wrong type), `ENOTEMPTY` (dir not empty), `EXDEV` (cross-device rename), `EMFILE` (too many open files), `ENOSPC` (disk full). Branch on `err.code`, never `err.message`.

**Three rules to never forget:**
1. `*Sync` blocks the event loop — startup and scripts only, never in a live server.
2. Build paths from `__dirname` with `path.join` — relative paths follow the CWD, not your file.
3. Handle filesystem errors by `code`, distinguish expected from fatal, and never swallow them silently.
