# Express Routing In Depth

Routing is the heart of every Express application. When a request arrives, Express's single most important job is to look at the request's **HTTP method** (`GET`, `POST`, ...) and its **URL path** (`/users/42`), and then decide *which piece of your code should handle it*. That decision-making machinery is **routing**.

If you come from the frontend, you've likely used a *client-side* router (React Router, Vue Router) that swaps components based on the URL in the browser. Server-side routing is conceptually similar but operates on incoming HTTP requests: it maps `method + path` to a **handler function** (and any middleware that runs along the way). Master routing and you've mastered the skeleton on which the entire rest of an Express app hangs — middleware, controllers, validation, auth, and error handling all attach to routes.

This module takes you from `app.get('/', ...)` all the way to a professionally organized, modular routing layer using `express.Router()`, including the often-confusing path-matching rules and the breaking changes introduced in **Express 5**.

> This module assumes you've already met Express and the request/response lifecycle in earlier modules. Here we go deep on routing specifically.

## What You'll Learn

By the end of this module you will be able to:

- Define routes using every **HTTP method handler**: `get`, `post`, `put`, `patch`, `delete`, and the catch-all `all`.
- Write **route paths** as plain strings, string patterns, and regular expressions — and know how these changed in **Express 5** (the `path-to-regexp@8` rewrite).
- Capture **route parameters** with `req.params`, handle **multiple params**, and implement **optional params** correctly in both Express 4 and 5.
- Read **query strings** from `req.query` and understand how Express parses arrays and nested objects.
- Attach **multiple/chained handlers** to a single route and group routes for one path with **`app.route()`**.
- Build **modular, mountable route files** with **`express.Router()`**.
- Apply **router-level middleware** and **mount routers under a path prefix**.
- Reason precisely about **route matching order** and why order is load-bearing.
- Implement robust **404 handling** and **catch-all routes** in both Express 4 and 5.
- Organize routing in a **real project** following industry conventions (routes → controllers → services).

---

## 1. The Mental Model: What a Route Actually Is

An Express **route** is a binding of three things:

1. An **HTTP method** (verb): `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, etc.
2. A **path pattern**: `/`, `/users`, `/users/:id`, a regex, etc.
3. One or more **handler functions** (middleware/controllers) that run when the request matches.

```js
// app.METHOD(PATH, HANDLER)
//      ^       ^      ^
//      |       |      └── function(req, res, next) { ... }
//      |       └───────── the URL path pattern to match
//      └───────────────── the HTTP method, lowercased
app.get('/users', (req, res) => {
  res.send('List of users');
});
```

When a request arrives, Express walks its internal stack of routes/middleware **in the order they were registered**, and runs the first handler whose method **and** path both match. That ordering rule is the single most important thing to internalize about routing — we'll return to it repeatedly.

> TIP: A route handler is just a middleware function with the signature `(req, res, next)`. The only difference between "middleware" and a "route handler" is that a route handler is the one that usually ends the request by sending a response. Everything you know about middleware applies to route handlers and vice versa.

### Minimal runnable app

Every example in this module runs against this skeleton. Save it as `app.js`, run `npm init -y && npm install express`, then `node app.js`.

```js
// app.js
const express = require('express'); // CommonJS — the default in this course
const app = express();

app.use(express.json()); // parse JSON request bodies into req.body

app.get('/', (req, res) => {
  res.send('Server is up');
});

const PORT = 3000;
app.listen(PORT, () => {
  console.log(`Listening on http://localhost:${PORT}`);
});
```

> TIP: To test routes without a browser, use `curl`. `GET` is the default: `curl http://localhost:3000/`. For other methods: `curl -X POST http://localhost:3000/users -H "Content-Type: application/json" -d '{"name":"Ada"}'`.

---

## 2. HTTP Method Handlers: `get`, `post`, `put`, `patch`, `delete`, `all`

Express exposes a routing method for (nearly) every HTTP verb. The method name is the **lowercased** HTTP verb. The standard CRUD-relevant ones:

| Express method | HTTP verb | Conventional meaning (REST) | Idempotent? | Has request body? |
| --- | --- | --- | --- | --- |
| `app.get()` | `GET` | Read a resource / collection | Yes | No (by convention) |
| `app.post()` | `POST` | Create a new resource | No | Yes |
| `app.put()` | `PUT` | Replace a resource entirely | Yes | Yes |
| `app.patch()` | `PATCH` | Update a resource partially | No* | Yes |
| `app.delete()` | `DELETE` | Remove a resource | Yes | Sometimes |
| `app.all()` | *any verb* | Match all methods at this path | — | — |

\* `PATCH` is not guaranteed idempotent by the spec; it depends on your implementation.

> The verbs are **conventions**, not enforcement. Express will happily let you mutate the database in a `GET` handler — but doing so violates HTTP semantics, breaks caching/proxies, and will get you dinged in code review and interviews. Follow the conventions.

### A full CRUD example for one resource

```js
// In-memory "database" for demonstration
let users = [
  { id: 1, name: 'Ada' },
  { id: 2, name: 'Linus' },
];
let nextId = 3;

// READ collection
app.get('/users', (req, res) => {
  res.json(users);
});

// READ one
app.get('/users/:id', (req, res) => {
  const user = users.find((u) => u.id === Number(req.params.id));
  if (!user) return res.status(404).json({ error: 'User not found' });
  res.json(user);
});

// CREATE
app.post('/users', (req, res) => {
  const user = { id: nextId++, name: req.body.name };
  users.push(user);
  res.status(201).json(user); // 201 Created
});

// REPLACE (full update)
app.put('/users/:id', (req, res) => {
  const idx = users.findIndex((u) => u.id === Number(req.params.id));
  if (idx === -1) return res.status(404).json({ error: 'User not found' });
  users[idx] = { id: users[idx].id, name: req.body.name }; // replace fields
  res.json(users[idx]);
});

// PARTIAL update
app.patch('/users/:id', (req, res) => {
  const user = users.find((u) => u.id === Number(req.params.id));
  if (!user) return res.status(404).json({ error: 'User not found' });
  if (req.body.name !== undefined) user.name = req.body.name; // only provided fields
  res.json(user);
});

// DELETE
app.delete('/users/:id', (req, res) => {
  const idx = users.findIndex((u) => u.id === Number(req.params.id));
  if (idx === -1) return res.status(404).json({ error: 'User not found' });
  const [removed] = users.splice(idx, 1);
  res.json(removed); // or res.status(204).end() to send "No Content"
});
```

