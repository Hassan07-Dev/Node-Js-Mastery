# Streams & Buffers

Welcome to one of the most defining topics in all of Node.js. If you want to understand *why* Node became the go-to runtime for I/O-heavy servers, real-time data, file processing, and proxies, you have to understand **Buffers** (how Node represents raw binary data) and **Streams** (how Node moves that data efficiently, in chunks, without blocking).

This module assumes you are a strong JavaScript developer who is new to backend work. We will build everything from first principles: what binary data even *is*, why streaming exists, and how to wield it like a senior engineer. By the end you will be able to copy gigabyte files using a few kilobytes of RAM, compress data on the fly, build custom stream components, and answer the streaming questions that show up in nearly every serious Node.js interview.

## What You'll Learn

- **Binary data fundamentals**: what bytes are, why `Buffer` exists, and how it differs from a JavaScript `string` or `Array`.
- **The Buffer API end to end**: `Buffer.from`, `Buffer.alloc`, `Buffer.allocUnsafe`, reading/writing integers, `Buffer.concat`, slicing/subarrays, and the encodings `utf8`, `hex`, and `base64`.
- **Why streams exist** — memory efficiency and time-to-first-byte — illustrated with a concrete before/after example.
- **The four stream types**: Readable, Writable, Duplex, and Transform, and how they relate.
- **Consuming Readable streams**: the `data` / `end` / `error` events, flowing vs. paused mode, and `.read()`.
- **Writing to Writable streams**: `write()`, `end()`, the `drain` event, and what the return value of `write()` means.
- **`pipe()` vs. `stream.pipeline()`** — and the concrete reasons `pipeline` is the safer default.
- **Backpressure** — the single most important streaming concept — explained from scratch.
- **Building custom streams**: your own Readable, Writable, and Transform implementations.
- **Real-world examples**: copying a file, gzip compression with `zlib`, and streaming an HTTP response.
- **Best practices, pitfalls, interview answers, and exercises** to lock it all in.

---

## Part 1: Buffers — Node's Binary Data Type

### What is binary data, really?

At the lowest level, computers store everything — text, images, audio, network packets — as sequences of **bytes**. A byte is 8 bits, and each bit is a 0 or 1, so a single byte can hold one of 256 values (0–255). A file on disk is just an ordered sequence of bytes. A network message is just an ordered sequence of bytes. There is no "text" or "image" at this level — those are *interpretations* we apply to bytes using an **encoding**.

JavaScript in the browser historically had no good way to deal with raw bytes. Strings are sequences of UTF-16 code units, and `Array` is a general-purpose, boxed, slow container. Neither is appropriate for efficiently handling raw I/O. When Node.js was created, it needed a first-class way to represent "a chunk of raw bytes read from a file or a socket." That type is the **`Buffer`**.

> TIP: A `Buffer` is essentially a fixed-length array of bytes (integers 0–255) that lives in memory *outside* the V8 JavaScript heap. This off-heap allocation is what makes it efficient for I/O and large data.

### Buffer vs. string vs. Array

| Feature | `Buffer` | `string` | `Array` |
| --- | --- | --- | --- |
| Stores | Raw bytes (0–255) | UTF-16 text | Arbitrary boxed values |
| Length | Fixed at creation | Immutable | Dynamic |
| Mutable contents | Yes | No | Yes |
| Memory location | Off-heap (mostly) | V8 heap | V8 heap |
| Use case | Binary I/O, encoding conversion | Human-readable text | General data structures |

### `Buffer` is a `Uint8Array`

A modern, important fact: in Node.js, `Buffer` is a **subclass of `Uint8Array`** (which is itself a kind of `TypedArray` backed by an `ArrayBuffer`). This means everything you know about typed arrays applies, and `Buffer` adds Node-specific conveniences like encoding-aware methods.

```js
const buf = Buffer.from('hi');
console.log(buf instanceof Uint8Array); // true
console.log(buf instanceof Buffer);     // true
```

> WARNING — Pitfall: Because `Buffer` extends `Uint8Array`, methods that the typed-array spec defines (like `.slice()`) historically had *different* behavior on Buffers than developers expected. We'll cover this in the slicing section — it is a classic interview trap.

### Creating Buffers

There are three primary creation APIs. The old `new Buffer()` constructor is **deprecated and unsafe** — never use it.

#### `Buffer.from(...)` — create from existing data

```js
// From a string (default encoding is 'utf8')
const fromStr = Buffer.from('Hello, 世界'); // multibyte-safe
console.log(fromStr);            // <Buffer 48 65 6c 6c 6f 2c 20 e4 b8 96 e7 95 8c>
console.log(fromStr.length);     // 13 -> BYTES, not characters!

// From an array of byte values
const fromArr = Buffer.from([0x48, 0x69]); // bytes for 'H', 'i'
console.log(fromArr.toString());           // 'Hi'

// From a hex string
const fromHex = Buffer.from('48656c6c6f', 'hex');
console.log(fromHex.toString('utf8'));     // 'Hello'

// From another Buffer (copies the data)
const copy = Buffer.from(fromArr);
copy[0] = 0x4a;                            // mutate the copy
console.log(fromArr.toString(), copy.toString()); // 'Hi' 'Ji' (original unchanged)
```

> WARNING — Pitfall: `Buffer.from(string).length` is the number of **bytes**, not characters. `Buffer.from('世').length` is `3` (in UTF-8), not `1`. Counting "characters" by buffer length will silently break with non-ASCII text and emojis.

#### `Buffer.alloc(size)` — create a zero-filled buffer (safe)

```js
const safe = Buffer.alloc(8);    // 8 bytes, all initialized to 0x00
console.log(safe);               // <Buffer 00 00 00 00 00 00 00 00>

// Optionally fill with a value and choose an encoding for the fill
const filled = Buffer.alloc(4, 'a');     // <Buffer 61 61 61 61>
const filledHex = Buffer.alloc(4, 'ff', 'hex'); // <Buffer ff ff ff ff>
```

#### `Buffer.allocUnsafe(size)` — fast but uninitialized

```js
const fast = Buffer.allocUnsafe(8);
// Contents are WHATEVER was previously in that memory region — possibly
// fragments of old data, including potentially sensitive information.
console.log(fast); // <Buffer ?? ?? ?? ?? ?? ?? ?? ??>  (unpredictable)
fast.fill(0);      // you must fill it yourself before relying on contents
```

`allocUnsafe` skips the zero-fill step, so it's faster, but the memory may contain leftover data from previously freed buffers.

| API | Initializes memory? | Speed | When to use |
| --- | --- | --- | --- |
| `Buffer.alloc(n)` | Yes (zero-filled) | Slower | Default — always safe |
| `Buffer.allocUnsafe(n)` | No | Fastest | Hot paths where you *immediately and fully* overwrite every byte |
| `Buffer.from(...)` | N/A (copies/wraps source) | — | Creating a buffer from existing data |

