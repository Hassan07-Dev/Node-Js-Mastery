# Essential Core Modules (os, util, crypto, url & more)

Node.js ships with a rich **standard library** of built-in ("core") modules. You do not install them from npm — they come bundled with the runtime, are loaded synchronously and instantly, and are versioned and maintained alongside Node itself. Reaching for a core module instead of a third-party package means fewer dependencies, smaller `node_modules`, fewer supply-chain risks, and APIs that are battle-tested by millions of programs.

This module is a deep tour of the core modules you will use constantly in real backend work but that rarely get a dedicated lesson: `os`, `util`, `crypto`, the URL/`url`/`querystring` family, `assert`, `timers/promises`, `perf_hooks`, `readline`, and `zlib`. We will also point ahead to `child_process`, which is covered deeply in a later module.

> TIP: Since Node 16, every built-in module is also available under the `node:` prefix — e.g. `require('node:os')` or `import os from 'node:os'`. The prefix makes it unambiguous that you mean the built-in (never a same-named npm package), and a few newer modules (like `node:test`) are **only** importable with the prefix. Prefer the `node:` prefix in new code. We use it throughout this module.

## What You'll Learn

- How to read host, CPU, memory, and network information with **`os`**, and why you should treat it as machine metadata, not application config.
- How **`util`** smooths over Node's history: `promisify`/`callbackify` to bridge callback and Promise worlds, `util.inspect`/`util.format` for debugging output, `util.types` for reliable type checks, and `util.deprecate` for graceful API sunsetting.
- Practical, security-correct **`crypto`**: hashing with `createHash`, message authentication with `createHmac`, cryptographically secure randomness (`randomBytes`, `randomUUID`), password hashing with `scrypt`/`pbkdf2`, and symmetric encryption with `createCipheriv`/`createDecipheriv`.
- The modern **WHATWG `URL`** class and `URLSearchParams`, the **legacy `url`** module, and the older **`querystring`** module — when each applies.
- Lightweight sanity checks and test assertions with **`assert`**.
- Promise-based timers via **`timers/promises`** (no more wrapping `setTimeout` by hand).
- Measuring code performance accurately with **`perf_hooks`**.
- Building interactive command-line tools with **`readline`**.
- Compressing and decompressing data with **`zlib`** (gzip/deflate), plus a forward pointer to **`child_process`**.

---

## The `os` Module — Machine & Operating-System Information

The `os` module exposes information about the operating system and the machine Node is running on. The **why**: backend services frequently need to adapt to their environment — size a worker pool to the number of CPU cores, report free memory in a health check, find the right temp directory, choose the platform-correct line ending, or log the hostname so you can tell which instance produced a log line.

> WARNING — Pitfall: `os` reports facts about the *machine*, not about your *application*. Do not use it as a configuration mechanism. Things like which port to bind or which database URL to use belong in environment variables or a config file, never derived from `os` data.

### Core functions

```js
// os-info.js
const os = require('node:os');

console.log('Platform:     ', os.platform());   // 'darwin' | 'linux' | 'win32' | 'aix' | 'freebsd' | 'sunos'
console.log('Architecture: ', os.arch());       // 'x64' | 'arm64' | 'arm' | 'ia32' ...
console.log('OS type:      ', os.type());        // 'Linux' | 'Darwin' | 'Windows_NT'
console.log('OS release:   ', os.release());     // kernel/OS version string
console.log('Hostname:     ', os.hostname());    // machine network name
console.log('Home dir:     ', os.homedir());     // current user's home directory
console.log('Temp dir:     ', os.tmpdir());      // OS temp directory (use for scratch files)
console.log('Uptime (s):   ', os.uptime());      // seconds since the OS booted
console.log('EOL:          ', JSON.stringify(os.EOL)); // '\n' on POSIX, '\r\n' on Windows
```

`os.platform()` returns the value of the compile-time `process.platform`; `os.arch()` mirrors `process.arch`. They are convenient when you already have `os` imported, but for a single check `process.platform` avoids an import.

### CPUs

`os.cpus()` returns an array with one entry per logical CPU (core/hyperthread). Each entry has the model string, speed in MHz, and a `times` object with cumulative tick counts for each CPU state.

```js
// cpus.js
const os = require('node:os');

const cpus = os.cpus();
console.log(`Logical cores: ${cpus.length}`);
console.log(`Model:         ${cpus[0].model}`);
console.log(`Speed (MHz):   ${cpus[0].speed}`);
console.log(cpus[0].times);
// { user, nice, sys, idle, irq }  -- cumulative milliseconds spent in each mode
```

The classic use of `os.cpus().length` is sizing a process pool — for example, deciding how many workers the `cluster` module or a worker pool should spawn.

```js
// pool-size.js
const os = require('node:os');

// One worker per logical core is a common default for CPU-bound work.
const workerCount = os.cpus().length;
console.log(`Spawning ${workerCount} workers`);
```

> WARNING — Pitfall: In a container (Docker/Kubernetes) `os.cpus().length` reports the **host's** core count, not the cgroup CPU limit assigned to the container. A 2-core-limited pod on a 64-core node will see 64 and badly over-spawn workers. Node added `os.availableParallelism()` (Node 19.4+, backported to 18.14+) which respects cgroup limits, `sched_setaffinity`, and `UV_THREADPOOL_SIZE` hints. **Prefer `os.availableParallelism()` for pool sizing in containerized environments.**

```js
// parallelism.js
const os = require('node:os');

// Container-aware. Falls back to logical core count when no limit is set.
const n = os.availableParallelism();
console.log(`Recommended parallelism: ${n}`);
```

To compute a CPU **usage percentage** you must sample `times` twice and compare deltas — a single reading only gives cumulative totals since boot:

```js
// cpu-usage.js
const os = require('node:os');

function snapshot() {
  // Sum idle and total ticks across all cores.
  let idle = 0;
  let total = 0;
  for (const cpu of os.cpus()) {
    for (const t of Object.values(cpu.times)) total += t;
    idle += cpu.times.idle;
  }
  return { idle, total };
}

const start = snapshot();
setTimeout(() => {
  const end = snapshot();
  const idleDelta = end.idle - start.idle;
  const totalDelta = end.total - start.total;
  const usage = 100 - (100 * idleDelta) / totalDelta; // percent busy over the window
  console.log(`CPU usage: ${usage.toFixed(1)}%`);
}, 1000);
```

### Memory

```js
// memory.js
const os = require('node:os');

const toGB = (bytes) => (bytes / 1024 ** 3).toFixed(2);

console.log(`Total memory: ${toGB(os.totalmem())} GB`);
console.log(`Free memory:  ${toGB(os.freemem())} GB`);
console.log(`Used:         ${(100 * (1 - os.freemem() / os.totalmem())).toFixed(1)}%`);

// System load averages over 1, 5, and 15 minutes (POSIX only; [0,0,0] on Windows).
console.log('Load avg:     ', os.loadavg());
```

> WARNING — Pitfall: `os.totalmem()`/`os.freemem()` report **system-wide** memory, again ignoring container limits. To see how much memory *your Node process* is using, use `process.memoryUsage()` (heap, RSS, external, arrayBuffers) instead.

### User and network information

```js
// user-network.js
const os = require('node:os');

console.log(os.userInfo());
// { username, uid, gid, shell, homedir }  (uid/gid are -1 on Windows)

const nets = os.networkInterfaces();
for (const [name, addrs] of Object.entries(nets)) {
  for (const addr of addrs) {
    // Skip internal (loopback) and non-IPv4 to find the LAN address.
    if (addr.family === 'IPv4' && !addr.internal) {
      console.log(`${name}: ${addr.address}`);
    }
  }
}
```

