# The Request & Response Objects (Deep Dive)

Every HTTP interaction in an Express application boils down to two objects: the **request** (`req`) and the **response** (`res`). The browser (or `curl`, or another service) sends a request; your server inspects it, decides what to do, and sends a response. Mastering these two objects is what separates someone who can *follow* an Express tutorial from someone who can *design* a production API and debug it under pressure.

Express does not invent `req` and `res` from scratch. Underneath, Node's core `http` module hands your server a raw `http.IncomingMessage` (the request) and a raw `http.ServerResponse` (the response). Express **wraps and enhances** those raw objects with a rich, convenient API — `req.params`, `res.json()`, content negotiation, cookie helpers, and dozens more. This module is a tour of that enhanced API, but we will keep one foot on the raw HTTP foundations the whole way, because that is exactly where interview questions and real bugs live.

## What You'll Learn

By the end of this module you will be able to:

- Explain how Express's `req`/`res` relate to Node's raw `http.IncomingMessage`/`http.ServerResponse`, and why that matters.
- Read **every important property** of the request: `params`, `query`, `body`, `headers`, `cookies`, `ip`, `method`, `path`, `originalUrl`, `baseUrl`, `protocol`, `hostname`, `secure`, and `xhr`.
- Use the request **helper methods** `req.get()`, `req.accepts()`, and `req.is()` to inspect headers and negotiate content.
- Understand exactly **how `req.body` gets populated** by body-parsing middleware, and how to read a **raw body** when you need the original bytes.
- Send responses correctly with `res.send()`, `res.json()`, `res.jsonp()`, `res.status()`, `res.sendStatus()`, and `res.end()`.
- Perform **redirects** with `res.redirect()` and **render views** with `res.render()`.
- Manipulate response headers with `res.set`/`res.header`, `res.append`, `res.type`, and `res.vary`.
- Set and clear **cookies** with `res.cookie()`/`res.clearCookie()` and choose secure cookie options.
- Do **content negotiation** with `res.format`, and serve files with `res.download`, `res.sendFile`, and `res.attachment`.
- Apply **method chaining**, choose **correct status codes**, and share data with views and middleware via `res.locals`.

> **Version note:** Examples target **Node.js current LTS** and **Express 5** (the current major), but the request/response API is overwhelmingly stable across Express 4 and 5. Where a v4-vs-v5 difference is relevant (for example, the removal of `res.sendfile`/`res.json(status, body)` legacy signatures, or the behavior of `res.redirect('back')`), it is called out explicitly.

---

## 1. The Foundation: What `req` and `res` Actually Are

When a request hits your server, this is the chain of events:

1. Node's `http` module parses the incoming bytes into an **`http.IncomingMessage`** stream (the request).
2. It creates an **`http.ServerResponse`** writable stream (the response).
3. Express takes those two objects and sets their prototypes to **`express.request`** and **`express.response`**, which inherit from the Node prototypes. This is how Express adds methods like `req.param()` and `res.json()` *without* losing the underlying Node API.

```js
// app.js — proof that Express's objects ARE Node's objects, enhanced
const express = require("express");
const http = require("http");
const app = express();

app.get("/", (req, res) => {
  // req is still an http.IncomingMessage under the hood:
  console.log(req instanceof http.IncomingMessage); // true
  // res is still an http.ServerResponse:
  console.log(res instanceof http.ServerResponse); // true
  // ...but it also has Express's enhancements:
  console.log(typeof res.json); // 'function' (added by Express)
  res.json({ ok: true });
});

app.listen(3000, () => console.log("Listening on http://localhost:3000"));
```

**Why this matters:** because `req` is a readable stream and `res` is a writable stream, anything you know about Node streams still applies. You can pipe a file straight into `res`, you can read raw request bytes off `req`, and you must remember that the response can only be sent **once** — once the stream ends, the conversation is over.

> TIP: A useful mental model — `req` is **read-only input** (you inspect what the client sent), and `res` is **write-once output** (you build up the answer and flush it exactly one time). Almost every "Cannot set headers after they are sent" bug comes from forgetting that `res` is write-once.

---

## 2. The Request Object (`req`): Reading What the Client Sent

Let's walk through the properties you will reach for constantly. We'll use one example URL throughout so the values are concrete. Suppose a client requests:

```
GET https://api.example.com/api/v2/users/42/posts?sort=desc&limit=10#frag
```

...and this request is routed through a router mounted at `/api/v2`.

### 2.1 `req.params` — route parameters

`req.params` holds the **named segments** captured from the route path. They are always **strings**, regardless of what they look like.

```js
// Mounted such that the full path is /api/v2/users/:userId/posts
router.get("/users/:userId/posts", (req, res) => {
  console.log(req.params.userId); // '42'  (a STRING, not the number 42)
  // Convert explicitly when you need a number:
  const id = Number(req.params.userId);
  res.json({ userId: id });
});
```

> WARNING — Pitfall: Route params are **always strings**. `req.params.userId === 42` is `false`; it's `'42'`. Comparing or doing math without converting is a classic source of subtle bugs.

### 2.2 `req.query` — the parsed query string

`req.query` is the parsed `?key=value` portion of the URL. By default Express parses it into an **object**, and it can produce **nested objects and arrays** depending on the query-parser setting.

```js
// URL: /search?sort=desc&limit=10&tags=node&tags=express
app.get("/search", (req, res) => {
  console.log(req.query.sort);  // 'desc'
  console.log(req.query.limit); // '10'  (still a STRING!)
  console.log(req.query.tags);  // ['node', 'express']  (repeated keys → array)
  res.json(req.query);
});
```

Query values are **strings** (or arrays/objects of strings) — never numbers or booleans. `?active=true` gives you the string `'true'`, which is truthy even when you meant `false`.

> **v4 vs v5 note:** In Express 5, `req.query` is a **getter** and is effectively read-only — you cannot reassign `req.query = ...` the way you sometimes could in v4. Validate and copy into your own variable instead.

You can control how the query string is parsed:

```js
app.set("query parser", "simple"); // Node's querystring: no nesting, no arrays via brackets
app.set("query parser", "extended"); // default in many setups: uses 'qs', supports a[b]=c nesting
// Or pass a custom function for full control:
app.set("query parser", (str) => new URLSearchParams(str));
```

> WARNING — Pitfall: The `extended`/`qs` parser allows deeply nested objects from user input (e.g. `?a[b][c][d]=1`). This is a known DoS / prototype-pollution surface. Set sane limits or use `simple` when you don't need nesting.

### 2.3 `req.body` — the parsed request payload

`req.body` contains the parsed request payload — but it is **`undefined` until a body-parsing middleware runs**. This is the single most common "why is `req.body` undefined?" gotcha. We cover body parsing in depth in Section 4.