> WARNING — Pitfall: `Buffer.allocUnsafe` is a real security risk if you return or transmit it without fully overwriting it — you could leak passwords, keys, or other users' data that happened to be in that memory. Default to `Buffer.alloc`. Only reach for `allocUnsafe` when you have measured a bottleneck and will overwrite the entire buffer.

### Reading and writing bytes

You can index a Buffer like an array, and there are typed read/write methods for multi-byte integers.

```js
const buf = Buffer.alloc(4);

// Byte-level access (each element is 0-255)
buf[0] = 0xde;
buf[1] = 0xad;
buf[2] = 0xbe;
buf[3] = 0xef;
console.log(buf.toString('hex')); // 'deadbeef'

// Typed integer writes/reads (endianness matters)
const nums = Buffer.alloc(8);
nums.writeUInt32BE(0x12345678, 0);  // write big-endian at offset 0
nums.writeUInt32LE(0x12345678, 4);  // write little-endian at offset 4
console.log(nums.toString('hex'));  // '1234567878563412'

console.log(nums.readUInt32BE(0).toString(16)); // '12345678'
console.log(nums.readUInt32LE(4).toString(16)); // '12345678'

// BigInt reads/writes for 64-bit values
const big = Buffer.alloc(8);
big.writeBigUInt64BE(9007199254740993n, 0);
console.log(big.readBigUInt64BE(0)); // 9007199254740993n
```

> TIP: "Endianness" is the byte order used to store multi-byte numbers. Big-endian (BE) stores the most significant byte first; little-endian (LE) stores it last. Network protocols generally use big-endian ("network byte order"); most CPUs (x86, ARM) work little-endian internally. When parsing a binary protocol, the spec tells you which to use — pick the matching method.

### Concatenating buffers

`Buffer.concat(list[, totalLength])` joins multiple buffers into one new buffer. This is the canonical way to assemble chunks you've collected from a stream.

```js
const a = Buffer.from('Hello, ');
const b = Buffer.from('world');
const c = Buffer.from('!');

const joined = Buffer.concat([a, b, c]);
console.log(joined.toString()); // 'Hello, world!'

// Providing totalLength is a micro-optimization (avoids a length scan) AND
// lets you truncate: only the first totalLength bytes are kept.
const truncated = Buffer.concat([a, b, c], 5);
console.log(truncated.toString()); // 'Hello'
```

> BEST PRACTICE: When collecting chunks from a stream, push each chunk into an array and call `Buffer.concat(chunks)` *once* at the end. Repeatedly doing `result = Buffer.concat([result, chunk])` inside a loop is O(n²) because it re-copies all prior data on every chunk.

### Slicing and subarrays

Here is one of the most important and surprising Buffer behaviors. `buf.subarray()` (and the legacy `buf.slice()`) return a buffer that **shares the same underlying memory** — it is a *view*, not a copy. Mutating the view mutates the original.

```js
const original = Buffer.from('abcdef');

const view = original.subarray(0, 3); // shares memory with `original`
view[0] = 0x7a;                       // 'z'
console.log(original.toString());     // 'zbcdef'  <-- original changed!

// To get an independent COPY, use Buffer.from(...) on the slice:
const independent = Buffer.from(original.subarray(0, 3));
independent[0] = 0x41;                 // 'A'
console.log(original.toString());      // 'zbcdef'  <-- unchanged this time
```

> WARNING — Pitfall: `Buffer.prototype.slice()` is the old name and is **soft-deprecated** in favor of `subarray()`. Note also that the *standard* `Uint8Array.prototype.slice()` returns a *copy*, while Node's historical `Buffer.prototype.slice()` returns a *view* — opposite semantics. To avoid all ambiguity: use `subarray()` when you want a view and `Buffer.from(buf.subarray(...))` when you want a copy. This "is a slice a copy or a view?" question is a frequent interview gotcha.

### Encodings: utf8, hex, base64, and friends

An **encoding** is the rule that maps bytes to characters and back. The same bytes look completely different depending on which encoding you use to interpret them.

```js
const buf = Buffer.from('Hello');

console.log(buf.toString('utf8'));   // 'Hello'
console.log(buf.toString('hex'));    // '48656c6c6f'
console.log(buf.toString('base64')); // 'SGVsbG8='
console.log(buf.toString('latin1')); // 'Hello'

// Round-trip the other direction:
console.log(Buffer.from('48656c6c6f', 'hex').toString());      // 'Hello'
console.log(Buffer.from('SGVsbG8=', 'base64').toString());     // 'Hello'
```

| Encoding | What it's for | Size effect | Notes |
| --- | --- | --- | --- |
| `utf8` | Default text encoding | Variable (1–4 bytes/char) | The default; multibyte-safe |
| `ascii` / `latin1` | Single-byte legacy text | 1 byte/char | `latin1` covers 0–255; `ascii` masks to 7 bits |
| `hex` | Human-readable byte dump | 2 chars/byte | Useful for hashes, debugging |
| `base64` | Binary-safe text transport | ~1.33× size | For embedding binary in JSON, data URLs, emails |
| `base64url` | URL/filename-safe base64 | ~1.33× size | `+`/`/` replaced with `-`/`_`, no padding |
| `utf16le` | UTF-16 little-endian | 2–4 bytes/char | Interop with some Windows/.NET data |

> TIP: Base64 is *not* encryption or compression — it's an encoding that turns arbitrary bytes into a safe ASCII subset, costing about 33% more size. Use it when a transport only allows text (JSON fields, URLs, `data:` URIs), never as a security measure.

> WARNING — Pitfall: Decoding a *partial* multibyte character corrupts text. If you slice a UTF-8 buffer in the middle of a 3-byte character (common when reading streams in fixed-size chunks) and call `.toString('utf8')`, you'll get a replacement character (�). For decoding streamed text safely across chunk boundaries, use `string_decoder`'s `StringDecoder`, which buffers incomplete sequences:

```js
const { StringDecoder } = require('node:string_decoder');
const decoder = new StringDecoder('utf8');

// Imagine '世' (e4 b8 96) split across two chunks:
console.log(decoder.write(Buffer.from([0xe4, 0xb8]))); // '' (incomplete, held back)
console.log(decoder.write(Buffer.from([0x96])));       // '世' (completed)
```

---

## Part 2: Why Streams Exist

### The problem: loading everything into memory

Imagine an HTTP endpoint that serves a large file. The naive approach reads the whole file into memory, then sends it:

```js
const fs = require('node:fs');
const http = require('node:http');

// BEFORE: buffer the entire file in memory, then respond
http.createServer((req, res) => {
  fs.readFile('./big-video.mp4', (err, data) => {  // reads ENTIRE file into RAM
    if (err) {
      res.statusCode = 500;
      return res.end('Error');
    }
    res.end(data); // sends only after the whole file is loaded
  });
}).listen(3000);
```

This has two serious problems:

