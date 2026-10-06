# Authorization & Web Security

Authentication answers the question *"who are you?"* — **authorization** answers the question *"are you allowed to do this?"* They are different problems, they fail in different ways, and conflating them is the source of an enormous number of real-world breaches. A user who is perfectly authenticated (a valid login, a valid JWT) can still be entirely *unauthorized* to delete another user's account — and it is the job of your authorization layer to stop them.

But authorization is only one floor of the security building. Around it sits a whole discipline of **web security**: defending against attackers who never authenticate at all, who inject malicious data into your queries, who trick *other* users' browsers into doing damage, who hammer your login endpoint with stolen passwords, or who simply read the AWS keys you accidentally committed to GitHub. A backend developer who can build a working API but cannot reason about these threats is not yet a professional backend developer.

This module is the security capstone of the course. It assumes you already understand authentication (sessions, JWTs, password hashing) from the previous module, and it builds the *authorization* layer on top, then widens out to the full perimeter: the OWASP Top 10, injection and NoSQL injection, XSS, CSRF, security headers via Helmet, CORS, rate limiting, parameter pollution, and secrets management. By the end you will be able to harden a real Express + MongoDB API and explain *why* each control exists — which is exactly what interviewers and code reviewers want to hear.

## What You'll Learn

By the end of this module you will be able to:

- Distinguish **authentication** from **authorization**, and explain why they must be separate layers.
- Compare the major **authorization models** — **RBAC** (role-based) and **ABAC** (attribute-based) — and choose the right one for a given system.
- Implement **role and permission middleware** (`restrictTo("admin")`, `hasPermission("article:delete")`) cleanly and composably.
- Defend against **ownership / IDOR** bugs (Broken Object Level Authorization) — the single most common real-world API vulnerability.
- Map the **OWASP Top 10** onto concrete Node/Express concerns and know which control mitigates which risk.
- Recognize and prevent **injection** and **NoSQL injection**, including how MongoDB **query operators** (`$gt`, `$ne`, `$where`) get abused, and sanitize input with `express-mongo-sanitize`.
- Understand **stored vs reflected XSS**, why **output escaping** (not just input filtering) is the real fix, and how to sanitize HTML safely.
- Explain **CSRF** from first principles and apply the modern defenses: **`SameSite` cookies**, the **double-submit / synchronizer token** pattern, and `csrf-csrf`.
- Configure **Helmet** and know what each security header (**CSP, HSTS, X-Frame-Options**, etc.) actually does.
- Configure **CORS** correctly — origins, **credentials**, and the **preflight (`OPTIONS`)** request — without accidentally disabling the same-origin policy.
- Add **rate limiting and brute-force protection** with `express-rate-limit`, and slow down repeated failed logins.
- Prevent **HTTP Parameter Pollution** with `hpp`.
- Manage **secrets and environment variables** safely, keep them out of git, and run **HTTPS in production**.
- Apply a complete, practical **security hardening checklist** to any Express API.

> Note: This module targets **Node.js current LTS** and **Express v5** (the default `express` on npm since 2024), and flags v4 vs v5 differences where they matter. All examples use **CommonJS (`require`)** by default. Express v5 automatically forwards rejected promises from async handlers to your error middleware — older v4 code needed a `catchAsync` wrapper, which we note where relevant.

---

## 1. Authentication vs Authorization (the foundation)

Before any code, lock in the distinction. These two words are constantly confused — even in production codebases — and the confusion *is* a vulnerability.

| | Authentication (AuthN) | Authorization (AuthZ) |
| --- | --- | --- |
| Question it answers | "Who are you?" | "What are you allowed to do?" |
| When it runs | Once per request (verify token/session) | After authentication, on every protected action |
| Typical mechanism | Password + hash, JWT, session cookie | Roles, permissions, ownership checks, policies |
| Failure mode | Anyone can log in as anyone | A logged-in user can do things they shouldn't |
| HTTP status on failure | **401 Unauthorized** | **403 Forbidden** |

> WARNING — Pitfall: The HTTP status codes are confusingly named. **401 "Unauthorized"** actually means **"unauthenticated"** — you have not proven who you are (no/invalid credentials). **403 "Forbidden"** means **"authenticated but not authorized"** — we know who you are, and you are not allowed. Use 401 when the token is missing/expired/invalid; use 403 when the token is valid but the user lacks the role/permission/ownership.

The mental model for an Express request to a protected endpoint:

```
Request
  → [authenticate]   verify JWT/session, attach req.user   → 401 if it fails
  → [authorize]      check role / permission / ownership    → 403 if it fails
  → [route handler]  do the actual work
```

Authentication runs **first** and populates `req.user`. Authorization runs **after** and *reads* `req.user`. Every example in this module assumes an upstream `protect`/`authenticate` middleware has already set `req.user` to something like:

```js
// Shape of req.user that authentication produced (from the previous module).
// Authorization middleware reads from this — it never re-verifies the token.
req.user = {
  _id: '652f...',
  email: 'ada@example.com',
  role: 'admin',                       // for RBAC
  permissions: ['article:read', 'article:write'], // for permission-based / ABAC
};
```

---

## 2. Authorization Models: RBAC and ABAC

There is no single "correct" way to model who-can-do-what. There is a spectrum from simple-and-coarse to flexible-and-complex. The two models you must know cold are **RBAC** and **ABAC**.

### 2.1 Role-Based Access Control (RBAC)

In RBAC, you assign each user one or more **roles** (`admin`, `editor`, `user`), and you grant **permissions to roles**, not to individual users. Access decisions are then: *"Does this user's role include the permission this action requires?"*

This is the most widely used model because it maps cleanly onto how organizations actually think ("managers can approve expenses; employees can submit them"). It is easy to reason about, easy to audit, and easy to implement.

```js
// The conceptual data model behind RBAC.
const ROLE_PERMISSIONS = {
  admin:  ['article:read', 'article:write', 'article:delete', 'user:manage'],
  editor: ['article:read', 'article:write'],
  user:   ['article:read'],
};

// A user simply HAS a role; permissions are derived from the role.
const user = { _id: '1', role: 'editor' };

function roleCan(role, permission) {
  return (ROLE_PERMISSIONS[role] || []).includes(permission);
}

roleCan(user.role, 'article:write');  // true
roleCan(user.role, 'article:delete'); // false
```

**Two flavours of RBAC** show up in interviews:

- **Role-checking** (coarse): `restrictTo("admin", "editor")` — "you must hold one of these roles." Simple, but couples your route to role *names*. Adding a new role means hunting through every route.
- **Permission-checking** (fine): `hasPermission("article:delete")` — "your role must grant this permission." The route names the *capability* it needs, not the role. You can invent new roles purely by editing the role→permission map, with zero route changes. **Prefer this** for anything beyond a tiny app.

> BEST PRACTICE: Check **permissions, not roles**, at the route. `hasPermission("article:delete")` is far more maintainable than `restrictTo("admin")`, because business rules ("editors can now delete drafts") become a data change in one place instead of a code change across many routes.

### 2.2 Attribute-Based Access Control (ABAC)

RBAC breaks down when the decision depends on **context** that a static role cannot capture:

- "A user can edit *their own* posts" (depends on ownership — *who* relative to *what*).
- "A manager can view salaries *only for their own department*."
- "Documents can be accessed *only during business hours* from a *corporate IP*."
- "A user can approve an invoice *only if its amount is under their approval limit*."