```js
app.use(express.json()); // populates req.body for application/json requests

app.post("/users", (req, res) => {
  console.log(req.body); // { name: 'Ada', email: 'ada@example.com' }
  res.status(201).json(req.body);
});
```

### 2.4 `req.headers` and `req.get()` — request headers

`req.headers` is a plain object of all request headers with **lowercased keys** (HTTP header names are case-insensitive; Node normalizes them to lowercase).

```js
app.get("/inspect", (req, res) => {
  console.log(req.headers["user-agent"]); // 'curl/8.4.0'
  console.log(req.headers["content-type"]); // 'application/json'
  res.end();
});
```

The idiomatic way to read **one** header is `req.get()` (alias `req.header()`), which is **case-insensitive** and handles the `Referer`/`Referrer` spelling quirk:

```js
req.get("Content-Type"); // 'application/json' (case doesn't matter)
req.get("content-type"); // same thing
req.get("referer");      // works
req.get("referrer");     // ALSO works — Express normalizes this historic misspelling
```

> BEST PRACTICE: Prefer `req.get('Header-Name')` over `req.headers['header-name']`. It's case-insensitive and self-documenting, and you won't trip over capitalization.

### 2.5 `req.cookies` — parsed cookies

`req.cookies` requires the **`cookie-parser`** middleware. Without it, `req.cookies` is `undefined`. With it, you get an object of name/value pairs; signed cookies land in `req.signedCookies`.

```js
const cookieParser = require("cookie-parser");
app.use(cookieParser("my-signing-secret")); // secret enables signed cookies

app.get("/dashboard", (req, res) => {
  console.log(req.cookies.theme);          // 'dark'  (unsigned)
  console.log(req.signedCookies.sessionId); // verified signed value, or undefined if tampered
  res.end();
});
```

### 2.6 `req.ip` and `req.ips` — the client address

`req.ip` is the remote IP address of the connection. Behind a proxy/load balancer (Heroku, Nginx, an ELB), the "real" client IP is in the `X-Forwarded-For` header — Express only honors that header when you enable **`trust proxy`**.

```js
app.set("trust proxy", true); // or a more specific value like 'loopback' or a subnet

app.get("/whereami", (req, res) => {
  console.log(req.ip);  // the client IP (uses X-Forwarded-For when trust proxy is on)
  console.log(req.ips); // ['client', 'proxy1', 'proxy2'] — the forwarding chain
  res.send(req.ip);
});
```

> WARNING — Pitfall: If you do rate-limiting or geo-blocking by IP but forget `trust proxy`, every request behind your load balancer will appear to come from the **proxy's** IP. Conversely, blindly trusting `X-Forwarded-For` lets clients spoof their IP. Set `trust proxy` to a **specific** trusted value, not blanket `true`, in production.

### 2.7 The URL anatomy: `method`, `path`, `originalUrl`, `baseUrl`

These four properties confuse almost everyone at first, so let's lay them out with our running example. Assume a router is mounted at `/api/v2` and contains a route `/users/:userId/posts`, and the client requested `/api/v2/users/42/posts?sort=desc`:

| Property | Value | Meaning |
| --- | --- | --- |
| `req.method` | `'GET'` | The HTTP verb (always uppercase). |
| `req.path` | `'/users/42/posts'` | Path **relative to the current handler**, no query string. |
| `req.baseUrl` | `'/api/v2'` | The mount path of the router/middleware currently handling the request. |
| `req.originalUrl` | `'/api/v2/users/42/posts?sort=desc'` | The **full, unmodified** URL as the client sent it, including query string. |
| `req.url` | `'/users/42/posts?sort=desc'` | The URL relative to the mount point (mutable; middleware may rewrite it). |

```js
const router = express.Router();
router.get("/users/:userId/posts", (req, res) => {
  res.json({
    method: req.method,         // 'GET'
    path: req.path,             // '/users/42/posts'
    baseUrl: req.baseUrl,       // '/api/v2'
    originalUrl: req.originalUrl, // '/api/v2/users/42/posts?sort=desc'
    url: req.url,               // '/users/42/posts?sort=desc'
    // Reconstruct the full path the handler matched:
    fullPath: req.baseUrl + req.path, // '/api/v2/users/42/posts'
  });
});
app.use("/api/v2", router);
```

> TIP: Rule of thumb — use **`req.originalUrl`** for logging (it never changes), **`req.path`** for routing logic inside a handler, and **`req.baseUrl`** when you need to know where the current router is mounted.

### 2.8 `req.protocol`, `req.hostname`, `req.secure`

These describe the URL scheme and host:

