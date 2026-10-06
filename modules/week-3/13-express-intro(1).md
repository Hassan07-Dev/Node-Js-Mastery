# Introduction to Express.js

You have spent the last two weeks learning Node.js from the inside out — the event loop, modules, the file system, streams, events, and the core modules including the built-in `http` server. You can already build a working web server with nothing but `node:http`. So why does virtually every Node web application in production use **Express.js**?

This module answers that question from first principles. Express is the most widely used web framework for Node, and it is the foundation the rest of this course builds on (routing, middleware, REST APIs, MongoDB integration, authentication, error handling — all of it sits on top of Express). Before we sprint into routing and middleware in later modules, we slow down and build a rock-solid mental model: **Express is a thin, unopinionated layer over the Node `http` module that turns the painful, repetitive parts of writing an HTTP server into a clean, declarative API.**

By the end of this module you will understand exactly what Express is, what problems it solves, how it relates to the raw `http` module, how to create and configure an application, how to define your first routes and send responses, how to set up a productive development loop with `nodemon`, how to structure a starter project, and the differences between Express 4 and Express 5 that interviewers love to probe.

> TIP: Throughout this course we use the `node:` prefix for built-in modules (e.g. `require('node:http')`) to make it unambiguous that we mean the built-in, never a same-named npm package. Express itself is an npm package, so it is imported as plain `require('express')`.

## What You'll Learn

- **What Express is** — a minimal, unopinionated web framework — and the concrete problems it solves over the raw `http` module.
- How to **install Express** and create your first server with the application object and `app.listen()`.
- The **application object** (`app`) and its most important methods and properties.
- How to **define routes** and **send responses** with `res.send()` and `res.json()` (and friends).
- How to use **`nodemon`** for automatic reloads during development, and why you should never ship it to production.
- A **sensible starter project structure** that scales from a toy server to a real API.
- How **Express sits on top of Node's `http` module** — that an Express `app` is literally a request-handler function.
- **Express 4 vs Express 5** — the key differences every Node developer is expected to know.
- **Application settings** with `app.set()`/`app.get()`, **environment-aware configuration**, and **`app.locals`** for app-wide data.

---

## What Is Express, and What Problems Does It Solve?

**Express.js is a minimal, flexible, unopinionated web application framework for Node.js.** It provides a thin layer of fundamental web-application features — routing, middleware, request/response helpers, view-engine integration — without obscuring the Node.js features you already know.

To appreciate *why* it exists, let us build a small server with the raw `http` module and feel the pain. Suppose we want three endpoints:

- `GET /` → an HTML greeting
- `GET /api/users` → a JSON list of users
- `POST /api/users` → accept a JSON body and echo it back

Here is what that looks like with **only** the core `http` module:

```js
// raw-http-server.js — the "before" picture
const http = require('node:http');

const users = [{ id: 1, name: 'Ada' }];

const server = http.createServer((req, res) => {
  // 1. We must hand-parse the method and the URL path ourselves.
  const { method, url } = req;

  // 2. Routing is a giant, error-prone if/else (or switch) ladder.
  if (method === 'GET' && url === '/') {
    res.statusCode = 200;
    res.setHeader('Content-Type', 'text/html; charset=utf-8'); // set headers by hand
    res.end('<h1>Hello from raw http</h1>');
    return;
  }

  if (method === 'GET' && url === '/api/users') {
    res.statusCode = 200;
    res.setHeader('Content-Type', 'application/json'); // remember the right header
    res.end(JSON.stringify(users));                    // serialize JSON by hand
    return;
  }

  if (method === 'POST' && url === '/api/users') {
    // 3. The body arrives as a STREAM of chunks; we must buffer it ourselves.
    let body = '';
    req.on('data', (chunk) => { body += chunk; });
    req.on('end', () => {
      try {
        const parsed = JSON.parse(body); // parse + guard against bad JSON
        res.statusCode = 201;
        res.setHeader('Content-Type', 'application/json');
        res.end(JSON.stringify({ received: parsed }));
      } catch {
        res.statusCode = 400;
        res.end(JSON.stringify({ error: 'Invalid JSON' }));
      }
    });
    return;
  }

  // 4. We must remember to handle "nothing matched" or the request hangs.
  res.statusCode = 404;
  res.setHeader('Content-Type', 'application/json');
  res.end(JSON.stringify({ error: 'Not found' }));
});

server.listen(3000, () => console.log('Raw server on http://localhost:3000'));
```

That works, but notice everything we had to do manually: match methods and paths with `if`/`else`, set status codes and `Content-Type` headers by hand, serialize JSON ourselves, buffer the request body from a stream, guard against malformed input, and remember a catch-all 404. Now multiply that across dozens of routes, add query-string parsing, cookies, file uploads, sessions, and reusable cross-cutting logic (logging, auth checks) — and the raw approach collapses under its own weight.

Here is the **exact same** server in Express:

```js
// express-server.js — the "after" picture
const express = require('express');
const app = express();

app.use(express.json()); // built-in middleware: parse JSON bodies for us

const users = [{ id: 1, name: 'Ada' }];

app.get('/', (req, res) => {
  res.send('<h1>Hello from Express</h1>'); // sets text/html + status 200 automatically
});

app.get('/api/users', (req, res) => {
  res.json(users); // serializes + sets application/json automatically
});

app.post('/api/users', (req, res) => {
  // req.body is already parsed by express.json()
  res.status(201).json({ received: req.body });
});

// One concise catch-all 404.
app.use((req, res) => {
  res.status(404).json({ error: 'Not found' });
});

app.listen(3000, () => console.log('Express server on http://localhost:3000'));
```

Shorter, clearer, and harder to get wrong. That is the value proposition.

### The concrete problems Express solves

| Problem with raw `http` | How Express solves it |
| --- | --- |
| Routing is a manual `if`/`switch` ladder on `method` + `url` | Declarative routing: `app.get(path, handler)`, `app.post(...)`, routers, route params (`/users/:id`) |
| You parse the URL path, query string, and params yourself | `req.path`, `req.query`, `req.params` are populated for you |
| Request bodies arrive as raw streams you must buffer and parse | Built-in body parsers: `express.json()`, `express.urlencoded()` populate `req.body` |
| Responses require manual status codes, headers, and serialization | Response helpers: `res.send()`, `res.json()`, `res.status()`, `res.redirect()`, `res.sendFile()` |
| Cross-cutting logic (logging, auth, CORS) is copy-pasted into every handler | **Middleware** — composable functions that run in a pipeline via `app.use()` |
| No structured error handling; one throw can crash the process | Centralized error-handling middleware (`(err, req, res, next) => {}`) |
| No first-class way to render HTML templates | Pluggable **view engines** (EJS, Pug, Handlebars) via `res.render()` |
| Everything lives in one giant callback | Modular structure: routers, controllers, and middleware in separate files |

