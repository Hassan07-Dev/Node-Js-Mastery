# API Best Practices & Documentation

You can build an Express app that returns JSON. You did that weeks ago. But there is a wide gulf between "an endpoint that works on my machine" and "an API that a thousand other developers can build a business on without ever needing to email you." That gulf is filled with *conventions* — decisions about URL shapes, status codes, error formats, pagination, versioning, rate limits, and documentation that, once chosen well and applied consistently, make your API feel obvious. The best APIs in the world (Stripe, GitHub, Twilio) feel almost boring to use. That boringness is the highest compliment a backend engineer can earn: it means the API is so predictable that nobody has to think about it.

This module is the synthesis of everything you have learned about HTTP, Express, and data modeling, elevated to *production professional practice*. We are not learning new syntax so much as learning **judgment**: which of several valid choices is the one that will not embarrass you in six months when a mobile team, a partner integration, and your own future self are all depending on it.

We will cover REST maturity and resource design, then the four hardest-to-get-right details of any list endpoint — versioning, pagination, filtering/sorting/field selection, and consistent response envelopes. We will make your API safe to retry and hard to abuse with rate limiting and idempotency keys. We will then make it *discoverable* and *trustworthy* through OpenAPI/Swagger documentation and Postman collections. Finally we will zoom out: when GraphQL is the better tool, how to send and receive webhooks securely, and a hard-won developer-experience and security checklist you can hold your own work against.

> All examples target **Node.js current LTS (the v20/v22 line)** with **Express** (v4 by default, with v5 differences flagged) and use CommonJS (`require`) unless ES Modules are explicitly relevant. The APIs shown are current and non-deprecated as of this writing.

## What You'll Learn

- **REST maturity (the Richardson Model)** and how to design consistent, predictable resources, URLs, status codes, and methods.
- **API versioning strategies** — URI path, custom header, and media-type (content) versioning — with concrete migration and deprecation tactics.
- **Pagination patterns**: offset/limit vs cursor (keyset) pagination, when each breaks, and how to design response metadata (`meta`, `links`).
- **Filtering, sorting, and sparse fieldsets** done cleanly and safely, including how to avoid NoSQL injection and unbounded queries.
- **Consistent success and error envelopes** — a single shape your whole API speaks, plus a centralized Express error handler that produces it.
- **Rate limiting, throttling, and idempotency keys** so clients can retry safely and abusers are kept out.
- **Documenting your API with OpenAPI 3** using `swagger-jsdoc` + `swagger-ui-express`, including reusable schemas and a servable spec.
- **Exploring and sharing APIs** with Postman and Thunder Client — environments, variables, tests, and shareable collections.
- **REST vs GraphQL** — a clear-eyed comparison and a decision framework for when GraphQL earns its complexity.
- **Webhooks** — sending them reliably (retries, signing) and receiving them securely (HMAC signature verification, replay protection).
- **A developer-experience and security checklist** you can apply to any API before you call it "done."

---

## Part 1: REST Maturity Recap & Consistent Resource Design

### What "RESTful" actually means

REST (Representational State Transfer) is an architectural *style*, not a protocol. It was described by Roy Fielding in his 2000 dissertation as a set of constraints that, when followed, give a web API the same desirable properties as the web itself: scalability, evolvability, and a uniform interface. The constraints worth internalizing:

- **Client–server**: the client and server evolve independently; they share only the interface.
- **Statelessness**: every request carries everything the server needs to process it. The server stores no per-client session in memory between requests. (This is what makes horizontal scaling trivial — any server can handle any request.)
- **Uniform interface**: resources are identified by URLs, manipulated through representations (usually JSON), and acted on with a small, standard set of methods (the HTTP verbs).
- **Cacheability**: responses declare whether they can be cached, so intermediaries and clients can avoid redundant work.
- **Layered system**: a client cannot tell whether it is talking to the origin server or a proxy/CDN/gateway in front of it.

Most "REST APIs" in the wild are really **HTTP JSON APIs** that honor *some* of these constraints. That is fine. The goal is not dogmatic purity; it is consistency and predictability.

### The Richardson Maturity Model

A useful ladder for grading how "RESTful" an API is, introduced by Leonard Richardson:

| Level | Name | What it means | Typical sign |
| --- | --- | --- | --- |
| 0 | The Swamp of POX | One endpoint, one verb (usually `POST`), everything is an RPC call | `POST /api` with `{ "action": "getUser" }` |
| 1 | Resources | Multiple URLs identify distinct resources, but verbs are not used semantically | `POST /getUser/123`, `POST /deleteUser/123` |
| 2 | HTTP Verbs | Resources + correct HTTP methods + correct status codes | `GET /users/123`, `DELETE /users/123` → `204` |
| 3 | Hypermedia (HATEOAS) | Responses include links telling the client what it can do next | Response embeds `"links": { "self": ..., "orders": ... }` |

**Level 2 is the practical sweet spot** for most APIs and is what interviewers mean by "a well-designed REST API." Level 3 (HATEOAS) is powerful but rarely fully adopted; we will show a lightweight, valuable subset of it (pagination links) later.

### Resource design rules that pay off

A resource is a *noun* — a thing in your domain (a user, an order, a comment). The URL identifies it; the HTTP method says what to do to it.

```text
GET    /users           → list users (a collection)
POST   /users           → create a user in the collection
GET    /users/123       → fetch one user (a member)
PUT    /users/123       → replace user 123 entirely
PATCH  /users/123       → partially update user 123
DELETE /users/123       → delete user 123
```

Nesting expresses ownership/containment:

```text
GET    /users/123/orders        → orders belonging to user 123
POST   /users/123/orders        → create an order for user 123
GET    /users/123/orders/55     → a specific order of that user
```

A small set of rules keeps things consistent:

1. **Use plural nouns for collections** (`/users`, not `/user` or `/getUsers`). Consistency beats your personal grammar preference.
2. **No verbs in URLs.** The HTTP method *is* the verb. `POST /users` not `POST /createUser`.
3. **Lowercase, hyphen-separated paths** (`/shipping-addresses`, not `/shippingAddresses` or `/shipping_addresses`). Hyphens are friendlier to URLs and SEO than underscores.
4. **IDs in the path, options in the query string.** `/users/123` identifies the resource; `/users?status=active&sort=-createdAt` filters/sorts the collection.
5. **Don't go deeper than one level of nesting.** `/users/123/orders/55/items/9` is hard to read and brittle. Prefer `/order-items/9` (and let it carry `orderId`) once you are two levels deep.

> BEST PRACTICE: For operations that genuinely are not CRUD on a noun — "publish this article," "cancel this subscription," "send a password reset" — model them as a sub-resource state change (`PATCH /articles/123 { "status": "published" }`) when possible, or as a controlled action sub-collection (`POST /subscriptions/123/cancellations`). When neither feels natural, a pragmatic `POST /articles/123/actions/publish` is acceptable. Don't twist yourself into knots; consistency and clarity matter more than purity.

### Choose the right HTTP method (and know their properties)

Two properties dominate correct API behavior and come up constantly in interviews:

- **Safe**: the method does not change server state (read-only). `GET`, `HEAD`, `OPTIONS`.
- **Idempotent**: making the same request N times has the same effect as making it once. Crucial for safe retries.

| Method | Safe | Idempotent | Use for | Typical success status |
| --- | --- | --- | --- | --- |
| `GET` | Yes | Yes | Read a resource/collection | `200 OK` |
| `HEAD` | Yes | Yes | Read headers only (no body) | `200 OK` |
| `POST` | No | **No** | Create a resource; non-idempotent actions | `201 Created` (or `200`) |
| `PUT` | No | Yes | Replace a resource wholesale | `200 OK` / `204 No Content` |
| `PATCH` | No | No* | Partially update a resource | `200 OK` |
| `DELETE` | No | Yes | Remove a resource | `204 No Content` |

> WARNING — Pitfall: `POST` is **not** idempotent. If a client's network drops and it retries a `POST /payments`, you may charge twice. This is exactly the problem **idempotency keys** (Part 6) solve. `PATCH` is idempotent *only if you design it to be* — `{ "balance": 100 }` is idempotent (sets an absolute value), but `{ "increment": 10 }` is not.

### Use HTTP status codes precisely

Status codes are the first thing a client inspects. Use them correctly and you eliminate an entire class of client-side guesswork.

| Code | Meaning | When to use |
| --- | --- | --- |
| `200 OK` | Success with a body | `GET`, successful `PUT`/`PATCH` returning the resource |
| `201 Created` | A new resource was created | After `POST` that creates; include a `Location` header |
| `202 Accepted` | Accepted for async processing | Job queued; result not ready yet |
| `204 No Content` | Success, no body | `DELETE`, or `PUT`/`PATCH` returning nothing |
| `301`/`308` | Moved permanently | URL changed (308 preserves method) |
| `304 Not Modified` | Cached copy is still valid | Conditional `GET` with `If-None-Match` |
| `400 Bad Request` | Malformed/invalid request | Validation failed, bad JSON |
| `401 Unauthorized` | Not authenticated | Missing/invalid credentials (really "unauthenticated") |
| `403 Forbidden` | Authenticated but not allowed | Permission denied |
| `404 Not Found` | Resource doesn't exist | Unknown ID or route |
| `405 Method Not Allowed` | Verb not supported on this URL | `DELETE /users` (collection) |
| `409 Conflict` | State conflict | Duplicate unique field, version conflict |
| `422 Unprocessable Entity` | Syntactically OK, semantically invalid | Validation errors (popular alternative to 400) |
| `429 Too Many Requests` | Rate limit exceeded | Throttling; include `Retry-After` |
| `500 Internal Server Error` | Unexpected server fault | Unhandled exception (your bug) |
| `502`/`503`/`504` | Upstream/availability problems | Bad gateway, service down, timeout |

> TIP: A reliable rule — **`4xx` means "you (the client) did something wrong," `5xx` means "we (the server) did something wrong."** If you find yourself returning `500` for a validation failure, you have misclassified a client error as a server error, and your on-call dashboards will lie to you.

---

## Part 2: API Versioning Strategies

### Why version at all

The moment a third party (or your own mobile app in the wild) depends on your API, you can no longer change response shapes freely. Renaming a field, removing one, or changing a type is a **breaking change** that will crash clients you cannot redeploy. Versioning gives you a controlled way to evolve: ship `v2` with the breaking changes, keep `v1` running, and migrate clients on a timeline.

A crucial distinction:

- **Non-breaking (backward-compatible) changes** — *add* a new optional field, *add* a new endpoint, *add* a new enum value clients can ignore. These do **not** require a new version. Clients that don't know about the addition keep working.
- **Breaking changes** — *remove* or *rename* a field, change a type, change semantics, make an optional field required, change error formats. These **require** a new version.

> BEST PRACTICE: Version *defensively but lazily*. Add new fields freely under the current version. Only mint a new major version when you genuinely must break compatibility. Each live version is a maintenance tax forever.

### The three main strategies

| Strategy | Example | Pros | Cons |
| --- | --- | --- | --- |
| **URI path** | `GET /v1/users` | Dead simple, visible, easy to route, easy to test in a browser | "Not purely RESTful" (URL of a resource changes); version sprawl in routing |
| **Custom header** | `GET /users` + `Api-Version: 2` | Clean URLs; resource URL is stable | Invisible in a browser; harder to test/cache; easy to forget |
| **Media type (content negotiation)** | `Accept: application/vnd.myapp.v2+json` | Most "RESTful"; per-representation versioning | Complex; poor tooling/cache support; confusing to consumers |

**URI path versioning wins in practice** for most teams. It is the most discoverable, the easiest to route and document, and the least surprising to consumers. Stripe famously uses a *date-based* header version (`Stripe-Version: 2022-11-15`) for fine-grained evolution, which is excellent but operationally heavy; GitHub uses both an `Accept` media type and now a date header. For your own APIs, start with the path.

### Implementing URI path versioning in Express

Mount versioned routers. Keep each version's routes and controllers separate so you can evolve them independently.

```js
// app.js
const express = require('express');
const app = express();

app.use(express.json());

// Each version is its own router with its own controllers.
const usersV1 = require('./routes/v1/users');
const usersV2 = require('./routes/v2/users');

app.use('/v1/users', usersV1);
app.use('/v2/users', usersV2);

// Optional: point bare /api at the current stable version.
app.use('/api/users', usersV2);

module.exports = app;
```

```js
// routes/v2/users.js
const router = require('express').Router();

router.get('/', (req, res) => {
  // v2 shape: nested "name" object instead of v1's flat "fullName".
  res.json({
    data: [{ id: '1', name: { first: 'Ada', last: 'Lovelace' }, email: 'ada@example.com' }],
  });
});

module.exports = router;
```

### Implementing header versioning in Express

Read the version from a header and dispatch. A small middleware keeps the logic in one place.

```js
// middleware/version.js
// Resolves the requested version from a header, defaulting to the latest.
const SUPPORTED = ['1', '2'];
const LATEST = '2';

function resolveVersion(req, res, next) {
  const requested = req.get('Api-Version') || LATEST;
  if (!SUPPORTED.includes(requested)) {
    return res.status(400).json({
      error: { code: 'UNSUPPORTED_VERSION', message: `Api-Version ${requested} is not supported.` },
    });
  }
  req.apiVersion = requested;          // available to downstream handlers
  res.set('Api-Version', requested);   // echo back so clients can confirm
  next();
}

module.exports = { resolveVersion };
```

```js
// usage
const { resolveVersion } = require('./middleware/version');
app.use(resolveVersion);

app.get('/users', (req, res) => {
  if (req.apiVersion === '1') return res.json({ data: [{ id: '1', fullName: 'Ada Lovelace' }] });
  return res.json({ data: [{ id: '1', name: { first: 'Ada', last: 'Lovelace' } }] });
});
```

### Implementing media-type versioning

```js
// Parse a version out of an Accept header like: application/vnd.myapp.v2+json
function resolveMediaVersion(req, res, next) {
  const accept = req.get('Accept') || '';
  const match = accept.match(/application\/vnd\.myapp\.v(\d+)\+json/);
  req.apiVersion = match ? match[1] : '2'; // default latest
  res.type(`application/vnd.myapp.v${req.apiVersion}+json`);
  next();
}
```

### Migration & deprecation tactics

Shipping a new version is half the job; *retiring* the old one without breaking clients is the other half.

1. **Announce early, in writing.** Document the new version, the changes, and a migration guide before you flip any switches.
2. **Signal deprecation in responses.** Use the standardized `Deprecation` and `Sunset` headers so clients learn programmatically.

   ```js
   // Mark a v1 endpoint as deprecated with a sunset date.
   app.use('/v1', (req, res, next) => {
     res.set('Deprecation', 'true'); // or an HTTP-date when it was deprecated
     res.set('Sunset', 'Wed, 31 Dec 2026 23:59:59 GMT'); // RFC 8594
     res.set('Link', '</v2/docs>; rel="deprecation successor-version"');
     next();
   });
   ```

3. **Instrument usage.** Log which versions/endpoints are still being called and by whom (API key). You cannot safely sunset what you cannot measure.
4. **Give a generous, communicated window.** Email known integrators. Consider "brownouts" (briefly returning `410 Gone` during off-peak windows) to flush out clients that ignored the warnings — used by large providers, but only after ample notice.
5. **Default thoughtfully.** If clients omit a version, default to a *pinned* version (the one they signed up with) or the *oldest supported*, not "latest" — otherwise you break silent clients every time you release.

> WARNING — Pitfall: Defaulting unspecified requests to "latest" turns every release into a surprise breaking change for clients who never sent a version. Stripe's model — pin each account to the version it was created with, and let them opt into upgrades — exists precisely to avoid this.

---

## Part 3: Pagination Patterns

Returning every row of a growing table in one response is a bug waiting to happen: it blows up memory, times out, and gets slower every day. Pagination is non-negotiable for any collection that can grow. There are two dominant strategies, and choosing the wrong one causes real production bugs.

### Offset / limit (page-based) pagination

The familiar approach: skip `offset` rows and return the next `limit`. Often surfaced as `page` + `pageSize`.

```text
GET /users?page=3&limit=20     →  skip (3-1)*20 = 40, take 20
GET /users?offset=40&limit=20  →  equivalent, explicit offset
```

```js
// controllers/users.js — offset pagination with Mongoose
async function listUsers(req, res, next) {
  try {
    // Parse and clamp inputs — never trust the client to send sane numbers.
    const page = Math.max(1, parseInt(req.query.page, 10) || 1);
    const limit = Math.min(100, Math.max(1, parseInt(req.query.limit, 10) || 20)); // hard cap at 100
    const skip = (page - 1) * limit;

    // Run the query and the count in parallel.
    const [data, total] = await Promise.all([
      User.find().sort({ createdAt: -1 }).skip(skip).limit(limit).lean(),
      User.countDocuments(),
    ]);

    res.json({
      data,
      meta: {
        page,
        limit,
        total,
        totalPages: Math.ceil(total / limit),
        hasNext: page * limit < total,
        hasPrev: page > 1,
      },
    });
  } catch (err) {
    next(err);
  }
}
```

**Pros:** intuitive ("page 3 of 10"), supports jumping to arbitrary pages, gives a total count.

**Cons — and they are serious:**