```js
app.get("/origin", (req, res) => {
  console.log(req.protocol); // 'http' or 'https'
  console.log(req.hostname); // 'api.example.com' (from the Host header)
  console.log(req.secure);   // true if req.protocol === 'https'

  // Reconstruct the origin reliably:
  const origin = `${req.protocol}://${req.get("host")}`;
  res.send(origin); // 'https://api.example.com'
});
```

Like `req.ip`, `req.protocol` and `req.secure` are **affected by `trust proxy`**. A TLS-terminating load balancer talks to your app over plain HTTP but sets `X-Forwarded-Proto: https`. Only with `trust proxy` enabled will `req.protocol` report `'https'` and `req.secure` be `true`.

> WARNING — Pitfall: A "force HTTPS" redirect middleware that checks `req.secure` will cause an **infinite redirect loop** behind a TLS-terminating proxy if you forgot to enable `trust proxy` — the app always thinks it's on HTTP.

### 2.9 `req.xhr` — was this an AJAX request?

`req.xhr` is `true` when the `X-Requested-With` header equals `XMLHttpRequest`. Historically jQuery and older AJAX libraries set this automatically.

```js
app.get("/data", (req, res) => {
  if (req.xhr) {
    res.json({ message: "Here's your JSON" }); // AJAX caller
  } else {
    res.render("data-page"); // full browser navigation
  }
});
```

> WARNING — Pitfall: `req.xhr` is **not reliable** on modern stacks. The browser `fetch()` API and `axios` do **not** set `X-Requested-With` by default. Treat `req.xhr` as a legacy hint, not a security boundary. For real content negotiation, use `req.accepts()` / `res.format()` (Sections 3 and 9).

---

## 3. Request Helper Methods: `get`, `accepts`, `is`

These three methods turn raw header strings into clean decisions.

### 3.1 `req.get(field)` / `req.header(field)`

Already shown above — the case-insensitive single-header reader. `req.header()` is an exact alias.

### 3.2 `req.accepts(types)` — "what does the client want back?"

The client sends an `Accept` header describing which response formats it can handle, with quality weights (e.g. `Accept: text/html, application/json;q=0.9`). `req.accepts()` reads that header and returns the **best match** from the types you offer, or `false` if none fit.

```js
app.get("/report", (req, res) => {
  // Offer several formats; Express picks the client's most-preferred one we support.
  switch (req.accepts(["json", "html", "text"])) {
    case "json":
      return res.json({ report: "data" });
    case "html":
      return res.send("<h1>Report</h1>");
    case "text":
      return res.type("txt").send("Report");
    default:
      // The client wants something we can't produce (e.g. Accept: image/png)
      return res.status(406).send("Not Acceptable");
  }
});
```

There are sibling methods for the other `Accept-*` headers: `req.acceptsCharsets()`, `req.acceptsEncodings()`, `req.acceptsLanguages()` — same idea, different dimension.

```js
app.get("/greeting", (req, res) => {
  const lang = req.acceptsLanguages(["en", "fr", "es"]) || "en";
  const messages = { en: "Hello", fr: "Bonjour", es: "Hola" };
  res.send(messages[lang]);
});
```

### 3.3 `req.is(type)` — "what did the client send me?"

Where `accepts` looks at the **`Accept`** header (what they want back), `req.is()` looks at the **`Content-Type`** header (what they sent). It returns the matched type string (truthy) or `false`.

```js
app.post("/upload", (req, res) => {
  if (req.is("application/json")) {
    // body is JSON
  } else if (req.is("multipart/*")) {
    // any multipart subtype (e.g. multipart/form-data)
  } else if (req.is("text/*")) {
    // any text type
  } else {
    return res.status(415).send("Unsupported Media Type");
  }
  res.end();
});
```

> TIP: Remember the pair: **`req.is()` = incoming `Content-Type`**, **`req.accepts()` = incoming `Accept`**. "Is this what you sent?" vs "Will you accept this back?"

---

## 4. How `req.body` Gets Populated (and Reading Raw Bodies)

This deserves its own section because it is the #1 beginner stumbling block and a frequent interview topic.

### 4.1 Why `req.body` starts as `undefined`

Remember that `req` is a **readable stream**. When the request arrives, Express has the *headers* immediately, but the **body has not been read yet** — it's still flowing in over the network as chunks. Express does **not** buffer and parse the body for you automatically, because doing so would waste memory on requests that don't need it (a `GET` has no body; a file upload might be gigabytes).

So `req.body` is `undefined` until a **body-parsing middleware** consumes the stream, parses it, and assigns the result to `req.body`.

### 4.2 The built-in parsers

Since Express 4.16, the common parsers ship **inside Express** (they're the `body-parser` package re-exported), so you usually don't need a separate dependency:

| Middleware | Parses `Content-Type` | Produces `req.body` as |
| --- | --- | --- |
| `express.json()` | `application/json` | a JS object/array |
| `express.urlencoded({ extended: true })` | `application/x-www-form-urlencoded` (HTML forms) | a JS object (nested with `extended:true`) |
| `express.text()` | `text/plain` | a string |
| `express.raw()` | `application/octet-stream` | a `Buffer` |

```js
const express = require("express");
const app = express();

// Order matters only relative to your routes — register parsers BEFORE the routes that need them.
app.use(express.json({ limit: "1mb" })); // cap payload size to prevent abuse
app.use(express.urlencoded({ extended: true, limit: "1mb" }));

app.post("/api/users", (req, res) => {
  // For a JSON request, req.body is the parsed object:
  res.status(201).json({ created: req.body });
});
```

A parser only acts on requests whose `Content-Type` matches. An `express.json()` middleware ignores a form-urlencoded request (leaving `req.body` as `{}`), which is why you typically register **both** `json()` and `urlencoded()`.

> WARNING — Pitfall: Forgetting `app.use(express.json())` is *the* reason `req.body` is `undefined`. The second most common reason: the client sent the wrong `Content-Type` (e.g. omitted `Content-Type: application/json`), so the JSON parser skipped it.

> BEST PRACTICE: Always set a `limit`. The default is `100kb`; without an explicit, deliberate limit an attacker can send a huge payload and exhaust memory. Match the limit to what each route legitimately needs.

### 4.3 File uploads are different

`multipart/form-data` (file uploads) is **not** handled by the built-in parsers. Use a dedicated library such as **`multer`**, which streams files to disk or memory and populates `req.file`/`req.files` plus `req.body` for the text fields.

```js
const multer = require("multer");
const upload = multer({ dest: "uploads/" });

app.post("/avatar", upload.single("avatar"), (req, res) => {
  console.log(req.file);  // metadata about the uploaded file
  console.log(req.body);  // any non-file form fields
  res.send("Uploaded");
});
```

### 4.4 Reading the raw body yourself

Sometimes you need the **original raw bytes** — most commonly to verify a **webhook signature** (Stripe, GitHub, Slack all sign the raw payload; if you let `express.json()` re-serialize it, the signature won't match).

Two clean approaches:

```js
// Approach A: express.raw() gives you a Buffer for a specific route/content-type
app.post(
  "/webhooks/stripe",
  express.raw({ type: "application/json" }), // req.body is a Buffer, untouched
  (req, res) => {
    const signature = req.get("Stripe-Signature");
    // verifySignature(req.body /* Buffer */, signature);
    const event = JSON.parse(req.body.toString("utf8"));
    res.json({ received: true });
  }
);
```

```js
// Approach B: capture the raw buffer DURING json parsing via the `verify` hook.
// Useful when you want both the parsed object AND the raw bytes.
app.use(
  express.json({
    verify: (req, res, buf, encoding) => {
      req.rawBody = buf; // stash the exact bytes for later signature checks
    },
  })
);
```

```js
// Approach C (no middleware): read the stream manually. Educational — shows what parsers do.
app.post("/raw", (req, res) => {
  const chunks = [];
  req.on("data", (chunk) => chunks.push(chunk)); // collect incoming bytes
  req.on("end", () => {
    const raw = Buffer.concat(chunks).toString("utf8");
    res.send(`Received ${raw.length} characters`);
  });
  req.on("error", () => res.status(400).end());
});
```

> WARNING — Pitfall: A body stream can only be **read once**. If `express.json()` already consumed it, your manual `req.on('data')` handler will never fire. For webhooks, mount the raw parser on the webhook route *before* (or instead of) the global JSON parser.

---

## 5. The Response Object (`res`): Sending the Answer

Now we switch to building and sending the response. The cardinal rule again: you may send **one** response per request. After you call a terminating method (`send`, `json`, `end`, `redirect`, `sendFile`, etc.), the response is committed.

### 5.1 `res.send()` — the smart, do-it-all sender

`res.send()` inspects its argument and does the right thing — sets a sensible `Content-Type`, sets `Content-Length`, and ends the response.

```js
res.send("Hello");               // Content-Type: text/html; charset=utf-8
res.send("<h1>Hi</h1>");         // also text/html
res.send({ user: "Ada" });       // Content-Type: application/json (object → JSON)
res.send([1, 2, 3]);             // application/json (array → JSON)
res.send(Buffer.from("raw"));    // Content-Type: application/octet-stream
res.send(404);                   // ⚠️ NOT a status code — see warning below
```

> WARNING — Pitfall: `res.send(404)` does **not** set the status to 404. In modern Express, passing a **number** is treated as a body. Use `res.sendStatus(404)` or `res.status(404).send(...)`. (Older Express versions even threw or behaved inconsistently here — never pass a bare number expecting a status.)

### 5.2 `res.json()` — explicit JSON

`res.json()` always serializes with `JSON.stringify`, sets `Content-Type: application/json`, and respects app settings like `json spaces` and `json replacer`. Use it whenever you intend to send JSON — it's clearer than relying on `res.send()`'s type-sniffing.

```js
res.json({ id: 1, name: "Ada" });          // {"id":1,"name":"Ada"}
res.json([{ a: 1 }, { b: 2 }]);            // arrays are fine
res.json(null);                             // null  (valid JSON!)

