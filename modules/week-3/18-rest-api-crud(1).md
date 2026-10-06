# Building REST APIs & CRUD (Design Principles)

By now you can stand up an Express server and route requests. This module is where you stop writing *ad hoc* endpoints and start designing **APIs** — coherent, predictable interfaces that other developers (and future-you) can understand without reading your source code. The single most valuable skill a backend developer brings to a team is not "I can make an endpoint return JSON" — it is "I can design an HTTP interface that is consistent, correct, and a pleasure to consume." That skill is almost entirely about *design principles*, and the dominant design style for HTTP APIs is **REST**.

This module teaches you REST from first principles, drills the exact semantics of every HTTP method and status code (interviewers love these), and then walks you through building a complete CRUD API in Express — starting with an in-memory store so nothing distracts from the design. We finish with the topics that separate a junior endpoint-writer from someone who designs production APIs: consistent response shapes, structured errors, pagination/filtering/sorting, versioning, idempotency, and HATEOAS.

## What You'll Learn

By the end of this module you will be able to:

- Explain **REST** from first principles: resources, statelessness, the uniform interface, and representations — and articulate *why* each constraint exists.
- Apply the precise **semantics of HTTP methods** (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`) and reason about which are **safe** and which are **idempotent**.
- Choose the **correct HTTP status code** for any situation using a complete 2xx/3xx/4xx/5xx reference.
- Design clean, predictable **resource URLs** using plural nouns, sensible nesting, and no verbs.
- Build a **full CRUD API** step by step in Express against an in-memory store, with create, read (list + single), update (full + partial), and delete.
- Implement **request validation** and return a **consistent JSON response envelope** for both success and failure.
- Design **structured error responses** that clients can parse programmatically.
- Add **pagination, filtering, and sorting** to a collection endpoint.
- Apply **API versioning**, reason correctly about **idempotency and safety**, and explain **HATEOAS** and why most APIs skip it.

---

## 1. What REST Actually Is (and Isn't)

**REST** stands for **RE**presentational **S**tate **T**ransfer. It is an *architectural style* described by Roy Fielding in his 2000 doctoral dissertation — not a protocol, not a standard, not a library. It is a set of **constraints** that, when applied to a networked system, produce desirable properties: scalability, simplicity, visibility, portability, and reliability.

This distinction matters because "REST" is one of the most abused words in our industry. Most APIs called "RESTful" are really "HTTP APIs that use JSON and nice URLs" — and that is perfectly fine. Knowing the *actual* constraints lets you decide deliberately which to follow and which to skip, instead of cargo-culting.

> TIP: In an interview, if asked "what is REST?", do not say "it's a way to make API calls." Say: "REST is an architectural style defined by a set of constraints — most importantly statelessness, a uniform interface, client-server separation, and addressable resources with representations." That single sentence signals you understand the *why*, not just the *how*.

### The core REST constraints

REST defines six constraints. Five are required for an API to be "RESTful"; the sixth is optional.

| Constraint | What it means | Why it matters |
| --- | --- | --- |
| **Client–Server** | UI/consumer concerns are separated from data/storage concerns. | Each side evolves independently; the same API serves web, mobile, CLI. |
| **Statelessness** | Every request contains everything needed to process it; the server stores no client session state between requests. | Any server instance can handle any request → horizontal scaling, resilience. |
| **Cacheability** | Responses declare whether they can be cached. | Reduces load and latency; clients/proxies reuse responses. |
| **Uniform Interface** | One consistent way to interact: addressable resources, manipulation via representations, self-descriptive messages, (optionally) hypermedia. | Decouples client from server internals; the interface is predictable. |
| **Layered System** | Client cannot tell whether it talks to the origin server or an intermediary (load balancer, CDN, gateway). | Enables proxies, caches, gateways, and security layers transparently. |
| **Code on Demand** (optional) | Server can ship executable code (e.g., JS) to the client. | Rarely used in practice; the only optional constraint. |

The two constraints you must be able to discuss fluently are **statelessness** and the **uniform interface**, because they drive almost every concrete decision you'll make.

### Resources: the central abstraction

In REST, everything is a **resource** — any concept worth naming and addressing: a user, an order, a blog post, a bank account, a search result, even "the collection of all users." A resource is *not* the row in your database; it is a conceptual entity. Each resource is identified by a **URI** (e.g., `/users/42`).

The data your server actually sends over the wire is a **representation** of that resource — typically a JSON document, but it could be XML, HTML, CSV, or a binary blob. The resource is the abstract thing; the representation is one concrete snapshot of its state in some format. This is the "**Representational**" in REST: the client and server exchange *representations of resource state*, never the resource itself.

```json
// One JSON representation of the resource identified by /users/42
{
  "id": 42,
  "name": "Ada Lovelace",
  "email": "ada@example.com",
  "createdAt": "2026-01-15T09:30:00.000Z"
}
```

The same resource (`/users/42`) could be represented differently depending on what the client asks for via the `Accept` header — JSON for an app, HTML for a browser. This idea is called **content negotiation**.

### Statelessness, concretely

Statelessness is the constraint juniors most often violate without realizing it. It means: **the server does not remember anything about previous requests from a client.** Each request must carry all the context needed to fulfill it — authentication credentials, the resource being addressed, any parameters.

```bash
# WRONG mental model: "First I log in, then the server remembers I'm logged in,
# then my next request is implicitly authenticated."
# That requires server-side session memory tied to this specific client.

# RIGHT (stateless): every request carries its own proof of identity.
curl https://api.example.com/orders \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."   # token sent EVERY time
```

> WARNING — Pitfall: Server-side sessions (storing logged-in state in server memory) technically break statelessness. In practice many apps use them anyway, but they harm horizontal scaling: request #2 might hit a different server instance that never saw your login. Token-based auth (JWT, opaque bearer tokens validated against a shared store) keeps requests self-contained and is the REST-aligned approach. We cover auth fully in a later module — for now, just internalize *why* stateless requests scale.

### Why these constraints pay off

- **Scalability:** Because the server holds no per-client state, you can put ten identical instances behind a load balancer and route any request to any of them.
- **Visibility & debugging:** A single request is self-contained, so a proxy, log, or you-with-curl can understand it in isolation.
- **Reliability:** If a server crashes mid-traffic, no client "session" is lost — the next request just goes elsewhere.
- **Evolvability:** The uniform interface decouples clients from server internals, so you can refactor the database without breaking consumers.

---

## 2. HTTP Method Semantics

REST piggybacks on HTTP's existing **methods** (also called verbs) to express *what kind of operation* a client wants to perform on a resource. The method is half the meaning of every request; the URL is the other half. "Where" is the URL; "what to do there" is the method.

The crucial insight for newcomers: **the method carries semantics that the entire web infrastructure relies on.** Browsers, proxies, caches, and CDNs all make assumptions based on the method. If you use `GET` to delete something, you are not just being unconventional — you are lying to the infrastructure, and something *will* break (a search-engine crawler following links will happily delete your data).

### The five methods you'll use constantly

| Method | CRUD op | Typical use | Request body? | Success status |
| --- | --- | --- | --- | --- |
| `GET` | Read | Fetch a resource or collection | No | `200 OK` |
| `POST` | Create | Create a new resource in a collection | Yes | `201 Created` |
| `PUT` | Update (full) | Replace a resource entirely | Yes | `200 OK` / `204 No Content` |
| `PATCH` | Update (partial) | Modify part of a resource | Yes | `200 OK` / `204 No Content` |
| `DELETE` | Delete | Remove a resource | Usually no | `204 No Content` / `200 OK` |

### Safe and idempotent methods

These two properties come up in *every* API design interview. Memorize the definitions precisely — the wording matters.

- **Safe**: the method is **read-only**; calling it does not change server state. Safe methods are for retrieval only.
- **Idempotent**: making the **same request N times has the same effect on server state as making it once**. (The *responses* may differ — e.g., a second DELETE may return 404 — but the resulting *state* is identical.)

| Method | Safe? | Idempotent? | Why |
| --- | --- | --- | --- |
| `GET` | Yes | Yes | Reads only; reading twice changes nothing. |
| `HEAD` | Yes | Yes | Like GET but headers only; read-only. |
| `OPTIONS` | Yes | Yes | Describes allowed methods; read-only. |
| `PUT` | No | **Yes** | Replacing a resource with the same body twice = same final state. |
| `DELETE` | No | **Yes** | Deleting an already-deleted resource leaves it... deleted. State is unchanged. |
| `PATCH` | No | **Not guaranteed** | Depends on the patch. `{"$inc": {"views": 1}}` is *not* idempotent; `{"status":"done"}` is. |
| `POST` | No | **No** | Each POST typically creates a *new* resource → calling twice creates two. |

> WARNING — Pitfall: The classic gotcha is asserting "PATCH is idempotent." It depends entirely on the operation. A PATCH that sets `email = "x@y.com"` is idempotent; a PATCH that says "add 1 to the counter" is not. Always answer: "PATCH is *not guaranteed* to be idempotent — it depends on the semantics of the patch."

> WARNING — Pitfall: Another favorite: "Is DELETE idempotent even though the second call returns 404?" Yes. Idempotency is about the *effect on server state*, not the response code. After one DELETE the resource is gone; after a hundred it is still just gone.

Why does idempotency matter operationally? **Retries.** Networks are unreliable. If a client sends a request and the connection drops before the response arrives, the client doesn't know whether the server processed it. For idempotent methods (GET, PUT, DELETE), the client can safely retry. For non-idempotent POST, a blind retry risks creating a duplicate — which is exactly why **idempotency keys** exist (more in §11).

### PUT vs PATCH — the most-tested distinction

Both update a resource, but they mean different things:

- **`PUT` replaces the entire resource** with the representation in the request body. Fields you omit are treated as *removed/reset*, because you're sending the complete new state.
- **`PATCH` applies a partial modification.** You send only the fields you want to change; everything else stays as-is.

```bash
# Suppose /users/42 is currently:
# { "id": 42, "name": "Ada", "email": "ada@example.com", "role": "admin" }

# PUT replaces the WHOLE resource. Omitting "role" means role is gone/reset.
curl -X PUT https://api.example.com/users/42 \
  -H "Content-Type: application/json" \
  -d '{ "name": "Ada Lovelace", "email": "ada@new.com" }'
# Result: { "id": 42, "name": "Ada Lovelace", "email": "ada@new.com" }  (role dropped!)

# PATCH changes ONLY what you send. Everything else is preserved.
curl -X PATCH https://api.example.com/users/42 \
  -H "Content-Type: application/json" \
  -d '{ "email": "ada@new.com" }'
# Result: { "id":42, "name":"Ada", "email":"ada@new.com", "role":"admin" }
```

> BEST PRACTICE: For most "edit a record" features in real apps, you want **PATCH** semantics (change a few fields), even though many teams informally implement `PUT` as "update with whatever you sent." Be explicit and consistent: if your `PUT` doesn't truly replace, document it — or better, use `PATCH` and reserve `PUT` for genuine full replacement (and for *creating a resource at a known URI*, which PUT also legitimately allows).

### POST is the "wildcard"

`POST` is the catch-all method whose semantics are "process this request body according to the resource's own rules." Its primary REST use is **creating a new resource in a collection** (`POST /users` creates a user and the server assigns the ID). But POST is also the correct choice for operations that don't map to CRUD at all — e.g., `POST /users/42/password-reset` or `POST /payments/123/refunds`. When an action isn't a clean create/read/update/delete, model it as a sub-resource and `POST` to it rather than inventing a verb-in-the-URL.

---

## 3. HTTP Status Codes: A Complete Reference

The status code is the server's one-line summary of what happened, in a number the entire web understands. Getting these right is a hallmark of a professional API. The codes are grouped by their first digit.

### The five classes

| Class | Meaning | Who "caused" it |
| --- | --- | --- |
| **1xx** Informational | Request received, continuing. Rarely hand-written. | — |
| **2xx** Success | The request succeeded. | — |
| **3xx** Redirection | Further action needed (usually go elsewhere). | — |
| **4xx** Client Error | The *client* did something wrong (bad input, no auth). | Client |
| **5xx** Server Error | The *server* failed despite a valid request. | Server |

The single most useful mental model: **4xx = "you messed up", 5xx = "I messed up".** A bug in your code that throws an unhandled exception should be a 5xx. A user sending an invalid email should be a 4xx. Confusing these two is the most common status-code mistake.

### 2xx — Success

| Code | Name | When to use |
| --- | --- | --- |
| **200** | OK | Generic success. Use for successful `GET`, and for `PUT`/`PATCH`/`DELETE` when you return the updated/affected resource in the body. |
| **201** | Created | A new resource was created (almost always after `POST`). Include a `Location` header pointing to the new resource and, conventionally, the created resource in the body. |
| **202** | Accepted | The request was accepted for **async** processing but isn't done yet (e.g., queued a job). Response should explain how to check status. |
| **204** | No Content | Success, but there is **nothing to return**. Ideal for `DELETE`, and for `PUT`/`PATCH` when you don't echo the resource. The body MUST be empty. |
| **206** | Partial Content | A partial/range response (e.g., serving a byte range of a video). |

### 3xx — Redirection

| Code | Name | When to use |
| --- | --- | --- |
| **301** | Moved Permanently | The resource has a new permanent URL. Caches/clients should update links. |
| **302** | Found | Temporary redirect (historically ambiguous about method). |
| **303** | See Other | "Go GET this other URL" — classic POST-then-redirect-to-result. |
| **304** | Not Modified | Used with caching/conditional requests (`If-None-Match`/`ETag`). Tells the client its cached copy is still valid; no body sent. |
| **307** | Temporary Redirect | Like 302 but **guarantees the method is preserved** (a redirected POST stays a POST). |
| **308** | Permanent Redirect | Like 301 but **preserves the method**. |

### 4xx — Client Errors (the ones you'll send most)

| Code | Name | When to use |
| --- | --- | --- |
| **400** | Bad Request | Malformed request the server can't/won't process: invalid JSON, missing required field, bad parameter. The generic validation-failure code. |
| **401** | Unauthorized | **Not authenticated.** No or invalid credentials. (Misnamed — it really means "unauthenticated".) Send a `WWW-Authenticate` header. |
| **403** | Forbidden | **Authenticated but not allowed.** The server knows who you are; you just lack permission for this resource. |
| **404** | Not Found | The resource doesn't exist (or you want to hide its existence). Also used to mask 403s for privacy. |
| **405** | Method Not Allowed | The URL exists but not for this method (e.g., `DELETE /users` on a read-only collection). Include an `Allow` header listing valid methods. |
| **406** | Not Acceptable | Server can't produce a representation matching the client's `Accept` header. |
| **409** | Conflict | The request conflicts with current state: duplicate unique field (email already taken), edit conflict, version mismatch. |
| **410** | Gone | The resource existed but is permanently removed (stronger, intentional 404). |
| **413** | Payload Too Large | The request body exceeds limits. |
| **415** | Unsupported Media Type | The body's `Content-Type` isn't supported (e.g., sent XML to a JSON-only endpoint). |
| **422** | Unprocessable Content | Syntactically valid (parseable JSON) but **semantically** invalid — failed business/validation rules. Many APIs prefer 422 over 400 for validation errors. |
| **429** | Too Many Requests | Rate limit exceeded. Include a `Retry-After` header. |

> TIP: The 400-vs-422 question is a popular interview probe. A defensible answer: "I use **400** when the request itself is malformed (unparseable JSON, missing/garbled parameters) and **422** when the JSON parses fine but fails validation rules (e.g., `age: -5`). Both are valid; consistency within an API matters more than the exact choice." The 401-vs-403 distinction is non-negotiable, though: 401 = who are you?, 403 = I know who you are, and no.

### 5xx — Server Errors

| Code | Name | When to use |
| --- | --- | --- |
| **500** | Internal Server Error | Generic server failure / unhandled exception. The "something broke in our code" code. **Never leak stack traces to clients here.** |
| **501** | Not Implemented | The server doesn't support the functionality to fulfill the request. |
| **502** | Bad Gateway | A gateway/proxy got an invalid response from an upstream server. |
| **503** | Service Unavailable | Server temporarily down/overloaded/in maintenance. Include `Retry-After`. |
| **504** | Gateway Timeout | A gateway/proxy timed out waiting for an upstream server. |

> WARNING — Pitfall: Returning `200 OK` with `{ "error": "..." }` in the body is a cardinal sin. Tooling, monitoring, retries, and caches all key off the status code. If something failed, the status code must reflect it. "Always 200" APIs are notoriously painful to consume and to monitor.

---

## 4. Resource Naming Conventions

URLs are the most visible part of your API — they're in documentation, logs, dashboards, and every client's source code. Consistent, predictable naming makes an API feel professional and lowers the cognitive load for everyone. The conventions below are near-universal across well-designed APIs.

### Rule 1 — Nouns, not verbs

The HTTP **method** is the verb. The URL names a **thing**. So the URL should contain nouns, never action words. The method already says *what to do*; the URL says *to what*.

| Don't | Do | Method |
| --- | --- | --- |
| `GET /getUsers` | `GET /users` | List users |
| `POST /createUser` | `POST /users` | Create a user |
| `GET /getUserById/42` | `GET /users/42` | Get one user |
| `POST /updateUser/42` | `PATCH /users/42` | Update a user |
| `POST /deleteUser/42` | `DELETE /users/42` | Delete a user |

### Rule 2 — Plural nouns for collections

Use **plural** nouns consistently. `/users` is the collection; `/users/42` is one member of that collection. Mixing singular and plural (`/user` here, `/orders` there) is a common, jarring inconsistency.

```text
GET    /products            → list of products (the collection)
GET    /products/99         → one product (a member)
POST   /products            → create a product in the collection
PATCH  /products/99         → update one product
DELETE /products/99         → delete one product
```

### Rule 3 — Nest to express relationships (but not too deep)

When a resource belongs to another, nesting reads naturally: "the reviews **of** product 99."

```text
GET  /products/99/reviews        → reviews belonging to product 99
POST /products/99/reviews        → add a review to product 99
GET  /products/99/reviews/7      → one specific review of product 99
```

> BEST PRACTICE: Nest at most **one level deep** for reads/writes. `/products/99/reviews/7/comments/3` is brittle and verbose. Once you have a unique handle on the leaf resource, address it directly: `/reviews/7` or `/comments/3`. Use nesting to *create* and *list* within a parent, but let deep resources have their own top-level paths.

### Rule 4 — Hyphens, lowercase, no file extensions

- **Lowercase** paths: `/user-profiles`, not `/UserProfiles`.
- **Hyphens** (`-`) for multi-word segments, not underscores or camelCase: `/order-items`, not `/order_items` or `/orderItems`.
- **No file extensions** (`/users` not `/users.json`); use the `Accept` header for format negotiation.
- **Query strings for filtering/sorting/pagination**, never for identifying a resource: `/users?role=admin&sort=name`, not `/users/role/admin`.

### Rule 5 — Filters and operations

Filtering, sorting, searching, and pagination belong in the **query string**, because they refine a *retrieval* of an existing collection rather than name a new resource:

```text
GET /articles?status=published&tag=node&sort=-createdAt&page=2&limit=20
```

For non-CRUD actions that genuinely don't fit, model the action as a sub-resource you POST to:

```text
POST /orders/55/cancel          → acceptable, pragmatic "controller" pattern
POST /orders/55/cancellation    → more RESTful: create a "cancellation" sub-resource
```

---

## 5. Building a Full CRUD API, Step by Step (In-Memory)

Now we build a complete, working CRUD API for a `tasks` resource. We start with an **in-memory store** (a plain array) on purpose: no database setup, no async query distractions — just the HTTP design. Everything you learn here transfers directly when you swap the array for MongoDB/Mongoose in a later module; only the data-access lines change.

> TIP: An in-memory store resets every time the server restarts. That's fine for learning and for tests. The goal is to nail the *interface*, then plug in persistence behind the same routes.

### Project setup

```bash
mkdir tasks-api && cd tasks-api
npm init -y                 # create package.json
npm install express         # Express 4 (current widely-used) — notes on v5 below
node -v                     # confirm you're on a current LTS (e.g., v20 or v22)
```

### Step 0 — The data layer (in-memory store)

Keep data access in its own module so the routes don't care *where* data lives. This separation is what lets you later swap in a database without touching route logic.

```js
// store.js — a tiny in-memory "database" for tasks (CommonJS)
let tasks = [
  { id: 1, title: 'Learn REST', completed: false, createdAt: new Date().toISOString() },
  { id: 2, title: 'Build CRUD API', completed: false, createdAt: new Date().toISOString() },
];
let nextId = 3; // simple auto-increment id generator

module.exports = {
  // Return a shallow copy so callers can't mutate our internal array directly.
  findAll: () => [...tasks],

  // Find one by id, or undefined if not found.
  findById: (id) => tasks.find((t) => t.id === id),

  // Create: assign an id + timestamp, store, and return the new record.
  create: (data) => {
    const task = {
      id: nextId++,
      title: data.title,
      completed: data.completed ?? false, // default when omitted
      createdAt: new Date().toISOString(),
    };
    tasks.push(task);
    return task;
  },

  // Replace the stored record at index with `replacement`, return it.
  replace: (id, replacement) => {
    const idx = tasks.findIndex((t) => t.id === id);
    if (idx === -1) return undefined;
    tasks[idx] = replacement;
    return replacement;
  },

  // Merge `patch` fields into the existing record (partial update).
  update: (id, patch) => {
    const task = tasks.find((t) => t.id === id);
    if (!task) return undefined;
    Object.assign(task, patch); // mutate in place
    return task;
  },

  // Remove by id; return true if something was removed.
  remove: (id) => {
    const idx = tasks.findIndex((t) => t.id === id);
    if (idx === -1) return false;
    tasks.splice(idx, 1);
    return true;
  },
};
```

### Step 1 — The Express app skeleton

```js
// app.js — the Express application (CommonJS)
const express = require('express');
const store = require('./store');

const app = express();

// Parse incoming JSON request bodies into req.body.
// Without this, req.body is undefined for POST/PUT/PATCH JSON payloads.
app.use(express.json());

// ... routes go here (added in the next steps) ...

module.exports = app;
```

```js
// server.js — entry point that starts the HTTP listener
const app = require('./app');

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Tasks API listening on http://localhost:${PORT}`);
});
```

> TIP: Splitting `app.js` (the configured Express app) from `server.js` (the thing that calls `listen`) is a small move with a big payoff: your tests can `require('./app')` and exercise routes with **supertest** without ever opening a real port.

### Step 2 — READ the collection: `GET /tasks`

```js
// GET /tasks → return the list of all tasks (200 OK)
app.get('/tasks', (req, res) => {
  const tasks = store.findAll();
  res.status(200).json({ data: tasks }); // consistent envelope (see §6)
});
```

```bash
curl http://localhost:3000/tasks
# 200 OK
# { "data": [ { "id":1, "title":"Learn REST", ... }, ... ] }
```

### Step 3 — READ one: `GET /tasks/:id`

This is the first place we must handle "not found" correctly. Note also that route params are **always strings**, so we parse the id.

```js
// GET /tasks/:id → one task, or 404 if it doesn't exist
app.get('/tasks/:id', (req, res) => {
  const id = Number(req.params.id);        // params are strings → coerce to number

  if (!Number.isInteger(id)) {             // reject "/tasks/abc"
    return res.status(400).json({
      error: { code: 'INVALID_ID', message: 'id must be an integer' },
    });
  }

  const task = store.findById(id);
  if (!task) {
    return res.status(404).json({
      error: { code: 'NOT_FOUND', message: `Task ${id} not found` },
    });
  }

  res.status(200).json({ data: task });
});
```

### Step 4 — CREATE: `POST /tasks`

Creation needs **validation** (the body might be garbage) and returns **201** with a `Location` header.

```js
// POST /tasks → create a task (201 Created)
app.post('/tasks', (req, res) => {
  const { title, completed } = req.body ?? {};

  // --- validation ---
  const errors = [];
  if (typeof title !== 'string' || title.trim() === '') {
    errors.push({ field: 'title', message: 'title is required and must be a non-empty string' });
  }
  if (completed !== undefined && typeof completed !== 'boolean') {
    errors.push({ field: 'completed', message: 'completed must be a boolean' });
  }
  if (errors.length > 0) {
    // 422: body parsed fine but failed our rules (400 is also acceptable here)
    return res.status(422).json({ error: { code: 'VALIDATION_ERROR', details: errors } });
  }

  const task = store.create({ title: title.trim(), completed });

  // 201 + Location header pointing to the freshly created resource.
  res
    .status(201)
    .location(`/tasks/${task.id}`)
    .json({ data: task });
});
```

```bash
curl -i -X POST http://localhost:3000/tasks \
  -H "Content-Type: application/json" \
  -d '{ "title": "Write docs" }'
