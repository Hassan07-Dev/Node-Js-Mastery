# The HTTP Module: Building Servers Without a Framework

You already know how to write JavaScript. What you have not yet done is make a JavaScript program **listen on a port, speak the HTTP protocol, and answer requests from the outside world**. That is precisely what this module is about. We are going to build real, working web servers using *nothing but* Node's built-in `http` module — no Express, no frameworks, no magic.

Why bother, when Express exists and is one `npm install` away? Because frameworks are abstractions *over* this layer. If you only ever learn Express, you learn a dialect. If you learn the raw `http` module first, you learn the language Express is written in. Every `req.body`, every `res.json()`, every router and middleware you will ever use is just sugar over the primitives in this module. By the end you will understand exactly what Express does for you — and why, after building a server by hand, you will *want* it.

## What You'll Learn

By the end of this module you will be able to:

- Explain the **HTTP request/response model** from first principles: methods, status codes, headers, and MIME types.
- Create a server with **`http.createServer`** and understand the `(req, res)` request handler.
- Read everything off the **`req` (`IncomingMessage`)** object: `method`, `url`, `headers`, and the **body as a stream**.
- Drive the **`res` (`ServerResponse`)** object: `statusCode`, `setHeader`/`writeHead`, `write`, and `end`.
- Build a **manual router** by inspecting the method and URL of each request.
- **Parse query strings** with the `URL` class and **parse JSON request bodies** by hand.
- **Serve JSON, HTML, and static files** correctly, with the right `Content-Type` headers.
- Handle **404s and errors** robustly so a single bad request never crashes the server.
- Make **outbound HTTP requests** with `http.request`/`http.get` and the `https` module.
- Articulate **why frameworks like Express exist** — the exact pain points this module exposes.

---

## 1. HTTP Protocol Basics: The Concept and the WHY

Before we write a single line of server code, you must internalize what HTTP actually *is*. A web server is a program that speaks a protocol; if you do not understand the protocol, the API will feel arbitrary.

### 1.1 HTTP is a request/response, text-based protocol over TCP

HTTP (HyperText Transfer Protocol) is the agreement that browsers, mobile apps, and other servers use to talk to your server. Its model is dead simple and worth committing to memory:

1. A **client** opens a connection to your server (over TCP, usually on port 80 for `http://` or 443 for `https://`).
2. The client sends a **request**.
3. The server sends back exactly one **response**.
4. (In HTTP/1.1, the TCP connection is often kept alive and reused for more request/response pairs.)

HTTP is **stateless**: each request is independent and the server remembers nothing between them by default. Any notion of "logged-in user" or "shopping cart" is layered *on top* of HTTP using cookies, tokens, or sessions — the protocol itself has no memory.

A raw HTTP/1.1 request is just text sent over the socket. If you could peek at the bytes, a `POST` would look like this:

```http
POST /api/users HTTP/1.1
Host: localhost:3000
Content-Type: application/json
Content-Length: 27

{"name":"Ada","age":36}
```

Three parts, always in this order:

1. **Request line** — `METHOD path HTTP/version` (`POST /api/users HTTP/1.1`).
2. **Headers** — one `Key: Value` per line (metadata about the request).
3. A **blank line**, then the optional **body** (here, the JSON payload).

The response has the exact same shape:

```http
HTTP/1.1 201 Created
Content-Type: application/json
Content-Length: 38

{"id":1,"name":"Ada","age":36}
```

1. **Status line** — `HTTP/version statusCode reasonPhrase` (`HTTP/1.1 201 Created`).
2. **Headers** — metadata about the response.
3. Blank line, then the **body** (the actual payload).

> TIP: Node's `http` module parses this text for you on the way in and serializes it for you on the way out. The `req` object *is* the parsed request; the `res` object is how you build the response. Keep that text picture in your head and the API stops being mysterious.

### 1.2 HTTP methods (verbs)

The **method** declares the *intent* of a request. The server is responsible for honoring that intent; HTTP only defines the conventions.

| Method | Intent | Has body? | Idempotent? | Safe? |
| --- | --- | --- | --- | --- |
| `GET` | Retrieve a resource | No (by convention) | Yes | Yes |
| `POST` | Create a resource / submit data | Yes | No | No |
| `PUT` | Replace a resource entirely | Yes | Yes | No |
| `PATCH` | Partially update a resource | Yes | No | No |
| `DELETE` | Remove a resource | Sometimes | Yes | No |
| `HEAD` | Like `GET` but headers only, no body | No | Yes | Yes |
| `OPTIONS` | Ask what methods/headers are allowed (CORS preflight) | No | Yes | Yes |

Two properties you will be asked about in interviews:

- **Safe** means the request should not change server state (`GET`, `HEAD`, `OPTIONS`). A `GET` must never delete data.
- **Idempotent** means making the request N times has the same effect as making it once. `PUT` and `DELETE` are idempotent (deleting the same resource twice leaves it deleted); `POST` is not (posting twice creates two resources).

> WARNING — Pitfall: HTTP does not *enforce* these semantics. You could write a `GET` handler that deletes records — the protocol won't stop you. But proxies, browsers, and crawlers *assume* `GET` is safe and may pre-fetch it, so a state-changing `GET` is a genuine production bug, not just bad style.

### 1.3 Status codes

The status code is a three-digit number summarizing the outcome. They are grouped by first digit:

| Range | Class | Meaning | Common examples |
| --- | --- | --- | --- |
| `1xx` | Informational | Request received, continuing | `100 Continue`, `101 Switching Protocols` |
| `2xx` | Success | The request succeeded | `200 OK`, `201 Created`, `204 No Content` |
| `3xx` | Redirection | Further action needed | `301 Moved Permanently`, `302 Found`, `304 Not Modified` |
| `4xx` | Client error | The *client* made a mistake | `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `405 Method Not Allowed`, `409 Conflict`, `422 Unprocessable Entity`, `429 Too Many Requests` |
| `5xx` | Server error | The *server* failed | `500 Internal Server Error`, `502 Bad Gateway`, `503 Service Unavailable`, `504 Gateway Timeout` |

A few distinctions that trip people up:

- `401 Unauthorized` actually means **unauthenticated** ("I don't know who you are"). `403 Forbidden` means **authenticated but not allowed** ("I know who you are, and you can't do this").
- `200 OK` vs `201 Created` vs `204 No Content`: use `201` when a `POST`/`PUT` creates something (and return it), and `204` when there is genuinely nothing to send back (e.g. a successful `DELETE`).
- `301` (permanent) vs `302`/`307` (temporary) redirects affect browser caching and SEO — pick deliberately.

> BEST PRACTICE: Return the *most specific accurate* status code. A validation failure is `400` (or `422`), not `500`. A missing record is `404`, not `200` with `{ "error": "not found" }`. Clients and monitoring tools rely on the status code being honest.

### 1.4 Headers

Headers are `Key: Value` metadata. They are **case-insensitive** keys (Node lowercases incoming header names for you). The ones you will use constantly:

| Header | Direction | Purpose |
| --- | --- | --- |
| `Content-Type` | Both | The MIME type of the body (`application/json`, `text/html`, …) |
| `Content-Length` | Both | Size of the body in bytes |
| `Accept` | Request | What MIME types the client can handle |
| `Authorization` | Request | Credentials, e.g. `Bearer <token>` |
| `Cookie` / `Set-Cookie` | Request / Response | Carry/set cookies |
| `Host` | Request | Which virtual host the request is for (required in HTTP/1.1) |
| `User-Agent` | Request | Identifies the client software |
| `Location` | Response | Target URL for a `3xx` redirect |
| `Cache-Control` | Response | Caching directives |

### 1.5 MIME types (media types)

A **MIME type** (a.k.a. media type or content type) tells the recipient how to interpret the bytes in the body. The format is `type/subtype`, e.g. `text/html`. The browser uses `Content-Type` to decide whether to render HTML, run JavaScript, display an image, or trigger a download.

| Extension | MIME type |
| --- | --- |
| `.html` | `text/html` |
| `.css` | `text/css` |
| `.js` | `text/javascript` (or `application/javascript`) |
| `.json` | `application/json` |
| `.png` | `image/png` |
| `.jpg` / `.jpeg` | `image/jpeg` |
| `.svg` | `image/svg+xml` |
| `.txt` | `text/plain` |
| (unknown binary) | `application/octet-stream` |

> WARNING — Pitfall: If you send HTML with `Content-Type: text/plain`, the browser shows the raw `<h1>…</h1>` tags as text instead of rendering them. If you send JSON without `Content-Type: application/json`, many clients will treat it as a string. The single most common "why isn't this working?" bug in raw HTTP servers is a wrong or missing `Content-Type`. For text formats it is also good practice to declare the charset: `text/html; charset=utf-8`.

---

## 2. `http.createServer` and the Request Handler

Now we write a server. The `http` module's `createServer` function returns an `http.Server` instance. You give it a **request handler** — a function called once per incoming request with two arguments: the request (`req`) and the response (`res`).

```js
// server.js — the smallest possible HTTP server.
const http = require('node:http'); // built-in; no install needed