These rules involve **attributes** of the *subject* (the user), the *resource* (the thing being accessed), the *action*, and the *environment* (time, IP, location). **ABAC** evaluates a **policy** — a function of all those attributes — to make a yes/no decision. ABAC is strictly more expressive than RBAC (RBAC is the special case where the only attribute that matters is the subject's role).

```js
// An ABAC policy is just a predicate over (subject, resource, action, environment).
function canEditArticle(user, article, env) {
  // Admins can edit anything.
  if (user.role === 'admin') return true;

  // Editors can edit any article, but only during business hours.
  if (user.role === 'editor') {
    const hour = env.now.getHours();
    return hour >= 9 && hour < 18;
  }

  // Regular users can edit ONLY their own articles (ownership attribute).
  return String(article.authorId) === String(user._id);
}

const allowed = canEditArticle(req.user, article, { now: new Date() });
```

### 2.3 Choosing a model

| Aspect | RBAC | ABAC |
| --- | --- | --- |
| Decision based on | The user's role(s) | Attributes of user + resource + action + context |
| Granularity | Coarse to medium | Arbitrarily fine |
| Ownership rules ("own posts only") | Awkward (role can't express "own") | Natural |
| Complexity / cost | Low — easy to build and audit | Higher — needs a policy engine and resource loading |
| Scales to | Most apps | Complex, multi-tenant, compliance-heavy systems |
| Real-world examples | Most CRUD apps, admin dashboards | AWS IAM, healthcare/finance, multi-tenant SaaS |

> BEST PRACTICE: Start with **RBAC for broad gates** (is this person staff?) and layer **ABAC-style ownership checks** for resource-specific rules (is this *their* record?). This hybrid covers the vast majority of applications: a small set of roles for coarse access, plus per-resource ownership predicates for fine access. You rarely need a full policy engine on day one.

> TIP: For real ABAC at scale, you do not hand-roll a policy engine. Libraries like **CASL** (`@casl/ability`), **AccessControl**, or external services (Open Policy Agent / OPA, Oso) let you declare abilities/policies as data and check them with `ability.can('update', subject('Article', article))`.

---

## 3. Implementing Role & Permission Middleware

Now the practical core: turning these models into Express middleware. The pattern is a **middleware factory** — a function that *returns* a middleware configured for a specific role/permission. (If the factory idea is unfamiliar, recall that middleware is just `(req, res, next)`; a factory is a function that builds one.)

### 3.1 `restrictTo(...roles)` — role gating

```js
// authz.js — role-based gate

// A reusable AppError (assume your error-handling middleware turns it into JSON).
class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.statusCode = statusCode;
    this.status = `${statusCode}`.startsWith('4') ? 'fail' : 'error';
    this.isOperational = true; // distinguishes expected errors from bugs
  }
}

/**
 * restrictTo('admin', 'editor') => middleware that allows only those roles.
 * MUST run AFTER an authentication middleware that set req.user.
 */
function restrictTo(...allowedRoles) {
  return (req, res, next) => {
    // Defensive: if authentication didn't run, fail closed (don't assume).
    if (!req.user || !req.user.role) {
      return next(new AppError('You are not logged in.', 401));
    }
    if (!allowedRoles.includes(req.user.role)) {
      // Authenticated but lacks the role => 403, not 401.
      return next(new AppError('You do not have permission to perform this action.', 403));
    }
    next(); // role is allowed — continue down the chain
  };
}

module.exports = { restrictTo, AppError };
```

Usage on routes:

```js
const express = require('express');
const router = express.Router();
const { protect } = require('./auth');          // sets req.user (authentication)
const { restrictTo } = require('./authz');      // checks role (authorization)
const articleController = require('./articleController');

// Everyone authenticated can read.
router.get('/', protect, articleController.list);

// Only admins and editors can create.
router.post('/', protect, restrictTo('admin', 'editor'), articleController.create);

// Only admins can hard-delete.
router.delete('/:id', protect, restrictTo('admin'), articleController.remove);

module.exports = router;
```

Note the **order**: `protect` (authN) *must* come before `restrictTo` (authZ), because `restrictTo` reads `req.user`. If you swap them, `req.user` is undefined and every request is rejected (fail-closed, which is at least safe — but wrong).

### 3.2 `hasPermission(...)` — fine-grained permission gating

The more maintainable approach checks capabilities, with permissions derived from the user's role.

```js
// permissions.js
const ROLE_PERMISSIONS = {
  admin:  ['article:read', 'article:write', 'article:delete', 'user:manage'],
  editor: ['article:read', 'article:write'],
  user:   ['article:read'],
};

// Resolve the effective permission set for a user.
// Supports multiple roles and explicit per-user permissions.
function permissionsFor(user) {
  const fromRoles = (user.roles || [user.role] || [])
    .flatMap((role) => ROLE_PERMISSIONS[role] || []);
  const explicit = user.permissions || []; // direct grants (ABAC-ish)
  return new Set([...fromRoles, ...explicit]);
}

/**
 * hasPermission('article:delete') => middleware requiring that capability.
 * Pass multiple to require ALL of them.
 */
function hasPermission(...required) {
  return (req, res, next) => {
    if (!req.user) return next(new AppError('You are not logged in.', 401));
    const granted = permissionsFor(req.user);
    const ok = required.every((perm) => granted.has(perm));
    if (!ok) {
      return next(new AppError('Insufficient permissions.', 403));
    }
    next();
  };
}

module.exports = { hasPermission, permissionsFor, ROLE_PERMISSIONS };
```

```js
// Now routes name the CAPABILITY, not the role. Roles can change freely.
router.post('/',       protect, hasPermission('article:write'),  articleController.create);
router.delete('/:id',  protect, hasPermission('article:delete'), articleController.remove);
router.post('/users',  protect, hasPermission('user:manage'),    userController.create);
```

### 3.3 Ownership checks (ABAC) — the IDOR defense

Here is the bug that role checks **cannot** catch and that breaks more APIs than anything else. Consider:

```js
// DANGEROUSLY INCOMPLETE: any logged-in user can update ANY article.
router.patch('/:id', protect, restrictTo('user', 'editor', 'admin'), async (req, res) => {
  const article = await Article.findByIdAndUpdate(req.params.id, req.body, { new: true });
  res.json(article);
});
```

The role check passes (the attacker *is* a `user`), but nothing checks that the article belongs to them. Attacker sends `PATCH /articles/<someone-elses-id>` and edits another person's content. This is **IDOR — Insecure Direct Object Reference**, the practical form of OWASP's **Broken Object Level Authorization (BOLA)**, and it is consistently the #1 API vulnerability found in the wild.

The fix is an **ownership-aware** authorization step that loads the resource and compares it to `req.user`:

```js
// Generic ownership guard: loads the doc and checks its owner field against req.user.
// Admins bypass ownership (a common, deliberate policy choice).
function restrictToOwner(Model, ownerField = 'authorId') {
  return async (req, res, next) => {
    const doc = await Model.findById(req.params.id);
    if (!doc) return next(new AppError('Resource not found.', 404));

    const isOwner = String(doc[ownerField]) === String(req.user._id);
    const isAdmin = req.user.role === 'admin';

    if (!isOwner && !isAdmin) {
      return next(new AppError('You can only modify your own resources.', 403));
    }

    req.resource = doc; // stash it so the handler doesn't re-query
    next();
  };
}

// Usage: role gate + ownership gate together.
router.patch('/:id', protect, restrictToOwner(Article), articleController.update);
router.delete('/:id', protect, restrictToOwner(Article), articleController.remove);
```

> WARNING — Pitfall: **404 vs 403 leaks information.** If you return `403 Forbidden` for resources that exist but the user doesn't own, and `404 Not Found` for resources that don't exist, an attacker can *enumerate which IDs exist* by watching the status code. For sensitive systems, return **404 for both** "doesn't exist" and "exists but you can't see it," so existence itself is hidden. Decide this consciously per resource.

> WARNING — Pitfall: **Never trust IDs or roles from the request body.** A classic privilege-escalation bug is letting `req.body.role` or `req.body.userId` flow into your update. Always derive the owner/role from `req.user` (set by trusted authentication), never from client input. Whitelist updatable fields explicitly.

```js
// Strip dangerous fields before updating. NEVER let the client set role/owner.
function pickAllowed(body, allowed) {
  return Object.fromEntries(Object.entries(body).filter(([k]) => allowed.includes(k)));
}

router.patch('/me', protect, async (req, res, next) => {
  // A user can update name/email — but NOT role, _id, isVerified, etc.
  const safe = pickAllowed(req.body, ['name', 'email']);
  const updated = await User.findByIdAndUpdate(req.user._id, safe, { new: true, runValidators: true });
  res.json(updated);
});
```

### 3.4 Composing it all

```js
// A fully guarded route reads top-to-bottom as a security policy:
router.delete(
  '/:id',
  protect,                      // 1. AuthN: must be logged in (else 401)
  hasPermission('article:delete'), // 2. AuthZ (RBAC): role must grant delete (else 403)
  restrictToOwner(Article),     // 3. AuthZ (ABAC): must own it, or be admin (else 403/404)
  articleController.remove,     // 4. The actual work
);
```

---

## 4. The OWASP Top 10, Mapped to Node/Express

The **OWASP Top 10** is the industry-standard awareness list of the most critical web application security risks (current edition: 2021; a 2025 revision is in progress). You should be able to name them and map each to a concrete Node/Express concern and control. This table is the backbone of the rest of the module.

| OWASP (2021) | What it is | Node/Express concern | Primary control(s) in this module |
| --- | --- | --- | --- |
| **A01 Broken Access Control** | Users act outside intended permissions (IDOR/BOLA, privilege escalation) | Missing/incorrect authorization, trusting `req.body.role`, no ownership check | §2–3: RBAC/ABAC middleware, ownership guards, field whitelisting |
| **A02 Cryptographic Failures** | Sensitive data exposed (weak/no encryption, plaintext secrets) | Plaintext passwords, secrets in code, no HTTPS, weak JWT secret | §11: secrets mgmt, HTTPS; bcrypt/argon2 (prev. module) |
| **A03 Injection** | Untrusted data interpreted as a command/query | NoSQL injection via query operators, command injection, XSS (a form of injection) | §5 (NoSQL), §6 (XSS) |
| **A04 Insecure Design** | Missing security controls by design | No rate limiting, no threat modeling, missing authZ layer | §3, §9 (rate limiting), defense-in-depth throughout |
| **A05 Security Misconfiguration** | Insecure defaults, verbose errors, missing headers | `x-powered-by`, stack traces leaked, no Helmet, permissive CORS | §7 (Helmet), §8 (CORS), error handling |
| **A06 Vulnerable & Outdated Components** | Known-CVE dependencies | Old npm packages with advisories | `npm audit`, Dependabot, lockfiles (§11/§12) |
| **A07 Identification & Authentication Failures** | Weak auth, brute-forceable logins, bad session mgmt | No rate limit on login, weak password policy, JWT issues | §9 (brute-force), auth module |
| **A08 Software & Data Integrity Failures** | Untrusted code/data, insecure deserialization, supply chain | Malicious npm packages, unsigned updates | Lockfile + `npm ci`, integrity hashes (§12) |
| **A09 Security Logging & Monitoring Failures** | Can't detect/respond to attacks | No request logging, no alerting on repeated 401/403 | Logging (morgan/pino), monitor auth failures |
| **A10 Server-Side Request Forgery (SSRF)** | Server fetches attacker-controlled URLs | Passing user input into `fetch`/`axios` to internal hosts | Validate/allowlist outbound URLs; block internal IPs |

> TIP: In interviews, you are not expected to recite all ten verbatim, but you *should* be able to (1) name **A01 Broken Access Control** as the #1 risk, (2) explain it with the IDOR example, and (3) connect at least injection, misconfiguration, and auth failures to specific Express controls. The mapping above is the thing to internalize.

The remaining sections work through the controls that matter most for an Express + Mongo API.

---

## 5. Injection & NoSQL Injection

**Injection** is what happens when untrusted input is interpreted as part of a *command* or *query* instead of as inert *data*. SQL injection is the famous example; with MongoDB the equivalent is **NoSQL injection**, and the attack vector is different and surprising: it abuses **MongoDB query operators**.

### 5.1 How NoSQL injection works

A naive login check:

```js
// VULNERABLE login. Looks innocent — but req.body is attacker-controlled.
app.post('/login', async (req, res) => {
  const user = await User.findOne({
    email: req.body.email,
    password: req.body.password, // (assume plaintext for illustration of the attack)
  });
  if (!user) return res.status(401).json({ error: 'Invalid credentials' });
  res.json({ token: signToken(user) });
});
```

If the body is parsed from **JSON**, the attacker can send *objects* where you expected strings:

```json
{
  "email": "admin@example.com",
  "password": { "$ne": null }
}
```

Now the query becomes `{ email: 'admin@example.com', password: { $ne: null } }` — "find the admin whose password is **not equal to null**", i.e. *any* password. The attacker logs in as admin **without knowing the password**. Other weaponized operators:

- `{ "$gt": "" }` — "greater than empty string" matches almost anything.
- `{ "$ne": -1 }` — matches everything not equal to -1.
- `{ "$regex": "^a" }` — used to **extract** secrets character-by-character (a blind injection / data exfiltration technique).
- `{ "$where": "this.password.length > 0" }` — `$where` and `mapReduce` can run **arbitrary JavaScript** on the server — extremely dangerous; disable server-side JS in production.

> WARNING — Pitfall: This attack works specifically because **`express.json()` turns JSON objects in the body into JS objects**, and Mongoose/Mongo happily interpret a `{ $ne: ... }` object as an operator. URL query strings can also produce objects: Express's default `qs` parser turns `?password[$ne]=` into `{ password: { $ne: '' } }`. **Both `req.body` and `req.query` are attack surfaces.**

### 5.2 Defenses

There are three complementary defenses; use them together (defense in depth).

**Defense 1 — Sanitize input with `express-mongo-sanitize`.** It strips keys containing `$` and `.` (the operator and dot-path characters) from `req.body`, `req.query`, and `req.params`.

```bash
npm install express-mongo-sanitize
```

```js
const mongoSanitize = require('express-mongo-sanitize');

// Remove any keys that start with '$' or contain '.' from user input.
app.use(mongoSanitize());

// Or replace prohibited characters instead of deleting keys:
app.use(mongoSanitize({ replaceWith: '_' }));

// With logging, so you can monitor attempted attacks (A09):
app.use(mongoSanitize({
  onSanitize: ({ req, key }) => {
    console.warn(`[security] sanitized prohibited key "${key}" from ${req.path}`);
  },
}));
```

> WARNING — Pitfall (Express v5 + `express-mongo-sanitize`): In Express **v5**, `req.query` is a **getter with no setter** — it cannot be reassigned. The classic `express-mongo-sanitize` middleware tries to overwrite `req.query` and **throws** ("Cannot set property query"). Workarounds: (a) sanitize each property's *contents* without reassigning the object, (b) use a maintained fork such as `@exortek/express-mongo-sanitize` built for v5, or (c) validate types yourself (Defense 3). Always test your sanitizer against your Express major version.

**Defense 2 — Type-check and validate.** The root cause is "a string field received an object." Validate that fields are the type you expect, ideally with a schema validator like **Zod** or **Joi**. This stops the attack regardless of any sanitizer quirks.

```bash
npm install zod
```

```js
const { z } = require('zod');

const loginSchema = z.object({
  email: z.string().email(),          // MUST be a string, and a valid email
  password: z.string().min(8).max(128), // MUST be a string — an object fails here
});

app.post('/login', async (req, res, next) => {
  const result = loginSchema.safeParse(req.body);
  if (!result.success) {
    return res.status(400).json({ error: 'Invalid input', details: result.error.flatten() });
  }
  const { email, password } = result.data; // now guaranteed to be strings
  // ... safe to query: password is a string, never an operator object
});
```

**Defense 3 — Coerce/cast at the query boundary.** Even without a validator, forcing the value to a primitive defeats operator injection:

```js
// Force the value to a string before it ever reaches the query.
const email = String(req.body.email);
const user = await User.findOne({ email }); // an object would become "[object Object]" — harmless
```

> BEST PRACTICE: Mongoose **schema typing** is itself a partial defense — if a field is declared `type: String`, Mongoose will reject/cast an operator object on *writes*, and casting helps on `find` by `_id`. But Mongoose does **not** validate the *shape* of arbitrary `find()` filters you build from `req.body`, so you still need explicit validation/sanitization for query construction. Disable server-side JavaScript (`--noscripting` / `security.javascriptEnabled: false`) to kill `$where` injection entirely.

### 5.3 Command injection (the non-Mongo cousin)

If your Node app ever shells out, you have a second injection surface:

```js
const { exec, execFile } = require('child_process');

// VULNERABLE: user input concatenated into a shell command.
exec(`convert ${req.body.filename} out.png`); // filename = "x.png; rm -rf /" → disaster

// SAFE: execFile passes args as an array — no shell interpretation.
execFile('convert', [req.body.filename, 'out.png']); // arg is data, not code
```

> BEST PRACTICE: Prefer `execFile`/`spawn` with an **argument array** over `exec` with an interpolated string. Never build shell commands by string concatenation with user input. The same principle underlies *all* injection defense: keep **data** and **code/query** strictly separated.

---

## 6. Cross-Site Scripting (XSS)

**XSS** is injection where the "interpreter" is the *victim's browser* and the injected "code" is JavaScript. If an attacker can get their script to run in another user's browser session, they can steal cookies/tokens, perform actions as that user, deface pages, or keylog. For an API that returns JSON consumed by a SPA, XSS risk shifts to the frontend — but a backend that stores or reflects user HTML is the *source* of stored XSS, so you must understand both sides.

### 6.1 The three types

| Type | Where the payload lives | Example trigger |
| --- | --- | --- |
| **Stored (persistent)** | Saved in your DB, served to every viewer | A comment containing `<script>` is saved and rendered to all readers — worst kind |
| **Reflected** | Bounced back in the immediate response | Search page echoes `?q=<script>...` into HTML without escaping |
| **DOM-based** | Entirely client-side | Frontend does `el.innerHTML = location.hash` — never touches the server |

### 6.2 The real fix is OUTPUT escaping, not input filtering

The single most important XSS concept: **context-aware output encoding**. The same data is dangerous in one context and safe in another. The fix is to **escape data at the moment you insert it into a page**, according to *where* you're putting it (HTML body, attribute, JS, URL, CSS — each needs different escaping).

```js
// Manual HTML escaping for the HTML-body context.
function escapeHtml(str) {
  return String(str)
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#39;');
}

// <script>alert(1)</script> becomes &lt;script&gt;alert(1)&lt;/script&gt; — inert text.
```

If you render server-side HTML, use a template engine that **auto-escapes by default** (Pug, EJS with `<%= %>`, Handlebars with `{{ }}`):

```js
// EJS: <%= comment %> escapes;  <%- comment %> does NOT (raw HTML — dangerous).
// Handlebars: {{ comment }} escapes; {{{ comment }}} does NOT.
// Pug: #{comment} escapes; != renders raw.
```

> WARNING — Pitfall: **Input filtering alone is not enough**, and "strip `<script>`" blacklists are trivially bypassed (`<img src=x onerror=alert(1)>`, `<svg onload=...>`, encoded variants, etc.). The robust strategy is: **store data as-is (validated), escape on output, and add a Content-Security-Policy as a backstop.** Output encoding is correct because it is context-specific and complete; input filtering is best-effort and lossy.

### 6.3 When you must accept rich HTML: sanitize with an allowlist

Sometimes you genuinely need to store user HTML (a rich-text editor, Markdown rendering). Then you must **sanitize with a vetted, allowlist-based library** — never a homemade regex. The gold standard is **DOMPurify** (via `jsdom` on the server) or **sanitize-html**.

```bash
npm install dompurify jsdom
```

```js
const { JSDOM } = require('jsdom');
const createDOMPurify = require('dompurify');
const DOMPurify = createDOMPurify(new JSDOM('').window);

// Allow only safe formatting tags; strip <script>, on* handlers, javascript: URLs, etc.
function sanitizeRichText(dirty) {
  return DOMPurify.sanitize(dirty, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'ul', 'ol', 'li', 'br'],
    ALLOWED_ATTR: ['href', 'title'],
  });
}

// Sanitize BEFORE storing rich HTML so the stored value is already safe.
const cleanBody = sanitizeRichText(req.body.content);
```

### 6.4 A note on `xss-clean`

For years the standard advice was the **`xss-clean`** middleware (`app.use(xss())`), which sanitizes `req.body`/`req.query`/`req.params` by encoding `<` `>` etc. globally. It still appears in countless tutorials and codebases, so know what it is — **but it is deprecated and unmaintained**, and like all global input-encoders it can corrupt legitimate data and gives a false sense of security. Modern guidance:

- Do **not** rely on global input sanitization as your XSS defense.
- Use **output escaping** (auto-escaping templates / proper JSON serialization) as the primary control.
- Use **DOMPurify/sanitize-html** at the points where you accept and render rich HTML.
- Add **CSP via Helmet** (next section) as defense in depth.

```js
// If you must mirror the old approach, prefer a maintained allowlist sanitizer
// applied to SPECIFIC fields, not a blanket global encoder:
const sanitizeHtml = require('sanitize-html');
req.body.bio = sanitizeHtml(req.body.bio, { allowedTags: [], allowedAttributes: {} }); // strip ALL html
```

> BEST PRACTICE: For a **JSON API** consumed by a SPA, your job is to (1) validate/whitelist input, (2) store it faithfully, (3) **never** build HTML by string-concatenating stored data, and (4) sanitize any field that will be rendered as HTML. The browser-side framework (React/Vue/Angular) escapes by default when rendering text nodes — the danger is `dangerouslySetInnerHTML`/`v-html`/`[innerHTML]` fed unsanitized server data.

---

## 7. CSRF (Cross-Site Request Forgery)

**CSRF** tricks a logged-in user's browser into making a state-changing request to *your* site **without the user's intent**. The attack exploits the fact that browsers **automatically attach cookies** to requests to a domain — so a malicious page can cause the victim's browser to send a cookie-authenticated request to your API.

### 7.1 The attack, concretely

Suppose your bank uses a session **cookie** and accepts:

```
POST /transfer  body: { to: "...", amount: "..." }
```

A victim is logged in (valid session cookie). They visit `evil.com`, which contains:

```html
<!-- Auto-submitting form on the attacker's page -->
<form id="f" action="https://bank.com/transfer" method="POST">
  <input name="to" value="attacker-account">
  <input name="amount" value="10000">
</form>
<script>document.getElementById('f').submit();</script>
```

The browser submits the form to `bank.com` **and attaches the victim's session cookie automatically**. The server sees a valid session and processes the transfer. The victim never clicked anything meaningful. *That* is CSRF.

> Key insight: CSRF specifically attacks **cookie/automatic credential** auth. If your API uses a **JWT in an `Authorization: Bearer` header** (read from JS-accessible storage and set manually by your frontend), the attacker's cross-site request **cannot** add that header → such APIs are largely **immune to classic CSRF**. The trade-off: header-based JWTs in `localStorage` are exposed to **XSS**. There is no free lunch — pick the threat model deliberately.

### 7.2 Defenses

**Defense 1 — `SameSite` cookies (your first line of defense, free and modern).** The `SameSite` cookie attribute tells the browser whether to send a cookie on cross-site requests.

```js
res.cookie('sessionId', token, {
  httpOnly: true,   // JS can't read it (mitigates XSS-based theft)
  secure: true,     // only over HTTPS
  sameSite: 'lax',  // NOT sent on cross-site POST/PUT/DELETE → blocks most CSRF
  maxAge: 1000 * 60 * 60,
});
```

| `SameSite` value | Cross-site behavior | CSRF protection | UX cost |
| --- | --- | --- | --- |
| `Strict` | Cookie never sent cross-site at all | Strongest | Following a link from another site logs you out / loses session |
| `Lax` (modern default) | Sent on top-level **GET** navigations only, **not** on cross-site POST/PUT/DELETE | Strong for state-changing requests | Minimal — good default |
| `None` | Always sent (requires `Secure`) | None | Needed for legit cross-site cookies (e.g. embedded widgets) |

> BEST PRACTICE: Set `SameSite=Lax` (or `Strict` for highly sensitive apps) on auth cookies. Combined with `httpOnly` and `secure`, this neutralizes the large majority of CSRF with zero added tokens. But `SameSite` is browser-dependent and not a complete guarantee — for sensitive, cookie-based actions, **add a CSRF token** (Defense 2).

**Defense 2 — CSRF tokens (synchronizer / double-submit pattern).** The server issues an unpredictable token that the legitimate frontend includes in each state-changing request (in a header or hidden field). The attacker's cross-site page **cannot read** this token (same-origin policy), so it cannot forge a valid request.

> WARNING — Pitfall: The long-popular **`csurf`** package is **deprecated and unmaintained** (last published years ago, with known issues). Do not use it in new code despite the thousands of tutorials that show it. Use the maintained **`csrf-csrf`** (double-submit cookie) or **`csrf-sync`** (synchronizer token) packages, or a framework-provided mechanism.

```bash
npm install csrf-csrf cookie-parser
```

```js
const cookieParser = require('cookie-parser');
const { doubleCsrf } = require('csrf-csrf');

app.use(cookieParser());

const { doubleCsrfProtection, generateCsrfToken } = doubleCsrf({
  getSecret: () => process.env.CSRF_SECRET,     // strong, from env
  getSessionIdentifier: (req) => req.user?.id ?? req.ip, // bind token to a session/user
  cookieName: '__Host-psifi.x-csrf-token',       // __Host- prefix hardens the cookie
  cookieOptions: { sameSite: 'lax', secure: true, httpOnly: true },
  getCsrfTokenFromRequest: (req) => req.headers['x-csrf-token'], // where the SPA sends it
});

// Endpoint the SPA calls to fetch a token for subsequent requests.
app.get('/csrf-token', (req, res) => {
  res.json({ csrfToken: generateCsrfToken(req, res) });
});

// Protect state-changing routes. GET/HEAD/OPTIONS are typically exempt.
app.post('/transfer', doubleCsrfProtection, (req, res) => {
  // Reaches here only if a valid CSRF token accompanied the request.
  res.json({ ok: true });
});
```

The frontend flow: `GET /csrf-token` once → store the token → send it as `X-CSRF-Token` on every POST/PUT/PATCH/DELETE.

> BEST PRACTICE: CSRF protection is needed when you authenticate with **cookies**. For pure header-token APIs it is usually unnecessary. Apply CSRF tokens to **state-changing** methods only (never to safe `GET`s — and ensure your GETs truly are side-effect-free, because a GET that mutates state is itself a CSRF vulnerability).

---

## 8. Helmet & Security Headers

Browsers enforce many security behaviors **only if the server tells them to**, via **HTTP response headers**. **Helmet** is Express middleware that sets a sensible bundle of these headers in one line. It does not "secure" your app by itself — it hardens the browser-side perimeter.

```bash
npm install helmet
```

```js
const helmet = require('helmet');
app.use(helmet()); // sets a strong default set of headers
```

### What Helmet sets (and why each matters)

| Header (Helmet middleware) | What it does | Risk it mitigates |
| --- | --- | --- |
| **Content-Security-Policy** (`contentSecurityPolicy`) | Allowlist of sources for scripts/styles/images/etc. Blocks inline & foreign scripts | XSS (the strongest backstop), data exfiltration |
| **Strict-Transport-Security / HSTS** (`hsts`) | Forces browsers to use HTTPS for `maxAge` seconds, even if user types `http://` | Protocol downgrade & SSL-strip MITM |
| **X-Content-Type-Options: nosniff** (`noSniff`) | Stops the browser from MIME-sniffing a response into a different type | "Drive-by" execution of misdetected content |
| **X-Frame-Options / frame-ancestors** (`frameguard`) | Forbids putting your page in an `<iframe>` | **Clickjacking** |
| **Referrer-Policy** (`referrerPolicy`) | Limits the `Referer` header sent to other sites | Leaking URLs/tokens via referrer |
| **Cross-Origin-Resource-Policy / -Opener / -Embedder** | Isolates your resources/window from other origins | Cross-origin leaks (Spectre-class) |
| **Origin-Agent-Cluster** | Requests origin-keyed process isolation | Cross-origin memory side-channels |
| **X-DNS-Prefetch-Control** | Controls speculative DNS prefetching | Minor privacy leak |
| **X-Powered-By removal** (`hidePoweredBy`) | Removes the `X-Powered-By: Express` header | Reduces fingerprinting (A05) |
| **X-Download-Options / X-Permitted-Cross-Domain-Policies** | Legacy IE / Flash/PDF hardening | Legacy plugin attacks |

> Note: Helmet does **not** set HSTS unless the response is served over HTTPS-appropriate conditions in some setups, and its **default CSP is fairly strict** — it can break inline scripts/styles and CDNs, which is *intended*. You almost always must **customize CSP** for a real app.

```js
// A realistic customized policy. Add only the sources you actually use.
app.use(
  helmet({
    contentSecurityPolicy: {
      useDefaults: true,
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'", 'https://cdn.jsdelivr.net'], // avoid 'unsafe-inline'
        styleSrc: ["'self'", "'unsafe-inline'", 'https://fonts.googleapis.com'],
        imgSrc: ["'self'", 'data:', 'https:'],
        connectSrc: ["'self'", 'https://api.example.com'],
        fontSrc: ["'self'", 'https://fonts.gstatic.com'],
        objectSrc: ["'none'"],        // no <object>/<embed> (Flash-era attacks)
        frameAncestors: ["'self'"],   // anti-clickjacking
        upgradeInsecureRequests: [],  // auto-upgrade http subresources to https
      },
    },
    hsts: { maxAge: 31536000, includeSubDomains: true, preload: true }, // 1 year
  }),
);
```

> BEST PRACTICE: Put `app.use(helmet())` **near the very top** of your middleware stack (before routes) so the headers apply to every response, including errors. For a pure JSON API you can often run the **strictest** CSP (e.g. `default-src 'none'`) because the API never serves HTML — the policy then only matters for any docs/error pages. Avoid `'unsafe-inline'`/`'unsafe-eval'` in `script-src`; they defeat CSP's main XSS protection. Test CSP in **report-only** mode first (`Content-Security-Policy-Report-Only`).

---

## 9. CORS In Depth

**CORS — Cross-Origin Resource Sharing** is one of the most misunderstood topics in web dev. The confusion comes from a backwards mental model. CORS does **not** secure your server; it is a mechanism for **relaxing** the browser's default **Same-Origin Policy (SOP)** in a controlled way.

### 9.1 Same-Origin Policy first

An **origin** is the triple **(scheme, host, port)**. `https://app.com` and `https://api.app.com` are *different origins* (different host). The browser's SOP blocks JavaScript on one origin from **reading** responses from another origin. This is a browser-enforced safety default. CORS is the server's way of saying *"these specific other origins are allowed to read my responses."*

```
https://app.com        and  https://api.app.com   → different (host)
https://app.com        and  http://app.com        → different (scheme)
https://app.com        and  https://app.com:8443   → different (port)
https://app.com/a      and  https://app.com/b      → SAME (path doesn't matter)
```

> WARNING — Pitfall: CORS is enforced by the **browser**, not your server. A server that *omits* CORS headers is **not protected** — `curl`, Postman, mobile apps, and other servers ignore CORS entirely and can call your API freely. CORS only controls whether **browser JavaScript on another origin** is permitted to *read* the response. Real access control is still your authorization layer (§3).

### 9.2 The preflight (`OPTIONS`) request

For "non-simple" cross-origin requests, the browser first sends an automatic **preflight** `OPTIONS` request to ask permission *before* the real request. A request is **"simple"** (no preflight) only if it's `GET`/`HEAD`/`POST` with a safelisted `Content-Type` (`text/plain`, `application/x-www-form-urlencoded`, `multipart/form-data`) and no custom headers. **Anything else triggers a preflight** — notably:

- `Content-Type: application/json` (almost every modern API!)
- Methods `PUT`, `PATCH`, `DELETE`
- Custom headers like `Authorization` or `X-CSRF-Token`

The preflight exchange:

```
# Browser → server (automatic, before the real request)
OPTIONS /articles HTTP/1.1
Origin: https://app.com
Access-Control-Request-Method: DELETE
Access-Control-Request-Headers: authorization, content-type

# Server → browser (grant permission)
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.com
Access-Control-Allow-Methods: GET,POST,PUT,PATCH,DELETE
Access-Control-Allow-Headers: authorization, content-type
Access-Control-Max-Age: 600           # cache this preflight for 10 min
```

Only if the preflight is granted does the browser send the real `DELETE`.

### 9.3 Configuring the `cors` middleware

```bash
npm install cors
```

```js
const cors = require('cors');

// Simplest (DEV ONLY): allow all origins. Never use '*' with credentials.
app.use(cors());
```

A production-grade configuration with an **origin allowlist** and **credentials**:

```js
const allowedOrigins = [
  'https://app.example.com',
  'https://admin.example.com',
];

const corsOptions = {
  origin(origin, callback) {
    // `origin` is undefined for same-origin requests and non-browser clients (curl).
    if (!origin || allowedOrigins.includes(origin)) {
      return callback(null, true);     // allowed
    }
    return callback(new Error('Not allowed by CORS'));
  },
  credentials: true,                    // allow cookies / Authorization on cross-origin
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-CSRF-Token'],
  exposedHeaders: ['X-Total-Count'],    // headers the browser JS is allowed to READ
  maxAge: 600,                          // cache preflight 10 minutes
};

app.use(cors(corsOptions));
```

> WARNING — Pitfall: **`Access-Control-Allow-Origin: *` together with credentials is forbidden by the spec** — the browser will reject it. If you need cookies/`Authorization` on cross-origin requests (`credentials: true`), you **must** echo back a **specific** origin, not `*`. The allowlist-callback pattern above does this correctly. Also: never just reflect *any* incoming `Origin` blindly while allowing credentials — that's equivalent to `*` with credentials and exposes user data to any site.

> WARNING — Pitfall (Express v5 + preflight): In Express **v4** you often saw `app.options('*', cors(...))` to handle preflight. In Express **v5** the path-matching changed and a bare `'*'` string is no longer a valid path pattern (it throws). Using `app.use(cors(corsOptions))` handles preflight automatically and is the recommended approach across versions; if you need an explicit catch-all use a named wildcard like `app.options('/*splat', cors(corsOptions))` or a regex.

> BEST PRACTICE: Maintain an **explicit allowlist** of origins (from env config), enable `credentials` only if you actually use cross-origin cookies, set `maxAge` to reduce preflight chatter, and remember that CORS is **not** authorization — it gates browser *reads*, not API *access*.

---

## 10. Rate Limiting & Brute-Force Protection

Without rate limiting, an attacker can hammer your login endpoint with thousands of stolen credentials per second (**credential stuffing** / brute force), scrape your data, or simply DoS you. Rate limiting caps how many requests a client may make in a window; brute-force protection specifically throttles **repeated failed** auth attempts.

```bash
npm install express-rate-limit
```

### 10.1 A global limiter

```js
const rateLimit = require('express-rate-limit');

// Cap each IP to 100 requests / 15 minutes across the whole API.
const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  standardHeaders: 'draft-7', // emit RateLimit-* headers so clients can self-throttle
  legacyHeaders: false,       // drop the old X-RateLimit-* headers
  message: { error: 'Too many requests, please try again later.' },
});

app.use('/api', apiLimiter);
```

### 10.2 A stricter limiter for auth endpoints

The login route is the high-value target, so it gets a much tighter limit:

```js
// Only 5 login attempts / 15 min per IP. Don't count SUCCESSFUL logins.
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  skipSuccessfulRequests: true, // a correct login shouldn't burn the budget
  message: { error: 'Too many login attempts. Try again in 15 minutes.' },
});

app.post('/api/auth/login', loginLimiter, authController.login);
app.post('/api/auth/forgot-password', loginLimiter, authController.forgotPassword);
```

### 10.3 Slowing repeated attempts (`express-slow-down`)

Instead of a hard block, you can add **progressive delay** — each successive request gets slower, frustrating automated tooling while barely affecting humans:

```bash
npm install express-slow-down
```

```js
const slowDown = require('express-slow-down');

const speedLimiter = slowDown({
  windowMs: 15 * 60 * 1000,
  delayAfter: 3,                 // first 3 requests are fast
  delayMs: (hits) => hits * 500, // then add 500ms per extra hit (0.5s, 1s, 1.5s, ...)
});

app.post('/api/auth/login', speedLimiter, loginLimiter, authController.login);
```

### 10.4 Account-level lockout

IP limits are coarse (attackers rotate IPs; many users can share one NAT IP). For sensitive accounts, also track **failed attempts per account** and lock temporarily:

```js
// Conceptual: track failures on the user document.
async function recordFailedLogin(user) {
  user.failedLoginAttempts = (user.failedLoginAttempts || 0) + 1;
  if (user.failedLoginAttempts >= 5) {
    user.lockUntil = Date.now() + 15 * 60 * 1000; // lock 15 min
  }
  await user.save();
}

function isLocked(user) {
  return user.lockUntil && user.lockUntil > Date.now();
}
```

> WARNING — Pitfall: Behind a proxy/load balancer (Nginx, Heroku, AWS ALB, Cloudflare), `req.ip` is the *proxy's* IP unless you configure `app.set('trust proxy', ...)` correctly. If you don't, **every user shares one IP** and your rate limiter throttles everyone together — or, set too loosely, an attacker can **spoof `X-Forwarded-For`** to bypass limits. Set `trust proxy` to the *exact* number of trusted hops, not blanket `true`, in production.

> BEST PRACTICE: For multi-instance deployments, the default **in-memory** store means each server counts separately (so N servers = N× the limit). Use a **shared store** (`rate-limit-redis`) so limits are global. Combine IP-based limiting (cheap, broad) with account-based lockout (precise) and progressive delays. Always return **`429 Too Many Requests`** with a `Retry-After` header.

---

## 11. HTTP Parameter Pollution (HPP)

**HPP** happens when the same parameter appears **multiple times** in a query string or body, and the parser turns it into an **array** — which your code didn't expect.

```
GET /search?sort=price&sort=name
```

Express's `qs` parser turns this into `req.query.sort === ['price', 'name']` (an **array**, not a string). If your code does `sort.split(',')` or passes it to Mongoose expecting a string, you get crashes, bypassed validation, or unexpected query behavior. Attackers also use HPP to **smuggle** extra values past a filter that only inspects the first occurrence.

```js
// VULNERABLE shape: code assumes a string but receives an array.
app.get('/search', (req, res) => {
  const sort = req.query.sort;        // could be 'price' OR ['price', 'name']
  Model.find().sort(sort);            // breaks / behaves unexpectedly with an array
});
```

The `hpp` middleware collapses duplicated params to their **last** value (by default), restoring the expected scalar shape:

```bash
npm install hpp
```

```js
const hpp = require('hpp');

// Place AFTER the body parser so it can clean req.body/req.query.
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

app.use(
  hpp({
    // Some params SHOULD legitimately be arrays — whitelist them.
    whitelist: ['tags', 'category', 'duration'],
  }),
);
```

> BEST PRACTICE: Mount `hpp()` **after** body/query parsing, and **whitelist** the parameters that are genuinely meant to be repeated (filters, multi-select tags). The deeper fix is to **validate types** (Zod/Joi) so an unexpected array is rejected with a 400 rather than silently mishandled — HPP middleware and schema validation are complementary.

---

## 12. Secrets & Environment Management; HTTPS

### 12.1 Never commit secrets

Database URIs, JWT signing secrets, API keys, OAuth client secrets — these are credentials. Committing them to git is one of the most common real breaches (bots scan public *and* private GitHub for leaked AWS keys within seconds).

```bash
# .gitignore — make sure these never enter version control
.env
.env.local
.env.*.local
*.pem
*.key
```

Load secrets from the **environment** using `process.env`, never hard-coded:

```js
// config.js — centralize config; validate presence at startup (fail fast).
const required = ['MONGO_URI', 'JWT_SECRET', 'CSRF_SECRET'];
for (const key of required) {
  if (!process.env[key]) {
    throw new Error(`Missing required env var: ${key}`); // crash now, not at request time
  }
}

module.exports = {
  mongoUri: process.env.MONGO_URI,
  jwtSecret: process.env.JWT_SECRET,
  port: Number(process.env.PORT) || 3000,
  nodeEnv: process.env.NODE_ENV || 'development',
};
```

For local dev, Node 20.6+ has a **built-in** `--env-file` flag (no library needed), and `dotenv` remains the classic choice:

```bash
# Node 20.6+ built-in — no dependency:
node --env-file=.env server.js
```

```js
// Or the dotenv library (load it FIRST, before any code reads process.env):
require('dotenv').config();
```

> WARNING — Pitfall: If a secret was *ever* committed, deleting it in a later commit is **not enough** — it lives in git history forever. You must (1) **rotate/revoke the secret immediately** (assume it's compromised) and (2) purge history (`git filter-repo` / BFG) if needed. The secret is the thing that matters; the file is secondary.

> BEST PRACTICE: Use a **secrets manager** in production (AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault, Doppler) rather than plaintext `.env` files on servers. Use **different secrets per environment**, generate JWT/CSRF secrets with high entropy (`crypto.randomBytes(64).toString('hex')`), commit a non-secret **`.env.example`** documenting required keys, and run **`npm audit`** / Dependabot to catch vulnerable dependencies (A06). Use **`npm ci`** with a committed lockfile in CI for reproducible, integrity-checked installs (A08).

### 12.2 HTTPS in production

All the cookie flags and HSTS in the world are meaningless if traffic flows over plaintext HTTP — anyone on the network path can read tokens and inject content. **Always serve production traffic over HTTPS/TLS.**

In practice you usually **terminate TLS at a reverse proxy or platform** (Nginx, Caddy, a cloud load balancer, Cloudflare, Render/Railway/Heroku) rather than in Node directly, then forward plain HTTP to your app on a private network. When you do, configure Express to trust the proxy and redirect/enforce HTTPS:

```js
// Behind a TLS-terminating proxy:
app.set('trust proxy', 1); // trust the first proxy hop (so req.secure / req.ip are correct)

// Force HTTPS in production (the proxy sets X-Forwarded-Proto).
app.use((req, res, next) => {
  if (process.env.NODE_ENV === 'production' && req.headers['x-forwarded-proto'] !== 'https') {
    return res.redirect(301, `https://${req.headers.host}${req.url}`);
  }
  next();
});
```

Pair this with **HSTS** (via Helmet, §7) so browsers refuse to talk HTTP to your domain after the first visit. For local development with HTTPS, use **mkcert** to generate a locally-trusted cert. If you ever terminate TLS in Node directly:

```js
const https = require('https');
const fs = require('fs');