1. **Memory blow-up.** If `big-video.mp4` is 2 GB and 50 clients request it at once, your process tries to hold ~100 GB in RAM. It will crash (or hit `ERR_FS_FILE_TOO_LARGE` — Node refuses to `readFile` anything over ~2 GB anyway).
2. **High time-to-first-byte (TTFB).** The client receives *nothing* until the entire file has been read from disk. For a 2 GB file, that's a long, awkward wait before a single byte goes out.

### The solution: process data in chunks

A **stream** lets you process data piece by piece (in *chunks*) as it arrives, instead of waiting for and holding the whole thing. Here's the same endpoint, streamed:

```js
const fs = require('node:fs');
const http = require('node:http');

// AFTER: stream the file chunk-by-chunk straight to the client
http.createServer((req, res) => {
  const stream = fs.createReadStream('./big-video.mp4'); // a Readable stream

  stream.on('error', (err) => {
    res.statusCode = 500;
    res.end('Error');
  });

  stream.pipe(res); // pump chunks into the response as they are read
}).listen(3000);
```

The difference is dramatic:

| Aspect | Buffer-everything (`readFile`) | Streaming (`createReadStream` + `pipe`) |
| --- | --- | --- |
| Peak memory per request | Whole file (e.g., 2 GB) | A few chunks (e.g., 64 KB) |
| Time to first byte | After full read | Almost immediately |
| Handles files > 2 GB | No | Yes |
| Concurrency at scale | Falls over | Stays flat |
| Backpressure-aware | No | Yes (via `pipe`/`pipeline`) |

> BEST PRACTICE: Whenever data is large, of unknown size, or arrives over time (files, HTTP bodies, sockets, database cursors, child-process output), reach for streams. The memory footprint stays roughly **constant** regardless of total data size — that's the whole point.

### The two superpowers of streams

1. **Memory efficiency.** You hold only a small window of data at a time, so a 4 GB file and a 4 KB file use about the same RAM.
2. **Time-to-first-byte / composability.** Downstream consumers start working on the first chunk while later chunks are still being produced — and you can chain transformations (read → decompress → parse → write) so data flows through a pipeline without ever being fully materialized.

---

## Part 3: The Four Stream Types

Node's `stream` module defines four base classes. All of them are `EventEmitter`s.

| Type | Direction | You can... | Canonical examples |
| --- | --- | --- | --- |
| **Readable** | Source (out) | Read data *from* it | `fs.createReadStream`, `http` request (server), `process.stdin` |
| **Writable** | Sink (in) | Write data *to* it | `fs.createWriteStream`, `http` response (server), `process.stdout` |
| **Duplex** | Both, independent | Read and write (separate channels) | TCP socket (`net.Socket`) |
| **Transform** | Both, coupled | Write in, read out *transformed* | `zlib.createGzip`, `crypto.createCipheriv` |

A few clarifying notes:

- A **Duplex** stream has *two independent* channels — what you write is unrelated to what you read. A TCP socket is the classic example: you send bytes out and receive different bytes back.
- A **Transform** is a *special Duplex* where the readable side is the transformed version of what you wrote to the writable side. Gzip compression is the textbook case: you write raw bytes, you read compressed bytes.

```text
Readable:   [source] ──data──►
Writable:              ──data──► [sink]
Duplex:     [read side] ◄── ── ──► [write side]   (independent)
Transform:  write ──► [ f(x) ] ──► read           (coupled)
```

### Object mode

By default, streams carry `Buffer` (or string) chunks. In **object mode**, each chunk can be *any* JavaScript value (an object, a number, etc.). This is heavily used by libraries (e.g., a CSV parser stream that emits one row-object per record). Enable it with `{ objectMode: true }`.

```js
const { Readable } = require('node:stream');

const objStream = Readable.from([{ id: 1 }, { id: 2 }, { id: 3 }]); // object mode
objStream.on('data', (obj) => console.log(obj.id)); // 1, 2, 3
```

---

## Part 4: Consuming Readable Streams

A Readable stream emits chunks. There are two "modes" for reading and several events you must understand.

### The core events

```js
const fs = require('node:fs');

const readable = fs.createReadStream('./notes.txt', {
  encoding: 'utf8',       // decode chunks to strings instead of Buffers
  highWaterMark: 16 * 1024 // chunk/buffer size hint in bytes (default 64 KB)
});

readable.on('data', (chunk) => {
  // Fires for every chunk. Attaching this listener puts the stream in FLOWING mode.
  console.log(`Got ${chunk.length} characters`);
});

readable.on('end', () => {
  // Fires once, after the last chunk — no more data will come.
  console.log('Done reading.');
});

readable.on('error', (err) => {
  // ALWAYS handle this. I/O can fail at any time.
  console.error('Read failed:', err.message);
});

readable.on('close', () => {
  // Fires when the stream and its underlying resource are fully closed.
  console.log('Stream closed; file descriptor released.');
});
```

> WARNING — Pitfall: An unhandled `error` event on *any* EventEmitter (including streams) throws and **crashes your process**. Always attach an `error` listener to every stream you touch.

### Flowing mode vs. paused mode

A Readable stream is in one of two modes:

- **Paused mode (the default).** Data is not pushed at you; you must explicitly pull it with `.read()`.
- **Flowing mode.** Data is pushed to you as fast as it arrives via `data` events.

How you switch between them:

| Action | Resulting mode |
| --- | --- |
| Attach a `data` listener | → Flowing |
| Call `.pipe(destination)` | → Flowing |
| Call `.resume()` | → Flowing |
| Call `.pause()` | → Paused |
| Remove all `data` listeners + unpipe | → Paused |
| Use `for await...of` | Managed for you (paused-style pull) |

> WARNING — Pitfall: Switching to flowing mode (e.g., by attaching `data`) and then not consuming the data fast enough — or attaching a `data` listener but never doing anything with the chunks — can cause data loss or unbounded memory growth. If you switch to flowing mode, you must actually handle every chunk.

### Paused mode with `.read()` and the `readable` event

In paused mode you pull data yourself. The `readable` event fires when there is data available to be read (or when the end has been reached).

```js
const fs = require('node:fs');
const readable = fs.createReadStream('./notes.txt');

readable.on('readable', () => {
  let chunk;
  // .read() returns a chunk, or null when nothing is currently buffered.
  while ((chunk = readable.read()) !== null) {
    console.log(`Pulled ${chunk.length} bytes`);
  }
});

readable.on('end', () => console.log('No more data.'));
readable.on('error', (err) => console.error(err));
```

`.read(size)` can request a specific number of bytes; called with no argument it returns all currently buffered data. It returns `null` if not enough data is buffered yet.

### The modern way: `for await...of`

Async iteration is the cleanest, safest way to consume a readable stream in modern Node. It handles backpressure automatically (it pulls the next chunk only when your loop body is ready) and integrates with `try/catch` for errors.

```js
const fs = require('node:fs');

async function countBytes(path) {
  const readable = fs.createReadStream(path);
  let total = 0;

  try {
    for await (const chunk of readable) { // pulls one chunk at a time
      total += chunk.length;
      // any awaited work here naturally applies backpressure to the source
    }
  } catch (err) {
    console.error('Stream error:', err.message);
    throw err;
  }

  return total;
}

countBytes('./big-file.bin').then((n) => console.log(`${n} bytes`));
```