// The request handler runs for EVERY incoming request.
const server = http.createServer((req, res) => {
  // res.end() finishes the response. Whatever you pass becomes the body.
  res.end('Hello, world!');
});

// Start listening. The callback fires once the server is ready.
server.listen(3000, () => {
  console.log('Server running at http://localhost:3000');
});
```

Run it with `node server.js`, then visit `http://localhost:3000`. Every request — for `/`, `/anything`, with any method — gets `Hello, world!`. We have not yet inspected the request, so the server treats them all identically.

### 2.1 `http.Server` is an EventEmitter

`createServer((req, res) => {...})` is just a shortcut for registering a listener on the server's `'request'` event. The following is exactly equivalent:

```js
const http = require('node:http');

const server = http.createServer(); // no handler passed

// The 'request' event fires for every request — same as the createServer arg.
server.on('request', (req, res) => {
  res.end('Hello from the request event!');
});

// The server emits other lifecycle events too:
server.on('connection', (socket) => console.log('New TCP socket connected'));
server.on('close', () => console.log('Server closed'));
server.on('error', (err) => console.error('Server error:', err.message));

server.listen(3000);
```

Understanding that the server is an `EventEmitter` explains *why* the handler signature is what it is: it is an event listener, and the request/response objects are the event's arguments.

### 2.2 `server.listen` and binding to a port

`listen` can take a port, an optional host, and an optional callback:

```js
// Listen on port 3000, all interfaces, log when ready.
server.listen(3000, () => console.log('Listening on 3000'));

// Bind to a specific host (e.g. localhost only — not reachable externally).
server.listen(3000, '127.0.0.1', () => console.log('localhost only'));

// Port 0 asks the OS for a random free port — handy in tests.
const s = http.createServer().listen(0, () => {
  console.log('Got port', s.address().port);
});
```

> BEST PRACTICE: In real deployments read the port from the environment so the platform can assign it: `const PORT = process.env.PORT || 3000;`. Hardcoding the port breaks on most hosting providers.

> TIP: If you see `EADDRINUSE`, another process is already on that port. Find it with `lsof -i :3000` (macOS/Linux) and kill it, or pick another port. Always attach a `server.on('error', ...)` handler so this surfaces cleanly instead of as an uncaught exception.

---

## 3. The `req` Object (`IncomingMessage`)

The first argument to the handler is an instance of `http.IncomingMessage`. It is a **readable stream** representing the parsed incoming request. Here are the properties you read off it.

### 3.1 `req.method`, `req.url`, `req.headers`

```js
const http = require('node:http');

const server = http.createServer((req, res) => {
  console.log('Method:', req.method);   // 'GET', 'POST', ... (always uppercase)
  console.log('URL:', req.url);         // '/users?active=true' — path + query ONLY
  console.log('Headers:', req.headers); // a plain object, keys lowercased
  console.log('HTTP version:', req.httpVersion); // e.g. '1.1'

  res.end('Logged your request — check the server console.');
});

server.listen(3000, () => console.log('http://localhost:3000'));
```

Crucial details that interviewers love:

- **`req.url` is *not* the full URL.** It is only the path and query string (`/users?active=true`). The scheme and host are not in `req.url` — the host lives in `req.headers.host`. To build an absolute URL you combine them (we do this in §6).
- **`req.headers` keys are lowercased.** Always access `req.headers['content-type']`, never `req.headers['Content-Type']`. HTTP header names are case-insensitive, and Node normalizes them to lowercase.
- **`req.method` is always uppercase.** Compare against `'GET'`, `'POST'`, etc.

### 3.2 Reading the request body as a stream

This is the single most important — and most surprising — fact about the `req` object:

> **The body does NOT arrive with the request handler.** `req.method`, `req.url`, and `req.headers` are available immediately (they come from the request line and headers, which Node has already parsed). But the **body** is delivered asynchronously, in chunks, as a **stream**. You must read it yourself.

Why? Bodies can be huge (a file upload, a video). Node does not buffer the entire body into memory before calling your handler — that would be wasteful and dangerous. Instead it hands you a readable stream and lets *you* decide how to consume it.

`req` (being a readable stream) emits:

- `'data'` — fired repeatedly, once per chunk of bytes (a `Buffer`).
- `'end'` — fired once, when the body is fully received.
- `'error'` — fired if the connection breaks mid-read.

The classic pattern to collect a full body:

```js
const http = require('node:http');

const server = http.createServer((req, res) => {
  // Only POST/PUT/PATCH typically carry a body.
  if (req.method === 'POST') {
    const chunks = []; // collect Buffer chunks here

    // 'data' fires for each chunk as bytes stream in.
    req.on('data', (chunk) => {
      chunks.push(chunk); // chunk is a Buffer
    });

    // 'end' fires once the whole body has arrived.
    req.on('end', () => {
      const body = Buffer.concat(chunks).toString('utf8'); // assemble + decode
      console.log('Received body:', body);
      res.end(`You sent ${body.length} characters.`);
    });

    // Always handle stream errors (e.g. client disconnects mid-upload).
    req.on('error', (err) => {
      console.error('Request stream error:', err.message);
      res.statusCode = 400;
      res.end('Bad request');
    });
  } else {
    res.end('Send a POST request with a body.');
  }
});

server.listen(3000, () => console.log('http://localhost:3000'));
```

Test it:

```bash
curl -X POST http://localhost:3000 -d 'hello server'
# -> You sent 12 characters.
```

> WARNING — Pitfall: A beginner expectation is that `req.body` exists. **It does not in raw Node.** There is no `req.body` until you build one by reading the stream. (Express adds `req.body` *only* because middleware like `express.json()` does exactly the stream-reading shown above and attaches the result. This is one of the first pains that motivates a framework.)

> WARNING — Pitfall (security): Concatenating an unbounded body lets a malicious client send gigabytes and exhaust your memory. Always cap the size:

```js
const MAX_BODY = 1e6; // 1 MB limit
let size = 0;
const chunks = [];
req.on('data', (chunk) => {
  size += chunk.length;
  if (size > MAX_BODY) {
    res.statusCode = 413; // Payload Too Large
    res.end('Body too large');
    req.destroy(); // stop reading; tear down the socket
    return;
  }
  chunks.push(chunk);
});
```

### 3.3 A modern, promise-based body reader

Streams support `async`/`await` via async iteration. This is the clean, modern way to read a body and is worth memorizing:

```js
// Read and return the entire request body as a string (with a size cap).
async function readBody(req, maxBytes = 1e6) {
  let size = 0;
  const chunks = [];
  // req is an async-iterable readable stream.
  for await (const chunk of req) {
    size += chunk.length;
    if (size > maxBytes) {
      const err = new Error('Payload too large');
      err.statusCode = 413;
      throw err;
    }
    chunks.push(chunk);
  }
  return Buffer.concat(chunks).toString('utf8');
}

// Usage inside a handler:
const http = require('node:http');
http.createServer(async (req, res) => {
  try {
    const body = await readBody(req);
    res.end(`Body length: ${body.length}`);
  } catch (err) {
    res.statusCode = err.statusCode || 500;
    res.end(err.message);
  }
}).listen(3000);
```