https
  .createServer(
    { key: fs.readFileSync('key.pem'), cert: fs.readFileSync('cert.pem') },
    app,
  )
  .listen(443);
```

---

## 13. Putting It All Together: A Hardened Express App

A reference middleware stack showing **order**, which is itself a security concern (e.g. Helmet and rate limiting before routes; error handler last):

```js
const express = require('express');
const helmet = require('helmet');
const cors = require('cors');
const rateLimit = require('express-rate-limit');
const mongoSanitize = require('express-mongo-sanitize');
const hpp = require('hpp');
const cookieParser = require('cookie-parser');

const app = express();

// 0. Trust the proxy (TLS termination, correct req.ip) — first.
app.set('trust proxy', 1);

// 1. Security headers — as early as possible, applies to every response.
app.use(helmet());

// 2. CORS — before routes; handles preflight automatically.
app.use(cors({ origin: process.env.ALLOWED_ORIGINS.split(','), credentials: true }));

// 3. Global rate limiting.
app.use('/api', rateLimit({ windowMs: 15 * 60 * 1000, max: 100 }));

// 4. Body parsing with size limits (prevents huge-payload DoS).
app.use(express.json({ limit: '10kb' }));
app.use(express.urlencoded({ extended: true, limit: '10kb' }));
app.use(cookieParser());

