# Project Architecture & Best Practices

A working API and a *maintainable* API are two very different things. Early in a project, almost any structure works: you can throw everything into one `index.js`, wire your database calls directly into your route handlers, and ship. It feels fast. The trap is that this speed is borrowed against the future at a brutal interest rate. The first time you need to reuse a piece of logic, write a unit test, swap your database, onboard a teammate, or hunt a bug across 4,000 lines of tangled route handlers, you pay it all back at once.

This module is about the discipline that separates a hobby script from a production service. We are going to build a mental model of how to *organize* an Express + Node application so that it stays understandable and changeable as it grows from 100 lines to 100,000. We will cover **separation of concerns**, the **MVC** pattern, **layered architecture** (routes → controllers → services → repositories → models), a **production folder structure**, rigorous **configuration and environment management**, **thin controllers**, the **repository pattern**, **DTOs and a validation layer**, **centralized error handling and logging**, **naming conventions**, and finally the **Twelve-Factor App** principles that tie it all together for deployment.

None of these patterns are framework features. Express will happily let you ignore every one of them. They are *engineering conventions* — agreements about where things go and who is allowed to do what — and their entire payoff is human: a codebase a team can reason about, test, and change without fear.

## What You'll Learn

By the end of this module you will be able to:

- Explain **why structure matters** as an application grows, and articulate **separation of concerns** from first principles.
- Apply the **MVC** pattern to an Express application and explain how it maps onto HTTP.
- Design a **layered architecture** — routes → controllers → services → repositories → models — and explain the single responsibility of each layer.
- Lay out a **production-grade folder structure** and justify every directory.
- Build robust **configuration management**: `dotenv`, per-environment config, and **fail-fast validation of environment variables at startup**.
- Keep **controllers thin** and push **business logic into services**, and recognize the smells of a "fat controller."
- Implement the **repository pattern** and explain precisely *why it makes code testable* (and when it is overkill).
- Use **DTOs** and a **validation layer** to decouple your API contract from your database schema, and **map DB models to API responses** safely.
- Integrate **centralized error handling and logging** into the architecture as cross-cutting concerns rather than scattered `try/catch` and `console.log`.
- Apply consistent **naming conventions and code organization**.
- Apply the **Twelve-Factor App** methodology to a real Node API and explain each factor in an interview.

> Note: This module targets **Express v5** (the default `express` on npm since 2024) and **Node.js current LTS**. Examples use CommonJS (`require`) by default and show ES Modules where it matters. Architecture is language-agnostic in spirit — these same layers appear in Spring (Java), NestJS, Django, and Rails — so what you learn here transfers far beyond Express.

---

## 1. Why Structure Matters as Apps Grow

### The "it works on my one file" fallacy

Here is a complete, working Express endpoint that creates a user. It is also a textbook example of everything we are about to fix.

```js
// app.js — everything in one place. Works. Doesn't scale.
const express = require("express");
const { MongoClient } = require("mongodb");
const bcrypt = require("bcrypt");

const app = express();
app.use(express.json());

app.post("/users", async (req, res) => {
  // 1. validation (inline)
  if (!req.body.email || !req.body.password) {
    return res.status(400).json({ error: "email and password required" });
  }
  // 2. business rule (inline)
  if (req.body.password.length < 8) {
    return res.status(400).json({ error: "password too short" });
  }
  // 3. data access (inline, connection created per request!)
  const client = await MongoClient.connect("mongodb://localhost:27017");
  const db = client.db("app");
  const existing = await db.collection("users").findOne({ email: req.body.email });
  if (existing) return res.status(409).json({ error: "email taken" });
  // 4. more business logic (inline)
  const hash = await bcrypt.hash(req.body.password, 10);
  const result = await db.collection("users").insertOne({
    email: req.body.email,
    password: hash, // oops — we will leak this back to the client below
    createdAt: new Date(),
  });
  // 5. response shaping (inline — leaks the password hash!)
  res.status(201).json({ _id: result.insertedId, ...req.body });
});

app.listen(3000);
```

This handler does **five completely different jobs**: HTTP plumbing, input validation, business rules, database access, and response formatting. Each is a different *concern*, and they are welded together. Concretely, this design fails the moment you need any of the following:

- **Reuse.** Suppose a CLI admin tool also needs to "create a user." You cannot call this logic without faking an HTTP `req`/`res`. The logic is imprisoned inside an HTTP handler.
- **Testing.** To test the password rule, you must spin up Express, open a real Mongo connection, and make an HTTP call. A rule that is one `if` statement now requires an integration test.
- **Change.** Switching from raw MongoDB to Mongoose, or from Mongo to Postgres, means editing every handler that touches the database.
- **Bugs.** Notice line 5 leaks the password hash back to the client (`...req.body` plus the inserted doc). With logic this tangled, that bug is easy to write and hard to see.
- **A new teammate.** There is no map. To change anything, you must read everything.

### Complexity is the enemy, and it is super-linear

The cost of a change in a codebase is not proportional to the size of the change — it is proportional to *how much of the system you must understand to make the change safely*. In a tangled codebase, that is "all of it," every time. Structure is the tool we use to keep the **blast radius** of any change small: ideally, to change how passwords are validated, you should open exactly one file, and be confident nothing else is affected.

> BEST PRACTICE: Optimize for **change**, not for the initial write. You write a feature once but you read, debug, and modify it dozens of times. Architecture trades a little up-front ceremony for a large, compounding reduction in the cost of every future change.

### When *not* to over-engineer

Architecture is a cost too. A 50-line internal script with three endpoints does not need five layers, a repository abstraction, and a DTO mapper — that is **gold-plating**, and it has its own maintenance cost. The skill is matching the structure to the lifespan and complexity of the project.

| Project type | Suggested structure |
| --- | --- |
| Throwaway script / spike | One file. Be honest that it is disposable. |
| Small service (< ~10 endpoints, solo) | Routes + controllers + a thin model layer. Skip repositories. |
| Production service / team / long-lived | Full layered architecture (this module). |
| Large modular monolith / micro-services | Layered architecture **per feature module**, plus boundaries between modules. |

The rest of this module assumes the third row — a real, production, team-owned service — because that is where these patterns earn their keep and what interviews probe.

---

## 2. Separation of Concerns (The Core Principle)

Every pattern in this module is a specific application of one idea: **separation of concerns (SoC)**. A "concern" is a single, distinct responsibility — a reason the code might need to change. SoC says: *group code by the responsibility it serves, and give each responsibility one place to live.*

### The Single Responsibility Principle, in practice

The "S" in SOLID — the **Single Responsibility Principle (SRP)** — is SoC at the level of a module or class: *a module should have one, and only one, reason to change.* Map that onto our user-creation example. It has at least five reasons to change:

1. The HTTP contract changes (new status codes, headers). → an **HTTP concern**.
2. The validation rules change (password must be 12 chars). → a **validation concern**.
3. The business policy changes (new users get a free trial). → a **domain/business concern**.
4. The persistence changes (Mongo → Postgres). → a **data-access concern**.
5. The database schema changes shape. → a **model concern**.

SoC tells us to give each of these a home so that a change to one does not force you to touch — or even understand — the others.

### Cohesion and coupling: the two dials

SoC is really about tuning two properties:

- **Cohesion** (want it *high*): how related the things inside one module are. A `userService` that only contains user business logic is highly cohesive.
- **Coupling** (want it *low*): how much one module depends on the internals of another. If your controller knows the exact Mongo query syntax, it is tightly coupled to Mongo.

> TIP: A quick gut-check for good separation — *"If I change X, what else do I have to open?"* If the answer is "a lot of unrelated files," your coupling is too high. If a single file is full of unrelated jobs, your cohesion is too low.

### The data-flow picture

In a well-separated Express app, a request flows *down* through layers and a response flows *back up*, with each layer speaking only to its immediate neighbors:

```
HTTP Request
   │
   ▼
┌─────────────┐   knows about HTTP, status codes, req/res
│   Routes    │   maps URL + verb → controller
└─────────────┘
   │
   ▼
┌─────────────┐   the only layer that touches req/res
│ Controllers │   parse input, call a service, shape the response
└─────────────┘
   │   (plain JS objects in, plain JS objects out — no req/res below here)
   ▼
┌─────────────┐   the brain: business rules, orchestration, transactions
│  Services   │   pure-ish, framework-agnostic, the most valuable code
└─────────────┘
   │
   ▼
┌─────────────┐   the only layer that knows the persistence tech
│Repositories │   CRUD against the DB; hides Mongoose/SQL details
└─────────────┘
   │
   ▼
┌─────────────┐   schema/shape of persisted data (Mongoose models)
│   Models    │
└─────────────┘
   │
   ▼
 Database
```

The golden rule of this diagram: **dependencies point downward only.** A controller may call a service; a service may call a repository. A repository must *never* import a controller, and a service must *never* touch `req`/`res`. This one-directional flow is what keeps the lower (more valuable) layers reusable and testable.

---

## 3. The MVC Pattern Applied to Express

**MVC (Model–View–Controller)** is the most widely taught web architecture, and it is the natural starting point before we extend it into full layering. It splits an app into three roles:

| Role | Responsibility | In an Express **API** |
| --- | --- | --- |
| **Model** | Data + the rules for that data's shape/persistence | Mongoose schemas/models; the data layer |
| **View** | Presentation of data to the user | For a JSON API, the **JSON response** *is* the view (with templating engines like EJS/Pug for server-rendered HTML) |
| **Controller** | Receives input, coordinates Model + View, returns a response | Express route handlers (parse `req`, call model/service, send `res`) |

The controller is the *mediator*: input arrives, it asks the Model for data (or to change data), and it produces a View (a JSON payload or rendered page). The Model and the View never talk to each other directly.

### A minimal MVC slice

Let's restructure the user example into clean MVC. First, the **Model** (Mongoose):

```js
// models/user.model.js — the Model: shape + persistence rules
const mongoose = require("mongoose");

const userSchema = new mongoose.Schema(
  {
    email: { type: String, required: true, unique: true, lowercase: true, trim: true },
    password: { type: String, required: true, select: false }, // never returned by default
    name: { type: String, trim: true },
  },
  { timestamps: true } // adds createdAt / updatedAt
);

module.exports = mongoose.model("User", userSchema);
```

> TIP: `select: false` on `password` means Mongoose excludes it from query results unless you explicitly ask (`.select("+password")`). That is a *model-level* defense against the password-leak bug we saw in §1 — the model itself refuses to hand the field out casually.

Next, the **Controller** (the Express handler — note it still does too much for now; we fix that in §5):

```js
// controllers/user.controller.js — the Controller: HTTP in, HTTP out
const User = require("../models/user.model");

exports.createUser = async (req, res, next) => {
  try {
    const user = await User.create({
      email: req.body.email,
      password: req.body.password,
      name: req.body.name,
    });
    // The "View" for an API: a JSON representation. toJSON strips nothing yet — see §8.
    res.status(201).json({ id: user._id, email: user.email, name: user.name });
  } catch (err) {
    next(err); // hand errors to centralized handler (see §10)
  }
};
```

Finally, the **routes** wire URLs to controller methods (in classic MVC, routing is sometimes folded into the controller; in Express we keep a dedicated routing file — see §4):

```js
// routes/user.routes.js — the URL → controller mapping
const router = require("express").Router();
const userController = require("../controllers/user.controller");

router.post("/", userController.createUser);
router.get("/:id", userController.getUserById);

module.exports = router;
```

### MVC's strength and its limit

MVC is a huge step up from the one-file mess: it cleanly separates *data*, *presentation*, and *coordination*. But classic MVC has a famous gap — **it has no obvious home for business logic.** Where does "new users get a free trial and a welcome email" go?

- Put it in the **Controller** → controllers become bloated, un-reusable "fat controllers."
- Put it in the **Model** → models become bloated "fat models," and Mongoose schemas start orchestrating emails and payments, which is not their job.

The pragmatic answer is to add layers *between* the controller and the model. That is exactly what the next section does: it keeps MVC's clarity and gives business logic a proper home.

---

## 4. Layered Architecture: Routes → Controllers → Services → Repositories → Models

Layered architecture (a.k.a. **n-tier**) takes MVC and refines the "controller" and "model" roles into distinct, single-purpose layers. Each layer has exactly one job and talks only to the layer directly beneath it.

### The five layers and their one job each

| Layer | One job | Knows about | Must NOT |
| --- | --- | --- | --- |
| **Routes** | Map `METHOD /path` → a controller method; attach route-specific middleware | HTTP verbs, paths, middleware | Contain business or data logic |
| **Controllers** | Translate between HTTP and the application; orchestrate the request | `req`/`res`, status codes, services | Touch the database or hold business rules |
| **Services** | Business logic, rules, orchestration, transactions | Domain concepts, repositories | Touch `req`/`res` or know it is "the web" |
| **Repositories** | Encapsulate all data access (CRUD); hide the persistence tech | Mongoose/SQL, models | Hold business rules |
| **Models** | Define the shape/schema of persisted data | The database schema | Orchestrate other systems |

The key conceptual move from MVC: we split the "controller" into a thin **HTTP adapter** (the controller) and a fat **business core** (the service), and we split the "model" into **data access** (the repository) and **data shape** (the model). The result is that the most valuable, most-likely-to-be-reused code — your business logic — lives in plain, framework-free JavaScript functions.

### A complete vertical slice

Let's implement "create a user" through all five layers. Read it top to bottom and notice how *thin* each layer is.

**Routes** — pure wiring, plus validation middleware (see §8):

```js
// modules/user/user.routes.js
const router = require("express").Router();
const userController = require("./user.controller");
const validate = require("../../middleware/validate");
const { createUserSchema } = require("./user.validation");

// Validation runs first; only valid requests reach the controller.
router.post("/", validate(createUserSchema), userController.createUser);

module.exports = router;
```

**Controller** — thin HTTP adapter. It extracts input, calls the service, and shapes the response. No business logic, no DB:

```js
// modules/user/user.controller.js
const userService = require("./user.service");
const { toUserResponse } = require("./user.dto");

exports.createUser = async (req, res, next) => {
  try {
    // req.body is already validated by the route middleware
    const user = await userService.registerUser(req.body);
    // Map the domain object to the API response shape (DTO) — see §8
    res.status(201).json(toUserResponse(user));
  } catch (err) {
    next(err); // delegate to centralized error handler
  }
};
```

**Service** — the brain. Pure business logic, no `req`/`res`, no Mongoose syntax. This is the code you will reuse from a CLI, a queue worker, or a test:

```js
// modules/user/user.service.js
const bcrypt = require("bcrypt");
const userRepository = require("./user.repository");
const { ConflictError } = require("../../errors");

// Business rule: emails must be unique; passwords are hashed before storage.
async function registerUser({ email, password, name }) {
  const existing = await userRepository.findByEmail(email);
  if (existing) {
    throw new ConflictError("Email is already registered"); // domain-level error
  }
  const passwordHash = await bcrypt.hash(password, 12);
  const user = await userRepository.create({ email, password: passwordHash, name });
  // (Future: enqueue a welcome email here — services orchestrate, controllers don't)
  return user;
}

module.exports = { registerUser };
```

**Repository** — the only place that knows we use Mongoose. It speaks "CRUD," not "business":

```js
// modules/user/user.repository.js
const User = require("./user.model");

// Each method is a single persistence operation. No business rules here.
function findByEmail(email) {
  return User.findOne({ email });
}

function create(data) {
  return User.create(data);
}

function findById(id) {
  return User.findById(id);
}

module.exports = { findByEmail, create, findById };
```

**Model** — the schema (same as §3):

```js
// modules/user/user.model.js
const mongoose = require("mongoose");

const userSchema = new mongoose.Schema(
  {
    email: { type: String, required: true, unique: true, lowercase: true, trim: true },
    password: { type: String, required: true, select: false },
    name: { type: String, trim: true },
  },
  { timestamps: true }
);

module.exports = mongoose.model("User", userSchema);
```

### Why this is worth the extra files

Count what you can now do that you could not in §1:

- **Test the business rule in isolation.** Call `registerUser` with a *fake* repository; assert it throws `ConflictError`. No HTTP, no database (see §7).
- **Reuse from anywhere.** A CLI script can `require("./user.service")` and call `registerUser` directly.
- **Swap the database** by rewriting only `user.repository.js`. The service, controller, and routes never change.
- **Find anything instantly.** Validation bug? Look in `*.validation.js`. Business rule wrong? `*.service.js`. Response shape wrong? `*.dto.js`. The structure *is* the map.

> WARNING — Pitfall: The most common way to ruin a layered architecture is **layer bleed** — a service that reaches up and touches `res.json(...)`, or a controller that writes a Mongoose query. The moment a layer does another layer's job, you lose every benefit above. Be ruthless: services never see `req`/`res`; only repositories see the database.

### Dependency injection vs. direct `require`

In the examples above, each module `require`s its dependency directly (the service requires the repository). This is simple and fine for most apps. The more advanced alternative is **dependency injection (DI)**: pass dependencies in rather than importing them, which makes substitution (e.g., a fake repo in tests) trivial and explicit.

```js
// A DI-friendly service: dependencies are arguments, not hard-coded requires.
function makeUserService({ userRepository, hasher }) {
  async function registerUser({ email, password, name }) {
    if (await userRepository.findByEmail(email)) {
      throw new ConflictError("Email is already registered");
    }
    const passwordHash = await hasher.hash(password, 12);
    return userRepository.create({ email, password: passwordHash, name });
  }
  return { registerUser };
}
module.exports = makeUserService;

// Wiring it up (composition root, e.g. in a container or app bootstrap):
const userService = makeUserService({
  userRepository: require("./user.repository"),
  hasher: require("bcrypt"),
});
```

DI pays off in large apps and test-heavy teams; for a first production service, direct `require` plus the repository pattern is usually enough. Frameworks like **NestJS** make DI the default; in plain Express it is opt-in.

---

## 5. Keeping Controllers Thin, Business Logic in Services

This deserves its own section because it is the single most violated rule in real Express codebases — the **"fat controller"**.

### What a thin controller looks like

A controller has exactly three responsibilities, and ideally three corresponding lines:

1. **Extract** the input it needs from the HTTP request (`req.body`, `req.params`, `req.query`, `req.user`).
2. **Call** a service method, passing plain data.
3. **Send** the result back as an HTTP response with the right status code (or call `next(err)`).

```js
// A perfectly thin controller. Reads like a sentence.
exports.getOrder = async (req, res, next) => {
  try {
    const order = await orderService.getOrderForUser(req.params.id, req.user.id);
    res.json(toOrderResponse(order));
  } catch (err) {
    next(err);
  }
};
```

That is the *whole* controller. Everything interesting — "does this user own this order?", "is the order cancellable?" — lives in `orderService`.

### Smells of a fat controller

You have a fat controller if its handler does any of these:

- Builds database queries (`Model.find({...})`) directly.
- Contains `if`/business-rule branches about the *domain* (e.g., "if the order total exceeds $1000, apply manual review").
- Calls external APIs (payment gateways, email providers).
- Computes derived values, applies discounts, or enforces invariants.
- Is longer than ~15 lines, or you cannot describe it as "read input, call service, send response."

### Before and after

```js
// ❌ FAT controller — business logic trapped in the HTTP layer
exports.checkout = async (req, res) => {
  const cart = await Cart.findById(req.params.cartId).populate("items.product");
  let total = 0;
  for (const item of cart.items) {
    total += item.product.price * item.quantity; // business rule
  }
  if (req.user.isPremium) total *= 0.9; // business rule
  if (total > 1000) {                     // business rule
    await ReviewQueue.create({ cartId: cart.id, total });
  }
  const order = await Order.create({ userId: req.user.id, total });
  await stripe.charges.create({ amount: total * 100, source: req.body.token });
  res.json(order);
};
```

```js
// ✅ THIN controller — the logic moved to a service
exports.checkout = async (req, res, next) => {
  try {
    const order = await checkoutService.placeOrder({
      cartId: req.params.cartId,
      user: req.user,
      paymentToken: req.body.token,
    });
    res.status(201).json(toOrderResponse(order));
  } catch (err) {
    next(err);
  }
};
```

The "after" controller is trivial and stable — it almost never needs to change. The pricing rules, premium discount, manual-review threshold, and payment integration now live in `checkoutService.placeOrder`, where they can be unit-tested without a browser, a token, or a running server.

> BEST PRACTICE: A useful litmus test — *could you call this logic from a cron job or a CLI command, unchanged?* If the only way to trigger it is an HTTP request, the logic is in the wrong layer. Push it down into a service.

> TIP: Eliminate repetitive `try/catch` in controllers with a small `asyncHandler` wrapper. In Express **v5**, rejected promises from `async` handlers are forwarded to the error handler automatically, so the wrapper is often unnecessary — but it is still common in v4 codebases:

```js
// utils/asyncHandler.js — wraps an async controller so rejections reach next(err)
// (Mainly for Express v4. Express v5 forwards async rejections automatically.)
const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);

module.exports = asyncHandler;

// Usage: no try/catch needed in the controller body
// router.post("/", asyncHandler(userController.createUser));
```

---

## 6. A Recommended Production Folder Structure

There is no single "correct" structure, but there are two well-trodden patterns. Below is the more scalable of the two — **feature-based (modular) organization** — followed by the simpler **layer-based** alternative.

### Feature-based structure (recommended for non-trivial apps)

Group files by *feature* (the thing they serve), and inside each feature keep the layers. This keeps everything about "users" in one folder, which is how teams actually think and work.

```text
my-api/
├── src/
│   ├── config/                 # configuration & env validation (§7)
│   │   ├── index.js            # exports validated, typed config object
│   │   ├── env.js              # loads + validates process.env (fail-fast)
│   │   └── database.js         # DB connection setup (Mongoose connect)
│   │
│   ├── modules/                # ★ one folder per feature/bounded context
│   │   ├── user/
│   │   │   ├── user.routes.js       # URL → controller mapping
│   │   │   ├── user.controller.js   # thin HTTP adapter
│   │   │   ├── user.service.js      # business logic
│   │   │   ├── user.repository.js   # data access
│   │   │   ├── user.model.js        # Mongoose schema
│   │   │   ├── user.validation.js   # request validation schemas (Joi/Zod)
│   │   │   ├── user.dto.js          # model → API response mapping
│   │   │   └── user.test.js         # tests live next to the code they test
│   │   ├── auth/
│   │   │   └── ... (same shape)
│   │   └── order/
│   │       └── ... (same shape)
│   │
│   ├── middleware/             # cross-cutting Express middleware
│   │   ├── errorHandler.js     # centralized error handler (§10)
│   │   ├── notFound.js         # 404 handler
│   │   ├── validate.js         # generic validation middleware (§8)
│   │   ├── authenticate.js     # JWT/session auth
│   │   └── rateLimiter.js
│   │
│   ├── errors/                 # custom error classes (§10)
│   │   └── index.js            # AppError, NotFoundError, ConflictError, ...
│   │
│   ├── utils/                  # generic, domain-agnostic helpers
│   │   ├── logger.js           # configured logger instance (pino/winston) (§10)
│   │   └── asyncHandler.js
│   │
│   ├── jobs/                   # background jobs / queue workers (optional)
│   ├── app.js                  # builds the Express app (middleware + routes). NO listen()
│   └── server.js               # imports app, connects DB, calls listen(). The entry point.
│
├── tests/                      # integration / e2e tests (unit tests live by code)
├── .env                        # local secrets — gitignored, NEVER committed
├── .env.example                # documented template of required vars — committed
├── .gitignore
├── .eslintrc.json
├── .prettierrc
├── Dockerfile
├── package.json
└── README.md
```

### Layer-based structure (fine for smaller apps)

Group files by *layer* instead of feature. Simpler to start, but every feature is smeared across many folders, so a single change touches files in `routes/`, `controllers/`, `services/`, etc.

```text
src/
├── routes/         # all route files
├── controllers/    # all controllers
├── services/       # all services
├── repositories/   # all repositories
├── models/         # all models
├── middleware/
├── config/
└── app.js
```

| | Layer-based | Feature-based (modular) |
| --- | --- | --- |
| Find "everything about users" | Scattered across 5+ folders | One folder |
| Onboarding mental model | "What kind of thing is it?" | "What feature does it serve?" |
| Scales to many features | Folders grow huge | Stays flat; add a folder per feature |
| Extracting a feature into a microservice | Painful | Copy one folder |
| Best for | Small apps, tutorials | Real, growing, team apps |

> BEST PRACTICE: Separate **`app.js`** (builds and configures the Express app but does *not* listen) from **`server.js`** (connects the DB and calls `app.listen`). This separation is what makes integration testing clean — your test suite imports `app` and hits it with `supertest` *without* binding a port or needing the full server lifecycle.

```js
// src/app.js — assembles the app; exportable and testable. No side effects like listen().
const express = require("express");
const userRoutes = require("./modules/user/user.routes");
const errorHandler = require("./middleware/errorHandler");
const notFound = require("./middleware/notFound");

const app = express();
app.use(express.json());

app.use("/api/v1/users", userRoutes); // mount feature routers under a versioned prefix

app.use(notFound);     // 404 for unmatched routes (must come after routes)
app.use(errorHandler); // centralized error handler (must be LAST)

module.exports = app;
```

```js
// src/server.js — the real entry point: side effects live here, not in app.js
const app = require("./app");
const config = require("./config");
const { connectDatabase } = require("./config/database");
const logger = require("./utils/logger");

async function start() {
  await connectDatabase(config.databaseUrl); // connect before accepting traffic
  const server = app.listen(config.port, () => {
    logger.info(`Server listening on port ${config.port} [${config.env}]`);
  });

  // Graceful shutdown: stop accepting new requests, then exit (Twelve-Factor §12)
  const shutdown = (signal) => {
    logger.info(`${signal} received, shutting down gracefully`);
    server.close(() => process.exit(0));
  };
  process.on("SIGTERM", () => shutdown("SIGTERM"));
  process.on("SIGINT", () => shutdown("SIGINT"));
}

start().catch((err) => {
  logger.error({ err }, "Failed to start server");
  process.exit(1);
});
```

---

## 7. The Repository Pattern (and Why It Helps Testing)

The **repository pattern** puts a thin abstraction between your business logic and your database. A repository exposes *intent-revealing* methods — `findByEmail`, `findActiveOrders`, `create` — and hides *how* those are implemented (Mongoose, raw SQL, an in-memory map, an HTTP call to another service). To the service layer, a repository looks like a collection of objects it can query and mutate; it has no idea a database exists.

### Why bother — three concrete payoffs

**1. Testability (the headline reason).** Because the service depends on the *repository interface* rather than on Mongoose, you can substitute a fake repository in tests. Your business-logic tests then run in milliseconds, with no database, deterministically.

