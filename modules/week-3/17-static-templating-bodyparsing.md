# Static Files, Templating & Body Parsing

So far you have built Express apps that respond with JSON or plain strings. Real web applications need to do three more things almost immediately: **serve files the browser asked for** (CSS, JavaScript bundles, images, fonts, PDFs), **read the data that arrives in a request body** (a submitted form, a JSON payload from a frontend, a webhook), and — if you are rendering pages on the server — **generate HTML from templates**.

These three concerns are tightly related because they all live at the boundary between your server and the outside world. Static-file serving is about getting bytes *out* efficiently. Body parsing is about turning the raw bytes that come *in* into usable JavaScript values. Templating is about *producing* HTML on the server instead of shipping a blank page and letting the browser build it. This module covers all three from first principles, with an emphasis on the decisions a backend engineer actually makes: which parser to mount and where, how big a body to allow, when to render HTML server-side versus exposing a JSON API, and how to keep templated output safe from cross-site scripting.

> TIP: Everything here builds on the Express request/response lifecycle and middleware concepts from earlier modules. If "middleware is just a function with the signature `(req, res, next)` that runs in order" does not feel automatic yet, revisit that material first — static serving and body parsing are *both* implemented as middleware, and understanding mount order is the key to using them correctly.

## What You'll Learn

- How to serve static assets with **`express.static`** — including multiple source directories, a virtual path prefix, and HTTP caching via `maxAge`/`immutable`/`etag`.
- How HTTP request bodies actually arrive (as a stream of bytes) and why you must opt in to parsing them.
- The built-in body parsers — **`express.json`**, **`express.urlencoded`** (and what the `extended` option really means), **`express.raw`**, and **`express.text`** — plus how to set **size limits** to protect your server.
- The concrete difference between handling **HTML form submissions** (`application/x-www-form-urlencoded`, `multipart/form-data`) and **JSON API bodies** (`application/json`).
- A clear-eyed overview of **template engines** and a decision framework for **server-side rendering (SSR) vs. a JSON API**.
- Setting up **EJS**: the `views` folder, the `view engine` setting, and `res.render`.
- The full **EJS syntax**: interpolation, escaping, conditionals, loops, includes/partials, and layout patterns.
- How to **pass data to templates** safely and idiomatically.
- How to **serve a Single Page Application (SPA) build** (React/Vue/etc.) from an Express server, including the catch-all route that makes client-side routing work.
- The **XSS security model** of templating and exactly which EJS tag escapes and which does not.

---

## 1. Serving Static Assets with `express.static`

### 1.1 What "static" means and why it deserves its own mechanism

A **static asset** is a file whose bytes are sent to the client unchanged: a stylesheet, a compiled JavaScript bundle, an image, a font, a `favicon.ico`, a downloadable PDF. The server does not compute anything per request — it locates the file on disk and streams it back. This is in contrast to **dynamic** responses, where the body is generated at request time (a rendered template, a database-backed JSON document).

You *could* write a route for every file:

```js
// DON'T do this — it's tedious, error-prone, and slow.
const fs = require('node:fs');
app.get('/style.css', (req, res) => {
  res.type('text/css');
  res.send(fs.readFileSync('./public/style.css')); // sync read = blocks the event loop!
});
```

That approach is wrong on several axes: it blocks the event loop with synchronous I/O, it does not set caching headers, it does not handle conditional requests (`If-Modified-Since`), it does not support range requests (needed for video seeking), and you would have to repeat it for every file. Serving files correctly is genuinely fiddly, which is why Express ships a purpose-built middleware: **`express.static`**.