> BEST PRACTICE: For consuming a readable stream in application code, prefer `for await...of` (or `pipeline`). It's concise, applies backpressure, propagates errors via exceptions, and won't silently drop data the way a careless `data` handler can.

### Collecting an entire stream into a Buffer/string

Sometimes you genuinely need the whole thing in memory (e.g., a small request body). Do it correctly:

```js
const { buffer } = require('node:stream/consumers'); // Node 16.7+

async function readAll(readable) {
  return buffer(readable); // returns a single Buffer; also: text(), json(), arrayBuffer()
}

// Manual equivalent (good to know for interviews):
async function readAllManual(readable) {
  const chunks = [];
  for await (const chunk of readable) chunks.push(chunk);
  return Buffer.concat(chunks); // ONE concat at the end — O(n), not O(n²)
}
```

---

## Part 5: Writing to Writable Streams

A Writable stream is a sink. You push data into it with `write()` and signal completion with `end()`.

```js
const fs = require('node:fs');
const writable = fs.createWriteStream('./out.txt');

writable.write('first line\n');   // returns true or false (see below)
writable.write('second line\n');
writable.end('last line\n');      // optional final chunk, then closes the stream

writable.on('finish', () => {
  // Fires after end() and all buffered data has been flushed to the underlying resource.
  console.log('All writes flushed.');
});

writable.on('error', (err) => console.error('Write failed:', err.message));
```

> WARNING — Pitfall: Do **not** call `write()` after `end()`. It emits an `ERR_STREAM_WRITE_AFTER_END` error. `end()` means "I'm done sending data."

### `write()`'s return value and the `drain` event

This is the heart of writing efficiently. `writable.write(chunk)` returns a **boolean**:

- `true` — the internal buffer is below the `highWaterMark`; keep writing.
- `false` — the internal buffer is full (above `highWaterMark`); you *should stop writing* and wait for the `drain` event before continuing.

Ignoring a `false` return value means you keep stuffing data into a buffer that the destination can't drain fast enough — memory grows without bound. Here's how to write a large amount of data correctly by respecting backpressure manually:

```js
const fs = require('node:fs');

function writeOneMillionLines(path) {
  return new Promise((resolve, reject) => {
    const writable = fs.createWriteStream(path);
    writable.on('error', reject);

    let i = 0;
    const TOTAL = 1_000_000;

    function writeMore() {
      let ok = true;
      while (i < TOTAL && ok) {
        i += 1;
        const line = `line ${i}\n`;
        if (i === TOTAL) {
          writable.end(line, resolve); // last write; resolve on finish
        } else {
          // write() returns false when the buffer is full
          ok = writable.write(line);
        }
      }
      if (i < TOTAL) {
        // Buffer is full: WAIT for 'drain', then resume writing.
        writable.once('drain', writeMore);
      }
    }

    writeMore();
  });
}
```

> TIP: The `drain` event is the Writable stream telling you "OK, I've caught up — you may write more now." Honoring `write()`'s return value plus `drain` is *manual backpressure handling*. In practice you'll usually let `pipe`/`pipeline` do this for you, but interviewers love to ask you to implement it by hand.

### Key Writable methods and events

| Member | Meaning |
| --- | --- |
| `write(chunk[, enc][, cb])` | Queue a chunk. Returns `false` if you should pause. |
| `end([chunk][, enc][, cb])` | Optional final write, then finish the stream. |
| `cork()` / `uncork()` | Batch many small writes into fewer underlying operations. |
| `'drain'` | Buffer drained below `highWaterMark`; safe to resume writing. |
| `'finish'` | `end()` was called and all data was flushed. |
| `'close'` | Stream and underlying resource fully closed. |
| `'error'` | Something failed. Always handle. |

---

## Part 6: Connecting Streams — `pipe()` and `pipeline()`

### `readable.pipe(writable)`

`pipe()` connects a Readable's output to a Writable's input and — crucially — **handles backpressure automatically**. It pauses the source when the destination is full and resumes it on `drain`.

```js
const fs = require('node:fs');

const src = fs.createReadStream('./input.txt');
const dst = fs.createWriteStream('./output.txt');

src.pipe(dst); // copies input.txt -> output.txt, backpressure handled for you
```

You can chain pipes through Transform streams:

```js
const fs = require('node:fs');
const zlib = require('node:zlib');

fs.createReadStream('./input.txt')
  .pipe(zlib.createGzip())            // Transform: compress
  .pipe(fs.createWriteStream('./input.txt.gz'));
```

### The fatal flaw of `pipe()`: error handling

`pipe()`'s convenience hides a serious problem: **it does not forward errors, and it does not clean up the other streams when one fails.** If `src` errors, `dst` is left open (leaking a file descriptor). If `dst` errors, `src` keeps reading into the void. And an unhandled `error` on either stream crashes the process.

```js
// FRAGILE: each stream needs its own error handler, and even then,
// a failure in one does NOT automatically destroy/clean up the others.
src
  .on('error', (e) => { /* must handle, and manually clean up dst */ })
  .pipe(dst)
  .on('error', (e) => { /* must handle, and manually clean up src */ });
```

Getting this right by hand — destroying every stream in the chain on any error, in every direction — is tedious and easy to botch. This is a notorious source of file-descriptor leaks in production Node apps.

### `stream.pipeline()` — the safe default

`stream.pipeline()` pipes streams together *and* automatically:

- forwards errors,
- destroys **all** streams in the chain if any of them fails or closes early,
- calls a callback (or resolves a promise) exactly once when the whole pipeline finishes or errors.

Callback form:

```js
const fs = require('node:fs');
const zlib = require('node:zlib');
const { pipeline } = require('node:stream');

pipeline(
  fs.createReadStream('./input.txt'),
  zlib.createGzip(),
  fs.createWriteStream('./input.txt.gz'),
  (err) => {                    // single, central completion handler
    if (err) console.error('Pipeline failed:', err.message);
    else console.log('Pipeline succeeded.');
  }
);
```

Promise form (preferred in modern async code):

```js
const fs = require('node:fs');
const zlib = require('node:zlib');
const { pipeline } = require('node:stream/promises');

async function gzipFile(input, output) {
  await pipeline(
    fs.createReadStream(input),
    zlib.createGzip(),
    fs.createWriteStream(output)
  ); // resolves on success, rejects (and cleans up) on any error
}

gzipFile('./input.txt', './input.txt.gz')
  .then(() => console.log('Done'))
  .catch((err) => console.error('Failed:', err.message));
```

| Feature | `pipe()` | `pipeline()` |
| --- | --- | --- |
| Handles backpressure | Yes | Yes |
| Forwards/propagates errors | No | Yes |
| Destroys all streams on failure | No | Yes |
| Single completion signal | No (per-stream events) | Yes (callback / promise) |
| Prevents FD/memory leaks on error | No (manual) | Yes |
| Recommended for production | No | **Yes** |