// Pretty-print in development:
app.set("json spaces", 2);
```

A key difference: `res.json()` will JSON-encode values that `res.send()` would not — notably `null` and bare numbers/booleans become proper JSON.

> **v4 vs v5 note:** The legacy two-argument form `res.json(status, body)` was **removed**. Always write `res.status(201).json(body)`.

### 5.3 `res.jsonp()` — JSON with padding (legacy)

`res.jsonp()` wraps the JSON in a function call for old cross-origin `<script>`-tag hacks (`?callback=fn` → `fn({...})`). It's largely obsolete now that **CORS** exists, but you should recognize it.

```js
app.set("jsonp callback name", "cb"); // default is 'callback'
app.get("/legacy", (req, res) => {
  res.jsonp({ data: 42 }); // /legacy?cb=handle → handle({"data":42});
});
```

> BEST PRACTICE: Don't reach for JSONP in new code — it bypasses the same-origin policy and has security baggage. Use CORS (the `cors` middleware) instead.

### 5.4 `res.status()` — set the code (chainable)

`res.status(code)` sets the HTTP status and **returns `res`**, so it chains.

```js
res.status(201).json({ created: true });
res.status(404).send("Not found");
res.status(204).end(); // 204 No Content — no body
```

### 5.5 `res.sendStatus()` — status + matching text body

`res.sendStatus(code)` sets the status **and** sends the standard reason phrase as a plain-text body, then ends. It's a shortcut for trivial responses.

```js
res.sendStatus(200); // → 200, body: "OK"
res.sendStatus(403); // → 403, body: "Forbidden"
res.sendStatus(404); // → 404, body: "Not Found"

// Equivalent longhand:
res.status(404).send("Not Found");
```

> TIP: Use `res.sendStatus()` for fire-and-forget acknowledgements (health checks, simple `204`/`200` replies). Use `res.status().json()` when you want a structured error body.

### 5.6 `res.end()` — terminate with no (or raw) body

`res.end()` comes straight from Node's `http.ServerResponse`. It ends the response without Express's content-type magic. Use it for empty responses or when you've already written everything.

```js
res.status(204).end();        // No Content
res.write("part 1");          // stream chunks manually...
res.write("part 2");
res.end();                    // ...then finish
```

> WARNING — Pitfall: `res.end()` does **not** set `Content-Type` or run JSON serialization. `res.end({ a: 1 })` will throw — `end()` only accepts strings/Buffers. Use `res.json()` for objects.

---

## 6. Redirects and Rendering Views

### 6.1 `res.redirect()`

`res.redirect([status,] path)` sends a redirect response (`Location` header + 3xx status). The default status is **302 Found**.

```js
res.redirect("/login");                 // 302 to /login
res.redirect(301, "/new-home");         // 301 Moved Permanently (SEO-friendly permanent move)
res.redirect("https://example.com");    // absolute URL is fine
res.redirect("../sibling");             // relative paths are resolved against the current URL
```

Common status choices: **301** (permanent), **302** (temporary, default), **303 See Other** (POST→GET after form submit, the PRG pattern), **307/308** (preserve the HTTP method).

```js
// Post/Redirect/Get: avoids duplicate form submission on refresh
app.post("/orders", (req, res) => {
  // ...create the order...
  res.redirect(303, "/orders/123"); // browser does a GET to the new resource
});
```

> **v4 vs v5 note:** `res.redirect('back')` (redirect to the `Referer`) was **deprecated/removed** in Express 5 because it trusts a client-controlled header. Do it explicitly and safely: `res.redirect(req.get('Referer') || '/')` — and validate the value to prevent open-redirect attacks.

> WARNING — Pitfall (open redirect): Never redirect to a raw user-supplied URL: `res.redirect(req.query.next)` lets an attacker send your users to a phishing site under your domain's trust. Whitelist allowed destinations or only allow same-origin relative paths.

### 6.2 `res.render()` — server-side templates

`res.render(view, locals)` renders a template (EJS, Pug, Handlebars, etc.) to HTML and sends it. You configure a view engine and a views directory first.

```js
app.set("view engine", "ejs");      // pick a template engine
app.set("views", "./views");        // where templates live

app.get("/profile/:id", async (req, res) => {
  const user = await getUser(req.params.id);
  // 'profile' → ./views/profile.ejs; second arg is the data ("locals")
  res.render("profile", { user, title: "Profile" });
});
```

Inside the template, both the `locals` you pass **and** anything on `res.locals` (Section 10) are available as variables.

```js
// res.render can also take a callback to get the rendered string instead of sending it:
res.render("email-template", { name: "Ada" }, (err, html) => {
  if (err) return next(err);
  sendEmail(html); // e.g. use the HTML somewhere else
  res.json({ sent: true });
});
```

---

## 7. Response Headers: `set`/`header`, `append`, `type`, `vary`

### 7.1 `res.set()` / `res.header()` — set headers

`res.set(field, value)` sets a response header; `res.header()` is an exact alias. Pass an object to set several at once. It **overwrites** any existing value for that header and returns `res` (chainable).

```js
res.set("Cache-Control", "no-store");
res.set("X-Powered-By", "Express-Course");

// Set multiple at once:
res.set({
  "Content-Type": "application/json",
  "X-Request-Id": req.id,
});

res.set("Cache-Control", "public, max-age=3600").json(data); // chains
```

To **read** a header you've set, use `res.get(field)`. To remove one, `res.removeHeader(field)`.

### 7.2 `res.append()` — add without overwriting

`res.append()` adds a value to a header, keeping any existing value (producing a multi-value header). Useful for headers that legitimately repeat, like `Set-Cookie`, `Link`, `Warning`, or `Vary`.

```js
res.append("Link", "<https://api.example.com/page2>; rel=\"next\"");
res.append("Link", "<https://api.example.com/page9>; rel=\"last\"");
// Two Link headers, both preserved.

