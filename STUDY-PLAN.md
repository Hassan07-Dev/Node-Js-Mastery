# 5-Week Node.js & Express.js Mastery — Study Plan

A day-by-day, build-as-you-learn roadmap that takes you from "I know JavaScript" to **interview-ready backend engineer** in 35 days. It maps all **31 modules**, **5 projects**, every module's **Practice Exercises**, and the **6 interview-prep question banks** across five focused weeks — with dedicated revision and mock-interview days so nothing is crammed at the end.

This plan assumes you already know JavaScript well but are **new to backend/server-side development**. It explains backend concepts from first principles inside the modules; this document is the schedule and the strategy that ties them together.

**Stack:** Node.js (current LTS) · Express.js (v4 baseline, v5 notes where relevant) · MongoDB + Mongoose · plain JavaScript (CommonJS by default, ES Modules where useful). No TypeScript.

**Your constraints (the plan is built around them):** ~2–4 hours on weekdays, a bit more on weekends, and a **job interview at the end of Week 5**.

---

## How to Use This Plan

Read this section once before Day 1. The difference between "I read a Node course" and "I can answer Node questions under pressure" is *method*, not hours. Five habits do almost all the work.

### 1. Active recall beats re-reading

After each module, **close the file and write down**, from memory, the 5 things you'd put on a flashcard. Then reopen and check. The act of *retrieving* a fact wires it in far more than re-reading it. Every module ends with **Key Interview Points** and a **Summary/Cheat Sheet** — use them as the answer key, not the study material.

> BEST PRACTICE: Keep a single `notes.md` (or a deck in Anki). One line per concept, phrased as a question: *"What happens to `process.nextTick` callbacks relative to Promise microtasks?"* You'll reuse this in Week 5 mock interviews.

### 2. Build, don't just read

Reading teaches *recognition*; building teaches *recall*. **Type every snippet** and run it — do not copy-paste. The projects are non-negotiable: they are your portfolio and your best interview stories. When an interviewer asks "tell me about something you built," you want a real answer with real trade-offs, not a summary of a tutorial.

> TIP: Keep a terminal open the entire time you study. A concept you've executed beats one you've only read. Break things on purpose — delete an `await`, throw inside a callback, return the wrong status code — and watch what happens.

### 3. Do the exercises (and look at the solutions *after* you try)

Every module ends with **Practice Exercises** (with solutions). Attempt each one *before* peeking. A failed attempt followed by the solution sticks better than a smooth read-through. If you're short on time, do the exercises and skip re-reading the prose — the exercises are where the learning compounds.

### 4. Spaced repetition

Memory fades on a predictable curve. Beat it with cheap, scheduled reviews:

- **Next day:** re-read yesterday's module **Summary/Cheat Sheet** (5 min).
- **Each weekend:** skim the week's cheat sheets and run 15–20 questions from the relevant interview bank.
- **Week 5:** the whole week is spaced repetition by design (revision + mock interviews).

### 5. Learn in public / explain it out loud

You don't understand the event loop until you can explain it to a rubber duck without notes. After each major topic, **say the answer out loud** as if in an interview. The mock-interview days in Week 5 formalize this, but start the habit in Week 1.

### Daily rhythm (a ~3-hour weekday)

| Block | Time | What you do |
|-------|------|-------------|
| Warm-up | 10 min | Re-read **yesterday's** cheat sheet (spaced repetition). |
| Read actively | 60–90 min | Work through today's module, typing and running every snippet. |
| Build / exercises | 60–90 min | Do the module's Practice Exercises or push the current project forward. |
| Recall | 10–15 min | Close the file; write 5 recall notes. Say one answer out loud. |

> WARNING — Pitfall: Do not let "reading more modules" crowd out "building the projects." If you fall behind, **drop extra reading, never the project.** Interviewers probe what you've built; they can tell tutorial knowledge from hands-on knowledge in about two questions.

---

## Week 1 — Foundations: JavaScript for Backend, the Node Runtime & the Event Loop