```js
// user.service.test.js — testing business logic with a FAKE repository
const assert = require("node:assert");
const test = require("node:test");
const makeUserService = require("./user.service.di"); // the DI version from §4

test("registerUser rejects a duplicate email", async () => {
  // A hand-rolled fake repo — no Mongo, no network, fully under our control
  const fakeRepo = {
    findByEmail: async (email) => ({ id: "1", email }), // simulate "already exists"
    create: async () => assert.fail("create should NOT be called for a dup"),
  };
  const fakeHasher = { hash: async () => "hashed" };
  const service = makeUserService({ userRepository: fakeRepo, hasher: fakeHasher });

  await assert.rejects(
    () => service.registerUser({ email: "taken@x.com", password: "longenough" }),
    /already registered/
  );
});

test("registerUser hashes the password before storing", async () => {
  let storedPassword;
  const fakeRepo = {
    findByEmail: async () => null, // no existing user
    create: async (data) => { storedPassword = data.password; return { id: "1", ...data }; },
  };
  const fakeHasher = { hash: async () => "HASHED_VALUE" };
  const service = makeUserService({ userRepository: fakeRepo, hasher: fakeHasher });

  await service.registerUser({ email: "new@x.com", password: "plaintext123" });
  assert.strictEqual(storedPassword, "HASHED_VALUE"); // proves we never store plaintext
});
```

These two tests verify *critical* security rules with zero infrastructure. Without a repository abstraction, you would need a live MongoDB to run them — slower, flakier, and harder in CI.

**2. Swappable persistence.** If you migrate from MongoDB to PostgreSQL, or add a caching layer, you rewrite the repository and *nothing else*. The service signature stays identical.

```js
// Same interface, different backend — services don't notice the change.
// user.repository.js (Mongoose)
exports.findByEmail = (email) => User.findOne({ email });

// user.repository.js (hypothetical Postgres/Knex version — same method name & contract)
// exports.findByEmail = (email) =>
//   db("users").where({ email }).first();
```

**3. Centralized query logic.** Every query about users lives in one file. No more grepping the whole codebase to find every place that filters soft-deleted users — you change the repository once.

### When the repository pattern is overkill

Mongoose models are *already* a light data-access layer (they have `find`, `create`, etc.). For a small app where you will never change the database and your testing strategy uses a real test database (or `mongodb-memory-server`), a repository can be ceremony with little payoff. The honest trade-off:

| Use a repository when... | Skip it when... |
| --- | --- |
| You unit-test services in isolation | You test with a real/in-memory DB and that's fine |
| Persistence might change, or you have multiple data sources | You're certain you'll stay on one ORM forever |
| Query logic is complex and reused across services | Queries are trivial one-liners used once |
| The team is large and wants a hard boundary | Solo project / short-lived app |

> WARNING — Pitfall: A repository's job is **data access only**. If you find business rules (validation, calculations, "if premium then…") creeping into repository methods, you have recreated the fat-controller problem one layer down. Repositories return data; services decide what it means.

> TIP: Keep repository return types **persistence-agnostic where it matters**. If your service starts depending on Mongoose document methods (like `user.save()` mid-service), you have leaked the implementation back up. For maximum decoupling, repositories can return plain objects (`.lean()` in Mongoose) and expose explicit `update`/`save` methods. For most apps, returning documents is a fine pragmatic middle ground.

---

## 8. DTOs, the Validation Layer, and Mapping Models to API Responses

Two boundaries in every API need explicit, deliberate shaping: **data coming in** (request → validated input) and **data going out** (domain/model → API response). Both are handled by a *validation layer* and **DTOs (Data Transfer Objects)**.

### Why your DB model is not your API contract

A naive API does `res.json(user)` and returns the Mongoose document directly. This couples your public API to your database schema — a dangerous coupling:

- It **leaks fields** you never meant to expose (`password`, `__v`, internal flags, `passwordResetToken`).
- A schema change (renaming a column, adding an internal field) silently changes your public API and can break clients.
- It exposes database-specific shapes (`_id` instead of `id`, `ObjectId` types, timestamps you may not want public).

A **DTO** is a plain object that defines exactly what your API returns (or accepts), independent of how the data is stored. The mapping function from a model to a DTO is the *only* place your response shape is decided.

```js
// modules/user/user.dto.js — the OUTBOUND contract: model → API response
// This function is the single source of truth for "what a user looks like over the wire".
function toUserResponse(user) {
  return {
    id: user._id.toString(), // expose `id`, not Mongo's `_id`
    email: user.email,
    name: user.name ?? null,
    createdAt: user.createdAt,
    // Note what is ABSENT: password, __v, internal flags. They CANNOT leak.
  };
}

// Mapping a list reuses the single mapper — consistency for free.
function toUserListResponse(users) {
  return users.map(toUserResponse);
}

module.exports = { toUserResponse, toUserListResponse };
```

> BEST PRACTICE: An *allow-list* DTO (explicitly naming the fields to include) is far safer than a *deny-list* (`delete user.password`). With an allow-list, a newly added sensitive field is excluded **by default** — you have to opt in to expose it. With a deny-list, every new sensitive field leaks until someone remembers to add another `delete`.

### The inbound validation layer

Never trust input. A validation layer rejects malformed or malicious requests *before* they reach your controller or service, so the rest of the system can assume clean data. Use a schema library — **Joi**, **Zod**, or **express-validator**. Here is a reusable middleware approach with **Zod**:

```js
// modules/user/user.validation.js — the INBOUND contract: what a valid request looks like
const { z } = require("zod");

const createUserSchema = z.object({
  body: z.object({
    email: z.string().email(),
    password: z.string().min(8, "password must be at least 8 characters"),
    name: z.string().trim().min(1).optional(),
  }),
});

module.exports = { createUserSchema };
```

```js
// middleware/validate.js — generic validation middleware, reused by every route
// Parses & validates req against a schema; replaces req with the parsed (coerced) value.
const { ValidationError } = require("../errors");

const validate = (schema) => (req, res, next) => {
  const result = schema.safeParse({
    body: req.body,
    query: req.query,
    params: req.params,
  });
  if (!result.success) {
    // Convert library errors into a consistent shape; throw to centralized handler
    const details = result.error.issues.map((i) => ({
      path: i.path.join("."),
      message: i.message,
    }));
    return next(new ValidationError("Invalid request", details));
  }
  // Use the validated/coerced data downstream (it is now trustworthy & typed)
  req.body = result.data.body;
  next();
};

module.exports = validate;
```

Wired into the route, validation becomes a declarative line — the controller and service receive only valid data:

```js
router.post("/", validate(createUserSchema), userController.createUser);
```

### The full inbound/outbound picture

```
Client request
   │  (raw, untrusted JSON)
   ▼
[ Validation layer ]  ──reject──► 400 with details
   │  (clean, typed input — a "request DTO")
   ▼
Controller → Service → Repository → Model → DB
   │
   ▼  (domain object / Mongoose doc)
[ DTO mapper (toUserResponse) ]
   │  (safe, explicit "response DTO")
   ▼
Client response
```

> WARNING — Pitfall: Returning the raw Mongoose document (`res.json(user)`) is the most common source of accidental data leaks in Node APIs — passwords, tokens, and internal flags all ride along. Always pass domain objects through a DTO mapper before they hit `res.json`. The `select: false` model trick (§3) is a good second line of defense, but the DTO is your first.

> TIP: Mongoose lets you centralize part of the outbound transform via a schema `toJSON` transform (e.g., delete `_id`/`__v`, map to `id`). That is convenient, but it couples the response shape to the model again. Prefer explicit DTO functions for anything that is a real public contract; reserve `toJSON` transforms for universal hygiene like removing `__v`.

---

## 9. Configuration and Environment Management

Configuration is everything that varies between deployments: the port, the database URL, API keys, log level, feature flags. Two iron rules govern it:

1. **Never hard-code config in source**, especially secrets. A database password in `git` is a security incident waiting to happen and makes the same code unusable across environments.
2. **Strictly separate config from code** (this is Twelve-Factor §3, covered in §11). The *same build artifact* should run in dev, staging, and prod, differing only by its environment.

### `dotenv`: loading local config

In development, `.env` files hold your local config. The `dotenv` package reads `.env` into `process.env`. In production, you typically inject real environment variables via your platform (Docker, Kubernetes, Heroku, systemd) and may not use a `.env` file at all.

```bash
# .env  — LOCAL ONLY. Add to .gitignore. NEVER commit real secrets.
NODE_ENV=development
PORT=3000
DATABASE_URL=mongodb://localhost:27017/myapp
JWT_SECRET=dev-only-secret-change-me
LOG_LEVEL=debug
```

```bash
# .env.example  — COMMITTED. A documented template with NO real secrets.
# Copy to .env and fill in values: cp .env.example .env
NODE_ENV=development
PORT=3000
DATABASE_URL=
JWT_SECRET=
LOG_LEVEL=debug
```

> BEST PRACTICE: Commit a `.env.example` listing **every** variable the app needs (with safe placeholder values), and add `.env` to `.gitignore`. New developers run `cp .env.example .env`, and `.env.example` doubles as living documentation of your config surface.

### The single config module: load, validate, freeze

Do **not** scatter `process.env.X` reads throughout your codebase. Centralize them in one config module that (a) loads env, (b) **validates it at startup**, (c) applies defaults and type coercion, and (d) exports a frozen, typed object. Everything else imports *that*, never `process.env` directly.