> BEST PRACTICE: Express is described as **unopinionated**, meaning it does not force a folder structure, an ORM, or an architecture on you. That flexibility is a double-edged sword: it is powerful, but *you* must impose discipline. This module's "starter project structure" section gives you a sane default so you do not reinvent it on every project.

### Where Express fits in the ecosystem

Express is a **framework**, not a runtime and not a server. The layering looks like this:

```text
Your application code (routes, controllers, business logic)
        │
        ▼
   Express.js          ← routing, middleware, req/res helpers
        │
        ▼
 Node.js http module   ← createServer, IncomingMessage, ServerResponse
        │
        ▼
   libuv / OS sockets  ← the event loop, TCP, the network
```

Express does not replace the `http` module — it *wraps* it. We will make that concrete shortly. Other frameworks (Fastify, Koa, NestJS, Hapi) occupy the same layer and offer different trade-offs, but Express remains the most widely deployed and the lingua franca of Node backend interviews.

---

## Installing Express and Creating Your First Server

### Prerequisites

You need Node.js (current LTS) and npm, which ships with Node. Verify both:

```bash
node --version   # e.g. v22.x (current LTS) — Express 5 requires Node 18+
npm --version    # e.g. 10.x
```

> WARNING — Pitfall: **Express 5 requires Node.js 18 or newer.** If you are on an older Node, either upgrade (recommended) or pin Express 4. Running Express 5 on Node 16 or below will fail or behave unexpectedly.

### Step 1 — Initialize a project

```bash
mkdir my-first-express && cd my-first-express
npm init -y                 # creates package.json with sensible defaults
```

### Step 2 — Install Express

```bash
npm install express         # adds express to dependencies and installs it
```

As of mid-2025, `npm install express` installs **Express 5**, which is now the default `latest` tag. To be explicit (recommended in real projects so an unexpected major bump never surprises you):

```bash
npm install express@5       # explicitly install the v5 line
# or, if a project still needs the older line:
npm install express@4       # explicitly install the v4 line
```

After installation your `package.json` `dependencies` will contain something like `"express": "^5.x.x"`, and Express lives in `node_modules/express`.

> TIP: The caret (`^`) in `"^5.1.0"` allows compatible updates within the same major version (>=5.1.0 <6.0.0). Combined with the auto-generated `package-lock.json`, this gives you reproducible installs. Commit `package-lock.json`; do **not** commit `node_modules`.

### Step 3 — Write the server

Create `app.js`:

```js
// app.js — the smallest useful Express server
const express = require('express'); // 1. import the Express factory function

const app = express();              // 2. create an application instance
const PORT = 3000;

// 3. define a single route: respond to GET requests at "/"
app.get('/', (req, res) => {
  res.send('Hello, Express!');      // send a response and end the request
});

// 4. start listening for connections on PORT
app.listen(PORT, () => {
  console.log(`Server running at http://localhost:${PORT}`);
});
```

### Step 4 — Run it

```bash
node app.js
# → Server running at http://localhost:3000
```

Open `http://localhost:3000` in a browser, or test from the terminal:

```bash
curl http://localhost:3000
# → Hello, Express!
```

Let us dissect the four lines that matter.

1. **`const express = require('express')`** — Express's main export is a **factory function**. Calling it does not start anything; it just manufactures an application.
2. **`const app = express()`** — This `app` is the heart of everything. It is an object with methods like `.get()`, `.post()`, `.use()`, `.set()`, and `.listen()`. (It is also, secretly, a function — more on that soon.)
3. **`app.get('/', handler)`** — Registers a **route handler**: "when a `GET` request arrives for path `/`, run this function." The handler receives a `req` (request) and `res` (response) object.
4. **`app.listen(PORT, callback)`** — Binds to the TCP port and begins accepting connections. The callback fires once the server is ready.

### `app.listen()` in detail

`app.listen()` is a thin convenience wrapper around the Node `http` module. These two snippets are equivalent:

```js
// The convenience form:
app.listen(3000, () => console.log('ready'));

// What it does under the hood — Express creates an http server for you:
const http = require('node:http');
http.createServer(app).listen(3000, () => console.log('ready'));
```

`app.listen()` returns the underlying `http.Server` instance, which is occasionally useful (for graceful shutdown, or for attaching a WebSocket server to the same port):

```js
// capture the server instance for later control
const server = app.listen(3000, () => console.log('ready'));

// graceful shutdown on Ctrl+C
process.on('SIGINT', () => {
  server.close(() => {
    console.log('Server closed. Bye!');
    process.exit(0);
  });
});
```

> TIP: A common signature variant lets you bind a specific host and pass a backlog: `app.listen(port, hostname, backlog, callback)`. In containers you sometimes need `app.listen(PORT, '0.0.0.0')` so the server is reachable from outside the container, not just from `localhost`.

> WARNING — Pitfall: If you call `app.listen()` twice, or two processes try to bind the same port, you get `EADDRINUSE`. During development this usually means an old `node` process is still running — find and kill it (`lsof -i :3000` then `kill <pid>`), or let `nodemon` manage the lifecycle for you.

---

## How Express Sits on Top of Node's `http` Module

This is one of the most important — and most under-appreciated — facts about Express, and it is a classic interview question: **what *is* an Express `app`, really?**

The answer: **an Express `app` is a request-handler function.** Specifically, it is a function with the signature `(req, res) => { ... }` — exactly the kind of callback `http.createServer()` expects. Express then bolts a pile of methods (`.get`, `.use`, `.set`, `.listen`, …) onto that function object.

Because functions are objects in JavaScript, Express can do both: `app` is callable *and* it has properties. This is why you can hand `app` directly to `http.createServer()`:

```js
// app-is-a-handler.js — prove that app is just a request handler
const express = require('express');
const http = require('node:http');

const app = express();

app.get('/', (req, res) => res.send('Served via createServer(app)'));

// Pass the Express app straight into Node's http server as the request handler.
const server = http.createServer(app);
server.listen(3000, () => console.log('Listening on 3000'));
```

When a request arrives, Node's `http` server invokes `app(req, res)`. Express then runs its **middleware + routing pipeline** against that `req`/`res` pair: it matches the method and path, runs any matching middleware in order, calls your route handler, and ensures a response is sent.

This wrapping has practical consequences:

- **`req` is a Node `http.IncomingMessage`** that Express *augments* with extra properties (`req.params`, `req.query`, `req.body`, `req.path`, helper methods like `req.get()`). All the raw Node properties (`req.method`, `req.url`, `req.headers`) are still there.
- **`res` is a Node `http.ServerResponse`** that Express augments with helpers (`res.send`, `res.json`, `res.status`, `res.redirect`). Under the hood, `res.send()` eventually calls Node's `res.end()`.
- You can run **multiple Express apps** behind one HTTP/HTTPS server, or run the same app on both HTTP and HTTPS:

```js
// http-and-https.js — one app, two protocols
const express = require('express');
const http = require('node:http');
const https = require('node:https');
const fs = require('node:fs');

const app = express();
app.get('/', (req, res) => res.send('hi'));

// Plain HTTP on 80
http.createServer(app).listen(80);

// HTTPS on 443 using TLS certificates
const tlsOptions = {
  key: fs.readFileSync('server.key'),
  cert: fs.readFileSync('server.cert'),
};
https.createServer(tlsOptions, app).listen(443);
```

> TIP: Knowing that `app` is "just" a request handler demystifies a lot of Express. Middleware, routing, sub-apps (`app.use('/admin', adminApp)`), and testing tools like Supertest (which can drive your app without binding a real port) all flow from this single idea.

---

## The Application Object and Its Key Methods & Properties

The object returned by `express()` is your central control panel. Here are the members you will reach for constantly, grouped by purpose.

### Routing methods

| Method | Purpose |
| --- | --- |
| `app.get(path, ...handlers)` | Handle `GET` requests for `path` |
| `app.post(path, ...handlers)` | Handle `POST` requests |
| `app.put(path, ...handlers)` | Handle `PUT` requests |
| `app.patch(path, ...handlers)` | Handle `PATCH` requests |
| `app.delete(path, ...handlers)` | Handle `DELETE` requests |
| `app.all(path, ...handlers)` | Handle **every** HTTP method for `path` |
| `app.route(path)` | Start a chainable route definition (avoids repeating the path) |

```js
// app.route lets you group handlers for one path by method
app.route('/books')
  .get((req, res) => res.send('list books'))     // GET /books
  .post((req, res) => res.send('create a book')); // POST /books
```

### Middleware & configuration methods

| Method | Purpose |
| --- | --- |
| `app.use([path], ...handlers)` | Mount middleware (and sub-routers/sub-apps) into the pipeline |
| `app.set(name, value)` | Set an application setting |
| `app.get(name)` | **Read** a setting (when called with one string arg — see the gotcha below) |
| `app.enable(name)` / `app.disable(name)` | Set a boolean setting to `true` / `false` |
| `app.enabled(name)` / `app.disabled(name)` | Read a boolean setting |
| `app.engine(ext, fn)` | Register a template/view engine for a file extension |
| `app.param(name, fn)` | Add a trigger that runs when a given route parameter is present |

### Lifecycle & server methods

| Method | Purpose |
| --- | --- |
| `app.listen(...)` | Create an HTTP server and start listening; returns the `http.Server` |

### Key properties

| Property | What it holds |
| --- | --- |
| `app.locals` | An object of variables available to **every** request and to rendered views (app-wide state) |
| `app.mountpath` | The path pattern(s) on which a sub-app is mounted |
| `app.settings` | The internal object backing `app.set`/`app.get` (rarely accessed directly) |

> WARNING — Pitfall: `app.get` is **overloaded**. `app.get('/users', handler)` *registers a route*, but `app.get('view engine')` *reads a setting*. Express decides based on the number/type of arguments: one string argument → read a setting; a path plus handler(s) → define a route. This is a frequent source of confusion. We cover settings in depth below.

### A tour in one file

```js
// app-object-tour.js — many app members at a glance
const express = require('express');
const app = express();

// Settings
app.set('view engine', 'ejs');         // configure a view engine
app.set('x-powered-by', false);        // hide the X-Powered-By header (or use app.disable)
app.disable('etag');                   // boolean setting → off

// App-wide data shared with every request/view
app.locals.siteName = 'My Store';
app.locals.year = new Date().getFullYear();

// Middleware
app.use(express.json());               // parse JSON bodies
app.use((req, res, next) => {          // a custom logging middleware
  console.log(`${req.method} ${req.path}`);
  next();                              // pass control to the next handler
});

// Routes
app.get('/', (req, res) => res.send(`Welcome to ${app.locals.siteName}`));
app.route('/items')
  .get((req, res) => res.json([{ id: 1 }]))
  .post((req, res) => res.status(201).json(req.body));

// Read a setting back
console.log('View engine is:', app.get('view engine')); // → ejs

app.listen(3000, () => console.log('up on 3000'));
```

---

## Defining Your First Routes and Sending Responses

A **route** maps an HTTP method + URL pattern to one or more handler functions. The handler signature is `(req, res, next)`. You name the parameters yourself, but the conventional names are `req`, `res`, and `next`.

### The anatomy of a route

```js
//  method  path        handler
//   ▼        ▼            ▼
app.get('/products', (req, res) => {
  res.json([{ id: 1, name: 'Widget' }]);
});
```

### Route parameters, query strings, and the request object

Express parses the URL for you and exposes the pieces:

```js
// request-inputs.js
const express = require('express');
const app = express();
app.use(express.json());

// Route PARAMETERS — named segments in the path, available on req.params
app.get('/users/:id', (req, res) => {
  // GET /users/42  →  req.params.id === '42'  (always a string)
  res.send(`User id is ${req.params.id}`);
});

// QUERY STRING — everything after "?", parsed into req.query
app.get('/search', (req, res) => {
  // GET /search?q=node&page=2  →  req.query = { q: 'node', page: '2' }
  res.json({ term: req.query.q, page: req.query.page });
});

// REQUEST BODY — parsed by express.json() into req.body
app.post('/echo', (req, res) => {
  res.json({ youSent: req.body });
});

app.listen(3000);
```

Other useful `req` members you will meet constantly:

| Member | Meaning |
| --- | --- |
| `req.method` | `'GET'`, `'POST'`, … |
| `req.path` | Path portion of the URL, e.g. `/users/42` |
| `req.originalUrl` | The full original URL including query string |
| `req.params` | Route parameters as an object |
| `req.query` | Parsed query string as an object |
| `req.body` | Parsed request body (requires a body-parsing middleware) |
| `req.headers` / `req.get(name)` | Request headers |
| `req.ip` | The client's IP address |

### Sending responses

The response object provides ergonomic helpers. The two you will use most are `res.send()` and `res.json()`.

#### `res.send()` — smart, content-aware response

`res.send(body)` inspects the type of `body` and does the right thing:

```js
// res-send.js
app.get('/text', (req, res) => {
  res.send('plain string');
  // → Content-Type: text/html; charset=utf-8  (strings default to HTML)
});

app.get('/html', (req, res) => {
  res.send('<h1>An HTML string</h1>'); // also text/html
});

app.get('/object', (req, res) => {
  res.send({ ok: true });
  // → if you pass an object/array, send() delegates to res.json() for you
  //   Content-Type: application/json
});

app.get('/buffer', (req, res) => {
  res.send(Buffer.from('raw bytes'));
  // → Content-Type: application/octet-stream
});
```

`res.send()` also sets `Content-Length`, handles `ETag` generation, and ends the response. You do **not** call `res.end()` after it.

#### `res.json()` — explicit JSON

`res.json(value)` serializes `value` with `JSON.stringify()` and sets `Content-Type: application/json`. Use it whenever you mean to return JSON — it is clearer than relying on `res.send()`'s type inference, and it correctly serializes values that `res.send()` would otherwise stringify ambiguously.