// 5. Sanitization AFTER parsing (operator-injection + parameter pollution).
app.use(mongoSanitize());
app.use(hpp({ whitelist: ['tags', 'sort'] }));

// 6. Routes (each applies protect → restrictTo/hasPermission → ownership).
app.use('/api/auth', require('./routes/auth'));
app.use('/api/articles', require('./routes/articles'));

// 7. 404 handler.
app.use((req, res) => res.status(404).json({ error: 'Not found' }));

// 8. Centralized error handler — LAST. Never leak stack traces in prod.
app.use((err, req, res, next) => {
  const status = err.statusCode || 500;
  const body = { error: err.isOperational ? err.message : 'Something went wrong' };
  if (process.env.NODE_ENV !== 'production') body.stack = err.stack; // dev only
  res.status(status).json(body);
});

module.exports = app;
```

---

## Common Pitfalls & Gotchas

- **Confusing 401 and 403.** 401 = not authenticated (who are you?). 403 = authenticated but not allowed (you can't do that). Returning the wrong one confuses clients and can leak information.
- **Authorization order bug.** Putting `restrictTo`/ownership checks **before** the authentication middleware that sets `req.user` — every request then fails (or, worse, `req.user` is undefined and your check silently passes if written carelessly). AuthN first, always.
- **Trusting `req.body.role` / `req.body.userId`.** The classic privilege-escalation vector. Always derive identity and role from the verified `req.user`, never from client-supplied fields. Whitelist updatable fields.
- **The IDOR / BOLA gap.** A role check (`restrictTo('user')`) does **not** verify *ownership*. Without a per-resource ownership check, any user can act on any object by changing the URL ID. This is the #1 real-world API vuln.
- **Thinking input filtering stops XSS.** Blacklists are bypassable. The real defense is **context-aware output escaping** (auto-escaping templates), plus DOMPurify for rich HTML, plus CSP as a backstop.
- **NoSQL injection via objects.** `express.json()` and the `qs` query parser produce objects; `{ "$ne": null }` in a field bypasses auth. Validate types; sanitize with `express-mongo-sanitize` (mind Express v5 `req.query` immutability).
- **Using deprecated security packages.** `csurf` (CSRF) and `xss-clean` (XSS) are deprecated/unmaintained — they fill tutorials but should not be used in new code. Prefer `csrf-csrf`/`csrf-sync` and output escaping + DOMPurify.
- **`Access-Control-Allow-Origin: *` with credentials.** Spec-forbidden and silently broken; never reflect arbitrary origins while allowing credentials. Use an explicit allowlist.
- **Believing CORS secures your API.** CORS is browser-enforced; `curl`/Postman/servers ignore it. It controls cross-origin browser *reads*, not API *access*. Authorization is your real gate.
- **Forgetting `trust proxy` behind a load balancer.** Breaks rate limiting (everyone shares the proxy IP) and `req.secure`/HTTPS detection. Set it to the exact number of trusted hops.
- **In-memory rate limiter across multiple instances.** Each instance counts separately, multiplying your effective limit. Use a shared Redis store.
- **Committing secrets — then "removing" them in a later commit.** They persist in git history. Rotate the secret immediately; purging history is secondary.
- **Leaking stack traces in production.** Verbose errors reveal structure, paths, and library versions to attackers (A05). Send generic messages in prod; log details server-side.
- **Forgetting body-size limits.** No `limit` on `express.json()` invites huge-payload memory-exhaustion DoS.

---

## Best Practices

- **Separate AuthN and AuthZ into distinct middleware** and run them in that order; keep authorization logic out of controllers where possible.
- **Authorize on permissions, not role names**, at the route; derive permissions from roles in one central map so business rules are a data change.
- **Always add ownership/ABAC checks** for resource-scoped operations; never rely on role checks alone for "own data."
- **Fail closed.** When in doubt (missing `req.user`, unknown role), deny. Default-deny beats default-allow.
- **Validate all input with a schema** (Zod/Joi) at the edge — this single discipline mitigates NoSQL injection, HPP, and many bad-data bugs at once.
- **Escape on output, sanitize rich HTML with DOMPurify, and set a strict CSP** — three layers, not one.
- **Use `SameSite=Lax/Strict` + `httpOnly` + `secure` cookies**; add CSRF tokens (`csrf-csrf`) for cookie-authenticated, state-changing routes.
- **Run Helmet near the top**, customize CSP for your app, and enable HSTS once you're fully on HTTPS.
- **Maintain an explicit CORS origin allowlist** from env config; only enable `credentials` if you truly need cross-origin cookies.
- **Rate-limit globally and lock down auth endpoints harder**; use a Redis store in multi-instance setups; combine IP limits with per-account lockout and progressive delay.
- **Mount `hpp()` after parsing** and whitelist legitimately-repeated params.
- **Keep secrets in the environment / a secrets manager**, never in code or git; use high-entropy secrets, rotate on suspicion, and commit a `.env.example`.
- **Enforce HTTPS in production** (TLS at the proxy + redirect + HSTS).
- **Keep dependencies patched**: `npm audit`, Dependabot, `npm ci` with a committed lockfile.
- **Log and monitor security events** (repeated 401/403/429) so you can detect attacks (A09). Practice **defense in depth** — assume any single layer can fail.

---

## Key Interview Points

- **What is the difference between authentication and authorization?** AuthN proves *who* you are (credentials → identity); AuthZ decides *what* you're allowed to do (roles/permissions/ownership). AuthN runs first and sets `req.user`; AuthZ reads it. Failures map to **401** and **403** respectively.
- **RBAC vs ABAC?** RBAC grants permissions to **roles** and assigns roles to users — simple, coarse, great for most apps. ABAC decides based on **attributes** of subject/resource/action/environment — handles "edit only your own posts," time/location rules; more expressive, more complex. Most apps use RBAC for broad gates plus ABAC-style ownership checks.
- **How do you implement `restrictTo("admin")`?** A **middleware factory** that returns `(req, res, next)`, checks `req.user.role` against the allowed list, calls `next()` or `next(new AppError(..., 403))`. Must run after authentication. Prefer permission checks (`hasPermission('article:delete')`) for maintainability.
- **What is IDOR / Broken Object Level Authorization and why is it #1?** When an endpoint uses a client-supplied ID without verifying the requester **owns/may access** that object, any user can act on others' data by changing the ID. Fix: load the resource and compare its owner to `req.user`. It's #1 because role checks pass while ownership is silently unchecked.
- **What is NoSQL injection and how do you prevent it?** Attacker sends **operator objects** (`{ "$ne": null }`, `{ "$gt": "" }`) where a string was expected, bypassing auth or extracting data; `$where` can even run arbitrary JS. Prevent with **type validation** (Zod/Joi), `express-mongo-sanitize`, casting to primitives, and disabling server-side JS.
- **Stored vs reflected XSS, and the real fix?** Stored = payload persisted in the DB and served to all viewers; reflected = bounced back in the immediate response. Real fix is **context-aware output escaping** (auto-escaping templates), **DOMPurify/sanitize-html** for rich HTML, and **CSP** as a backstop — not input blacklists.
- **What is CSRF and how do you defend against it?** A logged-in user's browser is tricked into sending a state-changing, **cookie-authenticated** request to your site. Defenses: **`SameSite` cookies** (first line), **CSRF tokens** (`csrf-csrf` double-submit / synchronizer). Header-token (Bearer JWT) APIs are largely immune to classic CSRF but exposed to XSS instead.
- **What does Helmet do, and name a few headers.** Sets security response headers: **CSP** (XSS backstop), **HSTS** (force HTTPS), **X-Content-Type-Options: nosniff**, **X-Frame-Options** (clickjacking), Referrer-Policy, and removes **X-Powered-By**. It hardens the browser-side perimeter; it doesn't secure server logic.
- **Explain CORS and the preflight request.** CORS **relaxes** the browser Same-Origin Policy to let specified origins read your responses. For non-simple requests (JSON content-type, PUT/DELETE, custom headers) the browser sends a preflight **`OPTIONS`** asking permission; the server replies with `Access-Control-Allow-*` headers. CORS is browser-enforced — it is **not** authorization.
- **Why is `Allow-Origin: *` with credentials invalid?** The spec forbids it; browsers reject it. With `credentials: true` you must echo a **specific** origin from an allowlist, never `*` or a blindly-reflected origin.
- **How do you prevent brute-force / credential stuffing?** **Rate limiting** (`express-rate-limit`) — strict on auth endpoints, `skipSuccessfulRequests`; **progressive delay** (`express-slow-down`); **per-account lockout**; CAPTCHA; and strong password hashing. Use a Redis store across instances; set `trust proxy` so limits key on the real client IP.
- **What is HTTP Parameter Pollution?** Duplicated params (`?sort=a&sort=b`) become **arrays**, breaking code that expects strings or smuggling values past filters. Mitigate with **`hpp`** (collapse to last value, whitelist real arrays) plus type validation.
- **How do you manage secrets?** Load from **environment** / a **secrets manager**, never hard-code; `.gitignore` the `.env`; high-entropy, per-environment secrets; **rotate** if leaked (history is permanent); validate required vars at startup; serve over **HTTPS** with HSTS.

---

## Practice Exercises

Work through these in order; each builds on the last. Try before reading the solutions.

1. **(Easy) Role middleware.** Write `restrictTo(...roles)` that allows only users whose `req.user.role` is in the list, returning a `403` (via `next(err)`) otherwise and `401` if `req.user` is missing. Add a route `DELETE /articles/:id` usable only by `admin`.

2. **(Easy–Medium) Permission middleware.** Given the `ROLE_PERMISSIONS` map from §3.2, write `hasPermission(...perms)` that requires the user to hold **all** listed permissions (derived from one or more roles plus explicit `user.permissions`). Protect `POST /articles` with `article:write`.

3. **(Medium) Fix the IDOR.** The route below lets any authenticated user edit any article. Add an ownership check (admins bypass), strip dangerous fields from the body, and return `404` (not `403`) when the article doesn't exist *or* isn't owned, to avoid ID enumeration.
   ```js
   router.patch('/:id', protect, async (req, res) => {
     const article = await Article.findByIdAndUpdate(req.params.id, req.body, { new: true });
     res.json(article);
   });
   ```

4. **(Medium) Defeat NoSQL injection.** This login is vulnerable to `{ "$ne": null }` in `password`. Make it safe **without** changing the response shape, using input validation. Explain in a comment why the attack no longer works.
   ```js
   app.post('/login', async (req, res) => {
     const user = await User.findOne({ email: req.body.email });
     if (!user || !(await user.verifyPassword(req.body.password))) {
       return res.status(401).json({ error: 'Invalid credentials' });
     }
     res.json({ token: signToken(user) });
   });
   ```

5. **(Medium–Hard) Brute-force-resistant login.** Add (a) an IP rate limiter of 5 attempts / 15 min that does **not** count successful logins, and (b) progressive delay after 3 attempts, to the login route. Then sketch per-account lockout after 5 consecutive failures.

6. **(Hard) Secure CORS + CSRF for a cookie-auth SPA.** Configure CORS to allow exactly `https://app.example.com` with credentials, and add `csrf-csrf` double-submit protection to all state-changing routes, plus a `GET /csrf-token` endpoint. Note which cookie attributes you set and why.