# HTTP/1.1 201 Created
# Location: /tasks/3
# { "data": { "id":3, "title":"Write docs", "completed":false, "createdAt":"..." } }
```

### Step 5 — REPLACE (full update): `PUT /tasks/:id`

`PUT` replaces the whole resource. The client must send a complete representation; we keep the immutable bits (`id`, `createdAt`) but reset everything else from the body.

```js
// PUT /tasks/:id → full replacement (200 OK) or 404
app.put('/tasks/:id', (req, res) => {
  const id = Number(req.params.id);
  if (!Number.isInteger(id)) {
    return res.status(400).json({ error: { code: 'INVALID_ID', message: 'id must be an integer' } });
  }

  const existing = store.findById(id);
  if (!existing) {
    return res.status(404).json({ error: { code: 'NOT_FOUND', message: `Task ${id} not found` } });
  }

  const { title, completed } = req.body ?? {};

  // PUT requires the FULL representation → both fields validated as required.
  const errors = [];
  if (typeof title !== 'string' || title.trim() === '') {
    errors.push({ field: 'title', message: 'title is required' });
  }
  if (typeof completed !== 'boolean') {
    errors.push({ field: 'completed', message: 'completed is required (boolean)' });
  }
  if (errors.length > 0) {
    return res.status(422).json({ error: { code: 'VALIDATION_ERROR', details: errors } });
  }

  // Build the complete replacement, preserving server-managed immutable fields.
  const replacement = {
    id: existing.id,
    createdAt: existing.createdAt,
    title: title.trim(),
    completed,
  };
  const updated = store.replace(id, replacement);
  res.status(200).json({ data: updated });
});
```

### Step 6 — PARTIAL update: `PATCH /tasks/:id`

`PATCH` changes only the supplied fields. Each provided field is validated; omitted fields are left untouched.

```js
// PATCH /tasks/:id → partial update (200 OK) or 404
app.patch('/tasks/:id', (req, res) => {
  const id = Number(req.params.id);
  if (!Number.isInteger(id)) {
    return res.status(400).json({ error: { code: 'INVALID_ID', message: 'id must be an integer' } });
  }

  const existing = store.findById(id);
  if (!existing) {
    return res.status(404).json({ error: { code: 'NOT_FOUND', message: `Task ${id} not found` } });
  }

  const body = req.body ?? {};
  const patch = {};
  const errors = [];

  // Only validate + apply fields that are actually present.
  if ('title' in body) {
    if (typeof body.title !== 'string' || body.title.trim() === '') {
      errors.push({ field: 'title', message: 'title must be a non-empty string' });
    } else {
      patch.title = body.title.trim();
    }
  }
  if ('completed' in body) {
    if (typeof body.completed !== 'boolean') {
      errors.push({ field: 'completed', message: 'completed must be a boolean' });
    } else {
      patch.completed = body.completed;
    }
  }
  if (errors.length > 0) {
    return res.status(422).json({ error: { code: 'VALIDATION_ERROR', details: errors } });
  }
  if (Object.keys(patch).length === 0) {
    return res.status(400).json({ error: { code: 'EMPTY_PATCH', message: 'No updatable fields provided' } });
  }

  const updated = store.update(id, patch);
  res.status(200).json({ data: updated });
});
```

### Step 7 — DELETE: `DELETE /tasks/:id`

```js
// DELETE /tasks/:id → 204 No Content on success, 404 if absent
app.delete('/tasks/:id', (req, res) => {
  const id = Number(req.params.id);
  if (!Number.isInteger(id)) {
    return res.status(400).json({ error: { code: 'INVALID_ID', message: 'id must be an integer' } });
  }

  const removed = store.remove(id);
  if (!removed) {
    return res.status(404).json({ error: { code: 'NOT_FOUND', message: `Task ${id} not found` } });
  }

  // 204: success with no body. Do NOT call res.json() here — send nothing.
  res.status(204).end();
});
```

```bash
curl -i -X DELETE http://localhost:3000/tasks/1
# HTTP/1.1 204 No Content   (empty body)
```

That is a complete, correct CRUD API: list, read-one, create, full-update, partial-update, delete — each with the right method, status code, and error handling. Everything from here on is about making it *production-grade*.

> BEST PRACTICE: Notice the validation logic is repeated across handlers. In §6–§7 we centralize it. In a real app you'd extract validation into middleware (or a schema library like **Zod**/**Joi**/**express-validator**) and the not-found check into a reusable helper. Repetition like this is a code smell that should push you toward middleware — a topic we lean on heavily next.

### Refactor preview — Express Router

As an API grows you don't keep stacking routes in `app.js`. You group a resource's routes in an `express.Router` and mount it under a base path:

```js
// routes/tasks.js
const express = require('express');
const router = express.Router();
const store = require('../store');