> TIP: In Node 18+, `addr.family` is the string `'IPv4'`/`'IPv6'`. In older versions it could be the number `4`/`6`. If you support old runtimes, check both.

### `os` quick reference

| Function | Returns | Common use |
| --- | --- | --- |
| `os.platform()` | `'linux'`, `'darwin'`, `'win32'`… | Branch on OS |
| `os.arch()` | `'x64'`, `'arm64'`… | Branch on CPU architecture |
| `os.cpus()` | Array of core info | CPU stats; legacy pool sizing |
| `os.availableParallelism()` | Number | **Container-aware** pool sizing |
| `os.totalmem()` / `os.freemem()` | Bytes | System memory metrics |
| `os.hostname()` | String | Tag logs/metrics by instance |
| `os.homedir()` | String | Locate user config files |
| `os.tmpdir()` | String | Scratch/temp file location |
| `os.networkInterfaces()` | Object | Discover local IP addresses |
| `os.EOL` | `'\n'` or `'\r\n'` | Platform-correct line endings |
| `os.loadavg()` | `[1m, 5m, 15m]` | System load (POSIX) |
| `os.userInfo()` | Object | Current user details |

---

## The `util` Module — Glue and Diagnostics

`util` is a grab-bag of utilities, many of which exist for historical reasons: Node predates Promises and `async/await`, so it was originally an all-callback platform. `util` provides the bridges between that callback past and the Promise-based present, plus first-class tools for inspecting and formatting values.

### `util.promisify` — callback APIs to Promises

Node's classic asynchronous APIs use the **error-first callback** convention: the last argument is `callback(err, result)`. `util.promisify` wraps such a function and returns a new one that omits the callback and returns a Promise instead.

```js
// promisify.js
const util = require('node:util');
const fs = require('node:fs');

// fs.readFile uses (path, options, callback). Promisify it:
const readFile = util.promisify(fs.readFile);

async function main() {
  const text = await readFile(__filename, 'utf8'); // now awaitable
  console.log(`This file is ${text.length} characters long.`);
}
main().catch(console.error);
```

The function being promisified must follow the error-first convention with the callback **last**. The Promise resolves with the callback's `result` and rejects with its `err`.

> BEST PRACTICE: Before reaching for `util.promisify`, check whether the module already offers a Promise-based variant. `fs` has `fs.promises` (a.k.a. `require('node:fs/promises')`), and many modules expose promisified APIs natively. Use those first; reserve `promisify` for older or third-party callback APIs that have no Promise version.

#### The `promisify.custom` symbol

Some functions don't follow the plain error-first shape — for instance a callback that yields multiple values. A library can attach a custom promisified implementation via the `util.promisify.custom` symbol, and `promisify` will use it:

```js
// promisify-custom.js
const util = require('node:util');

function legacyDivmod(a, b, cb) {
  // Calls back with TWO success values, which the default promisify can't handle.
  cb(null, Math.floor(a / b), a % b);
}

// Teach promisify how to handle it:
legacyDivmod[util.promisify.custom] = (a, b) =>
  new Promise((resolve, reject) => {
    legacyDivmod(a, b, (err, quotient, remainder) => {
      if (err) return reject(err);
      resolve({ quotient, remainder }); // bundle into one object
    });
  });

(async () => {
  console.log(await util.promisify(legacyDivmod)(17, 5)); // { quotient: 3, remainder: 2 }
})();
```

### `util.callbackify` — Promises back to callbacks

The inverse: take an `async`/Promise-returning function and expose it with an error-first callback. Useful when you must integrate modern code into an older callback-only API or framework.

```js
// callbackify.js
const util = require('node:util');

async function fetchUser(id) {
  if (id <= 0) throw new Error('invalid id');
  return { id, name: 'Ada' };
}

const fetchUserCb = util.callbackify(fetchUser);

fetchUserCb(42, (err, user) => {
  if (err) return console.error('Failed:', err.message);
  console.log('Got:', user);
});

// If the async fn throws a non-Error (e.g. rejects with null/undefined),
// callbackify wraps it so the first callback arg is still a real Error
// with the original value on err.reason.
```

### `util.inspect` — turn any value into a readable string

`console.log` uses `util.inspect` under the hood to render objects. Calling it directly gives you control over depth, colors, and more — invaluable for debugging deeply nested structures that `console.log` truncates.

```js
// inspect.js
const util = require('node:util');

const data = {
  user: { id: 1, roles: ['admin', 'editor'] },
  nested: { a: { b: { c: { d: { e: 'deep' } } } } },
  secret: 'do-not-log',
  created: new Date(),
  pattern: /ab+c/gi,
};

// Default depth is 2 — deeper objects show as [Object].
console.log(util.inspect(data));

// Override options for full visibility:
console.log(
  util.inspect(data, {
    depth: null,       // recurse with no depth limit (null = infinite)
    colors: true,      // ANSI colors (great in a terminal)
    compact: false,    // one property per line
    maxArrayLength: 100,
    maxStringLength: 80,
    breakLength: 80,
  })
);
```

You can also customize how *your* objects render in logs by defining a `[util.inspect.custom]` method:

```js
// inspect-custom.js
const util = require('node:util');

class Money {
  constructor(cents) { this.cents = cents; }
  // Controls what console.log / util.inspect shows.
  [util.inspect.custom](depth, options) {
    return `Money<$${(this.cents / 100).toFixed(2)}>`;
  }
}

console.log(new Money(12345)); // Money<$123.45>
```

> TIP: `util.inspect.defaultOptions.depth = null;` set once at startup makes every `console.log` in your process recurse fully. Handy for debugging, but noisy in production logs.

### `util.format` — printf-style strings

`util.format` is what powers `console.log`'s placeholder substitution. It supports `%s` (string), `%d`/`%i` (integer), `%f` (float), `%j` (JSON), `%o`/`%O` (object via inspect), and `%%` (literal percent).

```js
// format.js
const util = require('node:util');

console.log(util.format('%s is %d years old', 'Ada', 36));     // Ada is 36 years old
console.log(util.format('payload=%j', { ok: true }));          // payload={"ok":true}
console.log(util.format('obj=%o', { a: 1, b: [2, 3] }));       // obj={ a: 1, b: [ 2, 3 ] }
console.log(util.format('100%% done'));                        // 100% done

// Extra args with no placeholder are appended, space-separated:
console.log(util.format('values:', 1, 2, 3));                  // values: 1 2 3
```

> TIP: For string building in application code, template literals are clearer than `util.format`. Reach for `format` when you need printf semantics — e.g. implementing your own logger that mirrors `console.log`'s behavior.

### `util.types` — reliable runtime type checks

JavaScript's `typeof` and `instanceof` are unreliable across realms (e.g. values passed from a Worker, a VM context, or a different `<iframe>`), where `instanceof Date` can be `false` for a genuine `Date`. `util.types` inspects the internal type tag, so it works regardless of realm.

```js
// types.js
const util = require('node:util');

console.log(util.types.isDate(new Date()));            // true
console.log(util.types.isRegExp(/x/));                  // true
console.log(util.types.isPromise(Promise.resolve()));  // true
console.log(util.types.isAsyncFunction(async () => {})); // true
console.log(util.types.isTypedArray(new Uint8Array())); // true
console.log(util.types.isArrayBuffer(new ArrayBuffer(8))); // true
console.log(util.types.isNativeError(new TypeError())); // true even across realms
console.log(util.types.isProxy(new Proxy({}, {})));     // true
```