res.append("Set-Cookie", "a=1");
res.append("Set-Cookie", "b=2"); // both cookies sent
```

> TIP: `res.set('X', v)` **replaces**; `res.append('X', v)` **adds**. Reach for `append` when a header can carry multiple values.

### 7.3 `res.type()` — shorthand for `Content-Type`

`res.type()` sets the `Content-Type` from a MIME type, a file extension, or a full type string. It returns `res`.

```js
res.type("json");                 // application/json
res.type("html");                 // text/html
res.type(".png");                 // image/png (extension lookup)
res.type("application/pdf");      // exact MIME
res.type("txt").send("plain");    // chains
```

### 7.4 `res.vary()` — manage the `Vary` header for caching

The `Vary` response header tells caches which **request** headers affect the response, so they don't serve the wrong cached variant. `res.vary(field)` safely adds a field without creating duplicates.

```js
app.get("/content", (req, res) => {
  res.vary("Accept-Language");          // response differs by language
  res.vary("Accept-Encoding");
  const lang = req.acceptsLanguages(["en", "fr"]) || "en";
  res.send(lang === "fr" ? "Bonjour" : "Hello");
});
```

> WARNING — Pitfall: Content negotiation **without** a correct `Vary` header causes cache poisoning — a CDN may serve the French page to an English user (or vice versa) because it cached the first response and ignored `Accept-Language`. Helpers like `res.format()` set `Vary: Accept` for you; for manual negotiation, set `Vary` yourself.

---

## 8. Cookies: `res.cookie()` and `res.clearCookie()`

`res.cookie(name, value, options)` sets a `Set-Cookie` header. The **options** are where security lives.

```js
res.cookie("theme", "dark"); // simplest form

res.cookie("sessionId", token, {
  httpOnly: true,                    // JS in the browser cannot read it (blocks XSS theft)
  secure: true,                      // only sent over HTTPS
  sameSite: "lax",                   // CSRF mitigation: 'strict' | 'lax' | 'none'
  maxAge: 1000 * 60 * 60 * 24,       // lifetime in MILLISECONDS (1 day)
  // expires: new Date(Date.now() + 86400000), // alternative to maxAge (a Date)
  path: "/",                         // scope to a path
  domain: ".example.com",            // share across subdomains
  signed: true,                      // sign it (requires cookie-parser with a secret)
});
```

Key options at a glance:

| Option | Purpose | Security note |
| --- | --- | --- |
| `httpOnly` | Hide cookie from `document.cookie` | **Always set for session/auth cookies** — defends against XSS exfiltration |
| `secure` | Only transmit over HTTPS | **On in production**; off in local dev over HTTP |
| `sameSite` | Restrict cross-site sending | `'lax'` is a good default; `'none'` **requires** `secure` |
| `maxAge` | Expiry as a duration (ms) | Note: **milliseconds**, unlike the raw `Max-Age` attribute which is seconds |
| `signed` | Tamper-evident value | Needs `cookieParser('secret')`; verify via `req.signedCookies` |

To delete a cookie, use `res.clearCookie(name, options)` — and the options (especially `path` and `domain`) must **match** how it was set, or the browser won't clear it.

```js
res.clearCookie("sessionId", { path: "/", httpOnly: true, secure: true });
res.json({ loggedOut: true });
```

> WARNING — Pitfall: `maxAge` in `res.cookie()` is **milliseconds**, but the underlying HTTP `Max-Age` attribute is **seconds**. Express does the conversion for you — passing `maxAge: 60` means **60 ms**, not 60 seconds. For one minute, write `maxAge: 60 * 1000`.

> BEST PRACTICE: For authentication cookies, set `httpOnly: true`, `secure: true` (in prod), and an explicit `sameSite`. Combine with `trust proxy` so `secure` cookies actually work behind a TLS-terminating proxy.

---

## 9. Content Negotiation and File Responses

### 9.1 `res.format()` — clean content negotiation

`res.format()` is the declarative version of the `req.accepts()` switch from Section 3. You give it an object keyed by MIME type / shorthand, and Express runs the handler matching the client's `Accept` header. It also sets `Vary: Accept` automatically.

```js
app.get("/users/:id", (req, res) => {
  const user = { id: req.params.id, name: "Ada" };
  res.format({
    "application/json": () => res.json(user),
    "text/html": () => res.send(`<h1>${user.name}</h1>`),
    "text/plain": () => res.send(`${user.name}`),
    default: () => res.status(406).send("Not Acceptable"), // none matched
  });
});
```

> TIP: `res.format()` automatically sends `406 Not Acceptable` if no key matches **and** you didn't supply a `default`. Supplying `default` lets you control that behavior.

### 9.2 `res.sendFile()` — send a file's contents

`res.sendFile(path, options)` streams a file to the client with the right `Content-Type` (guessed from the extension), `Content-Length`, `Last-Modified`, and ETag for caching. The path must be **absolute** unless you provide a `root` option.

```js
const path = require("path");

app.get("/manual", (req, res) => {
  // Absolute path is required (or use the root option):
  res.sendFile(path.join(__dirname, "files", "manual.pdf"));
});

app.get("/docs/:name", (req, res) => {
  res.sendFile(req.params.name, { root: path.join(__dirname, "docs") }, (err) => {
    if (err) {
      // sendFile reports 404s and stream errors here:
      res.status(err.statusCode || 404).end();
    }
  });
});
```

> WARNING — Pitfall (path traversal): Never concatenate user input into a file path without the `root` option and validation. `res.sendFile(req.params.name, { root })` with a name like `../../etc/passwd` is blocked by Express when `root` is set, but be defensive: reject `..`, null bytes, and absolute paths.

> **v4 vs v5 note:** The lowercase `res.sendfile()` (deprecated) was **removed** in Express 5. Use the camelCase `res.sendFile()`.

### 9.3 `res.download()` — force a file download

`res.download()` is `sendFile()` plus a `Content-Disposition: attachment` header, so the browser saves the file instead of displaying it. You can override the suggested filename.

```js
app.get("/export", (req, res) => {
  const file = path.join(__dirname, "reports", "q3.csv");
  res.download(file, "Q3-Report.csv", (err) => { // second arg = filename shown to user
    if (err && !res.headersSent) res.status(404).end();
  });
});
```

### 9.4 `res.attachment()` — set the disposition header only

`res.attachment([filename])` sets `Content-Disposition: attachment` (and, if you pass a filename, the `Content-Type` from its extension) **without** sending a file. Use it when you generate content on the fly and want the browser to download it.

```js
app.get("/generate-csv", (req, res) => {
  res.attachment("data.csv");        // Content-Disposition: attachment; filename="data.csv"
                                     // + Content-Type: text/csv
  res.send("id,name\n1,Ada\n2,Linus"); // body generated in memory
});
```

| Method | Sends a file? | Sets `Content-Disposition: attachment`? | Typical use |
| --- | --- | --- | --- |
| `res.sendFile()` | Yes | No (displays inline) | Serve an image/PDF to view in browser |
| `res.download()` | Yes | Yes | Offer an existing file for download |
| `res.attachment()` | No (you send body yourself) | Yes | Stream/generate content as a download |

---

## 10. Method Chaining, Status Codes, and `res.locals`

### 10.1 Method chaining

Most `res` setters return `res`, enabling fluent chains. The **terminating** method must come last.

```js
res
  .status(201)                                   // returns res
  .set("Location", `/users/${user.id}`)          // returns res
  .type("json")                                  // returns res
  .json(user);                                   // TERMINATES — must be last