router.get('/', (req, res) => res.json({ data: store.findAll() }));
router.get('/:id', /* ...handler... */);
// router.post('/', ...), router.put('/:id', ...), etc.

module.exports = router;
```

```js
// app.js
const tasksRouter = require('./routes/tasks');
app.use('/tasks', tasksRouter); // all routes now relative to /tasks
```

---

## 6. Request Validation & Consistent Response Shapes

### Why a consistent envelope

When every endpoint returns data in a different shape — sometimes a raw array, sometimes a bare object, sometimes wrapped — consumers must special-case each endpoint. A **consistent response envelope** means clients write *one* parser for success and *one* for errors, across the whole API.

A common, pragmatic envelope:

```json
// Success (single resource)
{ "data": { "id": 3, "title": "Write docs", "completed": false } }

// Success (collection, with metadata)
{
  "data": [ { "id": 1, "title": "Learn REST" } ],
  "meta": { "page": 1, "limit": 20, "total": 57 }
}

// Error
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The request body failed validation.",
    "details": [ { "field": "title", "message": "title is required" } ]
  }
}
```

The contract: **success responses always have a `data` key; error responses always have an `error` key.** A client can branch on which key is present. (There are other valid conventions — e.g., JSON:API — but the principle is *pick one and apply it everywhere*.)

> TIP: Don't wrap *every* primitive needlessly, but do be consistent. The most common mistake is returning a bare array from list endpoints and a wrapped object from others. If you ever need to add pagination metadata later, a bare array leaves you nowhere to put it without a breaking change — the envelope future-proofs you.

### Validation basics

Validation is the practice of rejecting bad input *before* it reaches your business logic. The rules of thumb:

1. **Never trust the client.** Validate types, presence, ranges, formats, and enums.
2. **Fail fast and clearly.** Return all validation errors at once (an array of `{ field, message }`), not just the first — it saves the client round-trips.
3. **Validate at the edge.** Do it in middleware/handlers before touching the store, so invalid data never propagates.
4. **Choose 400 or 422 consistently.** (See §3.)

Hand-rolled validation (as in §5) is fine for tiny APIs, but it gets verbose and error-prone. Real projects use a **schema validation library**. Here's the same task-creation validation expressed declaratively with **Zod**:

```js
// validation with Zod — declarative, reusable, single source of truth
const { z } = require('zod');

