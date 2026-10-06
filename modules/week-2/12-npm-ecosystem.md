# NPM & the Node Ecosystem

When you came from front-end JavaScript, you probably typed `npm install` a few times and watched a `node_modules` folder balloon to thousands of files. That command is the visible tip of an enormous iceberg. **npm** — the package manager that ships with Node.js — is the gateway to the largest software registry on Earth, and mastering it is the difference between a developer who *copies install commands from Stack Overflow* and one who can reason about reproducible builds, security audits, version ranges, monorepos, and publishing their own libraries.

This module treats npm as a serious engineering tool. We will start from "what even is a package manager and why does the server world need one" and end with you publishing a scoped package to the public registry, auditing dependencies for vulnerabilities, and explaining the difference between `npm install` and `npm ci` well enough to satisfy any interviewer.

## What You'll Learn

By the end of this module you will be able to:

- Explain **what npm is**, the relationship between the CLI, the **registry**, and your local project, and how `npm init` scaffolds a project.
- Read and write a **`package.json`** fluently — every field that matters: `name`, `version`, `description`, `main`, `type`, `scripts`, `dependencies`, `devDependencies`, `peerDependencies`, `optionalDependencies`, and `engines`.
- Master **Semantic Versioning (SemVer)** — `major.minor.patch`, the `^` (caret) and `~` (tilde) ranges, exact pins, and complex range syntax — and predict exactly which versions a range will accept.
- Install packages correctly: **local vs global**, `--save-dev`, `--save-exact`, and understand where each lands.
- Write and run **npm scripts**, chain them, pass arguments, and use **lifecycle hooks** (`pre`/`post`).
- Understand **`package-lock.json`** and the theory of **reproducible installs**, and choose correctly between **`npm install` and `npm ci`**.
- Use **`npx`** to run binaries without polluting your global namespace.
- Maintain a project with **`npm outdated`**, **`npm update`**, and **`npm audit` / `npm audit fix`**.
- **Publish your own package**, including scoped and private packages, version bumping, and dist-tags.
- Compare **npm vs Yarn vs pnpm** and know when each shines.
- Understand the **`node_modules` resolution algorithm**, hoisting, and concrete techniques to **avoid dependency bloat**.

---

## 1. What npm Is (and the WHY behind it)

### The problem package managers solve

On the front end you may have lived without a package manager for a long time — you dropped a `<script>` tag pointing at a CDN and moved on. Server-side development makes that approach untenable for three reasons:

1. **There is no `<script>` tag on a server.** A Node process is a long-running program. To use someone else's code you must have those files physically present on disk and load them with `require()`/`import`.
2. **Transitive dependencies explode.** The library you want depends on five others, each of which depends on more. Resolving that graph by hand is impossible.
3. **Reproducibility matters.** The code that runs in production must be byte-for-byte the same code you tested on your laptop. "It works on my machine" is a career-limiting phrase in backend engineering.

A **package manager** automates all of this: it downloads packages and their entire dependency tree, records exactly what was installed, and gives you a repeatable way to recreate that state anywhere.

### npm is three things

The single word "npm" is overloaded. Be precise, because interviewers test this:

| "npm" can mean | What it actually is |
| --- | --- |
| The **CLI** | The `npm` command-line tool bundled with every Node.js install. This is what runs `npm install`. |
| The **registry** | A massive public database/server at `https://registry.npmjs.org` storing published package tarballs and metadata. |
| The **company/website** | npm, Inc. (owned by GitHub/Microsoft) and the `npmjs.com` website where you browse packages. |

> TIP: "npm" historically stood for "Node Package Manager," but the project officially says it is **not** an acronym (the website jokes it stands for "npm is not an acronym"). Don't put much weight on the expansion in an interview — describe what it *does* instead.

### A "package" vs a "module" vs a "dependency"

- A **module** is any file or folder Node can load with `require()`/`import` — even a single `.js` file.
- A **package** is a folder described by a `package.json` file. Every package is loadable as a module, but a package is the *unit of distribution* on the registry.
- A **dependency** is a package your project relies on, recorded in your `package.json`.

### How the pieces fit together

```bash
# 1. The CLI talks to the registry over HTTPS to fetch metadata + tarballs.
npm install express

# 2. npm resolves express's entire dependency tree,
#    downloads every tarball, and extracts them into ./node_modules.

# 3. It records the top-level dependency in package.json
#    and the EXACT resolved tree in package-lock.json.
```

You can inspect the registry directly — it is just an HTTP API:

```bash
# Fetch the full metadata document for the 'express' package (all versions).
npm view express

# Show just the latest version number.
npm view express version

# Show the dependencies of the latest published express.
npm view express dependencies
```

> BEST PRACTICE: Get comfortable with `npm view <pkg>` (also spelled `npm info`). Before adding a dependency, check its latest version, its own dependency count, its license, and its last-publish date. A package last published 6 years ago with 40 dependencies is a maintenance risk.

---

## 2. `npm init` and the Anatomy of `package.json`

### Creating a project

Every Node project begins with a `package.json`. The `npm init` command scaffolds one interactively:

```bash
# Interactive: asks you questions (name, version, description, ...).
npm init

# Non-interactive: accept all defaults immediately. The flag is -y / --yes.
npm init -y

# Set defaults so future `npm init` runs pre-fill your info.
npm set init-author-name "Your Name"
npm set init-license "MIT"
```

A freshly generated `package.json` looks like this:

```json
{
  "name": "my-app",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```

> WARNING — Pitfall: `package.json` must be **valid JSON** — no comments, no trailing commas, double-quoted keys. The code blocks in this module sometimes show `// comments` for teaching, but you must remove them from a real `package.json` or your tooling will throw `JSONError`.

### The fields that matter

Below is an exhaustive, annotated `package.json`. Read every comment — each field is a topic interviewers probe.

```jsonc
{
  // REQUIRED for publishing. Lowercase, URL-safe, <= 214 chars, no spaces.
  // Can be "scoped": "@myorg/my-app".
  "name": "my-app",

  // REQUIRED for publishing. Must be valid SemVer (see section 3).
  "version": "1.0.0",

  // Free-text summary shown in registry search results. Helps discoverability.
  "description": "A demo backend service.",

  // The entry point: what `require('my-app')` resolves to when your package
  // is imported by ANOTHER project. Defaults to index.js. Irrelevant for apps
  // that are never imported, but critical for libraries.
  "main": "src/index.js",

  // Module system for *your own* .js files:
  //   "commonjs" (default if omitted) -> require/module.exports, files are CJS
  //   "module"                        -> import/export, .js files are ES Modules
  "type": "commonjs",

  // Named shortcuts you run with `npm run <name>`. See section 5.
  "scripts": {
    "start": "node src/index.js",
    "dev": "node --watch src/index.js",
    "test": "node --test"
  },

  // Runtime dependencies: needed for the app to RUN in production.
  "dependencies": {
    "express": "^4.19.2",
    "mongoose": "^8.4.0"
  },

  // Build/test/lint tools: needed only during DEVELOPMENT, never shipped to prod.
  "devDependencies": {
    "nodemon": "^3.1.0",
    "eslint": "^9.4.0"
  },

  // "I need the host project to provide this, but I won't bundle my own copy."
  // Classic for plugins (e.g., an Express middleware that needs Express present).
  "peerDependencies": {
    "express": ">=4.0.0"
  },

  // Nice-to-have deps. If install FAILS, npm continues anyway instead of erroring.
  // Your code MUST handle the package being absent at runtime.
  "optionalDependencies": {
    "fsevents": "^2.3.3"
  },

  // Declares which Node/npm versions your package supports. Acts as a guardrail.
  "engines": {
    "node": ">=20.0.0",
    "npm": ">=10.0.0"
  },

  // Searchable tags on the registry.
  "keywords": ["backend", "api", "demo"],

  // Author and license metadata.
  "author": "Your Name <you@example.com>",
  "license": "MIT",

  // If true, this package can NEVER be accidentally published. Use for apps.
  "private": true
}
```