```js
// res-json.js
app.get('/api/status', (req, res) => {
  res.json({ status: 'ok', time: Date.now() });
});

// res.json honors the "json spaces" and "json replacer" app settings:
app.set('json spaces', 2); // pretty-print responses with 2-space indentation

// You can also send arrays and primitives:
app.get('/api/ids', (req, res) => res.json([1, 2, 3]));
app.get('/api/answer', (req, res) => res.json(42)); // → 42
```

> TIP: Prefer `res.json()` over `res.send()` for API endpoints even when sending objects. It documents intent ("this is a JSON API") and avoids surprises with non-object JSON values like `null`, numbers, and booleans.

#### Setting status codes

`res.status(code)` sets the HTTP status and is **chainable**:

```js
// status-codes.js
app.post('/api/users', (req, res) => {
  if (!req.body.name) {
    return res.status(400).json({ error: 'name is required' }); // 400 Bad Request
  }
  // ... create the user ...
  res.status(201).json({ id: 1, name: req.body.name });          // 201 Created
});

app.get('/missing', (req, res) => {
  res.sendStatus(404); // sets status AND sends the status text "Not Found" as the body
});
```

> WARNING — Pitfall: Notice the `return` before `res.status(400).json(...)`. Sending a response does **not** stop the function — without `return`, execution continues and you may try to send a *second* response, triggering the dreaded `Error [ERR_HTTP_HEADERS_SENT]: Cannot set headers after they are sent to the client`. Always `return` after you respond inside a branch, or use `else`.

#### Other response helpers

| Helper | Purpose |
| --- | --- |
| `res.status(code)` | Set the status code (chainable) |
| `res.sendStatus(code)` | Set status and send its standard message as the body |
| `res.json(obj)` | Send JSON |
| `res.send(body)` | Send a content-aware response |
| `res.redirect([status,] url)` | Send a redirect (default 302) |
| `res.set(field, value)` / `res.header(...)` | Set a response header |
| `res.type(type)` | Set the `Content-Type` |
| `res.cookie(name, value, opts)` | Set a cookie |
| `res.sendFile(path)` | Stream a file as the response |
| `res.render(view, locals)` | Render a view template (needs a view engine) |
| `res.end()` | End the response with no body (Node-level) |

```js
// more-responses.js
app.get('/old', (req, res) => res.redirect(301, '/new'));   // permanent redirect
app.get('/new', (req, res) => res.send('the new place'));
app.get('/download', (req, res) => {
  res.set('X-Custom', 'value');           // custom header
  res.type('text/plain');                 // explicit content type
  res.send('file-ish content');
});
```

> BEST PRACTICE: Each request must end in **exactly one** response. If a handler can take several paths, make sure every path either sends a response or calls `next()` to defer — never both, and never neither (the latter leaves the request hanging until it times out).

---

## Using `nodemon` for Auto-Reload During Development

When you run `node app.js`, the process loads your code once. Edit a file and the running server keeps the *old* code in memory — you must stop it (`Ctrl+C`) and restart it to see changes. That stop/start cycle, repeated hundreds of times a day, is a productivity tax.

**`nodemon`** ("node monitor") watches your project files and automatically restarts the Node process whenever a watched file changes. It is a *development* tool only.

### Installing nodemon

Install it as a **dev dependency** so it is not shipped to production:

```bash
npm install --save-dev nodemon   # or: npm i -D nodemon
```

### Running with nodemon

```bash
npx nodemon app.js
# [nodemon] starting `node app.js`
# Server running at http://localhost:3000
# ... edit app.js and save ...
# [nodemon] restarting due to changes...
# [nodemon] starting `node app.js`
```

`npx` runs the locally installed binary. Even better, add scripts to `package.json` so collaborators do not need to remember the command:

```json
{
  "name": "my-first-express",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "dev": "nodemon app.js"
  },
  "dependencies": {
    "express": "^5.1.0"
  },
  "devDependencies": {
    "nodemon": "^3.1.0"
  }
}
```

Now the workflow is:

```bash
npm run dev    # development: auto-restarts on save
npm start      # production-like: plain node, no watcher
```

### Configuring nodemon

Drop a `nodemon.json` in the project root to control what it watches and ignores:

```json
{
  "watch": ["app.js", "src"],
  "ext": "js,json",
  "ignore": ["node_modules", "logs", "*.test.js"],
  "delay": "500",
  "env": { "NODE_ENV": "development" }
}
```

> WARNING — Pitfall: **Never use `nodemon` in production.** It is a dev-only convenience: it adds file-watching overhead, restarts the whole process on any file change (terrible under real load), and is not a process manager. In production use a real process manager such as `pm2`, `systemd`, or your platform's container orchestrator, and start the app with plain `node` (your `start` script). Keep `nodemon` in `devDependencies` exactly so `npm install --production` excludes it.

> TIP: Node has a built-in watcher: `node --watch app.js` (stable since Node 18). For simple projects it replaces `nodemon` with zero dependencies. `nodemon` still wins for fine-grained config (custom extensions, ignore globs, delays, exec rules), but try `--watch` first — fewer dependencies is usually better.

---

## A Sensible Starter Project Structure

Express is unopinionated, so it will not tell you how to lay out files. Dumping everything into one `app.js` is fine for a 20-line demo but becomes unmanageable fast. Here is a pragmatic structure that scales smoothly into the REST-API and MongoDB modules later in this course.

```text
my-app/
├── src/
│   ├── app.js              # builds & configures the Express app (no listen here)
│   ├── server.js           # imports app, reads PORT, calls app.listen()
│   ├── config/
│   │   └── index.js        # centralized, env-aware configuration
│   ├── routes/
│   │   ├── index.js        # mounts all sub-routers onto the app
│   │   └── users.routes.js # the /users router
│   ├── controllers/
│   │   └── users.controller.js  # request handlers (the "what to do")
│   ├── middleware/
│   │   ├── logger.js
│   │   └── errorHandler.js
│   ├── models/             # (later: Mongoose schemas/models)
│   └── services/           # (later: business logic, DB access)
├── public/                 # static assets (css, images, client js)
├── tests/
├── .env                    # local secrets/config (NOT committed)
├── .env.example            # template of required env vars (committed)
├── .gitignore
├── nodemon.json
└── package.json
```

### Why separate `app.js` from `server.js`?

This is a small decision with a big payoff. `app.js` *builds and returns* the configured Express application but does **not** start listening. `server.js` is the only place that calls `app.listen()`. The reason: automated tests (with Supertest) can `require('./app')` and exercise routes **without binding a real network port**, while production code uses `server.js` to actually listen.

```js
// src/config/index.js — one place to read environment-aware config
require('dotenv').config(); // loads variables from .env into process.env

const config = {
  env: process.env.NODE_ENV || 'development',
  port: Number(process.env.PORT) || 3000,
  isProduction: process.env.NODE_ENV === 'production',
};

module.exports = config;
```