// Define the shape and rules once.
const createTaskSchema = z.object({
  title: z.string().trim().min(1, 'title is required'),
  completed: z.boolean().optional(),
});

// A reusable middleware factory that validates req.body against a schema.
function validateBody(schema) {
  return (req, res, next) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      // Convert Zod's issues into our standard error envelope.
      const details = result.error.issues.map((i) => ({
        field: i.path.join('.'),
        message: i.message,
      }));
      return res.status(422).json({ error: { code: 'VALIDATION_ERROR', details } });
    }
    req.body = result.data; // sanitized/coerced data going forward
    next();
  };
}

// Usage: app.post('/tasks', validateBody(createTaskSchema), createTaskHandler);
```

> BEST PRACTICE: Make validated/sanitized data the *only* thing downstream code sees (overwrite `req.body` with the parsed result, as above). This eliminates an entire class of bugs where you validate one variable but use the unsanitized original.

---

## 7. Structured Error Responses

When something goes wrong, the response must be **machine-readable**, not just a human sentence. A client app needs to *programmatically* distinguish "email already taken" from "you're not logged in" so it can show the right UI. That requires structure.

### Anatomy of a good error response

```json
{
  "error": {
    "code": "EMAIL_TAKEN",                       // stable, machine-readable identifier
    "message": "An account with that email already exists.", // human-readable
    "details": [ { "field": "email", "message": "already in use" } ], // optional specifics
    "requestId": "req_8c1f...",                  // optional, for support/log correlation
    "docs": "https://api.example.com/docs/errors#EMAIL_TAKEN" // optional link
  }
}
```

The **`code`** is the most important field. The HTTP status (`409`) tells the client the *class* of problem; the `code` tells it the *specific* problem. Status codes are coarse (many situations are "409 Conflict"), so a stable application-level `code` lets clients branch precisely without parsing English messages that may be localized or reworded later.

### Centralizing errors with an Express error handler

Scattering `res.status(...).json(...)` across every handler (as in §5) is repetitive and easy to get inconsistent. The robust pattern: throw/`next()` a typed error and catch it in **one** error-handling middleware.

```js
// errors.js — a typed application error
class ApiError extends Error {
  constructor(status, code, message, details) {
    super(message);
    this.status = status;   // HTTP status to send
    this.code = code;       // stable machine code
    this.details = details; // optional array of field errors
  }
}