`express.static` is the only middleware bundled *inside* Express itself (it is a re-export of the battle-tested [`serve-static`](https://www.npmjs.com/package/serve-static) package, which builds on [`send`](https://www.npmjs.com/package/send)). It handles streaming, MIME-type detection, `ETag`/`Last-Modified` generation, conditional requests, range requests, and caching headers — correctly — for you.

### 1.2 The basic setup

```js
// app.js
const path = require('node:path');
const express = require('express');

const app = express();

// Serve everything inside the "public" directory at the site root.
app.use(express.static(path.join(__dirname, 'public')));

app.listen(3000, () => console.log('http://localhost:3000'));
```

Given this directory layout:

```text
project/
├── app.js
└── public/
    ├── index.html
    ├── style.css
    ├── app.js
    └── images/
        └── logo.png
```

The files become reachable like this:

| File on disk                     | URL                                  |
| -------------------------------- | ------------------------------------ |
| `public/index.html`              | `http://localhost:3000/` *and* `/index.html` |
| `public/style.css`               | `http://localhost:3000/style.css`    |
| `public/app.js`                  | `http://localhost:3000/app.js`       |
| `public/images/logo.png`         | `http://localhost:3000/images/logo.png` |

Notice two things. First, the directory name `public` does **not** appear in the URL — the directory is the *root* you are exposing, not a path segment. Second, a request for `/` automatically serves `index.html` (configurable via the `index` option).

> WARNING — Pitfall: Always pass an **absolute path** built with `path.join(__dirname, ...)`. A relative path like `express.static('public')` is resolved against `process.cwd()` — the directory you happened to launch the process from — not the directory of your source file. Start the app from a different folder and your assets vanish with a confusing 404. Building the path from `__dirname` makes it robust regardless of the working directory.

### 1.3 A virtual path prefix

Often you do not want assets at the root; you want them namespaced under something like `/static` or `/assets`. Pass a **mount path** as the first argument to `app.use`:

```js
// Files in ./public are now served under the /static prefix.
app.use('/static', express.static(path.join(__dirname, 'public')));
```

Now `public/images/logo.png` is reachable at `http://localhost:3000/static/images/logo.png`. The prefix is "virtual" because nothing called `static` exists on disk — Express simply strips that segment before looking up the file. This is the recommended pattern for production: it makes asset URLs predictable, keeps them out of the way of your API and page routes, and makes it trivial to later point a CDN or reverse proxy at the `/static` path.

> BEST PRACTICE: Use a virtual prefix (`/static`, `/assets`, `/public`) rather than serving at the root, *unless* you are serving a SPA build whose `index.html` must live at `/`. A prefix prevents a file named `users.html` in your `public` folder from shadowing a `/users` API route, and it makes caching/CDN configuration cleaner.

### 1.4 Multiple static directories

You can mount `express.static` more than once. When several static middlewares are registered, Express tries them **in registration order** and serves the file from the first directory that contains it. A directory that does not contain the requested file simply calls `next()` and the next middleware gets a turn.

```js
// Look in ./public first, then fall back to ./node_modules-cdn, then ./uploads.
app.use(express.static(path.join(__dirname, 'public')));
app.use(express.static(path.join(__dirname, 'vendor')));
app.use('/uploads', express.static(path.join(__dirname, 'uploads')));
```

A common real-world use: a `public` folder for your own assets and a separate `vendor` (or `node_modules`-sourced) folder for third-party libraries you want to self-host. Because order matters, put the directory you want to "win" on naming collisions first.

> TIP: Mounting `express.static` directly on `node_modules` is tempting but exposes your entire dependency tree — including package metadata and potentially files you did not mean to publish. Instead, copy or symlink only the specific files you need into a dedicated served directory, or use a build step (Vite, webpack, esbuild) that produces a clean `dist`.

### 1.5 Caching and the `maxAge` / `immutable` options

By default, `express.static` adds an `ETag` and a `Last-Modified` header to every response. These enable **conditional requests**: the browser caches the file and, on the next visit, asks "has this changed since the ETag I have?" If not, the server replies `304 Not Modified` with an empty body — the browser reuses its cached copy. This saves bandwidth but still costs a round trip.

To eliminate the round trip entirely, set a `Cache-Control: max-age` so the browser does not even ask until the freshness window expires:

```js
app.use(
  '/static',
  express.static(path.join(__dirname, 'public'), {
    maxAge: '1d',        // Cache-Control: public, max-age=86400 (one day, in ms or a ms-string)
    etag: true,          // keep ETag generation (default true) for revalidation after expiry
    lastModified: true,  // keep Last-Modified header (default true)
  })
);
```

`maxAge` accepts either a number of **milliseconds** or a human-readable string parsed by the [`ms`](https://www.npmjs.com/package/ms) package (`'1d'`, `'365d'`, `'10m'`, `'1y'`). The value lands in the `Cache-Control` header.

The most powerful pattern combines `maxAge` with `immutable` and **content-hashed filenames**. Modern bundlers emit files like `app.4f3a9c2b.js` where the hash changes whenever the content changes. Because the filename itself changes on every edit, you can cache the file *forever*:

```js
// Serve hashed build artifacts that never change for a given URL.
app.use(
  '/assets',
  express.static(path.join(__dirname, 'dist', 'assets'), {
    maxAge: '1y',     // a full year
    immutable: true,  // tells the browser: never revalidate, this URL's content can never change
  })
);
```

`Cache-Control: public, max-age=31536000, immutable` tells the browser it may serve this file from cache without *ever* checking back. That is safe precisely because a content change produces a new filename (and thus a new URL).

> BEST PRACTICE: Cache content-hashed bundles aggressively (`maxAge: '1y', immutable: true`), but serve `index.html` with **no caching** (`maxAge: 0` or `Cache-Control: no-cache`). The HTML is the entry point that references the hashed assets by name; if the HTML is cached, browsers keep loading old asset URLs and your deploy never takes effect.

#### Other useful `express.static` options

| Option         | Type / Default        | What it does |
| -------------- | --------------------- | ------------ |
| `maxAge`       | number/string, `0`    | `Cache-Control: max-age` value (ms or `ms`-string). |
| `immutable`    | boolean, `false`      | Adds `immutable` to `Cache-Control` (use only with content hashing). |
| `etag`         | boolean, `true`       | Generate weak `ETag` headers for revalidation. |
| `lastModified` | boolean, `true`       | Send `Last-Modified` from the file's mtime. |
| `index`        | string/array, `'index.html'` | Directory index file(s); set `false` to disable. |
| `dotfiles`     | string, `'ignore'`    | How to treat dotfiles: `'ignore'`, `'allow'`, or `'deny'`. |
| `fallthrough`  | boolean, `true`       | If `false`, a missing file ends the chain with 404 instead of calling `next()`. |
| `redirect`     | boolean, `true`       | Redirect `/dir` to `/dir/` when a directory is requested. |
| `setHeaders`   | function              | Hook to set custom headers per file before sending. |

Example using `setHeaders` to force-download a folder of PDFs:

```js
app.use(
  '/downloads',
  express.static(path.join(__dirname, 'files'), {
    setHeaders(res, filePath) {
      if (path.extname(filePath) === '.pdf') {
        // Suggest the browser download rather than render inline.
        res.setHeader('Content-Disposition', 'attachment');
      }
    },
  })
);
```

> WARNING — Pitfall: The default `dotfiles: 'ignore'` means files like `.env` inside a served directory return 404 — but **do not rely on this for security**. Never place secrets (`.env`, `.git`, key files) inside a directory you serve statically. The safest posture is a dedicated `public`/`dist` directory that contains *only* assets meant to be public, plus `dotfiles: 'deny'` as defense in depth.

### 1.6 Static middleware and route order

Because `express.static` is just middleware, **where you mount it relative to your routes matters**. Put it before your dynamic routes if you want files to take precedence, but be aware that a static middleware mounted at the root will attempt a file lookup (a disk `stat`) on *every* request that reaches it before falling through to your routes. For high-traffic APIs, mounting static under a prefix avoids that per-request disk hit on API paths.

```js
app.use('/static', express.static(path.join(__dirname, 'public'))); // disk hit only for /static/*
app.use('/api', apiRouter);  // API requests never touch the filesystem
```

> TIP: In production, you frequently offload static serving to a reverse proxy (nginx) or a CDN entirely, and Node serves only dynamic responses. `express.static` is excellent for development and small deployments, and perfectly fine for moderate traffic, but the fastest static file is one your Node process never has to touch.

---

## 2. Body Parsing — Turning Request Bytes Into JavaScript

### 2.1 Why bodies need parsing at all

When a client sends a `POST`, `PUT`, or `PATCH`, the payload arrives as a **stream of raw bytes** on the request object. Express (and Node's `http` module beneath it) does *not* read or interpret that stream for you, for good reasons: the body could be 5 bytes or 5 gigabytes, it could be JSON, a form, a file upload, or binary, and reading it costs memory. So Node hands you `req` as a readable stream and leaves the decision to you.

Without a parser, `req.body` is `undefined`:

```js
app.post('/echo', (req, res) => {
  console.log(req.body); // undefined — nobody read the stream yet
  res.send('ok');
});
```

A **body-parsing middleware** consumes that stream, buffers the bytes, decodes them according to the request's `Content-Type` header, and assigns the result to `req.body`. Express ships four parsers, all re-exports of the [`body-parser`](https://www.npmjs.com/package/body-parser) package (which was merged back into Express core in 4.16):

| Middleware            | Parses Content-Type                         | `req.body` becomes |
| --------------------- | ------------------------------------------- | ------------------ |
| `express.json()`      | `application/json`                          | a parsed JS object/array |
| `express.urlencoded()`| `application/x-www-form-urlencoded`         | an object of form fields |
| `express.text()`      | `text/plain` (configurable)                 | a string |
| `express.raw()`       | `application/octet-stream` (configurable)   | a `Buffer` |

> IMPORTANT — How content-type matching works: Each parser only acts on requests whose `Content-Type` matches its target. If `express.json()` sees a `text/plain` request, it does nothing and calls `next()`, leaving `req.body` for another parser. This means you can mount several parsers simultaneously and each will handle only the requests it understands.

### 2.2 `express.json()` — parsing JSON API bodies

This is the workhorse for modern APIs. Mount it globally and `req.body` will be the parsed object on any request that arrives with `Content-Type: application/json`.

```js
const express = require('express');
const app = express();

// Parse application/json bodies into req.body. Mount before routes that need it.
app.use(express.json());

app.post('/api/users', (req, res) => {
  // req.body is the parsed object, e.g. { name: 'Ada', email: 'ada@x.com' }
  const { name, email } = req.body;
  res.status(201).json({ id: 1, name, email });
});
```

Test it from the command line:

```bash
curl -X POST http://localhost:3000/api/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Ada","email":"ada@x.com"}'
# -> {"id":1,"name":"Ada","email":"ada@x.com"}
```

> WARNING — Pitfall: If the client forgets the `Content-Type: application/json` header (or sends `text/plain`), `express.json()` skips the body and `req.body` is `{}` (an empty object, in Express 4) or `undefined`. The classic symptom is "my `req.body` is empty but I'm definitely sending data." Check the request's `Content-Type` first.

#### Useful `express.json()` options

```js
app.use(
  express.json({
    limit: '100kb',        // reject bodies larger than this (default '100kb')
    strict: true,          // only accept arrays/objects at top level (default true)
    type: 'application/json', // which Content-Type(s) to parse
    // Verify hook: gets the raw buffer BEFORE parsing. Essential for webhook
    // signature verification (e.g. Stripe), where you need the exact raw bytes.
    verify(req, res, buf) {
      req.rawBody = buf; // stash the raw buffer for later HMAC checks
    },
  })
);
```

- **`limit`** caps the body size — covered in depth in §2.6. The default is a deliberately small `100kb`.
- **`strict: true`** (default) means only a JSON object or array is accepted at the top level. With `strict: false`, primitives like `42` or `"hello"` or `true` parse successfully. Keep it strict unless you have a specific reason.
- **`type`** lets you parse JSON sent under a different content type (e.g. some clients send `application/vnd.api+json`).
- **`verify`** is the canonical way to capture the raw body for signature verification while still getting parsed JSON.

### 2.3 `express.urlencoded()` — parsing HTML form submissions

When a classic HTML `<form>` is submitted (without JavaScript intercepting it), the browser encodes the fields as `application/x-www-form-urlencoded` — the same `key=value&key=value` format as a URL query string. `express.urlencoded()` parses that into `req.body`.

```js
// Parse application/x-www-form-urlencoded bodies (HTML form submissions).
app.use(express.urlencoded({ extended: true }));

app.post('/contact', (req, res) => {
  // For a form with fields "name" and "message":
  // req.body === { name: 'Ada', message: 'Hello!' }
  res.send(`Thanks, ${req.body.name}!`);
});
```

The matching HTML form:

```html
<form action="/contact" method="POST">
  <input name="name" />
  <textarea name="message"></textarea>
  <button type="submit">Send</button>
</form>
```

#### The `extended` option — what it actually controls

`express.urlencoded()` *requires* you to think about `extended`. It chooses the library used to parse the query-string-style body:

- **`extended: false`** uses Node's built-in `querystring` module. It parses **flat** key/value pairs only. `a=1&b=2` becomes `{ a: '1', b: '2' }`. Nested keys are not interpreted — values are always strings.
- **`extended: true`** uses the [`qs`](https://www.npmjs.com/package/qs) library, which supports **rich, nested objects and arrays** via bracket notation.

The difference is clearest with an example. Suppose the body is:

```text
user[name]=Ada&user[roles][]=admin&user[roles][]=editor
```

| `extended` value | Resulting `req.body` |
| ---------------- | -------------------- |
| `false`          | `{ 'user[name]': 'Ada', 'user[roles][]': ['admin', 'editor'] }` (keys are literal strings) |
| `true`           | `{ user: { name: 'Ada', roles: ['admin', 'editor'] } }` (proper nesting) |

```js
// extended: true → rich nested objects via the qs library
app.use(express.urlencoded({ extended: true }));

// extended: false → flat key/values via Node's built-in querystring
app.use(express.urlencoded({ extended: false }));
```

> BEST PRACTICE: For simple flat forms, `extended: false` is lighter (no extra dependency behavior) and perfectly adequate. Use `extended: true` when your forms genuinely need nested structures. **You must pass the option explicitly** — older Express versions printed a deprecation warning when it was omitted because the default changed across versions. Being explicit removes all ambiguity.

> WARNING — Pitfall: With `extended: true` (the `qs` parser), be aware that deeply nested or array-heavy input can be abused (parameter pollution, prototype pollution attempts, large object graphs). `qs` has built-in `depth` and `parameterLimit` guards, but if you do not need nesting, prefer `extended: false` for a smaller attack surface. Always validate parsed bodies against an explicit schema.

### 2.4 `express.text()` and `express.raw()` — the specialist parsers

Most APIs only need JSON and urlencoded, but two more parsers cover edge cases.

**`express.text()`** buffers the body and gives you a plain string. Useful for plain-text payloads, GraphQL queries sent as text, or CSV uploads as text.

```js
app.use(express.text({ type: 'text/plain', limit: '1mb' }));

app.post('/notes', (req, res) => {
  // req.body is a string, e.g. "Remember the milk"
  res.send(`Saved ${req.body.length} characters`);
});
```

**`express.raw()`** leaves the body as a Node `Buffer` — no decoding at all. This is the right choice for binary payloads and, critically, for **webhooks that require signature verification over the exact raw bytes** (Stripe, GitHub, etc.), because any re-encoding would break the signature.

```js
// Mount raw() ONLY on the webhook route, not globally — see the Stripe example below.
app.post(
  '/webhooks/stripe',
  express.raw({ type: 'application/json' }), // req.body is a Buffer of the exact bytes
  (req, res) => {
    const signature = req.headers['stripe-signature'];
    // verifySignature(req.body, signature) — req.body is the untouched Buffer
    res.sendStatus(200);
  }
);
```

> IMPORTANT: For webhook signature verification you have two valid approaches: (1) mount `express.raw()` *only* on the webhook route so it never collides with a global `express.json()`, or (2) use the `verify` hook of `express.json()` to stash `req.rawBody`. Approach (1) is the simplest and most common. Do **not** mount a global `express.json()` and then expect the raw bytes — by then the buffer has been consumed and re-stringifying the parsed object will not reproduce the original bytes.

### 2.5 Mounting parsers globally vs. per-route

You do not have to apply parsers to the whole app. Because they are middleware, you can scope them:

```js
// Global JSON parsing for the whole app...
app.use(express.json());

// ...but a specific route can mount a different/extra parser inline.
app.post('/upload-text', express.text({ limit: '10mb' }), handler);

// ...and a webhook route uses raw bytes, bypassing the global json parser
// because raw() runs first on this route and json() ignores non-json types here.
app.post('/hook', express.raw({ type: '*/*' }), webhookHandler);
```

> BEST PRACTICE: For most apps, mount `express.json()` and `express.urlencoded({ extended: true })` once, globally, near the top of your middleware stack (after security middleware like `helmet`, before your routes). Reserve `raw`/`text` and route-scoped parsers for the few endpoints that genuinely need them. Scoping parsers also slightly improves performance: a `GET`-heavy API need not run a parser on every request.

### 2.6 Size limits — a security control, not a nicety

Every parser accepts a `limit` option, and the default is `100kb`. This is one of the most important and most overlooked security settings in Express.

The threat: without a limit, a malicious client can `POST` a multi-gigabyte body. The parser dutifully buffers it all into memory, and your process exhausts its heap and crashes — a trivial **denial-of-service**. The `limit` caps the bytes the parser will buffer; exceeding it produces a `413 Payload Too Large` error instead of an out-of-memory crash.

```js
app.use(express.json({ limit: '1mb' }));            // generous for JSON APIs
app.use(express.urlencoded({ extended: true, limit: '1mb' }));
app.use('/import', express.text({ limit: '20mb' })); // bigger for a bulk-import endpoint
```

When a body exceeds the limit, the parser passes an error to `next(err)` with `err.status === 413` and `err.type === 'entity.too.large'`. Handle it in your error middleware to return a clean response:

```js
// Error-handling middleware (4 args). Mount AFTER all routes.
app.use((err, req, res, next) => {
  if (err.type === 'entity.too.large') {
    return res.status(413).json({ error: 'Request body too large' });
  }
  if (err.type === 'entity.parse.failed') {
    // Thrown by express.json() on malformed JSON.
    return res.status(400).json({ error: 'Invalid JSON' });
  }
  next(err); // not ours — pass it on
});
```

> BEST PRACTICE: Set an explicit `limit` on every parser, sized to the largest *legitimate* body that endpoint expects. Do not blindly raise it to accommodate file uploads — handle file uploads with a streaming multipart parser (`multer`/`busboy`) that writes to disk or object storage instead of buffering into memory. JSON and urlencoded parsers are for *structured data*, not files.

### 2.7 A note on Express 5 behavior

In Express 5, async route handlers that throw automatically forward errors to your error middleware (no more manual `try/catch` + `next(err)` for promise rejections), and several body-parser edge cases were tidied. The body-parser API itself (`express.json`, `express.urlencoded`, `express.raw`, `express.text`) is **identical** across v4 and v5 — your parser setup carries over unchanged. The one behavior to note: in Express 5, when no body parser matches, `req.body` is `undefined` rather than `{}` in some paths, so guard with optional chaining (`req.body?.name`) or validation rather than assuming an object.

---

## 3. HTML Form Submissions vs. JSON API Bodies

These two are the most common request bodies, and the difference between them is a frequent source of confusion. They differ in *who sends them*, *how they are encoded*, and *which parser reads them*.

### 3.1 The classic HTML form

A native HTML `<form>` submission — no JavaScript involved — encodes its fields according to the form's `enctype` and issues a **full-page navigation** (the browser leaves the current page and loads the response).

| `enctype`                            | Used for                          | Express parser |
| ------------------------------------ | --------------------------------- | -------------- |
| `application/x-www-form-urlencoded` (default) | text fields, checkboxes, selects | `express.urlencoded()` |
| `multipart/form-data`                | file uploads + text fields        | `multer` / `busboy` (not a built-in parser) |
| `text/plain`                         | rarely used; raw text             | `express.text()` |

```html
<!-- Default urlencoded form -->
<form action="/signup" method="POST">
  <input name="email" type="email" />
  <input name="password" type="password" />
  <button>Sign up</button>
</form>

<!-- File upload requires multipart/form-data -->
<form action="/avatar" method="POST" enctype="multipart/form-data">
  <input name="avatar" type="file" />
  <button>Upload</button>
</form>
```

> WARNING — Pitfall: `express.urlencoded()` and `express.json()` **cannot parse `multipart/form-data`** — that encoding interleaves field boundaries and binary file data, which needs a streaming parser. For file uploads use [`multer`](https://www.npmjs.com/package/multer). If `req.body` is empty on a file-upload form, this mismatch is almost always why.

After processing a native form POST, the conventional response is a **redirect** (the Post/Redirect/Get pattern), which prevents the browser from re-submitting the form if the user refreshes:

```js
app.post('/signup', (req, res) => {
  const { email, password } = req.body; // from express.urlencoded()
  // ... create the user ...
  res.redirect('/welcome'); // PRG pattern: redirect after a successful POST
});
```

### 3.2 The JSON API body

A frontend framework (React, Vue) or a mobile client sends data via `fetch`/`XMLHttpRequest` with `Content-Type: application/json`. There is **no page navigation** — the JavaScript reads the JSON response and updates the UI itself.

```js
// Browser-side (frontend) code
await fetch('/api/users', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' }, // <-- required, or express.json() ignores it
  body: JSON.stringify({ name: 'Ada', email: 'ada@x.com' }),
});
```

```js
// Server-side
app.use(express.json());
app.post('/api/users', (req, res) => {
  const user = createUser(req.body);
  res.status(201).json(user); // respond with JSON, no redirect
});
```

### 3.3 Side-by-side comparison

| Aspect              | HTML form (urlencoded)            | JSON API body                    |
| ------------------- | --------------------------------- | -------------------------------- |
| Sent by             | Browser native form submit        | `fetch`/XHR / mobile / other servers |
| `Content-Type`      | `application/x-www-form-urlencoded` | `application/json`               |
| Express parser      | `express.urlencoded()`            | `express.json()`                 |
| Value types         | everything is a **string**        | preserves numbers, booleans, null, nested objects |
| Nested data         | only with `extended: true` + bracket syntax | native via JSON                  |
| Typical response    | `302` redirect (PRG) or rendered HTML | `200/201` JSON                   |
| File uploads        | via `multipart/form-data` + `multer` | usually a separate upload endpoint |

> TIP: A subtle but important difference: urlencoded form values are **always strings**. A checkbox sends `on` (or nothing if unchecked), and a number input sends `"42"`, not `42`. Coerce and validate on the server. JSON preserves the original types, so `42` arrives as a number — but you must *still* validate, because clients lie.

---

## 4. Template Engines & Server-Side Rendering vs. JSON APIs

### 4.1 What a template engine is and why it exists

A **template engine** lets you write HTML with placeholders and control flow, then *render* it on the server by injecting data. Instead of building HTML by string concatenation (error-prone, unsafe, unreadable), you write a template file and call `res.render('page', { data })`. The engine merges the template with the data and produces a complete HTML string sent to the browser.

```js
// Without a template engine — fragile and XSS-prone:
res.send('<h1>Hello ' + req.query.name + '</h1>'); // DON'T: unescaped, hard to maintain

// With a template engine — clean, escaped, maintainable:
res.render('greeting', { name: req.query.name });
```

Express supports any engine that conforms to its simple interface; popular choices include **EJS**, **Pug** (formerly Jade), and **Handlebars** (via `express-handlebars`).

| Engine       | Syntax style                          | Notable trait |
| ------------ | ------------------------------------- | ------------- |
| **EJS**      | HTML with `<% %>` JavaScript tags     | Closest to plain HTML + JS; easiest to learn. |
| **Pug**      | Indentation-based, no closing tags    | Terse but a whole new syntax to learn. |
| **Handlebars** | `{{ }}` logic-less mustache style    | Enforces separation of logic from markup. |

We use **EJS** in this module because it is the gentlest on-ramp for a JavaScript developer: an EJS template *is* HTML with embedded JavaScript expressions.

### 4.2 Server-Side Rendering (SSR) vs. a JSON API — the decision

This is one of the most important architectural decisions you will make, and interviewers love it. Both patterns are valid; choosing well depends on the application.

| Dimension                 | Server-Side Rendering (templates) | JSON API + client rendering (SPA) |
| ------------------------- | --------------------------------- | --------------------------------- |
| Who builds the HTML       | The server (Express + EJS)        | The browser (React/Vue) from JSON |
| Initial response          | Fully-formed HTML page            | Minimal HTML shell + JS bundle    |
| First paint / SEO         | Fast first paint, easy SEO        | Slower first paint; needs SSR/SSG for SEO |
| Interactivity after load  | Full page reloads per navigation  | Instant client-side navigation    |
| Backend role              | Renders pages *and* data          | Serves data only; UI is separate  |
| Best for                  | Content sites, blogs, admin panels, dashboards, server-driven apps | Highly interactive apps, multiple clients (web + mobile) sharing one API |
| Coupling                  | Frontend & backend tightly coupled | Cleanly decoupled; API reusable by mobile, third parties |

**Choose SSR (templates) when:** you want fast first paint and strong SEO without a complex build pipeline; the app is content-heavy (marketing pages, blogs, documentation); you are building an internal admin tool or dashboard where a build toolchain is overkill; or a small team wants to ship pages quickly with one codebase.

**Choose a JSON API when:** the frontend is a rich SPA; you have multiple clients (web app + mobile app + partners) that should share one backend; you want the frontend and backend to evolve and deploy independently; or the interactivity demands client-side state and routing.

**The hybrid reality:** Many production systems do both. A common shape is a JSON API for the application proper, plus a few server-rendered templates for emails, PDFs, the public marketing site, or an admin panel. And meta-frameworks (Next.js, Nuxt, Remix) blur the line by doing SSR *of a SPA*. For this module, the mental model is: **render HTML on the server with EJS** when the server owns the page, **return JSON** when a client owns the page.

> BEST PRACTICE: Do not agonize over picking one forever. A clean Express app can expose `/api/*` JSON routes and `/admin/*` rendered routes side by side. Keep your business logic in a service layer so both the JSON controllers and the template controllers call the same functions — the only difference is `res.json(data)` vs. `res.render('view', data)`.

---

## 5. Setting Up EJS

### 5.1 Install and configure

```bash
npm install ejs
```

You do **not** `require('ejs')` directly in typical use — Express loads it for you when you set the view engine. Three settings matter:

```js
const path = require('node:path');
const express = require('express');
const app = express();

// 1. Tell Express which engine to use for files without an explicit extension.
app.set('view engine', 'ejs');

// 2. Tell Express where template files live (default is ./views).
app.set('views', path.join(__dirname, 'views'));

app.get('/', (req, res) => {
  // 3. Render views/index.ejs, passing it data.
  res.render('index', { title: 'Home', user: { name: 'Ada' } });
});

app.listen(3000);
```

How this works under the hood:

1. `app.set('view engine', 'ejs')` registers EJS as the default engine and lets you call `res.render('index')` instead of `res.render('index.ejs')` — Express appends the extension automatically.
2. `app.set('views', ...)` sets the directory Express searches for templates. The default is a `views` folder relative to the process working directory; setting it explicitly from `__dirname` makes it robust.
3. `res.render(view, locals)` finds the template, compiles it (caching the compiled function in production), executes it with `locals`, and sends the resulting HTML with `Content-Type: text/html`.

The expected layout:

```text
project/
├── app.js
└── views/
    ├── index.ejs
    ├── about.ejs
    └── partials/
        ├── header.ejs
        └── footer.ejs
```

> TIP: In production (`NODE_ENV=production`), Express **caches** compiled templates so each file is read and compiled from disk only once. In development it recompiles on every request so your edits show up without a restart. This is automatic — just set `NODE_ENV` appropriately. You can force it with `app.set('view cache', true)`.

### 5.2 `res.render` in detail

```js
res.render(view, locals, callback);
```

- **`view`** — the template name relative to the `views` directory (no extension needed). Subfolders work: `res.render('users/profile')` renders `views/users/profile.ejs`.
- **`locals`** — an object whose properties become *variables* inside the template. `{ title: 'Home' }` makes `title` usable as a bare variable in the template.
- **`callback`** *(optional)* — `(err, html) => {}`. If provided, Express does **not** auto-send; you get the rendered HTML string to do with as you like (e.g. render an email body without sending an HTTP response).

```js
// Render to a string instead of sending it (e.g. for an email body).
res.render('emails/welcome', { name: 'Ada' }, (err, html) => {
  if (err) return next(err);
  sendEmail({ to: 'ada@x.com', html });
  res.sendStatus(202);
});
```

#### `app.locals` and `res.locals` — data available to every template

Sometimes you want data in *every* template without passing it to each `render` call: the app name, the current year, the logged-in user. Express provides two scopes:

- **`app.locals`** — application-wide, set once, available in all templates for the app's lifetime.
- **`res.locals`** — per-request, typically set in middleware, available to the template rendered during that request.

```js
// Available to every template, forever:
app.locals.siteName = 'Acme';
app.locals.year = new Date().getFullYear();

// Available to templates for THIS request only — perfect for the current user:
app.use((req, res, next) => {
  res.locals.currentUser = req.user || null; // set by your auth middleware
  next();
});
```

Inside any template you can now use `siteName`, `year`, and `currentUser` without passing them through `res.render`. Locals you pass directly to `res.render` are merged on top of these.

---

## 6. EJS Syntax — The Complete Tour

EJS templates are HTML with embedded JavaScript inside special tags. Here is every tag you need, in order of importance.

### 6.1 The tag reference

| Tag        | Name                        | What it does |
| ---------- | --------------------------- | ------------ |
| `<%= %>`   | **Escaped output**          | Evaluates JS and outputs the result **HTML-escaped** (XSS-safe). |
| `<%- %>`   | **Unescaped output**        | Evaluates JS and outputs the **raw** result (NOT escaped — dangerous with user data). |
| `<% %>`    | **Scriptlet**               | Runs JS for control flow (if/for/etc.); outputs nothing itself. |
| `<%# %>`   | **Comment**                 | A comment; not rendered, not executed. |
| `<%_ _%>`  | **Whitespace-slurping**     | Like `<% %>` but trims leading/trailing whitespace/newlines. |
| `<%%`      | **Literal**                 | Outputs a literal `<%`. |

### 6.2 Interpolation and escaping

The single most important distinction in EJS — and the heart of the XSS section later — is `<%= %>` versus `<%- %>`.

```html
<!-- views/demo.ejs -->
<h1>Welcome, <%= user.name %></h1>   <!-- ESCAPED: safe for user data -->
<div><%- article.htmlBody %></div>   <!-- RAW: only for trusted, pre-sanitized HTML -->
```

Suppose `user.name` is the malicious string `<script>alert('xss')</script>`:

- `<%= user.name %>` outputs `&lt;script&gt;alert(&#39;xss&#39;)&lt;/script&gt;` — the browser displays it as literal text. **Safe.**
- `<%- user.name %>` outputs `<script>alert('xss')</script>` — the browser *executes* it. **A live XSS vulnerability.**

> WARNING — Pitfall: Use `<%= %>` (escaped) by default for **anything** that could contain user-influenced data. Reserve `<%- %>` for HTML you produced and trust, or HTML you have already sanitized with a library like `sanitize-html`. Confusing these two tags is the number-one cause of XSS in EJS apps.

### 6.3 Conditionals

Use scriptlet tags `<% %>` for the JavaScript control flow and output tags for the content:

```html
<% if (currentUser) { %>
  <p>Signed in as <%= currentUser.name %></p>
  <a href="/logout">Log out</a>
<% } else { %>
  <a href="/login">Log in</a>
<% } %>

<!-- Ternaries inline are also fine for small choices -->
<span class="badge <%= order.paid ? 'green' : 'red' %>">
  <%= order.paid ? 'Paid' : 'Unpaid' %>
</span>
```

Notice the JavaScript braces `{ ... }` are split across separate `<% %>` tags, with HTML in between. EJS stitches the pieces together into one function, so the braces match across tags exactly as they would in a normal JS file.

### 6.4 Loops

Any JavaScript iteration works. The idiomatic choice is `Array.prototype.forEach` or a `for...of` loop:

```html
<ul>
  <% products.forEach(function (product) { %>
    <li>
      <a href="/products/<%= product.id %>"><%= product.name %></a>
      — $<%= product.price.toFixed(2) %>
    </li>
  <% }); %>
</ul>

<!-- Handle the empty case explicitly -->
<% if (products.length === 0) { %>
  <p>No products found.</p>
<% } %>
```

```html
<!-- for...of works too, and reads cleanly -->
<table>
  <% for (const user of users) { %>
    <tr>
      <td><%= user.name %></td>
      <td><%= user.email %></td>
    </tr>
  <% } %>
</table>
```

> TIP: Keep logic in the template minimal. A loop and a conditional are fine, but sorting, filtering, and complex calculations belong in your route handler (or a service), passed in as ready-to-render data. Templates that do heavy logic become unreadable and untestable.

### 6.5 Includes and partials

A **partial** is a reusable fragment — a header, footer, navigation bar, or card — that you `include` into multiple pages. EJS provides the `include()` function, called from inside an *output* tag.

```text
views/
├── partials/
│   ├── header.ejs
│   └── footer.ejs
└── index.ejs
```

```html
<!-- views/partials/header.ejs -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <title><%= title %> — <%= siteName %></title>
  <link rel="stylesheet" href="/static/style.css" />
</head>
<body>
  <nav><a href="/">Home</a> · <a href="/about">About</a></nav>
```

```html
<!-- views/partials/footer.ejs -->
  <footer>&copy; <%= year %> <%= siteName %></footer>
</body>
</html>
```

```html
<!-- views/index.ejs -->
<%- include('partials/header', { title: 'Home' }) %>

<main>
  <h1>Welcome to <%= siteName %></h1>
</main>

<%- include('partials/footer') %>
```

Key details about `include`:

- Use `<%- %>` (raw output) with `include`, not `<%= %>`. The included partial is already-rendered, trusted HTML; escaping it would mangle the markup. This is the one routine, legitimate use of `<%- %>`.
- The path is relative to the current template's directory (or the `views` root).
- You can pass data as a second argument: `include('partials/header', { title: 'Home' })`. The included file sees those variables **plus** the parent's locals.

> WARNING — Pitfall: `<%= include(...) %>` (escaped) will HTML-escape your entire partial, dumping visible `&lt;div&gt;` tags onto the page. Always use `<%- include(...) %>` for partials.

### 6.6 Layouts

Plain EJS has **no built-in layout system** (unlike Pug's `extends`/`block` or Handlebars layouts). The include pattern above — a `header` partial at the top and a `footer` partial at the bottom of every page — is the canonical "layout" in vanilla EJS, and it works well.

If you want true layouts (one master template with a content slot), add the [`express-ejs-layouts`](https://www.npmjs.com/package/express-ejs-layouts) package:

```bash
npm install express-ejs-layouts
```

```js
const expressLayouts = require('express-ejs-layouts');

app.set('view engine', 'ejs');
app.use(expressLayouts);
app.set('layout', 'layouts/main'); // views/layouts/main.ejs is the master template
```

```html
<!-- views/layouts/main.ejs -->
<!DOCTYPE html>
<html lang="en">
<head><title><%= title %></title></head>
<body>
  <nav>...</nav>
  <%- body %>   <!-- each page's rendered content is injected here -->
  <footer>...</footer>
</body>
</html>
```

```html
<!-- views/index.ejs — just the page-specific content; the layout wraps it -->
<h1>Home</h1>
<p>Welcome!</p>
```

> TIP: For a small app, the header/footer-partials pattern is simpler and dependency-free. Reach for `express-ejs-layouts` once you have several pages sharing a complex shell and the repetition of `include('header')` / `include('footer')` becomes annoying.

---

## 7. Passing Data to Templates

You have seen `res.render('view', locals)`. Let's make the data flow concrete and idiomatic.

```js
app.get('/products', async (req, res, next) => {
  try {
    const products = await Product.find().lean(); // fetch from DB
    res.render('products/list', {
      title: 'Our Products',
      products,                       // an array → loop over it in the template
      filters: req.query,             // echo back the active filters
      isEmpty: products.length === 0, // precompute booleans for clean templates
    });
  } catch (err) {
    next(err); // forward to error middleware → render an error page
  }
});
```

Inside `views/products/list.ejs`, `title`, `products`, `filters`, and `isEmpty` are all available as bare variables, *merged with* anything in `app.locals` and `res.locals`.

> WARNING — Pitfall: Referencing a variable in a template that was **not** passed in throws `ReferenceError: x is not defined` and crashes the render. Guard optional values: use `typeof user !== 'undefined' && user` or pass explicit defaults (`user: req.user || null`) so the template can rely on the variable always existing. Setting safe defaults in `res.locals` middleware is the cleanest fix.

> BEST PRACTICE: Compute presentation-ready data in the controller, not the template. Format dates, compute totals, derive booleans (`isEmpty`, `hasNextPage`), and shape arrays *before* `res.render`. The template should mostly *display* — that keeps it readable and pushes logic into code you can unit-test.

---

## 8. Serving a Single Page Application (SPA) Build from Express

When you build a React/Vue/Svelte app with Vite (or CRA, webpack, etc.), the build step emits a `dist` (or `build`) folder containing a single `index.html`, a bundle of hashed JS/CSS in an `assets` subfolder, and static media. Express can serve this directly, which is great for small/medium apps where you do not want a separate static host.

Two things must be true for a SPA to work when served from Express:

1. The hashed asset files must be served statically (with aggressive caching).
2. **Every** non-API, non-asset URL must return `index.html`, because the SPA uses *client-side routing*. When a user navigates to `/dashboard/settings` and refreshes, the browser asks the server for `/dashboard/settings` — a path that does not exist as a file. The server must respond with `index.html` so the SPA boots and its router renders the right view.

### 8.1 The complete pattern

```js
const path = require('node:path');
const express = require('express');
const app = express();

// --- 1. API routes FIRST, so they are not swallowed by the SPA catch-all ---
app.use(express.json());
app.use('/api', require('./routes/api'));

// --- 2. Serve the built static assets with long-lived caching ---
const distDir = path.join(__dirname, 'client', 'dist');
app.use(
  express.static(distDir, {
    // index.html is handled by the catch-all below, so disable auto-index here
    index: false,
    // hashed assets can be cached for a year; immutable since names change on edit
    maxAge: '1y',
    immutable: true,
  })
);

// --- 3. Catch-all: send index.html for any other GET so client routing works ---
//     Express 5 uses the named-splat / RegExp form; '*' alone is no longer valid.
app.get('/*splat', (req, res) => {
  res.sendFile(path.join(distDir, 'index.html'));
});

app.listen(3000);
```

A few critical details:

- **Order matters.** API routes are registered *before* the catch-all. If the catch-all came first, every `/api/...` request would receive `index.html` instead of JSON.
- **`index: false`** on the static middleware prevents it from auto-serving `index.html` for `/`, leaving that to the catch-all so all page routes flow through one place. (Leaving the default `index: 'index.html'` also works for `/`; `index: false` just centralizes the logic.)
- **The catch-all route shape differs between Express versions.** In **Express 4**, the catch-all is `app.get('*', ...)`. In **Express 5**, the routing layer switched to `path-to-regexp@8`, and a bare `'*'` is no longer a valid path — you must name the wildcard: `app.get('/*splat', ...)` (or use a RegExp like `app.get(/.*/, ...)`). This is one of the most common Express-4-to-5 migration breakages, so it is worth memorizing.

```js
// Express 4 catch-all (note the bare asterisk):
app.get('*', (req, res) => res.sendFile(path.join(distDir, 'index.html')));

// Express 5 catch-all (named wildcard — bare '*' throws):
app.get('/*splat', (req, res) => res.sendFile(path.join(distDir, 'index.html')));
```

> WARNING — Pitfall: If you serve `index.html` with the same aggressive `maxAge: '1y'` as your hashed assets, browsers will cache the *entry point* and keep loading stale asset URLs after a deploy. Serve `index.html` with no caching. Using `res.sendFile` in the catch-all (rather than letting `express.static` serve it) gives you control: `res.sendFile(file, { maxAge: 0 })` or set `Cache-Control: no-cache` explicitly.

> BEST PRACTICE: Keep API routes under a clear prefix (`/api`) and consider returning a JSON `404` for unknown `/api/*` paths *before* the SPA catch-all, so a typo'd API call gets a proper error instead of a confusing chunk of HTML. Mount a small router: `app.use('/api', (req, res) => res.status(404).json({ error: 'Not found' }))` after your real API routes but before the catch-all.

---

## 9. Security — Escaping Output to Prevent XSS

**Cross-Site Scripting (XSS)** is the injection of attacker-controlled script into a page that other users view. The classic vector in server-rendered apps: you take user input (a username, a comment, a search term) and place it into HTML without escaping it. The browser cannot tell your intended markup from the attacker's injected `<script>`, so it runs both.

### 9.1 How escaping defeats it

HTML escaping converts the characters that have special meaning in HTML into their entity equivalents, so they render as *text* instead of being parsed as *markup*:

| Character | Escaped to |
| --------- | ---------- |
| `<`       | `&lt;`     |
| `>`       | `&gt;`     |
| `&`       | `&amp;`    |
| `"`       | `&quot;`   |
| `'`       | `&#39;`    |

After escaping, `<script>alert(1)</script>` becomes `&lt;script&gt;alert(1)&lt;/script&gt;`, which the browser displays literally and does not execute.

### 9.2 The EJS escaping rule, stated plainly

- **`<%= value %>` escapes.** This is your default for *all* dynamic content. It is safe even when `value` is fully attacker-controlled.
- **`<%- value %>` does NOT escape.** It outputs raw HTML. Using it with any user-influenced data is a direct XSS hole.

```html
<!-- SAFE: even a malicious comment renders as harmless text -->
<p class="comment"><%= comment.body %></p>

<!-- DANGEROUS: if comment.body contains <script>, it executes -->
<p class="comment"><%- comment.body %></p>
```

### 9.3 When you legitimately need raw HTML

Sometimes you genuinely need to render HTML — a rich-text article body, sanitized Markdown. The rule is: **sanitize first, then output raw.** Never trust raw user HTML.

```js
const sanitizeHtml = require('sanitize-html');

app.get('/posts/:id', async (req, res, next) => {
  const post = await Post.findById(req.params.id).lean();
  // Strip dangerous tags/attributes BEFORE the template renders it raw.
  post.safeBody = sanitizeHtml(post.body, {
    allowedTags: ['p', 'b', 'i', 'em', 'strong', 'a', 'ul', 'ol', 'li', 'code'],
    allowedAttributes: { a: ['href'] },
  });
  res.render('posts/show', { post });
});
```

```html
<!-- Now <%- %> is acceptable because the HTML was sanitized server-side -->
<article><%- post.safeBody %></article>
```

### 9.4 Beyond the template — defense in depth

Output escaping is necessary but not sufficient. Layer additional protections:

- **Escape by context.** HTML escaping is for HTML body content. Data placed inside a `<script>` block, a URL, or an HTML attribute needs *context-specific* escaping. Avoid injecting user data into `<script>` at all; if you must pass data to client JS, serialize it safely: `<script>window.__DATA__ = <%- JSON.stringify(data) %></script>` — and even then, prefer a JSON-safe serializer that escapes `<`, `>`, and U+2028/U+2029.
- **Content-Security-Policy (CSP).** A `Content-Security-Policy` header (set via `helmet`) tells the browser which script sources to trust, neutralizing injected inline scripts even if one slips through.
- **Validate and sanitize input** on the way in, in addition to escaping on the way out.
- **`helmet`** sets a bundle of protective headers (CSP, `X-Content-Type-Options`, etc.) with one line: `app.use(helmet())`.

> BEST PRACTICE: Treat *all* dynamic data as untrusted and escape it with `<%= %>` by default. Make `<%- %>` a code-review red flag: every use should be paired with either an `include()` (trusted partial) or a documented sanitization step. Combine output escaping with a strict CSP via `helmet` for defense in depth.

> WARNING — Pitfall: `<%= %>` escapes for **HTML element content**, not for HTML *attributes* in every edge case, and not for JavaScript or URL contexts. Putting user data into `href="javascript:..."`, an `onclick=`, or a `<style>` block can still be exploited even with `<%= %>`. The safest rule: never place user data in dangerous contexts (inline event handlers, `javascript:` URLs, `<script>`/`<style>` bodies) at all.

---

## Common Pitfalls & Gotchas

- **Relative static paths.** `express.static('public')` resolves against the *current working directory*, not your file. Always `path.join(__dirname, 'public')`.
- **Missing `Content-Type` header.** `express.json()` silently skips bodies that are not `application/json`. "My `req.body` is empty" almost always means the client did not send `Content-Type: application/json` (or sent the wrong type).
- **Forgetting to mount a parser.** No `express.json()` / `express.urlencoded()` → `req.body` is `undefined`. The parser must be registered *before* the route that reads `req.body`.
- **`multipart/form-data` and the built-in parsers.** Neither `json` nor `urlencoded` parses file uploads. Use `multer`. An empty `req.body` on a file form is this mismatch.
- **Omitting `extended`.** Always pass it explicitly to `express.urlencoded({ extended: true|false })` to avoid version-dependent defaults and deprecation noise.
- **No size limit.** Relying on a parser without a sane `limit` invites memory-exhaustion DoS. Set `limit` on every parser.
- **Consuming the raw body twice.** A global `express.json()` consumes the stream; you cannot then recover the exact raw bytes for webhook signature checks. Use route-scoped `express.raw()` or the `verify` hook.
- **`<%- %>` vs `<%= %>`.** `<%- %>` does not escape — using it with user data is XSS. Default to `<%= %>`; use `<%- %>` only for `include()` and pre-sanitized HTML.
- **Escaping an `include`.** `<%= include(...) %>` dumps escaped tag soup onto the page. Use `<%- include(...) %>`.
- **Undefined template variables.** Referencing a local that was not passed throws `ReferenceError` and 500s the page. Pass defaults or guard with `typeof`.
- **SPA catch-all swallowing the API.** Register API routes *before* the `index.html` catch-all, or every API call returns HTML.
- **Express 5 catch-all syntax.** `app.get('*', ...)` throws in Express 5; use `app.get('/*splat', ...)` or a RegExp.
- **Caching `index.html`.** Aggressively caching the SPA entry point pins users to stale asset URLs after deploys. Cache hashed assets forever, `index.html` never.

## Best Practices

- **Serve static assets under a virtual prefix** (`/static`, `/assets`) rather than the root, except for the SPA `index.html`. It avoids route collisions and simplifies CDN setup.
- **Cache content-hashed bundles aggressively** (`maxAge: '1y', immutable: true`) and **never cache `index.html`**.
- **Keep a clean `public`/`dist` directory** with only public assets; never let secrets or source live in a served folder. Add `dotfiles: 'deny'` as defense in depth.
- **Mount `express.json()` and `express.urlencoded({ extended: true })` once, globally**, after security middleware and before routes; scope `raw`/`text` to the specific endpoints that need them.
- **Always set an explicit `limit`** sized to the largest legitimate body. Handle `413`/`400` parser errors in your error middleware for clean responses.
- **Use route-scoped `express.raw()` for webhooks** that need signature verification over exact bytes.
- **Validate parsed bodies against a schema** (Zod, Joi, express-validator). Parsing is not validation; clients lie and urlencoded values are always strings.
- **Default to `<%= %>` everywhere**; make `<%- %>` a review flag paired with `include()` or documented sanitization.
- **Compute presentation-ready data in the controller**, keeping templates focused on display.
- **Set per-request data via `res.locals` middleware** (current user, flash messages) so every template can rely on it.
- **Layer XSS defenses:** output escaping + input validation + `helmet` CSP.
- **Choose SSR vs. JSON API deliberately**, and keep business logic in a service layer so both can share it.

## Key Interview Points

- **What is `express.static` and what does it do for you?** It is Express's built-in middleware (re-exporting `serve-static`) that serves files from a directory: streaming, MIME detection, `ETag`/`Last-Modified`, conditional and range requests, and caching headers — all handled correctly so you do not hand-roll file routes.
- **Why use an absolute path with `express.static`?** Relative paths resolve against `process.cwd()`, which varies with how the process is launched; `path.join(__dirname, ...)` is stable.
- **How do you cache static assets effectively?** Set `Cache-Control: max-age` via `maxAge`; for content-hashed filenames add `immutable` and a one-year max-age. Cache hashed bundles forever, `index.html` never, because the HTML references assets by hashed name.
- **Why is `req.body` undefined without a parser?** The body arrives as a raw byte stream; Express does not read it by default (size/format are unknown). A body-parsing middleware buffers, decodes per `Content-Type`, and sets `req.body`.
- **`express.json()` vs `express.urlencoded()` — when each?** `json()` parses `application/json` (SPA/API/mobile clients); `urlencoded()` parses `application/x-www-form-urlencoded` (native HTML form submits).
- **What does the `extended` option control?** `false` uses Node's `querystring` (flat keys, string values); `true` uses `qs` (nested objects and arrays via bracket syntax). Always set it explicitly.
- **How do you handle file uploads?** Not with the built-in parsers — `multipart/form-data` needs a streaming parser like `multer`/`busboy` that writes to disk/storage rather than buffering into memory.
- **Why set a body size `limit`?** Without it, a large body can exhaust memory and crash the process — a denial-of-service. `limit` caps buffered bytes and returns `413` instead.
- **How do you verify a webhook signature when you also use `express.json()` globally?** Use a route-scoped `express.raw()` so the exact bytes survive, or capture them via the `json()` `verify` hook — re-stringifying parsed JSON does not reproduce the original bytes.
- **SSR vs. JSON API — how do you decide?** SSR for content sites/admin panels/SEO/fast first paint with minimal build tooling; JSON API for rich SPAs, multiple clients sharing one backend, and independent frontend/backend deployment. Many apps do both.
- **In EJS, what is the difference between `<%= %>` and `<%- %>`?** `<%= %>` HTML-escapes output (XSS-safe, the default); `<%- %>` outputs raw HTML (unsafe with user data; used for `include()` and pre-sanitized content).
- **How do you prevent XSS in templates?** Escape output by default (`<%= %>`), sanitize any HTML you must render raw, escape by context, and add a CSP via `helmet`.
- **How do you serve a SPA build from Express so client routing works on refresh?** Serve the hashed assets statically, then add a catch-all GET that returns `index.html` for non-API paths — registered *after* the API routes. In Express 5 the catch-all is `app.get('/*splat', ...)`, not `app.get('*', ...)`.
- **Where should body parsers sit in the middleware stack?** After security middleware (`helmet`, CORS) and before the routes that read `req.body`; scope `raw`/`text` to specific endpoints.

## Practice Exercises

Work through these in order — each builds on the previous. Try them before reading the solutions.

1. **Static with caching.** Create an Express app that serves a `public` directory under the virtual prefix `/static`, with a one-day cache (`maxAge`). Add an `images` subfolder containing a PNG and confirm in the browser dev tools Network tab that the response includes a `Cache-Control: public, max-age=86400` header and an `ETag`.

2. **JSON echo with a size limit.** Build a `POST /echo` route that parses JSON bodies (limit `50kb`) and responds with `{ received: <the body>, length: <number of top-level keys> }`. Add error middleware that returns `413` for oversized bodies and `400` for malformed JSON. Test all three cases with `curl`.

3. **Form vs. JSON on one app.** Serve a static HTML page at `/` with a contact form (`name`, `message`) that POSTs to `/contact` as urlencoded. Also expose `POST /api/contact` that accepts the same data as JSON. Both should store the message in an in-memory array and respond appropriately (the form route redirects to `/thanks`; the API route returns `201` JSON).

4. **EJS-rendered list.** Configure EJS. Create a `GET /products` route that renders a `products/list.ejs` template showing an array of products (name + price) in a table, with a "no products" message when the list is empty. Use a `header` and `footer` partial. Pull `siteName` and `year` from `app.locals`.

5. **XSS demonstration.** Add a `GET /search?q=...` route that renders the search term back to the page. First render it with `<%- %>` and confirm that `?q=<script>alert(1)</script>` executes (in a throwaway/local context only). Then switch to `<%= %>` and confirm it now renders as harmless text. Write a one-paragraph explanation of why.

6. **Serve a SPA build (challenge).** Given a `client/dist` folder (you can fake it with a hand-written `index.html` that includes a script referencing `/assets/app.[hash].js`), wire up Express to: serve `/api/health` returning JSON; serve the hashed assets with `immutable` one-year caching; and return `index.html` for any other GET so that visiting `/some/deep/route` and refreshing still loads the app. Ensure `index.html` is served with no caching. Make it work under Express 5 catch-all syntax.

### Solutions

#### Solution 1 — Static with caching

```js
// server.js
const path = require('node:path');
const express = require('express');
const app = express();

app.use(
  '/static',
  express.static(path.join(__dirname, 'public'), {
    maxAge: '1d',     // → Cache-Control: public, max-age=86400
    etag: true,       // default; enables 304 revalidation after expiry
    lastModified: true,
  })
);

app.listen(3000, () => console.log('http://localhost:3000/static/images/logo.png'));
```

```text
project/
├── server.js
└── public/
    ├── style.css
    └── images/
        └── logo.png
```

Verify the headers:

```bash
curl -I http://localhost:3000/static/style.css
# Cache-Control: public, max-age=86400
# ETag: W/"..."
# Last-Modified: ...
```

#### Solution 2 — JSON echo with a size limit

```js
// echo.js
const express = require('express');
const app = express();

app.use(express.json({ limit: '50kb' })); // cap at 50kb

app.post('/echo', (req, res) => {
  const body = req.body ?? {};
  res.json({
    received: body,
    length: Object.keys(body).length, // number of top-level keys
  });
});

// Error middleware AFTER routes: handles parser errors.
app.use((err, req, res, next) => {
  if (err.type === 'entity.too.large') {
    return res.status(413).json({ error: 'Body too large (max 50kb)' });
  }
  if (err.type === 'entity.parse.failed') {
    return res.status(400).json({ error: 'Malformed JSON' });
  }
  next(err);
});

app.listen(3000);
```

```bash
# Success
curl -s -X POST localhost:3000/echo -H 'Content-Type: application/json' \
  -d '{"a":1,"b":2}'
# -> {"received":{"a":1,"b":2},"length":2}

# Malformed JSON -> 400
curl -s -X POST localhost:3000/echo -H 'Content-Type: application/json' \
  -d '{not json}'
# -> {"error":"Malformed JSON"}

# Oversized -> 413  (generate a >50kb payload)
node -e "process.stdout.write(JSON.stringify({x:'a'.repeat(60000)}))" > big.json
curl -s -X POST localhost:3000/echo -H 'Content-Type: application/json' \
  --data-binary @big.json
# -> {"error":"Body too large (max 50kb)"}
```

#### Solution 3 — Form vs. JSON on one app

```js
// contact-app.js
const path = require('node:path');
const express = require('express');
const app = express();

const messages = []; // in-memory store

app.use(express.urlencoded({ extended: false })); // for the HTML form
app.use(express.json());                           // for the API
app.use(express.static(path.join(__dirname, 'public')));

// HTML form submission → store and redirect (Post/Redirect/Get)
app.post('/contact', (req, res) => {
  const { name, message } = req.body;
  if (!name || !message) return res.status(400).send('Missing fields');
  messages.push({ name, message, via: 'form', at: new Date() });
  res.redirect('/thanks');
});

// JSON API submission → store and return 201 JSON
app.post('/api/contact', (req, res) => {
  const { name, message } = req.body ?? {};
  if (!name || !message) {
    return res.status(400).json({ error: 'name and message are required' });
  }
  const entry = { id: messages.length + 1, name, message, via: 'api', at: new Date() };
  messages.push(entry);
  res.status(201).json(entry);
});

app.get('/thanks', (req, res) => res.send('<h1>Thanks for your message!</h1>'));

app.listen(3000);
```

```html
<!-- public/index.html -->
<!DOCTYPE html>
<html lang="en">
<head><meta charset="utf-8" /><title>Contact</title></head>
<body>
  <form action="/contact" method="POST">
    <input name="name" placeholder="Your name" required />
    <textarea name="message" placeholder="Your message" required></textarea>
    <button>Send</button>
  </form>
</body>
</html>
```

```bash
# API path:
curl -s -X POST localhost:3000/api/contact -H 'Content-Type: application/json' \
  -d '{"name":"Ada","message":"Hi"}'
# -> {"id":1,"name":"Ada","message":"Hi","via":"api","at":"..."}
```

#### Solution 4 — EJS-rendered list

```js
// products-app.js
const path = require('node:path');
const express = require('express');
const app = express();

app.set('view engine', 'ejs');
app.set('views', path.join(__dirname, 'views'));

app.locals.siteName = 'Acme Store';
app.locals.year = new Date().getFullYear();

app.get('/products', (req, res) => {
  const products = [
    { id: 1, name: 'Notebook', price: 4.5 },
    { id: 2, name: 'Pen', price: 1.25 },
  ];
  res.render('products/list', { title: 'Products', products });
});

app.listen(3000);
```

```html
<!-- views/partials/header.ejs -->
<!DOCTYPE html>
<html lang="en">
<head><meta charset="utf-8" /><title><%= title %> — <%= siteName %></title></head>
<body>
  <h1><%= siteName %></h1>
```

```html
<!-- views/partials/footer.ejs -->
  <footer>&copy; <%= year %> <%= siteName %></footer>
</body>
</html>
```

```html
<!-- views/products/list.ejs -->
<%- include('../partials/header', { title: title }) %>

<h2><%= title %></h2>

<% if (products.length === 0) { %>
  <p>No products found.</p>
<% } else { %>
  <table border="1" cellpadding="6">
    <thead><tr><th>Name</th><th>Price</th></tr></thead>
    <tbody>
      <% products.forEach(function (p) { %>
        <tr>
          <td><%= p.name %></td>
          <td>$<%= p.price.toFixed(2) %></td>
        </tr>
      <% }); %>
    </tbody>
  </table>
<% } %>

<%- include('../partials/footer') %>
```

#### Solution 5 — XSS demonstration

```js
// xss-demo.js  (run LOCALLY only — it intentionally contains a vulnerability)
const path = require('node:path');
const express = require('express');
const app = express();

app.set('view engine', 'ejs');
app.set('views', path.join(__dirname, 'views'));

app.get('/search', (req, res) => {
  res.render('search', { q: req.query.q || '' });
});

app.listen(3000);
```

```html
<!-- views/search.ejs -->
<!DOCTYPE html>
<html lang="en">
<body>
  <!-- VULNERABLE version: raw output executes injected script -->
  <!-- <p>You searched for: <%- q %></p> -->

  <!-- SAFE version: escaped output renders as text -->
  <p>You searched for: <%= q %></p>
</body>
</html>
```

**Explanation.** With `<%- q %>` (raw), visiting `/search?q=<script>alert(1)</script>` writes the literal `<script>` tag into the HTML the server returns; the browser parses it as markup and executes the script — a reflected XSS. With `<%= q %>` (escaped), EJS converts `<`, `>`, `&`, `"`, and `'` to HTML entities (`&lt;script&gt;...`), so the browser displays the characters as visible text and never treats them as an executable tag. Escaping works because it removes the attacker's ability to "break out" of text context and introduce new HTML elements.

#### Solution 6 — Serve a SPA build (challenge)

```js
// spa-server.js  (Express 5)
const path = require('node:path');
const express = require('express');
const app = express();

const distDir = path.join(__dirname, 'client', 'dist');

// 1. API first
app.use(express.json());
app.get('/api/health', (req, res) => res.json({ status: 'ok' }));
// Unknown API paths get JSON 404, not HTML:
app.use('/api', (req, res) => res.status(404).json({ error: 'Not found' }));

// 2. Hashed assets, cached for a year, immutable. index.html excluded here.
app.use(
  express.static(distDir, { index: false, maxAge: '1y', immutable: true })
);

// 3. Catch-all → index.html with NO caching, so deploys take effect.
app.get('/*splat', (req, res) => {
  res.sendFile(path.join(distDir, 'index.html'), {
    headers: { 'Cache-Control': 'no-cache' },
  });
});

app.listen(3000);
```

```html
<!-- client/dist/index.html (hand-written stand-in for a real build) -->
<!DOCTYPE html>
<html lang="en">
<head><meta charset="utf-8" /><title>My SPA</title></head>
<body>
  <div id="root"></div>
  <script src="/assets/app.4f3a9c2b.js"></script>
</body>
</html>
```

```text
client/dist/
├── index.html
└── assets/
    └── app.4f3a9c2b.js
```

Verification:

- `GET /api/health` → `{"status":"ok"}`.
- `GET /api/does-not-exist` → `404 {"error":"Not found"}` (JSON, not HTML).
- `GET /assets/app.4f3a9c2b.js` → served with `Cache-Control: public, max-age=31536000, immutable`.
- `GET /any/deep/route` → returns `index.html` with `Cache-Control: no-cache`, so the SPA boots and its client router renders the right view on refresh.

## Summary / Cheat Sheet

| Task | Code |
| ---- | ---- |
| Serve a static dir | `app.use(express.static(path.join(__dirname, 'public')))` |
| With a virtual prefix | `app.use('/static', express.static(dir))` |
| Cache hashed assets | `express.static(dir, { maxAge: '1y', immutable: true })` |
| Parse JSON bodies | `app.use(express.json({ limit: '1mb' }))` |
| Parse HTML forms | `app.use(express.urlencoded({ extended: true, limit: '1mb' }))` |
| Parse plain text | `app.use(express.text())` → `req.body` is a string |
| Parse raw bytes | `express.raw({ type: 'application/json' })` → `req.body` is a Buffer |
| Webhook raw body | route-scoped `express.raw()` *or* `json({ verify })` |
| File uploads | `multer` (not a built-in parser; `multipart/form-data`) |
| Set view engine | `app.set('view engine', 'ejs')` |
| Set views dir | `app.set('views', path.join(__dirname, 'views'))` |
| Render a template | `res.render('view', { title, data })` |
| App-wide template data | `app.locals.x = ...` |
| Per-request template data | middleware sets `res.locals.x = ...` |
| EJS escaped output (default) | `<%= value %>` |
| EJS raw output (dangerous) | `<%- value %>` |
| EJS logic (no output) | `<% if/for ... %>` |
| EJS include a partial | `<%- include('partials/header', { title }) %>` |
| SPA catch-all (Express 5) | `app.get('/*splat', (req, res) => res.sendFile(indexHtml))` |
| SPA catch-all (Express 4) | `app.get('*', (req, res) => res.sendFile(indexHtml))` |
| Handle 413 / 400 parser errors | check `err.type === 'entity.too.large'` / `'entity.parse.failed'` |

**Mental model:**

- **Static out:** `express.static` streams files with correct headers — absolute paths, virtual prefix, aggressive caching for hashed assets, none for `index.html`.
- **Bytes in:** request bodies are raw streams; mount a parser (`json`/`urlencoded`/`raw`/`text`) matched to `Content-Type`, always with a `limit`, then **validate**.
- **HTML vs. JSON:** native forms → `urlencoded` + redirect; API/SPA clients → `json` + JSON response.
- **SSR vs. API:** render HTML with EJS when the *server* owns the page; return JSON when a *client* owns the page; many apps do both.
- **Safety:** default to escaped `<%= %>`; reserve raw `<%- %>` for `include()` and sanitized HTML; layer in `helmet`/CSP.