```js
// src/app.js — build and configure the app; DO NOT listen here
const express = require('express');
const config = require('./config');
const logger = require('./middleware/logger');
const errorHandler = require('./middleware/errorHandler');
const routes = require('./routes');

const app = express();

// Environment-aware settings
app.set('env', config.env);
app.set('json spaces', config.isProduction ? 0 : 2); // pretty JSON only in dev
app.locals.appName = 'My App';                        // app-wide data

// Global middleware
app.use(express.json());                  // parse JSON bodies
app.use(express.urlencoded({ extended: true })); // parse form bodies
app.use(express.static('public'));        // serve files from /public
app.use(logger);                          // custom request logger

// Routes
app.use('/api', routes);                  // mount all routers under /api

// 404 + centralized error handling come LAST
app.use((req, res) => res.status(404).json({ error: 'Not found' }));
app.use(errorHandler);

module.exports = app;                      // export for server.js AND for tests
```

```js
// src/server.js — the only file that starts the network server
const app = require('./app');
const config = require('./config');

const server = app.listen(config.port, () => {
  console.log(`[${config.env}] listening on http://localhost:${config.port}`);
});

// Graceful shutdown
for (const signal of ['SIGINT', 'SIGTERM']) {
  process.on(signal, () => server.close(() => process.exit(0)));
}
```

```js
// src/routes/index.js — aggregate sub-routers
const express = require('express');
const usersRouter = require('./users.routes');

const router = express.Router(); // a mini-app you can mount with app.use()
router.use('/users', usersRouter);

module.exports = router;
```

```js
// src/routes/users.routes.js — the /api/users router
const express = require('express');
const ctrl = require('../controllers/users.controller');

const router = express.Router();
router.get('/', ctrl.list);        // GET    /api/users
router.post('/', ctrl.create);     // POST   /api/users
router.get('/:id', ctrl.getById);  // GET    /api/users/:id

module.exports = router;
```

```js
// src/controllers/users.controller.js — the handlers
const users = [{ id: 1, name: 'Ada' }];

exports.list = (req, res) => res.json(users);
exports.getById = (req, res) => {
  const user = users.find((u) => u.id === Number(req.params.id));
  if (!user) return res.status(404).json({ error: 'User not found' });
  res.json(user);
};
exports.create = (req, res) => {
  const user = { id: users.length + 1, name: req.body.name };
  users.push(user);
  res.status(201).json(user);
};
```

```js
// src/middleware/logger.js
module.exports = (req, res, next) => {
  console.log(`${new Date().toISOString()} ${req.method} ${req.originalUrl}`);
  next(); // ALWAYS call next() (or send a response) in middleware
};
```

```js
// src/middleware/errorHandler.js — note the FOUR arguments
module.exports = (err, req, res, next) => {
  console.error(err.stack);
  res.status(err.status || 500).json({ error: err.message || 'Server error' });
};
```

> BEST PRACTICE: Keep route files *thin* — they only wire paths to controller functions. Put the actual logic in controllers/services. This separation (routing → controllers → services → models) is exactly the layering you will flesh out in the REST and MongoDB modules, and it is what interviewers mean by "structuring an Express app."

We do not deep-dive middleware, routers, or `express.static` here — those have their own modules. The point is to *see* where they live in a real layout.

---

## Express 4 vs Express 5: Key Differences to Know

Express 5 became the default (`npm install express` gives you v5) in 2025 after a long beta. Most v4 apps run on v5 with minor changes, but the differences are a favorite interview topic. Here are the ones that matter.

| Topic | Express 4 | Express 5 |
| --- | --- | --- |
| **Node.js requirement** | Older Node supported | **Requires Node 18+** |
| **Async errors** | A rejected Promise / thrown error in an `async` handler is **not** caught — it can crash the app unless you wrap it | A rejected Promise from a route handler is **automatically forwarded to your error-handling middleware** |
| **Path route syntax** | `path-to-regexp` v0.x; loose patterns | Updated `path-to-regexp`; stricter patterns. Optional params use `{}` not `?`; the bare `*` wildcard must be **named** |
| **`app.del()`** | Existed (alias for delete) | **Removed** — use `app.delete()` |
| **`res.json(status, obj)` / `res.send(status, body)`** | Deprecated signatures still worked | **Removed** — use `res.status(code).json(obj)` |
| **`res.sendfile()`** (lowercase) | Existed (deprecated) | **Removed** — use `res.sendFile()` |
| **`req.param(name)`** | Existed (deprecated) | **Removed** — use `req.params`, `req.body`, or `req.query` |
| **Body parsers** | Needed external `body-parser` historically | `express.json()` / `express.urlencoded()` built in (true since Express 4.16 too) |
| **`app.router`** | Deprecated property | Removed |
| **Pluralized rejections** | — | Several long-deprecated method signatures are now hard errors |

### The single most important change: async error handling

In Express 4, this handler is a landmine:

```js
// Express 4 — DANGEROUS: a rejected promise is NOT caught by Express.
app.get('/data', async (req, res) => {
  const data = await db.find();   // if this rejects...
  res.json(data);                  // ...this never runs, and the error escapes
});
// → In Express 4 you must wrap it yourself, e.g. with a try/catch + next(err),
//   or a wrapper like express-async-handler, or you risk an unhandled rejection.
```

The Express 4 fix was to wrap every async handler:

```js
// Express 4 pattern — manual forwarding to the error handler
app.get('/data', async (req, res, next) => {
  try {
    const data = await db.find();
    res.json(data);
  } catch (err) {
    next(err); // hand the error to error-handling middleware
  }
});
```

In **Express 5**, the framework awaits your handler and forwards rejections automatically:

```js
// Express 5 — a rejected promise is sent to your error handler for you.
app.get('/data', async (req, res) => {
  const data = await db.find(); // if this rejects, Express 5 calls next(err) for you
  res.json(data);
});
```

> WARNING — Pitfall: Express 5's auto-forwarding applies to errors returned by **rejected Promises and `async` handlers**. A *synchronous* `throw` inside a non-async handler has always been caught by Express (both v4 and v5). But errors thrown inside callbacks (e.g. `setTimeout`, event emitters) are *not* caught by either version — you must handle those yourself.

### Path-matching changes that bite during upgrades

```js
// Express 4 wildcard:
app.get('/files/*', handler);        // matched anything under /files/

// Express 5 — the wildcard must be NAMED:
app.get('/files/*splat', handler);   // capture the rest into req.params.splat

// Express 4 optional segment:
app.get('/users/:id?', handler);     // id optional

// Express 5 — optional uses braces:
app.get('/users{/:id}', handler);    // id optional via {}
```

> TIP: The official Express **migration guide (4 → 5)** lists every removed method and signature. When upgrading, the highest-value win is *removing your async-wrapper boilerplate* (or `express-async-handler`) — Express 5 makes most of it unnecessary.

---

## App Settings, Environment-Aware Config, and `app.locals`

### `app.set()` and `app.get()` for settings

Express keeps a per-application settings table. You write to it with `app.set(name, value)` and read from it with `app.get(name)`:

```js
// settings.js
const express = require('express');
const app = express();

