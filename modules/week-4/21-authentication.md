# Authentication (Sessions & JWT)

Authentication is the moment a web application stops trusting *anyone* who shows up and starts asking *"who are you, and can you prove it?"*. Almost every non-trivial backend you will ever build needs it, and it is one of the most heavily probed topics in backend interviews — partly because it sits at the intersection of HTTP, cryptography, databases, and security, and partly because it is so easy to get subtly, dangerously wrong.

This module teaches authentication from first principles for a developer who knows JavaScript but is new to the server side. We will start with the conceptual bedrock (authentication vs authorization, why you must never store plaintext passwords), then build up two complete, production-shaped systems: **session-based auth** (stateful, cookie + server store) and **token-based auth with JWT** (stateless, signed tokens). Along the way we cover cookies in depth, bcrypt password hashing, access/refresh token flows, route-protecting middleware, secure client-side token storage, logout, password reset by email, and finally an overview of Passport.js and OAuth social login.

By the end you should be able to design an auth system, defend each decision in an interview, and spot the security pitfalls that trip up most developers.

> **Version note:** Examples target **Node.js current LTS**, **Express 5** (the current major), **Mongoose 8**, and plain JavaScript with CommonJS `require`. Express 4 vs 5 differences are called out where they matter (they are minor for auth). The libraries used — `bcrypt`, `express-session`, `connect-mongo`, `jsonwebtoken`, `cookie-parser`, `passport` — are all current and actively maintained.

## What You'll Learn

By the end of this module you will be able to:

- Precisely define **authentication vs authorization** and explain where each lives in a request lifecycle.
- Hash passwords correctly with **bcrypt** — understanding salts, salt rounds, the cost factor, and why plaintext (or fast hashes like MD5/SHA-256) are unacceptable.
- Implement **session-based authentication** with `express-session`, understand how server sessions and cookies cooperate, and use a persistent **session store** (`connect-mongo`).
- Master **cookies**: `httpOnly`, `secure`, `sameSite`, `maxAge`/`expires`, `domain`, `path`, and cookie **signing**.
- Explain the anatomy of a **JWT** (`header.payload.signature`), how signing and verification work, and when to choose **symmetric (HS256)** vs **asymmetric (RS256)** algorithms.
- Use the **`jsonwebtoken`** library correctly: `sign`, `verify`, expiry, and common option pitfalls.
- Design an **access token + refresh token** flow, including rotation and revocation.
- Build a complete **register / login** flow backed by a **Mongoose `User` model**.
- Write **auth middleware** that verifies a token and attaches `req.user` to protect routes.
- Decide **where to store tokens on the client** (cookie vs `localStorage`) and articulate the XSS/CSRF tradeoffs.
- Implement **logout** and a secure **password-reset-by-email** flow.
- Understand what **Passport.js** is for and how **OAuth 2.0 / social login** works at a high level.

---

## 1. Authentication vs Authorization

These two words are used interchangeably in casual speech, but in security they mean different things, and conflating them is a classic interview red flag.

- **Authentication (AuthN)** — *"Who are you?"* The process of verifying an identity. Logging in with email + password, presenting a valid token, or signing in with Google are all authentication.
- **Authorization (AuthZ)** — *"What are you allowed to do?"* The process of deciding whether an *already-authenticated* identity may perform a specific action (read this document, delete that user, access the admin panel).

The order is fixed: **you authenticate first, then authorize.** You cannot decide what someone is allowed to do until you know who they are.

| Aspect | Authentication (AuthN) | Authorization (AuthZ) |
| --- | --- | --- |
| Question answered | Who are you? | What can you do? |
| Happens | First | After authentication |
| Typical mechanism | Password check, token verification, OAuth | Roles, permissions, ownership checks, scopes |
| Typical HTTP failure code | **401 Unauthorized** (you are not logged in / token invalid) | **403 Forbidden** (you are logged in, but not allowed) |
| Data involved | Credentials, sessions, tokens | Roles, ACLs, policies, resource ownership |

> WARNING — Pitfall: The HTTP status code names are historically misleading. **401 "Unauthorized" actually means *unauthenticated*** (we don't know who you are). **403 "Forbidden" means *not authorized*** (we know who you are, you just can't do this). Interviewers love this one.

A tiny illustration of the two concepts living in the same request:

```js
// Conceptual middleware chain — authN then authZ
app.delete(
  "/posts/:id",
  requireAuth,         // AuthN: verifies token, sets req.user, else 401
  requireRole("admin"),// AuthZ: checks req.user.role === 'admin', else 403
  deletePostHandler    // The actual work, only reached if both pass
);
```

This module focuses primarily on **authentication**. Authorization (roles, permissions, RBAC) is a large topic of its own, but we will show the basic ownership/role checks because they naturally ride on top of `req.user`.

---

## 2. Password Hashing with bcrypt

Before we can authenticate users, we must store their passwords — and storing passwords is where developers most often create catastrophic, headline-making vulnerabilities. The rule is absolute:

> BEST PRACTICE: **Never store passwords in plaintext. Never store them reversibly encrypted. Always store a slow, salted *hash*.**

### 2.1 Why not plaintext?

If your database is leaked (and breaches happen constantly), plaintext passwords mean every account is instantly compromised — and worse, because people reuse passwords, you have handed attackers the keys to your users' email, banking, and everything else. Storing plaintext is negligence, not a bug.

### 2.2 Hashing vs encryption

- **Encryption** is *reversible*: with the key, ciphertext goes back to plaintext. Wrong tool for passwords — if you can reverse it, so can an attacker who steals the key.
- **Hashing** is *one-way*: a hash function maps input to a fixed-size digest, and there is no practical way to invert it. To check a password, you hash the submitted password and compare digests. You never need to recover the original.

### 2.3 Why not a fast hash like MD5 or SHA-256?

General-purpose hashes (MD5, SHA-1, SHA-256) are designed to be *fast* — billions of hashes per second on a GPU. That speed is a disaster for passwords: attackers can brute-force or use precomputed **rainbow tables** to crack stolen hashes at enormous rates. Password hashing needs the opposite: a function that is *deliberately slow* and *memory/CPU costly*, so each guess is expensive.

That is exactly what **bcrypt** (and Argon2, scrypt, PBKDF2) provides.

### 2.4 What a salt is, and why it matters

A **salt** is a random value mixed into the password before hashing, unique per password. Salts defeat two attacks:

1. **Rainbow tables** — precomputed tables of hash→password mappings are useless when every password has a unique random salt, because the attacker would need a separate table per salt.
2. **Identical-password detection** — without a salt, two users with the password `hunter2` produce the *same* hash, revealing that they share a password. With unique salts, the hashes differ.

A wonderful property of bcrypt is that **the salt is stored inside the output hash itself** — you do not manage salts separately.

### 2.5 The cost factor (salt rounds)

bcrypt takes a **cost factor** (often called "salt rounds"), e.g. `12`. This is a *logarithmic* work factor: the number of internal iterations is `2^cost`. So cost `12` = `2^12` = 4096 iterations; cost `13` doubles the work; cost `14` doubles it again. You tune it so hashing takes a noticeable but acceptable amount of time on your hardware (a common target is ~100–250 ms per hash). As hardware gets faster, you raise the cost.

| Cost (rounds) | Internal iterations | Relative time |
| --- | --- | --- |
| 10 | 1,024 | baseline (fast, low-security default) |
| 12 | 4,096 | ~4x slower than 10 — common modern default |
| 14 | 16,384 | ~16x slower — high security, noticeable latency |

> TIP: **10–12 is a sensible default for most apps in 2026.** Benchmark on *your* production hardware. If a login takes too long under load, that is a sign your cost may be too high (or you need more workers), not a reason to drop below ~10.

### 2.6 Anatomy of a bcrypt hash

A bcrypt hash is a self-describing string:

```
$2b$12$eImiTXuWVxfM37uY4JANjQ==.../LW0r3.uRq3o0K2g8nFqYwS2eGm
└┬┘ └┬┘ └────────────┬────────────┘└──────────────┬─────────────┘
 │   │               │                             │
 │   │           22-char salt              31-char hash digest
 │   └─ cost factor (12)
 └─ algorithm version ($2b$ is the modern variant)
```

Because the algorithm version, cost, and salt all live in the string, `bcrypt.compare()` can re-derive everything it needs to verify a password — you only ever store this one string.

### 2.7 Installing and using bcrypt

There are two popular packages:

- **`bcrypt`** — native C++ bindings, fast, requires compilation on install.
- **`bcryptjs`** — pure JavaScript, no compilation, slightly slower, easier to install everywhere (serverless, Windows, etc.).

They are API-compatible. We use `bcrypt` here; swap the import for `bcryptjs` if native builds are a problem.

```bash
npm install bcrypt
# or, the pure-JS alternative:
# npm install bcryptjs
```

**Hashing a password (async — preferred):**

```js
// hash.js
const bcrypt = require("bcrypt");

async function hashPassword(plainPassword) {
  const SALT_ROUNDS = 12; // cost factor; tune to your hardware
  // bcrypt.hash generates a random salt AND hashes in one step.
  // The returned string already embeds the salt + cost.
  const hash = await bcrypt.hash(plainPassword, SALT_ROUNDS);
  return hash;
}

(async () => {
  const hash = await hashPassword("S3cret-Pa$$w0rd");
  console.log(hash);
  // -> $2b$12$C6UzMDM.H6dfI/f/IKcEeO... (different every time, even for same input)
})();
```