// Convenience factories for common cases.
ApiError.notFound = (msg = 'Resource not found') => new ApiError(404, 'NOT_FOUND', msg);
ApiError.validation = (details) => new ApiError(422, 'VALIDATION_ERROR', 'Validation failed', details);
ApiError.conflict = (msg) => new ApiError(409, 'CONFLICT', msg);

module.exports = { ApiError };
```

```js
// In a route handler, just throw (or next(err)):
const { ApiError } = require('./errors');

app.get('/tasks/:id', (req, res, next) => {
  const task = store.findById(Number(req.params.id));
  if (!task) return next(ApiError.notFound(`Task ${req.params.id} not found`));
  res.json({ data: task });
});
```

```js
// The SINGLE error-handling middleware — note the 4 args (err, req, res, next).
// Express identifies it as an error handler by its arity. Register it LAST.
app.use((err, req, res, next) => {
  // Known, expected application errors:
  if (err instanceof ApiError) {
    return res.status(err.status).json({
      error: { code: err.code, message: err.message, details: err.details },
    });
  }

  // Body-parser threw on malformed JSON → that's a client 400, not a 500.
  if (err.type === 'entity.parse.failed') {
    return res.status(400).json({ error: { code: 'INVALID_JSON', message: 'Malformed JSON body' } });
  }

  // Anything else is an unexpected bug → 500, and DON'T leak internals.
  console.error(err); // log the full error server-side for debugging
  res.status(500).json({ error: { code: 'INTERNAL', message: 'Something went wrong' } });
});
```

> WARNING — Pitfall: Express 4 does **not** automatically catch errors thrown in *async* handlers. If you `throw` inside an `async` function, the rejected promise won't reach your error middleware unless you `.catch(next)` or wrap with a helper. Express **5** fixes this — it forwards rejected promises from async handlers to the error handler automatically. Until you're on v5, use a wrapper:

```js
// asyncHandler: forwards async errors to Express's error middleware (needed in Express 4)
const asyncHandler = (fn) => (req, res, next) => Promise.resolve(fn(req, res, next)).catch(next);

// Usage in Express 4:
app.get('/tasks/:id', asyncHandler(async (req, res) => {
  const task = await store.findByIdAsync(Number(req.params.id));
  if (!task) throw ApiError.notFound();   // safely routed to the error handler
  res.json({ data: task });
}));
```

> WARNING — Pitfall: Never include stack traces, SQL, or internal exception messages in a 5xx response body. They leak implementation details to attackers and confuse legitimate clients. Log them server-side; return a generic message plus (optionally) a `requestId` clients can quote to support.

---

## 8. Pagination, Filtering, and Sorting

A `GET /tasks` that returns *every* row is a time bomb: it works with 10 rows and falls over with 10 million. Collection endpoints need to let clients ask for a slice, a subset, and an order. All three live in the **query string**, because they refine a retrieval.

### Pagination

Two dominant strategies:

| Strategy | How it works | Pros | Cons |
| --- | --- | --- | --- |
| **Offset/limit** | `?page=2&limit=20` → skip `(page-1)*limit`, take `limit`. | Trivial to implement; jump to any page. | Slow on huge offsets; items can shift between pages if data changes (duplicates/skips). |
| **Cursor (keyset)** | `?limit=20&cursor=<opaque>` → "give me 20 after this marker." | Stable under inserts/deletes; fast at scale. | Can't jump to arbitrary page; more complex. |

Offset pagination is the right default for learning and for most CRUD apps; reach for cursors when datasets are large or change rapidly (feeds, logs).

```js
// GET /tasks?page=2&limit=20  — offset pagination over the in-memory store
app.get('/tasks', (req, res) => {
  // Parse + clamp inputs; never trust raw query values.
  const page = Math.max(1, parseInt(req.query.page, 10) || 1);
  const limit = Math.min(100, Math.max(1, parseInt(req.query.limit, 10) || 20)); // cap at 100!
  const offset = (page - 1) * limit;

  const all = store.findAll();
  const total = all.length;
  const pageItems = all.slice(offset, offset + limit);

  res.json({
    data: pageItems,
    meta: {
      page,
      limit,
      total,                                 // total matching records
      totalPages: Math.ceil(total / limit),  // helps clients build pagers
    },
  });
});
```

> WARNING — Pitfall: Always **cap `limit`** to a sane maximum (e.g., 100). Without a cap, a client can request `?limit=10000000` and exhaust your memory/database. Also always validate that `page`/`limit` are positive integers — `parseInt('abc')` is `NaN`, which would corrupt your math.

### Filtering

Filters narrow *which* records come back, keyed by field. Whitelist allowed filter fields so clients can't filter on arbitrary internal columns.

```js
// GET /tasks?completed=true&search=rest
function applyFilters(items, query) {
  let result = items;

  // Boolean filter — query strings are ALWAYS strings, so compare to "true".
  if (query.completed !== undefined) {
    const wantCompleted = query.completed === 'true';
    result = result.filter((t) => t.completed === wantCompleted);
  }

  // Free-text search over a field (case-insensitive contains).
  if (typeof query.search === 'string' && query.search.trim() !== '') {
    const needle = query.search.trim().toLowerCase();
    result = result.filter((t) => t.title.toLowerCase().includes(needle));
  }

  return result;
}
```

### Sorting

A common convention: a `sort` param naming the field, with a leading `-` for descending. Whitelist sortable fields to prevent abuse.

```js
// GET /tasks?sort=-createdAt   (descending) or ?sort=title (ascending)
const SORTABLE = new Set(['title', 'createdAt', 'completed']); // whitelist