app.set('view engine', 'pug');     // tell Express which template engine to use
app.set('views', './views');       // where templates live
app.set('json spaces', 2);         // pretty-print JSON responses
app.set('case sensitive routing', true); // /Foo and /foo become different routes
app.set('strict routing', true);   // /foo and /foo/ become different routes
app.set('trust proxy', 1);         // trust the first proxy (for req.ip behind a load balancer)

console.log(app.get('view engine')); // → 'pug'   (one string arg = READ a setting)
```

Boolean settings have shortcut methods:

```js
app.disable('x-powered-by');   // same as app.set('x-powered-by', false)
app.enable('trust proxy');     // same as app.set('trust proxy', true)

console.log(app.enabled('trust proxy'));  // → true
console.log(app.disabled('x-powered-by')); // → true
```

> WARNING — Pitfall: Reusing `app.get`. `app.get('view engine')` reads a setting because it has a single string argument. `app.get('/users', handler)` defines a route because it has a path plus a handler. They are the *same method name* doing two unrelated jobs. If you accidentally write `app.get('some-setting', () => {})`, you have silently created a route for the path `/some-setting`, not read a setting. Be deliberate.

Some commonly used built-in settings:

| Setting | Effect |
| --- | --- |
| `env` | Application environment; defaults to `process.env.NODE_ENV` or `'development'` |
| `view engine` | Default template engine used by `res.render()` |
| `views` | Directory (or array) where view templates are located |
| `trust proxy` | How to interpret `X-Forwarded-*` headers behind proxies |
| `json spaces` | Indentation for `res.json()` output |
| `case sensitive routing` | Treat `/Foo` and `/foo` as different (default off) |
| `strict routing` | Treat `/foo` and `/foo/` as different (default off) |
| `x-powered-by` | Whether to send the `X-Powered-By: Express` header (default on) |

> BEST PRACTICE: Set `app.disable('x-powered-by')` (or use the `helmet` middleware) to stop advertising your framework to attackers. It is a one-line security hygiene improvement.

### Environment-aware configuration

A real application behaves differently across environments — verbose logging and pretty JSON in development, strict and fast in production. Two ingredients make this clean: the `NODE_ENV` environment variable and a single config module.

`NODE_ENV` is a convention (read from `process.env.NODE_ENV`). Express itself reads it to set the `env` setting and to tune defaults (for example, in production it caches view templates and uses terser error output).

```bash
# set NODE_ENV when starting the process
NODE_ENV=production node src/server.js
```

Branch on it explicitly where behavior should differ:

```js
// env-aware.js
const express = require('express');
const app = express();

if (app.get('env') === 'development') {
  app.set('json spaces', 2);                 // readable JSON locally
  app.use((req, res, next) => {              // dev-only verbose logging
    console.log(`DEV ${req.method} ${req.url}`);
    next();
  });
} else {
  app.set('json spaces', 0);                 // compact JSON in production
  app.disable('x-powered-by');               // tighten up
}
```

> WARNING — Pitfall: `NODE_ENV` defaults to nothing if you never set it. Many libraries (including Express) treat *anything other than* `'production'` as non-production. So forgetting to set `NODE_ENV=production` in your deployment silently leaves you in development mode — slower, more verbose, and leaking stack traces. Always set it explicitly in production.

The cleanest pattern is to centralize all config behind one module and load secrets from a `.env` file using the `dotenv` package (covered fully in a later config module):

```js
// config/index.js — single source of truth for configuration
require('dotenv').config(); // populate process.env from a .env file

module.exports = {
  env: process.env.NODE_ENV || 'development',
  port: Number(process.env.PORT) || 3000,
  dbUrl: process.env.DATABASE_URL,      // never hard-code secrets
  jsonSpaces: process.env.NODE_ENV === 'production' ? 0 : 2,
};
```

```bash
# .env  (NEVER commit this file — add it to .gitignore)
NODE_ENV=development
PORT=4000
DATABASE_URL=mongodb://localhost:27017/myapp
```

> BEST PRACTICE: Keep secrets and environment-specific values in environment variables, not in source code. Commit a `.env.example` (with empty/placeholder values) so teammates know which variables are required, and add `.env` to `.gitignore`.

### `app.locals` — application-wide data

`app.locals` is a plain object whose properties are available throughout the *lifetime of the application*: in every request handler (via `req.app.locals`) and, importantly, automatically merged into the data available to rendered views. Use it for values that are constant for the whole app: the site name, the current year, helper functions, feature flags.

```js
// app-locals.js
const express = require('express');
const app = express();

// Set once at startup; visible everywhere.
app.locals.siteName = 'Acme Corp';
app.locals.supportEmail = 'help@acme.test';
app.locals.formatPrice = (cents) => `$${(cents / 100).toFixed(2)}`; // helper fn

app.get('/', (req, res) => {
  // Accessible from any handler via req.app.locals
  res.send(`Welcome to ${req.app.locals.siteName}`);
});

app.get('/price', (req, res) => {
  res.send(app.locals.formatPrice(1299)); // → $12.99
});