**Comparing a password at login:**

```js
// compare.js
const bcrypt = require("bcrypt");

async function verifyPassword(plainPassword, storedHash) {
  // compare() extracts the salt + cost from storedHash, re-hashes the
  // submitted password the same way, and does a constant-time comparison.
  return bcrypt.compare(plainPassword, storedHash);
}

(async () => {
  const storedHash = await bcrypt.hash("S3cret-Pa$$w0rd", 12);

  console.log(await verifyPassword("S3cret-Pa$$w0rd", storedHash)); // true
  console.log(await verifyPassword("wrong-guess", storedHash));     // false
})();
```

> WARNING — Pitfall: **Never compare hashes with `===`.** Use `bcrypt.compare()`. It performs a *constant-time* comparison that does not leak information through timing, and it knows how to parse bcrypt's embedded salt/cost. A naive `hash(input) === storedHash` would also fail anyway, because each `hash()` call produces a different salt.

> WARNING — Pitfall: bcrypt **silently truncates input at 72 bytes**. Extremely long passwords or passphrases beyond 72 bytes ignore the remainder. For most apps this is fine; if you must support arbitrarily long inputs, pre-hash with SHA-256 and base64-encode before bcrypt, or use Argon2 instead.

> TIP: Always use the **async** `bcrypt.hash`/`bcrypt.compare`. The sync versions (`hashSync`, `compareSync`) block Node's single event loop for the entire (intentionally slow) computation, which can stall every other request under load.

---

## 3. The Two Models of Authentication

Once a user proves their identity (correct password), the server must *remember* that fact across subsequent requests, because HTTP is **stateless** — each request is independent and carries no memory of the last. There are two dominant strategies for carrying that "I'm logged in" proof:

| | **Session-based (stateful)** | **Token-based / JWT (stateless)** |
| --- | --- | --- |
| Where state lives | On the **server** (session store), client holds only a session ID | In the **token itself** (signed, self-contained), server stores nothing |
| What the client sends | A **cookie** with an opaque session ID | A signed **JWT** (cookie or `Authorization` header) |
| Revoke a single login | Easy — delete the session row | Hard — token is valid until it expires (need a denylist) |
| Horizontal scaling | Needs shared session store (Redis/Mongo) | Trivial — any server can verify with the key |
| Server memory/lookup | DB/store lookup per request | Just a signature check (no lookup needed) |
| Best fit | Classic web apps, server-rendered sites | APIs, SPAs, mobile, microservices |

Neither is universally "better." Many real systems combine them (e.g., short-lived JWT access tokens + a server-tracked refresh token, which is essentially a session). We will build both.

---

## 4. Cookies in Depth

Both models lean on cookies, so let's understand them properly before going further. A **cookie** is a small piece of data the server asks the browser to store and **automatically send back** on every subsequent request to that origin. The server sets a cookie with the `Set-Cookie` response header; the browser returns it in the `Cookie` request header.

```
# Server response:
Set-Cookie: sessionId=abc123; HttpOnly; Secure; SameSite=Lax; Max-Age=3600; Path=/

# Browser then sends, automatically, on later requests:
Cookie: sessionId=abc123
```

### 4.1 The critical cookie attributes

| Attribute | What it does | Why it matters for security |
| --- | --- | --- |
| `HttpOnly` | Cookie is **not** readable by JavaScript (`document.cookie`) | Defends against **XSS** stealing the cookie. Essential for auth cookies. |
| `Secure` | Cookie only sent over **HTTPS** | Prevents interception over plaintext HTTP. Required in production. |
| `SameSite` | Controls cross-site sending: `Strict`, `Lax`, or `None` | Primary defense against **CSRF**. |
| `Max-Age` | Lifetime in **seconds** (relative) | Controls expiry; absent = session cookie (dies when browser closes). |
| `Expires` | Absolute expiry date | Older form of `Max-Age`; `Max-Age` wins if both present. |
| `Domain` | Which hosts receive the cookie | Scope carefully; broad domains leak cookies to subdomains. |
| `Path` | URL path prefix the cookie applies to | Usually `/`. |

**`SameSite` values explained:**

- **`Strict`** — the cookie is *never* sent on cross-site requests, even when a user clicks a link from another site. Maximum CSRF protection, but can be inconvenient (e.g., following a link to your site shows you logged out until you navigate).
- **`Lax`** (the modern browser default) — the cookie is sent on top-level navigations (clicking a link) but *not* on cross-site sub-requests (like an `<img>` or background `fetch` from another origin). A good default for auth cookies.
- **`None`** — sent on all cross-site requests; **requires `Secure`**. Needed when you genuinely have cross-origin frontends/backends, but pair it with CSRF tokens.

> WARNING — Pitfall: `Max-Age` is in **seconds** in the raw HTTP header, but Express's `res.cookie()` `maxAge` option is in **milliseconds**. Mixing these up is a constant source of cookies that expire instantly or never.

### 4.2 Reading and writing cookies in Express

To read incoming cookies easily, install `cookie-parser`:

```bash
npm install cookie-parser
```

```js
// cookies-demo.js
const express = require("express");
const cookieParser = require("cookie-parser");

const app = express();

// The argument is a SECRET used to SIGN cookies (see below).
app.use(cookieParser("a-long-random-cookie-signing-secret"));

app.get("/set", (req, res) => {
  // res.cookie(name, value, options). maxAge is in MILLISECONDS here.
  res.cookie("theme", "dark", {
    httpOnly: true,                 // not readable by JS
    secure: process.env.NODE_ENV === "production", // HTTPS-only in prod
    sameSite: "lax",                // CSRF mitigation
    maxAge: 1000 * 60 * 60 * 24,    // 1 day, in milliseconds
    path: "/",
  });

  // A SIGNED cookie: tamper-evident (see 4.3)
  res.cookie("sid", "session-abc123", { httpOnly: true, signed: true });

  res.send("Cookies set");
});

app.get("/read", (req, res) => {
  res.json({
    unsignedCookies: req.cookies,        // populated by cookie-parser
    signedCookies: req.signedCookies,    // verified signed cookies
  });
});

app.get("/clear", (req, res) => {
  res.clearCookie("theme");
  res.send("Cleared");
});

app.listen(3000, () => console.log("http://localhost:3000/set"));
```

### 4.3 Cookie signing (what it is, and what it is NOT)

When you pass a secret to `cookieParser(secret)` and set a cookie with `{ signed: true }`, Express appends an **HMAC signature** of the value. On the way back in, `cookie-parser` verifies the signature and exposes the value via `req.signedCookies` only if it is intact.