### `dependencies` vs `devDependencies` vs `peerDependencies` vs `optionalDependencies`

This four-way distinction is one of the most common interview questions. Internalize the table:

| Field | Installed when… | Purpose | Example |
| --- | --- | --- | --- |
| `dependencies` | always (incl. `npm install` in prod) | Code the app needs to **run**. | `express`, `mongoose` |
| `devDependencies` | only in dev (`npm install` without `--production`) | Tooling for build/test/lint. | `eslint`, `jest`, `nodemon` |
| `peerDependencies` | **not auto-installed** (npm 7+ does try, see note) | "Host must provide this." Avoids duplicate copies; for plugins. | a React component lib peer-depending on `react` |
| `optionalDependencies` | always attempted; **failure ignored** | Nice-to-have native modules that may not build everywhere. | `fsevents` (macOS-only file watcher) |

> WARNING — Pitfall: Since **npm 7**, peer dependencies are auto-installed by default (this changed from npm 6, which only *warned*). If a peer dependency conflicts with another package's requirement, npm 7+ will **error out** with `ERESOLVE`. The escape hatches are `--legacy-peer-deps` (behave like npm 6 and ignore peers) or `--force` (install anyway, accepting a possibly broken tree). Reach for these only when you understand *why* the conflict exists.

> BEST PRACTICE: Put a package in `devDependencies` unless your **shipped, running** code imports it. A bloated `dependencies` list means slower production installs and a larger attack surface. A quick test: "If I delete this in production, does the app still start and serve requests?" If yes → `devDependencies`.

The `--production` install (or `NODE_ENV=production`) skips `devDependencies`:

```bash
# Install ONLY dependencies, not devDependencies. Great for prod containers.
npm install --omit=dev          # modern syntax (npm 8+)
npm install --production         # older alias, still works
```

---

## 3. Semantic Versioning (SemVer)

Versioning is where most developers have fuzzy, superstitious knowledge. Let's make it precise.

### The `MAJOR.MINOR.PATCH` contract

Every published package version is three numbers separated by dots:

```
   2   .   8   .   1
   │       │       │
 MAJOR   MINOR   PATCH
```

The numbers are a **promise** from the maintainer to you:

| Segment | Bumped when… | Backward compatible? | Example transition |
| --- | --- | --- | --- |
| **PATCH** | A bug is fixed, no API change. | Yes | `2.8.0 → 2.8.1` |
| **MINOR** | New functionality added, nothing removed/broken. | Yes | `2.8.1 → 2.9.0` |
| **MAJOR** | A breaking change (removed/changed behavior). | **No** | `2.9.0 → 3.0.0` |

When MINOR bumps, PATCH resets to 0. When MAJOR bumps, both MINOR and PATCH reset to 0.

> TIP: SemVer is a **social contract**, not a technical guarantee. A careless maintainer can ship a breaking change in a PATCH release. This is precisely *why* lockfiles exist (section 6) — they protect you from misbehaving version ranges.

### Pre-release and `0.x` versions

- **Pre-release tags** look like `2.0.0-beta.1` or `1.5.0-rc.2`. They sort *before* the final release (`2.0.0-beta.1 < 2.0.0`) and are **not** matched by normal ranges unless you opt in.
- **`0.x.y` versions** are special: the SemVer spec says anything below `1.0.0` is "unstable," and tooling treats MINOR bumps in the `0.x` range as potentially breaking. This affects how `^` behaves (see below).

### Version ranges: `^`, `~`, exact, and more

In `package.json` you rarely pin one exact version. Instead you express a **range** — the set of versions you're willing to accept. This is the most exam-worthy part of the whole module.

#### The caret `^` — "compatible with" (the npm default)

`^` allows changes that do **not** modify the **left-most non-zero** version segment. In practice, for a normal `1.x+` package it means "any version up to (but not including) the next MAJOR."

```jsonc
"^1.2.3"   // >=1.2.3  <2.0.0   → accepts 1.2.4, 1.9.0, but NOT 2.0.0
"^0.2.3"   // >=0.2.3  <0.3.0   → left-most non-zero is MINOR, so locked there
"^0.0.3"   // >=0.0.3  <0.0.4   → effectively an exact pin (only PATCH frozen)
```

> WARNING — Pitfall: `^` behaves **differently for `0.x` packages**. People assume `^0.2.3` allows `0.9.0`. It does **not** — it stops at `<0.3.0`. The rule is "don't change the left-most non-zero digit," and for `0.2.3` that digit is the `2`. This trips up nearly everyone.

#### The tilde `~` — "approximately this minor"

`~` allows PATCH-level changes if a MINOR is specified; it allows MINOR-level changes if only MAJOR is specified.

```jsonc
"~1.2.3"   // >=1.2.3  <1.3.0   → only PATCH updates (1.2.x)
"~1.2"     // >=1.2.0  <1.3.0   → same as above
"~1"       // >=1.0.0  <2.0.0   → MINOR + PATCH updates (1.x)
```

#### Exact pins and explicit ranges

```jsonc
"1.2.3"        // EXACTLY 1.2.3, nothing else.
"=1.2.3"       // Same thing, explicit equals.
">=1.2.3"      // 1.2.3 or anything higher.
">1.2.3 <2.0"  // A compound range (space = AND).
"1.2.x"        // Any patch in 1.2 (equivalent to ~1.2.0).
"1.x" / "1.*"  // Any 1.y.z.
"*" / ""       // Literally any version (dangerous).
"1.2.3 || 1.5.0" // OR: either of these specific versions.
"latest"       // A dist-tag, not a range — resolves to whatever is tagged latest.
```

#### Caret vs Tilde at a glance

| Range | Expands to | Updates allowed | Use when… |
| --- | --- | --- | --- |
| `^1.2.3` | `>=1.2.3 <2.0.0` | MINOR + PATCH | You trust the maintainer's SemVer (default). |
| `~1.2.3` | `>=1.2.3 <1.3.0` | PATCH only | You want bug fixes but no new features. |
| `1.2.3` | `=1.2.3` | none | Maximum caution / known-fragile dependency. |