- **Performance degrades with depth.** `OFFSET 1000000` (or Mongo's `.skip(1000000)`) forces the database to read and discard a million rows before returning your page. Deep pages get linearly slower.
- **Drift / instability.** If rows are inserted or deleted between page loads, items can be **skipped or duplicated**. A user reading page 2 might see an item they already saw on page 1 because a new row pushed everything down.

> WARNING — Pitfall: Offset pagination over data that changes (a feed, a leaderboard, anything sorted by `createdAt` with active writes) will show users duplicate or missing items. This is not a hypothetical — it's the classic "I keep seeing the same tweet" bug.

### Cursor (keyset / seek) pagination

Instead of "skip N rows," cursor pagination says "give me rows *after this specific point*." The cursor encodes the sort key of the last item you saw. The query becomes a `WHERE` filter, which the database can satisfy with an index — fast and stable regardless of depth.

```text
GET /users?limit=20                          → first page
GET /users?limit=20&after=<cursor>           → next page, items after the cursor
```

```js
// controllers/users.js — cursor pagination with Mongoose, sorted by _id (or createdAt + _id)
async function listUsersCursor(req, res, next) {
  try {
    const limit = Math.min(100, Math.max(1, parseInt(req.query.limit, 10) || 20));

    // Decode the opaque cursor. We base64-encode it so clients treat it as a black box.
    let filter = {};
    if (req.query.after) {
      const decoded = Buffer.from(req.query.after, 'base64url').toString('utf8');
      // For a single monotonic key (_id), the cursor is just the last id.
      filter = { _id: { $gt: decoded } };
    }

    // Fetch limit + 1 to detect whether a next page exists, without a separate count.
    const rows = await User.find(filter).sort({ _id: 1 }).limit(limit + 1).lean();

    const hasNext = rows.length > limit;
    const data = hasNext ? rows.slice(0, limit) : rows;

    // Build the next cursor from the last returned row.
    const nextCursor = hasNext
      ? Buffer.from(String(data[data.length - 1]._id), 'utf8').toString('base64url')
      : null;

    res.json({
      data,
      meta: { limit, hasNext, nextCursor },
      links: {
        next: nextCursor ? `/users?limit=${limit}&after=${nextCursor}` : null,
      },
    });
  } catch (err) {
    next(err);
  }
}
```

**Pros:** constant-time performance at any depth (it's an indexed range scan); stable under inserts/deletes; ideal for infinite scroll and real-time feeds.

**Cons:** cannot jump to an arbitrary page ("go to page 47" is meaningless); typically no total count; the sort key must be unique and stable (combine `createdAt` with `_id` as a tiebreaker if `createdAt` can collide).

> BEST PRACTICE: Make cursors **opaque** (base64-encode them, optionally sign them). Clients should treat a cursor as a meaningless token, not parse it. This lets you change the internal cursor format later without breaking anyone. Never expose raw offsets or internal IDs that clients might start depending on.

### Composite cursors (when the sort key isn't unique)

If you sort by `createdAt` and two rows share a timestamp, a `>` filter can skip or repeat rows. Encode *both* the timestamp and the `_id` and compare lexicographically.

```js
// Build a composite filter: items strictly after (createdAt, _id).
function buildKeysetFilter(after) {
  if (!after) return {};
  const { createdAt, id } = JSON.parse(Buffer.from(after, 'base64url').toString('utf8'));
  return {
    $or: [
      { createdAt: { $lt: new Date(createdAt) } },                 // older timestamp
      { createdAt: new Date(createdAt), _id: { $lt: id } },        // same timestamp, smaller id
    ],
  };
}
// Sort by { createdAt: -1, _id: -1 } and ensure a compound index on the same keys.
```

### Choosing between them

| Need | Use |
| --- | --- |
| Admin table, "page 5 of 200", jump to any page, show total | **Offset/limit** |
| Infinite scroll, social feed, large/active dataset, deep pages | **Cursor** |
| Public API exposed to many clients at scale | **Cursor** (predictable cost) |
| Small, mostly static dataset where simplicity wins | **Offset/limit** |

> TIP: Always send pagination **metadata**, not just an array. Wrapping the list in `{ data, meta, links }` lets you add total counts, next-cursors, and HATEOAS-style links without breaking clients. The JSON:API spec and GitHub's `Link` header are good references for the `links` shape.

---

## Part 4: Filtering, Sorting, and Field Selection

A good list endpoint lets clients ask for exactly what they need — no more, no less. Done carelessly, this is also where **NoSQL injection** and **unbounded, database-melting queries** sneak in. The art is offering power while staying safe.

### Filtering

Expose filters as query parameters. The key decision is **whitelist, never pass-through.** Do not hand `req.query` straight to your database.

```js
// GET /products?category=books&minPrice=10&maxPrice=50&inStock=true
function buildFilter(query) {
  const filter = {};

  // 1. Exact-match fields — only allow ones you explicitly support.
  if (query.category) filter.category = String(query.category);
  if (query.inStock !== undefined) filter.inStock = query.inStock === 'true';

  // 2. Range fields — translate friendly names to operators yourself.
  const price = {};
  if (query.minPrice) price.$gte = Number(query.minPrice);
  if (query.maxPrice) price.$lte = Number(query.maxPrice);
  if (Object.keys(price).length) filter.price = price;

  return filter;
}
```

> WARNING — Pitfall (NoSQL injection): If you do `User.find(req.query)` and a caller sends `?role[$ne]=admin` or `?password[$gt]=`, Express's query parser builds a nested object and you've just handed an attacker a query operator. Combined with body parsing, attackers can craft `{ "$ne": null }` style payloads to bypass auth. **Defenses:** whitelist fields and coerce types as above; run `express-mongo-sanitize` to strip keys containing `$` and `.`; and validate the query with a schema (Zod/Joi). Never spread untrusted input into a query object.

```js
// Strip $ and . from keys in req.body/req.query/req.params to block operator injection.
const mongoSanitize = require('express-mongo-sanitize');
app.use(mongoSanitize());
```

For richer query languages (e.g., `?filter[price][gte]=10`), pick a convention and document it. A common professional pattern maps suffixes to operators explicitly:

```js
// ?price__gte=10&price__lte=50&name__like=phone
const OPERATORS = { gte: '$gte', lte: '$lte', gt: '$gt', lt: '$lt', ne: '$ne' };
const ALLOWED_FIELDS = new Set(['price', 'name', 'category', 'rating']);

function parseAdvancedFilter(query) {
  const filter = {};
  for (const [rawKey, rawVal] of Object.entries(query)) {
    const [field, op] = rawKey.split('__');
    if (!ALLOWED_FIELDS.has(field)) continue;       // ignore unknown fields
    if (op && OPERATORS[op]) {
      filter[field] = { ...(filter[field] || {}), [OPERATORS[op]]: coerce(rawVal) };
    } else if (!op) {
      filter[field] = coerce(rawVal);
    }
  }
  return filter;
}
function coerce(v) {
  if (v === 'true') return true;
  if (v === 'false') return false;
  if (v !== '' && !isNaN(v)) return Number(v);
  return String(v);
}
```

### Sorting

Adopt the widely understood convention: comma-separated fields, a leading `-` for descending.

```text
GET /products?sort=-price          → price descending
GET /products?sort=category,-price → category asc, then price desc
```

```js
const SORTABLE = new Set(['price', 'createdAt', 'rating', 'name']); // whitelist!

function buildSort(sortParam) {
  if (!sortParam) return { createdAt: -1 }; // sensible default
  const sort = {};
  for (const token of String(sortParam).split(',')) {
    const desc = token.startsWith('-');
    const field = desc ? token.slice(1) : token;
    if (SORTABLE.has(field)) sort[field] = desc ? -1 : 1; // silently ignore unknown fields
  }
  return Object.keys(sort).length ? sort : { createdAt: -1 };
}
```

> WARNING — Pitfall: Allowing sorting on arbitrary, unindexed fields lets a single request trigger a full collection scan and sort in memory — a denial-of-service vector. Whitelist sortable fields and ensure each has a supporting index.

### Field selection (sparse fieldsets)

Let clients request only the fields they need to shrink payloads — great for mobile and bandwidth-sensitive consumers. The JSON:API convention is `fields=...`; Google's is `fields=`; pick one.

```text
GET /users/123?fields=id,name,email   → return only those fields
```

```js
function buildProjection(fieldsParam, allowed) {
  if (!fieldsParam) return null; // null → return the default representation
  const requested = String(fieldsParam).split(',').map((f) => f.trim());
  const projection = {};
  for (const f of requested) {
    if (allowed.has(f)) projection[f] = 1; // whitelist again
  }
  // Always include _id unless explicitly excluded; never let projection select secrets.
  return Object.keys(projection).length ? projection : null;
}

// Usage
const ALLOWED_USER_FIELDS = new Set(['name', 'email', 'createdAt', 'role']); // note: no "passwordHash"
const projection = buildProjection(req.query.fields, ALLOWED_USER_FIELDS);
const user = await User.findById(req.params.id, projection).lean();
```

> BEST PRACTICE: Field selection must respect a server-side allowlist that **excludes sensitive fields** (`passwordHash`, internal flags, other users' PII). A client should never be able to project a column out of the database just by naming it. The allowlist is your security boundary, not a convenience.

### Putting it together in one clean handler

```js
async function listProducts(req, res, next) {
  try {
    const filter = parseAdvancedFilter(req.query);
    const sort = buildSort(req.query.sort);
    const projection = buildProjection(req.query.fields, new Set(['name', 'price', 'category', 'rating']));
    const limit = Math.min(100, Math.max(1, parseInt(req.query.limit, 10) || 20));
    const page = Math.max(1, parseInt(req.query.page, 10) || 1);

    const [data, total] = await Promise.all([
      Product.find(filter, projection).sort(sort).skip((page - 1) * limit).limit(limit).lean(),
      Product.countDocuments(filter),
    ]);

    res.json({
      data,
      meta: { page, limit, total, totalPages: Math.ceil(total / limit) },
    });
  } catch (err) {
    next(err);
  }
}
```

---

## Part 5: Consistent Success & Error Envelopes

An API that returns `{ "id": 1 }` for one endpoint, `[{...}]` for another, and `{ "error": "oops" }` (sometimes a string, sometimes an object) for failures forces every client to write bespoke parsing for each route. **One envelope, everywhere** is the single highest-leverage consistency decision you can make.

### Should you wrap responses?

There are two schools:

- **Envelope everything**: every response is `{ "data": ... , "meta": ... }`. Pros: uniform, room to add metadata (pagination, warnings) without breaking changes. Cons: slightly more verbose; a single resource is `{ "data": {...} }` instead of `{...}`.
- **Bare bodies, envelopes only for collections/errors**: return the resource directly for `GET /users/1`, wrap lists and always wrap errors.

Either is defensible; **consistency is what matters.** This module uses the **envelope-everything** approach because it scales best for public APIs and pagination.

### A success envelope

```json
{
  "data": { "id": "123", "name": "Ada Lovelace", "email": "ada@example.com" },
  "meta": { "requestId": "req_a1b2c3" }
}
```

For collections, `meta` carries pagination; `links` carries navigation:

```json
{
  "data": [ { "id": "1" }, { "id": "2" } ],
  "meta": { "page": 1, "limit": 20, "total": 134, "totalPages": 7 },
  "links": { "self": "/users?page=1", "next": "/users?page=2", "prev": null }
}
```

### An error envelope

A great error tells the client (a) *that* it failed, (b) *why*, in machine-readable form, (c) *what* to fix, and (d) a correlation id for support. The IETF standard for this is **Problem Details for HTTP APIs (RFC 9457, formerly 7807)**, served as `application/problem+json`:

```json
{
  "type": "https://api.example.com/errors/validation",
  "title": "Your request parameters didn't validate.",
  "status": 422,
  "detail": "Two fields failed validation.",
  "instance": "/users",
  "errors": [
    { "field": "email", "code": "INVALID", "message": "Must be a valid email address." },
    { "field": "age",   "code": "MIN",     "message": "Must be at least 18." }
  ],
  "requestId": "req_a1b2c3"
}
```

You don't have to adopt RFC 9457 literally, but a structured error with a stable **machine-readable `code`** is essential. Clients should branch on `code`, never on the human `message` (which you will want to reword/translate).

### A reusable AppError + centralized handler

The pattern: a custom error class, helper constructors, controllers that `throw`/`next(err)`, and **one** error-handling middleware that renders the envelope. This keeps every controller clean and guarantees a single error shape.

```js
// errors/AppError.js
class AppError extends Error {
  constructor(status, code, message, details = null) {
    super(message);
    this.status = status;       // HTTP status
    this.code = code;           // stable machine code, e.g. 'NOT_FOUND'
    this.details = details;     // optional array/object with specifics
    this.isOperational = true;  // distinguishes expected errors from bugs
    Error.captureStackTrace(this, this.constructor);
  }
}

// Convenience factories
AppError.notFound = (msg = 'Resource not found') => new AppError(404, 'NOT_FOUND', msg);
AppError.badRequest = (msg, details) => new AppError(400, 'BAD_REQUEST', msg, details);
AppError.validation = (details) => new AppError(422, 'VALIDATION_FAILED', 'Validation failed', details);
AppError.unauthorized = (msg = 'Authentication required') => new AppError(401, 'UNAUTHORIZED', msg);
AppError.forbidden = (msg = 'You do not have permission') => new AppError(403, 'FORBIDDEN', msg);
AppError.conflict = (msg, details) => new AppError(409, 'CONFLICT', msg, details);

module.exports = AppError;
```

```js
// middleware/errorHandler.js
const AppError = require('../errors/AppError');

// 404 handler — mount AFTER all routes, BEFORE the error handler.
function notFoundHandler(req, res, next) {
  next(AppError.notFound(`Route ${req.method} ${req.originalUrl} not found`));
}

// Central error handler — Express identifies it by its FOUR arguments.
function errorHandler(err, req, res, next) {
  // Normalize well-known framework/DB errors into AppErrors.
  if (err.name === 'ValidationError') {                 // Mongoose validation
    const details = Object.values(err.errors).map((e) => ({ field: e.path, message: e.message }));
    err = AppError.validation(details);
  } else if (err.name === 'CastError') {                // bad ObjectId
    err = AppError.badRequest(`Invalid value for ${err.path}`);
  } else if (err.code === 11000) {                      // Mongo duplicate key
    const field = Object.keys(err.keyValue || {})[0];
    err = AppError.conflict(`Duplicate value for ${field}`, { field });
  } else if (!(err instanceof AppError)) {              // unknown → 500, don't leak internals
    console.error('UNEXPECTED ERROR:', err); // log full error/stack server-side only
    err = new AppError(500, 'INTERNAL_ERROR', 'An unexpected error occurred');
    err.isOperational = false;
  }

  const body = {
    error: {
      code: err.code,
      message: err.message,
      ...(err.details ? { details: err.details } : {}),
    },
    meta: { requestId: req.id },
  };

  // Never leak stack traces in production; include them only in development.
  if (process.env.NODE_ENV !== 'production' && err.stack) {
    body.error.stack = err.stack;
  }

  res.status(err.status || 500).json(body);
}

module.exports = { notFoundHandler, errorHandler };
```

```js
// app.js wiring (order matters!)
const { notFoundHandler, errorHandler } = require('./middleware/errorHandler');

// ... all your routers ...
app.use(notFoundHandler); // catches unmatched routes
app.use(errorHandler);    // must be LAST, and must have 4 args
```

> WARNING — Pitfall (Express 4 vs 5 async errors): In **Express 4**, a rejected promise inside an `async` handler is **not** automatically passed to your error handler — an uncaught rejection just hangs the request. You must `try/catch` and call `next(err)`, or wrap handlers with a helper. **Express 5 fixes this**: it awaits handler promises and forwards rejections to the error handler automatically. Until you're on v5, use the wrapper below.

```js
// utils/asyncHandler.js — makes async controllers forward rejections to next() in Express 4.
const asyncHandler = (fn) => (req, res, next) => Promise.resolve(fn(req, res, next)).catch(next);
module.exports = asyncHandler;

// usage
router.get('/users/:id', asyncHandler(async (req, res) => {
  const user = await User.findById(req.params.id).lean();
  if (!user) throw AppError.notFound('User not found'); // bubbles to errorHandler
  res.json({ data: user });
}));
```

> BEST PRACTICE: Attach a **request id** to every request (e.g., with `nanoid` or the `Request-Id` header) and echo it in both success and error envelopes plus your logs. When a client reports "request `req_a1b2c3` failed," you can find the exact log line in seconds. This single practice transforms production debugging.

```js
// middleware/requestId.js
const { nanoid } = require('nanoid');
module.exports = (req, res, next) => {
  req.id = req.get('X-Request-Id') || `req_${nanoid(12)}`;
  res.set('X-Request-Id', req.id);
  next();
};
```

---

## Part 6: Rate Limiting, Throttling, and Idempotency

### Why rate limit

Without limits, a single buggy client (or a malicious one) can exhaust your CPU, database connections, or third-party quota and take everyone down. Rate limiting protects availability, enforces fair use across tenants, and is your first line of defense against credential-stuffing and scraping.

Terms worth distinguishing:

- **Rate limiting**: reject requests above a hard ceiling in a window (e.g., 100 req/min). Excess gets `429`.
- **Throttling**: slow down rather than reject — delay or queue excess requests so they trickle through.
- **Quotas**: longer-horizon totals (e.g., 10,000 req/day) often tied to a billing tier.

Common algorithms:

| Algorithm | Idea | Trade-off |
| --- | --- | --- |
| **Fixed window** | Count per clock window (e.g., per minute) | Simple, but allows 2x burst at window boundaries |
| **Sliding window log** | Track timestamps, count those within the last N seconds | Accurate, more memory |
| **Sliding window counter** | Weighted blend of current + previous window | Good accuracy/memory balance |
| **Token bucket** | Tokens refill at a steady rate; each request spends one; allows controlled bursts | Most flexible; industry standard |
| **Leaky bucket** | Requests queue and drain at a fixed rate | Smooths bursts into a steady stream |

### Implementing with `express-rate-limit`

```js
const rateLimit = require('express-rate-limit');

// A general limiter for the whole API.
const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,            // 15 minutes
  limit: 100,                          // 100 requests per window per key (v7 renamed `max` → `limit`)
  standardHeaders: 'draft-7',          // send RateLimit-* headers (IETF draft)
  legacyHeaders: false,                // disable old X-RateLimit-* headers
  // Key by API key if present, else IP. Behind a proxy, set app.set('trust proxy', 1).
  keyGenerator: (req) => req.get('X-Api-Key') || req.ip,
  handler: (req, res) => {
    res.status(429).json({
      error: { code: 'RATE_LIMITED', message: 'Too many requests. Please slow down.' },
    });
  },
});

app.use('/api', apiLimiter);

// A stricter limiter on auth endpoints to blunt brute-force/credential stuffing.
const authLimiter = rateLimit({ windowMs: 15 * 60 * 1000, limit: 5 });
app.use('/api/auth/login', authLimiter);
```

> WARNING — Pitfall: The **default in-memory store does not work across multiple processes or servers** — each instance keeps its own counter, so your effective limit multiplies by the number of instances. In production with PM2 cluster mode or multiple containers, use a shared store like `rate-limit-redis`.

```js
const { RedisStore } = require('rate-limit-redis');
const { createClient } = require('redis');
const redis = createClient({ url: process.env.REDIS_URL });
redis.connect();

const apiLimiter = rateLimit({
  windowMs: 60 * 1000,
  limit: 100,
  store: new RedisStore({ sendCommand: (...args) => redis.sendCommand(args) }),
});
```

> WARNING — Pitfall (trust proxy): Behind a load balancer/reverse proxy, `req.ip` is the *proxy's* IP unless you set `app.set('trust proxy', 1)` so Express reads the real client IP from `X-Forwarded-For`. Get this wrong and you either rate-limit everyone as one client (proxy IP) or trust a spoofable header. Set it to the specific number of proxy hops, not blindly to `true`.

### Communicating limits to clients

Always tell clients where they stand so well-behaved ones can self-throttle:

```text
RateLimit-Limit: 100
RateLimit-Remaining: 27
RateLimit-Reset: 42          (seconds until the window resets)
Retry-After: 42             (on a 429 response)
```

### Idempotency keys for safe retries

Recall that `POST` is not idempotent. A client that times out on `POST /payments` and retries might create two charges. The industry-standard fix (popularized by Stripe) is the **idempotency key**: the client generates a unique key per logical operation and sends it as a header. The server remembers the first result for that key and **returns the cached response** for any retry — so the operation happens exactly once.

```text
POST /payments
Idempotency-Key: 8f14e45f-ea29-4d72-9c1a-0b2c3d4e5f6a
{ "amount": 5000, "currency": "usd" }
```

```js
// middleware/idempotency.js — minimal, correct idempotency using Redis.
// Real implementations also store the response body/status and handle concurrent retries.
const { createClient } = require('redis');
const redis = createClient({ url: process.env.REDIS_URL });
redis.connect();

const IDEMPOTENT_METHODS = new Set(['POST', 'PATCH', 'DELETE']);

async function idempotency(req, res, next) {
  if (!IDEMPOTENT_METHODS.has(req.method)) return next();

  const key = req.get('Idempotency-Key');
  if (!key) return next(); // optional; or require it on payment-like routes

  const cacheKey = `idem:${req.method}:${req.originalUrl}:${key}`;

  // Atomically claim the key. NX = only set if not exists; EX = 24h TTL.
  const claimed = await redis.set(cacheKey, 'in-progress', { NX: true, EX: 86400 });

  if (!claimed) {
    // Key already seen. Return the stored response if it's finished...
    const stored = await redis.get(`${cacheKey}:response`);
    if (stored) {
      const { status, body } = JSON.parse(stored);
      return res.status(status).set('Idempotent-Replayed', 'true').json(body);
    }
    // ...or the original request is still processing → ask the client to retry shortly.
    return res.status(409).json({
      error: { code: 'IDEMPOTENCY_IN_PROGRESS', message: 'A request with this key is still processing.' },
    });
  }

  // First time we've seen this key. Wrap res.json to capture and cache the response.
  const originalJson = res.json.bind(res);
  res.json = (body) => {
    redis.set(`${cacheKey}:response`, JSON.stringify({ status: res.statusCode, body }), { EX: 86400 });
    return originalJson(body);
  };
  next();
}

module.exports = idempotency;
```

> BEST PRACTICE: Make idempotency keys **mandatory** on dangerous mutating endpoints (payments, order creation, fund transfers). Store the *full response* (status + body) keyed by `Idempotency-Key`, ideally scoped to the authenticated client and the request fingerprint, with a sensible TTL (Stripe uses 24h). Bonus: store a hash of the request body so a key reused with *different* parameters is rejected (a likely client bug) rather than silently replaying the old result.

> TIP: Idempotency keys and rate limiting work together with retries: clients should retry `5xx`/`429` responses with **exponential backoff + jitter**, carrying the same idempotency key each time so retries are safe.

---

## Part 7: Documenting Your API with OpenAPI / Swagger

An undocumented API does not really exist to the people who would use it. **OpenAPI** (formerly "Swagger") is the industry-standard, machine-readable description format for REST APIs. From one OpenAPI document you get: interactive docs (Swagger UI), client SDK generation, server stubs, mock servers, contract tests, and request validation. The document itself is YAML or JSON describing your paths, parameters, request/response schemas, and security.

A quick vocabulary note: **OpenAPI** is the specification; **Swagger** is the family of tooling (Swagger UI, Swagger Editor) that predates and supports it. People use the words interchangeably.

### Two ways to author the spec

1. **Spec-first**: write the OpenAPI YAML by hand (or in Swagger Editor), then generate/validate code against it. Best for large teams and design review.
2. **Code-first**: annotate your route files with JSDoc comments and generate the spec from them with `swagger-jsdoc`. Best for keeping docs next to the code that implements them. We'll use this.

### Setup: `swagger-jsdoc` + `swagger-ui-express`

```bash
npm install swagger-jsdoc swagger-ui-express
```

```js
// docs/swagger.js
const swaggerJsdoc = require('swagger-jsdoc');

const options = {
  definition: {
    openapi: '3.0.3',
    info: {
      title: 'Bookstore API',
      version: '2.0.0',
      description: 'A sample API demonstrating documentation best practices.',
      contact: { name: 'API Support', email: 'api@example.com' },
      license: { name: 'MIT' },
    },
    servers: [
      { url: 'http://localhost:3000/api', description: 'Local' },
      { url: 'https://api.example.com', description: 'Production' },
    ],
    // Reusable building blocks referenced via $ref throughout the spec.
    components: {
      securitySchemes: {
        bearerAuth: { type: 'http', scheme: 'bearer', bearerFormat: 'JWT' },
        apiKey: { type: 'apiKey', in: 'header', name: 'X-Api-Key' },
      },
      schemas: {
        Book: {
          type: 'object',
          required: ['id', 'title', 'author'],
          properties: {
            id: { type: 'string', example: '64f0c2a1e1b2c3d4e5f6a7b8' },
            title: { type: 'string', example: 'The Pragmatic Programmer' },
            author: { type: 'string', example: 'Andrew Hunt' },
            price: { type: 'number', format: 'float', example: 39.99 },
            createdAt: { type: 'string', format: 'date-time' },
          },
        },
        Error: {
          type: 'object',
          properties: {
            error: {
              type: 'object',
              properties: {
                code: { type: 'string', example: 'NOT_FOUND' },
                message: { type: 'string', example: 'Book not found' },
              },
            },
          },
        },
      },
    },
    // Apply bearer auth globally; individual ops can override with `security: []`.
    security: [{ bearerAuth: [] }],
  },
  // Files to scan for @openapi JSDoc annotations.
  apis: ['./routes/*.js'],
};

module.exports = swaggerJsdoc(options);
```

```js
// app.js — serve interactive docs and the raw spec.
const swaggerUi = require('swagger-ui-express');
const swaggerSpec = require('./docs/swagger');

app.use('/docs', swaggerUi.serve, swaggerUi.setup(swaggerSpec, { explorer: true }));

// Serve the raw spec so tools (Postman, codegen) can import it.
app.get('/openapi.json', (req, res) => res.json(swaggerSpec));
```

### Annotating routes

Annotations live as JSDoc above each route. They reference the reusable schemas with `$ref`, keeping things DRY.

```js
// routes/books.js
/**
 * @openapi
 * /books:
 *   get:
 *     summary: List books
 *     tags: [Books]
 *     parameters:
 *       - in: query
 *         name: page
 *         schema: { type: integer, default: 1, minimum: 1 }
 *         description: Page number
 *       - in: query
 *         name: limit
 *         schema: { type: integer, default: 20, maximum: 100 }
 *         description: Items per page
 *       - in: query
 *         name: sort
 *         schema: { type: string, example: "-price" }
 *         description: "Comma-separated sort fields; prefix with - for descending"
 *     responses:
 *       200:
 *         description: A paginated list of books
 *         content:
 *           application/json:
 *             schema:
 *               type: object
 *               properties:
 *                 data:
 *                   type: array
 *                   items: { $ref: '#/components/schemas/Book' }
 *                 meta:
 *                   type: object
 *                   properties:
 *                     page: { type: integer }
 *                     total: { type: integer }
 */
router.get('/books', listBooks);

/**
 * @openapi
 * /books/{id}:
 *   get:
 *     summary: Get a book by id
 *     tags: [Books]
 *     parameters:
 *       - in: path
 *         name: id
 *         required: true
 *         schema: { type: string }
 *     responses:
 *       200:
 *         description: The book
 *         content:
 *           application/json:
 *             schema:
 *               type: object
 *               properties:
 *                 data: { $ref: '#/components/schemas/Book' }
 *       404:
 *         description: Not found
 *         content:
 *           application/json:
 *             schema: { $ref: '#/components/schemas/Error' }
 */
router.get('/books/:id', getBook);
```

Visit `http://localhost:3000/docs` and you get a fully interactive, "Try it out" documentation site generated from those comments.

> BEST PRACTICE: Keep your OpenAPI spec **honest and automated**. Generate it from code, serve it at a stable URL, and add a CI check that fails if the spec is invalid or drifts from reality. Even better: use the same schemas to *validate incoming requests* (e.g., `express-openapi-validator`) so your docs and your runtime validation can never disagree — the doc becomes the contract.

> WARNING — Pitfall: Hand-written docs in a wiki rot the instant code changes. The single most common documentation failure is docs that describe an API version that no longer exists. Code-adjacent, generated docs that ship with each deploy are the cure.

---

## Part 8: Exploring & Sharing APIs (Postman / Thunder Client)

Before clients consume your API, *you* need to drive it interactively — to develop, debug, and demo. **Postman** (standalone app) and **Thunder Client** (a lightweight VS Code extension) are the two dominant tools. They do the same core job; Thunder Client is simpler and stays in your editor, Postman is more powerful for teams.

### Core concepts (identical in both tools)

- **Request**: a saved HTTP call (method, URL, headers, body, auth).
- **Collection**: a folder of related requests you can run, share, and version.
- **Environment**: a named set of variables (`{{baseUrl}}`, `{{token}}`) so the same collection runs against local, staging, and prod by switching environments.
- **Variables**: placeholders resolved at send time; can be set by scripts (e.g., capture a token from a login response and reuse it).
- **Tests/scripts**: JavaScript assertions that run after a response, turning a collection into an executable test suite.

### A typical workflow

1. Create an environment with `baseUrl = http://localhost:3000/api` and an empty `token`.
2. Add a `POST {{baseUrl}}/auth/login` request. In its **Tests** tab, capture the token:

   ```js
   // Postman test script: save the returned JWT into the environment for later requests.
   const json = pm.response.json();
   pm.environment.set('token', json.data.token);
   pm.test('login returns 200', () => pm.response.to.have.status(200));
   ```

3. Other requests set `Authorization: Bearer {{token}}` and just work — the token flows automatically.
4. Add assertions to each request:

   ```js
   pm.test('status is 200', () => pm.response.to.have.status(200));
   pm.test('has data array', () => pm.expect(pm.response.json().data).to.be.an('array'));
   pm.test('responds under 500ms', () => pm.expect(pm.response.responseTime).to.be.below(500));
   ```

5. Run the whole collection with the **Collection Runner** (or `newman`, Postman's CLI, in CI):

   ```bash
   # Run a collection in CI against the staging environment.
   npx newman run bookstore.postman_collection.json -e staging.postman_environment.json
   ```

### Importing your OpenAPI spec

Both tools can **import your `/openapi.json`** and generate a ready-made collection with every endpoint, parameter, and example pre-filled. This closes the loop: document once, get an interactive client for free.

### Sharing collections

- **Postman**: publish to a Workspace (team), export as a `*.postman_collection.json` to commit to your repo, or share a public documentation link. Use **environments without secrets** when sharing publicly — never export a collection with a live token baked in.
- **Thunder Client**: collections and environments live as JSON files you can commit directly to the repo (`thunder-collection_*.json`), which is excellent for code review and keeping the API client in version control next to the code.

> BEST PRACTICE: Commit your collection (and a `*.example` environment with placeholder values, no secrets) to the repository. New teammates clone the repo and immediately have a working, documented set of API calls. Wire `newman` into CI as smoke tests against staging after each deploy.

> WARNING — Pitfall: It is dangerously easy to export a Postman environment that still contains a real bearer token or API key and commit it or post it in a ticket. Treat exported environments like `.env` files: scrub secrets, use variables, and add `*.postman_environment.json` with real values to `.gitignore`.

---

## Part 9: REST vs GraphQL — When to Consider GraphQL

REST is the default and the right choice for the vast majority of APIs. But you should understand GraphQL well enough to (a) decide when it earns its complexity and (b) answer the inevitable interview question.

### What GraphQL is

GraphQL is a query language and runtime (created at Facebook) where the client sends a single query describing *exactly* the shape of data it wants, and the server returns precisely that shape. There is typically **one endpoint** (`POST /graphql`); the query in the body determines everything.

```graphql
# The client asks for exactly these fields across related entities in ONE request.
query {
  user(id: "123") {
    name
    email
    orders(last: 3) {
      id
      total
      items { name price }
    }
  }
}
```

### The two problems GraphQL targets

- **Over-fetching**: a REST `GET /users/123` returns 30 fields when the mobile screen needs 3. GraphQL returns only what's asked.
- **Under-fetching / N+1 round-trips**: rendering a screen needs the user, then their orders, then each order's items — three REST round-trips. GraphQL resolves the whole tree in one request.

### Head-to-head

| Dimension | REST | GraphQL |
| --- | --- | --- |
| Endpoints | Many (one per resource) | One (`/graphql`) |
| Data fetching | Server decides the shape | Client decides the shape |
| Over/under-fetching | Common | Solved by design |
| HTTP caching | Excellent (URLs + status + `ETag`) | Hard (single POST URL); needs client cache (Apollo) |
| Versioning | Explicit (v1/v2) | Evolve the schema; deprecate fields |
| Error handling | HTTP status codes | Usually `200` with an `errors` array |
| Learning curve / tooling on server | Low | Higher (schema, resolvers, dataloaders) |
| File uploads, simple CRUD | Natural | Awkward |
| Rate limiting | Per endpoint (easy) | Per query *cost* (complex; queries vary wildly) |
| Built-in introspection/docs | Via OpenAPI | Built into the schema |

### When GraphQL is worth it

- You have **many heterogeneous clients** (web, iOS, Android, partners) with very different data needs, and you're tired of either bloated responses or a proliferation of `?fields=` and custom endpoints.
- Your UI aggregates **many related resources** per screen and round-trips hurt (mobile, high-latency networks).
- You're building a **gateway over multiple backend services** (GraphQL federation) and want a unified graph.
- A rich frontend team wants to iterate on data needs without backend changes for every screen.

### When to stay with REST

- Simple CRUD, server-to-server APIs, or public APIs where **HTTP caching and CDN behavior matter** a lot.
- File uploads/downloads, streaming, binary content.
- A small team without the appetite for resolvers, schema design, query-cost analysis, and the new class of security concerns (a malicious deeply-nested query can DoS your server — you need depth/complexity limits).
- You want the simplest possible operational story and the broadest tooling.

> BEST PRACTICE: You don't have to choose globally. Many teams run **REST for most things and a GraphQL endpoint (often a BFF — Backend For Frontend) for the data-heavy parts of a rich UI.** Start with REST; reach for GraphQL when over/under-fetching across many clients becomes a measured, recurring pain — not because it's fashionable.

> WARNING — Pitfall: GraphQL does not magically make your API faster or simpler. It moves complexity from many endpoints into resolvers and demands disciplined solutions for N+1 database queries (DataLoader), query-cost limiting, authorization per field, and caching. Adopting it without that discipline trades one set of problems for a worse one.

---

## Part 10: Webhooks — Sending and Receiving Securely

Polling ("is it done yet? is it done yet?") wastes resources and adds latency. **Webhooks** invert the flow: when an event happens, *your* server makes an HTTP `POST` to a URL the consumer registered, pushing the event to them in real time. Stripe, GitHub, Slack, and Twilio all work this way. As a backend engineer you'll both *send* webhooks (notifying your customers) and *receive* them (consuming third-party events). Both have non-obvious correctness and security requirements.

### Receiving webhooks securely

When you receive a webhook, anyone on the internet can `POST` to that URL. You must verify the request genuinely came from the expected sender. The standard mechanism is an **HMAC signature**: the sender computes `HMAC-SHA256(payload, sharedSecret)` and puts it in a header; you recompute it and compare.

```js
// Receiving a Stripe-style webhook. The signature is over the RAW body, so we must
// NOT let express.json() parse it first — we need the exact bytes that were signed.
const express = require('express');
const crypto = require('crypto');
const router = express.Router();

router.post(
  '/webhooks/provider',
  express.raw({ type: 'application/json' }), // gives us a Buffer in req.body
  (req, res) => {
    const signatureHeader = req.get('X-Signature');         // e.g. "t=1718800000,v1=abc123..."
    const secret = process.env.WEBHOOK_SECRET;

    // 1. Parse the timestamp and signature out of the header.
    const parts = Object.fromEntries(signatureHeader.split(',').map((p) => p.split('=')));
    const timestamp = parts.t;
    const provided = parts.v1;

    // 2. Reject replays: ignore anything older than 5 minutes.
    const ageSeconds = Math.abs(Date.now() / 1000 - Number(timestamp));
    if (ageSeconds > 300) {
      return res.status(400).json({ error: { code: 'WEBHOOK_TOO_OLD', message: 'Timestamp outside tolerance' } });
    }

    // 3. Recompute the HMAC over `${timestamp}.${rawBody}` and compare in constant time.
    const signedPayload = `${timestamp}.${req.body.toString('utf8')}`;
    const expected = crypto.createHmac('sha256', secret).update(signedPayload).digest('hex');

    const valid =
      provided &&
      expected.length === provided.length && // timingSafeEqual throws on length mismatch
      crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(provided));

    if (!valid) {
      return res.status(401).json({ error: { code: 'INVALID_SIGNATURE', message: 'Signature verification failed' } });
    }

    // 4. Now it's safe to parse and act on the event.
    const event = JSON.parse(req.body.toString('utf8'));

    // 5. Respond FAST (2xx) and do heavy work asynchronously (queue it).
    //    Most providers retry if you don't ack within a few seconds.
    enqueueEventForProcessing(event); // e.g., push to a job queue
    res.status(200).json({ received: true });
  }
);
```

The non-obvious but critical points:

- **Verify the signature over the raw, unparsed body.** If `express.json()` parses and re-serializes the body, the bytes change and the signature won't match. Mount `express.raw()` for the webhook route specifically (and ensure global `express.json()` doesn't run first for that path).
- **Use a constant-time comparison** (`crypto.timingSafeEqual`), never `===`, to avoid timing attacks that leak the signature byte by byte.
- **Check a timestamp and reject old requests** to prevent replay attacks (an attacker re-sending a captured valid request).
- **Acknowledge quickly, process asynchronously.** Return `2xx` immediately and hand the work to a queue. Slow handlers cause the sender to time out and retry, duplicating work.
- **Be idempotent.** Providers deliver *at least once* — you *will* receive duplicates. Dedupe on the event id (store processed ids).

```js
// Idempotent processing: skip events we've already handled.
async function handleEvent(event) {
  const seen = await ProcessedEvent.findOne({ eventId: event.id });
  if (seen) return; // already processed; safely ignore the duplicate
  await ProcessedEvent.create({ eventId: event.id }); // record before/within the same txn as the work
  // ... do the actual work ...
}
```

### Sending webhooks reliably

If you're the provider firing webhooks, reliability is your responsibility:

1. **Sign every payload** with a per-subscriber secret (HMAC), including a timestamp, so receivers can verify and reject replays.
2. **Retry with exponential backoff** on non-`2xx` responses (e.g., after 1m, 5m, 30m, 2h…), with a cap, then mark the endpoint failing/disabled and alert the customer.
3. **Send a stable event id** so receivers can dedupe.
4. **Set a short timeout** (a few seconds) on the delivery request — receivers should ack fast.
5. **Let customers verify deliveries** with a dashboard/log of attempts and responses, and offer a "resend" button.

```js
// Sending a signed webhook with a timestamp.
const crypto = require('crypto');

async function deliverWebhook(endpoint, secret, event) {
  const payload = JSON.stringify(event);
  const timestamp = Math.floor(Date.now() / 1000);
  const signature = crypto
    .createHmac('sha256', secret)
    .update(`${timestamp}.${payload}`)
    .digest('hex');

  const res = await fetch(endpoint.url, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-Signature': `t=${timestamp},v1=${signature}`,
      'X-Event-Id': event.id,
      'X-Event-Type': event.type,
    },
    body: payload,
    signal: AbortSignal.timeout(5000), // fail fast; the receiver must ack quickly
  });

  if (!res.ok) throw new Error(`Webhook delivery failed: ${res.status}`); // triggers your retry logic
}
```

> WARNING — Pitfall (SSRF when sending): When you let users register webhook URLs, you're making your server fetch arbitrary URLs they control — a Server-Side Request Forgery risk. Validate that the target is a public address: reject `localhost`, `127.0.0.1`, link-local (`169.254.x.x`), and private ranges (`10.x`, `192.168.x`, `172.16–31.x`) so an attacker can't make your server hit internal services or cloud metadata endpoints (`169.254.169.254`).

> TIP: For local development, tools like `ngrok`, `localtunnel`, or the Stripe CLI (`stripe listen --forward-to localhost:3000/webhooks/stripe`) expose your local server to the internet so providers can reach your machine.

---

## Common Pitfalls & Gotchas

- **Returning `200` for everything.** Encoding success/failure in a body field while always sending `200` defeats HTTP and breaks clients, proxies, and monitoring. Use real status codes.
- **Misusing `401` vs `403`.** `401` = not authenticated (who are you?); `403` = authenticated but not allowed (you can't do that). Swapping them confuses clients and security reviews.
- **Leaking internals in errors.** Returning stack traces, SQL/Mongo errors, or "user not found" vs "wrong password" distinctions leaks implementation details and aids attackers. Normalize errors; log details server-side only.
- **NoSQL injection via `find(req.query)`.** Spreading untrusted query/body into a database query lets `$ne`, `$gt`, `$where` operators through. Whitelist fields, coerce types, sanitize.
- **Unbounded list endpoints.** No default limit, no max limit, or sort on unindexed fields → a single request can OOM or DoS your DB. Always clamp `limit` and whitelist sortable fields.
- **Offset pagination on live data.** Causes skipped/duplicated items as rows shift. Use cursor pagination for feeds and large datasets.
- **In-memory rate limiting across instances.** Each process counts separately, multiplying your real limit. Use a shared (Redis) store in production.
- **Wrong `trust proxy` behind a load balancer.** Either everyone shares the proxy IP (one rate-limit bucket) or you trust a spoofable `X-Forwarded-For`. Set it to the actual hop count.
- **Treating `POST` as idempotent.** Network retries double-create. Use idempotency keys on dangerous mutations.
- **Parsing the webhook body before verifying the signature.** `express.json()` re-serialization breaks HMAC verification. Use `express.raw()` for webhook routes and compare in constant time.
- **Docs that drift from code.** Hand-maintained docs lie within a week. Generate OpenAPI from code and serve it with each deploy.
- **Express 4 swallowing async errors.** Rejected promises in async handlers don't reach your error handler in v4. Wrap with `asyncHandler` (or upgrade to v5).
- **Committing secrets in Postman exports.** Exported environments often contain live tokens. Scrub and `.gitignore` them.
- **Breaking changes without a new version.** Renaming/removing fields under the same version breaks every existing client silently.

---

## Best Practices

- **Be consistent above all.** Pick conventions (URL casing, envelope shape, error codes, pagination style) and apply them to *every* endpoint. Predictability is the whole game.
- **Design resources as nouns; let HTTP verbs and status codes carry meaning.** Aim for Richardson Level 2; add pagination `links` for a taste of Level 3.
- **Version with the URI path** for most APIs; signal deprecation with `Deprecation`/`Sunset` headers and measure usage before sunsetting.
- **Default to cursor pagination** for anything large or live; always return `meta`/`links`; clamp `limit` with a hard cap.
- **Whitelist everything client-controlled** — filters, sort fields, projections — and exclude sensitive fields from selection. Sanitize input and validate with a schema (Zod/Joi).
- **Speak one envelope.** Wrap data in `{ data, meta }`; return errors as `{ error: { code, message, details } }` with a stable machine-readable `code` and a `requestId`.
- **Centralize error handling** in one Express error middleware; never leak stack traces in production.
- **Rate limit in layers** (global + stricter auth limiter), with a shared store, standard headers, and `Retry-After` on `429`.
- **Support idempotency keys** on mutating endpoints; store full responses keyed by the idempotency key; have clients retry with exponential backoff + jitter.
- **Generate docs from code** (OpenAPI via `swagger-jsdoc`), serve them at `/docs`, expose the raw spec, and validate requests against the same schemas.
- **Commit your Postman/Thunder Client collection** (secrets scrubbed) and run it in CI with `newman` as post-deploy smoke tests.
- **Choose GraphQL deliberately**, only when over/under-fetching across many clients is a measured pain — and bring depth/complexity limits and DataLoader with you.
- **Verify webhook signatures over the raw body in constant time, reject replays, ack fast, process async, and dedupe.** Guard against SSRF when sending to user-supplied URLs.
- **Secure by default**: HTTPS only, `helmet` for headers, CORS configured to specific origins, payload size limits (`express.json({ limit })`), auth on every non-public route, and least-privilege API keys with rotation.

---

## Key Interview Points

- **"What makes an API RESTful?"** — Statelessness, a uniform interface (resources as URLs, standard HTTP verbs, representations), client–server separation, cacheability, layering. In practice, "RESTful" means Richardson Level 2: noun resources + correct verbs + correct status codes.
- **"Safe vs idempotent methods?"** — Safe = no state change (`GET`, `HEAD`, `OPTIONS`). Idempotent = same effect when repeated (`GET`, `PUT`, `DELETE`, `HEAD`). `POST` is neither safe nor idempotent; `PATCH` is idempotent only if you design it to set absolute values.
- **"`PUT` vs `PATCH`?"** — `PUT` replaces the whole resource (and is idempotent); `PATCH` partially updates it. Sending `{ name: 'x' }` to `PUT` may wipe unspecified fields; to `PATCH` it updates just `name`.
- **"`401` vs `403`?"** — `401` = unauthenticated (no/invalid credentials); `403` = authenticated but not permitted.
- **"How do you version an API and why?"** — Because breaking changes would crash existing clients you can't redeploy. Strategies: URI path (most practical/discoverable), header, media type. Add new fields freely (non-breaking); only mint a major version for breaking changes; deprecate with `Sunset` headers and usage metrics.
- **"Offset vs cursor pagination?"** — Offset is simple and supports random page access but degrades at depth (`skip` reads/discards rows) and drifts under writes (skipped/duplicate items). Cursor (keyset) is index-backed (constant cost at any depth) and stable, but can't jump to arbitrary pages and usually lacks a total count. Use cursor for large/live data and public APIs.
- **"How do you prevent a list endpoint from being abused?"** — Clamp `limit` with a hard max, whitelist filterable/sortable fields (with indexes), whitelist projectable fields (excluding secrets), sanitize input to block NoSQL operator injection, and rate limit.
- **"How do you make `POST` safe to retry?"** — Idempotency keys: client sends a unique `Idempotency-Key` header; server stores the first response and replays it for retries, so the operation runs exactly once.
- **"What rate-limiting algorithm would you use?"** — Token bucket (allows controlled bursts, steady refill) is the common industry choice; mention fixed/sliding window trade-offs and the need for a shared store across instances plus correct `trust proxy`.
- **"How do you document an API?"** — OpenAPI 3 spec, generated from code (e.g., `swagger-jsdoc`), served via Swagger UI, with reusable component schemas; ideally also used to validate requests so docs and runtime can't drift.
- **"REST vs GraphQL — when GraphQL?"** — When many heterogeneous clients have diverse data needs and over/under-fetching is a real pain, or you're federating multiple services. Trade-offs: harder HTTP caching, error handling (200 + errors array), rate limiting by query cost, N+1 (DataLoader), and query-complexity DoS limits.
- **"How do you secure a webhook receiver?"** — Verify an HMAC signature over the *raw* body using a constant-time compare, reject stale timestamps (replay protection), ack with `2xx` quickly and process asynchronously, and dedupe by event id (deliveries are at-least-once).
- **"How do you design a consistent error response?"** — A single envelope with a stable machine-readable `code`, a human `message`, optional `details`, and a `requestId`; correct HTTP status; centralized handler; no stack traces in production. RFC 9457 (Problem Details) is the standard.

---

## Practice Exercises

These build on each other. Start a fresh Express app (`npm init -y && npm i express`) and add packages as needed.

1. **Resource & status-code design.** Implement an in-memory `tasks` resource with `GET /v1/tasks`, `POST /v1/tasks` (returns `201` + `Location`), `GET /v1/tasks/:id`, `PATCH /v1/tasks/:id`, and `DELETE /v1/tasks/:id` (returns `204`). Return correct status codes for missing tasks (`404`) and validation failures (`422`).

2. **Consistent envelopes + central error handler.** Add an `AppError` class, an `asyncHandler` wrapper, a `notFoundHandler`, and a single `errorHandler` that emits `{ error: { code, message, details? }, meta: { requestId } }`. Attach a request id to every request and echo it. Make a deliberately throwing route and confirm it produces the envelope without a stack trace when `NODE_ENV=production`.

3. **List endpoint done right.** Extend `GET /v1/tasks` with offset pagination (`page`, `limit` clamped to max 50), whitelisted filtering (`status`, `priority`), whitelisted sorting (`?sort=-createdAt,priority`), and sparse fieldsets (`?fields=id,title,status`). Return `{ data, meta }` with `page/limit/total/totalPages`. Add a second route `GET /v1/tasks/cursor` implementing cursor pagination with an opaque base64 cursor.

4. **Rate limiting + idempotency.** Add a global `express-rate-limit` (e.g., 60/min) with `429` + `Retry-After`, and a stricter limiter on `POST /v1/tasks`. Then implement an `Idempotency-Key` middleware (in-memory `Map` is fine for the exercise) that makes `POST /v1/tasks` replay the original response for a repeated key.

5. **Document it with OpenAPI.** Add `swagger-jsdoc` + `swagger-ui-express`. Define a reusable `Task` schema and an `Error` schema in `components`. Annotate all five task routes. Serve interactive docs at `/docs` and the raw spec at `/openapi.json`. Import the spec into Postman/Thunder Client and verify every endpoint appears.

6. **Webhook receiver (stretch).** Add `POST /webhooks/tasks` that accepts a raw body, verifies an HMAC-SHA256 signature in the `X-Signature: t=...,v1=...` format over `${t}.${rawBody}` using a shared secret, rejects timestamps older than 5 minutes, uses `crypto.timingSafeEqual`, and dedupes by an `X-Event-Id`. Write a small client script that signs and sends a valid event, then a tampered one, and confirm `200` vs `401`.

### Solutions

#### Solution 1 — Resource & status-code design

```js
// app.js
const express = require('express');
const { randomUUID } = require('crypto');
const app = express();
app.use(express.json());

const router = express.Router();
const tasks = new Map(); // id -> task

router.get('/', (req, res) => {
  res.json({ data: [...tasks.values()] });
});

router.post('/', (req, res) => {
  const { title, status = 'todo', priority = 'normal' } = req.body || {};
  if (!title || typeof title !== 'string') {
    return res.status(422).json({
      error: { code: 'VALIDATION_FAILED', message: 'title is required', details: [{ field: 'title', message: 'required string' }] },
    });
  }
  const task = { id: randomUUID(), title, status, priority, createdAt: new Date().toISOString() };
  tasks.set(task.id, task);
  res.status(201).location(`/v1/tasks/${task.id}`).json({ data: task }); // 201 + Location
});

router.get('/:id', (req, res) => {
  const task = tasks.get(req.params.id);
  if (!task) return res.status(404).json({ error: { code: 'NOT_FOUND', message: 'Task not found' } });
  res.json({ data: task });
});

router.patch('/:id', (req, res) => {
  const task = tasks.get(req.params.id);
  if (!task) return res.status(404).json({ error: { code: 'NOT_FOUND', message: 'Task not found' } });
  // Partial update: only overwrite provided fields.
  const { title, status, priority } = req.body || {};
  if (title !== undefined) task.title = title;
  if (status !== undefined) task.status = status;
  if (priority !== undefined) task.priority = priority;
  res.json({ data: task });
});

router.delete('/:id', (req, res) => {
  if (!tasks.has(req.params.id)) return res.status(404).json({ error: { code: 'NOT_FOUND', message: 'Task not found' } });
  tasks.delete(req.params.id);
  res.status(204).end(); // 204 No Content
});

app.use('/v1/tasks', router);
app.listen(3000, () => console.log('http://localhost:3000'));
module.exports = { app, tasks };
```

#### Solution 2 — Consistent envelopes + central error handler

```js
// errors/AppError.js
class AppError extends Error {
  constructor(status, code, message, details = null) {
    super(message);
    this.status = status; this.code = code; this.details = details; this.isOperational = true;
    Error.captureStackTrace(this, this.constructor);
  }
}
AppError.notFound = (m = 'Not found') => new AppError(404, 'NOT_FOUND', m);
AppError.validation = (details) => new AppError(422, 'VALIDATION_FAILED', 'Validation failed', details);
module.exports = AppError;
```

```js
// utils/asyncHandler.js
module.exports = (fn) => (req, res, next) => Promise.resolve(fn(req, res, next)).catch(next);
```

```js
// middleware/requestId.js
const { randomUUID } = require('crypto');
module.exports = (req, res, next) => {
  req.id = req.get('X-Request-Id') || `req_${randomUUID()}`;
  res.set('X-Request-Id', req.id);
  next();
};
```

```js
// middleware/errorHandler.js
const AppError = require('../errors/AppError');

function notFoundHandler(req, res, next) {
  next(AppError.notFound(`Route ${req.method} ${req.originalUrl} not found`));
}

function errorHandler(err, req, res, next) {
  if (!(err instanceof AppError)) {
    console.error('UNEXPECTED:', err);
    err = new AppError(500, 'INTERNAL_ERROR', 'An unexpected error occurred');
  }
  const body = {
    error: { code: err.code, message: err.message, ...(err.details ? { details: err.details } : {}) },
    meta: { requestId: req.id },
  };
  if (process.env.NODE_ENV !== 'production' && err.stack) body.error.stack = err.stack;
  res.status(err.status).json(body);
}
module.exports = { notFoundHandler, errorHandler };
```

```js
// wiring in app.js
const requestId = require('./middleware/requestId');
const asyncHandler = require('./utils/asyncHandler');
const { notFoundHandler, errorHandler } = require('./middleware/errorHandler');

app.use(requestId);

// A deliberately throwing route to test the handler.
app.get('/boom', asyncHandler(async () => { throw new Error('kaboom'); }));

// ... routers ...
app.use(notFoundHandler);
app.use(errorHandler); // LAST, four args
```

#### Solution 3 — List endpoint done right

```js
// helpers
const SORTABLE = new Set(['createdAt', 'priority', 'title']);
const FILTERABLE = { status: String, priority: String };
const PROJECTABLE = new Set(['id', 'title', 'status', 'priority', 'createdAt']);

function buildFilter(query) {
  const out = {};
  for (const [field, cast] of Object.entries(FILTERABLE)) {
    if (query[field] !== undefined) out[field] = cast(query[field]);
  }
  return out;
}
function matches(task, filter) {
  return Object.entries(filter).every(([k, v]) => task[k] === v);
}
function buildSort(sortParam) {
  if (!sortParam) return [['createdAt', -1]];
  const tokens = String(sortParam).split(',');
  const sorts = [];
  for (const t of tokens) {
    const desc = t.startsWith('-'); const field = desc ? t.slice(1) : t;
    if (SORTABLE.has(field)) sorts.push([field, desc ? -1 : 1]);
  }
  return sorts.length ? sorts : [['createdAt', -1]];
}
function applySort(arr, sorts) {
  return arr.sort((a, b) => {
    for (const [field, dir] of sorts) {
      if (a[field] < b[field]) return -1 * dir;
      if (a[field] > b[field]) return 1 * dir;
    }
    return 0;
  });
}
function project(task, fieldsParam) {
  if (!fieldsParam) return task;
  const fields = String(fieldsParam).split(',').filter((f) => PROJECTABLE.has(f));
  if (!fields.length) return task;
  return Object.fromEntries(fields.map((f) => [f, task[f]]));
}

// GET /v1/tasks  (offset pagination + filter + sort + fields)
router.get('/', (req, res) => {
  const page = Math.max(1, parseInt(req.query.page, 10) || 1);
  const limit = Math.min(50, Math.max(1, parseInt(req.query.limit, 10) || 20));
  const filter = buildFilter(req.query);

  let rows = [...tasks.values()].filter((t) => matches(t, filter));
  rows = applySort(rows, buildSort(req.query.sort));
  const total = rows.length;
  const pageRows = rows.slice((page - 1) * limit, (page - 1) * limit + limit).map((t) => project(t, req.query.fields));

  res.json({ data: pageRows, meta: { page, limit, total, totalPages: Math.ceil(total / limit) } });
});

// GET /v1/tasks/cursor (cursor pagination over a stable createdAt+id sort)
router.get('/cursor', (req, res) => {
  const limit = Math.min(50, Math.max(1, parseInt(req.query.limit, 10) || 20));
  let rows = applySort([...tasks.values()], [['createdAt', 1], ['id', 1]]);

  if (req.query.after) {
    const afterId = Buffer.from(req.query.after, 'base64url').toString('utf8');
    const idx = rows.findIndex((t) => t.id === afterId);
    rows = idx >= 0 ? rows.slice(idx + 1) : rows;
  }
  const slice = rows.slice(0, limit + 1);
  const hasNext = slice.length > limit;
  const data = hasNext ? slice.slice(0, limit) : slice;
  const nextCursor = hasNext ? Buffer.from(data[data.length - 1].id, 'utf8').toString('base64url') : null;

  res.json({ data, meta: { limit, hasNext, nextCursor }, links: { next: nextCursor ? `/v1/tasks/cursor?limit=${limit}&after=${nextCursor}` : null } });
});
```

> Note: order matters — register `/cursor` **before** `/:id` so Express doesn't treat "cursor" as an id.

#### Solution 4 — Rate limiting + idempotency

```js
const rateLimit = require('express-rate-limit');

const globalLimiter = rateLimit({
  windowMs: 60 * 1000,
  limit: 60,
  standardHeaders: 'draft-7',
  legacyHeaders: false,
  handler: (req, res) => {
    res.set('Retry-After', '60');
    res.status(429).json({ error: { code: 'RATE_LIMITED', message: 'Too many requests' } });
  },
});
app.use('/v1', globalLimiter);

const createLimiter = rateLimit({ windowMs: 60 * 1000, limit: 10 });

// In-memory idempotency (fine for the exercise; use Redis in production).
const idemStore = new Map(); // key -> { status, body }
function idempotency(req, res, next) {
  const key = req.get('Idempotency-Key');
  if (!key) return next();
  if (idemStore.has(key)) {
    const { status, body } = idemStore.get(key);
    return res.status(status).set('Idempotent-Replayed', 'true').json(body);
  }
  const originalJson = res.json.bind(res);
  res.json = (body) => { idemStore.set(key, { status: res.statusCode, body }); return originalJson(body); };
  next();
}

// apply to the create route
router.post('/', createLimiter, idempotency, /* ...handler from Solution 1... */);
```

#### Solution 5 — Document it with OpenAPI

```js
// docs/swagger.js
const swaggerJsdoc = require('swagger-jsdoc');
module.exports = swaggerJsdoc({
  definition: {
    openapi: '3.0.3',
    info: { title: 'Tasks API', version: '1.0.0' },
    servers: [{ url: 'http://localhost:3000' }],
    components: {
      schemas: {
        Task: {
          type: 'object',
          required: ['id', 'title', 'status'],
          properties: {
            id: { type: 'string', format: 'uuid' },
            title: { type: 'string', example: 'Write docs' },
            status: { type: 'string', enum: ['todo', 'doing', 'done'], example: 'todo' },
            priority: { type: 'string', example: 'normal' },
            createdAt: { type: 'string', format: 'date-time' },
          },
        },
        Error: {
          type: 'object',
          properties: {
            error: { type: 'object', properties: { code: { type: 'string' }, message: { type: 'string' } } },
          },
        },
      },
    },
  },
  apis: ['./routes/*.js'],
});
```

```js
// app.js
const swaggerUi = require('swagger-ui-express');
const swaggerSpec = require('./docs/swagger');
app.use('/docs', swaggerUi.serve, swaggerUi.setup(swaggerSpec));
app.get('/openapi.json', (req, res) => res.json(swaggerSpec));
```

```js
// routes/tasks.js — annotate one route (repeat for the rest)
/**
 * @openapi
 * /v1/tasks:
 *   post:
 *     summary: Create a task
 *     tags: [Tasks]
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             type: object
 *             required: [title]
 *             properties:
 *               title: { type: string }
 *               status: { type: string, enum: [todo, doing, done] }
 *     responses:
 *       201:
 *         description: Created
 *         content:
 *           application/json:
 *             schema:
 *               type: object
 *               properties: { data: { $ref: '#/components/schemas/Task' } }
 *       422:
 *         description: Validation failed
 *         content:
 *           application/json:
 *             schema: { $ref: '#/components/schemas/Error' }
 */
```

In Postman/Thunder Client: **Import → Link/File → `http://localhost:3000/openapi.json`** generates a collection containing all annotated endpoints.

#### Solution 6 — Webhook receiver

```js
// routes/webhooks.js
const express = require('express');
const crypto = require('crypto');
const router = express.Router();
const SECRET = process.env.WEBHOOK_SECRET || 'shared-test-secret';
const processed = new Set(); // event ids we've already handled (dedupe)

router.post('/tasks', express.raw({ type: '*/*' }), (req, res) => {
  const sig = req.get('X-Signature') || '';
  const eventId = req.get('X-Event-Id');
  const parts = Object.fromEntries(sig.split(',').map((p) => p.split('=')));
  const { t, v1 } = parts;

  // 1. Replay protection.
  if (!t || Math.abs(Date.now() / 1000 - Number(t)) > 300) {
    return res.status(400).json({ error: { code: 'WEBHOOK_TOO_OLD', message: 'Bad/old timestamp' } });
  }

  // 2. Recompute and constant-time compare.
  const expected = crypto.createHmac('sha256', SECRET).update(`${t}.${req.body.toString('utf8')}`).digest('hex');
  const ok =
    v1 && expected.length === v1.length &&
    crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(v1));
  if (!ok) return res.status(401).json({ error: { code: 'INVALID_SIGNATURE', message: 'Bad signature' } });

  // 3. Dedupe + ack fast.
  if (eventId && processed.has(eventId)) return res.status(200).json({ received: true, duplicate: true });
  if (eventId) processed.add(eventId);

  const event = JSON.parse(req.body.toString('utf8'));
  // ...enqueue for async processing...
  res.status(200).json({ received: true, type: event.type });
});

module.exports = router;
```

```js
// client.js — sign and send a valid event, then a tampered one.
const crypto = require('crypto');
const SECRET = 'shared-test-secret';

async function send(tamper = false) {
  const event = { id: crypto.randomUUID(), type: 'task.created', data: { id: '1' } };
  const body = JSON.stringify(event);
  const t = Math.floor(Date.now() / 1000);
  let v1 = crypto.createHmac('sha256', SECRET).update(`${t}.${body}`).digest('hex');
  if (tamper) v1 = v1.replace(/.$/, (c) => (c === '0' ? '1' : '0')); // corrupt one char

  const res = await fetch('http://localhost:3000/webhooks/tasks', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', 'X-Signature': `t=${t},v1=${v1}`, 'X-Event-Id': event.id },
    body,
  });
  console.log(tamper ? 'tampered' : 'valid', '→', res.status);
}

(async () => { await send(false); await send(true); })(); // expect: valid → 200, tampered → 401
```

---

## Summary / Cheat Sheet

| Topic | The one-liner to remember |
| --- | --- |
| REST maturity | Aim for Richardson **Level 2**: noun resources + correct verbs + correct status codes. |
| Resource design | Plural nouns, no verbs in URLs, lowercase-hyphenated, IDs in path, options in query, ≤1 nesting level. |
| Method properties | Safe: `GET/HEAD/OPTIONS`. Idempotent: `GET/PUT/DELETE/HEAD`. `POST` is neither. |
| Status codes | `4xx` = client's fault, `5xx` = server's fault. `401` unauthenticated, `403` forbidden, `422` validation. |
| Versioning | URI path is most practical; add fields freely, version only on breaking changes; deprecate with `Sunset`. |
| Pagination | Offset = simple but slow/unstable deep; **cursor** = fast & stable for large/live data. Always send `meta`/`links`. |
| Filter/sort/fields | **Whitelist** filterable, sortable, and projectable fields; exclude secrets; sanitize input. |
| Envelopes | Success `{ data, meta }`; error `{ error: { code, message, details } }` with stable `code` + `requestId`. |
| Error handling | One central Express error middleware (4 args, mounted last); no stack traces in prod; Express 4 needs `asyncHandler`. |
| Rate limiting | Token bucket; shared (Redis) store across instances; `RateLimit-*` headers + `Retry-After`; correct `trust proxy`. |
| Idempotency | `Idempotency-Key` header → store & replay the first response so `POST` runs exactly once. |
| Docs | OpenAPI 3 generated from code (`swagger-jsdoc`), served via `swagger-ui-express` at `/docs`; expose raw spec. |
| API clients | Postman/Thunder Client: environments + variables + tests; commit collections (scrub secrets); `newman` in CI. |
| REST vs GraphQL | GraphQL for many clients with diverse data needs / heavy aggregation; mind caching, N+1, query-cost limits. |
| Webhooks | Verify HMAC over **raw body** in constant time, reject replays, ack `2xx` fast, process async, dedupe by event id. |
| Security defaults | HTTPS, `helmet`, scoped CORS, body-size limits, auth everywhere, least-privilege keys + rotation, sanitize input. |

> FINAL TIP: When in doubt about any design choice, ask "What would Stripe/GitHub do?" and read their public API docs. They have made every mistake in this module so you don't have to — their conventions (idempotency keys, cursor pagination, versioned releases, signed webhooks, immaculate docs) are this entire module made concrete and battle-tested at scale.