### `app.all()` — matching every method

`app.all()` runs for a path **regardless of the HTTP method**. It's most useful for cross-cutting logic that should apply to every verb on a path — logging, auth checks, or loading a shared resource.

```js
// Runs for GET, POST, PUT, DELETE... on /secret, before the verb-specific handler
app.all('/secret', (req, res, next) => {
  console.log(`Accessing /secret via ${req.method}`);
  if (!req.headers.authorization) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  next(); // pass control to the next matching handler
});

app.get('/secret', (req, res) => res.send('The treasure is X'));
app.post('/secret', (req, res) => res.send('Treasure updated'));
```

### `HEAD` and `OPTIONS` are handled for you

You rarely write `app.head()` or `app.options()` handlers. Express **auto-responds to `HEAD`** by running your matching `GET` handler and stripping the body, and it can auto-respond to `OPTIONS` with an `Allow` header listing the methods defined for that path. You *can* override them when you need custom behavior (e.g., CORS preflight).

> The complete list of supported methods comes from the `methods` npm package that Express depends on. There's a method for every standard HTTP verb plus some WebDAV ones — but `get`/`post`/`put`/`patch`/`delete`/`all` are 99% of real-world use.

---

## 3. Route Paths: Strings, String Patterns, and Regex

The path argument controls **what matches**. Express supports three flavors, and this is one of the biggest areas where **Express 4 and Express 5 differ**, so read carefully.

### 3.1 Plain string paths

The most common form. The path matches **exactly** (after Express normalizes the query string and, by default, ignores trailing-slash/case differences per its settings).

```js
app.get('/about', handler);          // matches GET /about
app.get('/users/profile', handler);  // matches GET /users/profile
```

By default routing is **case-insensitive** and **non-strict** about trailing slashes (`/about` and `/About` and `/about/` all match). You can change this with app settings:

```js
app.set('case sensitive routing', true); // /About !== /about
app.set('strict routing', true);         // /about !== /about/
```

### 3.2 String *patterns* (the big Express 4 vs 5 change)

Express paths are compiled by a library called **`path-to-regexp`**. In **Express 4** it used an older version that supported a glob-like mini-language directly in string paths:

```js
// ---- EXPRESS 4 ONLY ----
app.get('/ab?cd', handler);   // 'b' is optional → matches /acd and /abcd
app.get('/ab+cd', handler);   // one or more 'b'  → /abcd, /abbcd, /abbbcd
app.get('/ab*cd', handler);   // '*' = anything   → /abXYZcd, /abcd
app.get('/ab(cd)?e', handler); // optional group   → /abe and /abcde
```

**Express 5 upgraded to `path-to-regexp@8`, which removed most of this.** The characters `?`, `+`, `*`, and `()` are **no longer special inside string paths** in Express 5. If you write them, you'll either get a literal match or a thrown error. This was a deliberate change to remove ambiguous, hard-to-read patterns and to close [ReDoS](https://en.wikipedia.org/wiki/ReDoS) security holes.

Here's the practical migration table:

| Goal | Express 4 string pattern | Express 5 approach |
| --- | --- | --- |
| Optional segment | `'/users/:id?'` | `'/users{/:id}'` (braces = optional group) |
| Wildcard "match the rest" | `'/files/*'` | `'/files/*splat'` (**named** wildcard required) |
| Optional char / repetition | `'/ab?cd'`, `'/ab+cd'` | Use a **regex** route instead |
| Named param | `'/users/:id'` | `'/users/:id'` (unchanged) |

**New Express 5 path syntax to know:**

```js
// ---- EXPRESS 5 ----

// Optional group with braces {}
app.get('/users{/:id}', handler); // matches /users AND /users/42

// Named wildcard — captures the remainder into req.params.splat (an array)
app.get('/files/*splat', (req, res) => {
  // GET /files/img/logo.png  →  req.params.splat === ['img/logo.png']
  res.send(req.params.splat);
});

// Wildcard for a catch-all 404 (note: bare '*' is NOT allowed in v5)
app.use('/{*any}', (req, res) => res.status(404).send('Not found'));
```

> WARNING — Pitfall: The single most common Express 4→5 break is `app.get('*', ...)` or `app.use('*', ...)` for catch-all 404s. In Express 5 a **bare `*` throws an error** at startup (`TypeError: Missing parameter name`). You must name it: use `'/{*splat}'` or, cleaner, just use `app.use((req, res) => ...)` with **no path** to catch everything. See Section 11.

### 3.3 Regular expression paths

For full control, pass a **RegExp object** instead of a string. This works in both Express 4 and 5 and is the escape hatch when string patterns aren't expressive enough.

```js
// Matches any path that ENDS with 'fly': /butterfly, /dragonfly, /fly
app.get(/fly$/, (req, res) => res.send('Buzz'));

// Matches /user followed by digits only: /user1, /user42 — but NOT /userABC
app.get(/^\/user\d+$/, (req, res) => res.send(`User route: ${req.path}`));
```

You can **capture** with regex groups, which land in `req.params` keyed by **numeric index** (`0`, `1`, ...) rather than by name:

```js
// Capture the numeric id from a regex group
app.get(/^\/products\/(\d+)$/, (req, res) => {
  res.json({ productId: req.params[0] }); // group 1 → req.params[0]
});
```