---

## 4. The `res` Object (`ServerResponse`)

The second handler argument is an `http.ServerResponse` instance — a **writable stream** representing the response you are building. You set the status and headers, write the body, and finish.

### 4.1 The order of operations matters

An HTTP response must be sent in order: status line, then headers, then body. Node enforces this. **Once any part of the body (or the headers) has been sent, you cannot change the status code or headers.** The point of no return is called sending the *head*.

```js
res.statusCode = 200;        // (1) set status   — only before head is sent
res.setHeader('X-Foo', 'bar'); // (2) set headers — only before head is sent
res.write('chunk one');      // (3) writing body implicitly flushes the head
res.setHeader('X-Late', 'no'); // ❌ THROWS: "Cannot set headers after they are sent"
res.end('done');             // (4) finish the response
```

### 4.2 `res.statusCode`

```js
res.statusCode = 404;            // set the numeric status
res.statusMessage = 'Not Found'; // optional; Node fills a sensible default
res.end('Resource not found');
```

### 4.3 `res.setHeader` vs `res.writeHead`

There are two ways to set headers. They are not interchangeable in timing.

- **`res.setHeader(name, value)`** — set one header at a time. Can be called multiple times. Headers are buffered and sent when the head flushes. You can read/overwrite them (`res.getHeader`, `res.removeHeader`) until then.
- **`res.writeHead(statusCode[, statusMessage][, headers])`** — set the status code *and* a batch of headers, and **flush the head immediately**. After `writeHead`, you can no longer `setHeader`.

```js
// Style A — setHeader (flexible, buffered):
res.statusCode = 200;
res.setHeader('Content-Type', 'application/json');
res.setHeader('Cache-Control', 'no-store');
res.end(JSON.stringify({ ok: true }));

// Style B — writeHead (one shot, flushes immediately):
res.writeHead(200, {
  'Content-Type': 'application/json',
  'Cache-Control': 'no-store',
});
res.end(JSON.stringify({ ok: true }));
```

| | `setHeader` | `writeHead` |
| --- | --- | --- |
| Sets one header or many? | One at a time | Many at once |
| Sets the status code? | No (use `res.statusCode`) | Yes |
| When are headers sent? | Buffered until head flushes | Immediately |
| Can you change a header afterwards? | Yes (until flush) | No (already sent) |
| Can you mix them? | Headers set via `setHeader` *before* `writeHead` are merged in | — |

> BEST PRACTICE: Use `setHeader` while you are still "deciding" the response (e.g. setting common headers early), and finish with `writeHead(status)` or just `res.statusCode = …; res.end(...)`. Mixing is fine: any `setHeader` calls made before `writeHead` are merged, and `writeHead`'s object wins on conflicts.

### 4.4 `res.write` and `res.end`

- **`res.write(chunk)`** — append a chunk to the body. Can be called many times (useful for streaming). The first `write` flushes the head.
- **`res.end([chunk])`** — finish the response, optionally writing a final chunk. After `end`, the response is complete and cannot be written to again.

```js
const http = require('node:http');

http.createServer((req, res) => {
  res.setHeader('Content-Type', 'text/plain; charset=utf-8');
  res.write('First line\n');  // streamed out
  res.write('Second line\n'); // streamed out
  res.end('Last line\n');     // final chunk + finish
}).listen(3000);
```

> WARNING — Pitfall: **You must always call `res.end()`** (directly, or implicitly via `res.end(body)`). If you forget, the client hangs waiting for a response that never finishes — the request appears to "load forever" and eventually times out. Every code path in your handler, including error paths, must end the response exactly once.

> WARNING — Pitfall: Calling `res.end()` twice, or `res.write()` after `res.end()`, throws `ERR_STREAM_WRITE_AFTER_END`. Guard branches with early `return` so you never fall through into a second `end`.

### 4.5 Streaming a response with a pipe

Because `res` is a writable stream, you can `pipe` a readable stream straight into it — the right way to serve large files without loading them into memory:

```js
const http = require('node:http');
const fs = require('node:fs');

http.createServer((req, res) => {
  res.setHeader('Content-Type', 'video/mp4');
  // Stream the file directly to the client, chunk by chunk.
  const stream = fs.createReadStream('./big-video.mp4');
  stream.pipe(res); // pipe handles write() + end() + backpressure for you
  stream.on('error', (err) => {
    res.statusCode = 500;
    res.end('Could not read file');
  });
}).listen(3000);
```

---

## 5. Building a Manual Router

So far every request gets the same response. A real server routes: it inspects `req.method` and `req.url` and dispatches to different logic. There is no router built in — *you* are the router. This is just `if`/`switch` over the method and path.

```js
const http = require('node:http');

const server = http.createServer((req, res) => {
  const { method, url } = req;

  // Route: GET /
  if (method === 'GET' && url === '/') {
    res.setHeader('Content-Type', 'text/plain');
    return res.end('Home page'); // return to avoid falling through
  }

  // Route: GET /about
  if (method === 'GET' && url === '/about') {
    res.setHeader('Content-Type', 'text/plain');
    return res.end('About page');
  }

  // Route: POST /contact
  if (method === 'POST' && url === '/contact') {
    res.statusCode = 201;
    return res.end('Message received');
  }

  // Distinguish "wrong method on a known path" (405) from "unknown path" (404).
  if (url === '/about' || url === '/') {
    res.statusCode = 405; // Method Not Allowed
    res.setHeader('Allow', 'GET');
    return res.end('Method Not Allowed');
  }

  // Fallthrough: nothing matched -> 404.
  res.statusCode = 404;
  res.setHeader('Content-Type', 'text/plain');
  res.end('Not Found');
});

server.listen(3000, () => console.log('http://localhost:3000'));
```

> TIP: `return res.end(...)` is an idiom, not because `end` returns anything useful, but because the `return` stops execution so you do not accidentally fall through to a later branch and try to end the response twice.

### 5.1 A cleaner table-driven router

A chain of `if`s gets unwieldy fast. A small abstraction — a map of `"METHOD path"` keys to handler functions — is far more maintainable and previews how real routers work internally:

```js
const http = require('node:http');

// A registry of exact-match routes.
const routes = {
  'GET /': (req, res) => res.end('Home'),
  'GET /about': (req, res) => res.end('About'),
  'POST /contact': (req, res) => {
    res.statusCode = 201;
    res.end('Message received');
  },
};

const server = http.createServer((req, res) => {
  // Strip the query string for matching: '/about?x=1' -> '/about'.
  const path = req.url.split('?')[0];
  const key = `${req.method} ${path}`;

  const handler = routes[key];
  if (handler) {
    return handler(req, res);
  }

  res.statusCode = 404;
  res.end('Not Found');
});

server.listen(3000, () => console.log('http://localhost:3000'));
```

### 5.2 Dynamic route parameters (e.g. `/users/:id`)

Real APIs have parameterized paths like `/users/42`. Exact-string matching cannot handle these — you need a regular expression or a path-segment comparison. This is where doing it by hand starts to hurt:

```js
const http = require('node:http');

const server = http.createServer((req, res) => {
  const path = req.url.split('?')[0];

  // Match /users/:id where id is one or more digits.
  const userMatch = path.match(/^\/users\/(\d+)$/);
  if (req.method === 'GET' && userMatch) {
    const id = userMatch[1]; // the captured group
    res.setHeader('Content-Type', 'application/json');
    return res.end(JSON.stringify({ id: Number(id), name: `User ${id}` }));
  }

  res.statusCode = 404;
  res.end('Not Found');
});

server.listen(3000, () => console.log('http://localhost:3000'));
```