> BEST PRACTICE: Default to `stream.pipeline()` (or its promise form) over `.pipe()` in any real application. The only time bare `pipe()` is acceptable is a quick script where you don't care about robustness. Pipeline's automatic cleanup is the difference between a leak-free service and a slow descriptor leak under error conditions.

> TIP: In Express/Node HTTP handlers, prefer `pipeline(source, res, cb)` over `source.pipe(res)`. If the client disconnects mid-download, `pipeline` will destroy the source stream and free its resources; bare `pipe()` may leave the read stream dangling.

---

## Part 7: Backpressure — The Most Important Concept

### What is backpressure?

Backpressure is the mechanism that prevents a **fast producer** from overwhelming a **slow consumer**. Think of pouring water from a wide jug into a narrow funnel: if you pour faster than the funnel drains, it overflows. Backpressure is the signal that says "slow down — let me catch up."

In Node, every stream has an internal buffer with a size limit called the **`highWaterMark`** (default 64 KB for byte streams, 16 *objects* in object mode). Backpressure works through the contract we already saw:

1. You call `writable.write(chunk)`.
2. If the internal buffer is now over `highWaterMark`, `write()` returns **`false`**.
3. A backpressure-aware producer **stops** sending and waits for the `'drain'` event.
4. When the buffer empties below the mark, the stream emits `'drain'` and the producer resumes.

```text
Fast Readable  ──chunks──►  [ Writable buffer ]  ──slow flush──►  Disk/Network
                                  │
                  full? write() returns false → pause source
                  drained? emit 'drain'        → resume source
```

### Why it matters

Without backpressure, a fast source (reading from a local SSD) feeding a slow sink (uploading over the network) would queue gigabytes in memory and crash the process. Backpressure keeps memory bounded by coupling the producer's rate to the consumer's rate.

### Who handles it for you?

- `for await...of` on a Readable: backpressure handled (it pulls only when ready).
- `readable.pipe(writable)`: backpressure handled.
- `stream.pipeline(...)`: backpressure handled.
- **Manual `readable.on('data', chunk => writable.write(chunk))`**: backpressure **NOT** handled unless you check `write()`'s return value and `pause()`/`resume()` yourself.

```js
// ANTI-PATTERN: ignores write()'s return value → no backpressure → memory bloat
readable.on('data', (chunk) => {
  writable.write(chunk); // if this returns false, we ignore it and keep going
});

// CORRECT manual version (this is essentially what pipe() does internally):
readable.on('data', (chunk) => {
  const ok = writable.write(chunk);
  if (!ok) {
    readable.pause();                       // stop the source
    writable.once('drain', () => readable.resume()); // resume when caught up
  }
});
readable.on('end', () => writable.end());
```

> BEST PRACTICE: Never wire a `data` handler straight into `write()` without honoring backpressure. Use `pipeline`/`pipe`/`for await` instead — they encode this contract correctly. Hand-rolling it is for interviews and library authors.

---

## Part 8: Creating Custom Streams

You implement custom streams by extending the base classes (or passing functions to their constructors) and implementing specific methods. **Important: implementation methods are prefixed with an underscore** (`_read`, `_write`, `_transform`) and are called *by the stream machinery* — you never call them directly. You drive the stream through the public API (`push`, the `callback`, etc.).

### Custom Readable

Implement `_read(size)`. Call `this.push(chunk)` to emit data and `this.push(null)` to signal end-of-stream.

```js
const { Readable } = require('node:stream');

// A Readable that emits the numbers 1..max as string chunks.
class CounterStream extends Readable {
  constructor(max, options) {
    super(options);
    this.max = max;
    this.current = 1;
  }

  _read() {
    if (this.current > this.max) {
      this.push(null);                 // signal EOF
      return;
    }
    const chunk = `${this.current}\n`;
    this.push(chunk);                  // emit a chunk
    this.current += 1;
  }
}

new CounterStream(5)
  .on('data', (c) => process.stdout.write(c))  // 1\n 2\n 3\n 4\n 5\n
  .on('end', () => console.log('done'));
```

The simplest source of all — wrapping an iterable — needs no class:

```js
const { Readable } = require('node:stream');
const fromArray = Readable.from(['a', 'b', 'c']);          // sync iterable
const fromGen = Readable.from(async function* () {          // async generator
  for (let i = 0; i < 3; i++) yield `chunk ${i}\n`;
}());
```

### Custom Writable

Implement `_write(chunk, encoding, callback)`. You **must** call `callback()` when you've finished handling the chunk (or `callback(err)` on failure) — that's how the stream knows it can send the next chunk and how backpressure is applied.

```js
const { Writable } = require('node:stream');

// A Writable that "saves" each chunk by logging it (pretend it's a DB insert).
class LoggingWritable extends Writable {
  _write(chunk, encoding, callback) {
    // Simulate async I/O:
    setImmediate(() => {
      console.log('Persisted:', chunk.toString().trim());
      callback();        // signal success → stream may send the next chunk
      // callback(err);  // would propagate an error and destroy the stream
    });
  }
}

const sink = new LoggingWritable();
sink.write('alpha\n');
sink.write('beta\n');
sink.end('gamma\n');
```

> WARNING — Pitfall: Forgetting to call `callback()` in `_write` is a classic bug. The stream will appear to "hang" after the first chunk: `finish` never fires, the pipeline stalls, and there's no error. If your writable freezes, check that every code path in `_write` calls the callback exactly once.

### Custom Transform

A Transform reads input, transforms it, and produces output. Implement `_transform(chunk, encoding, callback)`; push transformed data with `this.push(...)` and/or pass it as the second arg to `callback`. Optionally implement `_flush(callback)` to emit any trailing data at the end.

```js
const { Transform } = require('node:stream');

// Uppercases text passing through it.
class UppercaseTransform extends Transform {
  _transform(chunk, encoding, callback) {
    const upper = chunk.toString().toUpperCase();
    callback(null, upper); // (error, transformedChunk)
  }
}

// A line-buffering Transform: splits a byte stream into individual lines,
// holding partial lines across chunk boundaries (demonstrates _flush).
class LineSplitter extends Transform {
  constructor(options) {
    super({ ...options, readableObjectMode: true }); // emit one line (string) per read
    this.tail = '';
  }

  _transform(chunk, encoding, callback) {
    const data = this.tail + chunk.toString();
    const lines = data.split('\n');
    this.tail = lines.pop();          // last element may be a partial line
    for (const line of lines) this.push(line);
    callback();
  }

  _flush(callback) {
    if (this.tail) this.push(this.tail); // emit whatever's left at EOF
    callback();
  }
}

const { pipeline } = require('node:stream');
pipeline(
  process.stdin,
  new LineSplitter(),
  new Transform({
    objectMode: true,
    transform(line, enc, cb) {
      cb(null, `> ${line.toUpperCase()}\n`);
    }
  }),
  process.stdout,
  (err) => err && console.error(err)
);
```