**Theme:** Understand *what Node actually is* — a JavaScript runtime built on V8 and libuv — and master the asynchronous model that every later topic depends on. By Friday you start your first project using only Node core.

| Day | Focus | Read (module files) | Do (exercises / project) | Outcome |
|-----|-------|---------------------|--------------------------|---------|
| **1 (Mon)** | JS refresher *for backend* — closures, `this`, modules-of-thought, error objects | `modules/week-1/01-javascript-essentials.md` | Practice Exercises in module 01; write 5 recall notes | Comfortable with the JS features backend code leans on (closures, higher-order fns, `Error` subclassing). |
| **2 (Tue)** | Async JS deep dive — callbacks → Promises → `async/await`, error propagation | `modules/week-1/02-async-javascript.md` | Practice Exercises in module 02; convert a callback example to `async/await` by hand | Can reason about Promise chaining, `Promise.all/allSettled/race`, and where `await` actually pauses. |
| **3 (Wed)** | What Node.js *is* — V8, libuv, thread pool, non-blocking I/O, CPU- vs I/O-bound | `modules/week-1/03-nodejs-architecture.md` | Practice Exercises in module 03; run the `crypto` thread-pool demo and explain the output | Can explain "single-threaded, event-driven, non-blocking I/O" from first principles. |
| **4 (Thu)** | Module systems — CommonJS `require`/`module.exports` vs ES Modules `import`/`export` | `modules/week-1/04-module-systems.md` | Practice Exercises in module 04; build the same tiny module both ways (`.cjs` and `.mjs`) | Know when to use CJS vs ESM, the `"type"` field, `__dirname` in ESM, and interop pitfalls. |
| **5 (Fri)** | The event loop, timers & `EventEmitter` (intro) | `modules/week-1/05-event-loop.md` | Practice Exercises in module 05; predict output of mixed `setTimeout`/`setImmediate`/`process.nextTick`/Promise snippets | Can order microtasks vs macrotasks and explain each event-loop phase. |
| **6 (Sat)** | **Project day** — start Project 1 (CLI Todo, Node core only) | `projects/project-1-cli-todo.md` (Overview → Step 5) | Build through the storage/render layers; commit as you go | A running CLI that adds and lists todos, persisting to a JSON file. |
| **7 (Sun)** | **Finish Project 1 + Week-1 revision** | `projects/project-1-cli-todo.md` (Steps 6–8 + Run/Test) | Finish the CLI (complete/delete/error handling); re-read modules 02 & 05 cheat sheets; do 15 questions from `interview-prep/04-async-eventloop-questions.md` | A complete, committed CLI app; first interview bank drilled. |

> BEST PRACTICE: The event loop is the **single most-asked Node interview topic**. Don't move to Week 2 until you can, on a whiteboard, place `nextTick`, microtasks, timers, I/O, and `setImmediate` in the right order and explain *why*.

> WARNING — Pitfall: Project 1 uses **only Node core** (`fs`, `path`, `process.argv`) — no npm packages. Resist installing libraries; the point is to feel raw Node before any framework hides it from you.

---

## Week 2 — Node Core Modules: fs, Streams, Events, HTTP & NPM

**Theme:** The standard library. You'll read and write files, stream large data efficiently, build an HTTP server **with no framework**, and learn how npm and the ecosystem work. This is the layer Express is built *on top of* — understanding it makes Express obvious instead of magical.