> WARNING — Pitfall: A regex path matches **anywhere in the string** unless you anchor it with `^` and `$`. `/fly$/` correctly requires the path to end in "fly", but a sloppy `/user/` (no anchors) would match `/superuser/account` too. Anchor your regexes.

> BEST PRACTICE: Reach for regex routes sparingly. They're powerful but harder to read and can introduce catastrophic-backtracking (ReDoS) vulnerabilities if written carelessly. Prefer named params and `{}` optional groups; use regex only when you genuinely need pattern matching a segment can't express.

---

## 4. Route Parameters: `req.params`

**Route parameters** are named URL segments that capture dynamic values. You declare them with a colon prefix (`:name`); Express parses the matched value into `req.params.name` as a **string**.

```js
// :id is a route parameter
app.get('/users/:id', (req, res) => {
  // GET /users/42  →  req.params = { id: '42' }
  res.send(`Fetching user ${req.params.id}`);
});
```

> WARNING — Pitfall: `req.params` values are **always strings**. `req.params.id` for `/users/42` is the string `'42'`, not the number `42`. Convert explicitly (`Number(req.params.id)` or `parseInt(req.params.id, 10)`) before doing arithmetic or strict-equality comparisons against numeric DB ids.

### 4.1 Multiple parameters

A path can contain several params. They each become keys on `req.params`.

```js
app.get('/users/:userId/posts/:postId', (req, res) => {
  // GET /users/3/posts/9  →  req.params = { userId: '3', postId: '9' }
  const { userId, postId } = req.params;
  res.send(`Post ${postId} by user ${userId}`);
});
```

Params can also share a single segment, separated by a literal character (useful for things like dates or ranges):

```js
// Match /range/10-25  →  req.params = { from: '10', to: '25' }
app.get('/range/:from-:to', (req, res) => {
  res.json({ from: req.params.from, to: req.params.to });
});

// Match a semver-ish path /v/1.2.3  →  { major:'1', minor:'2', patch:'3' }
app.get('/v/:major.:minor.:patch', (req, res) => res.json(req.params));
```

### 4.2 Optional parameters

This is another **Express 4 vs 5** divergence.

```js
// ---- EXPRESS 4 ----
// Trailing '?' makes the param optional
app.get('/posts/:year/:month?', (req, res) => {
  // /posts/2024        → { year:'2024', month: undefined }
  // /posts/2024/06     → { year:'2024', month:'06' }
  res.json(req.params);
});
```

```js
// ---- EXPRESS 5 ----
// '?' on params is gone. Wrap the optional part in braces {}
app.get('/posts/:year{/:month}', (req, res) => {
  // /posts/2024        → { year:'2024' }                (no month key at all)
  // /posts/2024/06     → { year:'2024', month:'06' }
  res.json(req.params);
});
```

Note the subtle behavioral difference: in Express 5 the optional `month` key is simply **absent** from `req.params` when not supplied (rather than `undefined`). Always guard with a default: `const month = req.params.month ?? 'all';`.

### 4.3 Param constraints with regex

You can constrain what a param matches by appending a regex in parentheses (works in both versions, though the inner syntax is stricter in v5). This lets two routes with the same shape resolve differently.

```js
// :id must be one-or-more digits, else this route doesn't match
app.get('/users/:id(\\d+)', (req, res) => {
  res.send(`Numeric user id: ${req.params.id}`);
});

// A different route for non-numeric "usernames"
app.get('/users/:username', (req, res) => {
  res.send(`Username lookup: ${req.params.username}`);
});
// Order matters: the constrained numeric route must be declared FIRST (see §10).
```

### 4.4 `app.param()` — preload logic for a parameter

`app.param(name, fn)` registers a callback that runs **whenever that param appears** in a matched route, *before* the route handler. It's the idiomatic place to load-and-validate a resource once instead of repeating `findById` in every handler.

```js
// Runs for ANY route that has a :id param, before its handler
app.param('id', (req, res, next, id) => {
  const user = users.find((u) => u.id === Number(id));
  if (!user) return res.status(404).json({ error: 'User not found' });
  req.user = user; // attach for downstream handlers
  next();
});

// Now handlers can assume req.user exists and is valid
app.get('/users/:id', (req, res) => res.json(req.user));
app.put('/users/:id', (req, res) => {
  req.user.name = req.body.name;
  res.json(req.user);
});
```

> BEST PRACTICE: `app.param()` (or its router-level equivalent `router.param()`) is the cleanest way to eliminate repetitive "fetch-or-404" boilerplate. It centralizes validation and keeps handlers focused on business logic.

---

## 5. Query Strings: `req.query`

The **query string** is the part of a URL after the `?`: `/search?q=node&page=2`. It's the standard way to pass **optional, non-hierarchical** data — filters, pagination, sorting, search terms. Unlike route params, query params are **not** part of the route path and do **not** affect which route matches.

```js
app.get('/search', (req, res) => {
  // GET /search?q=express&page=2&limit=10
  // req.query = { q: 'express', page: '2', limit: '10' }
  const { q, page = '1', limit = '10' } = req.query;
  res.json({ q, page: Number(page), limit: Number(limit) });
});
```

> WARNING — Pitfall: Just like `req.params`, **every `req.query` value is a string** (or an array/object of strings). `req.query.page` is `'2'`, never `2`. Convert before doing math or comparisons.

### Arrays and nested objects in queries

Express parses query strings into structured objects. The exact behavior depends on the **query parser** setting:

```js
// ?tags=a&tags=b           → { tags: ['a', 'b'] }         (repeated key → array)
// ?filter[role]=admin      → { filter: { role: 'admin' } } (bracket → nested object, "extended" parser)
app.get('/items', (req, res) => res.json(req.query));
```

By default Express uses the `extended` (`qs` library) parser, which understands bracket notation for nested objects and arrays. You can switch parsers:

```js
app.set('query parser', 'simple');   // uses Node's built-in querystring (no nesting)
app.set('query parser', 'extended'); // default in Express 4; richer parsing via qs
// In Express 5 you can also pass a custom function: app.set('query parser', fn)
```