### Solutions

**Solution 1 — Role middleware.**

```js
class AppError extends Error {
  constructor(message, statusCode) { super(message); this.statusCode = statusCode; this.isOperational = true; }
}

function restrictTo(...roles) {
  return (req, res, next) => {
    if (!req.user) return next(new AppError('You are not logged in.', 401)); // not authenticated
    if (!roles.includes(req.user.role)) {
      return next(new AppError('You do not have permission to perform this action.', 403)); // authenticated, not allowed
    }
    next();
  };
}

// Only admins may delete. protect (authN) runs first and sets req.user.
router.delete('/articles/:id', protect, restrictTo('admin'), articleController.remove);
```

**Solution 2 — Permission middleware.**

```js
const ROLE_PERMISSIONS = {
  admin:  ['article:read', 'article:write', 'article:delete', 'user:manage'],
  editor: ['article:read', 'article:write'],
  user:   ['article:read'],
};

function permissionsFor(user) {
  const roles = user.roles || (user.role ? [user.role] : []);
  const fromRoles = roles.flatMap((r) => ROLE_PERMISSIONS[r] || []);
  return new Set([...fromRoles, ...(user.permissions || [])]); // roles + explicit grants
}

function hasPermission(...required) {
  return (req, res, next) => {
    if (!req.user) return next(new AppError('You are not logged in.', 401));
    const granted = permissionsFor(req.user);
    if (!required.every((p) => granted.has(p))) {        // require ALL listed perms
      return next(new AppError('Insufficient permissions.', 403));
    }
    next();
  };
}

router.post('/articles', protect, hasPermission('article:write'), articleController.create);
```