> BEST PRACTICE: Leave the npm default of `^` for most dependencies and rely on the **lockfile** for reproducibility (section 6). The lockfile pins exact versions for installs; the range in `package.json` just defines what `npm update` is *allowed* to move to. Pinning everything exactly in `package.json` is unnecessary belt-and-suspenders that also makes security patches harder to receive.

You can verify your mental model with the bundled `semver` tooling:

```bash
# Does version 1.9.0 satisfy the range ^1.2.3? (prints the version if yes)
npx semver 1.9.0 --range "^1.2.3"     # prints 1.9.0
npx semver 2.0.0 --range "^1.2.3"     # prints nothing (does NOT satisfy)
npx semver 0.9.0 --range "^0.2.3"     # prints nothing (0.x caret rule!)
```

---

## 4. Installing Packages: Local, Global, and the Save Flags

### Local installs (the default and the right default)

```bash
# Adds to "dependencies" and installs into ./node_modules.
npm install express
npm i express          # `i` is the alias for install

# Install a specific version or range.
npm install express@4.19.2
npm install express@latest
npm install "express@>=4 <5"

# Install several at once.
npm install express mongoose dotenv
```

A **local** install places code in your project's `node_modules` and records it in *that* project's `package.json`. This is almost always what you want, because each project gets its own isolated, version-locked copy of its dependencies. Two projects on the same machine can use Express 4 and Express 5 with zero conflict.

### The save flags

| Command | Lands in | Meaning |
| --- | --- | --- |
| `npm i lodash` | `dependencies` | Default since npm 5 — saving is automatic. |
| `npm i -D eslint` | `devDependencies` | `-D` = `--save-dev`. |
| `npm i -E react` | `dependencies` (exact) | `-E` = `--save-exact`, writes `1.2.3` not `^1.2.3`. |
| `npm i -O fsevents` | `optionalDependencies` | `-O` = `--save-optional`. |
| `npm i --no-save foo` | nowhere | Installs but does NOT touch `package.json`. |
| `npm i -P axios` | `dependencies` | `-P` = `--save-prod` (explicit prod). |

```bash
# Dev tool — needed only while building/testing.
npm install --save-dev nodemon eslint jest

# Pin an exact version (no ^). Useful for known-fragile or app-critical deps.
npm install --save-exact react@18.3.1

# Combine: save-dev AND exact.
npm install --save-dev --save-exact prettier@3.3.2
```

> TIP: `--save` is the historical flag to write into `dependencies`. Since **npm 5** it is the *default*, so you almost never type `--save` anymore. You still type `--save-dev` and `--save-exact` because those override the default behavior.

### Global installs (use sparingly)

```bash
# Installs to a system-wide location, NOT into your project.
npm install -g typescript
npm install --global serve

# See where globals live and what's installed.
npm root -g          # the global node_modules path
npm ls -g --depth=0  # list top-level global packages
```

Global installs put a package (usually a CLI tool) on your system `PATH` so you can run it from anywhere. The temptation is to globally install everything; resist it.

> WARNING — Pitfall: Global installs are **not reproducible** and **not version-locked per project**. If your build relies on a globally installed tool at version X, a teammate (or your CI server) with version Y gets different results — and nothing in your repo documents the requirement. Prefer a **local devDependency + npm script**, or **`npx`** (section 7), over global installs for anything a project depends on. Reserve globals for personal, project-agnostic utilities (e.g., `serve`, `http-server`).

---

## 5. npm Scripts and Lifecycle Hooks

### Why scripts exist

A Node project accumulates a dozen incantations: how to start it, test it, lint it, build it, seed the database. Rather than documenting these in a README and hoping people read it, you encode them as **npm scripts** — named commands in `package.json` that anyone can run with `npm run <name>`. This makes the project self-documenting and consistent across machines.

```json
{
  "scripts": {
    "start": "node src/index.js",
    "dev": "node --watch src/index.js",
    "test": "node --test",
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "build": "node scripts/build.js",
    "seed": "node scripts/seed-db.js"
  }
}
```

```bash
npm run dev        # runs "node --watch src/index.js"
npm run lint
npm run lint:fix
```

### The special scripts you can run without `run`

A handful of script names are "blessed" and have shorthand invocations:

```bash
npm start          # shorthand for: npm run start
npm test           # shorthand for: npm run test  (also `npm t`)
npm stop
npm restart
```

Every *other* script requires the explicit `npm run`. So `npm run dev`, not `npm dev`.

### The magic of `node_modules/.bin`

Here is the single most useful thing to understand about scripts. When you install a package that ships a CLI (a "binary"), npm drops a symlink into `node_modules/.bin/`. **Inside an npm script, that folder is automatically on the `PATH`.**

```json
{
  "scripts": {
    // You can write `eslint` directly — npm finds ./node_modules/.bin/eslint.
    // No global install of eslint needed!
    "lint": "eslint .",
    "test": "jest --coverage"
  }
}
```

This is why you can `npm install --save-dev eslint` (local!) and then run it from a script without a global install. The local binary always wins, guaranteeing everyone uses the *project's* version.

### Passing arguments to scripts

Use `--` to forward arguments through `npm run` to the underlying command:

```bash
# Everything after `--` is passed to the script's command.
npm run test -- --watch          # runs: node --test --watch (the --watch is forwarded)
npm run lint -- --max-warnings=0
```

### Chaining scripts

```json
{
  "scripts": {
    "clean": "node scripts/clean.js",
    "compile": "node scripts/compile.js",

    // && = run sequentially, stop on first failure (cross-platform via the shell).
    "build": "npm run clean && npm run compile",

    // Reference another script with `npm run`.
    "ci": "npm run lint && npm run test && npm run build"
  }
}
```

> TIP: For running scripts **in parallel** or with prettier output, teams often add a tiny dev dependency like `npm-run-all` (`run-s` for serial, `run-p` for parallel) or `concurrently`. Plain `&&` is serial and fully cross-platform; `&` for background/parallel is **not** reliable on Windows.

### Lifecycle hooks: `pre` and `post`

npm has a powerful, sometimes surprising convention: for **any** script named `foo`, npm will automatically run `prefoo` *before* it and `postfoo` *after* it — if those scripts exist.

```json
{
  "scripts": {
    "prebuild": "npm run clean",   // runs automatically BEFORE build
    "build": "node scripts/build.js",
    "postbuild": "node scripts/report-size.js", // runs automatically AFTER build

    "pretest": "npm run lint",     // lint is enforced before every test run
    "test": "node --test"
  }
}
```

```bash
npm run build
# npm executes, in order:
#   1. prebuild   (npm run clean)
#   2. build      (node scripts/build.js)
#   3. postbuild  (node scripts/report-size.js)
# If prebuild fails (non-zero exit), build and postbuild never run.
```

This works for your custom scripts *and* for built-in lifecycle events npm fires during install/publish.

#### Install & publish lifecycle hooks

npm fires specific scripts automatically at certain moments. The most important ones:

| Hook | Fires when… |
| --- | --- |
| `preinstall` / `postinstall` | around `npm install` |
| `prepare` | after install (incl. local), and **before** `npm publish`/`pack` |
| `prepublishOnly` | only before `npm publish` (not on local installs) |
| `prepack` / `postpack` | around tarball creation |