> WARNING — Pitfall: In **Express 5**, `req.query` is a **getter** that returns a fresh parsed object; you cannot reassign or mutate it the way you might have in v4. If you need to normalize query values, copy them into your own variable rather than writing to `req.query`.

### Route param vs query param — when to use which

| Use a **route param** (`/users/:id`) when... | Use a **query param** (`/users?role=admin`) when... |
| --- | --- |
| The value **identifies a specific resource** | The value **filters, sorts, or paginates** a collection |
| It's **required** for the route to make sense | It's **optional** |
| It's hierarchical / part of the resource's identity | It's a modifier on the request |
| Example: `GET /orders/789` | Example: `GET /orders?status=shipped&sort=-date` |

---

## 6. Multiple & Chained Handlers

A single route can take **more than one handler function**. Express runs them in order; each must either send a response or call `next()` to pass control to the next handler in the chain. This is how you compose middleware (validation, auth) with the final controller.

```js
// Pass handlers as separate arguments
function validateUserBody(req, res, next) {
  if (!req.body.name) {
    return res.status(400).json({ error: 'name is required' });
  }
  next(); // hand off to the next function
}

function createUser(req, res) {
  const user = { id: nextId++, name: req.body.name };
  users.push(user);
  res.status(201).json(user);
}

// Two handlers: validation runs first, then the controller
app.post('/users', validateUserBody, createUser);
```

You can also pass handlers as an **array**, or mix arrays and individual functions — handy for reusing a stack of middleware:

```js
const auth = (req, res, next) => {
  if (!req.headers.authorization) return res.status(401).end();
  next();
};
const audit = (req, res, next) => {
  console.log(`${req.method} ${req.originalUrl}`);
  next();
};

// Array of middleware, then the final handler
app.delete('/users/:id', [auth, audit], (req, res) => {
  // ... delete logic
  res.status(204).end();
});
```

> WARNING — Pitfall: Every non-terminal handler **must call `next()`** (or send a response). Forget it and the request **hangs forever** — the client waits, eventually times out, and nothing in your logs explains why. If you call `next()` *and* also send a response in the same handler, you risk an `ERR_HTTP_HEADERS_SENT` error. Pick one per handler.

> TIP: `next('route')` is a special signal that **skips the remaining handlers of the current route** and moves on to the next matching route. `next(err)` (passing any argument other than the string `'route'`) jumps straight to error-handling middleware. These are powerful flow-control tools.

### Grouping with `app.route()`

When several methods share the same path, repeating the path string is noisy and error-prone (rename it in one place, forget another). **`app.route(path)`** returns a chainable route object so you declare the path once:

```js
app
  .route('/books')
  .get((req, res) => res.send('List books'))
  .post((req, res) => res.send('Create a book'));

app
  .route('/books/:id')
  .get((req, res) => res.send(`Get book ${req.params.id}`))
  .put((req, res) => res.send(`Replace book ${req.params.id}`))
  .patch((req, res) => res.send(`Update book ${req.params.id}`))
  .delete((req, res) => res.send(`Delete book ${req.params.id}`));
```

> BEST PRACTICE: Use `app.route()` (or `router.route()`) whenever a single path supports multiple verbs. It keeps the path DRY and visually groups everything that can happen at that URL.

---

## 7. `express.Router()`: Modular, Mountable Route Files

As an app grows, defining every route on the top-level `app` object turns one file into a thousand-line monster. **`express.Router()`** creates a mini-application — an isolated, "mini" instance of routing and middleware — that you can define in its own file and **mount** onto the main app. Think of a `Router` as a *complete middleware and routing system in miniature*; you can `.get()`, `.post()`, `.use()`, `.param()`, and `.route()` on it exactly like on `app`.

### A modular route file

```js
// routes/users.js
const express = require('express');
const router = express.Router(); // create a router instance

// Paths here are RELATIVE to wherever the router is mounted
router.get('/', (req, res) => res.send('All users'));         // → GET  /users
router.post('/', (req, res) => res.send('Create user'));      // → POST /users
router.get('/:id', (req, res) => res.send(`User ${req.params.id}`)); // → GET /users/:id

module.exports = router; // export the router so app.js can mount it
```

```js
// app.js
const express = require('express');
const app = express();
app.use(express.json());

const usersRouter = require('./routes/users'); // import the router

// Mount the router under the /users prefix
app.use('/users', usersRouter);

app.listen(3000);
```

Notice the **mount path** (`/users`) is specified **once**, where the router is attached. Inside `routes/users.js`, every path is relative to that prefix. This is the key to scalable routing: each resource gets its own file, and the prefix lives in exactly one place.

### ES Modules variant

The same thing with `import`/`export` (set `"type": "module"` in `package.json`):

```js
// routes/users.mjs
import { Router } from 'express';
const router = Router();
router.get('/', (req, res) => res.send('All users'));
export default router;
```

```js
// app.mjs
import express from 'express';
import usersRouter from './routes/users.mjs';
const app = express();
app.use('/users', usersRouter);
app.listen(3000);
```

> TIP: A `Router` is itself a valid middleware function, which is *why* `app.use('/users', usersRouter)` works — you're handing `app.use` a middleware that happens to contain its own routing table.

---

## 8. Router-Level Middleware & Mounting with a Prefix

Because a router is a mini-app, it has its **own middleware stack**. Middleware added with `router.use(...)` runs **only for requests that match the router's mount path** — a clean way to scope behavior (auth, logging, rate limiting) to one section of your API.

```js
// routes/admin.js
const express = require('express');
const router = express.Router();

// Router-level middleware: runs for EVERY request into this router
router.use((req, res, next) => {
  console.log(`[admin] ${req.method} ${req.originalUrl}`);
  next();
});

// Scoped auth — only admin routes require it
router.use((req, res, next) => {
  if (req.headers['x-admin-token'] !== 'secret') {
    return res.status(403).json({ error: 'Admins only' });
  }
  next();
});

router.get('/dashboard', (req, res) => res.send('Admin dashboard'));
router.get('/users', (req, res) => res.send('Manage users'));

module.exports = router;
```