| Day | Focus | Read (module files) | Do (exercises / project) | Outcome |
|-----|-------|---------------------|--------------------------|---------|
| **8 (Mon)** | Globals & the `process` object — `argv`, `env`, exit codes, signals, `process.nextTick` | `modules/week-2/06-globals-and-process.md` | Practice Exercises in module 06; read config from `process.env` with a safe default | Know the Node global scope (no `window`), how to read env/CLI args, and graceful-exit signals. |
| **9 (Tue)** | File System & `path` — sync vs async vs promises API, path safety | `modules/week-2/07-file-system.md` | Practice Exercises in module 07; refactor Project 1's storage to use `fs/promises` cleanly | Confident with `fs.promises`, `path.join`/`resolve`, and why sync I/O is dangerous in a server. |
| **10 (Wed)** | Streams & Buffers — readable/writable/transform, backpressure, `pipe`/`pipeline` | `modules/week-2/08-streams-and-buffers.md` | Practice Exercises in module 08; stream-copy a large file and pipe a transform | Can explain backpressure and *why* streams beat reading whole files into memory. |
| **11 (Thu)** | Events & `EventEmitter` in depth — listeners, `once`, error events, memory leaks | `modules/week-2/09-events-eventemitter.md` | Practice Exercises in module 09; build a small emitter-based mini pub/sub | Understand Node's event-driven core and the `'error'`-event rule that crashes processes. |
| **12 (Fri)** | Essential core modules — `os`, `util`, `crypto`, `url`, `querystring` | `modules/week-2/10-core-modules.md` | Practice Exercises in module 10; hash a password with `crypto.scrypt`, parse a URL | Know which batteries Node includes and reach for core before npm. |
| **13 (Sat)** | The HTTP module — build a server & a router **without a framework** | `modules/week-2/11-http-module.md` | Practice Exercises in module 11; build a tiny JSON API (GET/POST) using only `http` | Can build, route, and parse request bodies with raw `http` — you'll appreciate Express next week. |
| **14 (Sun)** | NPM & ecosystem **+ Week-2 revision** | `modules/week-2/12-npm-ecosystem.md` | Practice Exercises in module 12; re-read streams & HTTP cheat sheets; do 20 questions from `interview-prep/01-nodejs-questions.md` (Beginner + Intermediate) | Understand `package.json`, semver, lockfiles, `dependencies` vs `devDependencies`, scripts, and `npx`. |

> TIP: When you reach Express in Week 3, mentally map each Express feature back to this week: `req`/`res` are the raw `http` objects, middleware is the manual routing you wrote on Day 13, and `express.json()` is the body-parsing you did by hand. This mapping is gold in interviews.

> WARNING — Pitfall: Streams and backpressure trip up most candidates. If you can only deeply learn one Week-2 topic, make it **streams** — and be ready to explain `pipeline()` vs `pipe()` (error handling differs).

---

## Week 3 — Express.js: Routing, Middleware, REST & Your First API

**Theme:** Now the framework. You'll learn Express's request lifecycle, the middleware model that *is* Express, the `req`/`res` API, and REST design principles — then build a full Blog REST API with proper CRUD. (Mongoose comes Week 4; this week store data in memory or a JSON file, then swap to a DB next week.)

| Day | Focus | Read (module files) | Do (exercises / project) | Outcome |
|-----|-------|---------------------|--------------------------|---------|
| **15 (Mon)** | Intro to Express — what it adds over `http`; app setup; **v4 vs v5** notes | `modules/week-3/13-express-intro.md` | Practice Exercises in module 13; scaffold an Express app and a health-check route | Can stand up an Express server and explain why it exists vs raw `http`. |
| **16 (Tue)** | Routing in depth — route params, query strings, `Router`, route ordering | `modules/week-3/14-routing.md` | Practice Exercises in module 14; build a modular `Router` with params & 404 handling | Can structure routes cleanly with `express.Router()` and handle params/wildcards. |
| **17 (Wed)** | Middleware — the heart of Express; order, `next()`, error-handling middleware | `modules/week-3/15-middleware.md` | Practice Exercises in module 15; write a logger + an auth-gate middleware | Can explain the middleware chain, `next(err)`, and the 4-arg error handler. |
| **18 (Thu)** | The `req` & `res` objects in depth — headers, status, `res.json`, content negotiation | `modules/week-3/16-request-response.md` | Practice Exercises in module 16; set status codes, headers, cookies correctly | Fluent with the `req`/`res` API and proper HTTP status usage. |
| **19 (Fri)** | Static files, templating & body parsing | `modules/week-3/17-static-templating-bodyparsing.md` | Practice Exercises in module 17; serve static assets + parse JSON/urlencoded bodies | Know `express.static`, `express.json()`, and when you'd use a template engine. |
| **20 (Sat)** | REST API design & CRUD principles **+ start Project 2** | `modules/week-3/18-rest-api-crud.md`; `projects/project-2-blog-rest-api.md` (Overview → routing/controllers) | Practice Exercises in module 18; scaffold the Blog API (routes, controllers, in-memory store) | Understand resource modeling, HTTP verbs, status codes, and idempotency; Blog API skeleton runs. |
| **21 (Sun)** | **Build Project 2 + Week-3 revision** | `projects/project-2-blog-rest-api.md` (continue build) | Implement full CRUD for posts; re-read middleware cheat sheet; do 20 questions from `interview-prep/02-express-questions.md` | A working Blog REST API with clean routing, middleware, validation hooks, and error handling. |