> WARNING — Pitfall: Notice how quickly this grows. Every parameterized route needs its own hand-written regex, you must extract and decode params yourself, and ordering matters. Multiply this by 50 routes and you understand exactly why `app.get('/users/:id', handler)` exists. **This is the pain Express's router solves.**

---

## 6. Parsing Query Strings and JSON Bodies

Two of the most common things a server does are reading query parameters and reading JSON bodies. Neither is automatic in raw Node.

### 6.1 Parsing query strings with the `URL` class

`req.url` includes the query string but does not parse it. The modern, recommended tool is the WHATWG **`URL`** class (a global in Node; no import needed), which gives you a `searchParams` object.

Because `req.url` is relative (no scheme/host), you must provide a base when constructing the `URL`:

```js
const http = require('node:http');

const server = http.createServer((req, res) => {
  // Build an absolute URL: combine req.url with scheme + host header.
  const fullUrl = new URL(req.url, `http://${req.headers.host}`);

  const pathname = fullUrl.pathname;            // '/search' (no query)
  const params = fullUrl.searchParams;          // a URLSearchParams object

  const q = params.get('q');                     // ?q=node -> 'node'
  const page = params.get('page') || '1';        // default if missing
  const tags = params.getAll('tag');             // ?tag=a&tag=b -> ['a','b']

  res.setHeader('Content-Type', 'application/json');
  res.end(JSON.stringify({ pathname, q, page, tags }));
});

server.listen(3000, () => console.log('http://localhost:3000'));
```

```bash
curl 'http://localhost:3000/search?q=node&page=2&tag=a&tag=b'
# -> {"pathname":"/search","q":"node","page":"2","tags":["a","b"]}
```

Key facts about `URLSearchParams`:

- `.get(key)` returns the **first** value or `null`. `.getAll(key)` returns **all** values as an array (for repeated keys).
- `.has(key)`, `.entries()`, and direct iteration (`for (const [k, v] of params)`) are all available.
- Values are **already URL-decoded** for you (`%20` becomes a space).
- Everything is a **string** — convert numbers yourself with `Number(...)`.

> Note: The legacy `querystring` module (`querystring.parse`) and the older `url.parse()` still exist and you will see them in old code, but `url.parse` is **deprecated** in favor of the `URL` class. Use `URL`/`URLSearchParams` in new code.

### 6.2 Parsing a JSON request body manually

Combine the stream reading from §3 with `JSON.parse`. The critical part is wrapping `JSON.parse` in a try/catch — a client can always send malformed JSON, and an uncaught `SyntaxError` would crash your server.

```js
const http = require('node:http');

// Reusable helper: read the stream, then parse it as JSON.
async function readJsonBody(req, maxBytes = 1e6) {
  let size = 0;
  const chunks = [];
  for await (const chunk of req) {
    size += chunk.length;
    if (size > maxBytes) {
      const err = new Error('Payload too large');
      err.statusCode = 413;
      throw err;
    }
    chunks.push(chunk);
  }
  const raw = Buffer.concat(chunks).toString('utf8');
  if (raw.length === 0) return {}; // empty body -> empty object

  try {
    return JSON.parse(raw);
  } catch {
    const err = new Error('Invalid JSON');
    err.statusCode = 400; // malformed JSON is a CLIENT error
    throw err;
  }
}

const server = http.createServer(async (req, res) => {
  if (req.method === 'POST' && req.url === '/users') {
    try {
      // Defensive: only accept JSON.
      const type = req.headers['content-type'] || '';
      if (!type.includes('application/json')) {
        res.statusCode = 415; // Unsupported Media Type
        return res.end('Expected application/json');
      }

      const data = await readJsonBody(req);

      // Basic validation.
      if (!data.name) {
        res.statusCode = 400;
        return res.end(JSON.stringify({ error: 'name is required' }));
      }

      res.statusCode = 201;
      res.setHeader('Content-Type', 'application/json');
      return res.end(JSON.stringify({ id: 1, name: data.name }));
    } catch (err) {
      res.statusCode = err.statusCode || 500;
      return res.end(JSON.stringify({ error: err.message }));
    }
  }

  res.statusCode = 404;
  res.end('Not Found');
});

server.listen(3000, () => console.log('http://localhost:3000'));
```

```bash
curl -X POST http://localhost:3000/users \
  -H 'Content-Type: application/json' \
  -d '{"name":"Ada"}'
# -> {"id":1,"name":"Ada"}
```

> WARNING — Pitfall: Never call `JSON.parse(raw)` outside a try/catch on user input. A single `curl -d 'not json'` will throw a `SyntaxError`, and if it is unhandled in an async handler it can take the whole process down. **All this boilerplate — read the stream, cap the size, check the content type, parse safely — is exactly what `express.json()` does in one line. This is the second big pain that motivates a framework.**

### 6.3 Parsing URL-encoded form bodies

HTML `<form>` posts (without `enctype`) send bodies as `application/x-www-form-urlencoded` (`name=Ada&age=36`), not JSON. You parse those with `URLSearchParams`:

```js
const body = await readBody(req); // 'name=Ada&age=36'
const form = new URLSearchParams(body);
const name = form.get('name'); // 'Ada'
const age = form.get('age');   // '36' (string)
```

---

## 7. Serving JSON, HTML, and Static Files

The whole point of a web server is to return content. The recipe is always the same: set the right `Content-Type`, set the status, write the body, end. What changes is the body and the type.

### 7.1 Serving JSON

```js
const http = require('node:http');

http.createServer((req, res) => {
  const data = { id: 1, name: 'Ada', skills: ['node', 'http'] };

  res.statusCode = 200;
  res.setHeader('Content-Type', 'application/json; charset=utf-8');
  // JSON.stringify turns the object into the body string. res.end can't send objects.
  res.end(JSON.stringify(data));
}).listen(3000);
```

> WARNING — Pitfall: `res.end(someObject)` does **not** serialize for you — it throws (`res.end` accepts a string, Buffer, or Uint8Array, not a plain object). You must `JSON.stringify` yourself. Express's `res.json()` does the stringify *and* sets the content type — yet another convenience this module forces you to do by hand.

### 7.2 Serving HTML

```js
const http = require('node:http');

http.createServer((req, res) => {
  const html = `<!DOCTYPE html>
<html lang="en">
  <head><meta charset="utf-8"><title>Home</title></head>
  <body><h1>Hello from raw Node</h1></body>
</html>`;

  res.statusCode = 200;
  res.setHeader('Content-Type', 'text/html; charset=utf-8'); // so the browser RENDERS it
  res.end(html);
}).listen(3000);
```

### 7.3 Serving static files (by hand)

To serve files from disk, you read the file, pick the MIME type from its extension, and stream it back. Doing this *correctly and securely* is genuinely tricky — and that difficulty is part of the lesson.

```js
const http = require('node:http');
const fs = require('node:fs');
const path = require('node:path');

// Map file extensions to MIME types.
const MIME_TYPES = {
  '.html': 'text/html; charset=utf-8',
  '.css': 'text/css; charset=utf-8',
  '.js': 'text/javascript; charset=utf-8',
  '.json': 'application/json; charset=utf-8',
  '.png': 'image/png',
  '.jpg': 'image/jpeg',
  '.jpeg': 'image/jpeg',
  '.svg': 'image/svg+xml',
  '.ico': 'image/x-icon',
};

const PUBLIC_DIR = path.join(__dirname, 'public'); // the only directory we'll serve