```js
// app.js — mount under /admin; the prefix applies to every route inside
app.use('/admin', require('./routes/admin'));
// GET /admin/dashboard → passes through both router middlewares, then the handler
```

### Mount-path nuances

- The **mount prefix is stripped** from `req.url` inside the router but preserved in `req.originalUrl`. Inside `routes/admin.js`, a request to `/admin/dashboard` sees `req.url === '/dashboard'` but `req.originalUrl === '/admin/dashboard'` and `req.baseUrl === '/admin'`.
- You can mount the **same router under multiple prefixes**, or nest routers (mount a router inside another router) to build deep hierarchies like `/api/v1/users`.

```js
// Nesting routers to compose an API version prefix
const apiV1 = express.Router();
apiV1.use('/users', require('./routes/users'));
apiV1.use('/admin', require('./routes/admin'));
app.use('/api/v1', apiV1); // → /api/v1/users, /api/v1/admin/dashboard, ...
```

> WARNING — Pitfall: To capture a param **from the mount path** inside a nested router, you must create the router with `mergeParams`. By default a child router can't see params defined on its parent's mount path.

```js
// routes/comments.js — mounted at /posts/:postId/comments
const router = express.Router({ mergeParams: true }); // <-- crucial

router.get('/', (req, res) => {
  // Without mergeParams, req.params.postId would be undefined here
  res.send(`Comments for post ${req.params.postId}`);
});
module.exports = router;
```

```js
// In routes/posts.js
const commentsRouter = require('./comments');
postsRouter.use('/:postId/comments', commentsRouter); // → /posts/:postId/comments
```

> BEST PRACTICE: `mergeParams: true` is essential for **nested/REST-nested resources** (`/posts/:postId/comments/:commentId`). Forgetting it is one of the most common reasons "my param is undefined" in nested routers.

---

## 9. Application-Level vs Router-Level vs Route-Specific Middleware

It helps to see the three scopes side by side, because *where* you attach middleware determines *when* it runs.

| Scope | How to attach | Runs for... |
| --- | --- | --- |
| **Application-level** | `app.use(fn)` / `app.use('/path', fn)` | Every request (or every request under `/path`) |
| **Router-level** | `router.use(fn)` | Every request that enters that mounted router |
| **Route-specific** | `app.get('/x', fn1, fn2)` | Only that exact route + method |

```js
app.use(express.json());            // application-level: every request
app.use('/api', apiRouter);         // application-level, path-scoped
apiRouter.use(rateLimit());         // router-level: every /api/* request
apiRouter.get('/me', auth, getMe);  // route-specific: only GET /api/me
```

---

## 10. Route Matching Order & Why It Matters

Express matches routes **top to bottom, in registration order**, and runs the **first** route whose method and path both match. It does **not** look for the "best" or "most specific" match — it takes the **first** match. This single rule causes a huge fraction of real-world routing bugs, so let's nail it.

### The classic "shadowed route" bug

```js
// ❌ WRONG ORDER
app.get('/users/:id', (req, res) => res.send(`User ${req.params.id}`));
app.get('/users/new', (req, res) => res.send('New-user form'));

// Request: GET /users/new
// → '/users/:id' is registered first and ':id' happily matches the literal "new".
// → You get "User new". The '/users/new' route is DEAD CODE — never reached.
```

The fix: register **more specific / literal routes before dynamic ones**.

```js
// ✅ CORRECT ORDER — specific first, dynamic last
app.get('/users/new', (req, res) => res.send('New-user form'));
app.get('/users/:id', (req, res) => res.send(`User ${req.params.id}`));
```

The general principles:

1. **Static (literal) paths before dynamic (`:param`) paths** that could match them.
2. **Constrained params before unconstrained** (`/:id(\d+)` before `/:username`).
3. **Catch-all / 404 handlers last** — they match everything, so anything after them is unreachable.
4. **Middleware order is part of this too** — `app.use(express.json())` must come *before* any route that reads `req.body`.

> WARNING — Pitfall: This is the #1 source of "my new route isn't being hit" confusion. If a route seems ignored, scan upward for an earlier route (often a `:param` or a catch-all) that's swallowing the request. Express stops at the first match.

### Middleware-vs-route ordering

```js
const logger = (req, res, next) => { console.log(req.path); next(); };

app.get('/a', (req, res) => res.send('A')); // declared first
app.use(logger);                            // declared AFTER /a
// GET /a → handler responds; logger NEVER runs, because /a ended the request first.
```

> BEST PRACTICE: Put global middleware (body parsers, loggers, security headers, CORS) **at the very top**, then routes ordered from most-specific to least-specific, then your **404 catch-all and error handler last**.

---

## 11. 404 Handling & Catch-All Routes

A **404** ("Not Found") happens when **no route matched** the request. Express's default behavior is a plain-text "Cannot GET /whatever" with status 404 — fine for prototypes, unacceptable for real APIs. You provide your own 404 handler by adding a final middleware that runs only when nothing above it matched.

### The portable, version-proof 404 (recommended)

The cleanest catch-all works identically in Express 4 and 5: a **path-less middleware at the bottom** of the stack. Because nothing terminated the request before reaching it, it must be a 404.

```js
// ... all your real routes above ...

// 404 handler — path-less middleware as the LAST non-error middleware
app.use((req, res) => {
  res.status(404).json({
    error: 'Not Found',
    method: req.method,
    path: req.originalUrl,
  });
});

// Error-handling middleware — note the 4 args; goes absolutely last
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(err.status || 500).json({ error: err.message || 'Server Error' });
});
```

> WARNING — Pitfall: The 404 middleware **must be registered after all real routes**. If you put it earlier, it will swallow requests that a later route would have handled. And the **error handler (4 args) must come after the 404**, because Express distinguishes error middleware purely by its **four-parameter signature** `(err, req, res, next)`.