```

Non-chainable (terminating) methods: `send`, `json`, `jsonp`, `end`, `sendStatus`, `redirect`, `render`, `sendFile`, `download`. Put exactly one of them at the end.

### 10.2 Choosing correct status codes

Status codes are part of your API contract. Returning `200` for an error is a real-world bug that breaks clients. A practical map:

| Code | Name | Use it when |
| --- | --- | --- |
| `200` | OK | Successful GET/PUT/PATCH with a body |
| `201` | Created | A POST created a resource (set `Location`) |
| `202` | Accepted | Async work queued, not yet done |
| `204` | No Content | Success with no body (e.g. DELETE) |
| `301` / `308` | Moved Permanently | Permanent redirect (308 keeps method) |
| `302` / `307` | Found / Temporary Redirect | Temporary redirect (307 keeps method) |
| `303` | See Other | Post/Redirect/Get after a form POST |
| `304` | Not Modified | Conditional GET; cached copy is fresh |
| `400` | Bad Request | Malformed input / validation failure |
| `401` | Unauthorized | Missing/invalid authentication |
| `403` | Forbidden | Authenticated but not allowed |
| `404` | Not Found | Resource doesn't exist |
| `409` | Conflict | Duplicate / version conflict |
| `415` | Unsupported Media Type | Body `Content-Type` you don't accept |
| `422` | Unprocessable Entity | Syntactically valid but semantically wrong |
| `429` | Too Many Requests | Rate limit exceeded |
| `500` | Internal Server Error | Unexpected server failure |
| `503` | Service Unavailable | Down for maintenance / overloaded |

> BEST PRACTICE: Distinguish **401 vs 403** ("who are you?" vs "you can't do that") and **400 vs 422** ("I can't parse this" vs "I parsed it but it's invalid"). Interviewers love these distinctions.

### 10.3 `res.locals` — per-request shared data

`res.locals` is an object scoped to the **current request/response cycle**. Anything you put on it is:

1. Available to all **downstream middleware and route handlers** for this request.
2. Automatically exposed as variables to **`res.render()`** templates.

It's the idiomatic way to pass computed data (the logged-in user, a CSP nonce, request-scoped flags) from middleware to handlers and views.

```js
// Middleware: attach the current user once, reuse everywhere
app.use(async (req, res, next) => {
  res.locals.currentUser = await getUserFromSession(req); // available downstream + in views
  res.locals.requestId = req.id;
  next();
});