const server = http.createServer((req, res) => {
  // Only GET/HEAD make sense for static files.
  if (req.method !== 'GET' && req.method !== 'HEAD') {
    res.statusCode = 405;
    res.setHeader('Allow', 'GET, HEAD');
    return res.end('Method Not Allowed');
  }

  // Decode and strip the query string.
  let pathname = decodeURIComponent(new URL(req.url, `http://${req.headers.host}`).pathname);
  if (pathname === '/') pathname = '/index.html'; // default document

  // SECURITY: resolve against PUBLIC_DIR and verify the result stays inside it.
  // This defeats path-traversal attacks like /../../etc/passwd.
  const filePath = path.join(PUBLIC_DIR, pathname);
  if (!filePath.startsWith(PUBLIC_DIR + path.sep)) {
    res.statusCode = 403;
    return res.end('Forbidden');
  }

  // Check the file exists and is a regular file (not a directory).
  fs.stat(filePath, (err, stats) => {
    if (err || !stats.isFile()) {
      res.statusCode = 404;
      res.setHeader('Content-Type', 'text/plain; charset=utf-8');
      return res.end('404 Not Found');
    }

    const ext = path.extname(filePath).toLowerCase();
    const mime = MIME_TYPES[ext] || 'application/octet-stream';

    res.statusCode = 200;
    res.setHeader('Content-Type', mime);
    res.setHeader('Content-Length', stats.size);

    if (req.method === 'HEAD') {
      return res.end(); // HEAD: headers only, no body
    }

    // Stream the file to the client (memory-efficient).
    const stream = fs.createReadStream(filePath);
    stream.pipe(res);
    stream.on('error', () => {
      // If we haven't sent the head yet, we can still send a 500.
      if (!res.headersSent) res.statusCode = 500;
      res.end('Server Error');
    });
  });
});

server.listen(3000, () => console.log('http://localhost:3000'));
```

> WARNING — Pitfall (security — path traversal): If you naively do `fs.readFile(PUBLIC_DIR + req.url)`, a request for `/../../../../etc/passwd` (or URL-encoded `%2e%2e%2f`) escapes your public directory and serves arbitrary files. The `path.join` + `startsWith(PUBLIC_DIR + path.sep)` check above is the minimum defense. **This is a real, exploited vulnerability class.** A serious static server also needs caching headers (`ETag`, `Last-Modified`, `Cache-Control`), range requests (for video seeking), and compression — all of which Express's `express.static` handles for you. Hand-rolling production-grade static serving is a project in itself; this is one of the strongest arguments for not reinventing the wheel.

---

## 8. Handling 404s and Errors

A server that crashes on the first bad request is useless. Robust error handling has two layers: **per-request** handling (return a sane status) and **process-level** safety (never let one request take down the whole server).

### 8.1 The 404 fallthrough

A 404 is simply "no route matched." Put it at the very end of your handler so it catches everything that fell through:

```js
// ... after all your route checks ...
res.statusCode = 404;
res.setHeader('Content-Type', 'application/json; charset=utf-8');
res.end(JSON.stringify({ error: 'Not Found', path: req.url }));
```

### 8.2 Wrapping handlers in try/catch

In an `async` handler, any thrown error or rejected promise must be caught, or it becomes an unhandled rejection. Wrap the body and translate errors into responses:

```js
const http = require('node:http');

async function handleRequest(req, res) {
  // ... routing + work that might throw ...
  throw Object.assign(new Error('Something broke'), { statusCode: 500 });
}

const server = http.createServer(async (req, res) => {
  try {
    await handleRequest(req, res);
  } catch (err) {
    console.error(err); // log the real error server-side

    // Guard: if we already started the response, we can't change the status.
    if (res.headersSent) {
      return res.destroy(); // best we can do — tear down the socket
    }
    res.statusCode = err.statusCode || 500;
    res.setHeader('Content-Type', 'application/json; charset=utf-8');
    // Don't leak internal error details to clients in production.
    res.end(JSON.stringify({ error: err.statusCode ? err.message : 'Internal Server Error' }));
  }
});

server.listen(3000);
```

> BEST PRACTICE: Log the full error server-side, but return a **generic** message to clients for `5xx` errors. Leaking stack traces or internal messages (`"ECONNREFUSED to db:5432"`) is an information-disclosure vulnerability. For `4xx` client errors, a specific message ("name is required") is helpful and safe.

### 8.3 The `'clientError'` event and stream errors

Some errors happen at the socket level before your handler even runs — a malformed request, a client that disconnects abruptly. Handle them on the server:

```js
// Malformed requests at the protocol level.
server.on('clientError', (err, socket) => {
  if (socket.writable) {
    socket.end('HTTP/1.1 400 Bad Request\r\n\r\n');
  }
});

// A 'request' stream that errors mid-flight.
server.on('request', (req) => {
  req.on('error', (err) => console.error('Request error:', err.message));
});
```

### 8.4 Process-level safety nets

These are last resorts, not primary error handling, but every production server should have them:

```js
process.on('uncaughtException', (err) => {
  console.error('Uncaught exception:', err);
  // The process is now in an unknown state. Log, then exit and let a
  // supervisor (pm2, systemd, Kubernetes) restart cleanly.
  process.exit(1);
});

process.on('unhandledRejection', (reason) => {
  console.error('Unhandled rejection:', reason);
  process.exit(1);
});
```

> WARNING — Pitfall: Do **not** treat `uncaughtException` as a way to "ignore" errors and keep running. After an uncaught exception the process may have leaked resources or be in a corrupt state. Log and restart. The goal is graceful failure, not pretending nothing happened.

---

## 9. Making Outbound HTTP Requests

Your server is also a *client*: it calls third-party APIs, microservices, webhooks. The same module that builds servers also makes outbound requests. (For HTTPS targets, the API is identical but you use the `https` module.)

### 9.1 `http.get` — the shortcut for GET requests

`http.get` is a convenience wrapper around `http.request` that sets the method to `GET` and calls `.end()` for you. The response is — predictably — a readable stream you must consume:

```js
const http = require('node:http');

http.get('http://example.com/', (res) => {
  console.log('Status:', res.statusCode);
  console.log('Headers:', res.headers);

  let data = '';
  res.setEncoding('utf8');           // decode chunks as strings
  res.on('data', (chunk) => (data += chunk)); // accumulate the response body
  res.on('end', () => {
    console.log('Body length:', data.length);
  });
}).on('error', (err) => {
  console.error('Request failed:', err.message);
});
```

> WARNING — Pitfall: The response callback fires when the **headers** arrive, not the body. Just like the server side, you must read the response stream (`'data'`/`'end'`) to get the body. Forgetting this is the single most common mistake with `http.get`.

### 9.2 `http.request` — the full-control API

For anything other than a plain GET (POST, custom headers, etc.), use `http.request`. You configure it with an options object, write the body, and **you must call `req.end()`** to actually send it:

```js
const http = require('node:http');

const payload = JSON.stringify({ name: 'Ada', age: 36 });

const options = {
  hostname: 'localhost',
  port: 3000,
  path: '/users',
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Content-Length': Buffer.byteLength(payload), // byte length, not string length
  },
};

const req = http.request(options, (res) => {
  let data = '';
  res.setEncoding('utf8');
  res.on('data', (chunk) => (data += chunk));
  res.on('end', () => {
    console.log('Status:', res.statusCode);
    console.log('Response:', data);
  });
});

req.on('error', (err) => console.error('Request error:', err.message));

req.write(payload); // send the body
req.end();          // REQUIRED: finalize and actually send the request
```

> WARNING — Pitfall: Use `Buffer.byteLength(payload)` for `Content-Length`, not `payload.length`. With multi-byte UTF-8 characters (emoji, accented letters), the string length and byte length differ, and a wrong `Content-Length` causes truncated or hung requests.

### 9.3 The `https` module

For `https://` URLs you use the `https` module. The API is identical — just swap `http` for `https`. The difference is TLS: the connection is encrypted and the server's certificate is verified.

```js
const https = require('node:https');

https.get('https://api.github.com/users/nodejs', {
  headers: { 'User-Agent': 'node-course-demo' }, // GitHub requires a User-Agent
}, (res) => {
  let data = '';
  res.setEncoding('utf8');
  res.on('data', (chunk) => (data += chunk));
  res.on('end', () => {
    const json = JSON.parse(data);
    console.log(`${json.name} has ${json.public_repos} public repos`);
  });
}).on('error', (err) => console.error(err.message));
```