- **Signing detects tampering.** If the client edits the cookie value, the signature no longer matches and the cookie is rejected. This lets you trust that a value you set was not modified.
- **Signing does NOT encrypt.** The value is still readable (it's base64-ish, not secret). Do not put sensitive data in a cookie value and assume signing hides it.

> TIP: For session-based auth you typically don't sign the session cookie manually — `express-session` handles signing the session ID for you using its own secret.

---

## 5. Session-Based Authentication

In session-based auth, when a user logs in the server creates a **session** — a record of "this person is authenticated" stored *server-side* — and hands the client a cookie containing only an opaque **session ID**. On each subsequent request, the browser sends the cookie, the server looks up the session, and knows who the user is.

### 5.1 How it works, step by step

1. User POSTs credentials to `/login`. Server verifies the password with bcrypt.
2. Server creates a session: generates a random session ID, stores `{ sessionId -> { userId, ... } }` in a **session store**.
3. Server responds with `Set-Cookie: connect.sid=<signed session id>; HttpOnly; ...`.
4. Browser stores the cookie and **automatically** includes it on every future request.
5. On each request, middleware reads the cookie, looks up the session in the store, and populates `req.session`.
6. On logout, the server destroys the session and clears the cookie.

The crucial point: **the cookie holds only an ID, not the user data.** All the trusted state lives on the server. The client can't forge a session because session IDs are large random values and the cookie is signed.

### 5.2 Where sessions are stored: the session store

By default, `express-session` uses an **in-memory store**, which is fine for local development but **unsuitable for production**: it leaks memory, it is wiped on every server restart (logging everyone out), and it does not work across multiple server instances (each process has its own memory). For production you need a **shared, persistent store**. Common choices: **Redis** (`connect-redis`) and **MongoDB** (`connect-mongo`). We'll use `connect-mongo` to keep the stack consistent.

```bash
npm install express-session connect-mongo
```

### 5.3 A complete session-auth setup with connect-mongo

```js
// session-app.js
const express = require("express");
const session = require("express-session");
const MongoStore = require("connect-mongo");
const bcrypt = require("bcrypt");
const mongoose = require("mongoose");

const app = express();
app.use(express.json());

const MONGO_URI = process.env.MONGO_URI || "mongodb://127.0.0.1:27017/authdemo";

// --- Session middleware ---
app.use(
  session({
    name: "sid",                       // cookie name (default is connect.sid)
    secret: process.env.SESSION_SECRET || "change-me-to-a-long-random-string",
    resave: false,                     // don't save unchanged sessions
    saveUninitialized: false,          // don't create sessions for anonymous users (GDPR-friendly)
    store: MongoStore.create({
      mongoUrl: MONGO_URI,
      ttl: 60 * 60 * 24 * 7,           // session TTL in SECONDS (7 days) — store-side expiry
      touchAfter: 24 * 3600,           // only update the session in DB once per 24h if unchanged (perf)
    }),
    cookie: {
      httpOnly: true,                  // JS cannot read it
      secure: process.env.NODE_ENV === "production", // HTTPS only in prod
      sameSite: "lax",                 // CSRF mitigation
      maxAge: 1000 * 60 * 60 * 24 * 7, // cookie lifetime in MILLISECONDS (7 days)
    },
  })
);

// (User model defined in section 8; assume `User` is available here.)
const User = require("./models/User");

// --- Login: verify password, then establish a session ---
app.post("/login", async (req, res) => {
  const { email, password } = req.body;
  const user = await User.findOne({ email }).select("+passwordHash");
  if (!user || !(await bcrypt.compare(password, user.passwordHash))) {
    return res.status(401).json({ error: "Invalid credentials" });
  }
  // Store the minimal identity in the session. NEVER store the password hash.
  req.session.userId = user._id.toString();
  req.session.role = user.role;
  res.json({ message: "Logged in", user: { id: user._id, email: user.email } });
});

// --- Auth guard middleware for sessions ---
function requireSession(req, res, next) {
  if (!req.session.userId) {
    return res.status(401).json({ error: "Not authenticated" });
  }
  next();
}

// --- A protected route ---
app.get("/me", requireSession, async (req, res) => {
  const user = await User.findById(req.session.userId);
  res.json({ user });
});

// --- Logout: destroy the session and clear the cookie ---
app.post("/logout", (req, res) => {
  req.session.destroy((err) => {
    if (err) return res.status(500).json({ error: "Could not log out" });
    res.clearCookie("sid"); // must match the `name` above
    res.json({ message: "Logged out" });
  });
});

mongoose.connect(MONGO_URI).then(() => {
  app.listen(3000, () => console.log("Session app on http://localhost:3000"));
});
```

> BEST PRACTICE: Set `saveUninitialized: false` and `resave: false`. The former avoids creating empty sessions (and cookies) for visitors who never log in; the latter avoids needless writes to the store on every request. These are the recommended defaults in the `express-session` docs.

> WARNING — Pitfall: To **prevent session fixation**, regenerate the session ID at the privilege boundary (i.e., right after login). Call `req.session.regenerate(cb)` before setting `req.session.userId`, so an attacker who planted a known session ID cannot ride it into an authenticated session.

```js
// Session fixation–safe login fragment
app.post("/login", async (req, res) => {
  const { email, password } = req.body;
  const user = await User.findOne({ email }).select("+passwordHash");
  if (!user || !(await bcrypt.compare(password, user.passwordHash))) {
    return res.status(401).json({ error: "Invalid credentials" });
  }
  // Regenerate to get a fresh session ID before we authenticate it.
  req.session.regenerate((err) => {
    if (err) return res.status(500).json({ error: "Session error" });
    req.session.userId = user._id.toString();
    req.session.save(() => res.json({ message: "Logged in" }));
  });
});
```

---

## 6. JWT Deep Dive

A **JSON Web Token (JWT)** is a compact, URL-safe, **self-contained** token that carries claims about a user, *cryptographically signed* so the recipient can verify it was not tampered with. The big idea: instead of storing session state on the server, you put the (non-secret) identity data *inside the token* and sign it. Any server holding the verification key can validate the token without a database lookup — that's what makes JWT auth **stateless**.

### 6.1 Structure: `header.payload.signature`

A JWT is three Base64URL-encoded parts joined by dots:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI2NGYiLCJyb2xlIjoiYWRtaW4iLCJpYXQiOjE3MTg3ODQwMDAsImV4cCI6MTcxODc4NzYwMH0.K3Jq8nJ0v...signature
└──────────── header ───────────┘ └──────────────── payload ─────────────────┘ └──── signature ────┘
```

**1. Header** — describes the algorithm and token type:

```json
{
  "alg": "HS256",   // signing algorithm
  "typ": "JWT"
}
```

**2. Payload** — the **claims** (data). There are *registered* claims with standard meanings and your own *custom* claims:

```json
{
  "sub": "64f0c2...e9",   // subject — typically the user id (registered claim)
  "role": "admin",         // custom claim
  "iat": 1718784000,       // issued-at (seconds since epoch, registered)
  "exp": 1718787600        // expiry (registered) — token invalid after this
}
```

Common **registered claims**: `iss` (issuer), `sub` (subject), `aud` (audience), `exp` (expiration), `nbf` (not-before), `iat` (issued-at), `jti` (JWT ID — useful for revocation).

**3. Signature** — computed over the encoded header and payload using a secret/key:

```
signature = HMAC-SHA256( base64url(header) + "." + base64url(payload), secret )
```

> WARNING — Pitfall: **The payload is encoded, NOT encrypted.** Anyone can Base64URL-decode it and read every claim. **Never put secrets (passwords, card numbers, anything sensitive) in a JWT payload.** Signing proves integrity, not confidentiality. If you need confidentiality, use JWE (encrypted JWT) or simply don't put the secret in the token.

### 6.2 How signing and verification work

- **Signing** (at login): the server takes the header + payload, computes the signature with its secret/private key, and produces the token. Only a party with the secret/private key can produce a valid signature.
- **Verification** (on each request): the server recomputes the signature from the received header + payload and checks it matches the token's signature. If a single byte of the payload was altered, the signature won't match and the token is rejected. The server also checks `exp` (not expired), and optionally `iss`/`aud`.

Because verification only needs the key and some math — no database — any number of servers can independently verify tokens. That is the scaling superpower of JWTs.

### 6.3 Symmetric vs Asymmetric signing

| | **Symmetric (HS256/384/512)** | **Asymmetric (RS256, ES256)** |
| --- | --- | --- |
| Keys | One **shared secret** signs and verifies | **Private key** signs, **public key** verifies |
| Who can verify | Anyone who knows the secret | Anyone with the public key (safe to distribute) |
| Who can sign | Anyone who knows the secret | Only the holder of the private key |
| Best for | Single app / monolith that both issues and verifies | Multiple services, third parties verifying, public APIs |
| Risk | Sharing the secret with verifiers lets them *also forge* tokens | Public key can't forge; private key stays with the issuer |

- Use **HS256** when the same trusted service both issues and verifies tokens (most monoliths). It's simpler and fast.
- Use **RS256/ES256** when you have a central auth service that *issues* tokens and many other services (or third parties) that only need to *verify* them. You distribute the public key freely; they can verify but cannot mint tokens.

> WARNING — Pitfall: The infamous **`alg: none` attack**: early JWT libraries would accept a token whose header said `"alg": "none"` and skip signature verification entirely. A second class of attack swaps an RS256 token for an HS256 one and uses the *public key* as the HMAC secret. Modern `jsonwebtoken` defends against these *if you specify the allowed algorithm explicitly during verify* — always pass `{ algorithms: ["HS256"] }` (or your chosen list).

---

## 7. Using the `jsonwebtoken` Library

```bash
npm install jsonwebtoken
```

### 7.1 Signing a token

```js
// jwt-sign.js
const jwt = require("jsonwebtoken");

const ACCESS_SECRET = process.env.JWT_ACCESS_SECRET || "dev-access-secret-change-me";

function signAccessToken(user) {
  // First arg: payload (claims). Keep it small and non-sensitive.
  // Second arg: secret (HS256) or private key (RS256).
  // Third arg: options.
  return jwt.sign(
    { role: user.role },          // custom claims
    ACCESS_SECRET,
    {
      subject: String(user._id),  // sets the `sub` claim (the user id)
      expiresIn: "15m",           // sets `exp` — accepts "15m", "2h", "7d", or seconds
      issuer: "myapp.com",        // sets `iss`
      audience: "myapp-clients",  // sets `aud`
    }
  );
}

const token = signAccessToken({ _id: "64f0c2e9", role: "admin" });
console.log(token);
```

> WARNING — Pitfall: Do not also put `exp`, `iat`, `sub`, `iss`, or `aud` *manually inside the payload object* when you're using the corresponding options (`expiresIn`, `subject`, `issuer`, `audience`). `jsonwebtoken` throws "Bad options.expiresIn ... cannot both be present" if you set `exp` in the payload AND pass `expiresIn`. Use the options for registered claims; use the payload only for custom claims.

### 7.2 Verifying a token

```js
// jwt-verify.js
const jwt = require("jsonwebtoken");
const ACCESS_SECRET = process.env.JWT_ACCESS_SECRET || "dev-access-secret-change-me";

function verifyAccessToken(token) {
  // verify() checks the signature AND expiry (and iss/aud if you pass them).
  // ALWAYS pin the algorithm to prevent alg-confusion attacks.
  return jwt.verify(token, ACCESS_SECRET, {
    algorithms: ["HS256"],
    issuer: "myapp.com",
    audience: "myapp-clients",
  });
}

try {
  const decoded = verifyAccessToken(someToken);
  console.log(decoded.sub, decoded.role); // user id + role
} catch (err) {
  // jwt throws on bad signature, expiry, wrong iss/aud, etc.
  if (err.name === "TokenExpiredError") {
    // token expired — client should refresh
  } else if (err.name === "JsonWebTokenError") {
    // malformed / bad signature
  }
}
```

`jwt.verify` throws specific error types you should handle distinctly:

- **`TokenExpiredError`** — `exp` has passed (`err.expiredAt` tells you when). Treat as "please refresh / re-login," typically 401.
- **`JsonWebTokenError`** — invalid signature, malformed token, wrong `iss`/`aud`, etc. Treat as 401.
- **`NotBeforeError`** — token's `nbf` is in the future.

> TIP: `jwt.decode(token)` reads the payload **without verifying the signature.** It is *only* for inspection/debugging — never trust `decode()` output for auth decisions. Always use `verify()`.

### 7.3 Asymmetric (RS256) example

```js
// rs256.js — sign with a private key, verify with the public key
const jwt = require("jsonwebtoken");
const fs = require("fs");

// Generate keys once with:
//   openssl genpkey -algorithm RSA -out private.pem -pkeyopt rsa_keygen_bits:2048
//   openssl rsa -in private.pem -pubout -out public.pem
const privateKey = fs.readFileSync("private.pem");
const publicKey = fs.readFileSync("public.pem");

const token = jwt.sign({ role: "admin" }, privateKey, {
  algorithm: "RS256",       // note: ALGORITHM (singular) on sign
  subject: "64f0c2e9",
  expiresIn: "15m",
});

const decoded = jwt.verify(token, publicKey, { algorithms: ["RS256"] });
console.log(decoded);
```

---

## 8. The Mongoose User Model

Everything above needs a place to persist users. Here is a production-shaped `User` model that hashes passwords automatically and never leaks the hash.

```js
// models/User.js
const mongoose = require("mongoose");
const bcrypt = require("bcrypt");
const crypto = require("crypto");

const SALT_ROUNDS = 12;

const userSchema = new mongoose.Schema(
  {
    email: {
      type: String,
      required: true,
      unique: true,        // creates a unique index (case sensitivity: see note)
      lowercase: true,     // normalize before saving
      trim: true,
      index: true,
    },
    name: { type: String, required: true, trim: true },
    passwordHash: {
      type: String,
      required: true,
      select: false,       // EXCLUDED from query results by default — must opt in
    },
    role: { type: String, enum: ["user", "admin"], default: "user" },

    // For password reset (section 12)
    resetTokenHash: { type: String, select: false },
    resetTokenExpires: { type: Date, select: false },

    // For refresh-token revocation, you might track a token version:
    tokenVersion: { type: Number, default: 0, select: false },
  },
  { timestamps: true } // adds createdAt / updatedAt
);

// Virtual "password" setter: assign user.password = "plain" and it hashes on save.
// We store only the hash, never the plaintext.
userSchema.virtual("password").set(function (plain) {
  this._plainPassword = plain;
});

// Hash the password before saving, IF a new plaintext was provided.
userSchema.pre("save", async function (next) {
  if (this._plainPassword) {
    this.passwordHash = await bcrypt.hash(this._plainPassword, SALT_ROUNDS);
    this._plainPassword = undefined;
  }
  next();
});

// Instance method to verify a password at login.
userSchema.methods.comparePassword = function (candidate) {
  return bcrypt.compare(candidate, this.passwordHash);
};

// Create a password reset token: return the RAW token (emailed to user),
// but store only its HASH in the DB (so a DB leak can't be used to reset).
userSchema.methods.createPasswordResetToken = function () {
  const rawToken = crypto.randomBytes(32).toString("hex");
  this.resetTokenHash = crypto.createHash("sha256").update(rawToken).digest("hex");
  this.resetTokenExpires = new Date(Date.now() + 1000 * 60 * 15); // 15 minutes
  return rawToken;
};

// Ensure passwordHash and reset fields never serialize to JSON responses.
userSchema.set("toJSON", {
  virtuals: true,
  transform(_doc, ret) {
    delete ret.passwordHash;
    delete ret.resetTokenHash;
    delete ret.resetTokenExpires;
    delete ret.tokenVersion;
    delete ret.__v;
    return ret;
  },
});

module.exports = mongoose.model("User", userSchema);
```

> BEST PRACTICE: Mark `passwordHash` with `select: false` so it never comes back in ordinary queries. When you *do* need it (only at login), opt in with `.select("+passwordHash")`. This is a strong guard against accidentally serializing the hash to a response.

> WARNING — Pitfall: MongoDB's `unique` index is **case-sensitive** by default, so `Bob@x.com` and `bob@x.com` would be two different users. Normalizing email to lowercase (as above) fixes this for new records; for full safety use a case-insensitive collation on the index.

---

## 9. A Complete Register / Login Flow (JWT)

Now we assemble the pieces into real endpoints. We'll use JWTs delivered in **httpOnly cookies** (more on why in section 11), and add input validation.

```js
// auth.routes.js
const express = require("express");
const jwt = require("jsonwebtoken");
const User = require("./models/User");

const router = express.Router();

const ACCESS_SECRET = process.env.JWT_ACCESS_SECRET;
const REFRESH_SECRET = process.env.JWT_REFRESH_SECRET;

function signAccessToken(user) {
  return jwt.sign({ role: user.role }, ACCESS_SECRET, {
    subject: String(user._id),
    expiresIn: "15m",
  });
}

function signRefreshToken(user) {
  return jwt.sign({ tv: user.tokenVersion }, REFRESH_SECRET, {
    subject: String(user._id),
    expiresIn: "7d",
  });
}

// Helper to set both tokens as httpOnly cookies.
function setAuthCookies(res, accessToken, refreshToken) {
  const base = {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    sameSite: "lax",
  };
  res.cookie("accessToken", accessToken, { ...base, maxAge: 1000 * 60 * 15 });        // 15m
  res.cookie("refreshToken", refreshToken, {
    ...base,
    maxAge: 1000 * 60 * 60 * 24 * 7,   // 7d
    path: "/auth/refresh",             // refresh cookie only sent to the refresh route
  });
}

// --- REGISTER ---
router.post("/register", async (req, res) => {
  try {
    const { name, email, password } = req.body;

    // Basic validation (use a library like zod/joi in real apps).
    if (!name || !email || !password) {
      return res.status(400).json({ error: "name, email, password are required" });
    }
    if (password.length < 8) {
      return res.status(400).json({ error: "Password must be at least 8 characters" });
    }

    // Reject duplicate emails BEFORE attempting to create.
    const existing = await User.findOne({ email: email.toLowerCase() });
    if (existing) {
      // Note the deliberately vague message — see pitfalls.
      return res.status(409).json({ error: "Email already in use" });
    }

    // The virtual setter hashes on save.
    const user = await User.create({ name, email, password });

    const accessToken = signAccessToken(user);
    const refreshToken = signRefreshToken(user);
    setAuthCookies(res, accessToken, refreshToken);

    return res.status(201).json({ user }); // toJSON strips the hash
  } catch (err) {
    // Handle the race where two requests create the same email simultaneously.
    if (err.code === 11000) {
      return res.status(409).json({ error: "Email already in use" });
    }
    return res.status(500).json({ error: "Registration failed" });
  }
});

// --- LOGIN ---
router.post("/login", async (req, res) => {
  const { email, password } = req.body;
  if (!email || !password) {
    return res.status(400).json({ error: "email and password are required" });
  }

  // Must opt in to passwordHash since it's select:false.
  const user = await User.findOne({ email: String(email).toLowerCase() }).select(
    "+passwordHash +tokenVersion"
  );

  // Use ONE generic message for both "no user" and "wrong password".
  if (!user || !(await user.comparePassword(password))) {
    return res.status(401).json({ error: "Invalid email or password" });
  }

  const accessToken = signAccessToken(user);
  const refreshToken = signRefreshToken(user);
  setAuthCookies(res, accessToken, refreshToken);

  return res.json({ user });
});

module.exports = router;
```

> WARNING — Pitfall: **Always return the same generic error** ("Invalid email or password") whether the email doesn't exist or the password is wrong. Different messages let an attacker *enumerate* which emails are registered. (For registration, an "email in use" message does leak some info — mitigate with rate limiting and consider sending a confirmation email instead of a hard error in high-sensitivity apps.)

---

## 10. Protecting Routes with Auth Middleware

The whole point of authentication is to gate access. The pattern is a **middleware** that runs before protected handlers, verifies the token, and — on success — attaches the identity to `req.user`. Downstream handlers then trust `req.user`.

```js
// middleware/requireAuth.js
const jwt = require("jsonwebtoken");
const User = require("../models/User");

const ACCESS_SECRET = process.env.JWT_ACCESS_SECRET;

async function requireAuth(req, res, next) {
  try {
    // Accept token from the Authorization header OR an httpOnly cookie.
    let token;
    const header = req.headers.authorization;
    if (header && header.startsWith("Bearer ")) {
      token = header.slice(7); // strip "Bearer "
    } else if (req.cookies && req.cookies.accessToken) {
      token = req.cookies.accessToken;
    }

    if (!token) {
      return res.status(401).json({ error: "Authentication required" });
    }

    // Verify signature + expiry; pin the algorithm.
    const payload = jwt.verify(token, ACCESS_SECRET, { algorithms: ["HS256"] });

    // OPTION A (lean, stateless): trust the token's claims directly.
    // req.user = { id: payload.sub, role: payload.role };

    // OPTION B (safer): load the user to confirm they still exist / aren't banned.
    const user = await User.findById(payload.sub);
    if (!user) return res.status(401).json({ error: "User no longer exists" });

    req.user = user;          // attach for downstream handlers
    req.token = payload;      // optional: keep claims around
    next();
  } catch (err) {
    if (err.name === "TokenExpiredError") {
      return res.status(401).json({ error: "Token expired" });
    }
    return res.status(401).json({ error: "Invalid token" });
  }
}

// Authorization helper: require a specific role (runs AFTER requireAuth).
function requireRole(...roles) {
  return (req, res, next) => {
    if (!req.user || !roles.includes(req.user.role)) {
      return res.status(403).json({ error: "Forbidden" }); // 403, not 401!
    }
    next();
  };
}

module.exports = { requireAuth, requireRole };
```

**Using it:**

```js
// app.js (wiring)
const express = require("express");
const cookieParser = require("cookie-parser");
const { requireAuth, requireRole } = require("./middleware/requireAuth");

const app = express();
app.use(express.json());
app.use(cookieParser());

app.use("/auth", require("./auth.routes"));

// Protected: any authenticated user
app.get("/me", requireAuth, (req, res) => {
  res.json({ user: req.user });
});

// Protected + authorized: admins only
app.delete(
  "/users/:id",
  requireAuth,
  requireRole("admin"),
  async (req, res) => {
    // ... delete logic ...
    res.json({ message: "Deleted" });
  }
);
```

> TIP: **Option A vs Option B** is a real tradeoff. Pure-stateless (A) is faster (no DB hit) but means a banned/deleted user keeps working until their short token expires. Loading the user (B) costs a query but lets you enforce bans, soft-deletes, and `tokenVersion` checks immediately. With short access-token lifetimes (15m), A is often acceptable; for sensitive actions, prefer B.

---

## 11. Where to Store Tokens on the Client

This is one of the most asked — and most misunderstood — interview questions. There is no perfect option; you are choosing which attack to defend against most strongly.

| Storage | Sent automatically? | Readable by JS? | Vulnerable to XSS? | Vulnerable to CSRF? |
| --- | --- | --- | --- | --- |
| **`localStorage`** | No (you attach it manually) | **Yes** | **Yes — XSS can steal the token** | No (not auto-sent) |
| **`sessionStorage`** | No | Yes | Yes | No |
| **httpOnly cookie** | **Yes** | **No** | No (JS can't read it) | **Yes — needs CSRF defense** |
| **In-memory (JS variable)** | No | Only to your app code | Lost on reload, smaller XSS window | No |

Key reasoning:

- **`localStorage`**: convenient (send via `Authorization: Bearer`), immune to CSRF (the browser won't attach it automatically), but **any XSS vulnerability hands the attacker your token** because JS can read it. XSS is extremely common, which makes this riskier than it looks.
- **httpOnly cookie**: JavaScript *cannot* read it, so XSS can't exfiltrate it. But because the browser sends it automatically, you become vulnerable to **CSRF** — you must add `SameSite=Lax/Strict` and/or CSRF tokens.

> BEST PRACTICE: For browser apps, the modern consensus is **store the access token in memory (a JS variable) and the refresh token in an httpOnly, `SameSite` cookie scoped to the refresh endpoint.** This minimizes the XSS attack surface (no long-lived token in JS-readable storage) and the CSRF surface (refresh cookie only goes to one path). On page reload, silently hit the refresh endpoint to get a new access token. For server-rendered apps, httpOnly cookies for everything plus CSRF protection is simplest and safe.

> WARNING — Pitfall: "Just put the JWT in `localStorage`" is the default in many tutorials and is the wrong default for anything sensitive. If you do use `localStorage`, you are betting your entire auth on having zero XSS — a bet most apps lose eventually.

---

## 12. Access Tokens vs Refresh Tokens (and the Refresh Flow)

A single long-lived token is a poor design: if it leaks, the attacker has long-lived access, and you can't easily revoke it. The standard solution splits responsibilities into two tokens.

| | **Access token** | **Refresh token** |
| --- | --- | --- |
| Purpose | Prove identity on each API call | Obtain a *new* access token when it expires |
| Lifetime | Short (5–15 min) | Long (days–weeks) |
| Sent where | Every protected request | Only to the refresh endpoint |
| Stored where (recommended) | In memory / `Authorization` header | httpOnly cookie, path-scoped |
| Stored server-side? | No (stateless) | Often **yes** (to allow revocation) |

**Why this design works:** the short-lived access token limits the blast radius of a leak (it expires fast). The long-lived refresh token is sent rarely (only to `/auth/refresh`), can be stored more defensively, and — crucially — can be **tracked and revoked** server-side, giving you back the ability to "log someone out everywhere" that pure stateless JWT lacks.

### 12.1 The refresh flow

1. Login issues an access token (15m) **and** a refresh token (7d).
2. Client uses the access token until it expires (gets 401 `Token expired`).
3. Client POSTs to `/auth/refresh` (sending the refresh token cookie).
4. Server verifies the refresh token, **rotates** it (issues a new refresh token, invalidates the old), and returns a fresh access token.
5. On logout or compromise, server invalidates the refresh token.

```js
// refresh.route.js
const express = require("express");
const jwt = require("jsonwebtoken");
const User = require("./models/User");

const router = express.Router();
const ACCESS_SECRET = process.env.JWT_ACCESS_SECRET;
const REFRESH_SECRET = process.env.JWT_REFRESH_SECRET;

router.post("/refresh", async (req, res) => {
  const token = req.cookies.refreshToken;
  if (!token) return res.status(401).json({ error: "No refresh token" });

  let payload;
  try {
    payload = jwt.verify(token, REFRESH_SECRET, { algorithms: ["HS256"] });
  } catch {
    return res.status(401).json({ error: "Invalid refresh token" });
  }

  // Load user + their current token version for revocation checks.
  const user = await User.findById(payload.sub).select("+tokenVersion");
  if (!user) return res.status(401).json({ error: "User not found" });

  // Revocation via tokenVersion: if the stored version moved past the token's,
  // this refresh token has been invalidated (e.g., "log out everywhere").
  if (payload.tv !== user.tokenVersion) {
    return res.status(401).json({ error: "Refresh token revoked" });
  }

  // Issue a fresh access token.
  const accessToken = jwt.sign({ role: user.role }, ACCESS_SECRET, {
    subject: String(user._id),
    expiresIn: "15m",
  });

  // ROTATION: issue a new refresh token too (same tokenVersion here; bump
  // tokenVersion only on logout/compromise to invalidate all refresh tokens).
  const refreshToken = jwt.sign({ tv: user.tokenVersion }, REFRESH_SECRET, {
    subject: String(user._id),
    expiresIn: "7d",
  });

  res.cookie("accessToken", accessToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    sameSite: "lax",
    maxAge: 1000 * 60 * 15,
  });
  res.cookie("refreshToken", refreshToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    sameSite: "lax",
    path: "/auth/refresh",
    maxAge: 1000 * 60 * 60 * 24 * 7,
  });

  res.json({ message: "Refreshed" });
});