```json
{
  "scripts": {
    // Runs after `npm install` AND before publishing. Common for building TS, etc.
    "prepare": "npm run build",

    // A last-chance guard before publishing: run tests, refuse to ship if they fail.
    "prepublishOnly": "npm test && npm run lint"
  }
}
```

> WARNING — Pitfall: **`postinstall` scripts are a real-world security and supply-chain risk.** A malicious package can run arbitrary code on your machine via a `postinstall` hook the moment you `npm install` it. For CI and untrusted dependencies you can disable them with `npm install --ignore-scripts`, and you can set `ignore-scripts=true` in `.npmrc` globally. The trade-off: some legitimate native modules need their install scripts to compile. Know this tension exists — interviewers love the "how do you defend against supply-chain attacks" question.

> WARNING — Pitfall: There used to be a `prepublish` hook that ran on **both** publish *and* local install — a famous source of confusion. It is deprecated. Use **`prepare`** (runs on local install + publish) and **`prepublishOnly`** (publish only) instead.

---

## 6. `package-lock.json` and Reproducible Installs

### The problem the lockfile solves

Your `package.json` says `"express": "^4.19.2"`. That is a *range*, not a version. If you run `npm install` today you might get `4.19.2`; if a teammate runs it next week after Express ships `4.20.0`, they get `4.20.0`. Now you are running *different code* — and the bug that only reproduces "on their machine" is born. Worse, transitive dependencies (deps of your deps) are also ranges, so the entire tree can drift.

The **`package-lock.json`** file solves this. It is an auto-generated, machine-written record of the **exact** version, resolved URL, and integrity hash of *every* package in the tree — direct and transitive. With a lockfile committed, everyone resolves to the identical tree.

```jsonc
// Excerpt of a package-lock.json (you do NOT edit this by hand).
{
  "name": "my-app",
  "lockfileVersion": 3,        // v3 since npm 7; flat "packages" map
  "requires": true,
  "packages": {
    "node_modules/express": {
      "version": "4.19.2",                                   // the EXACT version locked
      "resolved": "https://registry.npmjs.org/express/-/express-4.19.2.tgz",
      "integrity": "sha512-...==",                           // SHA-512 hash for tamper detection
      "dependencies": { "accepts": "~1.3.8", "...": "..." }
    }
  }
}
```

> BEST PRACTICE: **Commit `package-lock.json` to version control.** It is not a build artifact to ignore — it is the source of truth for reproducible installs. The only time you might omit it is when authoring a *library* (a published package) where you want consumers to resolve their own trees; even then, committing it helps your own CI.

### `lockfileVersion` quick history

- **v1** — npm 5/6 format.
- **v2** — npm 7, backward compatible with v6.
- **v3** — npm 7+, drops the v1 compatibility data; smaller. This is current.

### How the lockfile and `package.json` interact

- `package.json` = the **intent** (which packages, which acceptable ranges).
- `package-lock.json` = the **resolved reality** (exact versions that satisfied those ranges last time).
- `npm install` reads both. It respects the lockfile **unless** the `package.json` ranges no longer permit the locked versions, in which case it re-resolves and **rewrites the lockfile**.

### `npm install` vs `npm ci` — the headline interview question

| Aspect | `npm install` | `npm ci` |
| --- | --- | --- |
| Reads `package.json` | Yes | Yes (only to validate) |
| Reads `package-lock.json` | Yes (and may **update** it) | Yes (and **never** modifies it) |
| If lock is missing | creates one | **errors out** |
| If lock & package.json disagree | reconciles, rewrites lock | **errors out** |
| Existing `node_modules` | updates in place | **deletes it entirely** first, then installs clean |
| Speed | slower (resolution work) | faster (no resolution; straight from lock) |
| Can add a new package | yes (`npm i x`) | no (install-from-lock only) |
| Intended for | local development | CI/CD and production builds |

```bash
# Local dev: add/upgrade things, lockfile evolves with you.
npm install
npm install some-new-dep

# CI / production: deterministic, clean, fast. Fails loudly if lock is stale.
npm ci
```

> BEST PRACTICE: Use **`npm ci`** in every CI pipeline and Docker production build. Because it (a) refuses to run with a missing/mismatched lockfile and (b) wipes `node_modules` for a clean slate, it guarantees the build matches exactly what you committed. `npm install` is for the developer's inner loop, where you *want* the lockfile to change as you add dependencies.

> WARNING — Pitfall: If `npm ci` fails in CI with "package.json and package-lock.json are out of sync," the fix is **not** to switch to `npm install` in CI — that hides the problem. The fix is to run `npm install` *locally*, commit the updated `package-lock.json`, and push. The mismatch means someone changed `package.json` without regenerating the lock.

---

## 7. `npx` — Running Binaries Without Installing

`npx` (bundled with npm since npm 5.2) runs a package's executable **without permanently installing it**. It is the antidote to "I need this CLI tool just once."

### How it resolves a command

When you run `npx <command>`, npx looks in this order:

1. A local `node_modules/.bin/<command>` (a project dependency).
2. The npx cache.
3. If still not found, it **downloads the package temporarily**, runs it, and (in modern npx) does not pollute your globals.

```bash
# Scaffold a new app without installing create-* tools globally.
npx create-react-app my-app
npx express-generator my-api

# Run a one-off tool you don't want forever.
npx cowsay "deploy succeeded"

# Run a SPECIFIC version of a tool, ignoring whatever you have installed.
npx eslint@9.4.0 .

# Run a binary from a package whose name differs from the command:
npx --package=typescript tsc --version
```

> TIP: Modern npx (npm 7+) **prompts before downloading** a package you don't have, e.g. `Need to install the following packages: cowsay@1.6.0  Ok to proceed? (y)`. Add `-y` to skip the prompt in scripts/CI: `npx -y cowsay hi`.

### `npx` vs global install vs local + script

| You want to… | Best tool |
| --- | --- |
| Run a project's own tool (eslint, jest) | local devDependency + npm script |
| Try/scaffold a tool once | `npx` |
| A personal CLI you use across many projects daily | global install |
| Pin a tool version for a one-off command | `npx tool@version` |

> BEST PRACTICE: For scaffolding commands (`create-*`), always use `npx` rather than globally installing the generator. The generator is run once and then irrelevant; a global install just goes stale.

---

## 8. Maintenance: `outdated`, `update`, and `audit`

A real project is never "done" with dependencies — they age, get bug fixes, and develop vulnerabilities. These three commands are your maintenance toolkit.

### `npm outdated` — what's behind?

```bash
npm outdated
```

```text
Package    Current  Wanted  Latest  Location              Depended by
express    4.19.2   4.19.2  5.1.0   node_modules/express  my-app
lodash     4.17.20  4.17.21 4.17.21 node_modules/lodash   my-app
```

Read the three version columns carefully — this is exam material:

| Column | Meaning |
| --- | --- |
| **Current** | What's actually installed right now. |
| **Wanted** | The newest version allowed by your `package.json` **range** (what `npm update` would move to). |
| **Latest** | The newest version on the registry, ignoring your range (the `latest` dist-tag). |