> TIP: Choosing the right module by URL scheme is annoying. The `URL` object's `protocol` property (`'http:'` vs `'https:'`) lets you pick: `const lib = url.protocol === 'https:' ? https : http;`.

### 9.4 A promise-wrapped client

The callback + stream style is verbose. Here is how to wrap it into a clean promise — and this is exactly why you will eventually reach for the built-in global `fetch` (available since Node 18) or a library like `axios`/`undici`:

```js
const http = require('node:http');
const https = require('node:https');

// Promise-based GET that resolves to { statusCode, headers, body }.
function get(urlString) {
  return new Promise((resolve, reject) => {
    const url = new URL(urlString);
    const lib = url.protocol === 'https:' ? https : http;

    const req = lib.get(url, (res) => {
      let data = '';
      res.setEncoding('utf8');
      res.on('data', (chunk) => (data += chunk));
      res.on('end', () => resolve({ statusCode: res.statusCode, headers: res.headers, body: data }));
    });
    req.on('error', reject);
    // Always set a timeout so a hung server doesn't hang you forever.
    req.setTimeout(5000, () => req.destroy(new Error('Request timed out')));
  });
}

// Usage:
get('https://api.github.com/users/nodejs')
  .then((res) => console.log(res.statusCode))
  .catch((err) => console.error(err.message));
```

> TIP: For new code making outbound requests, prefer the global **`fetch`** API — it is built into Node 18+, returns promises, and matches the browser API you already know:
>
> ```js
> const res = await fetch('https://api.github.com/users/nodejs');
> const data = await res.json(); // fetch handles streaming + parsing for you
> ```
>
> Knowing `http.request` is still essential — it is what `fetch` and every HTTP library are built on, and it appears constantly in interviews and legacy code.

---

## 10. Why Frameworks Like Express Exist

Now we collect the bill. Look back at everything this module forced you to do **by hand** for a server that is still missing most production features:

| Task | Raw `http` | Express |
| --- | --- | --- |
| Routing by method + path | Manual `if`/`switch` over `req.method`/`req.url` | `app.get('/users/:id', handler)` |
| Route parameters (`/users/:id`) | Hand-written regex + extraction | Built into the router |
| Parsing JSON bodies | Read stream, cap size, check type, `JSON.parse` in try/catch | `express.json()` |
| Parsing form bodies | Read stream + `URLSearchParams` | `express.urlencoded()` |
| Sending JSON | `setHeader` + `JSON.stringify` + `end` | `res.json(obj)` |
| Status helpers | `res.statusCode = 404` | `res.status(404).send(...)` |
| Serving static files | Hand-rolled, security-sensitive | `express.static('public')` |
| Reusable cross-cutting logic (auth, logging) | Copy/paste into every handler | Middleware pipeline (`app.use`) |
| 404 / error handling | Manual fallthrough + try/catch everywhere | Centralized error middleware |
| Query parsing | `new URL(...).searchParams` | `req.query` (auto) |

Every Express feature is a thin, well-designed convenience over the `http` primitives you now understand:

- `app` is built around an `http.Server`. Under the hood, `app.listen()` calls `http.createServer(app).listen()`. Express *is* an `http` request handler.
- `req` and `res` in Express are the **same** `IncomingMessage`/`ServerResponse` objects you have been using — Express just augments them with extra methods (`req.params`, `res.json`, `res.status`) and properties.
- **Middleware** is the genuinely powerful idea: a pipeline of functions, each receiving `(req, res, next)`, that lets you compose cross-cutting concerns (body parsing, auth, logging, CORS) instead of copy-pasting them into every handler. The raw `http` module has no concept of this; you would have to build the pipeline yourself.

> BEST PRACTICE: Reach for Express (or Fastify, Koa, Hapi) for real applications — reinventing routing, body parsing, and secure static serving is wasted effort and a source of bugs (especially security bugs like the path traversal we saw). But never treat it as a black box. Because you have built a server by hand, you can debug Express when it misbehaves, understand its docs, write your own middleware, and reason about performance. **The framework is a tool; the protocol and the `http` module are the foundation.**

---

## Common Pitfalls & Gotchas

- **Forgetting `res.end()`.** Any code path that doesn't end the response leaves the client hanging until it times out. End the response exactly once on *every* branch, including error branches.
- **Expecting `req.body` to exist.** There is no `req.body` in raw Node — the body is a stream you must read via `'data'`/`'end'` (or `for await`). This is the number-one beginner surprise.
- **Setting headers after the head is sent.** `res.setHeader`/`res.writeHead` after the first `res.write` or `res.end` throws `ERR_HTTP_HEADERS_SENT`. Decide your status and headers *before* writing the body. Use `res.headersSent` to guard.
- **Wrong or missing `Content-Type`.** HTML shown as raw text, JSON treated as a string, downloads that should be inline — almost always a `Content-Type` bug. Include `; charset=utf-8` for text formats.
- **Mixed-case header access.** `req.headers['Content-Type']` is `undefined`; incoming header keys are lowercased. Use `req.headers['content-type']`.
- **`req.url` is not the full URL.** It is path + query only. Build absolute URLs with `new URL(req.url, 'http://' + req.headers.host)`.
- **`res.end(object)` doesn't serialize.** You must `JSON.stringify` objects yourself; `end` only accepts strings, Buffers, and Uint8Arrays.
- **Unbounded body buffering.** Concatenating chunks without a size cap is a denial-of-service vector. Always enforce a max body size and respond `413`.
- **Unguarded `JSON.parse`.** Malformed JSON throws a `SyntaxError`. Always parse user input inside try/catch and respond `400`.
- **Path traversal in static servers.** Naive path concatenation lets clients read files outside your public directory. Always resolve and verify the path stays inside the intended root.
- **`Content-Length` using `.length` instead of `Buffer.byteLength`.** Multi-byte characters make the two differ, causing truncated/hung requests.
- **Not reading the response body on outbound requests.** The `http.get`/`http.request` callback fires on headers; you must consume the response stream to get the body.
- **Forgetting `req.end()` on outbound `http.request`.** Without it, the request is never actually sent.

## Best Practices

- **Always handle every stream's `'error'` event** — `req`, `res`, file streams, and outbound requests. An unhandled stream error can crash the process.
- **Cap request body size** and reject oversized payloads with `413` before buffering.
- **Validate and sanitize all input** — query params, headers, and bodies are attacker-controlled.
- **Set explicit, correct `Content-Type` headers** (with charset for text). Never rely on client guessing.
- **Return honest, specific status codes.** `400`/`404`/`405`/`415`/`422` for client mistakes; `5xx` only for genuine server faults.
- **Log full errors server-side; return generic messages to clients** for `5xx`. Don't leak stack traces or internals.
- **Read `PORT` from the environment** (`process.env.PORT || 3000`).
- **Set timeouts on outbound requests** so a slow upstream can't hang your server.
- **Stream large responses** (`fs.createReadStream(...).pipe(res)`) instead of buffering whole files into memory.
- **Use the `node:` prefix** for built-in modules (`require('node:http')`) to disambiguate from npm packages.
- **Add process-level safety nets** (`uncaughtException`, `unhandledRejection`) that log and exit for a supervisor to restart — but rely on per-request handling as your primary defense.
- **Prefer the WHATWG `URL`/`URLSearchParams`** over the deprecated `url.parse`; prefer global `fetch` over hand-rolled clients for new outbound code.
- **For real apps, use a framework** — but only after understanding the layer beneath it.

## Key Interview Points

