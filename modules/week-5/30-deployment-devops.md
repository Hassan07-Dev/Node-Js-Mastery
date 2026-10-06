# Deployment, Logging & DevOps

Up to this point in the course you have built an application that runs beautifully on *your* machine. You type `node server.js`, you hit `http://localhost:3000`, and everything works. But "works on my laptop" is not a product. Between your laptop and a paying user there is a vast, unglamorous territory: how do you run the process so it survives a crash at 3am? How do you know *what happened* when something breaks at 3am if you were asleep? How do you ship a new version without taking the site down? How do you keep your database connection string out of your Git history? How do you go from a `git push` to live traffic without manually SSHing into a box and praying?

This is **DevOps** — the discipline of operating software in production. For a backend developer it is not optional. An endpoint that returns the right JSON but falls over under load, leaks secrets, or goes dark with no logs is not "done." This module takes your working app and makes it **production-grade**: configured safely, observably logged, resiliently run, containerized, fronted by a real web server over HTTPS, shipped through an automated pipeline, and watched in production so you find out about problems before your users tweet about them.

We will move in the natural order a real deployment follows. First we prepare the app to *behave* differently in production (config and secrets). Then we make it *talk to us* (logging). Then we make it *stay up* (process management and graceful shutdown). Then we *package and ship* it (Docker, reverse proxy, HTTPS, CI/CD, hosting platforms). Finally we *watch* it (monitoring and error tracking). Everything targets **Node.js current LTS**, **Express 4/5**, **MongoDB + Mongoose**, and plain JavaScript with CommonJS, with notes where ES Modules or Express 5 change the picture.

> This module is tooling-heavy. You do not need to install everything at once. Each section names its dependency (`npm install winston`, etc.) so you can follow along incrementally. Docker and Nginx sections assume you can install Docker Desktop and have basic shell access; if you cannot right now, read for understanding and return when you have an environment.

## What You'll Learn

By the end of this module you will be able to:

- Explain the role of **`NODE_ENV`**, load configuration from **environment variables**, and understand why config belongs in the environment, not the code.
- Manage **secrets** safely with `.env` files, `dotenv`, and platform secret stores, and keep them out of version control.
- **Validate configuration at startup** so the app fails fast and loudly on misconfiguration instead of crashing mysteriously later.
- Set up **structured, leveled logging** with **Winston** (levels, transports, JSON output, child loggers) and **HTTP request logging** with **Morgan**, and articulate exactly why `console.log` is wrong for production.
- Run Node in production with **PM2**: start apps, use **cluster mode** to use all CPU cores, write an **`ecosystem.config.js`**, do **zero-downtime reloads**, manage logs, and configure **startup on boot**.
- Implement **graceful shutdown** and **health-check endpoints** that orchestrators and load balancers can use.
- **Containerize** a Node app with a production **Dockerfile**, a **`.dockerignore`**, and **multi-stage builds**, and run it alongside MongoDB with **`docker-compose`**.
- Understand the role of a **reverse proxy (Nginx)** and how to serve your app over **HTTPS** with TLS termination.
- Build a **CI/CD pipeline with GitHub Actions** that tests, builds, and deploys on push.
- Compare hosting options — **Render, Railway, Fly.io, and a self-managed VPS** — and choose appropriately.
- Add **monitoring, error tracking (Sentry), and basic observability** (the three pillars: logs, metrics, traces).

---

## 1. Preparing for Production: Config, Environment & Secrets

### The core principle: config lives in the environment

The single most important idea in this section comes from a widely cited set of guidelines for building deployable apps called the **Twelve-Factor App**. Factor III says: **store config in the environment**. Config is anything that varies between deployments — the database URL, the port, API keys, feature flags, the log level. Code is the same in every environment; config is what differs between your laptop, your staging server, and production.

Why does this matter? Three concrete reasons:

1. **Security.** If your production database password is a string literal in `db.js`, it is now in your Git history forever, visible to anyone who clones the repo. Environment variables keep secrets out of code.
2. **Portability.** The same built artifact (or Docker image) can run in dev, staging, and prod with zero code changes — you just feed it different environment variables. This is what makes "build once, deploy anywhere" possible.
3. **No accidental cross-talk.** You will never accidentally point a test run at the production database because the connection string is supplied externally, per environment.

> WARNING — Pitfall: Hardcoding any URL, key, password, or environment-specific value in source code is a production bug waiting to happen. The moment you write `const apiKey = 'sk_live_...'` you have created a security incident. Treat every such literal as a code smell.

### `NODE_ENV`: the one variable everything reads

`NODE_ENV` is a *de facto* standard environment variable that signals which environment the app is running in. By convention its values are `development`, `production`, and `test`. It is "magic" in two ways:

- **Express reads it.** When `NODE_ENV === 'production'`, Express enables view caching, disables verbose error stack traces in responses, and makes other internal optimizations. Forgetting to set it in production means Express runs in a slower, leakier mode.
- **Many libraries read it.** React, many loggers, and others change behavior based on it.

```js
// Reading NODE_ENV. process.env values are ALWAYS strings (or undefined).
const env = process.env.NODE_ENV || 'development'; // default to dev if unset

const isProd = env === 'production';
const isDev = env === 'development';
const isTest = env === 'test';

console.log(`Running in ${env} mode`); // never log secrets, env name is fine
```

```bash
# Setting NODE_ENV when starting the app (Linux / macOS).
NODE_ENV=production node server.js

# On Windows PowerShell the syntax differs:
$env:NODE_ENV="production"; node server.js

# cross-env normalizes this across OSes (great for npm scripts):
npm install --save-dev cross-env
# then in package.json: "start:prod": "cross-env NODE_ENV=production node server.js"
```

> WARNING — Pitfall: `process.env.NODE_ENV` is **always a string**. `process.env.PORT` is the string `"3000"`, not the number `3000`. Comparisons like `if (process.env.DEBUG === true)` are *always false* because the right side is a boolean and the left is a string. Coerce explicitly: `Number(process.env.PORT)`, `process.env.DEBUG === 'true'`.

> WARNING — Pitfall: Never set `NODE_ENV=production` while running `npm install` in CI if you still need devDependencies (test frameworks, build tools). With `NODE_ENV=production`, `npm install` skips `devDependencies`. Install everything first, build/test, *then* deploy with `NODE_ENV=production`. This bites people constantly.

### `.env` files and `dotenv`

In development, exporting a dozen variables by hand every time you open a terminal is tedious. The standard solution is a **`.env` file**: a plain-text file of `KEY=value` pairs at the project root, loaded into `process.env` at startup by the **`dotenv`** package.

```bash
npm install dotenv
```

```bash
# .env  — NEVER commit this file. It contains real secrets.
NODE_ENV=development
PORT=3000
MONGO_URI=mongodb://127.0.0.1:27017/myapp_dev
JWT_SECRET=super-secret-dev-key-change-me
LOG_LEVEL=debug
SENTRY_DSN=
```

Load it as the **very first thing** your app does, before any module that reads `process.env`:

```js
// server.js — load env BEFORE requiring anything that reads process.env.
require('dotenv').config(); // reads .env into process.env

// Now safe to require modules that depend on env vars:
const app = require('./app');
const port = Number(process.env.PORT) || 3000;

app.listen(port, () => console.log(`Listening on ${port}`));
```

For ES Modules, ordering is trickier because `import` statements hoist and run before top-level code. Use the `dotenv/config` side-effect import, which runs on import:

```js
// server.mjs (ES Modules) — this import has the side effect of loading .env.
import 'dotenv/config';
import app from './app.js'; // app can now read process.env safely
```

> Node.js 20.6+ has **built-in `.env` support** via the `--env-file` flag: `node --env-file=.env server.js`. This needs no dependency. It is great for simple cases, but `dotenv` still offers more features (variable expansion, multiple files, programmatic API), so both remain common.

### Keeping secrets out of Git

The cardinal rule: **`.env` must never be committed.** Add it to `.gitignore` on day one. Commit instead a `.env.example` that documents *which* variables exist, with placeholder (not real) values, so a new teammate knows what to fill in.

```bash
# .gitignore
node_modules/
.env
.env.local
*.log
npm-debug.log*
```

```bash
# .env.example  — SAFE to commit. Documents the shape, hides the secrets.
NODE_ENV=development
PORT=3000
MONGO_URI=mongodb://127.0.0.1:27017/myapp_dev
JWT_SECRET=
LOG_LEVEL=debug
SENTRY_DSN=
```

> WARNING — Pitfall: If you ever *did* commit a secret, removing it in a later commit is **not enough** — it is still in the Git history. You must rotate (regenerate) the leaked credential immediately and, if needed, scrub history with tools like `git filter-repo` or BFG. Assume a committed secret is compromised.

In production you typically do **not** ship a `.env` file at all. Instead the hosting platform injects environment variables: Render and Railway have an "Environment Variables" UI, Docker uses `-e` flags or an `env_file`, Kubernetes uses Secrets, and cloud providers offer dedicated secret managers (AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault). The app code does not care *how* the variable got into `process.env` — only that it is there. That uniformity is the whole point.