### Path-based catch-all (and the Express 5 gotcha)

If you prefer an explicit wildcard path (e.g., to scope a 404 to one API prefix), the syntax differs by version:

```js
// ---- EXPRESS 4 ----
app.all('*', (req, res) => res.status(404).json({ error: 'Not Found' }));
```

```js
// ---- EXPRESS 5 ----
// Bare '*' THROWS at startup. Use a NAMED wildcard:
app.all('/{*splat}', (req, res) => res.status(404).json({ error: 'Not Found' }));
// or scope it: app.all('/api/{*splat}', notFoundHandler);
```

> WARNING — Pitfall (Express 5): `app.all('*', ...)`, `app.get('*', ...)`, and `app.use('*', ...)` all throw `TypeError: Missing parameter name` in Express 5 because `*` must now be named. The migration is to use `'/{*splat}'` (the `splat` name is arbitrary) — or sidestep the issue entirely with the path-less middleware shown above, which is what most teams do.

---

## 12. Best Practices for Organizing Routes in a Real Project

A maintainable Express codebase separates **routing** (URL → which function) from **controllers** (request/response handling) from **services** (business logic). Routes should be thin: they wire URLs to controllers and apply middleware — nothing more.

### Recommended folder structure

```bash
src/
├── app.js                 # builds the Express app, mounts routers, error handlers
├── server.js              # imports app, calls app.listen() (separation of concerns)
├── routes/
│   ├── index.js           # aggregates all routers under /api/v1
│   ├── users.routes.js
│   ├── posts.routes.js
│   └── auth.routes.js
├── controllers/
│   ├── users.controller.js
│   └── posts.controller.js
├── services/
│   └── users.service.js
├── middleware/
│   ├── auth.js
│   ├── validate.js
│   └── errorHandler.js
└── models/
    └── user.model.js
```

### An aggregator router

Centralize where each resource mounts so `app.js` stays tiny and the versioned prefix lives in one place.

```js
// src/routes/index.js
const express = require('express');
const router = express.Router();

router.use('/users', require('./users.routes'));
router.use('/posts', require('./posts.routes'));
router.use('/auth', require('./auth.routes'));

module.exports = router;
```

```js
// src/app.js
const express = require('express');
const app = express();

app.use(express.json());
app.use('/api/v1', require('./routes')); // every resource lives under /api/v1

// 404 + error handler last
app.use((req, res) => res.status(404).json({ error: 'Not Found' }));
app.use(require('./middleware/errorHandler'));

module.exports = app;
```

### Thin routes delegating to controllers

```js
// src/routes/users.routes.js
const express = require('express');
const router = express.Router();
const ctrl = require('../controllers/users.controller');
const auth = require('../middleware/auth');

router.route('/')
  .get(ctrl.list)
  .post(auth, ctrl.create);

router.route('/:id')
  .get(ctrl.getOne)
  .patch(auth, ctrl.update)
  .delete(auth, ctrl.remove);

module.exports = router;
```

```js
// src/controllers/users.controller.js
const service = require('../services/users.service');

// Wrap async controllers so thrown errors reach the error handler (Express 4)
exports.list = async (req, res, next) => {
  try {
    res.json(await service.findAll(req.query));
  } catch (err) {
    next(err); // forward to error-handling middleware
  }
};
```

> TIP — Express 5 async errors: In **Express 5**, a rejected Promise from an `async` route handler is **automatically forwarded to your error handler** — you no longer strictly need the `try/catch + next(err)` dance (or an `express-async-errors`-style wrapper). In **Express 4**, an unhandled rejection in an async handler is *not* caught and will hang the request, so you must `try/catch` and call `next(err)` yourself.

> BEST PRACTICE: Keep handlers (controllers) out of the route files. The route file should read like a table of contents: "this URL + method → this controller, guarded by this middleware." That separation makes routes scannable, controllers unit-testable, and services reusable across transports (HTTP, CLI, queue workers).

---

## Common Pitfalls & Gotchas

- **Forgetting `next()`** in a non-terminal handler → the request hangs indefinitely with no error.
- **Calling `next()` *and* sending a response** in the same handler → `ERR_HTTP_HEADERS_SENT` ("Cannot set headers after they are sent").
- **Treating `req.params`/`req.query` as numbers** → they're always strings; `'42' === 42` is `false`. Convert with `Number()`.
- **Route order shadowing** → a dynamic `/users/:id` declared before the literal `/users/new` makes the literal route unreachable. Specific before dynamic.
- **404 handler placed too early** → it swallows requests meant for later routes. It must be last (before the error handler).
- **Express 5 bare `*`** → `app.get('*', ...)` / `app.use('*', ...)` throws at startup. Use `'/{*splat}'` or a path-less middleware.
- **Express 5 optional params** → `:id?` no longer works; use `{/:id}`. The key is *absent* (not `undefined`) when omitted.
- **Nested router params undefined** → forgot `express.Router({ mergeParams: true })`.
- **Reading `req.body` without a body parser** → `req.body` is `undefined`. Register `express.json()` / `express.urlencoded()` *before* the routes that need it.
- **Mutating `req.query` in Express 5** → it's a getter; copy values out instead of reassigning.
- **Unanchored regex routes** → `/user/` (no `^`/`$`) matches more than you think. Anchor them.
- **Async errors in Express 4** → unhandled rejections in async handlers hang silently; wrap with `try/catch` + `next(err)`.

## Best Practices

- **Order matters**: global middleware → specific routes → dynamic routes → 404 → error handler.
- **One router per resource**, each in its own file; mount with a single prefix in an aggregator (`routes/index.js`).
- **Version your API** via a mount prefix (`/api/v1`) so you can evolve without breaking clients.
- **Keep route files thin** — wire URLs to controllers; put logic in controllers/services.
- **Use `app.route()` / `router.route()`** to group multiple verbs sharing a path.
- **Use `router.param()` / `app.param()`** to centralize "fetch-or-404" resource loading.
- **Validate input** with dedicated middleware before the controller runs.
- **Always provide a custom 404 and a centralized error handler.**
- **Convert and validate** `req.params`/`req.query` values; never trust them as the right type.
- **Prefer named params and `{}` optional groups** over regex; reserve regex for genuine pattern needs and always anchor it.
- **Know your Express version** — v4 vs v5 path syntax differences will bite you during upgrades.