app.listen(3000);
```

> TIP: Do not confuse `app.locals` with `res.locals`. **`app.locals`** is app-wide and lives for the whole process. **`res.locals`** is *per-request* — it is reset for every incoming request and is the right place for request-scoped data (like the currently logged-in user) that views or downstream middleware need. Rule of thumb: constant for the whole app → `app.locals`; specific to one request → `res.locals`.

---

## Common Pitfalls & Gotchas

- **Sending more than one response.** Forgetting to `return` after `res.send`/`res.json` leads to `ERR_HTTP_HEADERS_SENT`. Each request gets exactly one response; `return` after you send inside a branch.
- **`app.get` is overloaded.** One string argument reads a *setting*; a path plus handler defines a *route*. Mixing them up silently creates phantom routes.
- **Running `nodemon` in production.** It is a dev-only watcher, not a process manager. Keep it in `devDependencies` and start production with plain `node` under `pm2`/systemd/containers.
- **Forgetting body-parsing middleware.** `req.body` is `undefined` unless you mount `express.json()` and/or `express.urlencoded()` *before* the routes that read it. Order matters: middleware registered after a route never runs for it.
- **Assuming async errors are caught in Express 4.** A rejected Promise in an Express 4 handler escapes Express entirely. Wrap with try/catch + `next(err)` (or upgrade to Express 5, which auto-forwards).
- **`req.params` values are strings.** `/users/:id` gives you `req.params.id === '42'` (a string). Convert with `Number(...)` before numeric comparisons or DB lookups.
- **Not setting `NODE_ENV=production`.** Express and many libraries treat anything but `'production'` as non-production, leaving you slower and leaking stack traces.
- **`EADDRINUSE` on restart.** A leftover process is holding the port. Find it (`lsof -i :3000`) and kill it, or let `nodemon` manage restarts.
- **Confusing `app.locals` and `res.locals`.** App-wide vs per-request. Putting request-specific data in `app.locals` causes cross-request data bleed.
- **Express 5 breaking signatures.** `app.del`, `res.json(status, obj)`, `res.sendfile`, and `req.param()` are gone in v5. Use `app.delete`, `res.status(code).json(obj)`, `res.sendFile`, and `req.params`/`req.query`/`req.body`.
- **Committing `node_modules` or `.env`.** Commit `package-lock.json` and `.env.example`; gitignore `node_modules` and `.env`.

---

## Best Practices

- **Separate `app.js` (build & configure) from `server.js` (listen).** It makes the app testable without binding a real port and keeps startup concerns isolated.
- **Prefer `res.json()` for APIs.** It documents intent and handles non-object JSON values correctly.
- **Centralize configuration** in one `config` module driven by environment variables; never hard-code secrets, ports, or DB URLs.
- **Pin and lock your dependencies.** Commit `package-lock.json`; be explicit about the Express major version (`express@5`) so upgrades are intentional.
- **Use `nodemon` (or `node --watch`) only in development;** keep it in `devDependencies`.
- **Keep route files thin;** put logic in controllers/services. Wire paths → controllers → services → models.
- **Always end every request** with exactly one response or a `next()`; never leave a path that does neither.
- **Disable `x-powered-by`** (or use `helmet`) to avoid advertising your stack.
- **Set `NODE_ENV=production` explicitly in deployment** to get Express's production optimizations.
- **Handle graceful shutdown** by capturing the server from `app.listen()` and calling `server.close()` on `SIGINT`/`SIGTERM`.
- **Mount middleware in the right order:** body parsers and global middleware first, routes next, then a 404 handler, and finally the error-handling middleware last.

---

## Key Interview Points

- **What is Express and why use it over the raw `http` module?** Express is a minimal, unopinionated web framework that wraps Node's `http` module and adds routing, middleware, request/response helpers, and view-engine support — eliminating the manual `if/else` routing, header/serialization boilerplate, and body buffering that raw `http` requires.
- **Is Express a replacement for the `http` module?** No. It sits *on top* of `http`. `app.listen()` internally calls `http.createServer(app).listen()`, and an Express `app` is itself a valid `(req, res)` request-handler function you can pass to `http.createServer`.
- **What is the Express `app`, technically?** A function (with extra methods attached) that has the signature of an `http` request handler. That is why it is both callable and passable to `http.createServer`.
- **What does `app.listen()` return and why does that matter?** The underlying `http.Server` instance — useful for graceful shutdown (`server.close()`) and for attaching things like WebSocket servers to the same port.
- **Difference between `res.send()` and `res.json()`?** `res.send()` is content-aware (strings → HTML, objects → JSON, Buffers → octet-stream); `res.json()` always serializes to JSON and sets `application/json`, and correctly handles non-object JSON values. Prefer `res.json()` for APIs.
- **What causes `ERR_HTTP_HEADERS_SENT`?** Sending more than one response — typically forgetting `return` after `res.send`/`res.json`, so a second response is attempted.
- **How are async errors handled in Express 4 vs 5?** In v4, a rejected Promise in a handler is *not* caught — you must `try/catch` and call `next(err)`. In v5, Express awaits handlers and forwards rejections to error-handling middleware automatically.
- **Name key Express 4 → 5 breaking changes.** Requires Node 18+; auto async-error forwarding; removed `app.del`, `res.json(status, body)`, `res.sendfile`, `req.param()`; stricter path matching (named wildcards `*splat`, optional segments via `{}`).
- **How do `app.set`/`app.get` work, and what's the gotcha?** They read/write per-app settings; `app.get('name')` with one string reads a setting, while `app.get('/path', handler)` defines a route — the same method name, two jobs.
- **`app.locals` vs `res.locals`?** `app.locals` is app-wide and lives for the whole process; `res.locals` is per-request and reset each time. App constants go in `app.locals`; request-scoped data (like the current user) goes in `res.locals`.
- **Why is `nodemon` a dev-only tool?** It is a file-watcher that restarts the process on change — fine for development, wrong for production where you need a real process manager (pm2/systemd/containers). Keep it in `devDependencies`.
- **Why set `NODE_ENV=production`?** Express and many libraries optimize (cache views, terse errors) only when `NODE_ENV` is exactly `'production'`; anything else is treated as non-production.
- **Why split `app.js` from `server.js`?** Testability — tests import the configured app and exercise routes without binding a real port, while `server.js` owns `app.listen()`.

---

## Practice Exercises

Work through these in order; each builds on the last. Solutions follow.

1. **Hello, Express.** Initialize a project, install Express, and create a server on port 3000 with a single `GET /` route that responds with the text `Hello, Express!`. Add `start` and `dev` npm scripts (the `dev` script must use `nodemon`).
2. **A tiny JSON API.** Add `GET /api/time` that returns `{ "now": <ISO timestamp> }` as JSON, and `GET /api/greet/:name` that returns `{ "message": "Hello, <name>!" }` using the route parameter. Make sure numeric/string params are handled correctly.
3. **Echo with status codes.** Add a `POST /api/echo` route that parses a JSON body and returns it under `{ "youSent": ... }` with status `201`. If the body is empty (`{}` or missing `message`), return `400` with `{ "error": "message is required" }`. Wire up `express.json()`.
4. **Settings and `app.locals`.** Configure the app so JSON is pretty-printed (2 spaces) only in development, disable the `x-powered-by` header, store a `siteName` in `app.locals`, and expose `GET /api/info` returning `{ "site": <siteName>, "env": <current env> }`.
5. **Refactor to a real structure.** Split the project into `src/app.js` (build + configure, no listen), `src/server.js` (listen), and `src/routes/api.routes.js` (a `Router` mounted at `/api`). Move the routes from exercises 2–4 into the router. Add a catch-all 404 JSON handler.
6. **Express 5 async error forwarding (stretch).** Add `GET /api/maybe` that randomly either responds with `{ ok: true }` or rejects an `async` operation (simulate with a Promise that rejects). Add an error-handling middleware that returns `500` with `{ "error": <message> }`. Verify that on Express 5 you do **not** need a try/catch in the handler.

### Solutions

#### Solution 1 — Hello, Express

```bash
mkdir express-exercises && cd express-exercises
npm init -y
npm install express
npm install --save-dev nodemon
```

```js
// app.js
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  res.send('Hello, Express!');
});

app.listen(3000, () => console.log('http://localhost:3000'));
```

```json
// package.json (scripts section)
{
  "scripts": {
    "start": "node app.js",
    "dev": "nodemon app.js"
  }
}
```

```bash
npm run dev   # auto-reloads on save
# verify:
curl http://localhost:3000   # → Hello, Express!
```

#### Solution 2 — A tiny JSON API

```js
// app.js (additions)
// GET /api/time → current timestamp as JSON
app.get('/api/time', (req, res) => {
  res.json({ now: new Date().toISOString() });
});