- **What does `http.createServer` return, and what is the handler signature?** It returns an `http.Server` (an `EventEmitter`). The handler is `(req, res)`, equivalent to `server.on('request', (req, res) => ...)`. `req` is an `IncomingMessage` (readable stream), `res` is a `ServerResponse` (writable stream).
- **Is `req.url` the full URL?** No — only the path and query string. The scheme isn't present and the host is in `req.headers.host`. Build an absolute URL with `new URL(req.url, 'http://' + req.headers.host)`.
- **How do you read a request body?** It arrives as a stream after the handler is invoked. Collect `'data'` chunks (Buffers), assemble on `'end'`, then decode/parse — or use `for await (const chunk of req)`. There is no `req.body` until you build one.
- **Why is reading the body async/streamed?** Bodies can be arbitrarily large; Node streams them so you control memory and can apply size limits, rather than buffering everything before your code runs.
- **`setHeader` vs `writeHead`?** `setHeader` sets one header at a time and buffers them; `writeHead` sets the status plus a batch of headers and flushes the head immediately. After the head is sent, neither can change headers (`ERR_HTTP_HEADERS_SENT`).
- **What's the order constraint on a response?** Status, then headers, then body. Once you `write` the body (or `writeHead`/`end`), the head is sent and the status/headers are frozen. `res.headersSent` tells you whether that has happened.
- **Idempotent vs safe methods?** Safe = no state change (`GET`, `HEAD`, `OPTIONS`). Idempotent = N calls equal one call (`GET`, `PUT`, `DELETE`, `HEAD`). `POST` is neither.
- **`401` vs `403`?** `401` = not authenticated (who are you?). `403` = authenticated but not permitted.
- **`200` vs `201` vs `204`?** `200` general success, `201` resource created (return it), `204` success with no body.
- **How do you parse query strings?** With the `URL` class: `new URL(req.url, base).searchParams`. `.get` returns the first value, `.getAll` returns all; values are auto-decoded strings.
- **How do you guard `JSON.parse` of a body?** Wrap it in try/catch and respond `400` on failure; never let a `SyntaxError` from client input crash the server.
- **`http.get` vs `http.request`?** `http.get` is a GET shortcut that auto-calls `.end()`. `http.request` is the general API where you set the method/headers, `write` the body, and must call `req.end()`. For HTTPS, use the `https` module (same API). In both, the response is a stream you must consume.
- **Why does Express exist?** It abstracts routing, route params, body parsing, JSON helpers, static serving, and (most importantly) a middleware pipeline over the raw `http` module. Express *is* an `http` request handler; `req`/`res` are the same objects, just augmented.
- **What's a path-traversal vulnerability in a static server?** Concatenating user-supplied paths lets a client request `../../etc/passwd` and escape the public directory. Defend by resolving the path and verifying it stays within the intended root.

## Practice Exercises

Work through these in order; each builds on the last. Try them yourself before reading the solutions.

1. **Echo server.** Build a server that, for any `POST /echo` request, reads the request body and sends it straight back in the response. For any other route, respond `404`.

2. **Method + path router.** Build a server with these routes: `GET /` returns the plain text `"Welcome"`; `GET /time` returns the current ISO timestamp as JSON (`{ "now": "..." }`); anything else returns a JSON `404`. Hitting a known path with the wrong method should return `405` with an `Allow` header.

3. **Query-driven greeting.** `GET /greet?name=Ada&times=3` should respond with `"Hello Ada! "` repeated `times` times. Default `name` to `"friend"` and `times` to `1`. Validate that `times` is a positive integer ≤ 10, returning `400` otherwise.

4. **JSON API with validation.** Build `POST /users` that accepts `{ "name": string, "age": number }`. Enforce `application/json`, cap the body at 1 MB, reject malformed JSON with `400`, reject missing/invalid fields with `422`, and on success return `201` with `{ "id": <n>, "name", "age" }` (use an in-memory counter for `id`). Also support `GET /users` returning all created users as a JSON array.

5. **Mini static file server.** Serve files from a `./public` directory. Default `/` to `index.html`, set correct `Content-Type` by extension, return `404` for missing files, and **defend against path traversal**. Bonus: support `HEAD`.

6. **Proxy / aggregator (outbound requests).** Build `GET /github/:user` that fetches `https://api.github.com/users/:user` server-side and returns a trimmed JSON object `{ login, name, public_repos }`. Handle upstream errors and a non-existent user (`404` from GitHub) gracefully, and set a request timeout.

### Solutions

**Solution 1 — Echo server**

```js
const http = require('node:http');

http.createServer((req, res) => {
  if (req.method === 'POST' && req.url === '/echo') {
    const chunks = [];
    req.on('data', (c) => chunks.push(c));
    req.on('end', () => {
      const body = Buffer.concat(chunks);
      // Echo the body back with whatever content type the client sent (or plain text).
      res.statusCode = 200;
      res.setHeader('Content-Type', req.headers['content-type'] || 'text/plain; charset=utf-8');
      res.end(body);
    });
    req.on('error', () => { res.statusCode = 400; res.end('Bad Request'); });
    return;
  }
  res.statusCode = 404;
  res.end('Not Found');
}).listen(3000, () => console.log('http://localhost:3000'));
```

**Solution 2 — Method + path router**

```js
const http = require('node:http');

http.createServer((req, res) => {
  const path = req.url.split('?')[0];

  if (path === '/') {
    if (req.method !== 'GET') {
      res.statusCode = 405;
      res.setHeader('Allow', 'GET');
      return res.end('Method Not Allowed');
    }
    res.setHeader('Content-Type', 'text/plain; charset=utf-8');
    return res.end('Welcome');
  }

  if (path === '/time') {
    if (req.method !== 'GET') {
      res.statusCode = 405;
      res.setHeader('Allow', 'GET');
      return res.end('Method Not Allowed');
    }
    res.setHeader('Content-Type', 'application/json; charset=utf-8');
    return res.end(JSON.stringify({ now: new Date().toISOString() }));
  }

  res.statusCode = 404;
  res.setHeader('Content-Type', 'application/json; charset=utf-8');
  res.end(JSON.stringify({ error: 'Not Found' }));
}).listen(3000, () => console.log('http://localhost:3000'));
```

**Solution 3 — Query-driven greeting**

```js
const http = require('node:http');

http.createServer((req, res) => {
  const url = new URL(req.url, `http://${req.headers.host}`);
  if (req.method === 'GET' && url.pathname === '/greet') {
    const name = url.searchParams.get('name') || 'friend';
    const timesRaw = url.searchParams.get('times') || '1';

    const times = Number(timesRaw);
    // Validate: must be an integer between 1 and 10.
    if (!Number.isInteger(times) || times < 1 || times > 10) {
      res.statusCode = 400;
      res.setHeader('Content-Type', 'application/json; charset=utf-8');
      return res.end(JSON.stringify({ error: 'times must be an integer between 1 and 10' }));
    }

    res.setHeader('Content-Type', 'text/plain; charset=utf-8');
    return res.end(`Hello ${name}! `.repeat(times).trimEnd());
  }

  res.statusCode = 404;
  res.end('Not Found');
}).listen(3000, () => console.log('http://localhost:3000'));
```

**Solution 4 — JSON API with validation**

```js
const http = require('node:http');

const users = [];      // in-memory store
let nextId = 1;        // id counter

// Reusable JSON body reader with size cap + safe parse.
async function readJsonBody(req, maxBytes = 1e6) {
  let size = 0;
  const chunks = [];
  for await (const chunk of req) {
    size += chunk.length;
    if (size > maxBytes) {
      throw Object.assign(new Error('Payload too large'), { statusCode: 413 });
    }
    chunks.push(chunk);
  }
  const raw = Buffer.concat(chunks).toString('utf8');
  if (!raw) return {};
  try {
    return JSON.parse(raw);
  } catch {
    throw Object.assign(new Error('Invalid JSON'), { statusCode: 400 });
  }
}

function sendJson(res, status, payload) {
  res.statusCode = status;
  res.setHeader('Content-Type', 'application/json; charset=utf-8');
  res.end(JSON.stringify(payload));
}