> BEST PRACTICE: Test every endpoint with `curl` or a REST client as you build (`curl -X POST localhost:3000/posts -H 'content-type: application/json' -d '{...}'`). Knowing your way around HTTP verbs and status codes by hand is itself an interview signal.

> TIP: Express **v5** is now the default install. Watch for the differences flagged in module 13/14/15 — notably stricter path-matching syntax, and that rejected promises/async errors are now forwarded to error middleware automatically (in v4 you needed a wrapper). Be ready to name at least two v4→v5 changes.

---

## Week 4 — Data & Security: MongoDB, Mongoose, Auth, Authorization & Validation

**Theme:** Persistence and protection. You'll learn MongoDB and Mongoose, wire the Blog API to a real database, then build a complete Authentication & Authorization system — the topics that dominate backend interviews after the event loop.

| Day | Focus | Read (module files) | Do (exercises / project) | Outcome |
|-----|-------|---------------------|--------------------------|---------|
| **22 (Mon)** | MongoDB fundamentals — documents, collections, queries, indexes, the aggregation idea | `modules/week-4/19-mongodb-fundamentals.md` | Practice Exercises in module 19; run CRUD + a couple of queries in the Mongo shell | Understand document modeling, BSON/`_id`, query operators, and when to index. |
| **23 (Tue)** | Mongoose ODM — schemas, models, validation, queries, population, middleware/hooks | `modules/week-4/20-mongoose.md` | Practice Exercises in module 20; define a `Post` schema with validation & timestamps | Can model data with Mongoose, validate at the schema level, and use `populate`. |
| **24 (Wed)** | **Project 2 → database** — swap Blog API's in-memory store for Mongoose | `modules/week-4/20-mongoose.md` (revisit) ; `projects/project-2-blog-rest-api.md` (DB integration) | Connect to MongoDB, replace the data layer, handle async DB errors | Blog REST API now persists to MongoDB with proper validation and error handling — **Project 2 done**. |
| **25 (Thu)** | Authentication — sessions vs JWT, password hashing, login flow **+ start Project 3** | `modules/week-4/21-authentication.md`; `projects/project-3-auth-system.md` (Overview → registration/login) | Practice Exercises in module 21; build register/login with hashed passwords + JWT issuance | Can implement secure signup/login and explain sessions vs JWT trade-offs. |
| **26 (Fri)** | Authorization & web security — RBAC, OWASP basics, CORS, helmet, rate limiting | `modules/week-4/22-authorization-security.md`; `projects/project-3-auth-system.md` (protect routes) | Practice Exercises in module 22; add role-based route guards + security middleware | Can gate routes by role and name the common web attacks (XSS, CSRF, injection) + defenses. |
| **27 (Sat)** | Error handling & input validation **+ finish Project 3** | `modules/week-4/23-error-handling-validation.md`; `projects/project-3-auth-system.md` (finish) | Practice Exercises in module 23; centralize an error handler + validate all inputs | Robust, centralized error handling and validated inputs; **Project 3 done**. |
| **28 (Sun)** | **Week-4 revision (data + security)** | re-read cheat sheets for modules 19–23 | Do `interview-prep/03-mongodb-mongoose-questions.md` (full) and `interview-prep/05-security-questions.md` (Beginner + Intermediate) | Can speak fluently about indexing, schema design, JWT, hashing, and OWASP top risks. |