> TIP: Notice the constructor-shorthand form (`new Transform({ transform(chunk, enc, cb) {...} })`). For one-off streams you don't need a subclass at all — just pass the implementation function(s) to the base constructor. Use a class when you have reusable, stateful logic.

---

## Part 9: Real-World Examples

### Example 1 — Copy a file (with proper error handling)

```js
const fs = require('node:fs');
const { pipeline } = require('node:stream/promises');

async function copyFile(src, dest) {
  await pipeline(
    fs.createReadStream(src),
    fs.createWriteStream(dest)
  ); // backpressure + cleanup handled; rejects on any error
}

copyFile('./photo.raw', './photo.copy.raw')
  .then(() => console.log('Copied'))
  .catch((err) => console.error('Copy failed:', err.message));
```

This copies a file of *any* size using only a small constant amount of memory — the opposite of `fs.readFile` + `fs.writeFile`.

### Example 2 — Gzip compression and decompression with `zlib`

`zlib` provides Transform streams for compression. Here's a round trip.

```js
const fs = require('node:fs');
const zlib = require('node:zlib');
const { pipeline } = require('node:stream/promises');

// Compress: input.txt -> input.txt.gz
async function compress(input, output) {
  const gzip = zlib.createGzip({ level: zlib.constants.Z_BEST_COMPRESSION });
  await pipeline(
    fs.createReadStream(input),
    gzip,
    fs.createWriteStream(output)
  );
}

// Decompress: input.txt.gz -> input.txt.restored
async function decompress(input, output) {
  await pipeline(
    fs.createReadStream(input),
    zlib.createGunzip(),
    fs.createWriteStream(output)
  );
}

(async () => {
  await compress('./input.txt', './input.txt.gz');
  await decompress('./input.txt.gz', './input.txt.restored');
  console.log('Round trip complete');
})().catch(console.error);
```

> TIP: `zlib` also offers Brotli (`createBrotliCompress`/`createBrotliDecompress`) which usually compresses better than gzip for text — handy for HTTP responses to clients that send `Accept-Encoding: br`.

### Example 3 — Streaming an HTTP response (with compression negotiation)

This server streams a file to the client, compresses on the fly if the client supports it, and cleans up correctly if the client disconnects mid-download.

```js
const fs = require('node:fs');
const http = require('node:http');
const zlib = require('node:zlib');
const { pipeline } = require('node:stream');

http.createServer((req, res) => {
  const filePath = './large-report.csv';
  const source = fs.createReadStream(filePath);

  // If the file is missing, respond before streaming starts.
  source.on('error', (err) => {
    if (err.code === 'ENOENT') {
      res.statusCode = 404;
      res.end('Not found');
    } else {
      res.statusCode = 500;
      res.end('Server error');
    }
  });

  const acceptsGzip = /\bgzip\b/.test(req.headers['accept-encoding'] || '');
  res.setHeader('Content-Type', 'text/csv');

  if (acceptsGzip) {
    res.setHeader('Content-Encoding', 'gzip');
    pipeline(source, zlib.createGzip(), res, (err) => {
      if (err) console.error('Stream aborted:', err.message); // e.g., client disconnect
    });
  } else {
    pipeline(source, res, (err) => {
      if (err) console.error('Stream aborted:', err.message);
    });
  }
}).listen(3000, () => console.log('Listening on :3000'));
```

> BEST PRACTICE: Using `pipeline(source, res, ...)` means that if the client aborts the request, Node destroys `source` and releases the file descriptor. With bare `source.pipe(res)`, you'd have to manually listen for `res`'s `close`/`aborted` and destroy `source` yourself — easy to forget.

### Example 4 — A small transform in a real pipeline (redact secrets)

```js
const { Transform } = require('node:stream');
const fs = require('node:fs');
const { pipeline } = require('node:stream/promises');

// Redacts anything that looks like an API key in a log file.
const redact = new Transform({
  transform(chunk, enc, cb) {
    const text = chunk.toString().replace(/sk-[A-Za-z0-9]{16,}/g, 'sk-[REDACTED]');
    cb(null, text);
  }
});

await pipeline(
  fs.createReadStream('./app.log'),
  redact,
  fs.createWriteStream('./app.redacted.log')
);
```