app.get("/dashboard", (req, res) => {
  // No need to re-fetch — it's on res.locals:
  if (!res.locals.currentUser) return res.redirect("/login");
  res.render("dashboard"); // the template can use <%= currentUser.name %> directly
});
```

> TIP: Use **`res.locals`** for **per-request** data (it resets each request) and **`app.locals`** for **app-wide** constants (site name, helper functions) that are the same for every request.

> WARNING — Pitfall: Don't confuse `res.locals` (per-request, safe for user-specific data) with `app.locals` (shared across **all** requests). Putting a user's data on `app.locals` leaks it to every other user — a serious security bug.

---

## Common Pitfalls & Gotchas

- **`req.body` is `undefined`.** You forgot `app.use(express.json())` / `express.urlencoded()`, registered it *after* the route, or the client didn't send the matching `Content-Type`.
- **"Cannot set headers after they are sent to the client."** You called a terminating method twice (e.g. `res.send()` then `res.json()`, or didn't `return` after a redirect). Always `return res.send(...)` to stop further execution, or use `if/else`.
- **Everything is a string.** `req.params`, `req.query`, and form fields are strings. `Number()`/explicit parsing is required for math and strict comparisons.
- **`res.send(404)` is not a status.** It's a body. Use `res.sendStatus(404)` or `res.status(404).send(...)`.
- **`maxAge` is milliseconds in `res.cookie()`** but seconds in the raw HTTP attribute. `maxAge: 60` = 60 ms.
- **Forgetting `trust proxy`** breaks `req.ip`, `req.protocol`, `req.secure`, and `secure` cookies behind a load balancer — and can cause HTTPS-redirect loops.
- **Reading the body twice.** The request stream is consumed once; a parser and a manual `req.on('data')` can't both read it. Mount the raw parser before the JSON parser for webhooks.
- **Open redirects.** `res.redirect(req.query.next)` without validation lets attackers redirect your users off-site.
- **Path traversal in `sendFile`/`download`.** Always use the `root` option and reject `..` in user-supplied filenames.
- **Negotiating content without `Vary`.** Causes CDN cache poisoning. Set `Vary` (or use `res.format()`, which does it for you).
- **`req.xhr` is unreliable.** `fetch`/`axios` don't set `X-Requested-With`. Don't gate behavior on it.
- **`res.clearCookie` not clearing.** The `path`/`domain` must match the original `res.cookie` call.

---

## Best Practices

- **Register body parsers early, with explicit `limit`s** matched to each route's real needs.
- **Always set a status code deliberately.** Default 200 is a decision, not an accident; use 201/204/4xx/5xx where they belong.
- **Prefer `res.json()` over `res.send()`** when sending JSON — it's explicit and handles edge cases (`null`, numbers) correctly.
- **Use `req.get()` / `res.set()`** instead of poking at `req.headers` / `res.setHeader` directly — they handle case and Express semantics.
- **Lock down cookies:** `httpOnly`, `secure` (prod), explicit `sameSite`, and `signed` for anything sensitive.
- **Enable `trust proxy`** to a *specific* trusted value when running behind a proxy, then rely on `req.ip`/`req.secure`.
- **Validate and whitelist** before redirecting to, or serving files from, anything derived from user input.
- **Use `res.locals` for per-request data** shared between middleware and views; never put user data on `app.locals`.
- **Always `return` after terminating the response** to avoid double-send bugs.
- **Set `Vary`** whenever the response content depends on a request header (`Accept`, `Accept-Language`, `Accept-Encoding`).
- **Read raw bodies via `express.raw()` or the `verify` hook** for webhook signature verification — never re-serialize.

---

## Key Interview Points

- **What's the relationship between Express's `req`/`res` and Node's `http` objects?** Express enhances Node's `http.IncomingMessage`/`http.ServerResponse` by changing their prototypes; the Node API is still fully available. `req` is a readable stream, `res` a writable stream.
- **Why is `req.body` undefined?** No body-parsing middleware ran (e.g. missing `express.json()`), it was registered after the route, or the request's `Content-Type` didn't match the parser. The body is a stream Express doesn't auto-parse.
- **Difference between `req.params`, `req.query`, and `req.body`?** Params = named route segments; query = `?key=value` string; body = request payload (needs a parser). All params/query values are strings.
- **`req.path` vs `req.originalUrl` vs `req.baseUrl` vs `req.url`?** `originalUrl` = full unmodified URL (best for logging); `path` = path relative to the handler (no query); `baseUrl` = the router's mount path; `url` = mount-relative and mutable.
- **`req.accepts()` vs `req.is()`?** `accepts` reads the incoming **`Accept`** header (what the client wants back); `is` reads the incoming **`Content-Type`** (what the client sent).
- **`res.send()` vs `res.json()` vs `res.end()`?** `send` auto-detects type and sets Content-Length; `json` always stringifies and sets `application/json` (handles `null`/numbers); `end` is raw Node with no content-type magic.
- **`res.status()` vs `res.sendStatus()`?** `status()` only sets the code and is chainable; `sendStatus()` sets the code *and* sends the reason phrase as the body and terminates.
- **How do you reliably get the client's IP / know if the request is HTTPS behind a proxy?** Enable `trust proxy`; then `req.ip`, `req.ips`, `req.protocol`, and `req.secure` honor `X-Forwarded-*` headers.
- **How do you secure a session cookie?** `httpOnly` (blocks XSS theft), `secure` (HTTPS only), `sameSite` (CSRF mitigation), and `signed` for tamper detection.
- **What causes "Cannot set headers after they are sent"?** Sending the response more than once — usually a missing `return` before a second terminating call.
- **301 vs 302 vs 307/308?** 301/308 permanent, 302/307 temporary; 307/308 preserve the original HTTP method, 301/302 historically may not.
- **401 vs 403, and 400 vs 422?** 401 = not authenticated, 403 = authenticated but not allowed; 400 = unparseable/malformed, 422 = parsed but semantically invalid.
- **How do you verify a webhook signature in Express?** Read the *raw* body (`express.raw()` or the `json({ verify })` hook) so the bytes match what the sender signed; don't let `express.json()` re-serialize it.
- **`res.locals` vs `app.locals`?** `res.locals` = per-request (safe for user data, available to downstream middleware and views); `app.locals` = app-wide constants shared across all requests.

---

## Practice Exercises

Work through these in order; each builds on the last. Solutions follow.

1. **Echo inspector.** Build a `GET /inspect/:id` route that returns JSON containing the route param `id` (as a **number**), the full query object, the client's `User-Agent`, the client IP, and the `originalUrl`. Verify a value like `?active=true` shows up as the string `"true"`.

2. **Content negotiation.** Build `GET /widgets/:id` that returns the same widget data as JSON, HTML, or plain text depending on the client's `Accept` header, and responds `406` when none match. Implement it twice — once with a `req.accepts()` switch and once with `res.format()` — and explain which sets `Vary` for you.

3. **Secure login cookie + logout.** Build `POST /login` that sets an `httpOnly`, `secure`, `sameSite=lax`, signed `sessionId` cookie lasting exactly one hour, then redirects (PRG) to `/dashboard`. Build `POST /logout` that clears it correctly. Use `cookie-parser`.

4. **Webhook with raw body.** Build `POST /webhooks/demo` that reads the **raw** request body (not the parsed JSON), computes its byte length, and returns it — while a global `express.json()` is also installed for the rest of the app. Make sure the global parser doesn't consume the webhook body.

5. **File responses.** Build three routes: `GET /view/:file` that serves a file inline from a `public/` directory, `GET /download/:file` that forces a download with a friendly filename, and `GET /report.csv` that generates CSV in memory and sends it as an attachment. Guard against path traversal.

6. **Per-request user via `res.locals`.** Write middleware that reads a (faked) user from a header and stashes it on `res.locals.currentUser`. Then build `GET /me` that returns 401 if there's no user, otherwise returns the user — and a `GET /profile` HTML route that renders the user from `res.locals` without re-reading the header.

### Solutions

**Solution 1 — Echo inspector**

```js
const express = require("express");
const app = express();
app.set("trust proxy", true); // so req.ip respects X-Forwarded-For behind a proxy

app.get("/inspect/:id", (req, res) => {
  res.json({
    id: Number(req.params.id),        // params are strings → convert
    idType: typeof Number(req.params.id), // 'number'
    query: req.query,                 // e.g. { active: 'true' } — note the STRING
    activeIsString: req.query.active === "true", // true; it's the string "true"
    userAgent: req.get("User-Agent"), // case-insensitive header read
    ip: req.ip,
    originalUrl: req.originalUrl,
  });
});

app.listen(3000);
// Try: curl 'http://localhost:3000/inspect/42?active=true&tag=a&tag=b'
```

**Solution 2 — Content negotiation (both styles)**

```js
const widget = (id) => ({ id: Number(id), name: "Sprocket" });

// Style A: manual switch with req.accepts() — you must set Vary yourself.
app.get("/widgets/:id", (req, res) => {
  const w = widget(req.params.id);
  res.vary("Accept"); // critical for correct caching
  switch (req.accepts(["json", "html", "text"])) {
    case "json": return res.json(w);
    case "html": return res.send(`<h1>${w.name} #${w.id}</h1>`);
    case "text": return res.type("txt").send(`${w.name} #${w.id}`);
    default:     return res.status(406).send("Not Acceptable");
  }
});

// Style B: res.format() — sets Vary: Accept automatically, and sends 406 if no default.
app.get("/widgets2/:id", (req, res) => {
  const w = widget(req.params.id);
  res.format({
    "application/json": () => res.json(w),
    "text/html": () => res.send(`<h1>${w.name} #${w.id}</h1>`),
    "text/plain": () => res.type("txt").send(`${w.name} #${w.id}`),
    default: () => res.status(406).send("Not Acceptable"),
  });
});
// res.format() is preferable: it adds `Vary: Accept` for you, preventing cache poisoning.
```

**Solution 3 — Secure login cookie + logout**

```js
const cookieParser = require("cookie-parser");
app.use(cookieParser("super-secret-signing-key")); // secret enables signed cookies
app.use(express.urlencoded({ extended: true }));