module.exports = router;
```

> BEST PRACTICE: **Rotate refresh tokens** on every use and detect reuse. If an old (already-rotated) refresh token is presented, treat it as theft and invalidate the whole family (bump `tokenVersion`). For stronger guarantees, store refresh tokens (or their hashes + a `jti`) in the DB so you can revoke individual sessions, not just "all sessions."

---

## 13. Logout

Logout differs between the two models:

- **Session-based:** truly destroy the session server-side (`req.session.destroy`) and clear the cookie. The session ID is now meaningless. (Shown in section 5.3.)
- **JWT (stateless access token):** you cannot "un-issue" a signed token — it remains valid until `exp`. Practical logout for JWT:
  1. **Clear the cookies** on the client (so the browser stops sending them).
  2. **Invalidate the refresh token** server-side (bump `tokenVersion`, or delete the stored refresh token), so it can't mint new access tokens.
  3. Keep access tokens short-lived so the lingering window is small.
  4. For instant access-token revocation, maintain a **denylist** (store `jti` of revoked tokens in Redis with a TTL = remaining lifetime).

```js
// logout.route.js
const express = require("express");
const User = require("./models/User");
const { requireAuth } = require("./middleware/requireAuth");

const router = express.Router();

router.post("/logout", requireAuth, async (req, res) => {
  // Invalidate ALL refresh tokens for this user (log out everywhere).
  await User.findByIdAndUpdate(req.user._id, { $inc: { tokenVersion: 1 } });

  // Clear cookies — options (path, etc.) must match how they were set.
  const base = { httpOnly: true, sameSite: "lax", secure: process.env.NODE_ENV === "production" };
  res.clearCookie("accessToken", base);
  res.clearCookie("refreshToken", { ...base, path: "/auth/refresh" });

  res.json({ message: "Logged out" });
});