> WARNING — Pitfall: Never store plaintext passwords and never roll your own crypto. Use the hashing approach the module teaches (e.g., `bcrypt`/`scrypt` with a salt), keep secrets in `process.env`, and **never** log tokens or password hashes. Interviewers *will* ask how you store passwords.

> BEST PRACTICE: A centralized error-handling middleware + a custom `AppError` class (operational vs programmer errors) is the single highest-leverage pattern from this week. Carry it into the capstone.

---

## Week 5 — Advanced Node, Real-Time, the Capstone & Interview Sprint

**Theme:** Round out advanced topics (scaling, performance, architecture, testing, real-time, jobs, deployment, API best practices), build the capstone, then convert everything into interview performance with revision and **mock-interview** days. The week is front-loaded with learning and back-loaded with drilling so you peak on interview day.

| Day | Focus | Read (module files) | Do (exercises / project) | Outcome |
|-----|-------|---------------------|--------------------------|---------|
| **29 (Mon)** | Advanced Node (threads, processes, scaling) + Performance & caching | `modules/week-5/24-advanced-node-scaling.md`; `modules/week-5/25-performance-caching.md` | Practice Exercises in modules 24 & 25; run a `cluster`/`worker_threads` demo; add a simple cache | Can explain clustering, worker threads, the thread pool, and caching strategies for scale. |
| **30 (Tue)** | Project architecture & best practices + Testing | `modules/week-5/26-project-architecture.md`; `modules/week-5/27-testing.md` | Practice Exercises in modules 26 & 27; write unit + integration tests for a Project-3 route | Know layered architecture (routes→controllers→services→models) and can test an Express app. |
| **31 (Wed)** | Real-time (WebSockets/Socket.IO) + **Project 4** (Real-Time Chat) | `modules/week-5/28-realtime-websockets.md`; `projects/project-4-realtime-chat.md` | Build the Socket.IO chat (rooms, broadcast, auth handshake) | A working real-time chat app; can explain WebSockets vs HTTP polling — **Project 4 done**. |
| **32 (Thu)** | File uploads, email & background jobs + API best practices & docs | `modules/week-5/29-file-uploads-jobs.md`; `modules/week-5/31-api-best-practices.md` | Practice Exercises in modules 29 & 31; add file upload + a queued job; sketch API docs | Can handle multipart uploads, offload work to a queue, and design versioned, documented APIs. |
| **33 (Fri)** | Deployment, logging & DevOps **+ start the Capstone** | `modules/week-5/30-deployment-devops.md`; `projects/project-5-capstone-ecommerce-api.md` (Overview → architecture/models) | Practice Exercises in module 30; scaffold the E-Commerce API (architecture, models, auth) | Understand env config, structured logging, process managers, and deployment; capstone skeleton runs. |
| **34 (Sat)** | **Capstone build day** + System design | `projects/project-5-capstone-ecommerce-api.md` (core build); `interview-prep/06-system-design-scenarios.md` | Implement products/cart/orders with auth, validation, and centralized errors; read system-design scenarios | A substantial capstone API demonstrating everything; can talk through system-design trade-offs. |
| **35 (Sun)** | **Mock-interview day + final revision** | all 6 banks in `interview-prep/` (rapid-fire + advanced sections); every module **Summary/Cheat Sheet** | Full timed mock interview (out loud): event loop, Express, Mongo, auth, scaling, system design; finish/polish the capstone README | **Interview-ready.** Can explain and defend every project and core concept on demand. |

> TIP: Treat Day 35 like the real thing. Set a 45-minute timer, have a friend (or a rubber duck + your `notes.md`) fire questions from the banks, and **answer out loud without looking**. Record yourself once — hearing your own "ums" around the event loop is the fastest way to fix them.