```js
// src/config/env.js — load and VALIDATE environment at startup (fail fast)
require("dotenv").config(); // populate process.env from .env (no-op if file absent)
const { z } = require("zod");

// Declare the contract for the environment. Coerce strings → numbers, set defaults.
const envSchema = z.object({
  NODE_ENV: z.enum(["development", "test", "production"]).default("development"),
  PORT: z.coerce.number().int().positive().default(3000), // "3000" → 3000
  DATABASE_URL: z.string().url(),                          // required, must be a URL
  JWT_SECRET: z.string().min(16, "JWT_SECRET must be ≥ 16 chars"),
  LOG_LEVEL: z.enum(["fatal", "error", "warn", "info", "debug", "trace"]).default("info"),
});

const parsed = envSchema.safeParse(process.env);
if (!parsed.success) {
  // Fail LOUD and EARLY — a misconfigured app should never boot.
  console.error("❌ Invalid environment configuration:");
  for (const issue of parsed.error.issues) {
    console.error(`  - ${issue.path.join(".")}: ${issue.message}`);
  }
  process.exit(1); // crash on startup, not at 3am on the first request that needs the var
}

module.exports = parsed.data; // validated, typed, defaulted env
```

```js
// src/config/index.js — the app's typed config object; the ONLY config import elsewhere
const env = require("./env");

const config = {
  env: env.NODE_ENV,
  isProd: env.NODE_ENV === "production",
  isTest: env.NODE_ENV === "test",
  port: env.PORT,
  databaseUrl: env.DATABASE_URL,
  jwt: {
    secret: env.JWT_SECRET,
    expiresIn: "1h",
  },
  logLevel: env.LOG_LEVEL,
};

module.exports = Object.freeze(config); // immutable — nobody can mutate config at runtime
```

Now usage everywhere is clean and safe:

```js
const config = require("./config");
// config.port, config.databaseUrl, config.jwt.secret — typed, validated, defaulted.
// No `process.env.PORT || 3000` scattered around. No typos silently becoming undefined.
```

### Why validate at startup — the fail-fast principle

Without validation, a missing `DATABASE_URL` produces `undefined`, and your app boots happily — then crashes (or worse, *misbehaves*) the first time a request hits the database, possibly hours later, in production. Validating at startup turns a subtle, delayed, hard-to-trace runtime failure into a loud, immediate, obvious boot failure with a precise message. This is the **fail-fast** principle, and it is one of the highest-leverage habits in backend engineering.

> WARNING — Pitfall: `process.env` values are **always strings** (or `undefined`). `process.env.PORT` is `"3000"`, not `3000`; `process.env.FEATURE_FLAG` is the string `"false"`, which is *truthy*. Always coerce types in your config layer (`z.coerce.number()`, explicit `=== "true"` checks). Comparing the string `"false"` as a boolean is a classic, maddening bug.

### Per-environment configuration

Most variation belongs in environment variables, but some structured, non-secret config (timeouts, rate limits, feature toggles) can differ by environment. A clean pattern: keep environment-specific *defaults* in code, selected by `NODE_ENV`, while still allowing env-var overrides.

```js
// src/config/index.js (extended) — per-environment defaults, overridable by env vars
const env = require("./env");

const common = {
  jwt: { expiresIn: "1h" },
};

const perEnv = {
  development: { rateLimit: 1000, cors: { origin: "*" } },
  test:        { rateLimit: 100000, cors: { origin: "*" } },
  production:  { rateLimit: 100, cors: { origin: "https://app.example.com" } },
};

module.exports = Object.freeze({
  ...common,
  ...perEnv[env.NODE_ENV],
  env: env.NODE_ENV,
  port: env.PORT,
  databaseUrl: env.DATABASE_URL,
});
```

> BEST PRACTICE: Secrets (DB passwords, API keys, JWT secrets) belong **only** in environment variables / a secrets manager — never in committed config files, even environment-specific ones. Non-secret tuning (timeouts, limits) can live in code keyed by `NODE_ENV`. The dividing line is: *would I be comfortable if this value were public on GitHub?* If not, it is a secret.

---

## 10. Centralized Error Handling and Logging in the Architecture

Error handling and logging are **cross-cutting concerns** — they apply everywhere but belong nowhere in particular. The architectural goal is to handle them in *one place* rather than smearing `try/catch` and `console.log` across every layer.

### A custom error hierarchy

Define a small family of error classes that carry the information the HTTP layer needs (a status code, an operational flag). Services throw *domain* errors (`ConflictError`, `NotFoundError`); the centralized handler maps them to HTTP responses. This keeps services free of HTTP knowledge while still giving them an expressive way to signal failure.

```js
// src/errors/index.js — operational error hierarchy
class AppError extends Error {
  constructor(message, statusCode = 500, details = undefined) {
    super(message);
    this.name = this.constructor.name;
    this.statusCode = statusCode;
    this.isOperational = true; // expected, handled errors (vs. programmer bugs)
    this.details = details;
    Error.captureStackTrace(this, this.constructor);
  }
}

class NotFoundError extends AppError {
  constructor(message = "Resource not found") { super(message, 404); }
}
class ValidationError extends AppError {
  constructor(message = "Validation failed", details) { super(message, 400, details); }
}
class ConflictError extends AppError {
  constructor(message = "Resource conflict") { super(message, 409); }
}
class UnauthorizedError extends AppError {
  constructor(message = "Unauthorized") { super(message, 401); }
}

module.exports = { AppError, NotFoundError, ValidationError, ConflictError, UnauthorizedError };
```

The distinction between **operational errors** (expected: bad input, missing resource, conflict) and **programmer errors** (bugs: `undefined is not a function`) is crucial. Operational errors get a clean HTTP response. Programmer errors should be logged loudly and, in serious cases, the process should crash and be restarted by a supervisor (Twelve-Factor §9) — because the app may be in an unknown state.

### The centralized error-handling middleware

Express recognizes error-handling middleware by its **four** parameters `(err, req, res, next)`. Mounted *last*, it catches everything routed via `next(err)` (and, in Express v5, every rejected async handler automatically).

```js
// src/middleware/errorHandler.js — the SINGLE place errors become HTTP responses
const logger = require("../utils/logger");
const config = require("../config");
const { AppError } = require("../errors");

// eslint-disable-next-line no-unused-vars  (Express needs all 4 params to detect this)
function errorHandler(err, req, res, next) {
  // Normalize: known operational errors keep their status; everything else is a 500.
  const isOperational = err instanceof AppError && err.isOperational;
  const statusCode = isOperational ? err.statusCode : 500;

  // Log with context. Programmer/unknown errors are logged at "error" with full stack.
  const logPayload = {
    err,
    method: req.method,
    url: req.originalUrl,
    requestId: req.id, // attach a per-request id upstream for traceability
  };
  if (statusCode >= 500) logger.error(logPayload, "Unhandled error");
  else logger.warn(logPayload, err.message);

  // Build a client-safe response. NEVER leak stack traces or internals in production.
  const body = {
    error: {
      message: isOperational ? err.message : "Internal Server Error",
      ...(err.details ? { details: err.details } : {}),
      ...(config.isProd ? {} : { stack: err.stack }), // stack only in non-prod
    },
  };
  res.status(statusCode).json(body);
}

module.exports = errorHandler;
```

```js
// src/middleware/notFound.js — a 404 for any unmatched route, funneled into the handler
const { NotFoundError } = require("../errors");
module.exports = (req, res, next) =>
  next(new NotFoundError(`Cannot ${req.method} ${req.originalUrl}`));
```

The payoff: every controller's `catch` block is just `next(err)`. There is no `res.status(...).json(...)` for errors anywhere except this one file. Change your error response format once, and the whole API updates.

> WARNING — Pitfall: Error-handling middleware **must** have exactly four parameters and **must** be registered **after** all routes and other middleware (`app.use(errorHandler)` last). If you mount it early, requests will reach it before your routes; if you give it three parameters, Express treats it as ordinary middleware and never routes errors to it.

### Structured logging as a first-class concern

`console.log` is not logging — it has no levels, no structure, no destinations, and it is synchronous (it can block the event loop under load). Use a real logger — **Pino** (fast, JSON-first) or **Winston** — configured once and imported everywhere. Structured (JSON) logs are *machine-parseable*, which is what log aggregators (Datadog, ELK, Loki) need.

```js
// src/utils/logger.js — one configured logger, imported everywhere
const pino = require("pino");
const config = require("./../config");

const logger = pino({
  level: config.logLevel, // from validated config — e.g. "debug" in dev, "info" in prod
  // Pretty-print in dev for humans; raw JSON in prod for log aggregators (12-factor §11)
  transport: config.isProd
    ? undefined
    : { target: "pino-pretty", options: { colorize: true } },
  // Redact sensitive fields so they never reach the logs
  redact: ["req.headers.authorization", "*.password", "*.token"],
});

module.exports = logger;
```

```js
// HTTP request logging middleware (or use pino-http for this out of the box)
const logger = require("../utils/logger");
const { randomUUID } = require("node:crypto");

module.exports = (req, res, next) => {
  req.id = randomUUID(); // correlation id — thread it through logs & error responses
  const start = process.hrtime.bigint();
  res.on("finish", () => {
    const ms = Number(process.hrtime.bigint() - start) / 1e6;
    logger.info(
      { requestId: req.id, method: req.method, url: req.originalUrl, status: res.statusCode, ms },
      "request completed"
    );
  });
  next();
};
```