module.exports = router;
```

> WARNING — Pitfall: `res.clearCookie` only works if the **options match** those used to set the cookie (especially `path` and `domain`). A common bug is "logout doesn't clear the cookie" because the `path` was `/auth/refresh` on set but omitted on clear.

---

## 14. Password Reset Flow (Token + Email)

A secure reset flow lets users who forgot their password set a new one, *without* the server ever knowing or emailing their old password. The trick: email a **one-time, time-limited, random token**, store only its **hash** in the DB, and require it to set a new password.

### 14.1 The flow

1. User submits their email to `/auth/forgot-password`.
2. Server generates a random token, stores its **SHA-256 hash** + expiry on the user, and emails the **raw** token as a link (`/reset-password?token=...`).
3. User clicks the link and submits a new password + the token to `/auth/reset-password`.
4. Server hashes the submitted token, finds the matching, unexpired user, sets the new password, clears the reset fields, and (best practice) invalidates existing sessions/refresh tokens.

```js
// password-reset.routes.js
const express = require("express");
const crypto = require("crypto");
const User = require("./models/User");
const { sendEmail } = require("./email"); // your mailer (nodemailer, etc.)

const router = express.Router();

// --- Step 1: request a reset ---
router.post("/forgot-password", async (req, res) => {
  const { email } = req.body;
  const user = await User.findOne({ email: String(email || "").toLowerCase() });

  // ALWAYS respond with success, even if no user — don't reveal who's registered.
  if (user) {
    const rawToken = user.createPasswordResetToken(); // raw returned, hash stored
    await user.save({ validateBeforeSave: false });

    const resetUrl = `https://myapp.com/reset-password?token=${rawToken}`;
    try {
      await sendEmail({
        to: user.email,
        subject: "Reset your password",
        text: `Reset your password (valid 15 min): ${resetUrl}`,
      });
    } catch (err) {
      // If sending fails, roll back the token so the user can retry cleanly.
      user.resetTokenHash = undefined;
      user.resetTokenExpires = undefined;
      await user.save({ validateBeforeSave: false });
      return res.status(500).json({ error: "Could not send reset email" });
    }
  }

  res.json({ message: "If that email exists, a reset link has been sent." });
});

