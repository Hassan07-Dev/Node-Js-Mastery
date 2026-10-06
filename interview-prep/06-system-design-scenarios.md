# System Design & Scenario-Based Interview Questions

A question bank for the *design* and *judgment* portion of a backend Node.js interview. These are the questions where there is no single right answer — the interviewer is probing how you reason about **trade-offs**, **failure modes**, and **scale**. A junior candidate recites definitions; a strong candidate sketches an architecture, names the constraints, and explains *why* they would choose option A over option B (and when they would reverse that decision).

The scenarios are framed around the stack you've been learning — **Node.js (current LTS), Express (v4 and v5 noted where it matters), MongoDB + Mongoose, and Redis** — but the principles (statelessness, idempotency, caching, queues, observability, graceful shutdown) transfer to any backend role.

## How to use this bank

- **Talk it out.** System-design answers are spoken, not written. For each question, practice giving a 2–4 minute spoken answer: state the goal, list the constraints, propose an approach, then call out the trade-offs and failure modes.
- **Always ask clarifying questions first.** In a real design interview, the first thing a senior engineer does is establish scope: expected traffic, read/write ratio, latency budget, consistency requirements, team size. The model answers below note the clarifying questions worth asking.
- **Lead with the trade-off, not the buzzword.** Saying "I'd add Redis" is weak. Saying "reads dominate writes ~50:1 and the data tolerates a few seconds of staleness, so I'd cache the read path in Redis with a short TTL and accept eventual consistency" is strong.
- **Last-minute revision:** jump to the [Rapid-Fire Quick Answers](#rapid-fire-quick-answers) table at the bottom.

> TIP: In every design answer, explicitly say what you are *optimizing for* (latency? throughput? cost? correctness? developer velocity?) and what you are willing to *sacrifice*. Interviewers reward candidates who make the trade-off visible instead of pretending one design is free.

---

## Beginner

### 1. **What does it mean for a Node service to be "stateless," and why does it matter for scaling?**

A service is **stateless** when no request depends on in-memory data left behind by a previous request *on that specific instance*. Each request carries everything needed to process it (or fetches it from a shared store), so any instance can handle any request interchangeably.

The classic statefulness trap is storing session data, in-memory caches, uploaded-file temp state, or counters on the process itself:

```js
// BAD: state lives in this one process's memory.
// If you run 3 instances behind a load balancer, a user who logs in on
// instance A is "logged out" the moment the LB routes them to instance B.
const sessions = new Map();
app.post("/login", (req, res) => {
  const sid = createSessionId();
  sessions.set(sid, { userId: req.user.id }); // in-memory => not shared
  res.cookie("sid", sid).send("ok");
});
```

```js
// GOOD: state lives in a shared store (Redis). Any instance can serve any request.
const session = require("express-session");
const RedisStore = require("connect-redis").default;
const { createClient } = require("redis");
const redisClient = createClient({ url: process.env.REDIS_URL });
redisClient.connect();

app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
}));
```

Statelessness matters because it is the precondition for **horizontal scaling**: to handle more load you add more identical instances behind a load balancer. If instances hold private state, you are forced into "sticky sessions" (pinning a user to one instance), which breaks failover, complicates deploys, and unbalances load. Stateless instances can be added, removed, restarted, and crash-replaced freely — which is also what makes zero-downtime deploys and autoscaling possible.

> BEST PRACTICE: Push all shared mutable state out of the process — sessions to Redis, files to object storage (S3), persistent data to the database. Keep the Node process as a pure request-processing unit.

> Follow-up: "Where would you store session state, and what about JWTs vs server-side sessions?"

---

### 2. **What is a load balancer, and what does it do for a Node application?**

A load balancer (LB) sits in front of multiple backend instances and distributes incoming requests across them. It serves several jobs at once:

- **Distribution** — spreads traffic so no single instance is overwhelmed (algorithms: round-robin, least-connections, IP-hash for stickiness).
- **Health checking** — periodically probes each instance (e.g. `GET /healthz`) and stops routing to instances that fail, enabling automatic failover.
- **TLS termination** — often decrypts HTTPS at the LB so backend instances handle plain HTTP internally (simpler, cheaper).
- **A single entry point** — clients hit one stable address; instances behind it can come and go.

In practice you rarely write the LB yourself — it's Nginx, HAProxy, AWS ALB/ELB, or a cloud equivalent. Within a single machine, Node's own `cluster` module acts as a mini load balancer across CPU cores (the OS or the primary process distributes connections to workers).

> WARNING — Pitfall: A load balancer only helps if your instances are stateless (Q1). With per-instance state you must enable sticky sessions, which undermines even distribution and failover.

> Follow-up: "What's the difference between an L4 and an L7 load balancer?" (L4 routes on TCP/IP; L7 understands HTTP and can route by path/header/cookie.)

---

### 3. **Walk me through designing a simple REST API for a blog (posts and comments). What endpoints and status codes would you use?**

Start by naming the **resources** (nouns), then map HTTP verbs onto them. Resources are `posts` and their nested `comments`.

| Method & path | Purpose | Success status | Notes |
| --- | --- | --- | --- |
| `GET /posts` | List posts (paginated) | `200` | Support `?page`, `?limit`, `?sort` |
| `POST /posts` | Create a post | `201` + `Location` header | Body validated |
| `GET /posts/:id` | Fetch one post | `200` / `404` | |
| `PUT /posts/:id` | Replace a post | `200` | Idempotent |
| `PATCH /posts/:id` | Partially update | `200` | |
| `DELETE /posts/:id` | Delete a post | `204` (no body) | Idempotent |
| `GET /posts/:id/comments` | List a post's comments | `200` | Nested resource |
| `POST /posts/:id/comments` | Add a comment | `201` | |

Design principles an interviewer wants to hear:

1. **Nouns in paths, verbs as HTTP methods** — `POST /posts`, never `POST /createPost`.
2. **Correct status codes** — `201` for creation (with a `Location` header pointing to the new resource), `204` for a successful delete with no body, `400` for validation errors, `401`/`403` for auth, `404` for missing, `409` for conflicts, `422` for semantic validation failures, `429` for rate limiting, `500` for unexpected server errors.
3. **Consistent error shape** — always return errors in the same JSON envelope so clients can parse them uniformly.
4. **Pagination, filtering, sorting** on collection endpoints, because `GET /posts` returning a million rows is a production incident.
5. **Versioning** — prefix with `/api/v1` so you can evolve the contract without breaking existing clients.

```js
// Consistent error envelope returned everywhere
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "title is required",
    "details": [{ "field": "title", "issue": "required" }]
  }
}
```

> Follow-up: "How would you paginate `GET /posts` for a feed with 10 million rows?" (See Q12 on offset vs cursor pagination.)

---

### 4. **What is caching, and what's the simplest way to add it to a read-heavy Node endpoint?**

Caching stores the result of an expensive operation (a DB query, an external API call, a computed view) so that subsequent identical requests can be served from a fast store instead of recomputing. It trades **freshness** for **speed and reduced load**.

For a single Node instance, the simplest cache is an in-memory map with a TTL. But the moment you run more than one instance, an in-memory cache becomes inconsistent across instances — so the standard answer is a **shared cache in Redis**:

```js
const { createClient } = require("redis");
const redis = createClient({ url: process.env.REDIS_URL });
redis.connect();

async function getProduct(id) {
  const cacheKey = `product:${id}`;

  // 1. Try the cache (cache-aside / lazy loading pattern)
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  // 2. Cache miss => hit the source of truth
  const product = await Product.findById(id).lean();
  if (!product) return null;

  // 3. Populate the cache with a TTL so stale data self-expires
  await redis.set(cacheKey, JSON.stringify(product), { EX: 300 }); // 5 min
  return product;
}
```

This is the **cache-aside** (lazy-loading) pattern: the application checks the cache, falls back to the DB on a miss, and writes the result back. The TTL bounds staleness automatically. The two hard problems are *invalidation* (Q20) and *stampede* (Q21).

> TIP: A short TTL is the cheapest form of cache invalidation. If the data tolerates being a few minutes stale, a TTL alone often beats elaborate invalidation logic.

> Follow-up: "What happens when the cached item changes in the DB?" (invalidation) and "What if 10,000 requests miss the cache at the same instant?" (stampede).

---

### 5. **What is rate limiting and why would an API need it?**

Rate limiting caps how many requests a client (by IP, API key, or user ID) may make in a time window. It exists to:

- **Protect the service** from being overwhelmed — by abusive clients, buggy retry loops, or DoS attempts.
- **Enforce fairness** so one heavy client doesn't starve everyone else.
- **Control cost** when downstream resources (a paid API, the database) are expensive per call.
- **Defend specific endpoints** — e.g. login and password-reset routes get tight limits to slow credential-stuffing.

A minimal Express implementation uses `express-rate-limit`, but for a *multi-instance* deployment the counter must live in a **shared store** (Redis), or each instance enforces its own separate limit and the effective limit becomes N times higher than intended:

```js
const rateLimit = require("express-rate-limit");
const { RedisStore } = require("rate-limit-redis");

const limiter = rateLimit({
  windowMs: 60_000,          // 1 minute window
  max: 100,                  // 100 requests per window per key
  standardHeaders: true,     // emit RateLimit-* headers
  legacyHeaders: false,
  // Shared store so the limit is global across all instances
  store: new RedisStore({ sendCommand: (...args) => redis.sendCommand(args) }),
});

app.use("/api/", limiter);
```

A correct rate-limited response uses status **`429 Too Many Requests`** and should include a `Retry-After` header telling the client when to try again.

> WARNING — Pitfall: An in-memory rate limiter on a 3-instance deployment behind a round-robin LB allows ~3× the configured limit. Always back the counter with Redis when you have more than one instance.

> Follow-up: "What rate-limiting *algorithm* would you use — fixed window, sliding window, token bucket?" (See Q14.)

---

### 6. **A user needs to upload a profile picture. How do you handle file uploads in Express at a small scale?**

At small scale, accept the upload with `multer` (the standard Express middleware for `multipart/form-data`), validate it, and store it. The two things to decide are *where bytes land* and *what you validate*.

```js
const multer = require("multer");

const upload = multer({
  storage: multer.memoryStorage(),  // keep the file in a Buffer (small files only)
  limits: { fileSize: 2 * 1024 * 1024 }, // 2 MB hard cap — reject oversized uploads
  fileFilter: (req, file, cb) => {
    // Validate the declared MIME type (necessary but not sufficient — see warning)
    const allowed = ["image/jpeg", "image/png", "image/webp"];
    cb(allowed.includes(file.mimetype) ? null : new Error("INVALID_FILE_TYPE"), allowed.includes(file.mimetype));
  },
});

app.post("/avatar", upload.single("avatar"), async (req, res) => {
  // req.file.buffer holds the bytes; persist to object storage, not local disk
  const url = await uploadToS3(req.file.buffer, req.file.mimetype);
  await User.findByIdAndUpdate(req.user.id, { avatarUrl: url });
  res.status(201).json({ url });
});
```

Key decisions:

- **Always set `limits.fileSize`.** Without it, a client can stream gigabytes and exhaust memory/disk.
- **Don't store uploads on the local filesystem of a stateless instance** — they vanish on redeploy and aren't shared across instances. Use object storage (S3, GCS) and keep only the URL in the DB.
- **Validate content, not just the declared MIME type.** The `Content-Type` header is client-controlled and easily spoofed. For real safety, sniff magic bytes (e.g. with `file-type`) and, for images, re-encode with `sharp` to strip malicious payloads.

> WARNING — Pitfall: Trusting `file.mimetype` (which comes straight from the client) is a common vulnerability. A `.php` file renamed to `.png` will pass that check. Verify the actual bytes.

> Follow-up: "Now suppose files are 5 GB videos and there are thousands of concurrent uploads." (See Q16 — presigned URLs.)

---

### 7. **What is graceful shutdown and why does a Node service need it?**

Graceful shutdown means that when the process receives a termination signal (`SIGTERM` from Kubernetes/Docker/PM2, `SIGINT` from Ctrl+C), it **stops accepting new work, finishes in-flight work, releases resources, and then exits** — instead of dying instantly and dropping live requests.

Without it, an orchestrator that sends `SIGTERM` during a deploy will kill the process mid-request: clients get connection resets, database transactions are left dangling, and queue messages may be lost.

```js
const server = app.listen(3000);

async function shutdown(signal) {
  console.log(`${signal} received — shutting down gracefully`);

  // 1. Stop accepting new connections; existing ones finish.
  server.close(async () => {
    try {
      // 2. Release downstream resources.
      await mongoose.connection.close(false);
      await redis.quit();
      console.log("clean shutdown complete");
      process.exit(0);
    } catch (err) {
      console.error("error during shutdown", err);
      process.exit(1);
    }
  });

  // 3. Safety net: force-exit if cleanup hangs (e.g. a stuck connection).
  setTimeout(() => {
    console.error("forced shutdown after timeout");
    process.exit(1);
  }, 10_000).unref(); // unref so this timer doesn't itself keep the process alive
}

process.on("SIGTERM", () => shutdown("SIGTERM"));
process.on("SIGINT", () => shutdown("SIGINT"));
```

> BEST PRACTICE: Always include the forced-exit timeout. A "graceful" shutdown that hangs forever on a stuck connection is just a different kind of outage. Bound it (commonly 10–30 s) so the orchestrator's hard `SIGKILL` never has to fire.

> Follow-up: "How does graceful shutdown enable zero-downtime deploys?" (See Q22.)

---

### 8. **What is logging, and what should (and should not) go into application logs?**

Logging is the practice of emitting a durable record of what your service is doing, so you can debug incidents, audit behavior, and feed monitoring. The interview-relevant points are *format*, *levels*, and *what not to log*.

- **Use structured (JSON) logs, not free-form strings.** Structured logs are machine-parseable, so you can search, filter, and aggregate them in a log platform (Datadog, ELK, CloudWatch). Use a real logger — `pino` (fast, JSON-first) or `winston` — never bare `console.log` in production.

```js
const pino = require("pino");
const logger = pino({ level: process.env.LOG_LEVEL || "info" });

// Structured: each field is queryable in your log platform
logger.info({ userId, route: req.path, durationMs: 42 }, "request completed");
logger.error({ err, orderId }, "failed to charge order");
```

- **Use log levels** (`trace` < `debug` < `info` < `warn` < `error` < `fatal`) so you can dial verbosity per environment.
- **Attach a correlation/request ID** to every log line for a request so you can trace one request across services (see Q24).

What must **never** go into logs: passwords, full credit-card numbers, API keys/tokens, session cookies, and personally identifiable information beyond what you're permitted to retain. Logging secrets is one of the most common real-world security incidents.

> WARNING — Pitfall: `console.log` is synchronous to a terminal/file and can block the event loop under high volume; it also produces unstructured output. Use an async, structured logger in production.

> Follow-up: "What's the difference between logging, metrics, and tracing?" (See Q25 — the three pillars of observability.)

---

### 9. **What is idempotency, and which HTTP methods are idempotent?**

An operation is **idempotent** if performing it multiple times has the same effect as performing it once. It does *not* mean the response is byte-identical each time — it means the *server state* ends up the same regardless of how many times the request is replayed.

| Method | Idempotent? | Why |
| --- | --- | --- |
| `GET` | Yes | Reads don't change state |
| `HEAD` | Yes | Reads (headers only) |
| `PUT` | Yes | "Set resource to X" — repeating sets it to X again |
| `DELETE` | Yes | Deleting an already-deleted resource leaves it deleted |
| `POST` | **No** | "Create a new resource" — repeating creates duplicates |
| `PATCH` | Not necessarily | Depends on semantics (`set status=paid` is idempotent; `increment balance` is not) |

This matters enormously in distributed systems because **networks retry**. A client whose request times out doesn't know whether the server processed it, so it retries. If the operation is `POST /payments`, a naive retry charges the customer twice. The fix is an **idempotency key** (Q15): the client sends a unique key, the server records it, and a retry with the same key returns the original result instead of re-executing.

> Follow-up: "How would you make `POST /orders` safely retryable?" (Idempotency keys — see Q15.)

---

### 10. **When would you choose MongoDB over a relational database like PostgreSQL, and vice versa?**

Lead with the honest framing: **the data's shape and access patterns decide, not fashion.** Then give concrete criteria.

| Choose a relational DB (PostgreSQL) when... | Choose MongoDB (document DB) when... |
| --- | --- |
| Data is highly relational with many-to-many joins | Data is document-shaped and mostly read/written as a unit |
| You need multi-row ACID transactions as the norm | Access patterns are denormalized; one read returns one document |
| Strong, enforced schema and constraints matter | Schema evolves rapidly / varies per record |
| Complex ad-hoc queries, reporting, aggregation across tables | High write throughput on self-contained documents |
| Strong consistency is a hard requirement | Flexible horizontal scaling (sharding) is a priority |

Important nuances a strong candidate adds:

- The line has blurred: PostgreSQL has excellent `JSONB` support (document-style storage with indexing), and MongoDB has multi-document ACID transactions since 4.0. So "I need JSON" is no longer an automatic MongoDB win, and "I need transactions" is no longer an automatic disqualifier for MongoDB.
- The real MongoDB design question is **embedding vs referencing** (Q11), driven by access patterns and the "16 MB document limit."
- The real cost of choosing wrong is *migration pain later*, so model around how you will **query** the data, not how it looks on a whiteboard.

> Follow-up: "Given a social app with users, posts, and comments, how would you model it in MongoDB?" (See Q11.)

---

## Intermediate

### 11. **In MongoDB/Mongoose, when do you embed documents vs. reference them? Model a blog with posts and comments.**

The decision is driven by access patterns, cardinality, and growth:

- **Embed** when the child data is *owned by* and *almost always read with* the parent, has bounded size, and doesn't need to be queried independently. Embedding gives you a single read, atomic updates within the document, and locality.
- **Reference** when the child is high-cardinality or unbounded, is queried/updated independently, or is shared across parents. Referencing avoids the 16 MB document cap and unbounded array growth, at the cost of extra queries (or a `$lookup` / `populate`).

For a blog: a post has a bounded set of metadata (author, tags) but a potentially *unbounded* number of comments. So embed small bounded data, reference the unbounded data.

```js
const mongoose = require("mongoose");

const postSchema = new mongoose.Schema({
  title: String,
  body: String,
  // Reference the author (shared, queried independently)
  author: { type: mongoose.Schema.Types.ObjectId, ref: "User", index: true },
  tags: [String],            // embedded: small, bounded, read with the post
  commentCount: Number,      // denormalized counter to avoid counting on every read
});

const commentSchema = new mongoose.Schema({
  // Reference back to the post (comments are unbounded => their own collection)
  post: { type: mongoose.Schema.Types.ObjectId, ref: "Post", index: true },
  author: { type: mongoose.Schema.Types.ObjectId, ref: "User" },
  body: String,
  createdAt: { type: Date, default: Date.now },
});
```

```js
// Fetch a post with author populated, then page its comments separately
const post = await Post.findById(id).populate("author", "name avatarUrl").lean();
const comments = await Comment.find({ post: id })
  .sort({ createdAt: -1 }).limit(20).lean();
```

The general rule of thumb: **"What's read together, stays together — unless it grows without bound."**

> WARNING — Pitfall: Embedding an unbounded array (e.g. all comments inside the post) eventually hits the 16 MB document limit and makes every post read drag the entire comment history. Unbounded growth is the canonical reason to reference instead of embed.

> Follow-up: "How does `populate` work under the hood, and what's its performance cost?" (It runs additional queries — it is not a SQL join; it's N+1 if used carelessly in a loop.)

---

### 12. **Design pagination for a feed endpoint with 10 million rows. Offset-based or cursor-based?**

Both work for small data; at 10M rows the difference is decisive.

**Offset pagination** (`?page=N&limit=20` → `skip(N*limit).limit(limit)`):

```js
// Simple but degrades badly deep into the dataset
const items = await Post.find().sort({ _id: -1 })
  .skip((page - 1) * limit).limit(limit).lean();
```

The problem: `skip(200000)` forces the database to *scan and discard* 200,000 documents to reach the page you want — cost grows linearly with page depth. It also produces **drift**: if rows are inserted/deleted while a user pages, items shift and they see duplicates or gaps.

**Cursor (keyset) pagination** (`?after=<lastId>`): instead of an offset, you carry the sort key of the last item seen and query for rows *after* it:

```js
// Constant-time per page; stable under inserts/deletes
const query = cursor ? { _id: { $lt: cursor } } : {};
const items = await Post.find(query).sort({ _id: -1 }).limit(limit + 1).lean();

const hasMore = items.length > limit;
const page = hasMore ? items.slice(0, limit) : items;
const nextCursor = hasMore ? page[page.length - 1]._id : null;
res.json({ items: page, nextCursor });
```

| | Offset | Cursor |
| --- | --- | --- |
| Deep-page performance | Degrades (O(n) skip) | Constant (index seek) |
| Stable under writes | No (drift) | Yes |
| "Jump to page 50" | Easy | Not supported (sequential only) |
| Total count / page numbers | Easy | Hard |

**Verdict:** for an infinite-scroll feed at scale, cursor pagination — it's fast and stable. Use offset only for small admin tables where "jump to page N" matters and the dataset is modest.

> TIP: The cursor must sort on an indexed, unique, monotonic field (commonly `_id`, or a `(createdAt, _id)` compound for ties). Without the index, cursor pagination is just as slow as offset.

---

### 13. **How do you scale a Node.js application from one process to handle production traffic? Take me through the levels.**

Scale in layers, cheapest and simplest first:

1. **Use all CPU cores on one box (vertical-ish):** Node runs your JS on one thread, so a single process uses one core. Run one process per core with the **`cluster`** module or, in practice, a process manager like **PM2** (`pm2 start app.js -i max`). The OS/primary distributes incoming connections across workers.

```js
const cluster = require("node:cluster");
const os = require("node:os");

if (cluster.isPrimary) {
  for (let i = 0; i < os.availableParallelism(); i++) cluster.fork();
  cluster.on("exit", (worker) => {
    console.log(`worker ${worker.process.pid} died — respawning`);
    cluster.fork(); // keep the pool full
  });
} else {
  require("./server"); // each worker runs the Express app
}
```

2. **Make instances stateless (Q1)** so they're interchangeable — sessions/cache in Redis, files in S3, data in the DB.
3. **Scale horizontally:** run many instances across many machines/containers behind a **load balancer** (Q2). This is the real scaling axis — add boxes, not bigger boxes.
4. **Offload the hot paths:**
   - **Cache reads** in Redis (Q4) to take pressure off the DB.
   - **Move slow/CPU-bound/background work to a queue** (Q17) so request handlers stay fast.
   - **Scale the database** independently: read replicas for read-heavy load, indexes, then sharding.
5. **Add a CDN** in front for static assets and cacheable responses, so most requests never reach Node at all.
6. **Autoscale** instances on a metric (CPU, request latency, queue depth) and observe everything (Q25).

> BEST PRACTICE: Profile before scaling. The most common "we need more servers" turns out to be a missing database index or a blocked event loop. Fix the bottleneck before adding hardware.

> Follow-up: "`cluster` vs `worker_threads` for scaling?" (cluster = multiple processes to handle more *connections*; worker_threads = parallelism for CPU-bound *computation* inside one process.)

---

### 14. **Compare rate-limiting algorithms: fixed window, sliding window, and token bucket.**

| Algorithm | How it works | Pro | Con |
| --- | --- | --- | --- |
| **Fixed window** | Count requests per fixed clock interval (e.g. per minute); reset at the boundary | Trivial to implement (one counter + TTL) | **Boundary burst:** a client can send `max` at 0:59 and `max` again at 1:00 — 2× the limit in 2 seconds |
| **Sliding window log** | Store a timestamp per request; count those within the last window | Accurate, no boundary burst | Memory grows with request volume (one entry per request) |
| **Sliding window counter** | Weighted blend of current + previous fixed window | Good accuracy, cheap | Slight approximation |
| **Token bucket** | Bucket holds N tokens, refills at a steady rate; each request spends one | Allows controlled bursts while bounding the average rate | Slightly more state (tokens + last-refill time) |
| **Leaky bucket** | Requests queue and drain at a fixed rate | Smooths output to a constant rate | Adds latency; queue can overflow |

The two you'll actually defend in an interview:

- **Token bucket** is the most popular for public APIs because it permits short legitimate bursts (a user clicking quickly) while still capping the long-run average. It's also cheap to implement atomically in Redis.
- **Sliding window** when you need a precise "no more than X in any rolling 60 s" guarantee and can't tolerate the fixed-window boundary burst.

Implementing this correctly across instances requires the counter operations to be **atomic** (Redis `INCR` + `EXPIRE`, or a Lua script for token bucket) so concurrent requests can't race past the limit.

> Follow-up: "Why does fixed-window allow 2× the limit at the boundary, and how does sliding window fix it?"

---

### 15. **A payment endpoint must be safely retryable. How do you implement idempotency keys?**

The client generates a unique key (a UUID) per logical operation and sends it in a header. The server uses that key to ensure the operation executes **at most once**, returning the stored result on any replay.

```js
async function createPayment(req, res) {
  const key = req.header("Idempotency-Key");
  if (!key) return res.status(400).json({ error: "Idempotency-Key required" });

  // Atomically claim the key. SET NX = set only if it doesn't already exist.
  const claimed = await redis.set(`idem:${key}`, "in-progress", { NX: true, EX: 86400 });

  if (!claimed) {
    // Key already seen. Either still running, or completed.
    const stored = await redis.get(`idem:${key}`);
    if (stored === "in-progress") {
      return res.status(409).json({ error: "request in progress, retry shortly" });
    }
    return res.status(200).json(JSON.parse(stored)); // replay the original result
  }

  // First time we've seen this key — actually do the work.
  const result = await chargeCard(req.body); // the non-idempotent side effect
  await redis.set(`idem:${key}`, JSON.stringify(result), { EX: 86400 }); // store result
  return res.status(201).json(result);
}
```

The critical details:

- **Claim the key atomically before doing the work** (`SET NX`), so two concurrent retries can't both charge the card. This is the whole game — a check-then-act without atomicity has a race.
- **Store the response** so a later retry returns the same body/status instead of re-executing.
- **Set a TTL** (e.g. 24 h) so keys don't accumulate forever; the window must exceed the client's maximum retry horizon.
- For true correctness, the side effect and the key record should be **transactionally consistent** — ideally the idempotency record lives in the same database as the side effect (e.g. a unique constraint on the key column in PostgreSQL), so a crash between "charge" and "store result" can't lose the linkage. Redis is fine when paired with the downstream provider's own idempotency support (Stripe, for instance, accepts an idempotency key too).

> Follow-up: "What if the process crashes after charging but before storing the result?" (You need the payment provider's idempotency or a DB unique constraint so the replay detects the existing charge — Redis alone leaves a gap.)

---

### 16. **Now handle file uploads at scale — thousands of concurrent 5 GB video uploads. What changes?**

The small-scale approach (Q6 — stream bytes *through* the Node process) collapses at this scale: every upload occupies a connection and memory/CPU on your app servers for the entire transfer, so a few hundred concurrent uploads can saturate the fleet. The fix is to **take the application server out of the data path** using **presigned URLs** (a.k.a. direct-to-storage uploads).

The flow:

1. The client asks your API for permission to upload. The API authorizes the request and returns a short-lived **presigned URL** (or presigned POST policy) for S3/GCS — signed with your credentials but usable by the client directly.
2. The client uploads the bytes **straight to object storage**, not through your server. Your Node process never touches the gigabytes.
3. Object storage notifies your backend (S3 event → SQS/Lambda/webhook) that the object landed, and you record metadata / kick off processing.

```js
// Step 1: your API issues a presigned PUT URL (AWS SDK v3)
const { S3Client, PutObjectCommand } = require("@aws-sdk/client-s3");
const { getSignedUrl } = require("@aws-sdk/s3-request-presigner");
const s3 = new S3Client({ region: "us-east-1" });

app.post("/uploads/presign", authRequired, async (req, res) => {
  const key = `videos/${req.user.id}/${crypto.randomUUID()}.mp4`;
  const command = new PutObjectCommand({
    Bucket: "my-uploads",
    Key: key,
    ContentType: "video/mp4",
  });
  // URL is valid for 5 minutes — bounds the window of exposure
  const url = await getSignedUrl(s3, command, { expiresIn: 300 });
  res.json({ uploadUrl: url, key });
});
```

Additional scale considerations:

- **Multipart upload** for large files so the client can upload chunks in parallel and resume after a network blip.
- **Post-upload processing** (transcoding, virus scan, thumbnailing) runs **asynchronously via a queue** (Q17) — it's slow and CPU-bound, so it must not block a request.
- **Validate after the fact**: since bytes bypass your server, validate the stored object (size, magic bytes, content scan) in the processing step, and only mark the upload "ready" once it passes.

> BEST PRACTICE: For anything beyond small images, prefer presigned direct-to-storage uploads. Streaming large files through your application tier wastes its scarcest resources (connections, memory, bandwidth) on work that storage is purpose-built to do.

> Follow-up: "How do you prevent a client from uploading something other than what they declared?" (Constrain content-type/size in the presigned policy, then re-validate the object server-side.)

---

### 17. **Why and how would you use a background job queue? Walk through a concrete example.**

A queue decouples *accepting* work from *doing* work. The request handler enqueues a job and returns immediately; one or more **worker** processes consume jobs and do the slow part asynchronously. This keeps the API fast and the event loop unblocked.

Use a queue whenever work is **slow, CPU-bound, can fail/retry, or doesn't need to complete within the request**: sending email, generating PDFs/reports, image/video processing, calling flaky third-party APIs, fan-out notifications, scheduled tasks.

In the Node/Redis world, **BullMQ** is the standard. The pattern is producer (API) + consumer (worker):

```js
// queue.js — shared definition
const { Queue } = require("bullmq");
const connection = { host: "localhost", port: 6379 };
const emailQueue = new Queue("email", { connection });
module.exports = { emailQueue, connection };
```

```js
// In the request handler (producer): enqueue and return fast
app.post("/signup", async (req, res) => {
  const user = await User.create(req.body);
  await emailQueue.add(
    "welcome",
    { userId: user.id, email: user.email },
    { attempts: 5, backoff: { type: "exponential", delay: 1000 } } // retry policy
  );
  res.status(201).json(user); // responds immediately; email sends in the background
});
```

```js
// worker.js — a SEPARATE process (scale independently of the API)
const { Worker } = require("bullmq");
const { connection } = require("./queue");

new Worker("email", async (job) => {
  await sendEmail(job.data.email, "Welcome!"); // slow / flaky => retried on failure
}, { connection, concurrency: 10 }); // process 10 jobs at a time
```

What a strong answer emphasizes:

- **Run workers as separate processes** so they scale independently and a heavy job never steals CPU from request handlers.
- **Retries with backoff** turn transient failures (a flaky SMTP server) into eventual success without manual intervention.
- **A dead-letter queue / failed-job handling** for jobs that exhaust retries, so failures are visible, not silent.
- **Jobs should be idempotent** (Q9/Q15) because at-least-once delivery means a job can run more than once (e.g. worker crashes after sending the email but before marking it done).

> WARNING — Pitfall: Almost all queues guarantee *at-least-once*, not *exactly-once*, delivery. Design every job handler to tolerate being run twice (idempotency), or you'll send duplicate emails / double-charge customers.

> Follow-up: "How do you handle a job that keeps failing forever?" (Cap attempts → dead-letter queue → alert.) "How would you schedule a recurring job?" (BullMQ repeatable jobs / a cron-style scheduler.)

---

### 18. **Design the backend for a real-time feature — say, live chat or presence. What's your approach?**

First clarify the requirement: how many concurrent connections, do messages need history/persistence, is it broadcast or point-to-point, what latency budget? Then choose a **transport**:

| Option | Direction | Use when |
| --- | --- | --- |
| **WebSockets** (e.g. `ws`, Socket.IO) | Full-duplex, bidirectional | Chat, multiplayer, collaborative editing — clients both send and receive frequently |
| **Server-Sent Events (SSE)** | Server → client only | Live feeds, notifications, dashboards — server pushes, client just listens |
| **Long polling** | Simulated push over HTTP | Fallback where WebSockets are blocked |

For chat, WebSockets. The architecture challenge appears the moment you scale past one instance: WebSocket connections are **stateful and long-lived**, so a user connected to instance A won't receive a message published on instance B. The standard solution is a **pub/sub backplane** (Redis Pub/Sub, or Redis adapter for Socket.IO):

```js
const { Server } = require("socket.io");
const { createAdapter } = require("@socket.io/redis-adapter");
const { createClient } = require("redis");

const io = new Server(httpServer);
const pubClient = createClient({ url: process.env.REDIS_URL });
const subClient = pubClient.duplicate();
await Promise.all([pubClient.connect(), subClient.connect()]);

// The adapter relays events across ALL instances via Redis pub/sub,
// so a message emitted on instance A reaches sockets on instance B/C.
io.adapter(createAdapter(pubClient, subClient));

io.on("connection", (socket) => {
  socket.on("chat:message", (msg) => {
    io.to(msg.roomId).emit("chat:message", msg); // fan out across the cluster
  });
});
```

Other real-time concerns to raise:

- **Sticky sessions or a connection-aware LB** — because WebSocket connections are long-lived, the LB must keep a client pinned to the instance holding its socket (the Redis adapter handles cross-instance *messaging*, but the socket itself lives on one box).
- **Authentication on connect**, then authorize each subscription (don't let a client join arbitrary rooms).
- **Persistence is separate from transport** — if chat history must survive, write messages to the DB; the socket layer is just delivery.
- **Backpressure / slow consumers** — buffer limits so one slow client can't balloon memory.

> Follow-up: "WebSockets vs SSE for a notifications feed?" (SSE — it's server→client only, auto-reconnects, runs over plain HTTP, and is far simpler than WebSockets when the client never needs to push.)

---

### 19. **How would you structure error handling across an entire Express service?**

The goal is **one consistent path** for turning any failure into a correct HTTP response, with no leaked stack traces and no duplicated try/catch sprawl. Four pieces:

**1. A custom error class** that carries an HTTP status and a machine-readable code, distinguishing *operational* errors (expected: validation, not-found, auth) from *programmer* errors (bugs):

```js
class AppError extends Error {
  constructor(message, statusCode, code) {
    super(message);
    this.statusCode = statusCode;     // e.g. 404
    this.code = code;                 // e.g. "POST_NOT_FOUND"
    this.isOperational = true;        // expected error, safe to show the client
    Error.captureStackTrace(this, this.constructor);
  }
}
```

**2. Throw it in handlers; let a central middleware format it.** In **Express 5**, async errors thrown in handlers are forwarded to error middleware automatically. In **Express 4** they are *not* — you must `next(err)` or wrap handlers with a `catchAsync` helper:

```js
// Express 4: wrapper needed so a rejected promise reaches the error handler
const catchAsync = (fn) => (req, res, next) => fn(req, res, next).catch(next);

app.get("/posts/:id", catchAsync(async (req, res) => {
  const post = await Post.findById(req.params.id);
  if (!post) throw new AppError("Post not found", 404, "POST_NOT_FOUND");
  res.json(post);
}));
// Express 5: the wrapper is optional — thrown async errors propagate to error middleware.
```

**3. A single error-handling middleware** (4 args — Express identifies it by arity) registered last:

```js
app.use((err, req, res, next) => {
  const isOperational = err.isOperational === true;
  const status = err.statusCode || 500;

  // Log every error with context; log unexpected ones loudly.
  req.log?.error({ err, code: err.code }, "request failed");

  res.status(status).json({
    error: {
      code: err.code || "INTERNAL_ERROR",
      // Never leak internals for non-operational (bug) errors.
      message: isOperational ? err.message : "Something went wrong",
    },
  });
});
```

**4. Process-level safety nets** for what slips through: log `unhandledRejection` and `uncaughtException`, and on an uncaught exception **let the process crash and be restarted** (the process is in an unknown state — limping on is more dangerous than restarting).

```js
process.on("unhandledRejection", (reason) => { logger.error({ reason }, "unhandledRejection"); });
process.on("uncaughtException", (err) => {
  logger.fatal({ err }, "uncaughtException — exiting");
  process.exit(1); // let the orchestrator restart a clean process
});
```

> BEST PRACTICE: Distinguish operational from programmer errors. Operational errors are handled and returned to the client; programmer errors (a real bug) should fail fast and crash-restart rather than being swallowed.

> Follow-up: "Why crash on `uncaughtException` instead of recovering?" (After an uncaught throw, app state and resources may be corrupt; continuing risks subtle data corruption. Restart from a known-good state.)

---

### 20. **How do you handle cache invalidation? Compare the main strategies.**

Cache invalidation is famously hard because the cache and the source of truth can disagree. The strategies, roughly from simplest to strongest:

| Strategy | How | Trade-off |
| --- | --- | --- |
| **TTL expiry** | Set an expiry; entries self-evict | Dead simple; tolerates bounded staleness; no write-path coupling |
| **Write-through** | On write, update DB *and* cache together | Cache always fresh on the written key; adds write latency, more code |
| **Write-around** | Write to DB only; cache fills lazily on next read | Avoids caching write-only data; first read after a write is a miss |
| **Write-back** | Write to cache, flush to DB asynchronously | Fast writes; risk of data loss if cache dies before flush |
| **Explicit invalidation (delete on write)** | On update/delete, `DEL` the affected key(s) | Precise; the hard part is knowing *every* key affected |
| **Versioned / tagged keys** | Bump a version namespace to invalidate a whole group at once | Clean group invalidation without enumerating keys |

A pragmatic real-world combination: **cache-aside reads + delete-on-write + a safety-net TTL**:

```js
async function updateProduct(id, patch) {
  const product = await Product.findByIdAndUpdate(id, patch, { new: true });
  await redis.del(`product:${id}`);        // invalidate the specific key
  await redis.del(`products:list`);        // invalidate any derived/list caches it affects
  return product;
}
```

The genuinely hard parts to call out:

- **Knowing the full blast radius** of a write — a product change may invalidate the product key *and* category lists, search results, and homepage modules. Missing one serves stale data.
- **Derived/aggregate caches** (counts, top-10 lists) are easy to forget.
- **TTL as the backstop:** even with explicit invalidation, keep a TTL so a missed `DEL` self-heals instead of serving stale data forever.

> TIP: "There are only two hard things in computer science: cache invalidation and naming things." The practical defense is to prefer short TTLs and explicit deletes on known keys, and treat invalidation correctness as a first-class part of every write path.

> Follow-up: "When would write-through beat cache-aside?" (When you can't tolerate the first-read-after-write miss and writes are infrequent relative to reads.)

---

## Advanced

### 21. **Explain the cache stampede (thundering herd) problem and how to prevent it.**

A **cache stampede** happens when a hot cache key expires (or the cache is cold) and a flood of concurrent requests all miss simultaneously, so they *all* hit the database at once to recompute the same value. The DB sees a sudden spike of identical expensive queries — exactly what the cache existed to prevent — and can topple over, which is especially dangerous right after a cache restart or for a very popular key.

Mitigations, often combined:

**1. Locking / request coalescing (single-flight):** the first miss acquires a short lock, recomputes, and populates the cache; everyone else waits briefly or serves slightly stale data. Only one DB query runs.

```js
async function getWithLock(key, ttl, loader) {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  // Try to become the single recomputing request.
  const gotLock = await redis.set(`lock:${key}`, "1", { NX: true, EX: 10 });
  if (gotLock) {
    try {
      const value = await loader();                 // the one DB hit
      await redis.set(key, JSON.stringify(value), { EX: ttl });
      return value;
    } finally {
      await redis.del(`lock:${key}`);
    }
  }

  // Lost the race: wait briefly and read the now-populated cache.
  await new Promise((r) => setTimeout(r, 50));
  return getWithLock(key, ttl, loader);
}
```

**2. Probabilistic early expiration:** recompute *before* the TTL ends, with a probability that rises as expiry approaches, so a single request refreshes the key while others still get cache hits.

**3. Stale-while-revalidate:** serve the stale value immediately and refresh it in the background, so users never wait on a recompute and the DB sees one refresh, not thousands.

**4. Jitter the TTLs:** if many keys are set at once (e.g. after a deploy), add random jitter to their expiries so they don't all expire on the same tick.

> WARNING — Pitfall: Setting the same TTL on a batch of popular keys causes a synchronized mass expiry — a stampede on every key at once. Always add jitter (`EX: base + random()`).

> Follow-up: "What's the difference between a stampede and a cache penetration/avalanche?" (Penetration = repeated misses for keys that *don't exist*, mitigated with negative caching; avalanche = many keys expiring together, mitigated with jitter.)

---

### 22. **Describe how you'd achieve zero-downtime deployments for a Node service.**

Zero-downtime means rolling out a new version with no dropped requests and no maintenance window. It rests on two foundations from earlier questions: **stateless instances** (Q1) and **graceful shutdown** (Q7). The mechanism is a **rolling deploy** coordinated with the load balancer:

1. **Health checks.** Each instance exposes a readiness probe (`GET /readyz`) that reports "ready" only when it can actually serve (DB connected, migrations applied, caches warm). The LB routes only to ready instances.
2. **Roll one (or a batch) at a time.** Start a new-version instance; wait for it to pass readiness; then take an old instance out of rotation.
3. **Drain connections gracefully.** Before stopping an old instance, the orchestrator stops sending it new requests, then sends `SIGTERM`. Graceful shutdown (Q7) lets in-flight requests finish before exit. A common subtlety: keep serving for a few seconds after `SIGTERM` because the LB may not have noticed the instance is leaving yet (the "fail readiness first, then sleep, then close" pattern).
4. **Repeat** until every instance runs the new version.

```js
// Readiness flips to false on SIGTERM so the LB drains this instance BEFORE we close.
let ready = true;
app.get("/readyz", (req, res) => res.status(ready ? 200 : 503).send());

process.on("SIGTERM", async () => {
  ready = false;                              // 1. LB stops routing new requests here
  await new Promise((r) => setTimeout(r, 5000)); // 2. let in-flight + LB lag settle
  server.close(async () => {                  // 3. finish remaining requests, then exit
    await cleanup();
    process.exit(0);
  });
});
```

Deployment strategies to name:

- **Rolling** — replace instances incrementally (default for most orchestrators).
- **Blue-green** — stand up a full parallel environment (green), switch the LB over atomically, keep blue for instant rollback.
- **Canary** — route a small % of traffic to the new version, watch error rates/latency, then ramp up.

The often-forgotten constraint: **backward-compatible database migrations.** During a rolling deploy, old and new code run simultaneously against the same schema. So migrations must be **expand/contract** — add the new column/field first (both versions work), deploy code that uses it, *then* remove the old one in a later release. A destructive migration deployed alongside code is the classic way "zero-downtime" turns into an outage.

> BEST PRACTICE: Separate *readiness* (can I serve traffic now?) from *liveness* (am I alive, or should I be restarted?). Flipping readiness to false on `SIGTERM` is what lets the load balancer drain an instance cleanly before it shuts down.

> Follow-up: "How do you roll back a bad deploy instantly?" (Blue-green: flip the LB back to the old environment. Canary: stop the ramp and shift traffic back.)

---

### 23. **An endpoint that was fast is now timing out under load, while CPU sits near 100% on one core. Diagnose it.**

This is the signature of a **blocked event loop** — synchronous, CPU-bound work running on the main thread, freezing *all* request processing (Node runs your JS on one thread). The single-core saturation is the tell: a true I/O-wait problem wouldn't peg a core.

Diagnostic approach, narrating the reasoning:

1. **Confirm event-loop lag.** Measure with `perf_hooks.monitorEventLoopDelay` or a library like `event-loop-lag`. If p99 lag is hundreds of ms, the loop is being blocked.

```js
const { monitorEventLoopDelay } = require("node:perf_hooks");
const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();
setInterval(() => {
  console.log("event loop p99 (ms):", (h.percentile(99) / 1e6).toFixed(1));
}, 5000);
```

2. **Find the hot function.** Capture a CPU profile (`node --prof`, `clinic flame`, or `--cpu-prof`) and read the flame graph. The widest frame is the culprit.
3. **Look for the usual suspects:** a giant synchronous `JSON.parse`/`JSON.stringify`, a regex with catastrophic backtracking (ReDoS) on user input, synchronous crypto/compression on the hot path (`crypto.pbkdf2Sync`, `zlib.gzipSync`), an O(n²) loop over a now-large dataset, or a `*Sync` filesystem call inside a handler.
4. **Fix by getting the work off the main thread:** move CPU-bound work to `worker_threads` or a job queue (Q17), replace sync APIs with async ones, paginate/stream large data instead of materializing it, and bound user input (regex timeouts, size limits).

```js
// Before: blocks the loop for every request that parses a big payload
const data = JSON.parse(hugeString);

// After: heavy parse/transform delegated to a worker thread
const { Worker } = require("node:worker_threads");
const result = await runInWorker("./parse-worker.js", hugeString);
```

> WARNING — Pitfall: A regular expression with nested quantifiers (e.g. `(a+)+$`) against attacker-controlled input can lock a core for seconds — a ReDoS denial-of-service. Treat user-supplied input to regexes as hostile; prefer linear-time validators.

> Follow-up: "Why does this only show up under load?" (A 50 ms sync block is invisible at low traffic but, serialized across thousands of requests on one thread, becomes a queue of compounding delays — the loop never catches up.)

---

### 24. **Requests intermittently fail across your microservices and you can't tell where. How do you make a distributed system debuggable?**

The core problem is **lack of correlation** — without a thread connecting logs across services, an error in service C is impossible to tie back to the originating request in service A. The fixes are the observability practices:

**1. Correlation / trace IDs.** Generate a unique request ID at the edge (or accept an incoming `X-Request-Id` / W3C `traceparent`), attach it to every log line, and **propagate it** on every downstream call. Now you can grep one ID and see the request's full journey.

```js
const { randomUUID } = require("node:crypto");

app.use((req, res, next) => {
  req.id = req.header("x-request-id") || randomUUID();
  res.setHeader("x-request-id", req.id);
  // Bind a child logger so every log in this request carries the id automatically
  req.log = logger.child({ requestId: req.id });
  next();
});

// Propagate downstream so the next service logs under the same id
await fetch(downstreamUrl, { headers: { "x-request-id": req.id } });
```

> TIP: `AsyncLocalStorage` lets you carry the request ID through deep async call chains without threading it through every function signature — store it once in middleware, read it anywhere during the request.

**2. Distributed tracing.** Adopt **OpenTelemetry** to emit spans per operation across services. A trace stitches spans by trace ID into a timeline, so you *see* which service/span is slow or erroring — turning "somewhere it's slow" into "the auth→billing call takes 2 s."

**3. Metrics + alerting.** Export RED metrics (Rate, Errors, Duration) per endpoint (Prometheus/StatsD). Alert on error-rate and latency-percentile thresholds so you learn about failures before users report them. Watch the **golden signals**: latency, traffic, errors, saturation.

**4. Health checks and dashboards** so you can see, at a glance, which service is degraded.

> BEST PRACTICE: Make the request ID the spine of your observability. Logs (with the id), metrics (aggregated), and traces (the id's timeline) are the "three pillars" — together they answer *what* broke, *how often*, and *where*.

> Follow-up: "What are the three pillars of observability?" (Logs, metrics, traces — see Q25.)

---

### 25. **What is observability, and how do logs, metrics, and traces differ? When do you reach for each?**

**Monitoring** answers known questions ("is CPU high?"). **Observability** is the property that lets you ask *new, unanticipated* questions about a running system from its outputs — crucial because production fails in ways you didn't predict. It rests on three complementary signal types:

| Pillar | What it is | Answers | Cost / cardinality |
| --- | --- | --- | --- |
| **Logs** | Discrete, timestamped events (ideally structured JSON) | "What exactly happened in *this* request?" | High volume; expensive at scale |
| **Metrics** | Numeric time-series, aggregated (counters, gauges, histograms) | "How often / how fast, overall, over time?" | Cheap; low cardinality |
| **Traces** | Causally-linked spans across services for one request | "*Where* in the call graph did time/errors go?" | Often sampled |

How they work together during an incident:

1. A **metric** alert fires — error rate on `/checkout` spiked, or p99 latency crossed the SLO. (Metrics tell you *that* something is wrong, cheaply and in aggregate.)
2. You open a **trace** for a failing request to see *which span* — which service or DB call — is slow or erroring. (Traces localize it.)
3. You read the **logs** for that exact span/request (joined by trace/request ID) to see the precise error and context. (Logs give the detail.)

The key insight to articulate: they're not redundant — they sit at different points on the volume/detail axis. Metrics are cheap and aggregate (good for dashboards/alerts but can't explain a single request); logs are detailed but expensive at volume; traces show causality across service boundaries. A mature service emits all three, tied together by a correlation ID, and uses **sampling** on high-volume traces/logs to control cost.

> BEST PRACTICE: Instrument for the questions you'll ask at 3 a.m. during an incident, not just green/red dashboards. Structured logs + RED metrics + sampled traces, all sharing a request ID, is the baseline for a service you can actually debug in production.

> Follow-up: "What are the RED and USE methods?" (RED = Rate, Errors, Duration per service/endpoint; USE = Utilization, Saturation, Errors per resource.)

---

### 26. **Design a URL shortener (like bit.ly). Walk through the API, data model, and how it scales.**

A favorite design question because it touches every layer. Start by clarifying scale and read/write ratio: a URL shortener is **massively read-heavy** (billions of redirects, far fewer creations) and redirects must be **very low latency**.

**API:**

| Endpoint | Purpose | Status |
| --- | --- | --- |
| `POST /urls` | Create a short code for a long URL | `201` (returns `{ shortCode, shortUrl }`) |
| `GET /:code` | Redirect to the long URL | `301`/`302` → `Location` |
| `GET /urls/:code/stats` | Analytics (clicks, referrers) | `200` |

**Generating the short code** — the central design choice:

- **Counter + base62 encoding:** maintain a global incrementing counter, encode it in base62 (`[0-9a-zA-Z]`) to get a short, collision-free code. 7 chars of base62 ≈ 3.5 trillion combinations. The counter must be distributed (e.g. a Redis `INCR`, or ranges handed out to each instance) to avoid a single bottleneck.
- **Random/hash code with collision check:** generate random base62, check uniqueness via a unique index, retry on the rare collision. Simpler but needs a uniqueness check per write.

```js
const ALPHABET = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
function base62(n) {
  let s = "";
  do { s = ALPHABET[n % 62n] + s; n /= 62n; } while (n > 0n);
  return s;
}
// A distributed counter (Redis INCR) gives a unique number per create — encode it.
const id = await redis.incr("url:counter");
const code = base62(BigInt(id));
```

**Data model** (simple key→value, perfect for a KV/document store):

```js
{ shortCode: "aZ3xK9", longUrl: "https://...", createdBy, createdAt, clicks: 0, expiresAt }
```

**Scaling the read path (the whole point):**

- **Cache aggressively.** The redirect lookup (`code → longUrl`) is immutable once created, so cache it in Redis with a long TTL. Most redirects never touch the DB.
- **CDN / edge** the redirects where possible.
- **301 vs 302:** `301` (permanent) lets browsers/CDNs cache the redirect, cutting traffic to your service — but then you can't reliably count clicks or change the target. `302` (temporary) keeps requests flowing to you for analytics. Choose per requirement; many shorteners use `302` precisely to retain control and analytics.
- **Analytics asynchronously:** don't write click stats synchronously on the redirect's hot path — fire an event to a queue (Q17) and aggregate offline, so the redirect stays sub-millisecond.

> TIP: This question is really testing read-path optimization. The winning moves are: immutable lookups → cache/CDN them, and click analytics → offload to a queue so they never slow the redirect.

> Follow-up: "Why might you avoid 301 even though it's more cacheable?" (You lose click analytics and the ability to update/expire the target, because clients cache it permanently.)

---

### 27. **Your MongoDB-backed API gets slow as data grows, even though the queries "look fine." How do you investigate and fix it?**

The overwhelmingly common cause is **missing indexes** → collection scans that grow linearly with data, invisible at small scale and crippling at large. Investigate methodically:

1. **Find the slow queries.** Enable the MongoDB profiler / slow-query log, or instrument query duration in the app. Identify the actual offenders rather than guessing.
2. **`explain("executionStats")` the query.** The smoking gun is `COLLSCAN` in the winning plan and `totalDocsExamined ≫ nReturned` (you scanned a million docs to return ten).

```js
const plan = await Post.find({ author: id, status: "published" })
  .sort({ createdAt: -1 }).explain("executionStats");
// stage: "COLLSCAN" and docsExamined ~= collection size => no usable index
```

3. **Add the right index**, matching the query's equality/sort/range shape (the **ESR rule**: Equality fields, then Sort fields, then Range fields in a compound index):

```js
// Supports filtering by author+status AND sorting by createdAt with one index
postSchema.index({ author: 1, status: 1, createdAt: -1 });
```

4. **Re-`explain`** and confirm the stage is now `IXSCAN` and `docsExamined ≈ nReturned`.

Other causes a strong candidate enumerates beyond indexes:

- **N+1 queries** from `populate` in a loop, or fetching related docs one-by-one — batch them.
- **Returning huge documents/arrays** — project only needed fields (`.select()`), and use `.lean()` to skip Mongoose hydration overhead on read paths.
- **Unbounded queries** (no `limit`) that return ever-growing result sets — paginate (Q12).
- **Index that doesn't fit RAM** or the wrong index chosen — check `explain`'s rejected plans; consider a covering index.
- **Write contention / lock**, or a hot shard key if sharded.

```js
// Read-path hygiene: project, lean, and limit
const posts = await Post.find({ author: id })
  .select("title createdAt")  // fetch only needed fields
  .sort({ createdAt: -1 }).limit(20)
  .lean();                    // plain objects, no Mongoose document overhead
```

> WARNING — Pitfall: Indexes speed reads but **slow writes** and consume RAM/disk — every write must update every relevant index. Don't index everything; index for your actual query patterns, and drop unused indexes.

> Follow-up: "Why must a compound index's field order match the query?" (A `{a, b}` index serves queries on `a` and `a+b`, but not `b` alone — the prefix rule. Order it by the ESR rule.)

---

### 28. **A deploy went out and error rates spiked. You have ~5 minutes. Walk me through your incident response.**

The interviewer wants to see a calm, prioritized process — **mitigate first, root-cause second**:

1. **Stop the bleeding before diagnosing.** If the spike correlates with the deploy, **roll back immediately** (or shift traffic off the new version if blue-green/canary). Restoring service is the priority; understanding *why* can wait until users aren't impacted. This is the single most important instinct.
2. **Confirm scope with metrics.** Which endpoints, which error codes (5xx vs 4xx), what % of traffic, which region/instances? This tells you whether it's total or partial, and where.
3. **Open a trace + logs for a failing request** (by request ID, Q24) to see the actual error — a stack trace, a downstream timeout, a connection-pool exhaustion, a bad migration.
4. **Check dependencies.** Is the database/Redis/third-party API healthy, or did the new code change how it's called (e.g. a missing index causing slow queries, or a connection leak)? Often the "deploy" isn't the code but a config/env change.
5. **Communicate.** Post status to the incident channel: what's affected, what you're doing, ETA. Keeping stakeholders informed is part of the job.
6. **After mitigation: root cause and a blameless postmortem.** What broke, why it wasn't caught (missing test? no canary?), and concrete prevention (add the test, add a canary stage, add an alert).

Concrete failure modes this scenario often hides: an uncaught exception crashing workers in a restart loop, a connection-pool/file-descriptor leak that degrades over minutes, a non-backward-compatible DB migration breaking the still-running old version (Q22), or a config/secret that's wrong in prod only.

> BEST PRACTICE: "Mitigate, then investigate." A fast rollback that restores service, followed by an unhurried root-cause analysis, beats heroically debugging in production while users suffer. Make rollback a one-command, well-rehearsed operation.

> Follow-up: "What would you add so this is caught *before* prod next time?" (Canary deploys with automatic rollback on error-rate regression, smoke tests post-deploy, backward-compatible migrations, and alerting on the golden signals.)

---

### 29. **How do you keep a Node service secure at the architecture level? Hit the high-value items.**

An interviewer wants the high-leverage defenses, not an exhaustive checklist. Group them:

**Input & data:**
- **Validate and sanitize all input** at the boundary (`zod`, `joi`, `express-validator`). Treat everything from the client as hostile.
- **Prevent injection.** Use parameterized queries / the ODM's query builder; never string-concatenate user input into queries. With Mongoose, guard against **NoSQL operator injection** (a body like `{ "$gt": "" }` slipping into a query) — cast/validate types and sanitize keys.
- **Set body size limits** (`express.json({ limit: "100kb" })`) to blunt payload-based DoS.

**AuthN / AuthZ:**
- **Authentication ≠ authorization.** Verify identity (JWT/session), then check permissions on *every* protected resource — the most common real bug is a valid user accessing another user's data (IDOR/broken object-level authorization).
- **Store passwords with `bcrypt`/`argon2`**, never plaintext or fast hashes. Keep JWT secrets/keys out of code (env/secret manager), with short token lifetimes + refresh.

**Transport & headers:**
- **HTTPS everywhere**, and set security headers with **`helmet`** (CSP, HSTS, `X-Content-Type-Options`, etc.).
- **CORS** locked to known origins, not `*`.

**Dependencies & ops:**
- **Audit dependencies** (`npm audit`, Dependabot) — most Node CVEs come from the dependency tree, not your code.
- **Least privilege** for DB users and cloud roles; run the process as a non-root user.
- **Don't leak internals** in errors/logs (Q8, Q19); rate-limit auth endpoints (Q5) against brute force.

```js
const helmet = require("helmet");
app.use(helmet());                                   // sane security headers
app.use(express.json({ limit: "100kb" }));            // bound request body size
app.use("/auth/login", rateLimit({ windowMs: 900_000, max: 5 })); // throttle brute force
```

> WARNING — Pitfall: Broken **object-level authorization** (e.g. `GET /orders/:id` returning any order to any logged-in user because you checked *authentication* but not *ownership*) is one of the most common and damaging real-world API vulnerabilities. Always scope queries to the requester: `Order.findOne({ _id: id, userId: req.user.id })`.

> Follow-up: "What's the difference between authentication and authorization?" / "How do you guard against NoSQL injection specifically in Mongoose?"

---

### 30. **A request makes several writes that must all succeed or all fail (e.g. transfer money, decrement inventory on checkout). How do you handle this in MongoDB, and across services?**

**Within a single MongoDB deployment** (replica set / sharded cluster), use **multi-document ACID transactions** (available since MongoDB 4.0 / 4.2 for sharded). All writes in the session commit together or abort together:

```js
const session = await mongoose.startSession();
try {
  await session.withTransaction(async () => {
    // Decrement inventory only if enough is in stock — conditional update.
    const res = await Product.updateOne(
      { _id: productId, stock: { $gte: qty } },
      { $inc: { stock: -qty } },
      { session }
    );
    if (res.modifiedCount === 0) throw new Error("OUT_OF_STOCK"); // aborts the txn
    await Order.create([{ productId, qty, userId }], { session });
  });
} finally {
  await session.endSession();
}
```

Key points: pass the `session` to *every* operation in the transaction; a thrown error rolls everything back; and a **conditional update** (`stock: { $gte: qty }` + checking `modifiedCount`) handles the race where two checkouts hit the last item — only one update matches. Note transactions add overhead and shouldn't be the default for single-document writes (those are already atomic in MongoDB).

**Across multiple services / databases**, a single ACID transaction is impossible (no shared transaction coordinator). Two patterns:

- **Saga pattern:** model the workflow as a sequence of local transactions, each with a **compensating action** to undo it if a later step fails (charge card → if reservation fails, refund). Eventual consistency, no global lock. Orchestrated (a coordinator drives steps) or choreographed (services react to events).
- **Transactional outbox:** to publish an event *and* commit a DB change atomically, write the event to an `outbox` table in the **same transaction** as the state change, then a separate relay publishes outbox rows to the queue. This avoids the dual-write problem (DB commit succeeds but the message publish fails, or vice versa).

> WARNING — Pitfall: The **dual-write problem** — writing to the DB and then publishing to a queue as two separate steps — is not atomic. A crash between them leaves them inconsistent (order saved but no "order placed" event, or an event with no order). The transactional outbox pattern fixes this.

> Follow-up: "When would you avoid MongoDB transactions?" (When you can redesign so the atomic unit is a single document — single-doc writes are atomic and far cheaper than multi-doc transactions.)

---

### 31. **(Output/behavior prediction) A handler does heavy work, then responds. Predict the behavior under concurrent load and fix it.**

```js
const crypto = require("node:crypto");

app.get("/report", (req, res) => {
  // Synchronous, CPU-bound: blocks the event loop for the whole computation.
  const hash = crypto.pbkdf2Sync(req.query.seed, "salt", 1_000_000, 64, "sha512");
  res.json({ hash: hash.toString("hex") });
});
```

**Prediction:** A single request returns correctly but slowly. Under concurrent load it falls apart: because `pbkdf2Sync` runs on the **single main thread**, while one `/report` is computing, **every other request to the entire server is blocked** — health checks, fast endpoints, everything queues behind it. Throughput collapses to "one heavy request at a time," latency for unrelated endpoints spikes, and the load balancer may start failing health checks and pulling the instance out of rotation. CPU pegs one core (Q23).

**Fix:** use the **async** variant, which offloads the work to the **libuv thread pool** so the event loop stays free to handle other requests concurrently (up to the pool size, default 4, tunable via `UV_THREADPOOL_SIZE`):

```js
const crypto = require("node:crypto");
const { promisify } = require("node:util");
const pbkdf2 = promisify(crypto.pbkdf2);

app.get("/report", async (req, res, next) => {
  try {
    // Async: runs on the libuv thread pool, event loop stays responsive.
    const hash = await pbkdf2(req.query.seed, "salt", 1_000_000, 64, "sha512");
    res.json({ hash: hash.toString("hex") });
  } catch (err) {
    next(err);
  }
});
```

For *pure-JS* CPU work (not a built-in that already uses the thread pool, like a custom O(n²) computation), the async variant doesn't exist — there you reach for **`worker_threads`** or a **job queue** (Q17) to get it off the main thread entirely.

> TIP: The async crypto/zlib/fs functions use the libuv thread pool (size 4 by default), so they parallelize only up to that pool size. If many heavy crypto requests arrive at once, raise `UV_THREADPOOL_SIZE` or move the work to dedicated worker threads / a queue.

> Follow-up: "Does the async version make it infinitely concurrent?" (No — it's bounded by the libuv thread pool size; beyond that, requests still queue, just without blocking the event loop.)

---

### 32. **Design a notification system that sends email, SMS, and push to millions of users, with delivery guarantees and without overwhelming providers.**

Clarify first: delivery guarantee (at-least-once is typical), latency expectation (near-real-time vs batch), and provider rate limits. Then design around **queues + workers + idempotency**, because this is a textbook async fan-out problem.

**Architecture:**

1. **Ingest:** an API/event accepts "notify user X with template Y." It validates, resolves the user's channel preferences, and **enqueues** a job per channel — it does *not* send synchronously.
2. **Queue per channel** (email, SMS, push) so each can scale and be rate-limited independently. A slow SMS provider mustn't back up email.
3. **Worker pools** consume each queue and call the provider (SendGrid/SES, Twilio, FCM/APNs). Workers scale horizontally with queue depth.
4. **Provider rate limiting:** throttle each worker pool to the provider's allowed rate (BullMQ rate limiting / a token bucket, Q14) so you don't get throttled or banned by the provider.
5. **Retries with exponential backoff** for transient failures, and a **dead-letter queue** for permanent failures (invalid number, hard bounce) that triggers alerting and suppression-list updates.
6. **Idempotency / dedup** (Q9/Q15): at-least-once delivery means a job may run twice; use a dedup key (`userId + notificationId + channel`) so a retried job doesn't double-send.
7. **User preferences & quiet hours / rate caps per user** so one event storm doesn't spam a user 50 times.
8. **Observability:** track sent/delivered/bounced/failed per channel; expose delivery status. Handle provider **webhooks** (delivery receipts, bounces) to close the loop.

```js
// Ingest: fan out one logical notification into per-channel jobs, then return fast.
async function notify(userId, templateId, data) {
  const prefs = await getChannelPrefs(userId);            // email? sms? push?
  const dedupBase = `${userId}:${templateId}:${data.eventId}`;
  for (const channel of prefs.enabled) {
    await queues[channel].add("send", { userId, templateId, data }, {
      jobId: `${dedupBase}:${channel}`,                    // dedup: same id => not re-added
      attempts: 5,
      backoff: { type: "exponential", delay: 2000 },
    });
  }
}
```

> BEST PRACTICE: One queue per channel with independent scaling and rate limits. It isolates failures (a Twilio outage doesn't stall email), lets you match each provider's throughput limit, and makes the slowest channel the only thing waiting on the slowest channel.

> Follow-up: "How do you guarantee a user isn't notified twice for the same event?" (Deterministic dedup job id from event+user+channel, plus a persisted sent-record check inside the worker.) "How do you handle a provider being down for an hour?" (Backoff + the queue buffers the jobs; they drain when the provider recovers.)

---

## Rapid-Fire Quick Answers

| Question | One-line answer |
| --- | --- |
| Why must instances be stateless to scale horizontally? | So any instance can serve any request — state goes to Redis/DB/S3, not process memory. |
| What does a load balancer do? | Distributes traffic across instances, health-checks them, and enables failover and TLS termination. |
| `cluster` vs `worker_threads`? | cluster = multiple processes for more concurrent connections; worker_threads = parallelism for CPU-bound work. |
| Which HTTP methods are idempotent? | GET, HEAD, PUT, DELETE (and conditional PATCH); POST is not. |
| How do you make POST safely retryable? | An idempotency key claimed atomically (`SET NX`), with the result stored for replay. |
| Cache-aside pattern? | Check cache → on miss, read DB → write result back to cache with a TTL. |
| Simplest cache invalidation? | A short TTL — let entries self-expire. |
| What is a cache stampede? | Many concurrent misses on the same expired key all hammering the DB at once. |
| Fixed-window rate limit flaw? | Boundary burst — a client can send 2× the limit straddling the window edge. |
| Most popular rate-limit algorithm for APIs? | Token bucket — allows bursts while bounding the average rate. |
| Embed vs reference in MongoDB? | Embed bounded data read together; reference unbounded or independently-queried data. |
| Offset vs cursor pagination at scale? | Cursor — constant-time and stable under writes; offset degrades and drifts. |
| Big files at scale — how to upload? | Presigned URLs: client uploads directly to S3, bypassing your app servers. |
| When to use a job queue? | For slow, retryable, or CPU-bound work that needn't finish within the request. |
| Queue delivery guarantee? | At-least-once — make every job idempotent because it may run twice. |
| Graceful shutdown steps? | Fail readiness → stop new work → finish in-flight → close resources → force-exit on timeout. |
| Zero-downtime deploy needs? | Stateless instances + graceful shutdown + rolling/blue-green + backward-compatible migrations. |
| 100% one core + timeouts means? | A blocked event loop — synchronous CPU-bound work on the main thread. |
| Three pillars of observability? | Logs (what happened), metrics (how often/fast), traces (where in the call graph). |
| What ties them together? | A correlation/request ID propagated across all services. |
| Crash on uncaughtException? | Yes — state may be corrupt; log it and let the orchestrator restart a clean process. |
| First move when a deploy spikes errors? | Roll back (mitigate first), then root-cause. |
| Multi-document atomicity in MongoDB? | A transaction via `session.withTransaction` — passed to every op, all-or-nothing. |
| Dual-write problem fix? | Transactional outbox — write the event in the same DB transaction, relay it separately. |
| Cross-service "transaction"? | Saga pattern — local transactions plus compensating actions; eventual consistency. |
| Most common API authz bug? | Broken object-level authorization (IDOR) — scope every query to `req.user.id`. |
| MongoDB query slow at scale — first check? | `explain()` for `COLLSCAN`; add a compound index by the ESR rule (Equality, Sort, Range). |
| Real-time across multiple instances? | WebSockets + a Redis pub/sub backplane so messages reach sockets on every instance. |