> BEST PRACTICE: For the capstone and Project 2/3, write a tight README with: what it does, how to run it, the architecture diagram in words, and 3 trade-offs you made. This *is* your interview script for "tell me about a project."

> WARNING — Pitfall: If you fall behind in Week 5, **protect the capstone and the mock-interview day.** Trim reading (skim cheat sheets instead of full modules) before you trim building or drilling. Recruiters test depth on what you built, not breadth of topics skimmed.

---

## Weekly Milestones

By the end of each week you should be able to **build the thing** and **explain it without notes**.

### End of Week 1 — Foundations
- **Build:** a complete command-line Todo app using only Node core (`fs`, `path`, `process`).
- **Explain:** what Node.js is (V8 + libuv), single-threaded non-blocking I/O, the event-loop phases, microtasks vs macrotasks, `process.nextTick` vs `setImmediate`, CommonJS vs ES Modules, and Promises vs `async/await`.

### End of Week 2 — Node Core
- **Build:** an HTTP server and JSON API **with no framework**, plus stream-based file processing.
- **Explain:** the `process` object and env config, sync vs async `fs`, streams & backpressure, `EventEmitter` and the `'error'`-event rule, useful core modules (`crypto`, `url`, `os`), and how npm/semver/lockfiles work.

### End of Week 3 — Express
- **Build:** a Blog REST API with modular routing, middleware, body parsing, validation hooks, and centralized error handling (in-memory or JSON store).
- **Explain:** the middleware chain and `next()`/`next(err)`, routing and route ordering, the `req`/`res` API, REST resource design and correct status codes, and at least two Express **v4 vs v5** differences.

### End of Week 4 — Data & Security
- **Build:** the Blog API persisted to MongoDB via Mongoose, plus a full Authentication & Authorization system (hashed passwords, JWT, role-based guards, security middleware).
- **Explain:** document modeling and indexing, Mongoose schemas/validation/`populate`/hooks, sessions vs JWT, password hashing, RBAC, and the common OWASP risks with their defenses.

### End of Week 5 — Advanced + Interview-Ready
- **Build:** a real-time chat app (Socket.IO) and a substantial E-Commerce capstone API with layered architecture, tests, file uploads, background jobs, logging, and deploy-readiness.
- **Explain:** clustering and worker threads, caching and performance, layered architecture, testing strategy, WebSockets vs HTTP, API versioning/docs, and end-to-end **system-design** scenarios — fluently and out loud.

---

## Final-Week Interview Checklist

Tick each box only when you can **explain it out loud** *and* **show code/a project that uses it**. Grouped by area; sourced from the modules and the `interview-prep/` banks.

### Node.js Runtime & Event Loop  (`interview-prep/01-nodejs-questions.md`, `04-async-eventloop-questions.md`)
- [ ] What Node.js is: V8 + libuv + the C++ bindings, and what each provides.
- [ ] Single-threaded, event-driven, non-blocking I/O — explained from first principles.
- [ ] The event-loop phases (timers → pending → poll → check → close) in order.
- [ ] Microtasks vs macrotasks; where Promise callbacks and `queueMicrotask` run.
- [ ] `process.nextTick` vs `setImmediate` vs `setTimeout(fn, 0)` — order and use cases.
- [ ] The libuv thread pool: what uses it (fs, crypto, zlib, DNS) and its default size.
- [ ] CPU-bound vs I/O-bound work; why a CPU-bound task blocks the loop and how to offload it.
- [ ] Predict the output of a mixed timers/Promise/`nextTick` snippet.

### Async JavaScript  (`interview-prep/04-async-eventloop-questions.md`)
- [ ] Callbacks → Promises → `async/await`; callback hell and how Promises fix it.
- [ ] `Promise.all` vs `allSettled` vs `race` vs `any`.
- [ ] Error propagation in async code; why an unhandled rejection can crash the process.
- [ ] Sequential vs concurrent `await`; not awaiting inside `forEach`.