// --- Step 2: perform the reset ---
router.post("/reset-password", async (req, res) => {
  const { token, password } = req.body;
  if (!token || !password || password.length < 8) {
    return res.status(400).json({ error: "Token and a valid new password are required" });
  }

  // Hash the incoming raw token and look up by the stored hash + unexpired.
  const tokenHash = crypto.createHash("sha256").update(token).digest("hex");
  const user = await User.findOne({
    resetTokenHash: tokenHash,
    resetTokenExpires: { $gt: new Date() }, // not expired
  }).select("+resetTokenHash +resetTokenExpires +tokenVersion");

  if (!user) {
    return res.status(400).json({ error: "Token is invalid or has expired" });
  }

  // Set the new password (virtual setter hashes on save) and clear reset fields.
  user.password = password;
  user.resetTokenHash = undefined;
  user.resetTokenExpires = undefined;
  // Invalidate all existing sessions/refresh tokens — a reset should log out everywhere.
  user.tokenVersion += 1;
  await user.save();

  res.json({ message: "Password updated. Please log in again." });
});

module.exports = router;
```

> BEST PRACTICE: Store the **hash** of the reset token, not the token itself. Then a database leak does not let an attacker reset accounts (they'd only have hashes). This mirrors how you treat passwords. Also keep reset tokens **short-lived** (10–30 min) and **single-use** (clear them after use).

> WARNING — Pitfall: Always return the same "if that email exists..." response on `/forgot-password`, regardless of whether the email is registered. Otherwise you leak which emails have accounts (user enumeration).

A minimal mailer for completeness:

```js
// email.js (dev example with nodemailer)
const nodemailer = require("nodemailer");

async function sendEmail({ to, subject, text }) {
  // In dev, use a test account (ethereal.email) or a local SMTP catcher.
  const transporter = nodemailer.createTransport({
    host: process.env.SMTP_HOST,
    port: Number(process.env.SMTP_PORT),
    auth: { user: process.env.SMTP_USER, pass: process.env.SMTP_PASS },
  });
  await transporter.sendMail({ from: '"MyApp" <no-reply@myapp.com>', to, subject, text });
}

module.exports = { sendEmail };
```

---

## 15. Passport.js and OAuth / Social Login (Overview)

### 15.1 What Passport is

**Passport.js** is authentication *middleware* for Express. It does not invent a new auth model; it provides a uniform interface over hundreds of pluggable **strategies** — `passport-local` (email/password), `passport-jwt` (JWT verification), `passport-google-oauth20`, `passport-github2`, `passport-facebook`, and many more. You configure a strategy once, then use `passport.authenticate("strategy")` as middleware. It standardizes the messy details of each provider behind one API and (for session mode) handles `serializeUser`/`deserializeUser` to store the user id in the session.

When should you use it? When you need **many auth methods** or **social login** and don't want to hand-roll each provider's OAuth dance. For a single email/password + JWT app, the hand-written code in this module is often clearer than pulling in Passport.

```js
// A tiny taste of passport-local
const passport = require("passport");
const LocalStrategy = require("passport-local").Strategy;
const User = require("./models/User");

passport.use(
  new LocalStrategy({ usernameField: "email" }, async (email, password, done) => {
    try {
      const user = await User.findOne({ email }).select("+passwordHash");
      if (!user || !(await user.comparePassword(password))) {
        return done(null, false, { message: "Invalid credentials" });
      }
      return done(null, user); // success: user attaches to req.user
    } catch (err) {
      return done(err);
    }
  })
);

// app.post("/login", passport.authenticate("local", { session: false }), (req, res) => {
//   res.json({ user: req.user });
// });
```

### 15.2 OAuth 2.0 / social login in one picture

**OAuth 2.0** is an *authorization-delegation* protocol: it lets your app act on a user's behalf at another service (Google, GitHub) *without* the user giving you their password. "Social login" layers identity on top (often via **OpenID Connect**, a thin identity layer over OAuth 2.0). The common **Authorization Code** flow:

1. User clicks "Sign in with Google." Your app redirects them to Google's authorization URL (with your `client_id`, requested `scope`, and a `redirect_uri`).
2. Google authenticates the user and asks them to consent to the requested scopes.
3. Google redirects back to your `redirect_uri` with a short-lived **authorization code**.
4. Your **server** exchanges that code (plus your `client_secret`) for an **access token** (and often an ID token / refresh token) — this exchange happens server-to-server, so the secret never touches the browser.
5. Your server uses the access token to fetch the user's profile (email, name, provider id).
6. You **find-or-create** a local user matching that provider identity, then issue *your own* session or JWT. From here on, the user is logged into your app with your own auth.

```js
// Conceptual passport-google-oauth20 wiring
const GoogleStrategy = require("passport-google-oauth20").Strategy;

passport.use(
  new GoogleStrategy(
    {
      clientID: process.env.GOOGLE_CLIENT_ID,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET,
      callbackURL: "/auth/google/callback",
    },
    async (accessToken, refreshToken, profile, done) => {
      // find-or-create your local user from the Google profile
      const email = profile.emails[0].value;
      let user = await User.findOne({ email });
      if (!user) {
        user = await User.create({
          name: profile.displayName,
          email,
          googleId: profile.id,
          // no password needed for social-only accounts
        });
      }
      return done(null, user);
    }
  )
);