// GET /api/greet/:name → uses the route parameter (always a string)
app.get('/api/greet/:name', (req, res) => {
  const { name } = req.params;                 // e.g. /api/greet/Ada → 'Ada'
  res.json({ message: `Hello, ${name}!` });
});
```

```bash
curl http://localhost:3000/api/time
# → {"now":"2026-06-19T...Z"}
curl http://localhost:3000/api/greet/Ada
# → {"message":"Hello, Ada!"}
```

#### Solution 3 — Echo with status codes

```js
// app.js
const express = require('express');
const app = express();

app.use(express.json()); // REQUIRED so req.body is populated for JSON requests

app.post('/api/echo', (req, res) => {
  // Validate: req.body may be {} if the client sent no/empty body.
  if (!req.body || !req.body.message) {
    return res.status(400).json({ error: 'message is required' }); // note the return
  }
  res.status(201).json({ youSent: req.body });
});

app.listen(3000);
```

```bash
curl -X POST http://localhost:3000/api/echo \
  -H 'Content-Type: application/json' \
  -d '{"message":"hi"}'
# → {"youSent":{"message":"hi"}}   (HTTP 201)

curl -i -X POST http://localhost:3000/api/echo \
  -H 'Content-Type: application/json' -d '{}'
# → HTTP/1.1 400 Bad Request ... {"error":"message is required"}
```

#### Solution 4 — Settings and `app.locals`

```js
// app.js
const express = require('express');
const app = express();

// Pretty JSON only in development.
if (app.get('env') === 'development') {
  app.set('json spaces', 2);
}

app.disable('x-powered-by');         // stop advertising the framework
app.locals.siteName = 'Acme Corp';   // app-wide constant

app.get('/api/info', (req, res) => {
  res.json({
    site: req.app.locals.siteName,   // read app.locals from the handler
    env: app.get('env'),             // current environment
  });
});

app.listen(3000);
```

```bash
NODE_ENV=development node app.js &
curl -i http://localhost:3000/api/info
# Headers will NOT include X-Powered-By.
# Body (pretty-printed because env is development):
# {
#   "site": "Acme Corp",
#   "env": "development"
# }
```

#### Solution 5 — Refactor to a real structure

```js
// src/routes/api.routes.js
const express = require('express');
const router = express.Router();

router.get('/time', (req, res) => {
  res.json({ now: new Date().toISOString() });
});

router.get('/greet/:name', (req, res) => {
  res.json({ message: `Hello, ${req.params.name}!` });
});

router.post('/echo', (req, res) => {
  if (!req.body || !req.body.message) {
    return res.status(400).json({ error: 'message is required' });
  }
  res.status(201).json({ youSent: req.body });
});

router.get('/info', (req, res) => {
  res.json({ site: req.app.locals.siteName, env: req.app.get('env') });
});

module.exports = router;
```

```js
// src/app.js — build & configure, NO listen
const express = require('express');
const apiRoutes = require('./routes/api.routes');

const app = express();

if (app.get('env') === 'development') app.set('json spaces', 2);
app.disable('x-powered-by');
app.locals.siteName = 'Acme Corp';

app.use(express.json());
app.use('/api', apiRoutes);                 // mount the router under /api

// Catch-all 404 (after all routes)
app.use((req, res) => res.status(404).json({ error: 'Not found' }));

module.exports = app;                        // export for server.js and tests
```

```js
// src/server.js — the only file that listens
const app = require('./app');
const PORT = Number(process.env.PORT) || 3000;

const server = app.listen(PORT, () => {
  console.log(`listening on http://localhost:${PORT}`);
});

for (const sig of ['SIGINT', 'SIGTERM']) {
  process.on(sig, () => server.close(() => process.exit(0)));
}
```

```json
// package.json scripts
{
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js"
  }
}
```

#### Solution 6 — Express 5 async error forwarding

```js
// src/routes/api.routes.js (add to the router)

// A simulated async operation that sometimes rejects.
function maybeFail() {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      Math.random() < 0.5
        ? resolve({ ok: true })
        : reject(new Error('random failure'));
    }, 50);
  });
}

// On Express 5, NO try/catch is needed: a rejected promise is forwarded
// to the error-handling middleware automatically.
router.get('/maybe', async (req, res) => {
  const result = await maybeFail(); // if this rejects, Express 5 calls next(err)
  res.json(result);
});
```

```js
// src/app.js — add the error handler LAST (note the four parameters)
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: err.message });
});
```

```bash
# Hit it a few times; about half should be 500 with {"error":"random failure"},
# the rest 200 with {"ok":true} — without any try/catch in the handler.
for i in 1 2 3 4 5 6; do curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3000/api/maybe; done
```

> TIP: To prove the difference, run the same handler under Express 4: the rejection would surface as an `UnhandledPromiseRejection` instead of a clean 500, because v4 does not await your handler.

---

## Summary / Cheat Sheet

| Concept | Key takeaway |
| --- | --- |
| **What Express is** | Minimal, unopinionated framework over Node's `http`: routing + middleware + req/res helpers |
| **Install** | `npm install express` (v5 is default; `express@4` to pin v4). Requires Node 18+ for v5 |
| **Create app** | `const app = express()` — a callable function object with methods attached |
| **Start server** | `app.listen(port, cb)` → wraps `http.createServer(app).listen()`, returns the `http.Server` |
| **`app` is a handler** | An Express `app` is a valid `(req, res)` function; `http.createServer(app)` works |
| **Define routes** | `app.get/post/put/patch/delete(path, handler)`; `app.all`, `app.route(path)` |
| **Request inputs** | `req.params` (route), `req.query` (query string), `req.body` (needs `express.json()`) |
| **Send responses** | `res.send()` (content-aware), `res.json()` (always JSON), `res.status(code)` (chainable) |
| **One response rule** | Exactly one response per request; `return` after sending to avoid `ERR_HTTP_HEADERS_SENT` |
| **nodemon** | Dev-only auto-restart: `npm i -D nodemon`, `nodemon app.js`; or built-in `node --watch` |
| **Project layout** | Split `app.js` (build) from `server.js` (listen); routers → controllers → services |
| **Express 4 vs 5** | v5: Node 18+, auto async-error forwarding, removed `app.del`/`res.json(status,body)`/`req.param`, stricter paths |
| **Settings** | `app.set(name, value)` / `app.get(name)`; `app.enable/disable` for booleans |
| **Env config** | Drive behavior off `NODE_ENV`; centralize in a config module; secrets in env vars, not code |
| **`app.locals`** | App-wide data for the whole process (vs `res.locals`, which is per-request) |
| **Security one-liners** | `app.disable('x-powered-by')`; set `NODE_ENV=production` in deployment |

You now have a precise mental model of what Express is, why it exists, and how it sits on top of Node's `http` module — plus the application object, routing, responses, the dev loop, project structure, the Express 4→5 differences, and environment-aware configuration. The next modules build directly on this: routing in depth, the middleware pipeline, building full REST APIs, and wiring in MongoDB with Mongoose.