### Centralize and validate config at startup

Scattering `process.env.X` reads throughout your codebase is fragile: a typo (`process.env.MONGO_URL` vs `MONGO_URI`) silently yields `undefined`, and you discover the misconfiguration only when something explodes deep in a request handler. The professional pattern is a **single config module** that reads, coerces, and **validates** every value once at startup — and crashes immediately with a clear message if anything is wrong. This is the **fail-fast** principle: it is far better to refuse to start than to start in a broken state.

A hand-rolled config module:

```js
// config/index.js — the one place env vars are read. Fail fast on bad config.
require('dotenv').config();

function required(name) {
  const value = process.env[name];
  if (value === undefined || value === '') {
    // Throwing here means the process exits before app.listen — fail fast.
    throw new Error(`Missing required environment variable: ${name}`);
  }
  return value;
}

const config = {
  env: process.env.NODE_ENV || 'development',
  port: Number(process.env.PORT) || 3000,
  mongoUri: required('MONGO_URI'),       // must be present, no fallback
  jwtSecret: required('JWT_SECRET'),     // secrets are always required
  logLevel: process.env.LOG_LEVEL || 'info',
  sentryDsn: process.env.SENTRY_DSN || null, // optional, defaults to off
};

// Freeze so nothing mutates config at runtime by accident.
module.exports = Object.freeze(config);
```

For anything beyond toy apps, use a schema validator so you get type coercion, ranges, allowed values, and a single aggregated error report. Two popular choices are **`envalid`** and **`zod`**.

```bash
npm install envalid
```

```js
// config/index.js — validated config with envalid.
require('dotenv').config();
const { cleanEnv, str, port, url, num } = require('envalid');

// cleanEnv reads process.env, validates, coerces types, and on failure
// prints a clear report and exits the process automatically.
const config = cleanEnv(process.env, {
  NODE_ENV: str({ choices: ['development', 'production', 'test'], default: 'development' }),
  PORT: port({ default: 3000 }),                 // validates it's a valid port number
  MONGO_URI: url(),                              // must be a valid URL
  JWT_SECRET: str(),                            // required string, no default
  LOG_LEVEL: str({ choices: ['error', 'warn', 'info', 'http', 'debug'], default: 'info' }),
  SENTRY_DSN: str({ default: '' }),             // optional
});

module.exports = config;
```

Now every module imports `require('./config')` and reads `config.mongoUri` — typed, validated, autocomplete-friendly, and impossible to misspell silently. If `MONGO_URI` is missing or malformed, the app never even reaches `app.listen`; it tells you exactly what is wrong and exits.

> BEST PRACTICE: Read `process.env` in exactly **one** module (your config module) and import that everywhere else. Validate it at startup. This turns a whole class of "works locally, mysteriously dies in prod" bugs into a loud, immediate, self-explaining crash.

---

## 2. Logging Done Right

### Why `console.log` is not a logging strategy

`console.log` got you through development, but in production it is actively harmful. Here is the honest list of why:

- **No levels.** You cannot distinguish a routine info message from a critical error. You cannot say "show me only errors in prod, but everything in dev." Every line is equally loud.
- **No structure.** `console.log('User', userId, 'failed login from', ip)` produces a string a human can read but a machine cannot query. Log aggregation tools (Datadog, Loki, CloudWatch, ELK) want **structured JSON** so you can filter by `userId`, `level`, `requestId`.
- **No metadata.** No timestamps, no severity, no service name, no request correlation — all things you desperately want at 3am.
- **It is synchronous to stdout/stderr.** Under heavy load, `console.log` can block the event loop because writes to a terminal or pipe can be synchronous, hurting throughput.
- **No transports.** You cannot route logs to a file, an external service, *and* the console simultaneously, or change destinations per environment.

A real logging library fixes all of this. We use **Winston** for application logging and **Morgan** for HTTP request logging.

### Logging levels and what they mean

Levels let you assign severity and then filter. Winston's default levels (npm levels), from most to least severe:

| Level | Numeric | Use it for |
| --- | --- | --- |
| `error` | 0 | A failure that needs attention — caught exceptions, failed DB writes, 500s. |
| `warn` | 1 | Something unexpected but handled — deprecated route hit, retry succeeded. |
| `info` | 2 | Normal operational milestones — server started, user registered, job ran. |
| `http` | 3 | HTTP request logs (Morgan can feed into this). |
| `verbose` | 4 | More detail than info, for diagnosing. |
| `debug` | 5 | Developer-facing detail — variable values, branch taken. |
| `silly` | 6 | Everything, firehose. |

The logger has a **configured level**; it emits a message only if the message's level is **at or above** (i.e., numerically at or below) the configured level. Set `LOG_LEVEL=info` in production and `error`/`warn`/`info` flow through while `debug` is silently dropped — no code changes, just config.

### Winston: a production logger

```bash
npm install winston
```

```js
// logger.js — a single shared Winston logger for the whole app.
const winston = require('winston');
const config = require('./config');

const { combine, timestamp, json, colorize, printf, errors } = winston.format;

// In dev we want pretty, human-readable colored lines.
const devFormat = combine(
  colorize(),
  timestamp({ format: 'HH:mm:ss' }),
  errors({ stack: true }), // if you log an Error, include its stack trace
  printf(({ level, message, timestamp, stack }) =>
    `${timestamp} ${level}: ${stack || message}`)
);

// In prod we want structured JSON so log tooling can parse and index it.
const prodFormat = combine(
  timestamp(),                 // ISO timestamp
  errors({ stack: true }),     // serialize Error objects with their stack
  json()                       // emit one JSON object per line
);

const logger = winston.createLogger({
  level: config.logLevel,                          // e.g. 'info' in prod
  format: config.env === 'production' ? prodFormat : devFormat,
  defaultMeta: { service: 'myapp' },               // attached to every log line
  transports: [
    new winston.transports.Console(),              // always log to stdout
  ],
  // Catch crashes that would otherwise be silent:
  exceptionHandlers: [new winston.transports.Console()],
  rejectionHandlers: [new winston.transports.Console()],
});

module.exports = logger;
```

Now use it everywhere instead of `console.log`:

```js
const logger = require('./logger');

logger.info('Server started', { port: 3000 });           // structured metadata
logger.warn('Falling back to default region');
logger.error('Failed to save order', { orderId, err: error.message });

// Logging an Error object directly captures its stack (because of errors()):
try {
  await saveOrder();
} catch (err) {
  logger.error('Order save failed', err); // stack included in prod JSON
}
```

In production this emits machine-parseable lines like:

```json
{"level":"error","message":"Failed to save order","orderId":"abc123","service":"myapp","timestamp":"2026-06-19T10:00:00.000Z"}
```

### Transports: where logs go

A **transport** is a destination. The power of Winston is that one logger can fan out to many. The Console transport writes to stdout/stderr. The File transport writes to disk (often with rotation). You can also send to external services.

```js
// Adding a rotating file transport (production, when writing to local disk).
// npm install winston-daily-rotate-file
const DailyRotateFile = require('winston-daily-rotate-file');

logger.add(new DailyRotateFile({
  filename: 'logs/app-%DATE%.log', // %DATE% is replaced per day
  datePattern: 'YYYY-MM-DD',
  zippedArchive: true,             // gzip old logs
  maxSize: '20m',                  // rotate when a file hits 20 MB
  maxFiles: '14d',                 // keep 14 days, then delete
  level: 'info',
}));

// A separate file for errors only, so on-call can grep one small file:
logger.add(new winston.transports.File({
  filename: 'logs/error.log',
  level: 'error',
}));
```

> BEST PRACTICE: In **containerized** environments (Docker, Kubernetes, most PaaS), do **not** write logs to files inside the container. Log to **stdout/stdent** only and let the platform collect, ship, and rotate them. Files inside an ephemeral container disappear when it restarts. This too is a Twelve-Factor rule (Factor XI: treat logs as event streams). File transports are for traditional VPS deployments.

### Child loggers and request correlation

A killer feature for debugging is **request correlation**: attach a unique ID to every log line emitted during a single HTTP request, so you can reconstruct exactly what happened in one request among thousands. Winston **child loggers** make this clean.

```js
// middleware/requestLogger.js
const { randomUUID } = require('crypto');
const logger = require('../logger');

module.exports = function attachLogger(req, res, next) {
  // Reuse an upstream ID (from a load balancer header) or make one.
  const requestId = req.headers['x-request-id'] || randomUUID();
  // child() returns a logger that injects { requestId } into every line.
  req.log = logger.child({ requestId });
  res.setHeader('x-request-id', requestId); // echo back for client/log correlation
  next();
};
```

```js
// In any handler, use req.log so every line carries the same requestId:
app.post('/orders', (req, res) => {
  req.log.info('Creating order', { items: req.body.items.length });
  // ... if anything fails later, req.log.error(...) shares the requestId
});
```