**Solution 3 — Fix the IDOR.**

```js
function pickAllowed(body, allowed) {
  return Object.fromEntries(Object.entries(body).filter(([k]) => allowed.includes(k)));
}

router.patch('/:id', protect, async (req, res, next) => {
  const article = await Article.findById(req.params.id);

  // Treat "not found" and "not owned" identically → can't enumerate IDs.
  const isOwner = article && String(article.authorId) === String(req.user._id);
  const isAdmin = req.user.role === 'admin';
  if (!article || (!isOwner && !isAdmin)) {
    return next(new AppError('Article not found.', 404));
  }

  // Never let the client set authorId/role/_id, etc. — whitelist updatable fields.
  const safe = pickAllowed(req.body, ['title', 'content', 'tags']);
  Object.assign(article, safe);
  await article.save(); // runs schema validators

  res.json(article);
});
```

**Solution 4 — Defeat NoSQL injection.**

```js
const { z } = require('zod');

// email and password MUST be strings — an object like { $ne: null } fails parsing.
const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(1).max(128),
});

app.post('/login', async (req, res) => {
  const parsed = loginSchema.safeParse(req.body);
  if (!parsed.success) {
    // Same 401 shape — we don't reveal that the input was malformed/an attack.
    return res.status(401).json({ error: 'Invalid credentials' });
  }
  const { email, password } = parsed.data; // guaranteed strings

  // Why the attack fails: `email`/`password` can no longer be operator objects.
  // findOne({ email }) gets a plain string, so { $ne: null } can never reach the query.
  const user = await User.findOne({ email });
  if (!user || !(await user.verifyPassword(password))) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }
  res.json({ token: signToken(user) });
});
```