In the example: `lodash` can be safely bumped (Wanted > Current, both within range). `express` has a new MAJOR (`5.1.0`) that your `^4` range refuses — upgrading requires a deliberate, possibly breaking, change.

### `npm update` — apply in-range upgrades

```bash
# Upgrade all packages to the "Wanted" version (within package.json ranges),
# and rewrite package-lock.json accordingly.
npm update

# Update a single package.
npm update lodash
```

`npm update` will **never** cross a range boundary. To jump to a new MAJOR you must explicitly install it (or edit the range):

```bash
# Deliberately move to Express 5 — this changes package.json's range.
npm install express@5
# Then read the migration guide and fix breakages!
```

> BEST PRACTICE: For MAJOR upgrades, change one package at a time, run your full test suite, and read the changelog/migration guide. Batch MAJOR bumps are how you spend a Friday debugging. For keeping deps fresh systematically, teams add bots like **Dependabot** or **Renovate** that open one PR per upgrade with CI attached.

### `npm audit` — find known vulnerabilities

`npm audit` cross-references your dependency tree against a database of known security advisories and reports anything affected.

```bash
# Report vulnerabilities with severity and remediation advice.
npm audit

# Machine-readable output for tooling/CI gates.
npm audit --json

# Limit the noise: only fail/report on high+ severity.
npm audit --audit-level=high
```

```text
# npm audit report

minimist  <1.2.6
Severity: critical
Prototype Pollution - https://github.com/advisories/GHSA-xvch-5gv4-984h
fix available via `npm audit fix`
```

### `npm audit fix` — auto-remediate

```bash
# Apply fixes that stay WITHIN your SemVer ranges (safe, non-breaking).
npm audit fix

# Apply fixes even if they require a breaking MAJOR bump. Use with care + tests.
npm audit fix --force

# See what WOULD change without doing it.
npm audit fix --dry-run
```

> WARNING — Pitfall: `npm audit fix --force` can and will install **breaking major versions** to escape a vulnerability. Never run it blindly on a production project — review the diff, then run your tests. Also be aware that `npm audit` can be noisy: many advisories live in **devDependencies** (e.g., a build tool) and are not exploitable in your shipped runtime. Use `--omit=dev` semantics and judgment; a "critical" in a CLI dev tool is not the same as one in `express`.

> TIP: Sometimes the vulnerable package is a **transitive** dependency you can't directly upgrade because the parent pins an old range. The modern fix is the **`overrides`** field in `package.json`, which forces a nested dependency to a patched version:
>
> ```json
> {
>   "overrides": {
>     "minimist": "^1.2.8"
>   }
> }
> ```
> (Yarn calls this `resolutions`; pnpm uses `pnpm.overrides`.)

---

## 9. Publishing Your Own Package

Publishing is where npm stops being a consumer tool and becomes a distribution channel. Even if you never publish publicly, understanding the flow demystifies how everything you install got there.

### Step 1 — Account and login

```bash
# Create an account at npmjs.com first, then authenticate the CLI.
npm login           # opens a browser flow (or prompts) and stores a token
npm whoami          # verify you're logged in
```

> BEST PRACTICE: Enable **two-factor authentication (2FA)** on your npm account and require it for publishes (`npm profile enable-2fa auth-and-writes`). In automated pipelines, use a **granular automation access token** stored as a CI secret rather than your password. This is a direct supply-chain defense — a compromised maintainer account is how major attacks happen.

### Step 2 — Prepare `package.json`

A publishable package needs, at minimum, a unique `name`, a valid `version`, and a `main` entry point:

```json
{
  "name": "@your-scope/string-utils",
  "version": "1.0.0",
  "description": "Tiny, dependency-free string helpers.",
  "main": "src/index.js",
  "type": "commonjs",
  "license": "MIT",
  "files": ["src", "README.md"],
  "engines": { "node": ">=20" },
  "repository": { "type": "git", "url": "https://github.com/you/string-utils.git" }
}
```

### Step 3 — Control what gets published

By default npm publishes everything except files matched by `.npmignore`/`.gitignore` and a few always-ignored files. The cleaner, allow-list approach is the **`files`** field — list exactly what to include.

```bash
# DRY RUN: see precisely which files would be in the tarball — do this every time!
npm pack --dry-run

# Actually create the tarball locally (without publishing) to inspect it.
npm pack
```

> WARNING — Pitfall: It is shockingly common to accidentally publish `.env` files, `node_modules`, test fixtures, or secrets. Always run `npm pack --dry-run` and read the file list before publishing. Prefer an allow-list (`"files": [...]`) over a deny-list (`.npmignore`) — allow-lists fail safe.

### Step 4 — Scoped vs unscoped packages

| Type | Name format | Default visibility | Use for |
| --- | --- | --- | --- |
| Unscoped | `string-utils` | public | Classic public packages (name must be globally unique & available). |
| Scoped | `@your-scope/string-utils` | **private** (paid) unless told otherwise | Org/personal namespacing; avoids name collisions. |

```bash
# Publish an UNSCOPED package (name must be free on the registry).
npm publish

# Publish a SCOPED package PUBLICLY (scoped default is private, so be explicit).
npm publish --access public
```

> TIP: Scopes (`@yourname/...`) are the easy way around "that package name is taken." `@your-username/string-utils` is yours as long as the scope is. Publishing scoped packages *publicly* is free; *private* scoped packages require a paid plan or a private registry.

### Step 5 — Versioning with `npm version`

Never hand-edit the `version` field. The `npm version` command bumps it correctly, commits the change, and creates a git tag:

```bash
npm version patch    # 1.0.0 -> 1.0.1  (bug fix)
npm version minor    # 1.0.1 -> 1.1.0  (new feature, compatible)
npm version major    # 1.1.0 -> 2.0.0  (breaking change)

# Pre-releases:
npm version prerelease --preid=beta   # 2.0.0 -> 2.0.1-beta.0
```

Each of these updates `package.json`, makes a git commit, and tags it (e.g. `v1.0.1`) so your release history is traceable.

### Step 6 — Publish, then verify

```bash
npm publish --access public
npm view @your-scope/string-utils    # confirm it's live on the registry
```

### dist-tags: beyond `latest`

When you `npm install foo`, you get `foo@latest` — `latest` is a **dist-tag** (a named pointer to a version). You can publish to other tags so users can opt into pre-release channels:

```bash
# Publish a beta WITHOUT moving the `latest` tag that normal users get.
npm publish --tag beta

# Users opt in explicitly:
npm install @your-scope/string-utils@beta

# Re-point tags later:
npm dist-tag add @your-scope/string-utils@2.0.0 latest
npm dist-tag ls  @your-scope/string-utils
```

### Unpublishing and deprecating

```bash
# Politely warn users a version (or whole package) shouldn't be used anymore.
npm deprecate @your-scope/string-utils@"<1.0.0" "Please upgrade to v1+."

# Unpublishing is HEAVILY restricted (24h window, conditions) to protect the
# ecosystem from the famous left-pad incident. Prefer deprecate over unpublish.
npm unpublish @your-scope/string-utils@1.0.0
```