## Key Interview Points

- **What are the parts of an Express route?** Method + path + one-or-more handler functions; the handler is just middleware that typically ends the response.
- **`req.params` vs `req.query` vs `req.body`?** `params` = named URL segments (route identity); `query` = `?key=value` modifiers (filter/sort/paginate); `body` = parsed request payload (needs a body-parser). All of `params`/`query` are strings.
- **Why does route order matter?** Express runs the **first** matching route in registration order — not the most specific. A dynamic route declared before a literal one shadows it; the 404 catch-all must be last.
- **How do you make routes modular?** `express.Router()` creates a mini-app you define in its own file and mount with `app.use('/prefix', router)`; the prefix is specified once at mount time.
- **What is `mergeParams`?** A `Router` option that lets a nested router access params from its parent's mount path (needed for `/posts/:postId/comments`).
- **How do you do a 404 in Express?** Add a path-less middleware after all routes (works in v4 and v5); follow it with a 4-arg error handler. In v4 you could also use `app.all('*', ...)`.
- **What changed in Express 5 routing?** `path-to-regexp@8`: removed `?`/`+`/`*`/`()` in string paths; bare `*` is illegal (must be named, e.g. `*splat`); optional params use `{}` not `:id?`; `req.query` is a getter; async handler rejections auto-forward to the error handler.
- **Difference between `app.use` and `app.get`?** `app.use` mounts middleware that matches the path **prefix** for **any** method; `app.get` matches the path **exactly** for the `GET` method (and is terminal-ish in the matching sense).
- **What does `next('route')` do vs `next(err)`?** `next('route')` skips remaining handlers of the current route to the next matching route; `next(err)` (any other arg) jumps to the error-handling middleware.
- **How are PUT and PATCH different?** `PUT` replaces the entire resource (idempotent); `PATCH` applies a partial update.
- **What's `app.route()` for?** Chaining multiple HTTP-method handlers on a single shared path to keep the path DRY.

## Practice Exercises

Work these against the minimal `app.js` skeleton from Section 1. Try each before peeking at the solutions.

1. **CRUD warm-up.** Build full CRUD for a `/tasks` resource backed by an in-memory array. Each task is `{ id, title, done }`. Return correct status codes (`201` on create, `404` when not found, `204` on delete).

2. **Param typing & validation.** Add a route `GET /tasks/:id` that returns `400` if `:id` is not a positive integer, `404` if no task has that id, and the task otherwise. Convert the param correctly.

3. **Query-driven filtering & pagination.** Implement `GET /tasks?done=true&page=1&limit=2` that filters by `done` (a string `'true'`/`'false'`), then paginates. Return `{ page, limit, total, data }`.

4. **Modularize with a Router.** Move all task routes into `routes/tasks.routes.js` using `express.Router()` and `router.route()`, mount it at `/api/v1/tasks`, and add a router-level logging middleware that prints `METHOD originalUrl` for every task request.

5. **Nested resource with `mergeParams`.** Add comments under a task: `GET /api/v1/tasks/:taskId/comments` and `POST /api/v1/tasks/:taskId/comments`. Use a separate router with `mergeParams: true`. A `404` if the parent task doesn't exist (use `router.param`).

6. **Version-proof 404 + error handler.** Add a catch-all 404 (works in both Express 4 and 5) and a centralized error-handling middleware. Then add one route that deliberately `throw`s and confirm the error handler catches it (note the v4 vs v5 async difference).

### Solutions

**Solution 1 — CRUD warm-up**

```js
let tasks = [{ id: 1, title: 'Learn routing', done: false }];
let nextTaskId = 2;

app.get('/tasks', (req, res) => res.json(tasks));

app.post('/tasks', (req, res) => {
  const task = { id: nextTaskId++, title: req.body.title, done: false };
  tasks.push(task);
  res.status(201).json(task); // 201 Created
});

app.put('/tasks/:id', (req, res) => {
  const idx = tasks.findIndex((t) => t.id === Number(req.params.id));
  if (idx === -1) return res.status(404).json({ error: 'Not found' });
  tasks[idx] = { id: tasks[idx].id, title: req.body.title, done: !!req.body.done };
  res.json(tasks[idx]);
});

app.patch('/tasks/:id', (req, res) => {
  const task = tasks.find((t) => t.id === Number(req.params.id));
  if (!task) return res.status(404).json({ error: 'Not found' });
  if (req.body.title !== undefined) task.title = req.body.title;
  if (req.body.done !== undefined) task.done = !!req.body.done;
  res.json(task);
});

app.delete('/tasks/:id', (req, res) => {
  const idx = tasks.findIndex((t) => t.id === Number(req.params.id));
  if (idx === -1) return res.status(404).json({ error: 'Not found' });
  tasks.splice(idx, 1);
  res.status(204).end(); // 204 No Content — no response body
});
```

**Solution 2 — Param typing & validation**

```js
app.get('/tasks/:id', (req, res) => {
  const raw = req.params.id;
  // Validate: must be a positive integer string like "1", "42"
  if (!/^\d+$/.test(raw)) {
    return res.status(400).json({ error: 'id must be a positive integer' });
  }
  const id = Number(raw); // safe to convert now
  const task = tasks.find((t) => t.id === id);
  if (!task) return res.status(404).json({ error: 'Not found' });
  res.json(task);
});
```

**Solution 3 — Query filtering & pagination**