### `util.deprecate` — sunset an API gracefully

When you maintain a library and want to retire a function without breaking callers immediately, wrap it with `util.deprecate`. The wrapper prints a deprecation warning **once** (per code path) to `stderr`, then delegates to the original.

```js
// deprecate.js
const util = require('node:util');

function oldApi(x) {
  return x * 2;
}

const oldApiDeprecated = util.deprecate(
  oldApi,
  'oldApi() is deprecated. Use newApi() instead.',
  'DEP0001' // optional stable deprecation code
);

console.log(oldApiDeprecated(21));
// On stderr (once): (node:1234) [DEP0001] DeprecationWarning: oldApi() is deprecated. Use newApi() instead.
```

> TIP: Users can run with `--throw-deprecation` to turn deprecation warnings into thrown errors (great in CI to catch deprecated usage), `--no-deprecation` to silence them, or `--trace-deprecation` to print a stack trace showing where the deprecated call originated.

### Other handy `util` members

- `util.parseArgs({ options })` (Node 18.3+, stable in 20+) — a built-in CLI argument parser, removing the need for `minimist`/`yargs` for simple cases.
- `util.inherits(ctor, superCtor)` — legacy prototypal inheritance helper; prefer ES `class extends` in new code.
- `util.isDeepStrictEqual(a, b)` — deep equality used by `assert.deepStrictEqual`.

```js
// parseargs.js
const { parseArgs } = require('node:util');

// Example argv: node parseargs.js --name Ada --verbose -- file.txt
const { values, positionals } = parseArgs({
  options: {
    name: { type: 'string', short: 'n' },
    verbose: { type: 'boolean', short: 'v', default: false },
  },
  allowPositionals: true,
});

console.log(values);      // { verbose: true, name: 'Ada' }
console.log(positionals); // [ 'file.txt' ]
```

---

## The `crypto` Module — Hashing, HMAC, Randomness, and Encryption

`crypto` wraps OpenSSL to provide cryptographic primitives. The **why**: backends constantly need to verify data integrity, authenticate messages, generate unguessable tokens, store passwords safely, and encrypt sensitive data at rest.

> WARNING — Pitfall: Cryptography is unforgiving — small mistakes (reusing an IV, using a fast hash for passwords, comparing secrets with `===`) create real vulnerabilities. Learn the primitives below, but for anything user-facing (sessions, tokens, password reset) prefer well-audited libraries (`jsonwebtoken`, `bcrypt`/`argon2`, Passport) that encode the correct patterns.

### Hashing with `createHash`

A cryptographic hash is a one-way function mapping arbitrary input to a fixed-size digest. Use it for **integrity checks**, content addressing, and ETags — *not* for passwords (see below) and *not* for authentication of untrusted data (use HMAC for that).

```js
// hash.js
const crypto = require('node:crypto');

// One-shot style: update once, then digest.
const digest = crypto.createHash('sha256')
  .update('hello world')   // accepts strings or Buffers
  .digest('hex');          // 'hex' | 'base64' | 'base64url' | Buffer if omitted
console.log(digest); // b94d27b9934d3e08a52e52d7da7dabfac484efe37a5380ee9088f7ace2efcde9

// You can call update() multiple times for streaming/chunked input.
const h = crypto.createHash('sha256');
h.update('hello ');
h.update('world');
console.log(h.digest('hex')); // same as above
```

To hash a large file without loading it into memory, stream it through the hash:

```js
// hash-file.js
const crypto = require('node:crypto');
const fs = require('node:fs');

function hashFile(path) {
  return new Promise((resolve, reject) => {
    const hash = crypto.createHash('sha256');
    const stream = fs.createReadStream(path);
    stream.on('error', reject);
    stream.on('data', (chunk) => hash.update(chunk));
    stream.on('end', () => resolve(hash.digest('hex')));
  });
}

hashFile(__filename).then(console.log);
```

> TIP: List the hash algorithms your OpenSSL build supports with `crypto.getHashes()`. Avoid `md5` and `sha1` for security purposes — they are broken for collision resistance. Use `sha256` or stronger.

### Message authentication with `createHmac`

An HMAC (Hash-based Message Authentication Code) combines a hash with a secret key. Unlike a plain hash, an HMAC proves a message came from someone holding the key and was not tampered with. Use it for webhook signature verification (Stripe, GitHub), signed cookies, and API request signing.

```js
// hmac.js
const crypto = require('node:crypto');

const secret = 'shared-secret-key';
const payload = '{"event":"payment.succeeded"}';

const signature = crypto.createHmac('sha256', secret)
  .update(payload)
  .digest('hex');

console.log('Signature:', signature);

// Verifying an incoming webhook — compare in CONSTANT TIME (see pitfall below).
function verify(payload, receivedSig) {
  const expected = crypto.createHmac('sha256', secret).update(payload).digest();
  const received = Buffer.from(receivedSig, 'hex');
  // Lengths must match for timingSafeEqual; guard first.
  return (
    expected.length === received.length &&
    crypto.timingSafeEqual(expected, received)
  );
}

console.log(verify(payload, signature)); // true
```

> WARNING — Pitfall: Never compare secrets, tokens, or signatures with `===` or `==`. String comparison short-circuits at the first differing character, leaking timing information an attacker can exploit to forge a valid signature byte by byte. Always use `crypto.timingSafeEqual(bufA, bufB)`, which compares in constant time. It requires both buffers to be the **same length**, so check lengths first (a length mismatch is itself a failed comparison).

### Secure randomness: `randomBytes` and `randomUUID`

`Math.random()` is **not** cryptographically secure and must never be used for tokens, passwords, salts, or session IDs — its output is predictable. Use `crypto.randomBytes` (CSPRNG) instead.

```js
// random.js
const crypto = require('node:crypto');

// Synchronous (fine at startup); pass a callback for the async version.
const token = crypto.randomBytes(32).toString('hex');     // 64 hex chars
const urlSafe = crypto.randomBytes(32).toString('base64url'); // URL-safe token
console.log('Hex token:    ', token);
console.log('URL-safe token:', urlSafe);

// Async form avoids blocking the event loop for large/byte-heavy requests:
crypto.randomBytes(16, (err, buf) => {
  if (err) throw err;
  console.log('Async bytes:', buf.toString('hex'));
});

// A cryptographically strong unbiased integer in [0, max):
console.log('Dice roll:', crypto.randomInt(1, 7)); // 1..6 inclusive

// RFC 4122 v4 UUID — perfect for request IDs, idempotency keys, etc.
console.log('UUID:', crypto.randomUUID());
```

> BEST PRACTICE: For unique identifiers prefer `crypto.randomUUID()` (built in, no dependency). For opaque secret tokens (password-reset links, API keys) use `crypto.randomBytes(32).toString('base64url')` — 256 bits of entropy, URL-safe.

### Password hashing: `scrypt` and `pbkdf2` (concept)

Passwords need **slow, salted** hashing — the opposite of fast file hashing. The goal is to make brute-forcing each guess expensive. A fast hash like SHA-256 can be computed billions of times per second on a GPU; a purpose-built **key derivation function (KDF)** like `scrypt`, `pbkdf2`, `bcrypt`, or `argon2` is deliberately slow and (for scrypt/argon2) memory-hard.

Two essential ideas:

1. **Salt** — a unique random value per password, stored alongside the hash. It ensures identical passwords produce different hashes and defeats precomputed rainbow tables.
2. **Work factor** — a tunable cost (iterations / memory) you raise over time as hardware gets faster.