function applySort(items, sortParam) {
  if (!sortParam) return items;

  const desc = sortParam.startsWith('-');
  const field = desc ? sortParam.slice(1) : sortParam;
  if (!SORTABLE.has(field)) return items; // ignore disallowed sort keys

  return [...items].sort((a, b) => {
    if (a[field] < b[field]) return desc ? 1 : -1;
    if (a[field] > b[field]) return desc ? -1 : 1;
    return 0;
  });
}
```

### Putting it together

```js
// GET /tasks?completed=true&search=rest&sort=-createdAt&page=1&limit=10
app.get('/tasks', (req, res) => {
  const page = Math.max(1, parseInt(req.query.page, 10) || 1);
  const limit = Math.min(100, Math.max(1, parseInt(req.query.limit, 10) || 20));
  const offset = (page - 1) * limit;

  // 1) filter → 2) sort → 3) paginate.  Order matters: total must reflect filters.
  const filtered = applyFilters(store.findAll(), req.query);
  const sorted = applySort(filtered, req.query.sort);
  const total = sorted.length;
  const pageItems = sorted.slice(offset, offset + limit);

  res.json({
    data: pageItems,
    meta: { page, limit, total, totalPages: Math.ceil(total / limit) },
  });
});
```

> BEST PRACTICE: Compute `total` *after* filtering but *before* paginating, so `totalPages` reflects the filtered result set, not the whole table. Getting this order wrong produces pagers that show the wrong number of pages.

---

## 9. API Versioning (Intro)

APIs change. New required fields, renamed properties, removed endpoints — any of these can break existing clients you don't control (mobile apps in the wild, third-party integrations). **Versioning** lets you evolve the API while keeping old clients working. The golden rule: **never make a breaking change to a released, versioned API; ship a new version instead.**

### Common versioning strategies

| Strategy | Example | Notes |
| --- | --- | --- |
| **URI path** | `/v1/tasks`, `/v2/tasks` | Most common, most visible, easy to route and cache. Slightly "un-RESTful" (the resource didn't change), but pragmatic and overwhelmingly popular. |
| **Query parameter** | `/tasks?version=2` | Easy, but easy to forget; muddies caching. |
| **Custom header** | `X-API-Version: 2` | Keeps URLs clean; harder to test in a browser/curl; less visible. |
| **`Accept` header (media type)** | `Accept: application/vnd.example.v2+json` | "Purest" REST (content negotiation), but awkward and rarely used outside large APIs. |

```js
// URI path versioning with separate routers per version
const v1 = require('./routes/v1/tasks');
const v2 = require('./routes/v2/tasks');

app.use('/v1/tasks', v1);
app.use('/v2/tasks', v2); // v2 can change shapes without breaking v1 clients
```

> BEST PRACTICE: Start with **URI path versioning** (`/v1/...`) — it's the simplest to implement, document, route, and cache, and it's what most consumers expect. Decide what counts as a "breaking change" up front (removing/renaming a field, changing a type, adding a required request field) versus a non-breaking one (adding an optional field) so you only mint new versions when truly necessary.

> TIP: Version from day one, even if you only ever have `v1`. Retrofitting versioning onto an unversioned API in production is painful; prefixing `/v1` costs nothing now and buys you a clean upgrade path later.

---

## 10. Idempotency & Safety (Putting It in Practice)

We defined safe and idempotent in §2. Here's how those properties drive concrete design and operational decisions — the part interviewers want to see you connect.

- **Safe methods must not mutate.** A `GET` that increments a view counter as a side effect is a classic bug: crawlers, prefetchers, and link-previews issue GETs constantly, inflating your counts and possibly causing harm. If you must record a view, do it via a separate explicit action, and never let a GET change meaningful state.
- **Make writes idempotent where you can.** `PUT` and `DELETE` are naturally idempotent — lean on that. It means clients (and your own retry logic, gateways, and queues) can safely re-send a request after a network blip without fear of double-applying it.
- **Protect non-idempotent POSTs with idempotency keys.** Because `POST` creates, a retried POST can duplicate. The industry pattern (used by Stripe and others): the client sends a unique `Idempotency-Key` header; the server records the key with the first response and, on any retry with the same key, returns the *original* result instead of acting again.

```js
// Minimal idempotency-key handling for POST (in-memory cache of past responses)
const seenKeys = new Map(); // key -> { status, body }  (use Redis/DB in production)