> WARNING — Pitfall: You cannot republish the **same version number** after publishing it, even if you unpublish — the version is burned. Bump the version for every change. This immutability is a feature: it guarantees `string-utils@1.0.0` is forever the same bytes for everyone.

---

## 10. Alternatives: Yarn and pnpm

npm is the default, but two alternatives are common enough that you must be conversant. All three read the same `package.json` and install from the same npm registry — they differ in **lockfile format**, **install strategy**, and **`node_modules` layout**.

| Feature | npm | Yarn (Classic v1 / Berry v2+) | pnpm |
| --- | --- | --- | --- |
| Lockfile | `package-lock.json` | `yarn.lock` | `pnpm-lock.yaml` |
| Install (deterministic) | `npm ci` | `yarn install --immutable` | `pnpm install --frozen-lockfile` |
| `node_modules` layout | flat, hoisted | flat, hoisted (v1); PnP (Berry) | **symlinked + global content-addressable store** |
| Disk usage | high (copies per project) | high | **low** (hard-links one global copy) |
| Speed | fast | fast | **fastest** (esp. cold cache) |
| Monorepo workspaces | yes (`workspaces`) | yes (mature) | yes (excellent) |
| Strictness | loose (phantom deps possible) | loose | **strict** (no phantom/undeclared deps) |

### The key conceptual differences

- **npm / Yarn Classic** install a **flat, hoisted** `node_modules`: most packages are deduplicated up to the top level. Downside: you can accidentally `require()` a package you never declared (a **phantom dependency**) because it happens to be hoisted into view — and that breaks when the tree changes.
- **pnpm** stores every package version **once** in a global content-addressable store (`~/.pnpm-store`) and links it into each project with symlinks/hard-links. Your project's `node_modules` only exposes packages you actually declared, eliminating phantom deps. It also saves enormous disk space across many projects.
- **Yarn Berry (v2+)** introduced **Plug'n'Play (PnP)**, which can do away with `node_modules` entirely, resolving modules from a single map file. Powerful but requires editor/tooling buy-in.

```bash
# Same logical operation, three CLIs:
npm install express
yarn add express
pnpm add express

# Dev dependency:
npm i -D eslint
yarn add -D eslint
pnpm add -D eslint

# Reproducible CI install:
npm ci
yarn install --immutable
pnpm install --frozen-lockfile
```

> BEST PRACTICE: **Pick one package manager per repository and commit only its lockfile.** Mixing managers (a `package-lock.json` and a `yarn.lock` in the same repo) leads to divergent trees and confusion. For most people the right default is **npm** (zero install, ubiquitous). Choose **pnpm** when disk space, strictness, or large monorepos matter; choose **Yarn** if your team already standardized on it.

---

## 11. `node_modules` Resolution and Avoiding Dependency Bloat

### How `require()` finds a package

Understanding the resolution algorithm explains why `node_modules` looks the way it does and why bugs like "cannot find module" happen. When you `require('lodash')` (a *bare specifier*, not a path), Node:

1. Looks for `lodash` in the `node_modules` folder of the **current** directory.
2. If not found, walks **up** to the parent directory's `node_modules`.
3. Keeps walking up, directory by directory, to the filesystem root.
4. If still not found after exhausting all ancestor `node_modules`, throws `Error: Cannot find module 'lodash'`.

```text
/project
├── node_modules
│   ├── express            <- found here for require('express') from /project/src
│   └── lodash
└── src
    └── index.js           <- require('lodash') walks: src/node_modules? no
                              -> /project/node_modules? YES
```

Once it finds the package folder, Node reads that package's `package.json` `main` (or `exports`) field to know which file to actually load.

### Hoisting and the flat tree

In the old npm 2 days, `node_modules` was **nested**: every dependency had its own `node_modules` inside it, leading to absurd depth (the infamous Windows "path too long" errors). Since **npm 3**, the tree is **flattened/hoisted**: shared dependencies are pulled up to the top level and deduplicated when versions are compatible.

```text
# Modern flat layout (npm 3+):
node_modules/
├── express/
├── accepts/          <- hoisted up, shared by multiple packages
├── body-parser/
└── debug/            <- one copy reused everywhere it's compatible
```

When two packages need *incompatible* versions of the same dependency, npm keeps the conflicting one nested inside the package that needs it. You can inspect the resulting tree:

```bash
npm ls                 # full dependency tree
npm ls --depth=0       # only your direct dependencies
npm ls express         # show where/which versions of express are installed
npm explain lodash     # WHY is lodash here? Trace what depends on it
npm dedupe             # attempt to flatten/deduplicate further
```

> WARNING — Pitfall: Hoisting enables **phantom dependencies** — your code may `require('some-transitive-pkg')` successfully because it got hoisted to the top level, even though you never declared it in `package.json`. Then a future install changes the tree, the package is no longer hoisted, and your app crashes in production. The rule: **only `require()` packages you explicitly list in your own `package.json`.** (pnpm enforces this for you; npm trusts your discipline.)

### Avoiding dependency bloat

Every dependency is code you didn't write, can't fully audit, must update, and that widens your attack surface. Disciplined dependency management is a hallmark of a senior backend engineer.

**1. Question each addition.** Do you need a 50KB library for a 3-line task? The classic example is `left-pad` — a one-function package. Modern JS has `String.prototype.padStart`. Prefer the platform.

**2. Audit weight before installing.**

```bash
npm view some-package dependencies        # how many deps does IT drag in?
npx howfat some-package                    # total install size + dep count
# Or check bundlephobia.com / packagephobia.com in a browser.
```

**3. Keep dev tools in `devDependencies`** so production installs (`npm ci --omit=dev`) stay lean.

**4. Prune and dedupe regularly.**

```bash
npm prune              # remove packages not listed in package.json
npm prune --omit=dev   # also strip devDependencies (prod cleanup)
npm dedupe             # collapse duplicate versions where ranges allow
```

**5. Watch transitive depth.** A package with 200 transitive dependencies is 200 maintainers who could be compromised. Favor small, focused, well-maintained packages with few dependencies.

**6. Use `.npmrc` to enforce policy** across a team:

```ini
# .npmrc (committed to the repo)
save-exact=true          # always pin exact versions on install
engine-strict=true       # FAIL install if Node version violates "engines"
audit-level=high         # default audit threshold
```

> BEST PRACTICE: Treat adding a dependency like a code review decision, not a reflex. Check its last publish date, weekly downloads, open issues, transitive dependency count, and license. The cheapest dependency is the one you didn't add — the standard library and a few lines of your own code are often the right answer.

---

## Common Pitfalls & Gotchas