Node ships `crypto.scrypt` (recommended over pbkdf2 for new code because it is memory-hard, which resists GPU/ASIC attacks):

```js
// password-scrypt.js
const crypto = require('node:crypto');
const { promisify } = require('node:util');

const scrypt = promisify(crypto.scrypt);

async function hashPassword(password) {
  const salt = crypto.randomBytes(16);                 // unique per user
  const derivedKey = await scrypt(password, salt, 64); // 64-byte key
  // Store salt and hash together so you can verify later.
  return `${salt.toString('hex')}:${derivedKey.toString('hex')}`;
}

async function verifyPassword(password, stored) {
  const [saltHex, keyHex] = stored.split(':');
  const salt = Buffer.from(saltHex, 'hex');
  const keyBuf = Buffer.from(keyHex, 'hex');
  const derivedKey = await scrypt(password, salt, keyBuf.length);
  // Constant-time comparison — never use === here.
  return crypto.timingSafeEqual(keyBuf, derivedKey);
}

(async () => {
  const stored = await hashPassword('correct horse battery staple');
  console.log('Stored:', stored);
  console.log('Valid:  ', await verifyPassword('correct horse battery staple', stored)); // true
  console.log('Invalid:', await verifyPassword('wrong', stored));                        // false
})();
```

The `pbkdf2` equivalent (still acceptable, and FIPS-compliant) tunes cost via iteration count:

```js
// password-pbkdf2.js
const crypto = require('node:crypto');
const { promisify } = require('node:util');

const pbkdf2 = promisify(crypto.pbkdf2);

async function hash(password) {
  const salt = crypto.randomBytes(16);
  const iterations = 600_000;          // OWASP-style guidance for PBKDF2-HMAC-SHA256
  const keylen = 32;
  const digest = 'sha256';
  const key = await pbkdf2(password, salt, iterations, keylen, digest);
  return [salt.toString('hex'), iterations, key.toString('hex')].join(':');
}

hash('hunter2').then(console.log);
```

> BEST PRACTICE: For production password storage, prefer the maintained npm packages **`argon2`** (modern winner of the Password Hashing Competition) or **`bcrypt`**. They package safe defaults and encode the parameters into the hash string for you. Use Node's built-in `scrypt`/`pbkdf2` when you cannot add a native dependency. **Never** store passwords with a plain SHA hash, salted or not.

### Symmetric encryption: `createCipheriv` / `createDecipheriv`

Symmetric encryption uses the same key to encrypt and decrypt — for data at rest (encrypting a field before saving to the DB) or in transit within your own systems. The modern recommended mode is **AES-256-GCM**, which is *authenticated*: it produces an auth tag that detects tampering on decrypt.

The `iv` in `createCipheriv` stands for **initialization vector** — a per-message random value that ensures encrypting the same plaintext twice yields different ciphertext.

```js
// encrypt-gcm.js
const crypto = require('node:crypto');

const ALGO = 'aes-256-gcm';

function encrypt(plaintext, key /* 32-byte Buffer */) {
  const iv = crypto.randomBytes(12);                 // 96-bit IV recommended for GCM
  const cipher = crypto.createCipheriv(ALGO, key, iv);
  const ciphertext = Buffer.concat([
    cipher.update(plaintext, 'utf8'),
    cipher.final(),
  ]);
  const authTag = cipher.getAuthTag();               // integrity/authenticity tag
  // Bundle IV + tag + ciphertext so decrypt has everything it needs.
  return Buffer.concat([iv, authTag, ciphertext]).toString('base64');
}

function decrypt(payloadB64, key) {
  const data = Buffer.from(payloadB64, 'base64');
  const iv = data.subarray(0, 12);
  const authTag = data.subarray(12, 28);             // GCM tag is 16 bytes
  const ciphertext = data.subarray(28);
  const decipher = crypto.createDecipheriv(ALGO, key, iv);
  decipher.setAuthTag(authTag);                      // must set before final()
  // .final() THROWS if the tag doesn't match (tampering detected).
  return Buffer.concat([decipher.update(ciphertext), decipher.final()]).toString('utf8');
}

// Derive a 32-byte key from a passphrase (or load from a secret manager / env).
const key = crypto.scryptSync('a strong passphrase', 'app-static-salt', 32);

const blob = encrypt('top secret data', key);
console.log('Encrypted:', blob);
console.log('Decrypted:', decrypt(blob, key)); // 'top secret data'
```

> WARNING — Pitfall: **Never reuse an IV/nonce with the same key** in GCM — doing so catastrophically breaks confidentiality and authenticity. Generate a fresh random IV for every message with `randomBytes`. Also: `crypto.createCipher`/`createDecipher` (without `iv`) are **deprecated and insecure** (they derive a key/IV weakly from a password). Always use the `...iv` variants.

> WARNING — Pitfall: For AES-256 the key must be exactly **32 bytes**. Passing a passphrase string directly is wrong; derive a fixed-length key (e.g. via `scrypt`) or load a properly-sized key from a secrets manager.

### `crypto` quick reference

| Need | API | Notes |
| --- | --- | --- |
| Integrity / content hash | `createHash('sha256')` | One-way; not for passwords |
| Authenticated message digest | `createHmac('sha256', key)` | Webhooks, signed cookies |
| Constant-time compare | `timingSafeEqual(a, b)` | Same-length Buffers only |
| Secure random bytes | `randomBytes(n)` | Tokens, salts, keys |
| Random integer | `randomInt(min, max)` | Unbiased |
| UUID v4 | `randomUUID()` | IDs, idempotency keys |
| Password hashing | `scrypt` / `pbkdf2` | Or `argon2`/`bcrypt` package |
| Symmetric encryption | `createCipheriv('aes-256-gcm', …)` | Authenticated; fresh IV each time |

---

## URLs: the `URL` class, legacy `url`, `URLSearchParams`, and `querystring`

Parsing and building URLs comes up constantly: reading request URLs, constructing outbound API calls, manipulating query strings. Node has three overlapping tools here, and knowing which to use matters.

### The WHATWG `URL` class (use this)

`URL` is a global (no import needed) implementing the same standard browsers use. It parses a URL into structured, mutable components.

```js
// url-class.js
const myUrl = new URL('https://user:pass@example.com:8080/path/to/page?q=node&page=2#section');

console.log(myUrl.protocol); // 'https:'
console.log(myUrl.hostname); // 'example.com'
console.log(myUrl.port);     // '8080'
console.log(myUrl.pathname); // '/path/to/page'
console.log(myUrl.search);   // '?q=node&page=2'
console.log(myUrl.hash);     // '#section'
console.log(myUrl.username); // 'user'
console.log(myUrl.origin);   // 'https://example.com:8080'
console.log(myUrl.href);     // full normalized URL string

// Components are mutable; href updates automatically.
myUrl.pathname = '/new/path';
myUrl.searchParams.set('page', '3');
console.log(myUrl.href); // https://user:pass@example.com:8080/new/path?q=node&page=3#section
```

Resolve a relative URL against a base by passing a second argument — the correct, standards-compliant way to join paths:

```js
// url-resolve.js
const base = 'https://api.example.com/v1/';
console.log(new URL('users/42', base).href);   // https://api.example.com/v1/users/42
console.log(new URL('/auth', base).href);      // https://api.example.com/auth  (leading / resets path)
console.log(new URL('../v2/users', base).href);// https://api.example.com/v2/users
```

> WARNING — Pitfall: `new URL()` **throws** a `TypeError` for an invalid URL. When parsing untrusted input, wrap it in `try/catch`, or use the non-throwing `URL.parse()` (Node 22.1+, returns `null` on failure) / `URL.canParse(str)` to validate first.