app.post('/tasks', (req, res) => {
  const key = req.get('Idempotency-Key');

  // If we've already processed this key, replay the stored response.
  if (key && seenKeys.has(key)) {
    const prev = seenKeys.get(key);
    return res.status(prev.status).json(prev.body);
  }

  // ... validate + create the task as usual ...
  const task = store.create({ title: req.body.title });
  const body = { data: task };

  if (key) seenKeys.set(key, { status: 201, body }); // remember for future retries
  res.status(201).location(`/tasks/${task.id}`).json(body);
});
```

> TIP: A clean interview soundbite: "Safe = no side effects (read-only). Idempotent = same request repeated yields the same *state*. GET/PUT/DELETE are idempotent; POST is not, which is why we protect it with idempotency keys for safe retries." Then mention that idempotency is fundamentally about **reliable retries over an unreliable network**.

---

## 11. A Note on HATEOAS

**HATEOAS** — **H**ypermedia **A**s **T**he **E**ngine **O**f **A**pplication **S**tate — is the part of the uniform-interface constraint that makes an API *fully* RESTful in Fielding's strict sense. The idea: responses include **links** describing what the client can do next, so the client navigates the API by following links rather than hard-coding URLs — exactly like a human follows links on a web page without knowing the site's URL structure in advance.

```json
// A HATEOAS-style task representation with hypermedia controls
{
  "data": {
    "id": 3,
    "title": "Write docs",
    "completed": false,
    "_links": {
      "self":     { "href": "/tasks/3", "method": "GET" },
      "update":   { "href": "/tasks/3", "method": "PATCH" },
      "delete":   { "href": "/tasks/3", "method": "DELETE" },
      "complete": { "href": "/tasks/3", "method": "PATCH", "body": { "completed": true } }
    }
  }
}
```

The theoretical payoff: clients become loosely coupled to URL structure, the server can move resources around, and available actions (e.g., a "cancel" link that only appears when an order is cancellable) become discoverable from the data itself.

**The reality:** the vast majority of "REST" APIs do *not* implement HATEOAS. It adds significant complexity, most clients ignore the links and hard-code URLs anyway, and the tooling/benefits rarely justify the cost for typical JSON APIs. According to the **Richardson Maturity Model** (a useful framework for grading API "RESTfulness": Level 0 = single endpoint/RPC; Level 1 = resources; Level 2 = HTTP verbs + status codes; Level 3 = hypermedia/HATEOAS), most production APIs sit at **Level 2** and are perfectly successful there.

> TIP: The honest, interview-ready take: "HATEOAS is the highest level of the Richardson Maturity Model and the constraint that makes an API truly REST per Fielding. In practice it's rarely fully implemented because the cost usually outweighs the benefit for JSON APIs — most production APIs are Level 2. Knowing it exists and *why* it's often skipped is more valuable than implementing it everywhere."

---

## Common Pitfalls & Gotchas

- **Verbs in URLs.** `/getUsers`, `/createOrder`. The method is the verb; the URL is a noun. (`GET /users`, `POST /orders`.)
- **Wrong status codes.** Returning `200` for errors, `200` instead of `201` on create, `404` where a `400`/`422` belongs (or vice versa), or a `500` for what is really invalid client input.
- **Confusing 401 and 403.** 401 = not authenticated; 403 = authenticated but not permitted. Mixing these confuses clients about whether to re-login or give up.
- **Asserting PATCH is idempotent.** It depends on the patch; an "increment" patch is not idempotent.
- **Forgetting `express.json()`.** Without `app.use(express.json())`, `req.body` is `undefined` and every POST/PUT/PATCH silently misbehaves.
- **Route params are strings.** `req.params.id` is `"42"`, not `42`. Coerce and validate before using it (and `Number('')` is `0`, `Number('abc')` is `NaN` — guard with `Number.isInteger`).
- **Mutating the internal store from outside.** Returning a direct reference to your in-memory array lets callers corrupt it. Return copies.
- **Unbounded list endpoints.** No pagination, or an uncapped `limit`, invites memory/DB exhaustion.
- **Inconsistent response shapes.** Bare array here, wrapped object there — every consumer pays the tax.
- **Leaking internals in 5xx.** Stack traces/SQL in error responses are a security and usability hazard.
- **Async errors not reaching the error handler (Express 4).** Thrown errors in `async` handlers vanish unless you `.catch(next)` or use a wrapper. (Express 5 handles this for you.)
- **`PUT` that doesn't actually replace.** If your `PUT` only updates supplied fields, you've built a `PATCH` and mislabeled it. Be deliberate.
- **Putting filters in the path.** `/users/role/admin` should be `/users?role=admin`. The path identifies resources; the query refines retrieval.

---

## Best Practices

- **Design the contract first.** Decide resource names, methods, status codes, and the response envelope before writing handlers. Sketch the routes table; the code follows.
- **One consistent envelope everywhere.** `data` for success, `error` (with a stable `code`) for failure. Add `meta` for pagination.
- **Use the right status code, always.** 201 + `Location` on create; 204 on delete; 200 on reads/updates that return a body; 4xx for client errors; 5xx only for genuine server faults.
- **Validate at the edge with schemas.** Use Zod/Joi/express-validator; overwrite `req.body` with sanitized output; return *all* errors at once.
- **Centralize error handling.** One typed `ApiError` class plus one error-handling middleware. Log full details server-side; return safe, structured messages.
- **Plural nouns, shallow nesting, hyphen-case, lowercase.** Keep URLs predictable and boring.
- **Paginate every collection, cap the limit.** Return pagination metadata in `meta`.
- **Whitelist filter and sort fields.** Never let clients sort/filter on arbitrary internal columns.
- **Version from day one** (`/v1`), and define what "breaking" means.
- **Lean on idempotency for safe retries**; protect POSTs with idempotency keys where duplicates would be costly.
- **Separate concerns:** data-access module, route/controller layer, validation layer, error layer. The in-memory store should be swappable for a database without touching routes.
- **Split `app` from `server`** so routes are testable with supertest.

---

## Key Interview Points

- **"What is REST?"** — An architectural style defined by constraints: client–server, statelessness, cacheability, a uniform interface, layered system, and optional code-on-demand. Resources are addressable by URI; clients exchange *representations* of resource state.
- **"What does statelessness mean and why does it matter?"** — Each request carries everything needed to process it; the server keeps no per-client session memory between requests. This enables horizontal scaling (any instance handles any request) and improves reliability and visibility.
- **"Difference between safe and idempotent?"** — Safe = read-only, no state change (GET, HEAD, OPTIONS). Idempotent = repeating the same request leaves the *state* identical (GET, PUT, DELETE; not POST; PATCH only sometimes). Idempotency enables safe retries.
- **"PUT vs PATCH?"** — PUT replaces the entire resource (omitted fields are dropped/reset); PATCH applies a partial update (only supplied fields change). PUT is idempotent; PATCH may or may not be.
- **"When 201 vs 200 vs 204?"** — 201 + `Location` for a created resource; 200 for success that returns a body; 204 for success with no body (typical for DELETE).
- **"401 vs 403?"** — 401 = unauthenticated (who are you?); 403 = authenticated but not permitted (I know you; no).
- **"400 vs 422?"** — 400 for malformed/unparseable requests; 422 for syntactically valid bodies that fail validation rules. Pick one convention and apply it consistently.
- **"How do you version an API?"** — Most commonly URI path versioning (`/v1`); also query param, custom header, or `Accept` media type. Never break a released version — ship a new one.
- **"What's HATEOAS and do you use it?"** — Hypermedia links in responses driving client navigation; Level 3 of the Richardson Maturity Model. Rarely fully implemented because cost usually outweighs benefit; most APIs are Level 2.
- **"How do you handle errors in an Express API?"** — Throw a typed error, catch it in a single error-handling middleware (4-arg signature, registered last), return a structured `{ error: { code, message, details } }`, log internals server-side, never leak stack traces. Note Express 4 needs an async wrapper; Express 5 forwards async rejections automatically.
- **"How do you paginate?"** — Offset/limit (`?page&limit`) for most cases; cursor/keyset for large or fast-changing datasets. Always cap `limit` and return total/totalPages in `meta`.

---

## Practice Exercises

Work these in order; each builds on the last. Start from the in-memory `tasks` API in §5.

1. **Status-code triage (no code).** For each scenario, give the correct status code and one sentence why: (a) created a task, (b) deleted a task that exists, (c) `GET /tasks/9999` where 9999 doesn't exist, (d) `POST /tasks` with an empty body, (e) a logged-out user calls a protected route, (f) a logged-in non-admin tries to delete another user, (g) your handler throws `TypeError: cannot read property 'x' of undefined`.

2. **Add a `categories` resource.** Implement full CRUD for `/categories` (`{ id, name }`, name required & unique). Return **409 Conflict** with code `NAME_TAKEN` when creating a duplicate name.

3. **Nest tasks under categories.** Add `GET /categories/:id/tasks` (list tasks in a category) and `POST /categories/:id/tasks` (create a task already assigned to that category). Return 404 if the category doesn't exist. (Add a `categoryId` field to tasks.)

4. **Pagination, filtering, sorting.** Extend `GET /tasks` to support `?completed=`, `?search=`, `?sort=` (with `-` prefix for descending over a whitelist), and `?page=`/`?limit=` (cap limit at 50). Return a `meta` block with `page`, `limit`, `total`, `totalPages`.

5. **Centralized errors + validation.** Refactor all handlers to throw a typed `ApiError` and handle everything in one error-handling middleware. Replace inline validation with a `validateBody(schema)` middleware (Zod or hand-rolled). Ensure malformed JSON yields a 400 with code `INVALID_JSON`.

6. **Idempotent create.** Add `Idempotency-Key` support to `POST /tasks`: a repeated request with the same key returns the original 201 response (and does not create a second task).

### Solutions

**Solution 1 — Status-code triage**

| Scenario | Code | Why |
| --- | --- | --- |
| (a) Created a task | **201 Created** | A new resource was created; include `Location`. |
| (b) Deleted an existing task | **204 No Content** | Success with nothing to return. |
| (c) GET a non-existent id | **404 Not Found** | The addressed resource doesn't exist. |
| (d) POST with empty body | **422** (or 400) | Body parsed but failed validation (required `title` missing). |
| (e) Logged-out user, protected route | **401 Unauthorized** | No valid credentials = unauthenticated. |
| (f) Non-admin deletes another user | **403 Forbidden** | Authenticated but lacks permission. |
| (g) Handler throws TypeError | **500 Internal Server Error** | Unexpected server bug; log it, return a generic message. |

**Solution 2 — `categories` resource with uniqueness**

```js
// store-categories.js
let categories = [];
let nextId = 1;

module.exports = {
  findAll: () => [...categories],
  findById: (id) => categories.find((c) => c.id === id),
  findByName: (name) => categories.find((c) => c.name.toLowerCase() === name.toLowerCase()),
  create: (name) => {
    const cat = { id: nextId++, name };
    categories.push(cat);
    return cat;
  },
  update: (id, patch) => {
    const cat = categories.find((c) => c.id === id);
    if (!cat) return undefined;
    Object.assign(cat, patch);
    return cat;
  },
  remove: (id) => {
    const idx = categories.findIndex((c) => c.id === id);
    if (idx === -1) return false;
    categories.splice(idx, 1);
    return true;
  },
};
```

```js
// routes/categories.js
const express = require('express');
const router = express.Router();
const cats = require('../store-categories');

// CREATE with uniqueness check → 409 on duplicate name.
router.post('/', (req, res) => {
  const { name } = req.body ?? {};
  if (typeof name !== 'string' || name.trim() === '') {
    return res.status(422).json({
      error: { code: 'VALIDATION_ERROR', details: [{ field: 'name', message: 'name is required' }] },
    });
  }
  if (cats.findByName(name.trim())) {
    return res.status(409).json({
      error: { code: 'NAME_TAKEN', message: `Category "${name.trim()}" already exists` },
    });
  }
  const cat = cats.create(name.trim());
  res.status(201).location(`/categories/${cat.id}`).json({ data: cat });
});

router.get('/', (req, res) => res.json({ data: cats.findAll() }));

router.get('/:id', (req, res) => {
  const cat = cats.findById(Number(req.params.id));
  if (!cat) return res.status(404).json({ error: { code: 'NOT_FOUND', message: 'Category not found' } });
  res.json({ data: cat });
});

router.patch('/:id', (req, res) => {
  const id = Number(req.params.id);
  if (!cats.findById(id)) return res.status(404).json({ error: { code: 'NOT_FOUND', message: 'Category not found' } });
  const { name } = req.body ?? {};
  if (name !== undefined) {
    if (typeof name !== 'string' || name.trim() === '') {
      return res.status(422).json({ error: { code: 'VALIDATION_ERROR', details: [{ field: 'name', message: 'invalid name' }] } });
    }
    const dupe = cats.findByName(name.trim());
    if (dupe && dupe.id !== id) {
      return res.status(409).json({ error: { code: 'NAME_TAKEN', message: 'name already in use' } });
    }
  }
  const updated = cats.update(id, name !== undefined ? { name: name.trim() } : {});
  res.json({ data: updated });
});