- **Treating `^0.x` like `^1.x`.** `^0.2.3` resolves to `>=0.2.3 <0.3.0`, **not** `<1.0.0`. For `0.x` packages, caret locks the MINOR. Verify with `npx semver`.
- **Committing `node_modules`.** Never. Commit `package.json` and `package-lock.json`; `.gitignore` the `node_modules` folder.
- **Forgetting to commit `package-lock.json`.** This silently destroys reproducibility — everyone resolves a different tree. Commit it for applications.
- **Using `npm install` in CI.** It can rewrite the lockfile and mask drift. Use `npm ci` for deterministic, clean CI builds.
- **Putting runtime deps in `devDependencies` (or vice-versa).** Then `npm ci --omit=dev` ships a broken app, or production carries needless build tooling. Ask: "does running code import it?"
- **Running `npm audit fix --force` blindly.** It happily installs breaking MAJOR versions. Always `--dry-run` first and run your tests.
- **Global-installing project tools.** Non-reproducible and undocumented. Use local devDependencies + npm scripts, or `npx`.
- **`npm dev` instead of `npm run dev`.** Only `start`, `test`, `stop`, `restart` get the no-`run` shorthand; everything else needs `npm run`.
- **Relying on phantom (hoisted, undeclared) dependencies.** Works until the tree reshuffles, then breaks in prod. Declare everything you import.
- **Accidentally publishing secrets.** `.env`, fixtures, or `node_modules` in your tarball. Use the `files` allow-list and `npm pack --dry-run`.
- **Expecting to overwrite a published version.** Versions are immutable; you cannot republish `1.0.0` after it ships. Bump the version every time.
- **Comments or trailing commas in `package.json`.** It is strict JSON. The annotated examples in this doc would break a real file.
- **Ignoring `engines` mismatches.** Without `engine-strict=true`, an `engines` field is just advisory — code may install on an unsupported Node and fail at runtime.

---

## Best Practices

- **Commit the lockfile; use `npm ci` in CI/CD and Docker.** Determinism is the whole point of a package manager.
- **Default to `^` ranges and let the lockfile pin exacts.** Pin exact (`--save-exact`) only for known-fragile or security-sensitive deps.
- **Categorize dependencies correctly.** Runtime → `dependencies`; tooling → `devDependencies`; plugin host requirement → `peerDependencies`.
- **Codify every workflow as an npm script.** Make the repo self-documenting; use `pre`/`post` hooks to enforce gates (lint before test, build before publish).
- **Audit and update on a cadence.** Run `npm outdated` and `npm audit` regularly; automate with Dependabot/Renovate; do MAJOR bumps one at a time with tests.
- **Defend the supply chain.** Enable 2FA on your npm account, use automation tokens in CI, review `postinstall` scripts, and consider `--ignore-scripts` for untrusted installs.
- **Minimize dependencies.** Prefer the standard library; vet weight, maintenance, and transitive count before adding; `npm prune`/`dedupe` periodically.
- **Use `npx` for one-off and scaffolding commands** instead of global installs.
- **Use the `files` allow-list and `npm pack --dry-run`** before publishing so you never leak files.
- **Pin your toolchain with `engines` + `engine-strict`** so the project fails fast on the wrong Node version.
- **One package manager per repo.** Commit only its lockfile.

---

## Key Interview Points

- **What is the difference between `dependencies` and `devDependencies`?** `dependencies` are needed to *run* the app in production; `devDependencies` are only needed during development (testing, linting, building) and are skipped by `npm install --omit=dev`.
- **Explain `^` vs `~`.** `^1.2.3` allows MINOR + PATCH updates (`>=1.2.3 <2.0.0`); `~1.2.3` allows PATCH only (`>=1.2.3 <1.3.0`). Caveat: for `0.x` packages, `^` locks the MINOR.
- **What does the major/minor/patch in SemVer signify?** MAJOR = breaking change, MINOR = backward-compatible feature, PATCH = backward-compatible bug fix.
- **`npm install` vs `npm ci`?** `install` may modify the lockfile and updates `node_modules` in place; `ci` requires a lockfile, never writes it, deletes `node_modules` for a clean install, is faster, and errors if `package.json` and the lock disagree — ideal for CI/production.
- **What is `package-lock.json` for?** It records the exact resolved version + integrity hash of every (direct and transitive) dependency, guaranteeing reproducible installs across machines.
- **Should you commit `package-lock.json`?** Yes for applications — it is the source of truth for reproducibility (not a build artifact to gitignore).
- **What are `peerDependencies`?** A declaration that the *host* project must provide a package (common for plugins/middleware) so a duplicate copy isn't bundled. npm 7+ auto-installs them and errors on conflicts (`ERESOLVE`).
- **What is `npx`?** A runner that executes a package's binary from local `node_modules`, the cache, or a temporary download — without a permanent global install. Great for scaffolding and one-off tools.
- **How do you handle a vulnerability in a transitive dependency you can't directly upgrade?** Use the `overrides` field in `package.json` (Yarn `resolutions`, pnpm `pnpm.overrides`) to force the nested dep to a patched version; or `npm audit fix`.
- **What are npm lifecycle hooks?** Auto-run scripts around named scripts (`prefoo`/`postfoo`) and around install/publish (`preinstall`, `postinstall`, `prepare`, `prepublishOnly`). Note the supply-chain risk of `postinstall`.
- **npm vs pnpm vs Yarn?** All use the npm registry; differ in lockfile + layout. pnpm uses a symlinked global content-addressable store (disk-efficient, strict, no phantom deps); Yarn Berry offers Plug'n'Play; npm is the ubiquitous default.
- **What is a phantom dependency?** A package you `require()` that works only because hoisting exposed a transitive dependency you never declared — it breaks when the tree changes. Always declare what you import.
- **How does Node resolve `require('pkg')`?** It searches `node_modules` in the current dir, then walks up parent directories' `node_modules` to the root, then errors `Cannot find module`.
- **What does the `engines` field do?** Declares supported Node/npm versions; advisory by default, enforced with `engine-strict=true` in `.npmrc`.

---

## Practice Exercises

Work through these in order; they build on each other. Solutions follow.

1. **(Warm-up) Project bootstrap.** From an empty folder, create a project named `task-tracker`, version `0.1.0`, with `express` as a dependency and `nodemon` + `eslint` as dev dependencies. Add `start` and `dev` scripts. Verify by listing only your direct dependencies.

2. **(SemVer reasoning) Predict the ranges.** Without running anything, state the exact `>=… <…` range each of these expands to, and name one version that satisfies it and one that does not: (a) `^2.4.1`, (b) `~2.4.1`, (c) `^0.4.1`, (d) `~2`, (e) `1.2.3`. Then verify (c) with `npx semver`.

3. **(Scripts & hooks) Build a script pipeline.** Add scripts so that running `npm test` first runs the linter automatically (failing the test run if lint fails), and running `npm run build` automatically cleans a `dist/` folder before building. Use lifecycle hooks, not manual chaining inside the `test` command.

4. **(Reproducibility) Diagnose a CI failure.** Your teammate added a dependency by editing `package.json` by hand (not via `npm install`) and pushed. CI runs `npm ci` and fails. Explain *why* it fails, and give the exact commands to fix it properly.

5. **(Maintenance & security) Remediate dependencies.** Given a project, show the commands to: (a) see which packages are outdated and interpret Current/Wanted/Latest, (b) safely apply in-range updates, (c) scan for vulnerabilities at "high" level only, (d) force a patched version of a transitive dep `minimist` to `^1.2.8` without upgrading its parent.