### Morgan: HTTP request logging

You also want a one-line record of *every HTTP request*: method, path, status, response time. That is exactly what **Morgan** does, and it is purpose-built for it.

```bash
npm install morgan
```

```js
const morgan = require('morgan');
const logger = require('./logger');

// In development: the concise, colored 'dev' format straight to the console.
if (config.env === 'development') {
  app.use(morgan('dev'));
  // Example output: GET /users 200 12.3 ms - 1024
}
```

In production you want Morgan's output to flow through Winston (so it is JSON, gets shipped to the same place, and carries the `http` level). Morgan accepts a custom `stream` whose `write` method you control:

```js
// Pipe Morgan into Winston at the 'http' level for unified, structured logs.
const morganFormat = config.env === 'production' ? 'combined' : 'dev';

app.use(morgan(morganFormat, {
  stream: {
    // Morgan writes a string with a trailing newline; trim and forward it.
    write: (message) => logger.http(message.trim()),
  },
  // Skip noisy health checks so they don't drown out real traffic:
  skip: (req) => req.url === '/healthz',
}));
```

The `combined` format is the Apache standard log line (includes IP, user agent, referrer) — good for production. The `dev` format is short and colorized — good for local work. You can also define fully custom formats and even structured JSON tokens.

> TIP: Morgan logs **HTTP traffic**; Winston logs **everything else** (and receives Morgan's stream). That division — Morgan for the request/response envelope, Winston for application events — is the standard Express logging setup and a great thing to be able to articulate in an interview.

> WARNING — Pitfall: Never log full request bodies, passwords, tokens, credit card numbers, or other PII/secrets. It is shockingly common to `logger.info('login', req.body)` and dump plaintext passwords into your logs (which then get shipped to a third-party service). Redact sensitive fields before logging.

---

## 3. Process Management with PM2

### The problem PM2 solves

Run `node server.js` directly in production and a long list of bad things can happen:

- The process **crashes** on an unhandled exception — and stays dead. Your site is down until a human notices and restarts it.
- You close your SSH session and the process, tied to that terminal, **dies with it**.
- A single Node process uses **one CPU core**. Your 8-core server is 87% idle while one core is pegged.
- The server **reboots** (kernel update, power blip) and your app does not come back.
- You deploy a new version and have to **kill and restart**, dropping in-flight requests and causing a visible blip.

**PM2** is a production process manager for Node that solves all of these: it daemonizes your app, restarts it on crash, runs it across all cores in **cluster mode**, restarts it on reboot, reloads it with zero downtime, and gives you logs and monitoring.

```bash
npm install -g pm2   # install globally on the server
```

### Starting and managing apps

```bash
# Start an app. PM2 keeps it alive and restarts it if it crashes.
pm2 start server.js --name myapp

# Common lifecycle commands:
pm2 list                 # show all managed processes, status, CPU, memory
pm2 stop myapp           # stop (but keep in the process list)
pm2 restart myapp        # hard restart (brief downtime)
pm2 reload myapp         # zero-downtime reload (cluster mode) — see below
pm2 delete myapp         # remove from PM2 entirely
pm2 logs myapp           # tail combined stdout/stderr logs
pm2 logs myapp --lines 200   # last 200 lines
pm2 monit                # live dashboard: CPU, memory, logs per process
pm2 describe myapp       # detailed info: restarts, uptime, path, env
```

### Cluster mode: using every CPU core

Node is single-threaded for your JavaScript. To use a multi-core machine you run **multiple instances** of your app behind a load balancer that distributes connections among them. PM2 does this for you with **cluster mode** — and crucially, it uses Node's built-in `cluster` module so all instances **share the same port** (the OS/round-robin distributes incoming connections). No code changes required, *provided your app is stateless*.

```bash
# Run one instance per CPU core ('max'), load-balanced, sharing the port.
pm2 start server.js --name myapp -i max

# Or a specific number of instances:
pm2 start server.js --name myapp -i 4
```

> WARNING — Pitfall: Cluster mode multiplies your processes, so **in-memory state is no longer shared**. In-memory sessions, in-process rate-limit counters, in-memory caches, and `setInterval` cron jobs will each exist *per instance* and behave inconsistently (a user's session lives in instance 2 but their next request hits instance 3). Externalize shared state to Redis/MongoDB, and run scheduled jobs in a single dedicated process, before turning on cluster mode.

### The `ecosystem.config.js` file

Passing flags on the command line does not scale. The canonical way to configure PM2 is an **ecosystem file** — a JS file describing your apps, their instances, env vars, and behavior. Generate a template with `pm2 init` or write it by hand:

```js
// ecosystem.config.js — the source of truth for how PM2 runs your app.
module.exports = {
  apps: [
    {
      name: 'myapp',
      script: './server.js',
      instances: 'max',          // one per CPU core
      exec_mode: 'cluster',      // enable cluster mode (load-balanced)
      watch: false,              // NEVER watch files in production
      max_memory_restart: '500M',// restart an instance if it exceeds 500 MB (leak guard)
      // Default env (used unless an --env is specified):
      env: {
        NODE_ENV: 'development',
        PORT: 3000,
      },
      // Activated with: pm2 start ecosystem.config.js --env production
      env_production: {
        NODE_ENV: 'production',
        PORT: 8080,
      },
      // Logging:
      error_file: './logs/pm2-error.log',
      out_file: './logs/pm2-out.log',
      log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
      merge_logs: true,          // merge logs from all cluster instances
      // Graceful shutdown coordination (see Section 4):
      kill_timeout: 5000,        // ms to wait for graceful exit before SIGKILL
      wait_ready: true,          // wait for process.send('ready') before marking up
      listen_timeout: 10000,     // give the app up to 10s to signal ready
    },
  ],
};
```

```bash
# Start using the ecosystem file with the production env block:
pm2 start ecosystem.config.js --env production

# Reload with the same env (zero downtime):
pm2 reload ecosystem.config.js --env production
```

> WARNING — Pitfall: Do **not** enable `watch: true` in production. File-watching auto-restart is a development convenience; in production it can restart your app on log writes or deploy file churn, causing chaos. Use `nodemon` in dev, PM2 without watch in prod.

### `restart` vs `reload`: zero-downtime deploys

This distinction is a frequent interview question.

| | `pm2 restart` | `pm2 reload` |
| --- | --- | --- |
| What it does | Kills all instances, then starts them | Restarts instances **one at a time**, waiting for each new one to be ready before killing the old |
| Downtime | Brief — there is a moment with no live process | **Zero** — there is always a live instance serving traffic |
| Requires | Anything | **Cluster mode** |
| Use when | Config that needs a clean slate; dev | Deploying new code to production |

With `reload` in cluster mode, you deploy new code with no dropped requests: PM2 spins up a new instance with the new code, waits until it is listening, routes traffic to it, then gracefully drains and kills an old one — repeating across all instances. This pairs with **graceful shutdown** (next section): old instances finish their in-flight requests before exiting.

### Startup on boot

To make PM2 (and your apps) survive a server reboot, generate and install a startup script for your init system (systemd, etc.), then **save** the current process list as the resurrection target:

```bash
pm2 startup            # prints a command to run with sudo; run it
# (it registers PM2 with systemd so it launches on boot)

pm2 save               # snapshot the current process list to disk
# On boot, PM2 resurrects exactly these processes.

pm2 unstartup          # undo the boot integration if needed
```

> BEST PRACTICE: Always run `pm2 save` after you have your processes in the desired state. A common gotcha is configuring `pm2 startup` but forgetting `pm2 save`, so after a reboot PM2 starts but resurrects nothing. Also re-run `pm2 save` whenever you change which apps run.

### Basic monitoring

`pm2 monit` gives a live terminal dashboard (CPU/memory/logs per process). `pm2 list` shows restart counts — a steadily climbing restart count is a red flag that your app is crash-looping. For hosted dashboards, alerting, and historical metrics there is **PM2 Plus** (a paid SaaS), but for many teams `pm2 monit` plus shipping logs to a log platform plus a proper monitoring tool (Section 9) is enough.

> Note: In a fully containerized world (Docker/Kubernetes), you often **do not use PM2 at all** — the orchestrator handles restarts, scaling, and rollouts, and the convention is one process per container. PM2 shines on a VPS or bare-metal server. Know both worlds; we cover containers next.

---

## 4. Graceful Shutdown & Health Checks

### Why graceful shutdown matters

When a deploy happens, a server scales down, or an orchestrator moves your container, your process receives a **termination signal** — usually `SIGTERM` (then `SIGKILL` if it does not exit in time). The naive Node default on `SIGTERM` is to exit immediately. That is bad: any request currently being processed is abandoned mid-flight (the user gets a connection reset), and open database connections are not closed cleanly.

**Graceful shutdown** means: on `SIGTERM`, stop accepting *new* connections, let *in-flight* requests finish, close database and other resources, then exit. Done right, deploys and scaling become invisible to users.

```js
// server.js — graceful shutdown done properly.
require('dotenv').config();
const app = require('./app');
const mongoose = require('mongoose');
const config = require('./config');
const logger = require('./logger');

async function start() {
  await mongoose.connect(config.mongoUri); // connect DB before listening
  logger.info('MongoDB connected');

  const server = app.listen(config.port, () => {
    logger.info(`Listening on ${config.port}`);
    // If using PM2 wait_ready, signal that we're up:
    if (process.send) process.send('ready');
  });

  // Reusable shutdown routine.
  async function shutdown(signal) {
    logger.info(`${signal} received — shutting down gracefully`);

    // 1) Stop accepting new connections; the callback fires once all
    //    existing connections have finished.
    server.close(async (err) => {
      if (err) {
        logger.error('Error during server.close', err);
        process.exit(1);
      }
      try {
        // 2) Close other resources (DB, queues, redis, etc.).
        await mongoose.connection.close(false); // false = wait for ops to finish
        logger.info('Resources closed — exiting cleanly');
        process.exit(0);
      } catch (closeErr) {
        logger.error('Error closing resources', closeErr);
        process.exit(1);
      }
    });

    // 3) Safety net: if graceful shutdown hangs, force-exit.
    setTimeout(() => {
      logger.error('Graceful shutdown timed out — forcing exit');
      process.exit(1);
    }, 10000).unref(); // unref so this timer doesn't keep the process alive
  }

  // Listen for the signals orchestrators send.
  process.on('SIGTERM', () => shutdown('SIGTERM')); // sent by Docker/K8s/PM2
  process.on('SIGINT', () => shutdown('SIGINT'));   // Ctrl+C in a terminal
}

// Last-resort handlers: log, then exit so PM2/orchestrator restarts us clean.
process.on('unhandledRejection', (reason) => {
  logger.error('Unhandled promise rejection', { reason });
  process.exit(1);
});
process.on('uncaughtException', (err) => {
  logger.error('Uncaught exception', err);
  process.exit(1); // do NOT try to keep running after an uncaught exception
});

start().catch((err) => {
  logger.error('Failed to start', err);
  process.exit(1);
});
```

> WARNING — Pitfall: After an `uncaughtException`, the process is in an **undefined state** — you do not know what is corrupted. The correct response is to log and **exit**, letting your process manager start a fresh, clean process. Do not "catch and continue." Use `uncaughtException` for last-breath logging, not for recovery.

> WARNING — Pitfall: Docker only sends `SIGTERM` to **PID 1**. If your container's entrypoint is `npm start`, npm becomes PID 1 and often does **not forward** the signal to your node process — so your graceful shutdown handler never runs and Docker kills you with `SIGKILL` after a grace period. Fix: make `node` itself PID 1 (`CMD ["node", "server.js"]`), or use an init like `tini` (`docker run --init`). More in the Docker section.

### Health-check endpoints

Load balancers, orchestrators, and uptime monitors need a cheap endpoint to ask "are you alive and ready?" There are two distinct concepts, and conflating them causes outages:

- **Liveness** (`/healthz`): Is the process up and the event loop responsive? If this fails, *restart* me. It must be cheap and must **not** depend on external services (a DB blip should not get your healthy app killed).
- **Readiness** (`/readyz`): Am I ready to *serve traffic*? This *does* check dependencies (DB connected, cache reachable). If this fails, the load balancer should *stop sending traffic* but not kill me — I may recover (e.g., DB reconnects).

```js
// routes/health.js
const express = require('express');
const mongoose = require('mongoose');
const router = express.Router();

// Liveness: trivially true if the event loop can answer. Keep it dumb & fast.
router.get('/healthz', (req, res) => {
  res.status(200).json({ status: 'ok', uptime: process.uptime() });
});

// Readiness: only "ready" if dependencies are actually reachable.
router.get('/readyz', (req, res) => {
  // mongoose.connection.readyState === 1 means 'connected'.
  const dbUp = mongoose.connection.readyState === 1;
  if (!dbUp) {
    return res.status(503).json({ status: 'not ready', db: 'down' });
  }
  res.status(200).json({ status: 'ready', db: 'up' });
});

module.exports = router;
```

> BEST PRACTICE: Keep liveness checks dependency-free, and exclude health endpoints from your request logging (the `skip` in Morgan earlier) — load balancers hit them every few seconds and they would otherwise dominate your logs.

---

## 5. Containerizing with Docker

### Why containers

"It works on my machine" dies the moment your machine differs from the server — a different Node version, a missing system library, a different OS. A **container** packages your app *together with* its exact runtime, dependencies, and a minimal OS into a single, immutable **image**. That image runs identically on your laptop, in CI, and in production. Containers are the lingua franca of modern deployment: every major platform (Render, Fly.io, AWS ECS, Kubernetes) speaks Docker images.

A quick vocabulary check: a **Dockerfile** is the recipe; an **image** is the built artifact (a layered, read-only filesystem snapshot); a **container** is a running instance of an image. You `build` an image from a Dockerfile, then `run` containers from the image.

### A naive Dockerfile, then a good one

The naive version that beginners write works but is bad — large, slow to rebuild, and runs as root:

```dockerfile
# BAD — do not ship this. Shown to contrast with the good version below.
FROM node:22
WORKDIR /app
COPY . .                 # copies everything, including node_modules & .env
RUN npm install          # installs dev deps too; no cache optimization
EXPOSE 3000
CMD ["node", "server.js"] # runs as root
```

Problems: the full `node:22` image is ~1 GB; copying everything before `npm install` defeats Docker's layer caching (any source change reinstalls all deps); it installs devDependencies; and it runs as root (a security risk).

### A production-grade multi-stage Dockerfile

A **multi-stage build** uses one stage to install/build and a second, clean stage that copies only what runtime needs — yielding a small, secure final image.

```dockerfile
# syntax=docker/dockerfile:1

# ---- Stage 1: dependencies & build ----
FROM node:22-alpine AS build
WORKDIR /app

# Copy ONLY manifests first. This layer is cached and only invalidated
# when dependencies actually change — not on every source edit.
COPY package*.json ./

# Install ALL deps (including dev) — we may need them to build/transpile/test.
# npm ci is reproducible (uses package-lock.json) and faster in CI than install.
RUN npm ci

# Now copy the source (changes here don't bust the deps layer above).
COPY . .

# If you had a build step (e.g. bundling), it would go here:
# RUN npm run build

# ---- Stage 2: lean runtime image ----
FROM node:22-alpine AS runtime
WORKDIR /app

# Run as the built-in non-root 'node' user, never as root.
ENV NODE_ENV=production
# Install ONLY production dependencies for a smaller, safer image.
COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force

# Copy the application code (and built assets) from the build stage.
COPY --from=build /app .

# Drop privileges.
USER node

EXPOSE 3000

# Run node directly so it is PID 1 and receives SIGTERM for graceful shutdown.
CMD ["node", "server.js"]
```

Key choices explained:

- **`node:22-alpine`** — Alpine Linux is tiny (~5 MB base), so images are small and have a smaller attack surface. (Caveat: Alpine uses musl libc; rarely, native modules misbehave — if so, use `node:22-slim`, a small Debian-based image.)
- **Copy `package*.json` before `COPY . .`** — this is the single most important Docker performance trick. Docker caches each layer; because deps rarely change, `npm ci` is skipped on most rebuilds, making iteration fast.
- **`npm ci` not `npm install`** — `ci` installs exactly what is in `package-lock.json`, deletes `node_modules` first for a clean slate, and is deterministic. Always use it in Docker/CI.
- **`--omit=dev`** in the runtime stage — no test frameworks or build tools shipped to production.
- **`USER node`** — the official Node images include a non-root `node` user. Running as root inside a container is a real security risk if the app is compromised.
- **`CMD ["node", ...]` (exec form, not shell form)** — exec form makes node PID 1, so it receives `SIGTERM` directly and your graceful shutdown works.

### `.dockerignore`

Just as `.gitignore` keeps junk out of Git, **`.dockerignore`** keeps junk out of the **build context** (the files Docker uploads to the daemon and can `COPY`). This makes builds faster, images smaller, and prevents leaking secrets.

```bash
# .dockerignore
node_modules        # never copy host node_modules; we install fresh in the image
npm-debug.log
.git
.gitignore
.env                # CRITICAL: never bake secrets into the image
.env.*
Dockerfile
.dockerignore
*.md
coverage
.nyc_output
logs
.vscode
.DS_Store
```

> WARNING — Pitfall: Copying host `node_modules` into the image is a classic bug. If your host is macOS/Windows and the container is Linux, native modules compiled for your host **will not run** in the container. Always `.dockerignore` `node_modules` and install inside the image.

> WARNING — Pitfall: Never `COPY .env` into an image, and never put secrets in `ENV`/`ARG` — they are baked into image layers and visible to anyone who pulls the image (`docker history` reveals them). Inject secrets at *runtime* via `-e`, `--env-file`, or your platform's secret store.

### Building and running

```bash
# Build the image, tagging it 'myapp:1.0.0'.
docker build -t myapp:1.0.0 .

# Run a container, mapping host port 3000 to container port 3000,
# injecting env vars at runtime (NOT baked in), with an init for signals.
docker run --init -p 3000:3000 \
  -e NODE_ENV=production \
  -e MONGO_URI="mongodb://host.docker.internal:27017/myapp" \
  -e JWT_SECRET="..." \
  --name myapp myapp:1.0.0

# Or load all vars from a file (the file is read by Docker, not copied in):
docker run --init -p 3000:3000 --env-file .env.production myapp:1.0.0

docker ps              # list running containers
docker logs -f myapp   # tail container stdout/stderr (your Winston JSON!)
docker stop myapp      # sends SIGTERM, then SIGKILL after grace period
```

The `--init` flag inserts a tiny init process (`tini`) as PID 1 that properly forwards signals and reaps zombies — a clean way to make graceful shutdown reliable even if you ever use shell-form CMD.

> TIP: Add a Docker `HEALTHCHECK` so the runtime knows if your app is healthy, using the readiness endpoint from Section 4: `HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost:3000/healthz || exit 1`.

### `docker-compose`: app + MongoDB together

In development you want your app *and* a MongoDB instance running together with one command, networked so they can talk. **Docker Compose** declares a multi-container stack in one YAML file.

```yaml
# docker-compose.yml — run the app and MongoDB as a unit.
services:
  app:
    build: .                       # build from the Dockerfile in this dir
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
      # 'mongo' is the service name below; Compose gives it that DNS name
      # on the internal network, so the app reaches Mongo at mongo:27017.
      MONGO_URI: mongodb://mongo:27017/myapp
      JWT_SECRET: ${JWT_SECRET}    # pulled from a .env file or the shell
    depends_on:
      mongo:
        condition: service_healthy # wait until Mongo passes its healthcheck
    restart: unless-stopped

  mongo:
    image: mongo:7
    ports:
      - "27017:27017"              # optional: expose Mongo to the host for tools
    volumes:
      - mongo-data:/data/db        # named volume = data SURVIVES container removal
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  mongo-data:                       # declared so Compose manages it
```

```bash
docker compose up --build      # build + start the whole stack
docker compose up -d           # detached (background)
docker compose logs -f app     # tail just the app's logs
docker compose down            # stop & remove containers (volume persists)
docker compose down -v         # also delete the volume (DESTROYS DB data)
```

Two things to internalize: services reach each other by **service name** as a hostname (`mongo`, not `localhost`), because each container has its own network namespace; and **named volumes** are how you persist data — without `mongo-data`, your database vanishes the moment the container is removed.

> WARNING — Pitfall: Inside a container, `localhost` means *that container*, not the host or another container. The app cannot reach Mongo at `mongodb://localhost:27017` in Compose — it must use the service name `mongodb://mongo:27017`. To reach a service on the host machine from inside a container, use `host.docker.internal` (Docker Desktop) or the host's IP.

---

## 6. Reverse Proxy (Nginx) & HTTPS

### Why put Nginx in front of Node

You *can* expose your Node app directly on port 80/443, but in production a **reverse proxy** like **Nginx** (or Caddy, or a cloud load balancer) almost always sits in front. A reverse proxy is a server that receives client requests and forwards them to your backend, then relays the response back. It earns its place by handling concerns Node should not:

- **TLS/HTTPS termination.** Nginx handles the certificate and encryption, talking plain HTTP to your local Node process. Node is freed from crypto overhead, and you manage certs in one place.
- **Serving static files** efficiently (images, CSS, the built frontend) — Nginx is far faster at this than Express.
- **Load balancing** across multiple Node instances/servers.
- **Compression, caching, rate limiting, and request buffering** at the edge.
- **A single public entry point** on ports 80/443, hiding your app on an internal port (e.g., 3000) that is never exposed to the internet.
- **Security:** absorbing slow-client attacks, hiding server details, enforcing limits.

### A minimal Nginx reverse-proxy config

```nginx
# /etc/nginx/sites-available/myapp — reverse proxy to Node on :3000
server {
    listen 80;
    server_name example.com www.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;   # forward to local Node app
        proxy_http_version 1.1;

        # Preserve client info so the app sees the real client, not Nginx:
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;     # http or https

        # Needed for WebSockets (Socket.IO etc.):
        proxy_set_header Upgrade           $http_upgrade;
        proxy_set_header Connection        "upgrade";
    }
}
```

Because Nginx terminates TLS and forwards plain HTTP, your Express app receives `http`, not `https`. The `X-Forwarded-*` headers carry the original details — but Express ignores them by default for security. You must opt in:

```js
// app.js — tell Express to trust the proxy so req.protocol, req.ip,
// secure cookies, and rate limiters see the REAL client values.
app.set('trust proxy', 1); // trust the first proxy hop (Nginx)

app.get('/whoami', (req, res) => {
  res.json({
    ip: req.ip,            // now the real client IP from X-Forwarded-For
    protocol: req.protocol // now 'https' when the user used HTTPS
  });
});
```

> WARNING — Pitfall: Without `app.set('trust proxy', ...)` behind a proxy, `req.ip` is always the proxy's IP (so per-IP rate limiting is useless), `req.protocol` is always `http` (so "redirect to HTTPS" loops forever), and `secure: true` cookies are never sent. But do **not** blindly `trust proxy: true` (trust all) when directly internet-facing — clients could spoof `X-Forwarded-For`. Trust the specific number of proxy hops you actually have.

### HTTPS with Let's Encrypt

HTTPS is non-negotiable in production: it encrypts traffic, prevents tampering, and is required by browsers for many APIs and by SEO. **Let's Encrypt** issues free, automatically renewable certificates, and **Certbot** automates getting and installing them into Nginx.

```bash
# On the server (Ubuntu/Debian):
sudo apt install certbot python3-certbot-nginx

# Certbot obtains a cert AND rewrites your Nginx config to serve HTTPS,
# set up the cert paths, and add an HTTP->HTTPS redirect:
sudo certbot --nginx -d example.com -d www.example.com

# Certbot installs a systemd timer that auto-renews before expiry.
# Verify the renewal process works (dry run):
sudo certbot renew --dry-run
```

After Certbot runs, Nginx listens on 443 with the cert and redirects 80 → 443. Your Node app never touches TLS — Nginx terminates it and proxies plain HTTP to `127.0.0.1:3000`. This separation (TLS at the edge, app on an internal port) is the standard, secure topology.

> TIP: If you would rather not manage Nginx + Certbot by hand, **Caddy** is a modern alternative that obtains and renews TLS certificates **automatically** with near-zero config. On managed platforms (Render, Railway, Fly.io) HTTPS is provided for you entirely — you never touch certs at all. Know that the *concept* (TLS termination at a reverse proxy/edge) is the same everywhere.

---

## 7. CI/CD with GitHub Actions

### What CI/CD is and why

**CI (Continuous Integration)** means every push automatically runs your linter and test suite, so broken code is caught in minutes, not in production. **CD (Continuous Delivery/Deployment)** means that once checks pass, the code is automatically built and shipped (to staging, or all the way to production). The payoff: humans stop manually running tests and copying files to servers, deploys become routine and reversible, and the pipeline is the same for everyone — no "it built on my machine."

**GitHub Actions** is GitHub's built-in CI/CD. You describe a **workflow** in a YAML file under `.github/workflows/`. A workflow has **triggers** (e.g., on push), **jobs** (which run on **runners**, fresh VMs), and **steps** (commands or reusable **actions**).

### A CI workflow: test on every push and PR

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:           # also run on PRs targeting any branch

jobs:
  test:
    runs-on: ubuntu-latest    # fresh Ubuntu VM per run

    # Spin up a real MongoDB as a service container for integration tests.
    services:
      mongo:
        image: mongo:7
        ports:
          - 27017:27017

    strategy:
      matrix:
        node-version: [20.x, 22.x]   # test across multiple Node LTS lines

    steps:
      - uses: actions/checkout@v4     # clone the repo into the runner

      - name: Use Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'                # cache ~/.npm between runs for speed

      - name: Install dependencies
        run: npm ci                   # reproducible install from lockfile

      - name: Lint
        run: npm run lint --if-present

      - name: Run tests
        run: npm test
        env:
          # Point tests at the service container above:
          MONGO_URI: mongodb://localhost:27017/test
          JWT_SECRET: test-secret
          NODE_ENV: test
```

This runs on every push to `main` and every pull request, across Node 20 and 22, with a throwaway MongoDB attached. A red check on a PR blocks the merge (if you enable branch protection) — broken code never reaches `main`.

### A CD workflow: build a Docker image and deploy

A common pattern: on a push to `main` (after CI passes), build a Docker image, push it to a registry (here, GitHub Container Registry), and trigger a deploy. Secrets (registry tokens, deploy hooks) live in **GitHub repository secrets**, never in the YAML.

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write          # allow pushing to GitHub Container Registry

    steps:
      - uses: actions/checkout@v4

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}   # auto-provided by Actions

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          # Tag with both the commit SHA (immutable) and 'latest':
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.sha }}
            ghcr.io/${{ github.repository }}:latest

      - name: Trigger production deploy
        # Many platforms (Render, etc.) expose a "deploy hook" URL you POST to.
        run: curl -fsSL -X POST "${{ secrets.RENDER_DEPLOY_HOOK }}"
```

> BEST PRACTICE: Store every credential as a **GitHub repository/organization secret** and reference it as `${{ secrets.NAME }}`. Never paste tokens into workflow files (they are committed and visible). Tag images with the immutable commit SHA so you always know exactly what is deployed and can roll back to a precise version.

> TIP: Make deploys depend on CI passing. Use `needs: test` to chain jobs, or use **GitHub Environments** with required reviewers for a manual approval gate before production. For deploying to a VPS, a common step uses `appleboy/ssh-action` to SSH in and run `docker compose pull && docker compose up -d`.

---

## 8. Deploying to Platforms: Tradeoffs

There is no single "best" place to deploy — it depends on how much operational control you want versus how much you want handled for you. The spectrum runs from fully managed **PaaS** (Platform as a Service: push code, they do the rest) to a raw **VPS** (Virtual Private Server: a Linux box you own end to end).

| Platform | Model | Strengths | Tradeoffs / watch-outs | Best for |
| --- | --- | --- | --- | --- |
| **Render** | PaaS | Dead-simple: connect GitHub, auto-deploy on push; free TLS; managed Postgres/Redis; built-in health checks & zero-downtime deploys | Less low-level control; can get pricey at scale; free tier services **spin down when idle** (cold starts) | Teams wanting Heroku-style simplicity with modern pricing |
| **Railway** | PaaS | Excellent DX; one-click databases (incl. MongoDB); usage-based pricing; great for prototypes and side projects | Usage billing can surprise you; less mature for large-scale ops | Fast iteration, prototypes, small-to-mid apps |
| **Fly.io** | PaaS-ish (runs containers/VMs at edge) | Deploys Docker images close to users in many regions; great for low latency & global apps; persistent volumes; scales to zero | More concepts to learn (regions, volumes, `fly.toml`); networking is more involved | Globally distributed, latency-sensitive apps |
| **VPS** (DigitalOcean, Hetzner, Linode, EC2) | IaaS (you own the box) | Full control; cheapest at scale; no platform lock-in; run anything | **You** manage OS updates, security, Nginx, TLS, PM2/Docker, backups, monitoring — real ops work | Cost-sensitive scale, special requirements, learning ops |

### How deployment looks on each

- **Render / Railway:** Connect your GitHub repo. The platform detects Node, runs `npm ci && npm run build`, and starts it with your `start` script (or builds your Dockerfile if present). You set environment variables in their dashboard. Push to `main` → automatic deploy. HTTPS, health checks, and rolling deploys are handled for you. This is the fastest path from code to URL.

- **Fly.io:** You provide a Dockerfile and a `fly.toml`. `fly launch` scaffolds it; `fly deploy` builds your image and rolls it out to one or more regions. You manage secrets with `fly secrets set` and attach volumes for stateful data.

- **VPS:** You provision an Ubuntu server, harden it (SSH keys only, firewall, non-root user), install Node/Docker, install and configure Nginx + Certbot for TLS, run your app under PM2 or Docker, set up `pm2 startup`/`docker compose` with restart policies, and wire up log shipping, monitoring, and backups. Your CI/CD (Section 7) SSHes in to deploy. Maximum control, maximum responsibility.

> BEST PRACTICE: Start on a managed PaaS (Render/Railway). The operational time you save is worth more than the marginal hosting cost while you are small. Move to a VPS or container orchestrator only when cost at scale, specific compliance/control needs, or platform limitations actually force the issue — not preemptively. "We might need Kubernetes someday" is rarely a reason to take on its complexity today.

> WARNING — Pitfall: Free tiers that "spin down when idle" (Render free web services, some others) cause **cold starts** — the first request after idle waits several seconds for the container to boot. Fine for demos, bad for anything user-facing. Also, **never run your database on the same ephemeral container as your app**; use a managed database or a dedicated, backed-up data store so a redeploy never wipes your data.

---

## 9. Monitoring, Error Tracking & Observability

### The three pillars of observability

Logging tells you what happened in one place; **observability** is the broader ability to understand your system's internal state from its outputs. It rests on three pillars:

1. **Logs** — discrete, timestamped events (you built this with Winston). Great for "what exactly happened in this request."
2. **Metrics** — numeric measurements over time (requests/sec, p95 latency, error rate, memory, event-loop lag). Great for "is the system healthy *right now* and trending which way," and for alerting.
3. **Traces** — the path of a single request across services and functions, with timing per span. Great for "*where* is the latency in this slow request."

A mature setup has all three. For a single-service Node app, prioritize good logs and error tracking first, add metrics + alerting next, and adopt tracing once you have multiple services.

### Error tracking with Sentry

Logs are passive — someone has to look at them. **Error tracking** is active: it captures every unhandled exception (and ones you report), **groups** identical errors together, attaches rich context (stack trace, request, user, release version, breadcrumbs), and **alerts** you. **Sentry** is the most common choice and has excellent Node/Express support.

```bash
npm install @sentry/node
```

```js
// instrument.js — MUST be imported BEFORE any other module so Sentry can
// auto-instrument them. Import it at the very top of your entry file.
const Sentry = require('@sentry/node');
const config = require('./config');

Sentry.init({
  dsn: config.sentryDsn,                  // from env; empty = disabled
  environment: config.env,                // tag events as prod/staging/etc.
  release: process.env.GIT_SHA,           // tie errors to a deployed version
  tracesSampleRate: 0.1,                  // sample 10% of requests for perf tracing
  // Only enable when a DSN is set so dev runs don't ship noise:
  enabled: Boolean(config.sentryDsn),
});
```

```js
// server.js (entry point) — Sentry first, then everything else.
require('./instrument'); // <-- before requiring express/app
const express = require('express');
const Sentry = require('@sentry/node');
const app = express();

// ... your routes and middleware ...

// Sentry's Express error handler must come AFTER your routes but
// BEFORE your own error-handling middleware (Express 4/5):
Sentry.setupExpressErrorHandler(app);

// Your own error handler (last):
app.use((err, req, res, next) => {
  // The error was already captured by Sentry above; respond cleanly.
  req.log?.error('Request failed', err);
  res.status(err.status || 500).json({ error: 'Internal Server Error' });
});
```

You can also report handled errors and add context manually:

```js
try {
  await chargeCard(order);
} catch (err) {
  // Attach context that will appear on the Sentry issue:
  Sentry.withScope((scope) => {
    scope.setTag('feature', 'checkout');
    scope.setUser({ id: req.user.id });
    Sentry.captureException(err);
  });
  throw err; // rethrow so your normal error handling still runs
}
```

> BEST PRACTICE: Always set `environment` and `release` in Sentry. `environment` keeps dev noise out of prod dashboards; `release` (your commit SHA) lets Sentry tell you exactly which deploy introduced a regression and even mark issues as resolved in a release. Pipe the SHA in from CI (`GIT_SHA`).

### Metrics, uptime, and alerting

- **Uptime monitoring:** A service like UptimeRobot, Better Stack, or Pingdom hits your `/healthz` endpoint from outside every minute and alerts you (email/Slack/SMS) if it goes down. This is the cheapest, highest-value monitoring you can add — set it up day one.
- **APM / metrics:** Tools like Datadog, New Relic, Grafana Cloud, or self-hosted **Prometheus + Grafana** collect metrics (latency, throughput, error rate, CPU/memory, **event-loop lag**) and let you build dashboards and alerts. For Prometheus, the `prom-client` npm package exposes a `/metrics` endpoint Prometheus scrapes.
- **What to alert on:** error rate spiking, p95/p99 latency rising, the health check failing, memory climbing toward your `max_memory_restart`, and event-loop lag growing (the classic Node "something is blocking the event loop" symptom).

```js
// Minimal Prometheus metrics endpoint with prom-client.
// npm install prom-client
const client = require('prom-client');
client.collectDefaultMetrics(); // CPU, memory, event-loop lag, GC, etc.

const httpDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'route', 'status'],
});

app.use((req, res, next) => {
  const end = httpDuration.startTimer();
  res.on('finish', () => {
    // Use req.route?.path (not req.url) to avoid label explosion from IDs.
    end({ method: req.method, route: req.route?.path || 'unknown', status: res.statusCode });
  });
  next();
});

// Prometheus scrapes this endpoint on an interval:
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});
```

> WARNING — Pitfall: Do not put high-cardinality values (user IDs, raw URLs with IDs) in metric **labels**. Each unique label combination is a separate time series; `/users/12345` as a label creates a new series per user and will overwhelm your metrics backend. Use the route *pattern* (`/users/:id`) instead.

> BEST PRACTICE: Monitoring is only useful if it reaches a human. Wire alerts into a channel people actually watch (Slack/PagerDuty), and tune thresholds to avoid alert fatigue — an alert that fires constantly gets ignored, and then the one that matters gets ignored too.

---

## Common Pitfalls & Gotchas

- **Committing `.env`.** Secrets end up in Git history forever. Add `.env` to `.gitignore` from the first commit; commit a `.env.example` instead. If leaked, rotate the secret — deleting the file is not enough.
- **Forgetting `NODE_ENV=production`.** Express and many libraries run in slower, more verbose, less secure modes. Set it explicitly in production.
- **Treating env vars as the wrong type.** `process.env.PORT` is the string `"3000"`, and `process.env.DEBUG === true` is always false. Coerce explicitly.
- **`npm install` with `NODE_ENV=production` in CI before building.** It skips devDependencies, so your build/test tooling vanishes. Install everything, test/build, *then* set production.
- **Using `console.log` in production.** No levels, no structure, no transports, and it can block the event loop under load. Use Winston; pipe Morgan through it.
- **Logging secrets/PII.** `logger.info('login', req.body)` ships plaintext passwords to your log service. Redact first.
- **Cluster mode with in-memory state.** PM2 `-i max` runs N processes that do not share memory; in-memory sessions, rate limiters, and `setInterval` jobs break. Externalize state (Redis) and isolate cron jobs first.
- **`pm2 startup` without `pm2 save`.** After a reboot PM2 starts but resurrects nothing. Always `pm2 save` after changing your process list.
- **Not handling `SIGTERM`.** Deploys and scaling kill in-flight requests and leave connections open. Implement graceful shutdown.
- **`npm start` as the Docker entrypoint.** npm becomes PID 1 and swallows `SIGTERM`, so graceful shutdown never runs and Docker `SIGKILL`s you. Use `CMD ["node", "server.js"]` or `--init`.
- **Copying host `node_modules` into a Docker image.** Native modules built for your host OS break in the Linux container. `.dockerignore` it and `npm ci` inside the image.
- **Baking secrets into images** via `COPY .env` or `ENV SECRET=...`. They are visible in image layers. Inject at runtime.
- **`localhost` inside a container.** It means *that container*. Use the Compose service name (`mongo`) or `host.docker.internal`.
- **No `trust proxy` behind Nginx.** `req.ip` is the proxy, `req.protocol` is always `http` (HTTPS redirect loops), and secure cookies break. Set `app.set('trust proxy', 1)` — but never blindly `true` when internet-facing.
- **Running the database in the app's ephemeral container/dyno.** A redeploy wipes your data. Use a managed or volume-backed database.
- **High-cardinality metric labels.** Putting IDs in labels creates a new time series per value and crushes your metrics backend. Use route patterns.

---

## Best Practices

- **Config in the environment, read in one place.** A single validated config module; fail fast at startup on bad config; `process.env` appears nowhere else.
- **Never log secrets; always log structured.** JSON logs in prod, leveled, with a `requestId` for correlation. Morgan for HTTP, Winston for everything (including Morgan's stream).
- **Log to stdout in containers; to rotated files only on a VPS.** Let the platform handle log collection where possible.
- **Run multiple processes for multiple cores** (PM2 cluster mode or one container per core), but externalize all shared state first.
- **Zero-downtime deploys:** `pm2 reload` (or rolling container deploys) + graceful shutdown + readiness checks so traffic only hits ready instances.
- **Small, secure, reproducible images:** multi-stage build, `node:*-alpine` or `-slim`, `npm ci --omit=dev` for runtime, `USER node`, `.dockerignore`, runtime-injected secrets.
- **TLS at the edge:** terminate HTTPS at Nginx/Caddy/the platform; keep Node on an internal port; set `trust proxy`.
- **Automate everything:** CI runs lint + tests on every PR; CD builds an immutable image tagged by commit SHA and deploys on green; credentials live in CI secrets.
- **Start managed, graduate deliberately:** PaaS first; move to a VPS/orchestrator only when a concrete need forces it.
- **Observe before you need to:** uptime monitor on `/healthz`, Sentry with `environment` + `release`, metrics with sane alerts in a channel humans watch.
- **Health checks split by purpose:** cheap dependency-free liveness; dependency-aware readiness.

---

## Key Interview Points

- **Why store config in environment variables?** Keeps secrets out of code/Git, lets one build run in any environment, and prevents accidental cross-environment mistakes (Twelve-Factor Factor III). Read and validate them once at startup; fail fast on misconfiguration.
- **What does `NODE_ENV=production` actually do?** It is a convention many libraries read; Express enables view caching and disables verbose error output, and tooling optimizes. It is a string, not magic — your code and dependencies must respect it.
- **Why not `console.log` in production?** No levels, no structure (machines can't query it), no metadata, no multiple transports, and synchronous writes can block the event loop. Use a logger (Winston) with leveled, structured (JSON) output and transports.
- **Winston vs Morgan?** Morgan logs HTTP request/response lines (method, path, status, latency); Winston is a general application logger with levels/transports. The standard setup pipes Morgan's output through Winston so all logs are unified and structured.
- **What is PM2 cluster mode and its caveat?** It runs N instances (one per core) sharing a port via Node's `cluster` module for load balancing and CPU utilization. Caveat: instances don't share memory, so in-memory state (sessions, rate limits, cron) must be externalized.
- **`pm2 restart` vs `pm2 reload`?** `restart` kills then starts (brief downtime); `reload` restarts instances one at a time waiting for readiness (zero downtime), requires cluster mode, and pairs with graceful shutdown.
- **What is graceful shutdown and why?** On `SIGTERM`, stop accepting new connections, finish in-flight requests, close DB/resources, then exit — so deploys/scaling don't drop requests or leak connections. After an `uncaughtException`, log and exit (don't continue in an undefined state).
- **Liveness vs readiness checks?** Liveness = "is the process alive?" (cheap, no dependencies; failure → restart). Readiness = "can I serve traffic?" (checks DB/deps; failure → stop routing traffic, don't kill).
- **Why a multi-stage Docker build?** Build/install (incl. dev deps) in one stage, copy only runtime artifacts into a clean, small final image; with `node:alpine`, `npm ci --omit=dev`, and `USER node` you get a small, secure, reproducible image. Copy `package*.json` before source to leverage layer caching.
- **Why a reverse proxy (Nginx)?** TLS termination, efficient static file serving, load balancing, compression/caching/rate limiting, and a single hardened public entry point — keeping Node on an internal port. Behind it, set `trust proxy` so `req.ip`/`req.protocol`/secure cookies work.
- **What is CI/CD?** CI runs tests/lint automatically on each push/PR to catch breakage early; CD automatically builds and deploys passing code. Benefits: fast feedback, reproducible deploys, no manual server steps; secrets stored in the CI provider, never in YAML.
- **The three pillars of observability?** Logs (discrete events), metrics (numbers over time, for alerting/trends), traces (a request's path/timing across the system). Error tracking (Sentry) groups exceptions with context and alerts; uptime monitoring on the health endpoint is the cheapest high-value check.
- **`SIGTERM` and Docker PID 1?** Docker sends `SIGTERM` only to PID 1; if `npm start` is PID 1 it may not forward the signal, so graceful shutdown never runs. Make `node` PID 1 or use `--init`/`tini`.

---

## Practice Exercises

Work through these in order; each builds on the last. Solutions follow.

1. **Validated config + secret hygiene.** Take an app that reads `process.env.PORT`, `process.env.MONGO_URI`, and `process.env.JWT_SECRET` directly in several files. Create a single `config/index.js` that loads `.env`, validates these (port is a valid port, Mongo URI is a URL, JWT secret is required), and exits with a clear message if any is missing. Add `.env` to `.gitignore` and create a `.env.example`.

2. **Production logging.** Add a Winston logger that emits colorized human-readable logs in development and JSON in production, controlled by `LOG_LEVEL`. Add Morgan that uses the `dev` format in development and pipes through Winston at the `http` level in production, skipping `/healthz`. Replace all `console.log` calls.

3. **Graceful shutdown + health checks.** Add `/healthz` (liveness, dependency-free) and `/readyz` (readiness, checks Mongoose connection) endpoints. Implement graceful shutdown on `SIGTERM`/`SIGINT` that stops the server, closes Mongoose, and force-exits after a 10s timeout.

4. **Dockerize.** Write a multi-stage production `Dockerfile` (alpine, `npm ci --omit=dev` for runtime, non-root user, `node` as PID 1) and a `.dockerignore`. Add a `docker-compose.yml` that runs the app plus MongoDB with a persistent volume, the app reaching Mongo by service name.

5. **PM2 in cluster mode.** Write an `ecosystem.config.js` that runs the app in cluster mode across all cores with a `max_memory_restart`, separate dev/production env blocks, and log files. Explain in a comment what you must externalize before enabling cluster mode and how to do a zero-downtime deploy.

6. **CI with GitHub Actions.** Write `.github/workflows/ci.yml` that, on push and PR, installs with `npm ci`, runs lint and tests across Node 20 and 22, with a MongoDB service container the tests connect to.

### Solutions

#### Solution 1 — Validated config + secret hygiene

```js
// config/index.js
require('dotenv').config();
const { cleanEnv, str, port, url } = require('envalid'); // npm i envalid

const config = cleanEnv(process.env, {
  NODE_ENV: str({ choices: ['development', 'production', 'test'], default: 'development' }),
  PORT: port({ default: 3000 }),         // validated + coerced to a number
  MONGO_URI: url(),                       // must be a valid URL, required
  JWT_SECRET: str(),                      // required, no default
  LOG_LEVEL: str({ default: 'info' }),
});

module.exports = config; // import this everywhere; never read process.env elsewhere
```

```bash
# .gitignore
node_modules/
.env
*.log
```

```bash
# .env.example  (commit this)
NODE_ENV=development
PORT=3000
MONGO_URI=mongodb://127.0.0.1:27017/myapp_dev
JWT_SECRET=
LOG_LEVEL=debug
```

If `MONGO_URI` is missing or `PORT` is "abc", `envalid` prints a clear report and exits before `app.listen` — fail fast.

#### Solution 2 — Production logging

```js
// logger.js
const winston = require('winston');
const config = require('./config');
const { combine, timestamp, json, colorize, printf, errors } = winston.format;

const logger = winston.createLogger({
  level: config.LOG_LEVEL,
  format: config.NODE_ENV === 'production'
    ? combine(timestamp(), errors({ stack: true }), json())
    : combine(colorize(), timestamp({ format: 'HH:mm:ss' }), errors({ stack: true }),
        printf(({ level, message, timestamp, stack }) =>
          `${timestamp} ${level}: ${stack || message}`)),
  defaultMeta: { service: 'myapp' },
  transports: [new winston.transports.Console()],
});

module.exports = logger;
```

```js
// logging middleware wiring (in app.js)
const morgan = require('morgan');
const logger = require('./logger');
const config = require('./config');

if (config.NODE_ENV === 'production') {
  app.use(morgan('combined', {
    stream: { write: (msg) => logger.http(msg.trim()) }, // pipe into Winston
    skip: (req) => req.url === '/healthz',
  }));
} else {
  app.use(morgan('dev'));
}

// Everywhere else: logger.info(...), logger.warn(...), logger.error(...)
// instead of console.log(...).
```

#### Solution 3 — Graceful shutdown + health checks

```js
// routes/health.js
const router = require('express').Router();
const mongoose = require('mongoose');

router.get('/healthz', (req, res) => res.json({ status: 'ok', uptime: process.uptime() }));
router.get('/readyz', (req, res) => {
  return mongoose.connection.readyState === 1
    ? res.json({ status: 'ready' })
    : res.status(503).json({ status: 'not ready' });
});
module.exports = router;
```

```js
// server.js
require('dotenv').config();
const mongoose = require('mongoose');
const app = require('./app');
const config = require('./config');
const logger = require('./logger');

(async () => {
  await mongoose.connect(config.MONGO_URI);
  const server = app.listen(config.PORT, () => logger.info(`Up on ${config.PORT}`));

  async function shutdown(signal) {
    logger.info(`${signal} received — shutting down`);
    server.close(async () => {
      await mongoose.connection.close(false);
      logger.info('Closed cleanly');
      process.exit(0);
    });
    setTimeout(() => {
      logger.error('Shutdown timed out — forcing exit');
      process.exit(1);
    }, 10000).unref();
  }

  process.on('SIGTERM', () => shutdown('SIGTERM'));
  process.on('SIGINT', () => shutdown('SIGINT'));
})();
```

#### Solution 4 — Dockerize

```dockerfile
# Dockerfile
# syntax=docker/dockerfile:1
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .

FROM node:22-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force
COPY --from=build /app .
USER node
EXPOSE 3000
CMD ["node", "server.js"]   # node is PID 1 -> receives SIGTERM
```

```bash
# .dockerignore
node_modules
.git
.env
.env.*
logs
*.md
Dockerfile
.dockerignore
```

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports: ["3000:3000"]
    environment:
      NODE_ENV: production
      MONGO_URI: mongodb://mongo:27017/myapp   # service name, not localhost
      JWT_SECRET: ${JWT_SECRET}
    depends_on:
      mongo:
        condition: service_healthy
    restart: unless-stopped
  mongo:
    image: mongo:7
    volumes: ["mongo-data:/data/db"]           # persist data
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5
volumes:
  mongo-data:
```

#### Solution 5 — PM2 in cluster mode

```js
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'myapp',
    script: './server.js',
    instances: 'max',          // one per CPU core
    exec_mode: 'cluster',      // load-balanced, shares the port
    max_memory_restart: '500M',
    watch: false,
    env: { NODE_ENV: 'development', PORT: 3000 },
    env_production: { NODE_ENV: 'production', PORT: 8080 },
    error_file: './logs/pm2-error.log',
    out_file: './logs/pm2-out.log',
    merge_logs: true,
    kill_timeout: 5000,        // grace period for graceful shutdown
  }],
  // BEFORE enabling cluster mode, externalize all shared state:
  //  - sessions/rate-limit counters/caches -> Redis or MongoDB
  //  - scheduled (setInterval/cron) jobs -> a single dedicated process
  // Zero-downtime deploy: pm2 reload ecosystem.config.js --env production
  //  (reload restarts instances one-by-one, waiting for each to be ready;
  //   combined with graceful shutdown, no requests are dropped).
};
```

```bash
pm2 start ecosystem.config.js --env production
pm2 reload ecosystem.config.js --env production   # zero-downtime redeploy
pm2 startup && pm2 save                             # survive reboots
```

#### Solution 6 — CI with GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      mongo:
        image: mongo:7
        ports: ["27017:27017"]
    strategy:
      matrix:
        node-version: [20.x, 22.x]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      - run: npm ci
      - run: npm run lint --if-present
      - run: npm test
        env:
          MONGO_URI: mongodb://localhost:27017/test
          JWT_SECRET: test-secret
          NODE_ENV: test
```

---

## Summary / Cheat Sheet

| Topic | Key takeaway | Core command / snippet |
| --- | --- | --- |
| Config | Env vars only; validate once at startup; fail fast | `cleanEnv(process.env, {...})` |
| Secrets | Never commit `.env`; commit `.env.example`; rotate if leaked | `.gitignore` → `.env` |
| `NODE_ENV` | Convention many libs read; always set in prod; values are strings | `NODE_ENV=production node server.js` |
| Logging | Winston for app logs (leveled, JSON in prod); no `console.log` | `logger.info(msg, meta)` |
| HTTP logging | Morgan for requests; pipe into Winston in prod | `morgan('combined', { stream })` |
| Correlation | `requestId` via child loggers per request | `logger.child({ requestId })` |
| PM2 | Keeps app alive, cluster mode for cores, boot on restart | `pm2 start ecosystem.config.js --env production` |
| Reload vs restart | `reload` = zero-downtime (cluster); `restart` = brief downtime | `pm2 reload myapp` |
| Boot persistence | `startup` then `save` (both!) | `pm2 startup && pm2 save` |
| Graceful shutdown | On SIGTERM: stop new conns, drain, close DB, exit; timeout fallback | `process.on('SIGTERM', shutdown)` |
| Health checks | Liveness (cheap) vs readiness (checks deps) | `/healthz`, `/readyz` |
| Docker | Multi-stage, alpine, `npm ci --omit=dev`, `USER node`, node as PID 1 | `docker build -t app .` |
| `.dockerignore` | Exclude `node_modules`, `.env`, `.git` | keeps image small & safe |
| Compose | App + Mongo; reach by service name; named volume persists data | `docker compose up -d` |
| Reverse proxy | Nginx terminates TLS, serves static, load balances; set `trust proxy` | `app.set('trust proxy', 1)` |
| HTTPS | Free, auto-renewing certs via Certbot/Let's Encrypt (or managed) | `certbot --nginx -d example.com` |
| CI/CD | Test on every push/PR; build immutable image; deploy on green | `.github/workflows/*.yml` |
| Platforms | PaaS (Render/Railway/Fly) for speed; VPS for control/cost at scale | start managed, graduate deliberately |
| Observability | Logs + metrics + traces; Sentry for errors; uptime on `/healthz` | `Sentry.init({ dsn, environment, release })` |

You now have the full production lifecycle: configure safely, log observably, run resiliently, package portably, serve securely, ship automatically, and watch continuously. An app is not "done" when it returns the right JSON — it is done when it can be operated. This is what separates a developer who writes endpoints from one who ships and runs systems.