router.delete('/:id', (req, res) => {
  if (!cats.remove(Number(req.params.id))) {
    return res.status(404).json({ error: { code: 'NOT_FOUND', message: 'Category not found' } });
  }
  res.status(204).end();
});

module.exports = router;
```

**Solution 3 — Nesting tasks under categories**

```js
// Assumes the tasks store gained a `categoryId` field and a findByCategory helper:
//   findByCategory: (categoryId) => tasks.filter((t) => t.categoryId === categoryId)

// GET /categories/:id/tasks → tasks in a category (404 if category missing)
router.get('/:id/tasks', (req, res) => {
  const categoryId = Number(req.params.id);
  if (!cats.findById(categoryId)) {
    return res.status(404).json({ error: { code: 'NOT_FOUND', message: 'Category not found' } });
  }
  res.json({ data: store.findByCategory(categoryId) });
});

// POST /categories/:id/tasks → create a task already assigned to this category
router.post('/:id/tasks', (req, res) => {
  const categoryId = Number(req.params.id);
  if (!cats.findById(categoryId)) {
    return res.status(404).json({ error: { code: 'NOT_FOUND', message: 'Category not found' } });
  }
  const { title } = req.body ?? {};
  if (typeof title !== 'string' || title.trim() === '') {
    return res.status(422).json({ error: { code: 'VALIDATION_ERROR', details: [{ field: 'title', message: 'title is required' }] } });
  }
  // The parent id comes from the URL, not the body — that's the point of nesting.
  const task = store.create({ title: title.trim(), categoryId });
  res.status(201).location(`/tasks/${task.id}`).json({ data: task });
});
```

**Solution 4 — Pagination, filtering, sorting**

```js
const SORTABLE = new Set(['title', 'createdAt', 'completed']);

app.get('/tasks', (req, res) => {
  // --- filter ---
  let items = store.findAll();
  if (req.query.completed !== undefined) {
    const want = req.query.completed === 'true';
    items = items.filter((t) => t.completed === want);
  }
  if (typeof req.query.search === 'string' && req.query.search.trim()) {
    const needle = req.query.search.trim().toLowerCase();
    items = items.filter((t) => t.title.toLowerCase().includes(needle));
  }

  // --- sort ---
  if (req.query.sort) {
    const desc = req.query.sort.startsWith('-');
    const field = desc ? req.query.sort.slice(1) : req.query.sort;
    if (SORTABLE.has(field)) {
      items = [...items].sort((a, b) =>
        a[field] < b[field] ? (desc ? 1 : -1) : a[field] > b[field] ? (desc ? -1 : 1) : 0
      );
    }
  }

  // --- paginate (compute total AFTER filtering, BEFORE slicing) ---
  const page = Math.max(1, parseInt(req.query.page, 10) || 1);
  const limit = Math.min(50, Math.max(1, parseInt(req.query.limit, 10) || 20)); // cap 50
  const total = items.length;
  const data = items.slice((page - 1) * limit, (page - 1) * limit + limit);

  res.json({ data, meta: { page, limit, total, totalPages: Math.ceil(total / limit) } });
});
```

**Solution 5 — Centralized errors + validation middleware**

```js
// errors.js
class ApiError extends Error {
  constructor(status, code, message, details) {
    super(message);
    this.status = status; this.code = code; this.details = details;
  }
}
ApiError.notFound = (m = 'Not found') => new ApiError(404, 'NOT_FOUND', m);
ApiError.validation = (details) => new ApiError(422, 'VALIDATION_ERROR', 'Validation failed', details);
module.exports = { ApiError };
```

```js
// middleware/validateBody.js (hand-rolled, no external dep)
const { ApiError } = require('../errors');

// rules: { field: (value) => string|null  (return message if invalid) }
module.exports = (rules) => (req, res, next) => {
  const body = req.body ?? {};
  const details = [];
  for (const [field, check] of Object.entries(rules)) {
    const msg = check(body[field], body);
    if (msg) details.push({ field, message: msg });
  }
  if (details.length) return next(ApiError.validation(details));
  next();
};
```

```js
// app.js — wiring + the single error handler
const { ApiError } = require('./errors');
const validateBody = require('./middleware/validateBody');

const requireTitle = (v) =>
  typeof v === 'string' && v.trim() !== '' ? null : 'title is required (non-empty string)';

app.post('/tasks', validateBody({ title: requireTitle }), (req, res) => {
  const task = store.create({ title: req.body.title.trim() });
  res.status(201).location(`/tasks/${task.id}`).json({ data: task });
});

app.get('/tasks/:id', (req, res, next) => {
  const task = store.findById(Number(req.params.id));
  if (!task) return next(ApiError.notFound(`Task ${req.params.id} not found`));
  res.json({ data: task });
});

// SINGLE error handler — registered LAST, 4 args.
app.use((err, req, res, next) => {
  if (err instanceof ApiError) {
    return res.status(err.status).json({ error: { code: err.code, message: err.message, details: err.details } });
  }
  if (err.type === 'entity.parse.failed') {
    return res.status(400).json({ error: { code: 'INVALID_JSON', message: 'Malformed JSON body' } });
  }
  console.error(err);
  res.status(500).json({ error: { code: 'INTERNAL', message: 'Something went wrong' } });
});
```

**Solution 6 — Idempotent create**

```js
const idempotencyCache = new Map(); // key -> { status, body }. Use Redis with TTL in production.

app.post('/tasks', validateBody({ title: requireTitle }), (req, res) => {
  const key = req.get('Idempotency-Key');

  // Replay the stored response if we've seen this key before.
  if (key && idempotencyCache.has(key)) {
    const prev = idempotencyCache.get(key);
    return res.status(prev.status).json(prev.body);
  }

  const task = store.create({ title: req.body.title.trim() });
  const body = { data: task };

  if (key) idempotencyCache.set(key, { status: 201, body }); // remember for retries
  res.status(201).location(`/tasks/${task.id}`).json(body);
});
```

```bash
# Same key twice → only ONE task created; second call replays the first response.
curl -s -X POST localhost:3000/tasks -H 'Content-Type: application/json' \
  -H 'Idempotency-Key: abc-123' -d '{"title":"Once"}'
curl -s -X POST localhost:3000/tasks -H 'Content-Type: application/json' \
  -H 'Idempotency-Key: abc-123' -d '{"title":"Once"}'   # identical response, no new task
```

---

## Summary / Cheat Sheet

**REST in one breath:** an architectural style — stateless, addressable resources, uniform interface, exchange representations.

**Methods → CRUD → status:**

| Method | CRUD | Safe | Idempotent | Typical success |
| --- | --- | --- | --- | --- |
| GET | Read | Yes | Yes | 200 |
| POST | Create | No | No | 201 (+ `Location`) |
| PUT | Replace | No | Yes | 200 / 204 |
| PATCH | Partial update | No | Maybe | 200 / 204 |
| DELETE | Delete | No | Yes | 204 |

**Status code quick map:** 200 OK · 201 Created · 204 No Content · 301/308 perm redirect · 304 Not Modified · 400 Bad Request · 401 Unauthenticated · 403 Forbidden · 404 Not Found · 405 Method Not Allowed · 409 Conflict · 422 Validation failed · 429 Too Many Requests · 500 Internal · 503 Unavailable.

**URL rules:** plural nouns · no verbs · lowercase + hyphens · nest one level deep · filters/sort/pagination in query string.

**Response envelope:** success → `{ "data": ... , "meta": {...} }`; error → `{ "error": { "code", "message", "details" } }`.

**Pagination:** `?page&limit` (cap the limit!) → return `page, limit, total, totalPages` in `meta`. Cursor pagination for big/fast-changing data.

**Errors:** one typed `ApiError` + one 4-arg error middleware registered last; log internals, never leak stack traces; Express 4 needs an async wrapper, Express 5 doesn't.

**Versioning:** prefer URI path (`/v1`); never break a released version.

**Idempotency:** GET/PUT/DELETE are idempotent → safe to retry; protect POST with `Idempotency-Key`.

**HATEOAS:** hypermedia links = Richardson Level 3; mostly skipped in practice; most APIs are Level 2.