// Step 1: kick off the flow
// app.get("/auth/google", passport.authenticate("google", { scope: ["profile", "email"] }));
// Step 2: Google redirects back here; then issue YOUR token/session
// app.get("/auth/google/callback",
//   passport.authenticate("google", { session: false, failureRedirect: "/login" }),
//   (req, res) => { /* sign your own JWT for req.user, set cookie, redirect */ }
// );
```

> TIP: The key mental model: **OAuth gets you the user's identity from a provider; you still issue your own app session/token afterward.** Never trust the provider's token as your app's auth token directly.

> WARNING — Pitfall: Always validate the OAuth **`state`** parameter (CSRF protection for the redirect) and use exact-match `redirect_uri`s registered with the provider. Misconfigured redirect URIs are a classic account-takeover vector. Passport's OAuth strategies handle `state` for you when enabled — don't disable it.

---

## Common Pitfalls & Gotchas

- **Confusing 401 and 403.** 401 = not authenticated (who are you?); 403 = authenticated but not allowed (you can't do this).
- **Storing plaintext or fast-hashed (MD5/SHA-256) passwords.** Always bcrypt/Argon2 with a per-password salt and a tuned cost.
- **Using sync bcrypt (`hashSync`/`compareSync`)** in request handlers — it blocks the event loop during the deliberately slow hash.
- **Comparing hashes with `===`.** Use `bcrypt.compare` (constant-time, parses the embedded salt).
- **Putting secrets in a JWT payload.** It is encoded, not encrypted — anyone can read it.
- **Not pinning the algorithm in `jwt.verify`.** Omitting `{ algorithms: [...] }` opens you to `alg: none` and RS256→HS256 confusion attacks.
- **Trusting `jwt.decode()` for auth.** It does *not* verify the signature; only `verify()` does.
- **`maxAge` unit confusion:** Express `res.cookie({ maxAge })` is **milliseconds**; the raw `Max-Age` header and `connect-mongo`'s `ttl` are **seconds**.
- **Forgetting `httpOnly`/`secure`/`sameSite`** on auth cookies — that's how cookies get stolen by XSS or sent over plaintext HTTP.
- **`res.clearCookie` not clearing** because the `path`/`domain` options don't match how the cookie was set.
- **In-memory session store in production** — leaks memory, loses sessions on restart, breaks across multiple instances. Use `connect-mongo`/`connect-redis`.
- **Session fixation:** not calling `req.session.regenerate()` after login.
- **User enumeration:** different error messages or response timing for "no such email" vs "wrong password," and for `/forgot-password`.
- **Assuming you can "revoke" a stateless JWT.** You can't until it expires — that's what short lifetimes + refresh tokens + denylists are for.
- **Leaking the password hash** in API responses — guard with `select: false` and a `toJSON` transform.
- **No rate limiting on `/login`, `/register`, `/forgot-password`** — invites credential stuffing and brute force.

## Best Practices

- **Hash with bcrypt cost 10–12** (benchmark on your hardware), always async. Consider Argon2id for new projects.
- **Validate and normalize input** (lowercase/trim emails, enforce password length/strength) with a schema validator (zod/joi).
- **Short-lived access tokens (5–15 min) + longer refresh tokens (days)**, with refresh rotation and reuse detection.
- **Cookies:** `httpOnly`, `secure` (prod), `sameSite=Lax` (or `Strict`), scope the refresh cookie to its endpoint via `path`.
- **Pin JWT algorithms** on `verify`, set `expiresIn`, and set/verify `iss`/`aud` for defense in depth.
- **Keep verification keys/secrets out of code** — use environment variables / a secret manager; use long, random secrets.
- **Generic auth errors** ("Invalid email or password") and identical `/forgot-password` responses to prevent enumeration.
- **Rate-limit and lock out** repeated failures (`express-rate-limit`); consider account lockout/backoff and CAPTCHA on abuse.
- **Always serve auth over HTTPS**; set `app.set("trust proxy", 1)` behind a proxy so `secure` cookies and `req.secure` work.
- **Regenerate session IDs** after login; **bump `tokenVersion`/destroy sessions** on logout and password reset.
- **Add security headers** with `helmet`; mitigate XSS (the root cause of token theft) with a strong Content Security Policy.
- **Store reset/refresh tokens hashed**, short-lived, single-use where applicable.

## Key Interview Points

- **Authentication vs authorization?** AuthN verifies identity (who you are); AuthZ verifies permissions (what you can do). AuthN first, then AuthZ. 401 = unauthenticated, 403 = unauthorized.
- **Why hash passwords, and why bcrypt over SHA-256?** Hashing is one-way; bcrypt is *deliberately slow* and salted, defeating brute-force and rainbow tables. SHA-256 is too fast for passwords.
- **What is a salt and where does bcrypt store it?** Random per-password value mixed in before hashing; bcrypt embeds the salt (and cost + version) inside the output hash string.
- **What are salt rounds / the cost factor?** A logarithmic work factor; iterations = `2^cost`. Tune so a hash takes ~100–250 ms; raise as hardware improves.
- **Sessions vs JWT — tradeoffs?** Sessions are stateful (server store, easy revocation, needs shared store to scale); JWTs are stateless (self-contained, scale trivially, hard to revoke before expiry).
- **JWT structure?** `header.payload.signature`, each Base64URL-encoded; signature = HMAC/asymmetric signature over header+payload. Payload is *encoded, not encrypted*.
- **Symmetric vs asymmetric JWT signing?** HS256 uses one shared secret (sign = verify); RS256/ES256 uses a private key to sign and a freely-shareable public key to verify — better for multi-service/third-party verification.
- **`alg: none` / algorithm confusion attack?** Mitigate by always passing `{ algorithms: ['HS256'] }` (or your list) to `jwt.verify`.
- **Access vs refresh tokens?** Short-lived access token for API calls; long-lived refresh token (sent only to the refresh endpoint, often DB-tracked) to mint new access tokens and enable revocation/rotation.
- **Where to store tokens client-side?** `localStorage` is XSS-exposed; httpOnly cookies are XSS-safe but CSRF-exposed (mitigate with `SameSite`/CSRF tokens). Recommended: access token in memory + refresh token in a path-scoped httpOnly cookie.
- **How do you log out with JWTs?** Clear cookies, invalidate the refresh token (e.g., bump `tokenVersion`), keep access tokens short; use a `jti` denylist for instant revocation.
- **httpOnly / secure / sameSite — what each does?** httpOnly blocks JS access (anti-XSS-theft); secure forces HTTPS; sameSite controls cross-site sending (anti-CSRF).
- **What is session fixation and how do you prevent it?** Reusing an attacker-known session ID after login; prevent by regenerating the session ID on login.
- **How do you build a secure password reset?** Email a random, short-lived, single-use token; store only its hash; verify by hashing the submitted token; invalidate sessions after reset.
- **What is Passport.js / OAuth?** Passport is pluggable Express auth middleware (strategies). OAuth 2.0 delegates authorization to a provider; you exchange an auth code server-side for tokens, fetch the profile, find-or-create a local user, then issue *your own* session/JWT.

## Practice Exercises

1. **bcrypt round-trip.** Write a script that hashes the password `"correct horse battery staple"` with cost 12, prints the hash, then verifies both the correct password and a wrong one. Bonus: time how long `bcrypt.hash` takes at cost 10 vs 14.
2. **JWT sign & verify.** Sign an HS256 token with a `sub` (user id) and `role` claim that expires in 30 seconds. Verify it immediately (should pass), then write code that waits past expiry and verifies again, handling `TokenExpiredError` gracefully. Pin the algorithm.
3. **Auth middleware.** Implement `requireAuth` that reads a Bearer token, verifies it, loads the user, and attaches `req.user`. Add a `requireRole("admin")` guard. Protect a `GET /admin` route so only admins get `200`, authenticated non-admins get `403`, and anonymous requests get `401`.
4. **Session login with connect-mongo.** Wire up `express-session` with `connect-mongo`, implement `/login` (regenerating the session), a protected `/profile`, and `/logout` (destroying the session and clearing the cookie). Verify with cookies that the session survives a server *request* but is gone after logout.
5. **Refresh-token flow.** Implement `/auth/login` issuing access (15m) + refresh (7d) tokens as httpOnly cookies, and `/auth/refresh` that verifies the refresh token, checks `tokenVersion`, and issues a new access token. Add `/auth/logout` that bumps `tokenVersion` and clears cookies. Demonstrate that after logout, refresh fails.
6. **Password reset.** Implement `/auth/forgot-password` (generate token, store its hash + 15-min expiry, "email" the raw token by logging it) and `/auth/reset-password` (verify token by hash + expiry, set new password, clear reset fields, bump `tokenVersion`). Ensure `/forgot-password` returns the same response for known and unknown emails.

### Solutions

**Solution 1 — bcrypt round-trip:**

```js
// ex1.js
const bcrypt = require("bcrypt");

(async () => {
  const pw = "correct horse battery staple";

  const hash = await bcrypt.hash(pw, 12);
  console.log("Hash:", hash);

  console.log("Correct:", await bcrypt.compare(pw, hash));   // true
  console.log("Wrong:  ", await bcrypt.compare("nope", hash)); // false

  for (const cost of [10, 14]) {
    const start = Date.now();
    await bcrypt.hash(pw, cost);
    console.log(`cost ${cost}: ${Date.now() - start} ms`); // ~16x slower at 14 vs 10
  }
})();
```

**Solution 2 — JWT sign & verify with expiry:**

```js
// ex2.js
const jwt = require("jsonwebtoken");
const SECRET = "test-secret";

const token = jwt.sign({ role: "user" }, SECRET, {
  subject: "user-123",
  expiresIn: "30s",
});

function check(tok) {
  try {
    const decoded = jwt.verify(tok, SECRET, { algorithms: ["HS256"] });
    console.log("Valid:", decoded.sub, decoded.role);
  } catch (err) {
    if (err.name === "TokenExpiredError") {
      console.log("Token expired at", err.expiredAt);
    } else {
      console.log("Invalid token:", err.message);
    }
  }
}

check(token); // Valid: user-123 user

// Simulate expiry without waiting: sign one that's already expired.
const expired = jwt.sign({ role: "user" }, SECRET, {
  subject: "user-123",
  expiresIn: "-10s", // negative => already expired
});
check(expired); // Token expired at ...
```

**Solution 3 — Auth + role middleware:**

```js
// ex3.js
const express = require("express");
const jwt = require("jsonwebtoken");

const app = express();
const SECRET = "test-secret";

// Fake user store
const users = {
  "u1": { id: "u1", role: "admin" },
  "u2": { id: "u2", role: "user" },
};

function requireAuth(req, res, next) {
  const header = req.headers.authorization || "";
  const token = header.startsWith("Bearer ") ? header.slice(7) : null;
  if (!token) return res.status(401).json({ error: "Authentication required" });
  try {
    const payload = jwt.verify(token, SECRET, { algorithms: ["HS256"] });
    const user = users[payload.sub];
    if (!user) return res.status(401).json({ error: "User not found" });
    req.user = user;
    next();
  } catch {
    return res.status(401).json({ error: "Invalid token" });
  }
}

function requireRole(...roles) {
  return (req, res, next) =>
    roles.includes(req.user.role)
      ? next()
      : res.status(403).json({ error: "Forbidden" });
}

app.get("/admin", requireAuth, requireRole("admin"), (req, res) => {
  res.json({ message: `Welcome, admin ${req.user.id}` });
});

// Helper to mint test tokens:
app.get("/token/:uid", (req, res) =>
  res.json({ token: jwt.sign({}, SECRET, { subject: req.params.uid, expiresIn: "1h" }) })
);