**Solution 5 — Brute-force-resistant login.**

```js
const rateLimit = require('express-rate-limit');
const slowDown = require('express-slow-down');

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  skipSuccessfulRequests: true, // (a) successful logins don't burn the budget
  standardHeaders: 'draft-7',
  legacyHeaders: false,
  message: { error: 'Too many login attempts. Try again later.' },
});

const loginSlowDown = slowDown({
  windowMs: 15 * 60 * 1000,
  delayAfter: 3,                  // (b) first 3 are fast
  delayMs: (hits) => hits * 500,  // then +0.5s per extra attempt
});

// Order: slow down first (cheap), then hard-limit, then the controller.
app.post('/api/auth/login', loginSlowDown, loginLimiter, authController.login);

// Per-account lockout sketch (inside the controller):
async function login(req, res) {
  const user = await User.findOne({ email: String(req.body.email) }).select('+password +lockUntil +failedLoginAttempts');
  if (user?.lockUntil && user.lockUntil > Date.now()) {
    return res.status(429).json({ error: 'Account temporarily locked. Try again later.' });
  }
  const ok = user && (await user.verifyPassword(req.body.password));
  if (!ok) {
    if (user) {
      user.failedLoginAttempts = (user.failedLoginAttempts || 0) + 1;
      if (user.failedLoginAttempts >= 5) user.lockUntil = Date.now() + 15 * 60 * 1000;
      await user.save();
    }
    return res.status(401).json({ error: 'Invalid credentials' });
  }
  user.failedLoginAttempts = 0; user.lockUntil = undefined; // reset on success
  await user.save();
  res.json({ token: signToken(user) });
}
```