```js
// url-safe-parse.js
const input = 'not a url';
console.log(URL.canParse(input));      // false
const parsed = URL.parse('https://ok.com'); // returns URL or null (Node 22.1+)
console.log(parsed?.hostname);         // 'ok.com'
```

### `URLSearchParams` — work with query strings

`searchParams` on a `URL` is a `URLSearchParams` instance; you can also construct one standalone. It handles encoding/decoding for you and supports repeated keys.

```js
// search-params.js
const params = new URLSearchParams('tag=node&tag=express&sort=desc');

console.log(params.get('tag'));     // 'node' (first value)
console.log(params.getAll('tag'));  // [ 'node', 'express' ]
console.log(params.has('sort'));    // true

params.append('tag', 'mongo');      // add another value
params.set('sort', 'asc');          // replace
params.delete('page');

console.log(params.toString());     // tag=node&tag=express&tag=mongo&sort=asc

// Build from an object (single-value keys):
const built = new URLSearchParams({ q: 'a b', page: '2' });
console.log(built.toString());      // q=a+b&page=2   (spaces are encoded)

// Iterable:
for (const [key, value] of new URLSearchParams('a=1&b=2')) {
  console.log(key, value);
}
```

> TIP: `URLSearchParams` encodes spaces as `+` (form-encoding). If you need `%20`, use `encodeURIComponent` directly. Both decode correctly on the way back.

### The legacy `url` module