app.listen(3000, () => console.log("ex3 on :3000"));
// curl -H "Authorization: Bearer <admin token>" localhost:3000/admin  -> 200
// curl -H "Authorization: Bearer <user token>"  localhost:3000/admin  -> 403
// curl localhost:3000/admin                                           -> 401
```

**Solution 4 — Session login with connect-mongo:**

```js
// ex4.js
const express = require("express");
const session = require("express-session");
const MongoStore = require("connect-mongo");
const mongoose = require("mongoose");
const bcrypt = require("bcrypt");

const app = express();
app.use(express.json());

const URI = "mongodb://127.0.0.1:27017/ex4";

app.use(
  session({
    name: "sid",
    secret: "session-secret-change-me",
    resave: false,
    saveUninitialized: false,
    store: MongoStore.create({ mongoUrl: URI, ttl: 60 * 60 }),
    cookie: { httpOnly: true, sameSite: "lax", maxAge: 1000 * 60 * 60 },
  })
);

// Seed one user (in real life, comes from your User model)
let seeded;
async function getUser() {
  if (!seeded) {
    seeded = { id: "u1", email: "a@b.com", passwordHash: await bcrypt.hash("pw12345678", 12) };
  }
  return seeded;
}

app.post("/login", async (req, res) => {
  const user = await getUser();
  const { email, password } = req.body;
  if (email !== user.email || !(await bcrypt.compare(password, user.passwordHash))) {
    return res.status(401).json({ error: "Invalid email or password" });
  }
  req.session.regenerate((err) => {            // prevent session fixation
    if (err) return res.status(500).json({ error: "Session error" });
    req.session.userId = user.id;
    req.session.save(() => res.json({ message: "Logged in" }));
  });
});

function requireSession(req, res, next) {
  if (!req.session.userId) return res.status(401).json({ error: "Not authenticated" });
  next();
}

app.get("/profile", requireSession, (req, res) =>
  res.json({ userId: req.session.userId })
);

app.post("/logout", (req, res) => {
  req.session.destroy((err) => {
    if (err) return res.status(500).json({ error: "Logout failed" });
    res.clearCookie("sid");
    res.json({ message: "Logged out" });
  });
});

mongoose.connect(URI).then(() =>
  app.listen(3000, () => console.log("ex4 on :3000"))
);
// curl -c jar -X POST localhost:3000/login -H 'content-type: application/json' -d '{"email":"a@b.com","password":"pw12345678"}'
// curl -b jar localhost:3000/profile      -> { userId: "u1" }
// curl -b jar -X POST localhost:3000/logout
// curl -b jar localhost:3000/profile      -> 401
```

**Solution 5 — Refresh-token flow:**

```js
// ex5.js
const express = require("express");
const cookieParser = require("cookie-parser");
const jwt = require("jsonwebtoken");

const app = express();
app.use(express.json());
app.use(cookieParser());

const ACCESS_SECRET = "access-secret";
const REFRESH_SECRET = "refresh-secret";

// Fake user with a tokenVersion we can bump on logout.
const user = { id: "u1", role: "user", tokenVersion: 0 };

function issue(res) {
  const accessToken = jwt.sign({ role: user.role }, ACCESS_SECRET, {
    subject: user.id,
    expiresIn: "15m",
  });
  const refreshToken = jwt.sign({ tv: user.tokenVersion }, REFRESH_SECRET, {
    subject: user.id,
    expiresIn: "7d",
  });
  const base = { httpOnly: true, sameSite: "lax" };
  res.cookie("accessToken", accessToken, { ...base, maxAge: 1000 * 60 * 15 });
  res.cookie("refreshToken", refreshToken, {
    ...base,
    path: "/auth/refresh",
    maxAge: 1000 * 60 * 60 * 24 * 7,
  });
}

app.post("/auth/login", (req, res) => {
  issue(res);
  res.json({ message: "Logged in" });
});

app.post("/auth/refresh", (req, res) => {
  const token = req.cookies.refreshToken;
  if (!token) return res.status(401).json({ error: "No refresh token" });
  let payload;
  try {
    payload = jwt.verify(token, REFRESH_SECRET, { algorithms: ["HS256"] });
  } catch {
    return res.status(401).json({ error: "Invalid refresh token" });
  }
  if (payload.tv !== user.tokenVersion) {
    return res.status(401).json({ error: "Refresh token revoked" });
  }
  issue(res); // rotate
  res.json({ message: "Refreshed" });
});

app.post("/auth/logout", (req, res) => {
  user.tokenVersion += 1; // invalidate all refresh tokens
  res.clearCookie("accessToken", { httpOnly: true, sameSite: "lax" });
  res.clearCookie("refreshToken", { httpOnly: true, sameSite: "lax", path: "/auth/refresh" });
  res.json({ message: "Logged out" });
});

app.listen(3000, () => console.log("ex5 on :3000"));
// curl -c jar -X POST localhost:3000/auth/login
// curl -b jar -X POST localhost:3000/auth/refresh   -> Refreshed
// curl -b jar -X POST localhost:3000/auth/logout
// curl -b jar -X POST localhost:3000/auth/refresh   -> 401 Refresh token revoked
```

**Solution 6 — Password reset:**

```js
// ex6.js
const express = require("express");
const crypto = require("crypto");
const bcrypt = require("bcrypt");

const app = express();
app.use(express.json());

// Fake single-user store with reset fields.
const user = {
  email: "a@b.com",
  passwordHash: null,
  resetTokenHash: null,
  resetTokenExpires: null,
  tokenVersion: 0,
};
(async () => { user.passwordHash = await bcrypt.hash("oldpassword", 12); })();

app.post("/auth/forgot-password", (req, res) => {
  const { email } = req.body;
  // Always identical response (no enumeration).
  if (email === user.email) {
    const rawToken = crypto.randomBytes(32).toString("hex");
    user.resetTokenHash = crypto.createHash("sha256").update(rawToken).digest("hex");
    user.resetTokenExpires = Date.now() + 1000 * 60 * 15; // 15 min
    console.log("RESET LINK: /reset?token=" + rawToken);  // "email" it
  }
  res.json({ message: "If that email exists, a reset link has been sent." });
});

app.post("/auth/reset-password", async (req, res) => {
  const { token, password } = req.body;
  if (!token || !password || password.length < 8) {
    return res.status(400).json({ error: "Token and a valid new password required" });
  }
  const tokenHash = crypto.createHash("sha256").update(token).digest("hex");
  const valid =
    user.resetTokenHash === tokenHash &&
    user.resetTokenExpires &&
    user.resetTokenExpires > Date.now();

  if (!valid) return res.status(400).json({ error: "Token is invalid or has expired" });

  user.passwordHash = await bcrypt.hash(password, 12);
  user.resetTokenHash = null;
  user.resetTokenExpires = null;
  user.tokenVersion += 1; // log out everywhere
  res.json({ message: "Password updated. Please log in again." });
});

app.listen(3000, () => console.log("ex6 on :3000"));
// 1) curl -X POST localhost:3000/auth/forgot-password -H 'content-type: application/json' -d '{"email":"a@b.com"}'
//    -> server logs "RESET LINK: /reset?token=<raw>"
// 2) curl -X POST localhost:3000/auth/reset-password -H 'content-type: application/json' -d '{"token":"<raw>","password":"newpassword1"}'
//    -> Password updated
// 3) Re-using the same token -> 400 invalid or expired
```

---

## Summary / Cheat Sheet

| Concept | Key takeaway |
| --- | --- |
| AuthN vs AuthZ | AuthN = who you are (401); AuthZ = what you can do (403). AuthN first. |
| Password storage | Never plaintext. bcrypt (cost 10–12), async, salted; salt is embedded in the hash. |
| `bcrypt.compare` | Constant-time, parses embedded salt/cost. Never use `===`. |
| Sessions | Stateful; server store + cookie holds session ID. Use `connect-mongo`/Redis in prod. |
| `express-session` options | `resave:false`, `saveUninitialized:false`, secure cookie, persistent store, regenerate on login. |
| Cookies | `httpOnly` (anti-XSS), `secure` (HTTPS), `sameSite` (anti-CSRF). Express `maxAge` is ms. |
| Cookie signing | Tamper-evident (HMAC), NOT encryption. |
| JWT structure | `header.payload.signature`, Base64URL. Payload is encoded, not encrypted. |
| JWT signing | HS256 = shared secret; RS256/ES256 = private signs, public verifies (multi-service). |
| `jwt.verify` | Always pass `{ algorithms: [...] }`; handle `TokenExpiredError`. `decode()` ≠ verify. |
| Access vs refresh | Short access (5–15m) + long refresh (days, path-scoped cookie, revocable). Rotate refresh. |
| Token storage (client) | Access in memory; refresh in httpOnly path-scoped cookie. localStorage = XSS risk. |
| Auth middleware | Verify token → load/attach `req.user` → `next()`. Add `requireRole` for AuthZ. |
| Logout (JWT) | Clear cookies + invalidate refresh (`tokenVersion`/denylist); access expires on its own. |
| Password reset | Email random short-lived single-use token; store only its hash; invalidate sessions after. |
| Passport / OAuth | Passport = strategy middleware. OAuth: exchange code server-side → fetch profile → issue YOUR token. |
| Always | HTTPS, rate limiting, generic auth errors, helmet/CSP, secrets in env, validate input. |