### Node Core Modules  (`interview-prep/01-nodejs-questions.md`)
- [ ] CommonJS vs ES Modules: `require`/`module.exports` vs `import`/`export`, `"type"`, interop.
- [ ] `fs` sync vs async vs promises; why sync I/O is dangerous in a server.
- [ ] Streams: readable/writable/duplex/transform, **backpressure**, `pipe` vs `pipeline`.
- [ ] `Buffer`: what it is and when you encounter it.
- [ ] `EventEmitter`: listeners, `once`, the `'error'` event, and listener-leak warnings.
- [ ] The `process` object: `argv`, `env`, exit codes, signals, graceful shutdown.
- [ ] npm: `package.json`, semver ranges, lockfiles, `dependencies` vs `devDependencies`, `npx`.

### Express.js  (`interview-prep/02-express-questions.md`)
- [ ] Why Express over raw `http`; the request lifecycle.
- [ ] Middleware: the chain, `next()`, `next(err)`, and the 4-argument error handler.
- [ ] Routing: params, query strings, `express.Router()`, route ordering, 404 handling.
- [ ] `req`/`res` essentials: `res.json`, status codes, headers, cookies.
- [ ] Body parsing (`express.json`/`urlencoded`) and serving static files.
- [ ] REST design: resource modeling, correct verbs/status codes, idempotency.
- [ ] At least two **Express v4 vs v5** differences (path matching, async error handling).

### MongoDB & Mongoose  (`interview-prep/03-mongodb-mongoose-questions.md`)
- [ ] Documents, collections, `_id`/BSON, query operators, and when to add an index.
- [ ] Embedding vs referencing; basic data-modeling trade-offs.
- [ ] Mongoose schemas, models, schema-level validation, and `timestamps`.
- [ ] `populate` and the N+1 concern; lean queries.
- [ ] Schema middleware/hooks (e.g., hashing a password on save).

### Authentication, Authorization & Security  (`interview-prep/05-security-questions.md`)
- [ ] Sessions vs JWT: how each works and their trade-offs.
- [ ] Password hashing (salt + bcrypt/scrypt); never store plaintext or log secrets.
- [ ] Role-based access control (RBAC) via middleware.
- [ ] OWASP basics: XSS, CSRF, injection, and their defenses.
- [ ] CORS, `helmet`, rate limiting, and keeping secrets in `process.env`.
- [ ] Centralized error handling: operational vs programmer errors, no stack traces to clients.

### Advanced Node, Architecture & System Design  (`interview-prep/06-system-design-scenarios.md`)
- [ ] Scaling: `cluster`, `worker_threads`, and a process manager.
- [ ] Caching strategies and common performance wins.
- [ ] Layered architecture: routes → controllers → services → models.
- [ ] Testing: unit vs integration; testing an Express route.
- [ ] WebSockets/Socket.IO vs HTTP polling; when real-time is the right tool.
- [ ] File uploads, background jobs/queues, structured logging, and deployment basics.
- [ ] Walk through a system-design scenario (e.g., "design a URL shortener / a chat backend").

### Projects — be ready to demo and defend
- [ ] **Project 1 — CLI Todo** (`projects/project-1-cli-todo.md`): Node core, `fs`, `process.argv`.
- [ ] **Project 2 — Blog REST API** (`projects/project-2-blog-rest-api.md`): Express + Mongoose CRUD.
- [ ] **Project 3 — Auth System** (`projects/project-3-auth-system.md`): JWT, RBAC, security middleware.
- [ ] **Project 4 — Real-Time Chat** (`projects/project-4-realtime-chat.md`): Socket.IO.
- [ ] **Project 5 — E-Commerce Capstone** (`projects/project-5-capstone-ecommerce-api.md`): full layered API.
- [ ] For each: can state what it does, how to run it, the architecture, and 3 trade-offs you made.

---

> Final word: you don't need to memorize this document — you need to *do* it. Build every project, do every exercise, and on Day 35 explain it all out loud. That's the difference between knowing Node.js and being **interview-ready**. Good luck.