Before the `URL` class, Node had `url.parse()` / `url.format()` / `url.resolve()`. These are **legacy** and have quirks (e.g. they don't validate, and `url.parse` is vulnerable to certain confusions). You will still see them in older codebases.

```js
// legacy-url.js
const url = require('node:url');

// Legacy API — avoid in new code.
const parsed = url.parse('https://example.com/p?x=1', true); // 2nd arg parses query to object
console.log(parsed.query); // { x: '1' }

// Bridges between the two worlds:
const myUrl = new URL('https://example.com/p?x=1');
console.log(url.format(myUrl));               // serialize a URL (with options)
console.log(url.fileURLToPath('file:///a/b'));// 'file:///...' -> OS path (very useful!)
console.log(url.pathToFileURL('/a/b').href);  // OS path -> file:// URL
```

> BEST PRACTICE: Use the WHATWG `URL` class for all parsing and building. The one place the legacy module remains essential is `url.fileURLToPath()` / `url.pathToFileURL()` — needed in ES Modules to convert `import.meta.url` into a filesystem path (since `__dirname`/`__filename` don't exist in ESM).

```js
// esm-dirname.mjs  (ES Module)
import { fileURLToPath } from 'node:url';
import { dirname } from 'node:path';

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
console.log(__dirname);
// Note: Node 20.11+/21.2+ also exposes import.meta.dirname and import.meta.filename directly.
```

### The `querystring` module

`querystring` predates `URLSearchParams` and parses/stringifies query strings to/from plain objects. It is slightly faster and returns a plain object (no `URLSearchParams` wrapper), but is considered legacy for most purposes.

```js
// querystring.js
const qs = require('node:querystring');

const obj = qs.parse('a=1&b=2&b=3');      // { a: '1', b: [ '2', '3' ] }
console.log(obj);

const str = qs.stringify({ a: 1, b: [2, 3] }); // 'a=1&b=2&b=3'
console.log(str);

// Custom separators (e.g. semicolons):
console.log(qs.parse('a=1;b=2', ';'));    // { a: '1', b: '2' }
```

### URL tooling comparison

| Task | Recommended | Avoid / Legacy |
| --- | --- | --- |
| Parse a full URL | `new URL(str)` | `url.parse()` |
| Validate untrusted URL | `URL.canParse` / `URL.parse` | — |
| Read/modify query string | `URLSearchParams` | `querystring` |
| Resolve relative URL | `new URL(rel, base)` | `url.resolve()` |
| `import.meta.url` → path | `url.fileURLToPath()` | — (this one is current) |

---

## The `assert` Module — Sanity Checks and Tests

`assert` throws an `AssertionError` when a condition is not met. The **why**: it documents and enforces invariants ("this must never be null here"), guards against programmer error, and serves as the backbone of simple test suites — including Node's own `node:test` runner.

> TIP: Always use the **strict** mode (`require('node:assert/strict')` or `require('node:assert').strict`). It makes equality checks use `===`/`Object.is` semantics instead of the loose `==` of the legacy methods, which is almost always what you want.

```js
// assert-basics.js
const assert = require('node:assert/strict');

assert(true);                              // truthy check; throws if falsy
assert.equal(1 + 1, 2);                    // strict (===) equality in /strict mode
assert.notEqual(1, 2);
assert.deepEqual({ a: 1 }, { a: 1 });      // deep structural equality
assert.deepStrictEqual([1, 2], [1, 2]);    // deep + strict type checks

assert.ok('non-empty', 'value should be truthy'); // 2nd arg = custom message

// Assert that a function throws (and optionally what it throws):
assert.throws(
  () => { throw new TypeError('bad input'); },
  { name: 'TypeError', message: 'bad input' }
);

// Assert it does NOT throw:
assert.doesNotThrow(() => JSON.parse('{}'));

// Async rejection assertions:
await assert.rejects(async () => { throw new Error('nope'); }, /nope/);
await assert.doesNotReject(async () => 42);

// assert.fail() to force a failure (e.g. an unreachable branch):
function handle(kind) {
  switch (kind) {
    case 'a': return 1;
    default: assert.fail(`unexpected kind: ${kind}`);
  }
}
```

> WARNING — Pitfall: `assert` is for **programmer errors and tests**, not for validating untrusted user input. An assertion failure crashes the process (uncaught `AssertionError`). For request validation, return a `400` response instead. Also, never wrap code with side effects inside `assert(...)` expecting it to run when assertions are disabled — Node has no flag to strip assertions, but conceptually treat them as checks, not control flow.

---

## `timers/promises` — Awaitable Timers

Historically you wrapped `setTimeout` in a `new Promise` to `await` a delay. Node 15+ provides `node:timers/promises` with Promise-native versions, including cancellation via `AbortSignal`.

```js
// timers-promises.js
const { setTimeout: sleep, setImmediate: immediate, setInterval } = require('node:timers/promises');

async function main() {
  console.log('start');
  await sleep(1000);                 // pause 1s without blocking the event loop
  console.log('1 second later');

  const value = await sleep(500, 'resolved value'); // resolve with a value
  console.log(value);                // 'resolved value'

  await immediate();                 // yield to the event loop (like setImmediate)

  // Async-iterable interval — emits every 1s:
  let count = 0;
  for await (const _ of setInterval(1000)) {
    console.log('tick', ++count);
    if (count === 3) break;          // breaking clears the interval
  }
}
main();
```

Cancellation with `AbortController` is the killer feature — you can race a delay against an abort signal cleanly:

```js
// timers-abort.js
const { setTimeout: sleep } = require('node:timers/promises');

async function withTimeout() {
  const ac = new AbortController();
  setTimeout(() => ac.abort(), 200); // (this is the GLOBAL setTimeout) cancel after 200ms

  try {
    await sleep(1000, undefined, { signal: ac.signal });
    console.log('completed');
  } catch (err) {
    if (err.name === 'AbortError') console.log('timer was aborted'); // this branch runs
    else throw err;
  }
}
withTimeout();
```

> BEST PRACTICE: Prefer `timers/promises` over hand-rolled `new Promise(r => setTimeout(r, ms))` — it is clearer, supports cancellation, and avoids leaking timers. Note the import renames `setTimeout` to avoid shadowing the global; pick clear aliases like `sleep`.

---

## `perf_hooks` — Measuring Performance Accurately

To know whether an optimization helped, you must measure. `Date.now()` is wall-clock time with millisecond resolution and can jump (NTP adjustments). `perf_hooks` gives a **monotonic, high-resolution** clock and structured measurement tools.

```js
// perf-now.js
const { performance } = require('node:perf_hooks');

const start = performance.now();   // sub-millisecond, monotonic timestamp
let sum = 0;
for (let i = 0; i < 1e7; i++) sum += i;
const elapsed = performance.now() - start;
console.log(`Loop took ${elapsed.toFixed(3)} ms`);
```

For named, reusable measurements use marks and measures, observed via a `PerformanceObserver`:

```js
// perf-marks.js
const { performance, PerformanceObserver } = require('node:perf_hooks');

// Subscribe to measurement entries.
const obs = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log(`${entry.name}: ${entry.duration.toFixed(2)} ms`);
  }
});
obs.observe({ entryTypes: ['measure'] });

performance.mark('A');
for (let i = 0; i < 1e6; i++) Math.sqrt(i);
performance.mark('B');
performance.measure('sqrt-loop', 'A', 'B'); // duration between marks A and B

// performance.timerify wraps a function to auto-measure each call:
function fib(n) { return n < 2 ? n : fib(n - 1) + fib(n - 2); }
const timedFib = performance.timerify(fib);
const obs2 = new PerformanceObserver((list) => {
  console.log('fib took', list.getEntries()[0].duration.toFixed(2), 'ms');
  obs2.disconnect();
});
obs2.observe({ entryTypes: ['function'] });
timedFib(30);
```

> TIP: For nanosecond precision without floating point, `process.hrtime.bigint()` returns a `BigInt` of nanoseconds — great for micro-benchmarks: `const t0 = process.hrtime.bigint(); /* ... */ const ns = process.hrtime.bigint() - t0;`. For statistically rigorous benchmarks, use a library like `benchmark` or `tinybench` that handles warm-up and variance.

> WARNING — Pitfall: A single timing is noisy — JIT warm-up, garbage collection, and OS scheduling all add variance. Run the workload many times, discard the first runs (warm-up), and report a median or average.

---

## `readline` — Interactive Command-Line Input

`readline` reads input line-by-line from a stream (usually `process.stdin`). Use it for interactive CLIs, prompts, and processing large text files one line at a time without loading them fully into memory.

### Promise-based interface (modern)

```js
// readline-prompt.mjs  (or .js with require below)
const readline = require('node:readline/promises');

const rl = readline.createInterface({
  input: process.stdin,
  output: process.stdout,
});

async function main() {
  const name = await rl.question('What is your name? ');
  const lang = await rl.question('Favorite language? ');
  console.log(`Hello ${name}, enjoy your ${lang}!`);
  rl.close(); // release stdin so the process can exit
}
main();
```

### Reading a file line-by-line (memory-efficient)

```js
// readline-file.js
const readline = require('node:readline');
const fs = require('node:fs');

const rl = readline.createInterface({
  input: fs.createReadStream('big.log'),
  crlfDelay: Infinity, // treat \r\n as a single line break (Windows-safe)
});

let lineNumber = 0;
rl.on('line', (line) => {            // 'line' fires once per line
  lineNumber++;
  if (line.includes('ERROR')) console.log(`${lineNumber}: ${line}`);
});
rl.on('close', () => console.log(`Done. ${lineNumber} lines processed.`));

// Modern alternative: for await (const line of rl) { ... }
```

> WARNING — Pitfall: Always `rl.close()` when done with an interactive interface, or the process will hang waiting on `stdin`. When reading files, the interface closes automatically at end-of-stream.

> TIP: For hidden input (passwords), `readline.question` echoes characters. To mask input you must intercept keypress events and write over them, or use a dedicated package like `inquirer` or `prompts`, which also give you menus, checkboxes, and validation out of the box.

---

## `zlib` — Compression and Decompression

`zlib` provides gzip, deflate, and brotli compression. The **why**: compress HTTP responses to save bandwidth, compress files/backups, and reduce the size of data stored or sent over the network.

### One-shot (buffer) API

```js
// zlib-buffer.js
const zlib = require('node:zlib');
const { promisify } = require('node:util');

const gzip = promisify(zlib.gzip);
const gunzip = promisify(zlib.gunzip);

async function main() {
  const original = Buffer.from('Node.js core modules are powerful! '.repeat(20));
  const compressed = await gzip(original);
  const restored = await gunzip(compressed);

  console.log('Original:  ', original.length, 'bytes');
  console.log('Compressed:', compressed.length, 'bytes');
  console.log('Ratio:     ', (compressed.length / original.length * 100).toFixed(1) + '%');
  console.log('Round-trip OK:', restored.equals(original));
}
main();
```

### Streaming API (for large data / files)

Streaming avoids buffering the whole payload in memory. Use `pipeline` (it propagates errors and cleans up correctly) to connect read → transform → write.

```js
// zlib-stream.js
const zlib = require('node:zlib');
const fs = require('node:fs');
const { pipeline } = require('node:stream/promises');

async function compressFile(src, dest) {
  await pipeline(
    fs.createReadStream(src),     // source
    zlib.createGzip(),            // gzip transform stream
    fs.createWriteStream(dest)    // destination .gz file
  );
  console.log(`Compressed ${src} -> ${dest}`);
}

compressFile(__filename, __filename + '.gz').catch(console.error);
```

> BEST PRACTICE: Use `stream.pipeline` (callback or `stream/promises` version), **not** `.pipe()`, when chaining streams that include compression. `pipeline` forwards errors and destroys all streams on failure, preventing file-descriptor and memory leaks that `.pipe()` can cause.

### gzip vs deflate vs brotli

| Codec | Create stream | Notes |
| --- | --- | --- |
| gzip | `createGzip` / `createGunzip` | gzip header+checksum; standard for HTTP `Content-Encoding: gzip` |
| deflate | `createDeflate` / `createInflate` | Raw zlib format; smaller header, less interoperable |
| brotli | `createBrotliCompress` / `createBrotliDecompress` | Best ratio for text; slower at high quality; `Content-Encoding: br` |

> TIP: In Express you usually don't call `zlib` directly for responses — the `compression` middleware handles negotiation. Reach for `zlib` when compressing files, building tools, or implementing custom protocols.

---

## A Note on `child_process` (covered deeply later)

`child_process` lets a Node program spawn and communicate with other processes — running shell commands, invoking other programs, or scaling across cores. We cover it fully in a dedicated later module, but here is the mental map so it fits with the rest of the standard library:

- `exec(cmd, cb)` — run a shell command, **buffer** all output, then call back. Convenient for short output; dangerous with untrusted input (shell injection) and memory-heavy for large output.
- `execFile(file, args, cb)` — run an executable directly (no shell). Safer than `exec`.
- `spawn(cmd, args)` — launch a process and **stream** stdio. Best for long-running processes or large output.
- `fork(modulePath)` — spawn a new Node process with an IPC channel for message passing; the basis of the `cluster` module.

```js
// child-process-preview.js (full coverage in the dedicated module)
const { execFile } = require('node:child_process');

// Prefer execFile over exec — no shell means no shell-injection risk.
execFile('node', ['--version'], (err, stdout) => {
  if (err) throw err;
  console.log('Node version:', stdout.trim());
});
```

> WARNING — Pitfall: Never build a shell command by concatenating untrusted input into `exec`. Use `execFile`/`spawn` with an arguments array so user data can never be interpreted as shell syntax.

---

## Common Pitfalls & Gotchas

- **`os` ignores container limits.** `os.cpus().length`, `os.totalmem()`, and `os.freemem()` report host-level values inside Docker/K8s. Use `os.availableParallelism()` for pool sizing and `process.memoryUsage()` for your process's memory.
- **`util.inspect` default depth is 2.** Deeply nested objects show `[Object]`. Pass `{ depth: null }` to see everything when debugging.
- **`util.promisify` requires error-first callbacks with the callback last.** It won't work on functions that pass multiple success values (use the `promisify.custom` symbol) or that put the callback elsewhere.
- **Comparing secrets with `===` leaks timing.** Always use `crypto.timingSafeEqual` for tokens, HMACs, and password hashes — and guard equal lengths first.
- **`Math.random()` is not secure.** Never use it for tokens, IDs, salts, or keys. Use `crypto.randomBytes` / `randomUUID` / `randomInt`.
- **Fast hashes for passwords are a vulnerability.** Use `scrypt`/`pbkdf2` (or `argon2`/`bcrypt`), always with a unique per-user salt and a tuned work factor.
- **Reusing an IV in AES-GCM is catastrophic.** Generate a fresh random IV per message; never use the deprecated `createCipher` (no `iv`).
- **`new URL()` throws on invalid input.** Wrap untrusted parsing in `try/catch` or use `URL.canParse` / `URL.parse`.
- **`URLSearchParams` encodes spaces as `+`.** Expected behavior, but surprising if you assumed `%20`.
- **`assert` crashes the process on failure.** It's for invariants and tests, not user-input validation.
- **Forgetting `rl.close()`** leaves an interactive process hanging on stdin.
- **Using `.pipe()` instead of `pipeline` with zlib** can leak resources on error.
- **Timing once is noisy.** Warm up and average when benchmarking with `perf_hooks`.

---

## Best Practices

- **Prefer the `node:` prefix** for all core-module imports — clarity and future-proofing.
- **Reach for built-in Promise APIs first** (`fs/promises`, `timers/promises`, `stream/promises`) before `util.promisify`.
- **Use `crypto.randomUUID()`** for IDs and **`randomBytes(32).toString('base64url')`** for secret tokens — no dependencies, correct entropy.
- **Always hash passwords with a slow KDF + salt + work factor**, and prefer maintained packages (`argon2`/`bcrypt`) when you can.
- **Encrypt with authenticated modes (AES-256-GCM)**, fresh IV per message, and store IV + tag + ciphertext together.
- **Always compare secrets with `crypto.timingSafeEqual`.**
- **Use the WHATWG `URL` class** for all URL work; keep the legacy `url` module only for `fileURLToPath`/`pathToFileURL` in ESM.
- **Use `assert/strict`** so equality is strict by default; keep assertions for invariants/tests only.
- **Use `timers/promises` with `AbortSignal`** for cancellable delays and timeouts.
- **Measure with `perf_hooks`** (monotonic clock), warming up and averaging; use `process.hrtime.bigint()` for nanosecond precision.
- **Use `stream.pipeline`** for any stream chain involving `zlib`.
- **Prefer `execFile`/`spawn` over `exec`** to avoid shell-injection.

---

## Key Interview Points

- **What is a "core module" and how is it different from an npm package?**
  Core modules ship with Node, load instantly and synchronously, require no install, and are versioned with the runtime. Importable via the `node:` prefix.

- **Why use `os.availableParallelism()` instead of `os.cpus().length`?**
  `cpus().length` reports the host's logical cores, ignoring container CPU limits, so a CPU-limited pod over-spawns workers. `availableParallelism()` respects cgroup limits and affinity.

- **When would you use `util.promisify` vs a native Promise API?**
  Use the native Promise variant (`fs.promises`, etc.) when it exists. Use `promisify` to wrap older/third-party error-first callback functions. For non-standard callbacks, define the `util.promisify.custom` symbol.

- **Hash vs HMAC — what's the difference and when do you use each?**
  A hash is a keyless one-way digest for integrity/content addressing. An HMAC adds a secret key to prove authenticity and integrity — used for webhook signatures and signed cookies. Verify HMACs with constant-time comparison.

- **Why can't you use SHA-256 to store passwords?**
  It's far too fast — billions of guesses per second on a GPU. Passwords need a deliberately slow, salted KDF (`scrypt`/`pbkdf2`/`argon2`/`bcrypt`) with a per-user salt and tunable work factor.

- **What is an IV and why must it be unique?**
  The initialization vector randomizes encryption so identical plaintexts produce different ciphertexts. In GCM, reusing an IV with the same key breaks both confidentiality and authenticity, so it must be random per message.

- **Why `crypto.timingSafeEqual` instead of `===`?**
  String/`===` comparison short-circuits on the first mismatch, leaking timing that lets attackers recover secrets byte-by-byte. `timingSafeEqual` always compares the full length in constant time.

- **`Math.random()` vs `crypto.randomBytes` — which for tokens?**
  `crypto.randomBytes` (a CSPRNG). `Math.random()` is predictable and unsuitable for any security purpose.

- **Difference between the WHATWG `URL` class and the legacy `url` module?**
  `URL` is the modern, standards-based, validating parser (also a browser global). The legacy `url` module is older and quirky; its still-useful piece is `fileURLToPath`/`pathToFileURL` for ESM path handling.

- **`URLSearchParams` vs `querystring`?**
  `URLSearchParams` is the modern standard, handles repeated keys, and integrates with `URL`. `querystring` is legacy, returns plain objects, and is marginally faster but discouraged for new code.

- **How do you measure code performance accurately in Node?**
  Use `perf_hooks` `performance.now()` (monotonic, high-resolution) or `process.hrtime.bigint()` for nanoseconds, with warm-up and averaging — not `Date.now()`, which is wall-clock and can jump.

- **Why prefer `stream.pipeline` over `.pipe()`?**
  `pipeline` propagates errors and destroys all streams on failure, preventing resource leaks that bare `.pipe()` chains suffer.

- **`exec` vs `execFile` vs `spawn` vs `fork`?**
  `exec` runs a shell command and buffers output (injection risk, memory bound); `execFile` runs an executable without a shell (safer); `spawn` streams stdio for long/large output; `fork` spawns a Node child with an IPC channel.

---

## Practice Exercises

Try each before reading its solution. Increasing difficulty.

1. **System report (`os`).** Write a script that prints a one-line system summary: hostname, platform/arch, number of logical cores (using the container-aware API), total and free memory in GB, and the first non-internal IPv4 address.

2. **Promisify a callback API (`util`).** Given a callback-style function `delayedAdd(a, b, cb)` that calls `cb(null, a + b)` after 100 ms, promisify it and `await` the result. Then write the same delay using `timers/promises` instead.

3. **Signed token verifier (`crypto`).** Implement `sign(payload, secret)` returning `payload.signatureHex` and `verify(signed, secret)` returning a boolean, using HMAC-SHA256 and a **constant-time** comparison. Make sure tampering with the payload fails verification.

4. **Password hashing round-trip (`crypto`).** Using `scrypt`, implement async `hashPassword(pw)` (returns `salt:hash`) and `verifyPassword(pw, stored)`. Demonstrate that the correct password verifies and a wrong one does not.

5. **URL query builder (`URL` / `URLSearchParams`).** Write `buildApiUrl(base, path, params)` that safely joins a base and path and appends an object of query params (supporting array values as repeated keys). Validate the final URL with `URL.canParse`.

6. **Compress-and-measure (`zlib` + `perf_hooks`).** Write an async function that gzips a given string, measures how long compression takes with `perf_hooks`, and returns `{ originalSize, compressedSize, ratio, ms }`. Verify a decompression round-trip equals the original.

### Solutions

#### Solution 1 — System report

```js
// sol1-sysreport.js
const os = require('node:os');

function firstIPv4() {
  for (const addrs of Object.values(os.networkInterfaces())) {
    for (const a of addrs) {
      if (a.family === 'IPv4' && !a.internal) return a.address;
    }
  }
  return 'n/a';
}

const gb = (b) => (b / 1024 ** 3).toFixed(2);

console.log(
  `${os.hostname()} | ${os.platform()}/${os.arch()} | ` +
  `${os.availableParallelism()} cores | ` +
  `mem ${gb(os.freemem())}/${gb(os.totalmem())} GB free | ` +
  `ip ${firstIPv4()}`
);
```

#### Solution 2 — Promisify + timers/promises

```js
// sol2-promisify.js
const util = require('node:util');
const { setTimeout: sleep } = require('node:timers/promises');

function delayedAdd(a, b, cb) {
  setTimeout(() => cb(null, a + b), 100); // error-first, callback last
}

const delayedAddP = util.promisify(delayedAdd);

(async () => {
  console.log('promisify result:', await delayedAddP(2, 3)); // 5

  // Same delay using timers/promises (no callback wrapping needed):
  await sleep(100);
  console.log('done waiting via timers/promises');
})();
```

#### Solution 3 — Signed token verifier

```js
// sol3-sign.js
const crypto = require('node:crypto');

function sign(payload, secret) {
  const sig = crypto.createHmac('sha256', secret).update(payload).digest('hex');
  return `${payload}.${sig}`;
}

function verify(signed, secret) {
  const idx = signed.lastIndexOf('.');
  if (idx === -1) return false;
  const payload = signed.slice(0, idx);
  const provided = signed.slice(idx + 1);

  const expected = crypto.createHmac('sha256', secret).update(payload).digest();
  const providedBuf = Buffer.from(provided, 'hex');

  return (
    expected.length === providedBuf.length &&
    crypto.timingSafeEqual(expected, providedBuf)
  );
}

const secret = 's3cr3t';
const token = sign('user=42', secret);
console.log('valid:    ', verify(token, secret));               // true
console.log('tampered: ', verify(token.replace('42', '99'), secret)); // false
```

#### Solution 4 — Password hashing round-trip

```js
// sol4-password.js
const crypto = require('node:crypto');
const { promisify } = require('node:util');
const scrypt = promisify(crypto.scrypt);

async function hashPassword(pw) {
  const salt = crypto.randomBytes(16);
  const key = await scrypt(pw, salt, 64);
  return `${salt.toString('hex')}:${key.toString('hex')}`;
}

async function verifyPassword(pw, stored) {
  const [saltHex, keyHex] = stored.split(':');
  const key = Buffer.from(keyHex, 'hex');
  const derived = await scrypt(pw, Buffer.from(saltHex, 'hex'), key.length);
  return crypto.timingSafeEqual(key, derived);
}

(async () => {
  const stored = await hashPassword('s3cret-pw');
  console.log('correct:', await verifyPassword('s3cret-pw', stored)); // true
  console.log('wrong:  ', await verifyPassword('nope', stored));      // false
})();
```

#### Solution 5 — URL query builder

```js
// sol5-buildurl.js
function buildApiUrl(base, path, params = {}) {
  const url = new URL(path, base); // resolves path against base correctly
  for (const [key, value] of Object.entries(params)) {
    if (Array.isArray(value)) {
      for (const v of value) url.searchParams.append(key, v); // repeated keys
    } else if (value != null) {
      url.searchParams.set(key, value);
    }
  }
  if (!URL.canParse(url.href)) throw new Error('built an invalid URL');
  return url.href;
}

console.log(
  buildApiUrl('https://api.example.com/v1/', 'search', {
    q: 'node js',
    tag: ['express', 'mongo'],
    page: 2,
  })
);
// https://api.example.com/v1/search?q=node+js&tag=express&tag=mongo&page=2
```

#### Solution 6 — Compress-and-measure

```js
// sol6-compress-measure.js
const zlib = require('node:zlib');
const { promisify } = require('node:util');
const { performance } = require('node:perf_hooks');

const gzip = promisify(zlib.gzip);
const gunzip = promisify(zlib.gunzip);

async function compressAndMeasure(input) {
  const original = Buffer.from(input);

  const start = performance.now();
  const compressed = await gzip(original);
  const ms = performance.now() - start;

  const restored = await gunzip(compressed);
  if (!restored.equals(original)) throw new Error('round-trip mismatch');

  return {
    originalSize: original.length,
    compressedSize: compressed.length,
    ratio: +(compressed.length / original.length).toFixed(3),
    ms: +ms.toFixed(3),
  };
}

compressAndMeasure('Repeat me! '.repeat(500)).then(console.log);
// e.g. { originalSize: 5500, compressedSize: 60, ratio: 0.011, ms: 0.842 }
```

---

## Summary / Cheat Sheet

| Module | Use it for | Headline APIs |
| --- | --- | --- |
| `os` | Machine/OS info, pool sizing | `platform`, `arch`, `availableParallelism`, `cpus`, `totalmem`/`freemem`, `hostname`, `homedir`, `networkInterfaces`, `EOL` |
| `util` | Callback↔Promise bridges, diagnostics | `promisify`, `callbackify`, `inspect`, `format`, `types`, `deprecate`, `parseArgs` |
| `crypto` | Hashing, auth, randomness, encryption | `createHash`, `createHmac`, `randomBytes`, `randomUUID`, `randomInt`, `scrypt`/`pbkdf2`, `createCipheriv`/`createDecipheriv`, `timingSafeEqual` |
| `URL`/`url` | Parse & build URLs | `new URL()`, `URL.canParse`, `searchParams`, `fileURLToPath`/`pathToFileURL` |
| `URLSearchParams` | Query strings | `get`, `getAll`, `set`, `append`, `delete`, `toString` |
| `querystring` | Legacy query parsing | `parse`, `stringify` |
| `assert` | Invariants & tests | `assert/strict`, `equal`, `deepStrictEqual`, `throws`, `rejects`, `fail` |
| `timers/promises` | Awaitable/cancellable timers | `setTimeout` (sleep), `setInterval`, `setImmediate`, `AbortSignal` |
| `perf_hooks` | Performance measurement | `performance.now`, `mark`/`measure`, `PerformanceObserver`, `timerify` |
| `readline` | Interactive CLI / line-by-line files | `createInterface`, `question` (promises), `'line'` event |
| `zlib` | Compression | `gzip`/`gunzip`, `createGzip`, `createBrotliCompress`, with `stream.pipeline` |
| `child_process` | Run other processes (later module) | `exec`, `execFile`, `spawn`, `fork` |

**Golden rules to remember:**

- Use the `node:` prefix; prefer built-in Promise APIs over `promisify`.
- `availableParallelism()` over `cpus().length`; `process.memoryUsage()` over `os.freemem()` for *your* process.
- `crypto.randomBytes`/`randomUUID` for anything secret; never `Math.random()`.
- Slow, salted KDF for passwords; AES-256-GCM with a fresh IV for encryption; `timingSafeEqual` for all secret comparisons.
- WHATWG `URL` for parsing; `assert/strict` for checks; `timers/promises` for delays; `perf_hooks` for measurement; `stream.pipeline` with `zlib`.