```js
app.get('/tasks', (req, res) => {
  let result = tasks;

  // Filter by done if provided (query values are strings)
  if (req.query.done !== undefined) {
    const wantDone = req.query.done === 'true';
    result = result.filter((t) => t.done === wantDone);
  }

  // Pagination with sane numeric defaults
  const page = Math.max(1, Number(req.query.page) || 1);
  const limit = Math.max(1, Number(req.query.limit) || 10);
  const start = (page - 1) * limit;
  const data = result.slice(start, start + limit);

  res.json({ page, limit, total: result.length, data });
});
```

**Solution 4 — Modularize with a Router**

```js
// routes/tasks.routes.js
const express = require('express');
const router = express.Router();

// Router-level middleware: logs every request into this router
router.use((req, res, next) => {
  console.log(`${req.method} ${req.originalUrl}`);
  next();
});

let tasks = [{ id: 1, title: 'Learn routing', done: false }];
let nextTaskId = 2;

router.route('/')
  .get((req, res) => res.json(tasks))
  .post((req, res) => {
    const task = { id: nextTaskId++, title: req.body.title, done: false };
    tasks.push(task);
    res.status(201).json(task);
  });

router.route('/:id')
  .get((req, res) => {
    const task = tasks.find((t) => t.id === Number(req.params.id));
    if (!task) return res.status(404).json({ error: 'Not found' });
    res.json(task);
  })
  .delete((req, res) => {
    const idx = tasks.findIndex((t) => t.id === Number(req.params.id));
    if (idx === -1) return res.status(404).json({ error: 'Not found' });
    tasks.splice(idx, 1);
    res.status(204).end();
  });

module.exports = router;
```

```js
// app.js
app.use('/api/v1/tasks', require('./routes/tasks.routes'));
// GET /api/v1/tasks → logs "GET /api/v1/tasks" then returns the list
```

**Solution 5 — Nested resource with `mergeParams`**

```js
// routes/comments.routes.js
const express = require('express');
const router = express.Router({ mergeParams: true }); // see parent's :taskId

// In a real app, share the tasks store via a module; simplified here:
const comments = []; // { id, taskId, body }
let nextCommentId = 1;

// Validate the parent task exists for ANY route with :taskId
router.param('taskId', (req, res, next, taskId) => {
  // Pretend lookup; replace with real store/DB check
  if (!/^\d+$/.test(taskId)) {
    return res.status(404).json({ error: 'Parent task not found' });
  }
  req.taskId = Number(taskId);
  next();
});

router.route('/')
  .get((req, res) => {
    res.json(comments.filter((c) => c.taskId === req.taskId));
  })
  .post((req, res) => {
    const comment = { id: nextCommentId++, taskId: req.taskId, body: req.body.body };
    comments.push(comment);
    res.status(201).json(comment);
  });

module.exports = router;
```

```js
// routes/tasks.routes.js — mount comments under :id
const commentsRouter = require('./comments.routes');
// Note the param name must match what comments.routes.js expects (taskId)
router.use('/:taskId/comments', commentsRouter);
// → GET/POST /api/v1/tasks/:taskId/comments
```

**Solution 6 — 404 + error handler**

```js
// A route that deliberately throws (sync throw IS caught by Express in v4 & v5)
app.get('/boom', (req, res) => {
  throw new Error('Something broke');
});

// For an ASYNC throw:
//  - Express 5: this is auto-forwarded to the error handler.
//  - Express 4: you MUST catch and call next(err), or it hangs.
app.get('/boom-async', async (req, res, next) => {
  try {
    await Promise.reject(new Error('Async failure'));
  } catch (err) {
    next(err); // required in Express 4; harmless/optional in Express 5
  }
});

// 404 — path-less middleware, works in Express 4 AND 5, placed after all routes
app.use((req, res) => {
  res.status(404).json({ error: 'Not Found', path: req.originalUrl });
});

// Centralized error handler — 4 args, placed absolutely last
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(err.status || 500).json({ error: err.message || 'Server Error' });
});
```

## Summary / Cheat Sheet

| Concept | Syntax / Key fact |
| --- | --- |
| Method handlers | `app.get/post/put/patch/delete(path, ...handlers)`; `app.all` matches every verb |
| Route param | `'/users/:id'` → `req.params.id` (**always a string**) |
| Multiple params | `'/users/:uid/posts/:pid'` → `req.params.uid`, `req.params.pid` |
| Optional param (v4) | `'/posts/:year/:month?'` |
| Optional param (v5) | `'/posts/:year{/:month}'` (key absent when omitted) |
| Param constraint | `'/users/:id(\\d+)'` (digits only) |
| Query string | `req.query` (also always strings; `extended` parser handles nesting) |
| Regex route | `app.get(/^\/user\d+$/, ...)`; captures → `req.params[0]`, `[1]`... (anchor with `^`/`$`) |
| Chained handlers | `app.post('/x', mw1, mw2, controller)`; each must `next()` or respond |
| Group by path | `app.route('/books').get(...).post(...)` |
| Modular router | `const r = express.Router()`; `app.use('/users', r)` |
| Router middleware | `router.use(fn)` — scoped to that router's mount path |
| Nested router params | `express.Router({ mergeParams: true })` |
| Match order | First match wins, top-to-bottom; **specific before dynamic**; 404 last |
| 404 (v4 & v5) | Path-less `app.use((req,res)=>res.status(404)...)` after all routes |
| 404 path-based (v5) | `app.all('/{*splat}', ...)` — bare `*` throws in v5 |
| Error handler | `app.use((err, req, res, next) => ...)` — 4 args, absolutely last |
| `next('route')` | Skip remaining handlers of current route → next matching route |
| `next(err)` | Jump straight to error-handling middleware |
| Express 5 path changes | `path-to-regexp@8`: no `?`/`+`/`*`/`()` in strings; named `*splat`; `{}` optional groups; `req.query` is a getter; async rejections auto-forwarded |

> BEST PRACTICE recap: global middleware → specific routes → dynamic routes → 404 → error handler. One router per resource, mounted once under a versioned prefix, with thin route files delegating to controllers.