http.createServer(async (req, res) => {
  try {
    if (req.method === 'GET' && req.url === '/users') {
      return sendJson(res, 200, users);
    }

    if (req.method === 'POST' && req.url === '/users') {
      const type = req.headers['content-type'] || '';
      if (!type.includes('application/json')) {
        return sendJson(res, 415, { error: 'Expected application/json' });
      }

      const data = await readJsonBody(req);

      // Validation -> 422 Unprocessable Entity.
      if (typeof data.name !== 'string' || data.name.trim() === '') {
        return sendJson(res, 422, { error: 'name must be a non-empty string' });
      }
      if (typeof data.age !== 'number' || !Number.isFinite(data.age) || data.age < 0) {
        return sendJson(res, 422, { error: 'age must be a non-negative number' });
      }

      const user = { id: nextId++, name: data.name.trim(), age: data.age };
      users.push(user);
      return sendJson(res, 201, user);
    }

    return sendJson(res, 404, { error: 'Not Found' });
  } catch (err) {
    return sendJson(res, err.statusCode || 500, { error: err.message });
  }
}).listen(3000, () => console.log('http://localhost:3000'));
```

**Solution 5 — Mini static file server**

```js
const http = require('node:http');
const fs = require('node:fs');
const path = require('node:path');

const PUBLIC_DIR = path.join(__dirname, 'public');
const MIME = {
  '.html': 'text/html; charset=utf-8',
  '.css': 'text/css; charset=utf-8',
  '.js': 'text/javascript; charset=utf-8',
  '.json': 'application/json; charset=utf-8',
  '.png': 'image/png',
  '.jpg': 'image/jpeg',
  '.svg': 'image/svg+xml',
};

http.createServer((req, res) => {
  if (req.method !== 'GET' && req.method !== 'HEAD') {
    res.statusCode = 405;
    res.setHeader('Allow', 'GET, HEAD');
    return res.end('Method Not Allowed');
  }

  let pathname = decodeURIComponent(
    new URL(req.url, `http://${req.headers.host}`).pathname
  );
  if (pathname === '/') pathname = '/index.html';

  const filePath = path.join(PUBLIC_DIR, pathname);
  // Path-traversal defense: the resolved path must stay inside PUBLIC_DIR.
  if (filePath !== PUBLIC_DIR && !filePath.startsWith(PUBLIC_DIR + path.sep)) {
    res.statusCode = 403;
    return res.end('Forbidden');
  }

  fs.stat(filePath, (err, stats) => {
    if (err || !stats.isFile()) {
      res.statusCode = 404;
      return res.end('404 Not Found');
    }
    res.statusCode = 200;
    res.setHeader('Content-Type', MIME[path.extname(filePath).toLowerCase()] || 'application/octet-stream');
    res.setHeader('Content-Length', stats.size);

    if (req.method === 'HEAD') return res.end();

    const stream = fs.createReadStream(filePath);
    stream.pipe(res);
    stream.on('error', () => {
      if (!res.headersSent) res.statusCode = 500;
      res.end('Server Error');
    });
  });
}).listen(3000, () => console.log('http://localhost:3000'));
```

**Solution 6 — Proxy / aggregator**

```js
const http = require('node:http');
const https = require('node:https');

// Promise-based GET returning parsed JSON, with timeout + status awareness.
function fetchJson(urlString) {
  return new Promise((resolve, reject) => {
    const url = new URL(urlString);
    const lib = url.protocol === 'https:' ? https : http;

    const upstream = lib.get(url, { headers: { 'User-Agent': 'node-course-demo' } }, (res) => {
      let data = '';
      res.setEncoding('utf8');
      res.on('data', (c) => (data += c));
      res.on('end', () => {
        try {
          resolve({ statusCode: res.statusCode, body: JSON.parse(data) });
        } catch {
          reject(Object.assign(new Error('Upstream returned invalid JSON'), { statusCode: 502 }));
        }
      });
    });
    upstream.on('error', (err) => reject(Object.assign(err, { statusCode: 502 })));
    upstream.setTimeout(5000, () => upstream.destroy(
      Object.assign(new Error('Upstream timed out'), { statusCode: 504 })
    ));
  });
}

http.createServer(async (req, res) => {
  const match = req.url.split('?')[0].match(/^\/github\/([^/]+)$/);
  if (req.method === 'GET' && match) {
    const user = decodeURIComponent(match[1]);
    try {
      const { statusCode, body } = await fetchJson(`https://api.github.com/users/${encodeURIComponent(user)}`);

      if (statusCode === 404) {
        res.statusCode = 404;
        res.setHeader('Content-Type', 'application/json; charset=utf-8');
        return res.end(JSON.stringify({ error: `No GitHub user "${user}"` }));
      }
      if (statusCode !== 200) {
        res.statusCode = 502;
        return res.end(JSON.stringify({ error: 'Upstream error' }));
      }

      // Trim the payload to only the fields we care about.
      const trimmed = { login: body.login, name: body.name, public_repos: body.public_repos };
      res.statusCode = 200;
      res.setHeader('Content-Type', 'application/json; charset=utf-8');
      return res.end(JSON.stringify(trimmed));
    } catch (err) {
      res.statusCode = err.statusCode || 500;
      res.setHeader('Content-Type', 'application/json; charset=utf-8');
      return res.end(JSON.stringify({ error: err.message }));
    }
  }

  res.statusCode = 404;
  res.end('Not Found');
}).listen(3000, () => console.log('http://localhost:3000'));
```

## Summary / Cheat Sheet

| Concept | Key API / Fact |
| --- | --- |
| Create a server | `http.createServer((req, res) => {...})` → `http.Server` (an `EventEmitter`) |
| Start listening | `server.listen(port[, host][, cb])`; read `process.env.PORT` in prod |
| Handler args | `req` = `IncomingMessage` (readable stream); `res` = `ServerResponse` (writable stream) |
| Request basics | `req.method` (uppercase), `req.url` (path+query only), `req.headers` (lowercased keys) |
| Read body | Stream: `'data'` chunks → `'end'`; or `for await (const c of req)`. No `req.body` by default |
| Cap body size | Track total length; respond `413` and `req.destroy()` if over the limit |
| Parse query | `new URL(req.url, 'http://' + req.headers.host).searchParams` → `.get` / `.getAll` |
| Parse JSON body | Read stream → `JSON.parse` inside try/catch → `400` on failure |
| Parse form body | `new URLSearchParams(rawBody)` |
| Set status | `res.statusCode = 201` (before head is sent) |
| Set headers | `res.setHeader(k, v)` (one at a time) or `res.writeHead(status, {…})` (batch + flush) |
| Write body | `res.write(chunk)` (repeatable) then `res.end([chunk])` — always call `end` once |
| Send JSON | `setHeader('Content-Type','application/json')` + `res.end(JSON.stringify(obj))` |
| Send HTML | `Content-Type: text/html; charset=utf-8` |
| Serve files | `fs.createReadStream(path).pipe(res)` + MIME by extension + path-traversal guard |
| 404 | Fallthrough at end: `res.statusCode = 404; res.end(...)` |
| Errors | try/catch async handlers; check `res.headersSent`; log full, return generic for `5xx` |
| Outbound GET | `http.get(url, res => {...})` — consume the response stream |
| Outbound (any) | `http.request(opts, cb)`; `req.write(body)`; **`req.end()`**; `Buffer.byteLength` for `Content-Length` |
| HTTPS | `https` module, identical API; pick by `url.protocol` |
| Modern client | Global `fetch` (Node 18+) for new code; `http.request` underpins it |
| Status codes | `2xx` success, `3xx` redirect, `4xx` client error, `5xx` server error |
| Methods | safe: `GET`/`HEAD`/`OPTIONS`; idempotent: `GET`/`PUT`/`DELETE`/`HEAD`; `POST` neither |
| Why Express | Abstracts routing, params, body parsing, JSON/static helpers, and middleware over `http` |

You can now build a working HTTP server from nothing, read and write the protocol correctly, handle errors and edge cases, and make outbound calls — and, crucially, you understand precisely what a framework like Express is doing on your behalf. That foundation is what separates a developer who *uses* Express from one who truly *understands* the web server beneath it.