6. **(Publishing) Ship a scoped package.** You have a finished library. Write the full sequence to: log in, confirm exactly which files will be published, bump the version for a new feature, publish it *publicly* under your scope, and verify it is live. Also add the field that ensures only `src/` and the README are shipped.

### Solutions

#### Solution 1 — Project bootstrap

```bash
mkdir task-tracker && cd task-tracker

# Scaffold package.json non-interactively, then set name/version.
npm init -y
npm pkg set name="task-tracker" version="0.1.0"

# Runtime dependency.
npm install express

# Dev-only tooling.
npm install --save-dev nodemon eslint

# Add scripts via npm pkg (avoids hand-editing JSON).
npm pkg set scripts.start="node src/index.js"
npm pkg set scripts.dev="nodemon src/index.js"

# Verify only direct dependencies.
npm ls --depth=0
```

The resulting `package.json` core:

```json
{
  "name": "task-tracker",
  "version": "0.1.0",
  "scripts": {
    "start": "node src/index.js",
    "dev": "nodemon src/index.js"
  },
  "dependencies": { "express": "^4.19.2" },
  "devDependencies": { "eslint": "^9.4.0", "nodemon": "^3.1.0" }
}
```

> `npm pkg set` is the clean, scriptable way to edit `package.json` without risking JSON syntax errors.

#### Solution 2 — SemVer reasoning

| Range | Expands to | Satisfies | Does NOT satisfy |
| --- | --- | --- | --- |
| (a) `^2.4.1` | `>=2.4.1 <3.0.0` | `2.9.0` | `3.0.0` |
| (b) `~2.4.1` | `>=2.4.1 <2.5.0` | `2.4.9` | `2.5.0` |
| (c) `^0.4.1` | `>=0.4.1 <0.5.0` | `0.4.7` | `0.5.0` |
| (d) `~2` | `>=2.0.0 <3.0.0` | `2.6.3` | `3.0.0` |
| (e) `1.2.3` | `=1.2.3` only | `1.2.3` | `1.2.4` |

The key trap is (c): the caret on a `0.x` package locks the MINOR, so `0.5.0` is rejected. Verify:

```bash
npx semver 0.4.7 --range "^0.4.1"   # prints 0.4.7  (satisfies)
npx semver 0.5.0 --range "^0.4.1"   # prints nothing (does NOT satisfy)
```

#### Solution 3 — Build a script pipeline

```json
{
  "scripts": {
    "lint": "eslint .",
    "pretest": "npm run lint",
    "test": "node --test",

    "clean": "node -e \"require('fs').rmSync('dist',{recursive:true,force:true})\"",
    "prebuild": "npm run clean",
    "build": "node scripts/build.js"
  }
}
```

- Running `npm test` triggers `pretest` (lint) first; a non-zero lint exit aborts before tests run.
- Running `npm run build` triggers `prebuild` (clean) first, removing `dist/` for a fresh build.

The `clean` script uses Node's `fs.rmSync` for a cross-platform delete (avoids `rm -rf`, which fails on Windows).

#### Solution 4 — Diagnose a CI failure

**Why it fails:** `npm ci` requires `package.json` and `package-lock.json` to be perfectly in sync. The teammate added a dependency to `package.json` by hand but never ran an install, so the lockfile has **no entry** for the new package. `npm ci` refuses to resolve or modify the lockfile (by design) and errors with something like *"npm ci can only install packages when your package.json and package-lock.json are in sync."*

**The correct fix (do NOT switch CI to `npm install`):**

```bash
# Locally, regenerate the lockfile from the updated package.json.
npm install

# Commit BOTH files together.
git add package.json package-lock.json
git commit -m "fix: sync lockfile with manually added dependency"
git push
```

CI's `npm ci` now succeeds because the lockfile matches. Switching CI to `npm install` would "work" but hide the drift and defeat reproducibility — the wrong fix.

#### Solution 5 — Remediate dependencies

```bash
# (a) See outdated packages.
npm outdated
#   Current = installed now; Wanted = newest allowed by your range (npm update target);
#   Latest  = newest on registry ignoring your range (may be a new MAJOR).

# (b) Apply safe, in-range updates and refresh the lockfile.
npm update

# (c) Scan, reporting only high+ severity.
npm audit --audit-level=high

# (d) Force a patched transitive dependency without touching its parent:
#     add an override in package.json, then reinstall.
npm pkg set overrides.minimist="^1.2.8"
npm install
```

The `overrides` entry tells npm to substitute `minimist@^1.2.8` everywhere it appears in the tree, even when a parent package requested an older range.

#### Solution 6 — Ship a scoped package

```bash
# Authenticate.
npm login
npm whoami                     # confirm identity

# Ensure only the intended files ship (allow-list).
npm pkg set files[0]="src" files[1]="README.md"

# Inspect EXACTLY what would be published — read this list!
npm pack --dry-run

# Bump version for a new (backward-compatible) feature: 1.0.0 -> 1.1.0.
npm version minor

# Publish a SCOPED package publicly (scoped default is private).
npm publish --access public

# Verify it's live.
npm view @your-scope/string-utils version
```

The `files` allow-list guarantees only `src/` and `README.md` (plus always-included files like `package.json` and `LICENSE`) end up in the tarball — no stray `.env`, tests, or `node_modules`.

---

## Summary / Cheat Sheet

| Task | Command |
| --- | --- |
| Init project (defaults) | `npm init -y` |
| Add runtime dep | `npm i <pkg>` |
| Add dev dep | `npm i -D <pkg>` |
| Add exact (no `^`) | `npm i -E <pkg>` |
| Install global | `npm i -g <pkg>` |
| Prod-only install | `npm i --omit=dev` |
| Reproducible CI install | `npm ci` |
| Run a script | `npm run <name>` |
| Run with args | `npm run <name> -- <args>` |
| Run a binary once | `npx <pkg>` |
| See outdated | `npm outdated` |
| Apply in-range updates | `npm update` |
| Security scan | `npm audit` |
| Auto-fix (safe) | `npm audit fix` |
| Trace why a dep exists | `npm explain <pkg>` |
| List direct deps | `npm ls --depth=0` |
| Bump version + tag | `npm version <patch\|minor\|major>` |
| Preview publish contents | `npm pack --dry-run` |
| Publish scoped public | `npm publish --access public` |
| Edit package.json safely | `npm pkg set <field>=<value>` |

**Version ranges:** `^1.2.3` → `>=1.2.3 <2.0.0` (MINOR+PATCH) · `~1.2.3` → `>=1.2.3 <1.3.0` (PATCH) · `1.2.3` → exact · `^0.2.3` → `>=0.2.3 <0.3.0` (0.x locks MINOR!).

**Dependency buckets:** `dependencies` (runtime) · `devDependencies` (tooling) · `peerDependencies` (host provides) · `optionalDependencies` (failure tolerated).

**`install` vs `ci`:** `install` may rewrite the lock and updates in place; `ci` needs a matching lock, wipes `node_modules`, never writes the lock — use it in CI/prod.

**Golden rules:** commit `package-lock.json` · `npm ci` in CI · default `^` + lockfile pins · only `require()` what you declare · vet every dependency you add · enable 2FA before publishing.