> WARNING — Pitfall: A naive line/token transform can break across chunk boundaries (a secret split between two chunks won't match). For correctness with patterns that may span chunks, buffer a tail like the `LineSplitter` example does, or split on a delimiter you control.

---

## Common Pitfalls & Gotchas

- **Unhandled `error` events crash the process.** Every stream must have an `error` handler — or use `pipeline`, which centralizes error handling for the whole chain.
- **Using `.pipe()` without manual cleanup leaks resources.** On error, `pipe()` does not destroy the other streams. Prefer `pipeline()`.
- **Ignoring `write()`'s return value defeats backpressure.** Memory grows unbounded when a fast producer outruns a slow consumer. Honor the `false` return and the `drain` event, or use `pipe`/`pipeline`/`for await`.
- **Forgetting the callback in `_write`/`_transform`.** The stream silently stalls — `finish` never fires. Call the callback exactly once on every path.
- **`Buffer.allocUnsafe` leaking old memory.** It returns uninitialized memory that may contain sensitive data. Default to `Buffer.alloc`.
- **`subarray()`/`slice()` returns a view, not a copy.** Mutating the slice mutates the original buffer. Wrap in `Buffer.from(...)` for an independent copy. (And remember `Buffer.slice` ≠ `Uint8Array.slice` semantics.)
- **Buffer length is bytes, not characters.** `Buffer.from('世').length === 3` in UTF-8. Don't use buffer length to count characters.
- **Decoding split multibyte characters.** Calling `.toString('utf8')` on a chunk that ends mid-character yields �. Use `StringDecoder` for streamed text.
- **O(n²) buffer concatenation.** Accumulate chunks in an array and `Buffer.concat` once; don't re-concat on every chunk.
- **`write()` after `end()`** throws `ERR_STREAM_WRITE_AFTER_END`.
- **Switching to flowing mode then not consuming** can lose data. If you attach `data`/`resume()`, you must process every chunk.
- **Mixing flowing and paused APIs** (`data` listeners + `.read()`/`readable` event) leads to confusing behavior. Pick one consumption style.

---

## Best Practices

- **Default to `stream.pipeline()` (promise form) for any multi-stream chain.** It gives you backpressure, error propagation, and automatic cleanup in one call.
- **Default to `for await...of` for consuming a single Readable** in application code — concise, backpressure-aware, and `try/catch`-friendly.
- **Always attach `error` handlers** (or rely on `pipeline`). Never leave a stream's `error` event unhandled.
- **Use `Buffer.alloc` by default; reserve `allocUnsafe` for measured hot paths** that immediately overwrite every byte.
- **Concatenate buffers once** via `Buffer.concat(chunks)` at the end.
- **Tune `highWaterMark` deliberately**, not by guesswork — larger marks trade memory for fewer I/O round trips; profile before changing the default.
- **Use object mode** when modeling logical records (rows, messages) rather than raw bytes.
- **Compress on the wire** with `zlib`/Brotli Transform streams when serving compressible content, negotiating via `Accept-Encoding`.
- **In HTTP handlers, pipe through `pipeline(source, res, cb)`** so client disconnects clean up the source.
- **Prefer stream consumers** (`stream/consumers`: `buffer`, `text`, `json`) when you must materialize a stream — they're correct and concise.
- **Use the `node:` import prefix** (`require('node:stream')`) to make built-in imports explicit and unambiguous.

---

## Key Interview Points

- **What is a Buffer and why does Node need it?** A fixed-length container of raw bytes (0–255), a subclass of `Uint8Array`, stored largely off the V8 heap. JavaScript strings/arrays can't efficiently represent binary I/O, so Node added `Buffer` for files, sockets, and encoding conversion.
- **`Buffer.alloc` vs. `allocUnsafe` vs. `from`?** `alloc(n)` returns zero-filled (safe, slower); `allocUnsafe(n)` returns uninitialized memory (fast, may leak old data — overwrite fully before use); `from(...)` creates a buffer from existing data (string/array/buffer), copying or wrapping the source.
- **Does `slice`/`subarray` copy or share memory?** They return a *view* sharing the underlying memory — mutating the slice mutates the original. Use `Buffer.from(buf.subarray(...))` to copy. Note Node's `Buffer.slice` differs from standard `Uint8Array.slice` (which copies).
- **What's the difference between utf8, hex, and base64?** Encodings mapping bytes↔text. utf8 is variable-width text; hex is 2 chars/byte for debugging; base64 packs arbitrary bytes into a text-safe subset (~33% larger), for transport in JSON/URLs — not encryption or compression.
- **Why do streams exist?** Constant, low memory footprint regardless of data size, and low time-to-first-byte/composability. Contrast `fs.readFile` (whole file in RAM, late first byte) with `createReadStream` (chunked, near-instant first byte, bounded memory).
- **Name the four stream types.** Readable (source), Writable (sink), Duplex (independent read+write, e.g., TCP socket), Transform (coupled — output is a function of input, e.g., gzip).
- **Flowing vs. paused mode?** Paused is the default — you pull with `.read()` on the `readable` event. Attaching a `data` listener, calling `.pipe()`, or `.resume()` switches to flowing (pushed via `data` events). `for await...of` manages it for you.
- **What is backpressure and how is it signaled?** The mechanism preventing a fast producer from overwhelming a slow consumer. `write()` returns `false` when the internal buffer exceeds `highWaterMark`; the producer pauses and resumes on the `drain` event. `pipe`/`pipeline`/`for await` implement this automatically.
- **Why prefer `pipeline()` over `pipe()`?** `pipe()` doesn't forward errors or destroy the other streams on failure, leaking file descriptors and risking process crashes from unhandled `error`. `pipeline()` propagates errors, destroys all streams on failure, and gives a single completion callback/promise.
- **What's `highWaterMark`?** The internal buffer threshold (default 64 KB for byte streams, 16 objects in object mode) that governs when `write()` returns `false` and when backpressure kicks in.
- **How do you build a custom Transform?** Extend `Transform` (or pass a `transform` fn to the constructor), implement `_transform(chunk, enc, cb)` calling `this.push()`/`cb(null, out)`, and optionally `_flush(cb)` for trailing output. Implementation methods are underscore-prefixed and invoked by the stream, not by you.
- **What happens if you forget the callback in `_write`/`_transform`?** The stream stalls after the first chunk — no further data, `finish` never fires, no error. The callback signals completion and drives backpressure.

---

## Practice Exercises

Try each before peeking at the solutions. They get progressively harder.

1. **Buffer basics.** Write a function `toBase64Hex(str)` that takes a UTF-8 string and returns an object `{ base64, hex }` with both encodings of the string's bytes. Then write the inverse `fromHex(hex)` that returns the original string.
2. **Manual stream collection.** Implement `streamToBuffer(readable)` that returns a Promise resolving to a single Buffer containing all of the stream's data — collecting chunks correctly (no O(n²) concatenation) and rejecting on error.
3. **Backpressure by hand.** Write `pumpWithBackpressure(readable, writable)` that copies a readable into a writable **without** using `pipe`/`pipeline`, correctly honoring `write()`'s return value and the `drain` event. Resolve a Promise when done; reject on error from either side.
4. **Custom Transform.** Build a Transform stream `CsvToObjects` that takes CSV text (first line is the header) and emits one plain object per data row in object mode. Demonstrate it with a `pipeline`.
5. **gzip CLI.** Write a small script `gz.js` that compresses a file given as `node gz.js <input> <output>` using `zlib` and `pipeline`, printing the compression ratio when done.
6. **(Challenge) Throttled Transform.** Build a Transform `Throttle(bytesPerSecond)` that passes data through but limits throughput to roughly the given rate (useful for simulating slow networks). Use it in a pipeline copying a file and observe that it respects backpressure.

### Solutions

#### Solution 1 — Buffer basics

```js
function toBase64Hex(str) {
  const buf = Buffer.from(str, 'utf8'); // encode the string to bytes
  return {
    base64: buf.toString('base64'),
    hex: buf.toString('hex')
  };
}

function fromHex(hex) {
  return Buffer.from(hex, 'hex').toString('utf8'); // decode bytes back to text
}

const enc = toBase64Hex('Hello, 世界');
console.log(enc);                 // { base64: '...', hex: '48656c6c6f2c20...' }
console.log(fromHex(enc.hex));    // 'Hello, 世界'
```

#### Solution 2 — Manual stream collection

```js
function streamToBuffer(readable) {
  return new Promise((resolve, reject) => {
    const chunks = [];
    readable.on('data', (chunk) => chunks.push(chunk)); // collect (no concat yet)
    readable.on('end', () => resolve(Buffer.concat(chunks))); // single concat = O(n)
    readable.on('error', reject);                       // propagate failures
  });
}

// Usage:
const fs = require('node:fs');
streamToBuffer(fs.createReadStream('./notes.txt'))
  .then((buf) => console.log(`Read ${buf.length} bytes`))
  .catch((err) => console.error(err));

// Modern alternative for completeness:
//   const { buffer } = require('node:stream/consumers');
//   const buf = await buffer(fs.createReadStream('./notes.txt'));
```

#### Solution 3 — Backpressure by hand

```js
function pumpWithBackpressure(readable, writable) {
  return new Promise((resolve, reject) => {
    let errored = false;
    const fail = (err) => {
      if (errored) return;
      errored = true;
      readable.destroy();
      writable.destroy();
      reject(err);
    };

    readable.on('error', fail);
    writable.on('error', fail);

    readable.on('data', (chunk) => {
      const ok = writable.write(chunk);   // false => buffer full
      if (!ok) {
        readable.pause();                 // stop reading
        writable.once('drain', () => readable.resume()); // resume when drained
      }
    });

    readable.on('end', () => writable.end()); // no more data → finish the sink
    writable.on('finish', () => { if (!errored) resolve(); });
  });
}

// Usage:
const fs = require('node:fs');
pumpWithBackpressure(
  fs.createReadStream('./big.bin'),
  fs.createWriteStream('./big.copy.bin')
).then(() => console.log('Copied with manual backpressure'))
 .catch(console.error);
```

#### Solution 4 — Custom Transform (CSV → objects)

```js
const { Transform } = require('node:stream');
const { Readable } = require('node:stream');
const { pipeline } = require('node:stream/promises');

class CsvToObjects extends Transform {
  constructor(options) {
    // input is text/buffer; output is objects → set readableObjectMode
    super({ ...options, readableObjectMode: true });
    this.tail = '';
    this.header = null;
  }

  _transform(chunk, encoding, callback) {
    const data = this.tail + chunk.toString();
    const lines = data.split('\n');
    this.tail = lines.pop(); // possibly partial last line

    for (const line of lines) {
      if (line.trim() === '') continue;
      const fields = line.split(',');
      if (!this.header) {
        this.header = fields;           // first non-empty line is the header
      } else {
        const obj = {};
        this.header.forEach((key, i) => { obj[key.trim()] = (fields[i] ?? '').trim(); });
        this.push(obj);                 // emit one object per row
      }
    }
    callback();
  }

  _flush(callback) {
    if (this.tail.trim() && this.header) {
      const fields = this.tail.split(',');
      const obj = {};
      this.header.forEach((key, i) => { obj[key.trim()] = (fields[i] ?? '').trim(); });
      this.push(obj);                   // emit the final buffered row
    }
    callback();
  }
}

// Demo:
const csv = 'id,name,role\n1,Ada,Engineer\n2,Linus,Maintainer\n3,Grace,Admiral';
(async () => {
  const collector = new Transform({
    objectMode: true,
    transform(obj, enc, cb) { console.log(obj); cb(); } // print each row-object
  });
  await pipeline(Readable.from([csv]), new CsvToObjects(), collector);
})().catch(console.error);
// { id: '1', name: 'Ada', role: 'Engineer' }
// { id: '2', name: 'Linus', role: 'Maintainer' }
// { id: '3', name: 'Grace', role: 'Admiral' }
```

#### Solution 5 — gzip CLI

```js
// gz.js — usage: node gz.js <input> <output>
const fs = require('node:fs');
const zlib = require('node:zlib');
const { pipeline } = require('node:stream/promises');

async function main() {
  const [input, output] = process.argv.slice(2);
  if (!input || !output) {
    console.error('Usage: node gz.js <input> <output>');
    process.exit(1);
  }

  await pipeline(
    fs.createReadStream(input),
    zlib.createGzip(),
    fs.createWriteStream(output)
  );

  const inSize = fs.statSync(input).size;
  const outSize = fs.statSync(output).size;
  const ratio = ((1 - outSize / inSize) * 100).toFixed(1);
  console.log(`Compressed ${inSize} → ${outSize} bytes (${ratio}% smaller)`);
}

main().catch((err) => {
  console.error('Failed:', err.message);
  process.exit(1);
});
```

#### Solution 6 — Throttled Transform (challenge)

```js
const { Transform } = require('node:stream');
const fs = require('node:fs');
const { pipeline } = require('node:stream/promises');

// Passes data through but limits throughput to ~bytesPerSecond.
// By delaying the callback, it naturally exerts backpressure upstream.
class Throttle extends Transform {
  constructor(bytesPerSecond, options) {
    super(options);
    this.bytesPerSecond = bytesPerSecond;
  }

  _transform(chunk, encoding, callback) {
    // Time this chunk "should" take at the target rate.
    const delayMs = (chunk.length / this.bytesPerSecond) * 1000;
    this.push(chunk);
    // Delaying the callback delays acceptance of the next chunk, which
    // (via the stream machinery) slows the upstream source. That's backpressure.
    setTimeout(callback, delayMs);
  }
}

// Demo: copy a file at ~64 KB/s.
(async () => {
  console.time('throttled-copy');
  await pipeline(
    fs.createReadStream('./input.bin'),
    new Throttle(64 * 1024), // 64 KB/s
    fs.createWriteStream('./input.throttled.bin')
  );
  console.timeEnd('throttled-copy');
})().catch(console.error);
```

> TIP: The throttle works *because* a Transform won't accept the next chunk until you call `callback()`. Delaying the callback delays acceptance, which propagates backpressure all the way up to the file read stream — a neat demonstration that backpressure is end-to-end across a pipeline.

---

## Summary / Cheat Sheet

**Buffers**

| Task | API |
| --- | --- |
| From string/array/buffer | `Buffer.from(src[, encoding])` |
| Zero-filled (safe) | `Buffer.alloc(size[, fill[, enc]])` |
| Uninitialized (fast) | `Buffer.allocUnsafe(size)` (then overwrite!) |
| Join buffers | `Buffer.concat([a, b], totalLen?)` |
| View (shares memory) | `buf.subarray(start, end)` |
| Copy of a slice | `Buffer.from(buf.subarray(start, end))` |
| Convert to text | `buf.toString('utf8' | 'hex' | 'base64')` |
| Read/write numbers | `readUInt32BE`, `writeUInt32LE`, `readBigUInt64BE`, ... |
| Safe streamed text decode | `new StringDecoder('utf8')` |

**Streams**

| Concept | Key points |
| --- | --- |
| Why streams | Constant memory + low time-to-first-byte; handle data of any size |
| Readable | Source; `data`/`end`/`error`; paused (default, `.read()`) vs. flowing |
| Writable | Sink; `write()` (returns `false` when full), `end()`, `drain`, `finish` |
| Duplex | Independent read + write (e.g., TCP socket) |
| Transform | Output = f(input) (e.g., gzip); `_transform` + optional `_flush` |
| Backpressure | `write()===false` → pause → wait for `drain` → resume |
| Consume cleanly | `for await (const chunk of readable)` |
| Connect streams | Prefer `pipeline(...)` over `.pipe(...)` |
| `pipeline` wins | Forwards errors, destroys all streams on failure, one completion signal |
| Custom impls | `_read`/`push(null)`, `_write(...,cb)`, `_transform(...,cb)` — always call `cb` |
| `highWaterMark` | Buffer threshold: 64 KB bytes / 16 objects (default) |

**The two golden rules**

1. For any multi-stream chain in production, use **`stream.pipeline()`** (promise form) — never bare `pipe()`.
2. **Always handle errors and respect backpressure.** Unhandled `error` crashes the process; ignored backpressure blows up memory. `pipeline` and `for await...of` give you both for free.

You now understand how Node represents binary data and how it moves that data efficiently through your application. These are the foundations beneath file handling, HTTP bodies, sockets, compression, and nearly every high-performance Node service you'll build.