> BEST PRACTICE: Treat logs as **event streams written to stdout/stderr** (Twelve-Factor §11). Do *not* have the app write to or rotate log files — that is the platform's job. Emit structured JSON to stdout and let the runtime (Docker, systemd, k8s) capture and route it. This keeps the app environment-agnostic.

> TIP: Always log to a configured logger, never `console.log`, and attach a **correlation id** (`requestId`) to every log line for a request. When a user reports an error at 14:32, that id lets you find every log line for their exact request across all your services.

---

## 11. Naming Conventions and Code Organization

Consistency is a feature. When every file, function, and folder follows predictable conventions, a developer can *guess* where things are and what they are called — and be right. Conventions are arbitrary in detail but valuable in agreement; pick a set and enforce it with tooling.

### Recommended conventions

| Thing | Convention | Example |
| --- | --- | --- |
| Files (modules) | kebab-case or `name.role.js` | `user.service.js`, `auth-token.js` |
| Folders | kebab-case, **singular** feature name | `modules/user/`, `modules/order/` |
| Classes & constructors | PascalCase | `class UserService`, `AppError` |
| Variables & functions | camelCase | `registerUser`, `userRepository` |
| Constants (true constants) | UPPER_SNAKE_CASE | `MAX_LOGIN_ATTEMPTS` |
| Env variables | UPPER_SNAKE_CASE | `DATABASE_URL`, `JWT_SECRET` |
| Mongoose models | PascalCase, singular | `User`, `OrderItem` |
| Async functions | Verb-first, intent-revealing | `findByEmail`, `placeOrder` |
| Booleans | `is`/`has`/`can` prefix | `isActive`, `hasPermission`, `canEdit` |
| REST routes | plural nouns, lowercase, hyphenated | `GET /api/v1/users`, `/order-items` |

### Naming reveals intent

Names are documentation that cannot go stale silently. Prefer names that say *what* and *why*, not *how*:

```js
// ❌ Unclear — what does this return? what does d mean?
function proc(d) { return d.filter((x) => x.s === 1); }

// ✅ Reads like a sentence; the role of every layer is obvious from its name
function findActiveSubscriptions(subscriptions) {
  return subscriptions.filter((sub) => sub.status === STATUS.ACTIVE);
}
```

### Organizational rules of thumb

- **Co-locate by feature.** Everything about "orders" lives in `modules/order/`. A developer fixing an order bug opens one folder.
- **One export per file's purpose.** A `*.service.js` exports the service; a `*.model.js` exports the model. Don't mix unrelated concerns in one file.
- **Tests next to code** (`user.service.test.js` beside `user.service.js`) for unit tests; a separate `tests/` for integration/e2e. Proximity keeps tests honest and discoverable.
- **`index.js` as a public facade.** A module's `index.js` can re-export its public surface (`module.exports = { router, service }`), keeping internal files private by convention.
- **Depth over breadth, but not too deep.** Two-to-three folder levels is the sweet spot. `src/modules/user/user.service.js` is clear; `src/api/v1/features/users/services/impl/...` is a maze.

> BEST PRACTICE: Enforce conventions with **ESLint** + **Prettier** and a pre-commit hook (Husky + lint-staged). Conventions that depend on human vigilance erode; conventions enforced by CI do not. Make the machine the gatekeeper so code review can focus on logic, not formatting.

---

## 12. The Twelve-Factor App, Applied to a Node API