**Solution 6 — Secure CORS + CSRF for a cookie-auth SPA.**

```js
const express = require('express');
const cors = require('cors');
const cookieParser = require('cookie-parser');
const { doubleCsrf } = require('csrf-csrf');

const app = express();
app.set('trust proxy', 1);
app.use(express.json({ limit: '10kb' }));
app.use(cookieParser());

// CORS: exactly one origin, credentials enabled (so the auth + CSRF cookies flow).
app.use(cors({
  origin: 'https://app.example.com', // specific origin REQUIRED with credentials (never '*')
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'X-CSRF-Token'],
}));

const { doubleCsrfProtection, generateCsrfToken } = doubleCsrf({
  getSecret: () => process.env.CSRF_SECRET,
  getSessionIdentifier: (req) => req.user?.id ?? req.ip,
  cookieName: '__Host-psifi.x-csrf-token',
  // httpOnly: token cookie unreadable by JS; secure: HTTPS only; sameSite=lax: CSRF defense layer.
  cookieOptions: { httpOnly: true, secure: true, sameSite: 'lax' },
  getCsrfTokenFromRequest: (req) => req.headers['x-csrf-token'],
});

// SPA fetches a token once, then sends it as X-CSRF-Token on writes.
app.get('/csrf-token', (req, res) => res.json({ csrfToken: generateCsrfToken(req, res) }));

// Safe methods (GET/HEAD/OPTIONS) are exempt; protect state-changing routes.
app.post('/api/articles', doubleCsrfProtection, articleController.create);
app.patch('/api/articles/:id', doubleCsrfProtection, articleController.update);
app.delete('/api/articles/:id', doubleCsrfProtection, articleController.remove);

// Cookie attributes rationale:
//  - httpOnly  → JS can't read the cookie, blocking XSS-based theft
//  - secure    → only sent over HTTPS, so it can't leak over plaintext
//  - sameSite=lax → not sent on cross-site POST, an independent CSRF layer
//  - __Host-   → prefix forces secure + path=/ + no Domain, hardening the cookie
```

---

## Summary / Cheat Sheet

| Topic | Control / Package | One-line takeaway |
| --- | --- | --- |
| AuthN vs AuthZ | (your middleware) | AuthN = who you are (401); AuthZ = what you can do (403). AuthN first. |
| RBAC | `restrictTo`, `hasPermission` | Permissions via roles; check permissions, not role names. |
| ABAC / ownership | `restrictToOwner` | Compare resource owner to `req.user`; fixes IDOR/BOLA (the #1 API vuln). |
| Privilege escalation | field whitelisting | Never trust `req.body.role`/`userId`; derive identity from `req.user`. |
| OWASP Top 10 | (the whole module) | A01 Broken Access Control is #1; map each risk to a control. |
| NoSQL injection | `express-mongo-sanitize` + Zod/Joi | Block operator objects (`$ne`, `$gt`, `$where`); validate types; cast to primitives. |
| Command injection | `execFile`/`spawn` | Pass args as an array; never interpolate input into a shell string. |
| XSS | output escaping + DOMPurify + CSP | Escape on output (not blacklist input); sanitize rich HTML with an allowlist. |
| CSRF | `SameSite` cookies + `csrf-csrf` | Only affects cookie auth; `SameSite=Lax` + tokens. (`csurf` is deprecated.) |
| Security headers | `helmet` | CSP, HSTS, nosniff, frameguard; mount near the top; customize CSP. |
| CORS | `cors` | Relaxes SOP for browser reads; allowlist origins; `*` + credentials is illegal; not authorization. |
| Preflight | `OPTIONS` | JSON/PUT/DELETE/custom headers trigger it; server grants via `Access-Control-Allow-*`. |
| Rate limiting | `express-rate-limit`, `express-slow-down` | Strict on `/login`; `skipSuccessfulRequests`; Redis store across instances; set `trust proxy`. |
| Account lockout | (per-user counter) | Lock after N failures; complements IP limits. |
| HPP | `hpp` | Duplicate params become arrays; collapse to last; whitelist real arrays. |
| Secrets | env / secrets manager | Never commit; `.gitignore .env`; rotate if leaked; validate at startup. |
| HTTPS | TLS at proxy + HSTS + redirect | No security control works over plaintext HTTP. |
| Body limits | `express.json({ limit })` | Cap payload size to prevent memory-exhaustion DoS. |
| Errors | central error handler | Generic message in prod; never leak stack traces. |
| Dependencies | `npm audit`, `npm ci`, Dependabot | Patch known-CVE packages (A06); reproducible installs (A08). |

**The hardened-stack order to memorize:** `trust proxy` → `helmet` → `cors` → `rate-limit` → body parsers (with limits) → `cookie-parser` → `mongo-sanitize` → `hpp` → routes (`protect` → `restrictTo`/`hasPermission` → ownership) → 404 → central error handler.

> Final principle — **Defense in depth**: no single control is sufficient. Validation, sanitization, escaping, headers, rate limits, least-privilege authorization, and HTTPS each catch what the others miss. Assume any one layer can fail, and make sure the next one still holds.