const cookieOpts = {
  httpOnly: true,                // not readable by document.cookie
  secure: process.env.NODE_ENV === "production", // HTTPS-only in prod
  sameSite: "lax",               // CSRF mitigation
  signed: true,                  // tamper-evident
  maxAge: 60 * 60 * 1000,        // ONE HOUR in milliseconds (not 60*60!)
  path: "/",
};

app.post("/login", (req, res) => {
  const token = "demo-session-token"; // in reality: create a real session/JWT
  res.cookie("sessionId", token, cookieOpts);
  res.redirect(303, "/dashboard"); // PRG: 303 → browser GETs /dashboard
});

app.post("/logout", (req, res) => {
  // Options (path) must match how it was set, or the browser won't clear it:
  res.clearCookie("sessionId", { path: "/", httpOnly: true, sameSite: "lax" });
  res.redirect(303, "/login");
});
```

**Solution 4 — Webhook with raw body alongside global JSON**

```js
const app2 = express();

// Mount the RAW parser on the webhook route FIRST, before the global JSON parser,
// so the JSON parser never consumes the webhook's stream.
app2.post("/webhooks/demo", express.raw({ type: "*/*" }), (req, res) => {
  // req.body is a Buffer of the exact bytes the client sent:
  const raw = req.body; // Buffer
  res.json({
    byteLength: raw.length,
    preview: raw.toString("utf8").slice(0, 50),
    // In a real webhook you'd verify a signature against these exact bytes here.
  });
});

// Global JSON parser for the rest of the app (does not affect the route above,
// because that route already terminated with its own parser).
app2.use(express.json());

app2.post("/normal", (req, res) => res.json({ parsed: req.body }));
// Alternative: keep raw bytes during JSON parsing via the verify hook:
//   express.json({ verify: (req, _res, buf) => { req.rawBody = buf; } })
```

**Solution 5 — File responses with traversal protection**

```js
const path = require("path");
const PUBLIC = path.join(__dirname, "public");

// Reject anything that isn't a safe, single-segment filename.
function safeName(name) {
  if (name.includes("..") || name.includes("/") || name.includes("\0")) return null;
  return name;
}

app.get("/view/:file", (req, res) => {
  const name = safeName(req.params.file);
  if (!name) return res.status(400).send("Bad filename");
  // root option confines resolution to PUBLIC and blocks traversal:
  res.sendFile(name, { root: PUBLIC }, (err) => {
    if (err && !res.headersSent) res.status(err.statusCode || 404).end();
  });
});

app.get("/download/:file", (req, res) => {
  const name = safeName(req.params.file);
  if (!name) return res.status(400).send("Bad filename");
  const abs = path.join(PUBLIC, name);
  res.download(abs, `download-${name}`, (err) => { // friendly download name
    if (err && !res.headersSent) res.status(404).end();
  });
});

app.get("/report.csv", (req, res) => {
  res.attachment("report.csv");           // sets Content-Disposition + text/csv
  res.send("id,name\n1,Ada\n2,Linus\n");  // generated in memory, sent as a download
});
```

**Solution 6 — Per-request user via `res.locals`**

```js
app.set("view engine", "ejs");
app.set("views", "./views");

// Middleware: derive the user ONCE and share it for this request only.
app.use((req, res, next) => {
  const userId = req.get("X-User-Id"); // faked auth source
  res.locals.currentUser = userId ? { id: userId, name: `User ${userId}` } : null;
  next();
});

app.get("/me", (req, res) => {
  if (!res.locals.currentUser) {
    return res.status(401).json({ error: "Not authenticated" }); // 401 = no identity
  }
  res.json(res.locals.currentUser);
});

app.get("/profile", (req, res) => {
  if (!res.locals.currentUser) return res.redirect("/login");
  // The template can read currentUser directly — res.locals is auto-passed to views:
  res.render("profile"); // views/profile.ejs: <h1><%= currentUser.name %></h1>
});
// Try: curl -H 'X-User-Id: 7' http://localhost:3000/me
```

---

## Summary / Cheat Sheet

**Request (`req`) — read what the client sent:**

| Member | What it gives you |
| --- | --- |
| `req.params` | Route params (strings): `/users/:id` → `req.params.id` |
| `req.query` | Parsed `?key=value` (strings/arrays) |
| `req.body` | Parsed payload — **needs a body parser** |
| `req.headers` / `req.get(f)` | All headers (lowercased) / one header (case-insensitive) |
| `req.cookies` / `req.signedCookies` | Parsed cookies — **needs cookie-parser** |
| `req.ip` / `req.ips` | Client IP / forwarding chain (honors `trust proxy`) |
| `req.method` | HTTP verb |
| `req.path` | Path relative to handler (no query) |
| `req.baseUrl` | Router mount path |
| `req.originalUrl` / `req.url` | Full unmodified URL / mount-relative URL |
| `req.protocol` / `req.secure` / `req.hostname` | Scheme / is-HTTPS / host (honor `trust proxy`) |
| `req.xhr` | Was `X-Requested-With: XMLHttpRequest` (unreliable today) |
| `req.accepts(t)` | Best response type for the client's `Accept` |
| `req.is(t)` | Does the request's `Content-Type` match `t`? |

**Response (`res`) — build and send the answer (write once):**

| Member | What it does |
| --- | --- |
| `res.send(x)` | Smart sender (auto Content-Type) |
| `res.json(x)` | JSON body + `application/json` |
| `res.jsonp(x)` | JSONP (legacy; prefer CORS) |
| `res.status(c)` | Set status code (chainable) |
| `res.sendStatus(c)` | Set status + send reason text + end |
| `res.end()` | Raw terminate (no content-type magic) |
| `res.redirect([c,] url)` | 3xx redirect (default 302) |
| `res.render(view, locals)` | Render a template to HTML |
| `res.set/header(f, v)` | Set a header (overwrites) |
| `res.append(f, v)` | Add to a header (keeps existing) |
| `res.type(t)` | Set `Content-Type` shorthand |
| `res.vary(f)` | Add to `Vary` (cache correctness) |
| `res.cookie(n, v, opts)` / `res.clearCookie` | Set / clear a cookie |
| `res.format({...})` | Content negotiation (+ auto `Vary: Accept`) |
| `res.sendFile(p)` / `res.download(p)` / `res.attachment(n)` | Serve inline / force download / disposition only |
| `res.locals` | Per-request data for downstream middleware + views |

**Golden rules:**

- Params/query/form values are **strings** — convert them.
- `req.body` needs a **body parser** registered **before** the route.
- Enable **`trust proxy`** behind a load balancer for correct `ip`/`protocol`/`secure`.
- Send the response **once**; `return` after terminating to avoid double-send.
- Set the **right status code**; secure cookies with `httpOnly` + `secure` + `sameSite`.
- Use **raw bodies** for webhook signatures; set **`Vary`** when negotiating content.