The [Twelve-Factor App](https://12factor.net/) is a methodology (originally from Heroku) for building software-as-a-service that is portable, scalable, and easy to deploy. It is the most-cited reference for "cloud-native" application design, and it ties together everything in this module. Each factor is a concrete rule; here is each one applied to a Node/Express API.

| # | Factor | What it means | In a Node API |
| --- | --- | --- | --- |
| 1 | **Codebase** | One codebase tracked in version control, many deploys | One Git repo; the same repo deploys to dev/staging/prod |
| 2 | **Dependencies** | Explicitly declare and isolate dependencies | All deps in `package.json`; commit `package-lock.json`; never rely on global installs |
| 3 | **Config** | Store config in the environment | `process.env` via validated config module (§9); no secrets in code |
| 4 | **Backing services** | Treat backing services as attached resources | DB, cache, queue addressed by URL in env; swap a local Mongo for Atlas by changing `DATABASE_URL` only |
| 5 | **Build, release, run** | Strictly separate the stages | CI builds an artifact/image; release = artifact + config; run executes it. No editing code on the server |
| 6 | **Processes** | Execute as one or more stateless processes | No in-memory sessions/uploads; store state in DB/Redis/S3 so any instance can serve any request |
| 7 | **Port binding** | Export services via port binding | App is self-contained and `app.listen(PORT)`; doesn't rely on an injected webserver |
| 8 | **Concurrency** | Scale out via the process model | Run multiple Node processes (PM2/cluster/k8s replicas); scale horizontally, not by one giant process |
| 9 | **Disposability** | Fast startup, graceful shutdown | Boot quickly; handle `SIGTERM` to finish in-flight requests then exit (the `shutdown` in §6) |
| 10 | **Dev/prod parity** | Keep environments as similar as possible | Same DB engine, same Node version everywhere; use Docker to match envs |
| 11 | **Logs** | Treat logs as event streams | Write structured JSON to stdout; the platform aggregates (§10). App never manages log files |
| 12 | **Admin processes** | Run admin tasks as one-off processes | Migrations/seeds as separate scripts (`node scripts/migrate.js`) using the *same* codebase & config |

### The factors that most shape your architecture

Three factors deserve emphasis because they drive concrete design decisions you make in this module:

**Factor 3 (Config in the environment)** is *why* §9 exists. The same Docker image must run unchanged in every environment, differing only by injected env vars. Hard-coding `mongodb://localhost` breaks this immediately.

**Factor 6 (Stateless processes)** is subtle and high-stakes. The moment you store session data in a module-level variable or save uploads to the local disk, you can no longer run two instances of your app — a request might land on the instance that *doesn't* have the state. This silently breaks horizontal scaling. State must live in shared backing services (DB, Redis, S3), not in-process.

```js
// ❌ Violates Factor 6 — in-memory state breaks the moment you run 2+ instances
const sessions = {}; // instance A's sessions are invisible to instance B
app.post("/login", (req, res) => { sessions[req.body.user] = Date.now(); });

// ✅ Stateless — session state lives in a shared backing service (Redis)
const session = require("express-session");
const RedisStore = require("connect-redis").default;
app.use(session({ store: new RedisStore({ client: redisClient }), secret: config.jwt.secret }));
```

**Factor 9 (Disposability)** is why the `server.js` in §6 listens for `SIGTERM` and calls `server.close()`. Orchestrators (Kubernetes, Docker, PM2) send `SIGTERM` to stop a process; a well-behaved app stops accepting new connections, finishes in-flight requests, closes DB connections, and exits cleanly — no dropped requests during a deploy.

> BEST PRACTICE: When you set up a new service, walk the twelve factors as a checklist. Most production incidents around deployment and scaling trace back to violating one of them — usually config-in-code (§3) or stateful processes (§6).

---

## Common Pitfalls & Gotchas

- **Layer bleed.** A service that imports `res` and calls `res.json`, or a controller that writes a Mongoose query. This collapses the architecture — the lower layers lose their reusability and testability. Keep `req`/`res` in controllers only; keep DB queries in repositories only.
- **The fat controller.** Business logic stuffed into the route handler. It cannot be reused or unit-tested without HTTP. If a controller is more than "extract → call service → respond," logic has leaked upward.
- **Returning raw DB documents.** `res.json(mongooseDoc)` leaks `password`, `__v`, and internal fields, and couples your public API to your schema. Always map through a DTO. Use allow-lists, not deny-lists.
- **Reading `process.env` everywhere.** Scattered `process.env.X` reads mean unvalidated, untyped, string-only config with no single source of truth. Centralize in one validated config module.
- **Forgetting env values are strings.** `process.env.PORT` is `"3000"`; `process.env.DEBUG === "false"` is the *truthy* string `"false"`. Coerce types in the config layer.
- **No startup validation.** A missing env var becomes `undefined`, the app boots, and it explodes hours later in production. Validate at startup and crash loudly (fail-fast).
- **Error middleware in the wrong place or with the wrong arity.** It must have four params `(err, req, res, next)` and be registered *last*. Three params = treated as normal middleware; mounted early = never reached.
- **Swallowing errors.** `catch (e) {}` or `catch (e) { console.log(e) }` without re-throwing or `next(e)` hides failures and returns wrong responses. Let errors propagate to the centralized handler.
- **`console.log` as logging.** No levels, no structure, no correlation ids, and it can block the event loop. Use a structured logger.
- **Stateful processes.** In-memory sessions, caches, or local-disk uploads break horizontal scaling (Twelve-Factor §6). Externalize state.
- **Over-engineering a small app.** Five layers, DI containers, and repositories for a three-endpoint internal tool is gold-plating. Match structure to the project's real complexity and lifespan.
- **Committing `.env`.** Secrets in version control is a security incident. Gitignore `.env`; commit `.env.example`.
- **`app.listen` inside `app.js`.** Coupling app construction to the server lifecycle makes integration testing awkward. Split `app.js` (build) from `server.js` (run).

## Best Practices

- **Optimize for change, not the first write.** The structure's payoff is the reduced cost of every future modification.
- **One responsibility per layer; dependencies point downward only.** Routes → controllers → services → repositories → models. Never upward.
- **Controllers stay thin.** Extract input, call a service, send a response. Push every business rule into a service.
- **Put business logic in framework-free services.** If you couldn't call it from a CLI unchanged, it's in the wrong layer.
- **Use repositories to isolate data access** when you unit-test services or might change persistence; skip them when they'd be pure ceremony.
- **Validate input at the edge; shape output through DTOs.** Never trust incoming data; never leak raw models outward (use allow-lists).
- **Centralize config in one validated, frozen module.** Load with `dotenv`, validate with a schema, coerce types, fail-fast at startup. Commit `.env.example`, gitignore `.env`.
- **Centralize error handling and logging.** One error-handler middleware (registered last); one structured logger; a custom operational-error hierarchy; correlation ids on every request.
- **Be consistent in naming and organization**, and enforce it with ESLint + Prettier + pre-commit hooks.
- **Organize by feature for non-trivial apps** so a change touches one folder.
- **Follow the Twelve Factors**, especially config-in-env, stateless processes, logs-as-streams, and graceful shutdown.
- **Separate `app.js` (build) from `server.js` (run)** for clean testing and a clear entry point.

## Key Interview Points

- **"Why does project structure matter?"** — Because the cost of a change scales with how much of the system you must understand to make it safely. Structure (separation of concerns) keeps the blast radius of changes small, enables reuse and testing, and lets teams work in parallel. It optimizes for change, which dominates a project's lifetime cost.
- **"What is separation of concerns / SRP?"** — Group code by responsibility; each module has one reason to change. Aim for high cohesion (related things together) and low coupling (modules don't depend on each other's internals). It's the principle behind every other pattern here.
- **"How does MVC map onto an Express API?"** — Model = Mongoose schema/data layer; View = the JSON response (or a template for SSR); Controller = the route handler that mediates input, model, and output. MVC's gap is that it has no natural home for business logic, which is why we add a service layer.
- **"Walk me through your layered architecture."** — Routes (URL→controller) → Controllers (HTTP adapter) → Services (business logic) → Repositories (data access) → Models (schema). Each has one job; dependencies point downward only; `req`/`res` live only in controllers; DB queries live only in repositories.
- **"Why keep controllers thin?"** — So business logic is reusable (callable from a CLI/queue/test), unit-testable without HTTP, and not duplicated. A thin controller is "extract input, call service, respond."
- **"What is the repository pattern and why use it?"** — An abstraction over data access exposing intent-revealing methods. It enables unit-testing services with fake repositories (no DB), makes the persistence layer swappable, and centralizes query logic. Trade-off: it's overkill for small apps that test against a real/in-memory DB.
- **"What's a DTO and why not return the DB model directly?"** — A DTO is a plain object defining your API contract independent of the schema. Returning raw models leaks sensitive fields (passwords, tokens) and couples the public API to the database. Map via an allow-list DTO so new fields don't leak by default.
- **"How do you handle configuration and secrets?"** — Config lives in the environment (Twelve-Factor §3), loaded via `dotenv` in dev and injected by the platform in prod. One central config module validates env at startup (fail-fast), coerces types (env values are strings!), and exports a frozen object. Secrets never go in code; `.env` is gitignored, `.env.example` is committed.
- **"How does centralized error handling work in Express?"** — Throw operational errors (custom classes with status codes) from services; controllers just `next(err)`; one error-handling middleware with four params, registered last, maps errors to HTTP responses and logs them. Distinguish operational errors (clean response) from programmer errors (log + crash/restart).
- **"What is the Twelve-Factor App? Name a few factors."** — A methodology for portable, scalable SaaS. Key ones: config in environment (3), stateless processes (6) for horizontal scaling, logs as event streams to stdout (11), disposability/graceful shutdown (9), dev/prod parity (10), and treating backing services as attached resources (4).
- **"Why split `app.js` from `server.js`?"** — `app.js` builds the Express app with no side effects, so tests can import it and use `supertest` without binding a port; `server.js` connects the DB, calls `listen`, and handles graceful shutdown.
- **"Feature-based vs. layer-based folders?"** — Layer-based groups by technical role (all controllers together); feature-based groups by domain (everything about users together). Feature-based scales better for real apps because a change touches one folder and a feature can be extracted to a microservice by copying a directory.

## Practice Exercises

Work through these in order; each builds on the last. Solutions follow.

1. **Refactor a fat handler (Beginner).** Take the single-file `POST /users` handler from §1 and split it into routes → controller → service → repository → model files. The behavior must be identical, but the password must no longer leak in the response, and there should be no `try/catch` business logic in the controller beyond `next(err)`.

2. **Validate the environment (Beginner–Intermediate).** Write a `config/env.js` that validates `PORT` (number, default 3000), `DATABASE_URL` (required URL), and `NODE_ENV` (`development`/`test`/`production`, default `development`). It must exit with a clear, multi-line message listing every problem if validation fails, and export a typed, frozen config.

3. **Add a DTO + validation layer (Intermediate).** Given a `Product` model with fields `name`, `price`, `costPrice` (internal — must never be exposed), `sku`, and `createdAt`, write (a) a Zod validation schema for `POST /products` that requires `name` (non-empty) and `price` (positive number) and forbids unknown fields, and (b) a `toProductResponse` DTO that exposes `id`, `name`, `price`, `sku`, `createdAt` but never `costPrice`.

4. **Build the repository + a unit test (Intermediate–Advanced).** Implement an `orderService.cancelOrder(orderId, userId)` with these rules: the order must exist (else `NotFoundError`), the order must belong to `userId` (else `UnauthorizedError`), and only orders with status `"pending"` can be cancelled (else `ConflictError`). Then write unit tests using a *fake* repository — no database — covering all four paths (success + three errors).

5. **Centralized error handling end-to-end (Advanced).** Wire up the custom error hierarchy, an `errorHandler` middleware, and a `notFound` middleware so that: domain errors produce their status code and message; unknown errors produce a generic 500 with no stack trace in production; unmatched routes produce a 404; and every error is logged with the request method and URL.

6. **Twelve-Factor audit (Advanced).** Given a description of an app that (a) hard-codes `mongodb://localhost:27017` in code, (b) stores login sessions in a module-level `{}` object, (c) writes logs to `./logs/app.log` with manual rotation, and (d) has no shutdown handling, identify which Twelve-Factor principles each violates and describe the fix for each.

### Solutions

**Solution 1 — Refactor the fat handler.**

```js
// models/user.model.js
const mongoose = require("mongoose");
const userSchema = new mongoose.Schema(
  {
    email: { type: String, required: true, unique: true, lowercase: true, trim: true },
    password: { type: String, required: true, select: false }, // not returned by default
  },
  { timestamps: true }
);
module.exports = mongoose.model("User", userSchema);
```

```js
// modules/user/user.repository.js — data access only
const User = require("../../models/user.model");
exports.findByEmail = (email) => User.findOne({ email });
exports.create = (data) => User.create(data);
```

```js
// modules/user/user.service.js — business logic, no HTTP, no Mongoose syntax
const bcrypt = require("bcrypt");
const repo = require("./user.repository");
const { ConflictError, ValidationError } = require("../../errors");

exports.registerUser = async ({ email, password }) => {
  if (!email || !password) throw new ValidationError("email and password required");
  if (password.length < 8) throw new ValidationError("password too short");
  if (await repo.findByEmail(email)) throw new ConflictError("email taken");
  const passwordHash = await bcrypt.hash(password, 12);
  return repo.create({ email, password: passwordHash });
};
```

```js
// modules/user/user.dto.js — explicit, safe response shape (no password!)
exports.toUserResponse = (u) => ({ id: u._id.toString(), email: u.email, createdAt: u.createdAt });
```

```js
// modules/user/user.controller.js — thin
const service = require("./user.service");
const { toUserResponse } = require("./user.dto");
exports.createUser = async (req, res, next) => {
  try {
    const user = await service.registerUser(req.body);
    res.status(201).json(toUserResponse(user)); // password cannot leak — DTO controls output
  } catch (err) {
    next(err);
  }
};
```

```js
// modules/user/user.routes.js
const router = require("express").Router();
const ctrl = require("./user.controller");
router.post("/", ctrl.createUser);
module.exports = router;
```

*Why this is correct:* each layer has one job, the password is stripped by the DTO (and protected by `select: false`), the controller only delegates errors, and the business rules can now be unit-tested without HTTP or a database.

---

**Solution 2 — Validate the environment.**

```js
// config/env.js
require("dotenv").config();
const { z } = require("zod");

const schema = z.object({
  NODE_ENV: z.enum(["development", "test", "production"]).default("development"),
  PORT: z.coerce.number().int().positive().default(3000),
  DATABASE_URL: z.string().url(),
});

const parsed = schema.safeParse(process.env);
if (!parsed.success) {
  console.error("❌ Invalid environment configuration:");
  for (const issue of parsed.error.issues) {
    console.error(`  - ${issue.path.join(".")}: ${issue.message}`);
  }
  process.exit(1); // fail-fast: never boot misconfigured
}

module.exports = Object.freeze(parsed.data);
```

*Why this is correct:* `z.coerce.number()` turns the string `"3000"` into a real number; `.url()` rejects a missing/invalid `DATABASE_URL`; defaults fill gaps; an invalid config crashes the process at startup with a precise, multi-line message rather than failing mysteriously later. `Object.freeze` prevents accidental runtime mutation.

---

**Solution 3 — DTO + validation layer.**

```js
// modules/product/product.validation.js
const { z } = require("zod");
// .strict() rejects unknown fields — clients can't sneak in costPrice or extras.
const createProductSchema = z.object({
  body: z
    .object({
      name: z.string().trim().min(1, "name is required"),
      price: z.number().positive("price must be positive"),
      sku: z.string().trim().min(1).optional(),
    })
    .strict(),
});
module.exports = { createProductSchema };
```

```js
// modules/product/product.dto.js
// Allow-list: costPrice is simply not listed, so it can NEVER be returned.
exports.toProductResponse = (p) => ({
  id: p._id.toString(),
  name: p.name,
  price: p.price,
  sku: p.sku ?? null,
  createdAt: p.createdAt,
});
```

*Why this is correct:* `.strict()` makes the inbound contract reject unknown keys, so a client cannot set internal fields; the DTO is an allow-list, so `costPrice` cannot leak outbound even if the model gains new sensitive fields later.

---

**Solution 4 — Repository + unit-tested service.**

```js
// modules/order/order.service.js (DI-style for easy testing)
const { NotFoundError, UnauthorizedError, ConflictError } = require("../../errors");

function makeOrderService({ orderRepository }) {
  async function cancelOrder(orderId, userId) {
    const order = await orderRepository.findById(orderId);
    if (!order) throw new NotFoundError("Order not found");
    if (order.userId !== userId) throw new UnauthorizedError("Not your order");
    if (order.status !== "pending") throw new ConflictError("Only pending orders can be cancelled");
    return orderRepository.update(orderId, { status: "cancelled" });
  }
  return { cancelOrder };
}
module.exports = makeOrderService;
```

```js
// modules/order/order.service.test.js — no database, fake repo
const test = require("node:test");
const assert = require("node:assert");
const makeOrderService = require("./order.service");

const baseOrder = { id: "o1", userId: "u1", status: "pending" };
const repoWith = (order) => ({
  findById: async () => order,
  update: async (id, patch) => ({ ...order, ...patch }),
});

test("cancels a pending order owned by the user", async () => {
  const svc = makeOrderService({ orderRepository: repoWith({ ...baseOrder }) });
  const result = await svc.cancelOrder("o1", "u1");
  assert.strictEqual(result.status, "cancelled");
});

test("throws NotFoundError when the order is missing", async () => {
  const svc = makeOrderService({ orderRepository: repoWith(null) });
  await assert.rejects(() => svc.cancelOrder("o1", "u1"), /Order not found/);
});

test("throws UnauthorizedError when another user owns the order", async () => {
  const svc = makeOrderService({ orderRepository: repoWith({ ...baseOrder, userId: "u2" }) });
  await assert.rejects(() => svc.cancelOrder("o1", "u1"), /Not your order/);
});

test("throws ConflictError when the order is not pending", async () => {
  const svc = makeOrderService({ orderRepository: repoWith({ ...baseOrder, status: "shipped" }) });
  await assert.rejects(() => svc.cancelOrder("o1", "u1"), /Only pending/);
});
```

*Why this is correct:* the service depends on a repository *interface*, so the fake repo lets all four branches be tested deterministically in milliseconds with no MongoDB. Run with `node --test`.

---

**Solution 5 — Centralized error handling end-to-end.**

```js
// errors/index.js  (as in §10)
class AppError extends Error {
  constructor(message, statusCode = 500) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = true;
    Error.captureStackTrace(this, this.constructor);
  }
}
class NotFoundError extends AppError { constructor(m = "Not found") { super(m, 404); } }
module.exports = { AppError, NotFoundError };
```

```js
// middleware/notFound.js
const { NotFoundError } = require("../errors");
module.exports = (req, res, next) =>
  next(new NotFoundError(`Cannot ${req.method} ${req.originalUrl}`));
```

```js
// middleware/errorHandler.js
const logger = require("../utils/logger");
const { AppError } = require("../errors");
const isProd = process.env.NODE_ENV === "production";

// eslint-disable-next-line no-unused-vars
module.exports = (err, req, res, next) => {
  const operational = err instanceof AppError && err.isOperational;
  const status = operational ? err.statusCode : 500;
  logger.error({ err, method: req.method, url: req.originalUrl }, "request error");
  res.status(status).json({
    error: {
      message: operational ? err.message : "Internal Server Error",
      ...(isProd ? {} : { stack: err.stack }),
    },
  });
};
```

```js
// app.js — order matters: routes, THEN notFound, THEN errorHandler (last)
const express = require("express");
const app = express();
app.use(express.json());
app.use("/api/v1/users", require("./modules/user/user.routes"));
app.use(require("./middleware/notFound"));     // 404 for anything unmatched
app.use(require("./middleware/errorHandler")); // 4-arg handler, registered LAST
module.exports = app;
```

*Why this is correct:* domain errors carry their own status/message; unknown errors become a generic 500 with the stack hidden in production; unmatched routes funnel into the same handler via `notFound`; and every error is logged with method + URL. There is exactly one place that turns errors into responses.

---

**Solution 6 — Twelve-Factor audit.**

| Issue | Factor(s) violated | Fix |
| --- | --- | --- |
| (a) Hard-coded `mongodb://localhost:27017` | **3 (Config)**, and arguably **4 (Backing services)** | Read the DB URL from `process.env.DATABASE_URL` via the validated config module; the same code then runs against any database by changing one env var. |
| (b) Sessions in a module-level `{}` object | **6 (Processes — stateless)** | Externalize session state to a shared backing service (Redis via `connect-redis`, or use stateless JWTs). Now any instance can serve any request and you can scale horizontally. |
| (c) Logs written to `./logs/app.log` with manual rotation | **11 (Logs as event streams)** | Emit structured JSON to stdout with a logger (Pino); let the platform (Docker/k8s/systemd) capture, route, and rotate. The app stops managing files. |
| (d) No shutdown handling | **9 (Disposability)** | Handle `SIGTERM`/`SIGINT`: stop accepting new connections (`server.close()`), finish in-flight requests, close DB connections, then `process.exit(0)`. Prevents dropped requests on deploy. |

*Why this is correct:* each problem maps to a specific factor, and each fix moves state and configuration out of the process and into the environment/backing services, making the app portable, scalable, and safe to deploy.

## Summary / Cheat Sheet

| Concept | One-line takeaway |
| --- | --- |
| **Why structure** | Cost of change scales with how much you must understand; structure shrinks the blast radius. Optimize for change. |
| **Separation of concerns** | One responsibility per module; high cohesion, low coupling. |
| **MVC in Express** | Model = data/Mongoose, View = JSON response, Controller = route handler. Its gap: no home for business logic → add services. |
| **Layered architecture** | Routes → Controllers → Services → Repositories → Models. One job each; dependencies point downward only. |
| **Thin controllers** | Extract input → call service → respond (or `next(err)`). No business logic, no DB. |
| **Services** | Framework-free business logic. If a CLI couldn't call it unchanged, it's in the wrong layer. |
| **Repository pattern** | Abstraction over data access → testable services (fake repos), swappable DB, centralized queries. Skip for tiny apps. |
| **DTOs** | Plain objects defining your API contract; allow-list mapping so the DB schema never leaks (no passwords/tokens/`__v`). |
| **Validation layer** | Reject bad input at the edge with a schema (Zod/Joi) before it reaches controllers/services. |
| **Config management** | One validated, frozen config module; `dotenv` in dev; coerce types; fail-fast at startup. Gitignore `.env`, commit `.env.example`. |
| **Error handling** | Custom operational-error hierarchy; controllers `next(err)`; one 4-arg error middleware registered last; hide stacks in prod. |
| **Logging** | Structured JSON to stdout via Pino/Winston; correlation ids; never `console.log`. |
| **Folder structure** | Feature-based (`modules/user/...`) for real apps; split `app.js` (build) from `server.js` (run). |
| **Naming** | kebab-case files, PascalCase classes, camelCase functions, UPPER_SNAKE env/constants; enforce with ESLint+Prettier. |
| **Twelve-Factor** | Config in env (3), stateless processes (6), logs as streams (11), graceful shutdown (9), dev/prod parity (10). |

> TIP: If you remember only one rule from this entire module: **`req`/`res` live only in controllers; the database lives only in repositories; the brains live in services.** Almost every architectural decision in a Node API follows from keeping those three boundaries clean.
