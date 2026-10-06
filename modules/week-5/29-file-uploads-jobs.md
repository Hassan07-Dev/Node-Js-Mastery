# File Uploads, Email & Background Jobs

So far your Express apps have exchanged small JSON documents: a client sends a few kilobytes of fields, you store them, you send a few kilobytes back. Real applications do messier, heavier work. Users upload profile pictures, PDFs, and CSVs. The system sends welcome emails, password-reset links, and receipts. Reports take ten seconds to generate, thumbnails take three seconds to render, and a bulk import touches fifty thousand rows. None of that fits the tidy "request in, JSON out, all in 40 milliseconds" model you have been living in.

This module is about the work that does **not** fit neatly inside a single fast request. It splits into three connected concerns. First, **binary uploads** — how a browser sends a file, how Express receives it (it can't, by default — you need Multer), where you put it, and how you serve it back safely. Second, **email** — how a Node process talks SMTP, sends HTML, and attaches files, plus the dev tooling that keeps you from spamming real inboxes. Third, and most important conceptually, **background jobs** — the discipline of moving slow, failure-prone, or bursty work *out* of the request/response cycle and into a separate process that can retry, throttle, and scale independently. We'll build a real job queue with BullMQ and Redis, schedule recurring work with node-cron, and design the request patterns (accept-then-process, status polling, webhooks) that let a fast API front a slow backend.

These three topics belong together because they are the first places a backend developer collides with the physical world: bytes on disk, packets to a mail server, seconds of CPU. Get them right and your app feels fast and reliable even when the work behind it is heavy.

## What You'll Learn

By the end of this module you will be able to:

- Explain how **`multipart/form-data`** encodes file uploads and why `express.json()`/`express.urlencoded()` cannot parse it.
- Use **Multer** for uploads: **disk vs. memory storage**, `single`/`array`/`fields`/`none`, `fileFilter`, `limits`, and robust error handling.
- **Validate** uploaded files by MIME type, extension, and magic bytes, and generate **safe, collision-free filenames** to avoid path-traversal and overwrite bugs.
- **Serve and stream** uploaded files efficiently, including HTTP **Range** requests for video/audio, and decide when to use `res.sendFile` vs. streams vs. `static`.
- Understand the **flow and tradeoffs** of uploading to **cloud storage** (Amazon S3, Cloudinary), including direct/pre-signed uploads.
- Send email with **Nodemailer**: SMTP transport config, plain/HTML bodies, templating, attachments, and dev tools like **Mailtrap** and Ethereal.
- Articulate **why slow work must leave the request cycle**, and the failure modes of doing otherwise.
- Build a production-style **job queue with BullMQ + Redis**: producers, workers, **concurrency**, **retries with backoff**, **delayed** and **repeatable** jobs, and events.
- Schedule recurring tasks with **node-cron** and know when cron vs. a queue is the right tool.
- Design API **patterns for long-running tasks**: accept-then-process (202), status polling, and webhook callbacks.

---

## 1. How File Uploads Actually Travel Over HTTP

Before any library, understand the wire format. When you submit an HTML form normally, the browser encodes the fields as `application/x-www-form-urlencoded` — basically a query string in the body: `name=Ada&age=36`. That is fine for text. It is hopeless for a 4 MB JPEG, because there is no clean, safe way to cram arbitrary binary bytes into a key=value string without massive bloat from percent-encoding.

So HTML forms that include `<input type="file">` switch to a different encoding: **`multipart/form-data`**. The body is split into **parts**, each separated by a randomly chosen **boundary** string declared in the `Content-Type` header. Each part has its own mini-headers describing the field name and, for files, the original filename and content type, followed by the raw bytes.

```http
POST /upload HTTP/1.1
Content-Type: multipart/form-data; boundary=----Boundary7MA4YWxkTrZu0gW

------Boundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="title"

My holiday photo
------Boundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="avatar"; filename="beach.jpg"
Content-Type: image/jpeg

<...raw binary JPEG bytes...>
------Boundary7MA4YWxkTrZu0gW--
```

Two consequences follow immediately:

1. **`express.json()` and `express.urlencoded()` cannot parse this.** Those middlewares understand JSON and urlencoded bodies only; they will leave `req.body` empty (or error) for a multipart request. You need a parser that understands boundaries and streams the binary parts somewhere. That parser is **Multer**.
2. **The body is a stream of potentially huge size.** A naive parser that buffers everything into memory can be DoS'd by a 2 GB upload. A good parser streams parts to disk (or to a bounded memory buffer) and enforces size limits.

The HTML side looks like this — note the two attributes that matter:

```html
<!-- enctype switches the encoding to multipart; method MUST be POST -->
<form action="/upload" method="POST" enctype="multipart/form-data">
  <input type="text" name="title" />
  <!-- `name` becomes the field name Multer keys on -->
  <input type="file" name="avatar" />
  <button type="submit">Upload</button>
</form>
```

> WARNING — Pitfall: Forgetting `enctype="multipart/form-data"` is the single most common upload bug. Without it the browser sends `application/x-www-form-urlencoded`, the file becomes just its filename text, and your server sees no file at all. The form *looks* correct; it silently sends nothing useful.

---

## 2. Multer: The Standard Upload Middleware

[**Multer**](https://github.com/expressjs/multer) is the Express team's middleware for handling `multipart/form-data`. It parses the request, writes file parts to your chosen storage, and exposes the results on `req.file` (single) or `req.files` (multiple), while putting any text fields on `req.body`. It is built on the battle-tested `busboy` streaming parser.

Install it:

```bash
npm install multer
```

### Disk storage vs. memory storage

Multer's most important decision is **where uploaded bytes go**. There are two built-in storage engines.

| | `diskStorage` | `memoryStorage` |
| --- | --- | --- |
| Where bytes land | A file on disk you name | A `Buffer` in RAM (`file.buffer`) |
| `req.file` has | `path`, `filename`, `destination` | `buffer` (the whole file in memory) |
| Best for | Files you'll keep locally; large files | Files you immediately forward (to S3, Sharp, a parser) and discard |
| Risk | Disk fills up; orphaned files | RAM blows up under big/concurrent uploads |
| Cleanup | You delete files yourself | GC frees the buffer once unreferenced |

Use **disk storage** when the file's final home is the local filesystem, or when files are large and you don't want them in RAM. Use **memory storage** when you will immediately pipe the bytes elsewhere — resize with `sharp`, parse a CSV, or upload to cloud storage — and never need them on the local disk. With memory storage, *always* set a strict size limit, because every concurrent upload holds its full size in RAM.

```js
// upload.js — configuring Multer with disk storage and safe filenames
const path = require('node:path');
const crypto = require('node:crypto');
const fs = require('node:fs');
const multer = require('multer');

// Ensure the destination exists at startup (Multer won't create it for you).
const UPLOAD_DIR = path.join(__dirname, 'uploads');
fs.mkdirSync(UPLOAD_DIR, { recursive: true });

const storage = multer.diskStorage({
  // `destination` chooses the folder. Can be sync (return) or async (cb).
  destination(req, file, cb) {
    cb(null, UPLOAD_DIR);
  },
  // `filename` chooses the stored name. We NEVER trust file.originalname here.
  filename(req, file, cb) {
    // Random, unguessable, collision-free base name + the real extension.
    const unique = crypto.randomBytes(16).toString('hex');
    const ext = path.extname(file.originalname).toLowerCase();
    cb(null, `${unique}${ext}`);
  },
});

module.exports = { storage, UPLOAD_DIR };
```

> WARNING — Pitfall: If you skip `filename`, Multer assigns a random name with **no extension**. If you instead naively use `file.originalname`, two users uploading `photo.jpg` overwrite each other, and a malicious `originalname` like `../../server.js` becomes a **path-traversal** attack. Always generate your own safe name; never feed `originalname` into a path.

### The upload field methods: single, array, fields, none

Once you have an instance (`const upload = multer({ storage })`), you attach it as **route-specific middleware** using one of these methods. Each tells Multer how many files and under which field name(s) to expect.

| Method | Expects | Populates |
| --- | --- | --- |
| `upload.single('avatar')` | One file in field `avatar` | `req.file` |
| `upload.array('photos', 5)` | Up to 5 files in field `photos` | `req.files` (array) |
| `upload.fields([{name:'avatar',maxCount:1},{name:'docs',maxCount:3}])` | Mixed named fields | `req.files` (object keyed by field) |
| `upload.none()` | No files, only text fields | `req.body` (errors if a file is sent) |
| `upload.any()` | Any files (avoid — see pitfall) | `req.files` (array) |

```js
// app.js — wiring routes to Multer methods
const express = require('express');
const { storage } = require('./upload');

const app = express();
const upload = multer({ storage });

// 1) Single file. After this middleware: req.file is the avatar, req.body has text fields.
app.post('/profile', upload.single('avatar'), (req, res) => {
  // req.file = { fieldname, originalname, mimetype, filename, path, size, ... }
  res.json({ saved: req.file?.filename, title: req.body.title });
});

// 2) Multiple files under one field name, capped at 5.
app.post('/gallery', upload.array('photos', 5), (req, res) => {
  res.json({ count: req.files.length, names: req.files.map((f) => f.filename) });
});

// 3) Different fields with different limits.
const cpUpload = upload.fields([
  { name: 'avatar', maxCount: 1 },
  { name: 'docs', maxCount: 3 },
]);
app.post('/onboard', cpUpload, (req, res) => {
  // req.files.avatar is an array (length 0 or 1); req.files.docs is an array.
  res.json({
    avatar: req.files.avatar?.[0]?.filename ?? null,
    docs: (req.files.docs ?? []).map((f) => f.filename),
  });
});
```

> WARNING — Pitfall: `upload.any()` accepts files under *any* field name. It is convenient and dangerous — an attacker can flood you with files you didn't expect. Prefer the explicit `single`/`array`/`fields` forms so the shape of the request is whitelisted.

> TIP: Multer parses text fields **in stream order**. A text field that appears in the multipart body *after* a file is not yet on `req.body` while the file is being processed. If your `fileFilter` or `filename` callback needs a form field (e.g. a folder name), put that field *before* the file input in the HTML form, or accept that it may be undefined during filtering.

### File filtering: rejecting wrong types early

The `fileFilter` option runs once per file before bytes are stored. Call `cb(null, true)` to accept, `cb(null, false)` to silently skip, or `cb(new Error(...))` to reject the whole request. This is your first line of type validation — but note it only sees the **declared** `mimetype` and `originalname`, both of which the client controls and can lie about. Treat it as a cheap pre-filter, not real security (Section 4 covers magic-byte checks).

```js
// Accept only common image types by both mimetype AND extension.
const ALLOWED_MIME = new Set(['image/jpeg', 'image/png', 'image/webp', 'image/gif']);
const ALLOWED_EXT = new Set(['.jpg', '.jpeg', '.png', '.webp', '.gif']);

function imageFilter(req, file, cb) {
  const ext = path.extname(file.originalname).toLowerCase();
  const okMime = ALLOWED_MIME.has(file.mimetype);
  const okExt = ALLOWED_EXT.has(ext);
  if (okMime && okExt) return cb(null, true);
  // Reject the request with a typed error we can catch in our error handler.
  const err = new Error('Only JPEG/PNG/WebP/GIF images are allowed');
  err.code = 'INVALID_FILE_TYPE';
  cb(err);
}

const upload = multer({ storage, fileFilter: imageFilter });
```

### Size and count limits

The `limits` option is your DoS shield. Set it even on small apps — without `fileSize`, a single client can stream gigabytes into your memory or disk.

```js
const upload = multer({
  storage,
  fileFilter: imageFilter,
  limits: {
    fileSize: 5 * 1024 * 1024, // 5 MB per file (bytes)
    files: 8,                  // max number of file parts
    fields: 20,                // max number of non-file fields
    fieldNameSize: 100,        // max field name length
    fieldSize: 1 * 1024 * 1024 // max non-file field value size
  },
});
```

When a limit is exceeded, Multer aborts and throws a `MulterError` (e.g. `LIMIT_FILE_SIZE`). Crucially, with disk storage Multer may have **already written part of the file** before hitting the limit — it cleans up files it wrote for the *failed* request, but you should still verify cleanup in your own error path for safety.

### Handling Multer errors properly

Multer surfaces errors two ways: a `MulterError` (limits, unexpected field) or whatever Error your `fileFilter` passed. Both arrive as the `err` argument to Express error-handling middleware (a `(err, req, res, next)` function) or, if you call Multer manually, as the callback error. Handle them explicitly and return clean 4xx codes.

```js
const { MulterError } = require('multer');

// Error-handling middleware — must have FOUR args to be recognized by Express.
app.use((err, req, res, next) => {
  if (err instanceof MulterError) {
    // Built-in upload errors: LIMIT_FILE_SIZE, LIMIT_FILE_COUNT, LIMIT_UNEXPECTED_FILE, ...
    return res.status(413).json({ error: `Upload error: ${err.code}`, field: err.field });
  }
  if (err && err.code === 'INVALID_FILE_TYPE') {
    return res.status(415).json({ error: err.message }); // 415 Unsupported Media Type
  }
  next(err); // not ours — pass along
});
```

> BEST PRACTICE: Put the Multer middleware on the *specific route* that needs it, not globally with `app.use(upload...)`. A global upload parser would try to parse every request as multipart, breaking your JSON endpoints and exposing every route to upload-based attacks.

A complete, self-contained upload server:

```js
// server.js — minimal but production-shaped upload endpoint
const express = require('express');
const path = require('node:path');
const crypto = require('node:crypto');
const fs = require('node:fs');
const multer = require('multer');
const { MulterError } = multer;

const app = express();
const UPLOAD_DIR = path.join(__dirname, 'uploads');
fs.mkdirSync(UPLOAD_DIR, { recursive: true });

const storage = multer.diskStorage({
  destination: (req, file, cb) => cb(null, UPLOAD_DIR),
  filename: (req, file, cb) =>
    cb(null, crypto.randomBytes(16).toString('hex') + path.extname(file.originalname).toLowerCase()),
});

const ALLOWED = new Set(['image/jpeg', 'image/png', 'image/webp']);
const upload = multer({
  storage,
  limits: { fileSize: 5 * 1024 * 1024, files: 4 },
  fileFilter: (req, file, cb) =>
    ALLOWED.has(file.mimetype) ? cb(null, true) : cb(Object.assign(new Error('bad type'), { code: 'INVALID_FILE_TYPE' })),
});

app.post('/upload', upload.single('image'), (req, res) => {
  if (!req.file) return res.status(400).json({ error: 'No file uploaded' });
  res.status(201).json({ url: `/files/${req.file.filename}`, size: req.file.size });
});

app.use((err, req, res, next) => {
  if (err instanceof MulterError) return res.status(413).json({ error: err.code });
  if (err?.code === 'INVALID_FILE_TYPE') return res.status(415).json({ error: err.message });
  next(err);
});

app.listen(3000, () => console.log('http://localhost:3000'));
```

---

## 3. Serving and Streaming Uploaded Files

Receiving files is half the job; delivering them back is the other half. There are three tools, in increasing order of control.

### Option A: `express.static` for a public folder

The simplest path. Mount a directory and Express serves any file under it, with correct `Content-Type`, `ETag`, `Last-Modified`, and conditional-request handling for free.

```js
// Serve everything in ./uploads under the /files URL prefix.
// A file stored as uploads/ab12.jpg is reachable at /files/ab12.jpg
app.use('/files', express.static(path.join(__dirname, 'uploads'), {
  maxAge: '7d',        // cache header for browsers/CDN
  index: false,        // don't serve directory listings
  dotfiles: 'ignore',  // never serve .env etc.
}));
```

This is perfect for **public** assets. It is wrong for **private** files, because anyone who guesses (or is shown) the URL gets the bytes — there is no auth check.

### Option B: `res.sendFile` for controlled, authorized delivery

When you need to check permissions, look up a DB record, or serve from a path the client never sees, route the request through a handler and use `res.sendFile`. It still streams efficiently and sets headers, but *you* decide who gets in.

```js
const fsp = require('node:fs/promises');

app.get('/download/:id', authRequired, async (req, res, next) => {
  const fileRecord = await db.files.findById(req.params.id); // your lookup
  if (!fileRecord) return res.status(404).json({ error: 'Not found' });

  // Authorization: only the owner may download.
  if (fileRecord.ownerId !== req.user.id) return res.status(403).json({ error: 'Forbidden' });

  const absPath = path.join(UPLOAD_DIR, fileRecord.storedName);

  // Force a download with the ORIGINAL human name, not the random stored name.
  res.download(absPath, fileRecord.originalName, (err) => {
    // res.download is res.sendFile + Content-Disposition: attachment
    if (err && !res.headersSent) next(err);
  });
});
```

> WARNING — Pitfall: Never build a file path by concatenating user input directly: `path.join(UPLOAD_DIR, req.params.name)` where `name` is `../../etc/passwd` escapes your directory. Always look up an internal record by an opaque id and map it to a stored name you control, or sanitize with `path.basename()` and re-check the resolved path stays inside `UPLOAD_DIR`.

### Option C: Manual streaming with Range support (video/audio)

For large media you want the browser to **stream** and **seek** — that requires honoring the HTTP `Range` header. The browser sends `Range: bytes=1000000-`, and you respond `206 Partial Content` with just that slice. `express.static` does this automatically; if you stream manually, you must implement it.

```js
app.get('/video/:name', (req, res) => {
  const filePath = path.join(UPLOAD_DIR, path.basename(req.params.name)); // basename() blocks traversal
  let stat;
  try { stat = fs.statSync(filePath); } catch { return res.sendStatus(404); }

  const range = req.headers.range;
  if (!range) {
    // No range: stream the whole file.
    res.writeHead(200, { 'Content-Length': stat.size, 'Content-Type': 'video/mp4' });
    return fs.createReadStream(filePath).pipe(res);
  }

  // Parse "bytes=START-END"
  const [startStr, endStr] = range.replace(/bytes=/, '').split('-');
  const start = parseInt(startStr, 10);
  const end = endStr ? parseInt(endStr, 10) : stat.size - 1;

  if (start >= stat.size || end >= stat.size) {
    return res.writeHead(416, { 'Content-Range': `bytes */${stat.size}` }).end();
  }

  res.writeHead(206, {
    'Content-Range': `bytes ${start}-${end}/${stat.size}`,
    'Accept-Ranges': 'bytes',
    'Content-Length': end - start + 1,
    'Content-Type': 'video/mp4',
  });
  // Stream just the requested byte window — never loads the whole file into memory.
  fs.createReadStream(filePath, { start, end }).pipe(res);
});
```

> TIP: `fs.createReadStream(...).pipe(res)` is the canonical way to send a file without loading it into RAM. Streams apply backpressure automatically: if the client reads slowly, Node pauses the disk read. Reading the whole file with `fs.readFileSync` and then `res.send`-ing it works for tiny files but will exhaust memory under load.

### Generating safe filenames — the rules

Filename safety deserves its own checklist because nearly every upload vulnerability lives here:

- **Never trust `originalname`** for the stored path. Use a random id (`crypto.randomBytes`, `crypto.randomUUID()`) as the base name.
- **Keep the extension** only if you validated it against an allowlist; lowercase it.
- **Store the original name separately** (in the DB) so you can offer it as the download name without using it on disk.
- **Reject empty, dotfile, and reserved names** (`.`, `..`, Windows `CON`/`NUL`, names with `/` or `\`).
- **Verify the resolved absolute path** still starts with your upload directory before reading/writing.

```js
// Belt-and-suspenders path safety check used before any fs operation.
function resolveInsideDir(baseDir, candidateName) {
  const safeName = path.basename(candidateName);           // strip any directory parts
  const resolved = path.resolve(baseDir, safeName);
  if (!resolved.startsWith(path.resolve(baseDir) + path.sep)) {
    throw new Error('Path traversal attempt blocked');
  }
  return resolved;
}
```

---

## 4. Validating File Type for Real (Magic Bytes)

The `mimetype` Multer reports comes from the browser's `Content-Type` header for that part, and the extension comes from the filename — **both are client-supplied and trivially spoofed.** An attacker can rename `malware.html` to `cat.png` and set `Content-Type: image/png`. Your `fileFilter` waves it through. If you then serve it back with the wrong headers, a browser may execute it (stored XSS).

Real validation inspects the file's **magic bytes** — the signature in the first few bytes that identifies the true format (JPEG starts with `FF D8 FF`, PNG with `89 50 4E 47`, PDF with `25 50 44 46`). The library [`file-type`](https://github.com/sindresorhus/file-type) reads these for you.

```bash
npm install file-type
```

With **memory storage** you have the buffer directly:

```js
const { fileTypeFromBuffer } = require('file-type'); // ESM-only; in CJS use dynamic import

const upload = multer({ storage: multer.memoryStorage(), limits: { fileSize: 5 * 1024 * 1024 } });

app.post('/avatar', upload.single('image'), async (req, res) => {
  if (!req.file) return res.status(400).json({ error: 'No file' });

  // Inspect actual bytes, ignore the claimed mimetype entirely.
  const type = await fileTypeFromBuffer(req.file.buffer);
  const allowed = new Set(['image/jpeg', 'image/png', 'image/webp']);
  if (!type || !allowed.has(type.mime)) {
    return res.status(415).json({ error: 'File content is not a supported image' });
  }

  // Safe to persist now: write to disk / S3 using type.ext for the extension.
  const name = `${crypto.randomUUID()}.${type.ext}`;
  await fsp.writeFile(path.join(UPLOAD_DIR, name), req.file.buffer);
  res.status(201).json({ url: `/files/${name}` });
});
```

> BEST PRACTICE: Defense in depth for uploads — (1) extension allowlist, (2) declared-mimetype allowlist in `fileFilter`, (3) **magic-byte check** on the actual bytes, (4) size limit, (5) store with a random name and **no executable permissions**, (6) serve with `Content-Type` you set yourself plus `Content-Disposition: attachment` and `X-Content-Type-Options: nosniff` so browsers don't sniff and execute. For user-generated images, re-encoding them through `sharp` also strips any embedded payload.

> WARNING — Pitfall: `file-type` is an **ESM-only** package in recent versions. In a CommonJS project you cannot `require('file-type')` directly; use a dynamic import: `const { fileTypeFromBuffer } = await import('file-type');` inside an async function.

---

## 5. Uploading to Cloud Storage (S3 / Cloudinary)

Storing uploads on your own server's disk works until it doesn't. Local disk is **ephemeral** on most modern hosting (Heroku, containers, serverless): a redeploy or restart wipes it. It doesn't scale horizontally — three app instances behind a load balancer each have their own disk, so a file uploaded to instance A is invisible to instance B. And serving large files from your app server wastes its CPU and bandwidth. The standard answer is **object storage**: a dedicated service (Amazon S3, Google Cloud Storage, Cloudflare R2) or a media-specialized service (Cloudinary, imgix) that stores blobs by key and serves them over a CDN.

### The two upload flows

**Flow 1 — proxy upload (file passes through your server):** The browser uploads to your Express endpoint; you receive it (memory storage), then forward the bytes to S3 with the SDK. Simple, keeps credentials server-side, lets you validate/transform before storing. The cost: every byte traverses your server twice (in, then out), consuming its bandwidth and memory.

```js
// Proxy upload to S3 with the AWS SDK v3.
const { S3Client, PutObjectCommand } = require('@aws-sdk/client-s3');
const s3 = new S3Client({ region: process.env.AWS_REGION }); // creds from env/role

app.post('/s3-upload', upload.single('file'), async (req, res) => {
  const key = `uploads/${crypto.randomUUID()}-${path.basename(req.file.originalname)}`;
  await s3.send(new PutObjectCommand({
    Bucket: process.env.S3_BUCKET,
    Key: key,
    Body: req.file.buffer,            // memoryStorage gives us the Buffer
    ContentType: req.file.mimetype,
  }));
  res.status(201).json({ key });
});
```

**Flow 2 — direct upload with a pre-signed URL (file bypasses your server):** Your server generates a short-lived, cryptographically signed URL that grants permission to `PUT` one specific object. The browser uploads **directly to S3** using that URL; the bytes never touch your server. This is how you handle large files at scale — your app does milliseconds of work (signing) instead of holding a 500 MB upload.

```js
// Server: mint a pre-signed PUT URL valid for 60 seconds.
const { getSignedUrl } = require('@aws-sdk/s3-request-presigner');

app.post('/sign-upload', authRequired, async (req, res) => {
  const key = `uploads/${req.user.id}/${crypto.randomUUID()}`;
  const command = new PutObjectCommand({
    Bucket: process.env.S3_BUCKET,
    Key: key,
    ContentType: req.body.contentType, // lock the type the client must use
  });
  const url = await getSignedUrl(s3, command, { expiresIn: 60 });
  res.json({ uploadUrl: url, key }); // client PUTs the file to uploadUrl, then tells us `key`
});
```

```js
// Browser: upload straight to S3, no server in the byte path.
await fetch(uploadUrl, { method: 'PUT', headers: { 'Content-Type': file.type }, body: file });
```

### S3 vs. Cloudinary at a glance

| | Amazon S3 (object storage) | Cloudinary (media platform) |
| --- | --- | --- |
| Stores | Any blob, by key | Images/video, with metadata |
| Transforms | None (you bring `sharp`/Lambda) | On-the-fly resize/crop/format via URL |
| CDN | Pair with CloudFront | Built in |
| Pricing model | Storage + egress + requests | Transformations + storage + bandwidth |
| Best for | General files, control, lowest storage cost | Image/video heavy apps wanting zero infra |

> BEST PRACTICE: For anything user-facing and large, prefer **pre-signed direct uploads**. Record the resulting object key in your database and have a background job (Section 8) verify the object exists and generate derivatives (thumbnails, transcodes) asynchronously. Keep your buckets **private** and serve via signed read URLs or a CDN — public buckets are a recurring source of data leaks.

---

## 6. Sending Email with Nodemailer

Email is how your app reaches users outside itself: verification links, password resets, receipts, notifications. Under the hood, sending email means speaking **SMTP** (Simple Mail Transfer Protocol) to a mail server. You almost never implement SMTP yourself — [**Nodemailer**](https://nodemailer.com) is the de facto Node library, with zero runtime dependencies and broad support for SMTP, attachments, HTML, and OAuth2.

```bash
npm install nodemailer
```

### The transport: how you connect to a mail server

A **transport** is the connection to whatever actually delivers mail. In production you do **not** run your own SMTP server — you use a provider (SendGrid, Amazon SES, Postmark, Mailgun, Resend) and point Nodemailer at their SMTP host (or use their API). The transport holds the host, port, and credentials.

```js
const nodemailer = require('nodemailer');

const transporter = nodemailer.createTransport({
  host: process.env.SMTP_HOST,       // e.g. smtp.sendgrid.net
  port: Number(process.env.SMTP_PORT), // 587 (STARTTLS) or 465 (implicit TLS)
  secure: Number(process.env.SMTP_PORT) === 465, // true only for port 465
  auth: {
    user: process.env.SMTP_USER,
    pass: process.env.SMTP_PASS,     // NEVER hardcode; load from env/secrets
  },
});

// Verify the connection/credentials at startup — fail fast if misconfigured.
transporter.verify()
  .then(() => console.log('SMTP ready'))
  .catch((err) => console.error('SMTP config error:', err.message));
```

> WARNING — Pitfall: The `secure` flag confuses everyone. Port **465** uses TLS from the first byte → `secure: true`. Port **587** starts plaintext and upgrades via STARTTLS → `secure: false` (Nodemailer upgrades automatically). Setting `secure: true` on port 587 produces a cryptic connection hang.

### Sending a message

`sendMail` takes a message object and returns a promise resolving to a result with the provider's `messageId`. The same call supports plain text, HTML, and attachments.

```js
async function sendWelcome(to, name) {
  const info = await transporter.sendMail({
    from: '"Acme App" <no-reply@acme.com>', // display name + address
    to,                                      // string or comma-list or array
    subject: 'Welcome to Acme',
    text: `Hi ${name}, welcome aboard!`,     // plaintext fallback (deliverability!)
    html: `<h1>Welcome, ${escapeHtml(name)}!</h1><p>We're glad you're here.</p>`,
    replyTo: 'support@acme.com',
  });
  console.log('Sent:', info.messageId);
  return info;
}
```

> BEST PRACTICE: Always include a `text` version alongside `html`. Some clients prefer plaintext, spam filters penalize HTML-only mail, and a text fallback dramatically improves deliverability. And always **escape user-provided values** interpolated into HTML emails — email is HTML, so the same XSS/injection concerns apply.

### HTML templates

Hardcoding HTML strings does not scale past two emails. Use a templating engine to keep content out of code. Two common approaches:

```js
// Approach 1: a tiny template engine (Handlebars) compiled once, reused per send.
const fs = require('node:fs');
const handlebars = require('handlebars');

const source = fs.readFileSync(path.join(__dirname, 'emails/welcome.hbs'), 'utf8');
const welcomeTemplate = handlebars.compile(source); // compile once at startup

async function sendWelcomeTemplated(to, data) {
  const html = welcomeTemplate(data); // { name, ctaUrl, ... }
  return transporter.sendMail({ from: 'no-reply@acme.com', to, subject: 'Welcome', html });
}
```

For richer needs the `email-templates` package pairs a template engine with automatic text generation and inlined CSS (email clients need inline styles, not `<style>` blocks). For React shops, `react-email` renders JSX to email-safe HTML.

### Attachments

Attachments are an array on the message. You can attach a local path, a Buffer, a stream, or a remote URL, and embed images inline via `cid`.

```js
await transporter.sendMail({
  from: 'no-reply@acme.com',
  to: 'customer@example.com',
  subject: 'Your invoice',
  html: 'Invoice attached. Logo: <img src="cid:logo123" />',
  attachments: [
    { filename: 'invoice.pdf', path: path.join(__dirname, 'invoices/abc.pdf') }, // from disk
    { filename: 'data.csv', content: 'a,b\n1,2\n' },                              // from string/Buffer
    { filename: 'report.pdf', content: pdfStream },                              // from a stream
    { filename: 'logo.png', path: './logo.png', cid: 'logo123' },               // inline (cid match)
  ],
});
```

### Dev email tools: Mailtrap and Ethereal

You must **never** send test emails to real addresses — you'll spam users and harm your domain's reputation. Two tools solve this:

- **Ethereal** — a free fake SMTP service built into Nodemailer. Mail is *captured*, never delivered, and you get a preview URL. Zero signup.
- **Mailtrap** — a hosted "email sandbox" that catches all mail in a shared inbox the team can inspect, with spam-score and HTML/CSS analysis.

```js
// Ethereal: generate throwaway creds and a preview link at runtime.
async function devTransport() {
  const testAccount = await nodemailer.createTestAccount();
  const t = nodemailer.createTransport({
    host: 'smtp.ethereal.email',
    port: 587,
    secure: false,
    auth: { user: testAccount.user, pass: testAccount.pass },
  });
  return t;
}

// After sending in dev, log the preview URL:
const info = await transporter.sendMail({ /* ... */ });
console.log('Preview:', nodemailer.getTestMessageUrl(info)); // opens the captured email
```

> BEST PRACTICE: Switch transports by environment. In dev → Ethereal/Mailtrap; in production → your real provider, configured purely through environment variables so no code changes between environments. And because `sendMail` is a slow network call that can fail, do it in a **background job** (Section 8), not inline in the request — a flaky SMTP server should never make your signup endpoint hang or 500.

---

## 7. Why Slow Work Must Leave the Request Cycle

This is the conceptual heart of the module. Node.js runs your JavaScript on a **single thread** with an event loop. A web request flows: client connects → your handler runs → you send a response → connection frees up. The whole model depends on each handler finishing **quickly** and yielding control. When you do slow work *inside* a request handler, several bad things happen at once.

Consider a signup endpoint that, inline, (1) writes the user, (2) generates a PDF welcome packet, (3) sends an email, (4) calls a third-party CRM:

```js
// ANTI-PATTERN: everything inline in the request.
app.post('/signup', async (req, res) => {
  const user = await createUser(req.body);     // 30ms
  await generateWelcomePdf(user);              // 4000ms  (CPU + disk)
  await sendWelcomeEmail(user);                // 800ms   (network, can fail)
  await syncToCrm(user);                       // 1500ms  (third party, often down)
  res.json({ ok: true });                      // user waited ~6.3 SECONDS
});
```

The problems:

1. **Terrible latency.** The user stares at a spinner for 6+ seconds for an action whose *meaningful* part (creating the account) took 30 ms.
2. **Coupled failure.** If the CRM is down, signup **fails entirely** — even though the account was created. The user sees a 500 and may retry, creating duplicates.
3. **No retries.** A transient email blip means the welcome email is simply lost. There's no mechanism to try again later.
4. **Event-loop starvation (for CPU work).** PDF generation, image processing, and crypto are CPU-bound. While that runs synchronously, the **single thread is blocked** and *every other request* on this process waits. A few concurrent PDF generations and your whole server stalls.
5. **No throttling.** If the CRM allows 10 req/s and you get a signup spike, you'll hammer past its rate limit and get banned.

The fix is to **return fast and do the slow work elsewhere**. The request handler does only the essential, fast, must-succeed part (create the user), then **enqueues** the rest as background jobs and responds immediately:

```js
// PATTERN: enqueue slow work, respond immediately.
app.post('/signup', async (req, res) => {
  const user = await createUser(req.body);          // must succeed, fast
  await emailQueue.add('welcome', { userId: user.id });   // enqueue (ms)
  await crmQueue.add('sync', { userId: user.id });        // enqueue (ms)
  res.status(202).json({ id: user.id });            // ~35ms total
});
```

A separate **worker** process pulls jobs off the queue and runs them with retries, backoff, and concurrency limits — and if the worker crashes, the jobs are still in the queue, ready to be retried. This decoupling is the difference between a toy and a production system.

> TIP: A simple `setTimeout(doWork, 0)` or "fire and forget" `doWork().catch(...)` *moves work off the response* but is **not** a background job system. It has no persistence (a restart loses the work), no retries, no visibility, and CPU work still blocks the same event loop. Use it only for truly trivial, loss-tolerant tasks. For anything that matters, use a real queue or `worker_threads` for CPU work.

---

## 8. Job Queues with BullMQ + Redis

A **job queue** is durable storage of "work to be done" that decouples the **producer** (who adds jobs) from the **consumer/worker** (who processes them). [**BullMQ**](https://docs.bullmq.io) is the modern standard for Node, built on **Redis** — an in-memory data store that provides the fast, atomic operations a queue needs (and persists to disk so jobs survive restarts).

```bash
npm install bullmq ioredis
# and run Redis, e.g.:  docker run -p 6379:6379 redis:7
```

### The mental model

- **Queue** — a named list of jobs (e.g. `"emails"`). Producers add to it.
- **Job** — one unit of work: a name + a JSON `data` payload + options (delay, attempts...).
- **Worker** — a process that pulls jobs and runs your handler. You can run many workers across many machines, all draining the same queue.
- **Connection** — both Queue and Worker connect to the same Redis.

> WARNING — Pitfall: BullMQ requires `maxRetriesPerRequest: null` on its ioredis connection (it manages retries itself). Forgetting this causes confusing connection errors under load.

### The producer (add jobs)

The producer lives inside your web app. It just adds jobs — it does no processing.

```js
// queue.js — define and export the queue (the PRODUCER side)
const { Queue } = require('bullmq');

const connection = { host: '127.0.0.1', port: 6379 }; // ioredis options

const emailQueue = new Queue('emails', { connection });

module.exports = { emailQueue, connection };
```

```js
// In a route: enqueue a job and return immediately.
const { emailQueue } = require('./queue');

app.post('/signup', async (req, res) => {
  const user = await createUser(req.body);
  // name = 'welcome', data = payload the worker will receive.
  await emailQueue.add('welcome', { to: user.email, name: user.name });
  res.status(202).json({ id: user.id });
});
```

### The worker (process jobs)

The worker is a **separate process** (`node worker.js`), so it can crash, restart, and scale independently of your web server. Its processor function receives the job and does the work; **returning** resolves the job, **throwing** fails it (and triggers retries if configured).

```js
// worker.js — the CONSUMER side, run as its own process
const { Worker } = require('bullmq');
const { connection } = require('./queue');
const { sendWelcomeEmail } = require('./mailer');

const worker = new Worker(
  'emails',
  async (job) => {
    // job.name tells you which kind of job; job.data is your payload.
    if (job.name === 'welcome') {
      await sendWelcomeEmail(job.data.to, job.data.name);
      return { delivered: true }; // becomes job.returnvalue
    }
  },
  {
    connection,
    concurrency: 5, // process up to 5 jobs at once in THIS worker
  }
);

// Observability: react to lifecycle events.
worker.on('completed', (job) => console.log(`✓ ${job.id} done`));
worker.on('failed', (job, err) => console.error(`✗ ${job?.id} failed: ${err.message}`));

// Graceful shutdown so in-flight jobs finish before exit.
process.on('SIGTERM', async () => { await worker.close(); process.exit(0); });
```

### Retries and backoff

Transient failures (a flaky SMTP server, a momentary network blip) should be retried automatically. Configure `attempts` and a `backoff` strategy when adding the job (or as queue defaults). BullMQ re-enqueues a failed job after the backoff delay, up to `attempts` times.

```js
await emailQueue.add(
  'welcome',
  { to, name },
  {
    attempts: 5,                              // try up to 5 times total
    backoff: { type: 'exponential', delay: 1000 }, // 1s, 2s, 4s, 8s between tries
    removeOnComplete: 1000,  // keep last 1000 completed (else Redis grows forever)
    removeOnFail: 5000,      // keep last 5000 failed for inspection
  }
);
```

| Backoff `type` | Delay between attempt n | Use when |
| --- | --- | --- |
| `fixed` | constant `delay` | Predictable, light retries |
| `exponential` | `delay * 2^(n-1)` | Network/3rd-party calls (avoids hammering) |

> BEST PRACTICE: Make job processors **idempotent** — safe to run twice. A job can be retried after a partial success (e.g., email sent but the worker crashed before marking it done), so "send email" jobs should be safe to re-run, or guarded by an idempotency key. Always set `removeOnComplete`/`removeOnFail`; otherwise completed jobs accumulate in Redis until you run out of memory.

### Delayed and repeatable (scheduled) jobs

A **delayed** job runs once after a wait — perfect for "send a reminder 24 hours after signup." A **repeatable** job runs on a recurring schedule (a cron expression) — BullMQ's queue-native alternative to node-cron, with the advantage that the schedule lives in Redis and survives restarts, and the work runs through the same worker/retry machinery.

```js
// Delayed: run once, 24 hours from now.
await emailQueue.add('reminder', { userId }, { delay: 24 * 60 * 60 * 1000 });

// Repeatable: every day at 09:00 (cron). BullMQ enqueues a job each tick.
await emailQueue.add(
  'daily-digest',
  {}, // payload is generated/looked up inside the worker
  { repeat: { pattern: '0 9 * * *' } } // standard cron: min hour dom mon dow
);
```

> WARNING — Pitfall: Adding a repeatable job is **idempotent by its repeat key**, but if you change the pattern you can end up with *two* schedules. On deploy, list and clean stale repeatable jobs (`queue.getRepeatableJobs()` / `removeRepeatableByKey`) rather than blindly re-adding.

### Concurrency and rate limiting

`concurrency` controls how many jobs *one worker* runs at once. To respect a third-party API's rate limit across all jobs, use the worker's `limiter`.

```js
const worker = new Worker('crm-sync', processor, {
  connection,
  concurrency: 10,
  limiter: { max: 100, duration: 60_000 }, // at most 100 jobs per 60s, queue-wide
});
```

> TIP: To monitor everything visually, add **Bull Board** (`@bull-board/express`) — a dashboard showing waiting/active/completed/failed jobs, retry buttons, and payloads. Invaluable in production.

---

## 9. Scheduled Tasks with node-cron

Some work isn't triggered by a user at all — it's triggered by *time*. Clean up expired sessions every hour. Email a daily report at 8 AM. Recompute leaderboards every 15 minutes. For in-process scheduling, [**node-cron**](https://github.com/node-cron/node-cron) runs a function on a cron schedule.

```bash
npm install node-cron
```

```js
const cron = require('node-cron');

// Cron fields:  ┌ second (optional)  ┌ minute  ┌ hour  ┌ day-of-month  ┌ month  ┌ day-of-week
//               *                    *         *       *               *        *
cron.schedule('0 8 * * *', async () => {
  // Runs every day at 08:00 server time.
  try {
    await sendDailyReport();
  } catch (err) {
    console.error('Daily report failed:', err); // a throw here is otherwise swallowed
  }
}, {
  timezone: 'Europe/London', // ALWAYS pin a timezone; servers default to UTC
});

cron.schedule('*/15 * * * *', () => recomputeLeaderboards()); // every 15 minutes
```

> WARNING — Pitfall: node-cron runs **inside your app process**, in memory. That means (1) if you run **multiple instances** of your app (load balancing, PM2 cluster), the job fires **once per instance** — your "daily email" goes out N times. (2) If the process is asleep/restarting at the trigger time, the run is simply **missed** (no catch-up). (3) An unhandled rejection in the callback is silently dropped.

### node-cron vs. BullMQ repeatable jobs

| | node-cron | BullMQ repeatable |
| --- | --- | --- |
| Where it runs | Inside the app process, in memory | Enqueued in Redis, run by workers |
| Survives restart | No (re-registered on boot) | Yes (lives in Redis) |
| Multi-instance safe | No (fires per instance) | Yes (one job per tick, any worker takes it) |
| Retries/backoff | No (DIY) | Yes (built in) |
| Missed runs | Silently skipped | Picked up when a worker is available |
| Best for | Tiny single-instance apps; trivial cleanup | Anything important, multi-instance, retryable |

> BEST PRACTICE: For a single-process app, node-cron is fine for trivial maintenance. The moment you scale to multiple instances or the job *matters* (sending money, emails, reports), either (a) use BullMQ repeatable jobs, or (b) keep node-cron but make the callback just **enqueue one BullMQ job** and add a distributed lock (e.g. a Redis `SET key NX EX`) so only one instance enqueues per tick.

---

## 10. Patterns for Long-Running Tasks

When work genuinely takes seconds to minutes (video transcode, big export, bulk import), you can't keep the HTTP connection open — it'll time out at the proxy/load balancer (often 30–60s), and the client can't show progress. The pattern is **accept-then-process**, paired with a way for the client to learn the result.

### Accept-then-process (HTTP 202)

The endpoint validates the request, **creates a job record**, enqueues the work, and returns **`202 Accepted`** with a way to check status — a resource URL for the job. The status code `202` literally means "I've accepted this for processing; it isn't done yet."

```js
app.post('/exports', authRequired, async (req, res) => {
  // 1) Create a tracking record so we can report status.
  const job = await db.exports.create({ userId: req.user.id, status: 'queued' });
  // 2) Enqueue the heavy work, passing the record id.
  await exportQueue.add('build-export', { exportId: job.id });
  // 3) Tell the client where to check and respond immediately.
  res.status(202)
     .location(`/exports/${job.id}`)
     .json({ id: job.id, status: 'queued', statusUrl: `/exports/${job.id}` });
});
```

The worker updates the record as it progresses:

```js
new Worker('exports', async (job) => {
  const { exportId } = job.data;
  await db.exports.update(exportId, { status: 'processing' });
  const fileKey = await buildExport(exportId);                 // the slow part
  await db.exports.update(exportId, { status: 'done', fileKey });
}, { connection });
```

### Option A: Status polling

The client periodically `GET`s the status URL until it's done, then downloads the result. Simple, works everywhere, no infrastructure beyond the status endpoint.

```js
app.get('/exports/:id', authRequired, async (req, res) => {
  const job = await db.exports.findById(req.params.id);
  if (!job || job.userId !== req.user.id) return res.sendStatus(404);
  res.json({
    id: job.id,
    status: job.status,                                      // queued | processing | done | failed
    downloadUrl: job.status === 'done' ? `/exports/${job.id}/file` : null,
  });
});
```

```js
// Client: poll with a sensible interval (and ideally exponential backoff).
async function waitForExport(id) {
  while (true) {
    const { status, downloadUrl } = await fetch(`/exports/${id}`).then((r) => r.json());
    if (status === 'done') return downloadUrl;
    if (status === 'failed') throw new Error('Export failed');
    await new Promise((r) => setTimeout(r, 2000)); // poll every 2s
  }
}
```

### Option B: Webhook / push callback

When the *consumer* is another server (not a browser), polling is wasteful. Instead, the client registers a **callback URL**, and when the job finishes your worker `POST`s the result to it. This inverts control: the server notifies the client. (For browsers, the push equivalents are **WebSockets** or **Server-Sent Events**.)

```js
// Worker notifies the caller's webhook when done.
new Worker('exports', async (job) => {
  const result = await buildExport(job.data.exportId);
  if (job.data.callbackUrl) {
    await fetch(job.data.callbackUrl, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        // Sign the payload so the receiver can verify it's really from you.
        'X-Signature': sign(result, process.env.WEBHOOK_SECRET),
      },
      body: JSON.stringify({ exportId: job.data.exportId, status: 'done', ...result }),
    });
  }
}, { connection });
```

| Pattern | Client type | Pros | Cons |
| --- | --- | --- | --- |
| **Polling** | Browser or server | Dead simple, no inbound endpoint needed | Wasteful requests, latency = poll interval |
| **Webhook** | Server with a public URL | Instant, efficient | Client needs a reachable signed endpoint |
| **WebSocket / SSE** | Browser | Real-time progress UI | More infra; stateful connections |

> BEST PRACTICE: Whatever the delivery mechanism, persist a **job status record** in your database (`queued → processing → done/failed`, plus progress %, error message, result location). It's the single source of truth: polling reads it, webhooks are derived from it, and a user reloading the page mid-job still sees correct state. BullMQ's `job.updateProgress(n)` can feed a progress bar.

---

## Common Pitfalls & Gotchas

- **Missing `enctype="multipart/form-data"`.** The form silently sends no file. Always set it on file forms.
- **Trying to parse uploads with `express.json()`.** It can't read multipart. Use Multer on the specific route.
- **Trusting `file.mimetype`/`originalname`.** Both are client-controlled. Validate with magic bytes (`file-type`) and never use `originalname` as a disk path.
- **Path traversal.** `path.join(dir, userInput)` with `../../` escapes your folder. Use `path.basename`, opaque ids, and verify the resolved path stays inside the base dir.
- **No `limits` on Multer.** A single client can exhaust your memory/disk. Always set `fileSize` and `files`.
- **memoryStorage for large files.** Each concurrent upload holds its full size in RAM. Use diskStorage or pre-signed direct uploads for big files.
- **Reading whole files into RAM to serve them.** Use `fs.createReadStream().pipe(res)` / `res.sendFile`; implement Range for media.
- **Storing uploads on ephemeral disk.** Containers/serverless wipe local disk and don't share it across instances. Use object storage.
- **`secure: true` on SMTP port 587.** Port 465 → `secure: true`; 587 → `secure: false` (STARTTLS).
- **HTML-only emails / unescaped interpolation.** Add a `text` fallback and escape user data into HTML.
- **Sending email inline in the request.** SMTP is slow and flaky; enqueue it as a job.
- **Doing CPU work in a request handler.** It blocks the single event loop for *all* requests. Offload to a worker (BullMQ) or `worker_threads`.
- **Treating `setTimeout`/fire-and-forget as a job queue.** No persistence, retries, or visibility. Use a real queue for anything that matters.
- **Forgetting `maxRetriesPerRequest: null` for BullMQ's Redis connection.** Causes connection errors under load.
- **Never setting `removeOnComplete`/`removeOnFail`.** Redis fills with finished jobs until it OOMs.
- **Non-idempotent job processors.** Retries can run a job twice; make them safe to re-run.
- **node-cron in a multi-instance deployment.** The job fires once per instance — duplicate emails/reports. Use BullMQ repeatable or a distributed lock.
- **No timezone on cron.** Servers run UTC; "8 AM" becomes the wrong hour. Always pass `timezone`.
- **Holding the HTTP connection open for long work.** Proxies time out at 30–60s. Use 202 + polling/webhooks.

## Best Practices

- **Validate uploads in layers:** extension allowlist → declared-mime allowlist → magic-byte check → size limit → random stored name → safe serving headers. Re-encode user images through `sharp` to strip payloads.
- **Generate filenames yourself** with `crypto.randomUUID()`/`randomBytes`; store the original name in the DB for display/download only.
- **Keep buckets private**; serve via signed URLs or a CDN. Prefer **pre-signed direct uploads** for large files so bytes bypass your server.
- **Stream, don't buffer.** Use read streams and Range support to serve files with bounded memory.
- **Send all email via background jobs**, with a `text` fallback, escaped HTML, templated bodies, and env-based transport (Ethereal/Mailtrap in dev, real provider in prod). Verify the transport at startup.
- **Get slow/CPU/flaky work out of the request cycle.** Respond fast (`202`), process in a worker.
- **Run workers as separate processes** that scale independently; shut down gracefully (`worker.close()` on `SIGTERM`).
- **Make jobs idempotent**, configure `attempts` + `exponential` backoff, set `removeOnComplete/removeOnFail`, and rate-limit calls to third parties.
- **Persist a job status record** as the source of truth; expose status via polling, webhooks, or SSE/WebSockets.
- **Use BullMQ repeatable jobs over node-cron** for anything important or multi-instance; if you keep node-cron, guard it with a distributed lock and pin a timezone.
- **Monitor queues** (Bull Board) and **load all secrets from environment variables**, never hardcoded.

## Key Interview Points

- **Why can't `express.json()` handle file uploads?** Uploads use `multipart/form-data` (parts separated by a boundary, with binary bodies); `express.json()`/`urlencoded()` only parse JSON/urlencoded text. You need a multipart parser — Multer (built on busboy).
- **Multer disk vs. memory storage — when each?** Disk for files kept locally or large files; memory when you immediately forward bytes (to S3, `sharp`, a parser) and discard them. Memory storage must have strict size limits since each upload sits in RAM.
- **How do you securely validate an uploaded file's type?** Don't trust the client-supplied `mimetype` or extension — both are spoofable. Read the **magic bytes** (e.g. `file-type`) to confirm the real format, and combine with extension/mime allowlists and size limits.
- **How do you prevent path traversal / overwrites in uploads?** Generate a random stored filename, never use `originalname` as a path, sanitize with `path.basename`, and verify the resolved path stays inside the upload directory.
- **How do you serve a large file efficiently?** Stream it (`fs.createReadStream().pipe(res)` or `res.sendFile`) instead of loading into memory, and honor the HTTP `Range` header (respond `206 Partial Content`) so media can seek.
- **Why upload to S3/Cloudinary instead of local disk?** Local disk is ephemeral and not shared across instances; object storage is durable, scalable, CDN-served, and (with pre-signed URLs) keeps large uploads off your app server.
- **What's a pre-signed URL?** A short-lived signed URL that authorizes one specific upload/download directly against the storage service, so the file bypasses your server entirely — your app only signs.
- **Why send email from a background job?** SMTP is slow and unreliable; doing it inline adds latency and couples request success to a flaky external service with no retries.
- **SMTP `secure` flag — 465 vs 587?** 465 = implicit TLS → `secure: true`; 587 = STARTTLS → `secure: false`.
- **Why must slow work leave the request cycle?** Node is single-threaded; CPU-bound work blocks the event loop for all requests, and slow/flaky I/O harms latency, couples failures, and offers no retries. Enqueue and return fast.
- **What is a job queue and what does BullMQ/Redis give you?** Durable decoupling of producers and workers, with persistence, retries + backoff, delayed/repeatable jobs, concurrency, rate limiting, and observability — none of which `setTimeout` provides.
- **Why must jobs be idempotent?** They can be retried after a partial success or crash; running twice must not cause double charges/emails.
- **node-cron vs. BullMQ repeatable jobs?** node-cron runs in-process (lost on restart, fires once per instance, no retries); BullMQ repeatable lives in Redis (survives restarts, multi-instance safe, retryable). Cron is fine for trivial single-instance tasks.
- **How do you design an API for a 2-minute task?** Accept-then-process: validate, create a status record, enqueue, return `202` with a status URL; client polls (or you webhook/SSE) and downloads when `done`.

## Practice Exercises

1. **Basic avatar upload.** Build a `POST /avatar` endpoint that accepts a single image (field `avatar`), limited to 2 MB, JPEG/PNG/WebP only, stored on disk with a random filename. Return the public URL. Serve uploads via `express.static`.

2. **Secure, validated upload.** Extend Exercise 1: switch to memory storage, validate the real type with `file-type` magic bytes (reject spoofed files), and only then persist. Add a proper error handler returning `413` for size and `415` for bad type.

3. **Private download with auth.** Add a `GET /files/:id` route that looks up a file record by id, checks the requesting user owns it, and streams the file with `res.download` using the original filename. Block path traversal.

4. **Email via a job queue.** Set up BullMQ + Redis. On `POST /signup`, create a user and enqueue a `welcome` email job (with `attempts: 3`, exponential backoff). Write a separate `worker.js` that sends the email through Nodemailer + Ethereal and logs the preview URL.

5. **Long-running export.** Implement accept-then-process: `POST /reports` returns `202` with a status URL and enqueues a job that "builds" a report (simulate with a 5s delay), updating a status record `queued → processing → done`. Add `GET /reports/:id` for polling.

6. **Scheduled cleanup, multi-instance safe.** Add a repeatable BullMQ job that runs every hour to delete export records older than 24h. Explain (in comments) why you chose BullMQ repeatable over node-cron and how it behaves with three app instances.

### Solutions

**Solution 1 — Basic avatar upload**

```js
const express = require('express');
const path = require('node:path');
const crypto = require('node:crypto');
const fs = require('node:fs');
const multer = require('multer');

const app = express();
const UPLOAD_DIR = path.join(__dirname, 'uploads');
fs.mkdirSync(UPLOAD_DIR, { recursive: true });

const storage = multer.diskStorage({
  destination: (req, file, cb) => cb(null, UPLOAD_DIR),
  filename: (req, file, cb) =>
    cb(null, crypto.randomBytes(16).toString('hex') + path.extname(file.originalname).toLowerCase()),
});

const ALLOWED = new Set(['image/jpeg', 'image/png', 'image/webp']);
const upload = multer({
  storage,
  limits: { fileSize: 2 * 1024 * 1024 }, // 2 MB
  fileFilter: (req, file, cb) =>
    ALLOWED.has(file.mimetype)
      ? cb(null, true)
      : cb(Object.assign(new Error('Unsupported type'), { code: 'INVALID_FILE_TYPE' })),
});

app.use('/uploads', express.static(UPLOAD_DIR, { index: false, dotfiles: 'ignore' }));

app.post('/avatar', upload.single('avatar'), (req, res) => {
  if (!req.file) return res.status(400).json({ error: 'No file uploaded' });
  res.status(201).json({ url: `/uploads/${req.file.filename}`, size: req.file.size });
});

app.use((err, req, res, next) => {
  if (err instanceof multer.MulterError) return res.status(413).json({ error: err.code });
  if (err?.code === 'INVALID_FILE_TYPE') return res.status(415).json({ error: err.message });
  next(err);
});

app.listen(3000);
```

**Solution 2 — Secure, validated upload**

```js
// Memory storage + magic-byte validation. file-type is ESM-only → dynamic import.
const upload = multer({
  storage: multer.memoryStorage(),
  limits: { fileSize: 2 * 1024 * 1024 },
});

app.post('/avatar', upload.single('avatar'), async (req, res, next) => {
  try {
    if (!req.file) return res.status(400).json({ error: 'No file uploaded' });

    const { fileTypeFromBuffer } = await import('file-type'); // ESM interop
    const type = await fileTypeFromBuffer(req.file.buffer);
    const allowed = new Set(['image/jpeg', 'image/png', 'image/webp']);

    if (!type || !allowed.has(type.mime)) {
      // Real bytes don't match an allowed image → reject regardless of claimed mimetype.
      return res.status(415).json({ error: 'File content is not a supported image' });
    }

    const name = `${crypto.randomUUID()}.${type.ext}`;
    await fs.promises.writeFile(path.join(UPLOAD_DIR, name), req.file.buffer);
    res.status(201).json({ url: `/uploads/${name}` });
  } catch (err) {
    next(err);
  }
});

app.use((err, req, res, next) => {
  if (err instanceof multer.MulterError && err.code === 'LIMIT_FILE_SIZE') {
    return res.status(413).json({ error: 'File too large (max 2 MB)' });
  }
  res.status(500).json({ error: 'Upload failed' });
});
```

**Solution 3 — Private download with auth**

```js
// Assume db.files.findById(id) -> { id, ownerId, storedName, originalName }
function resolveInsideDir(baseDir, name) {
  const safe = path.basename(name);                 // strip directory components
  const resolved = path.resolve(baseDir, safe);
  if (!resolved.startsWith(path.resolve(baseDir) + path.sep)) {
    throw new Error('Path traversal blocked');
  }
  return resolved;
}

app.get('/files/:id', authRequired, async (req, res, next) => {
  try {
    const record = await db.files.findById(req.params.id);
    if (!record) return res.status(404).json({ error: 'Not found' });
    if (record.ownerId !== req.user.id) return res.status(403).json({ error: 'Forbidden' });

    const abs = resolveInsideDir(UPLOAD_DIR, record.storedName);
    // res.download streams the file and sets Content-Disposition: attachment
    res.download(abs, record.originalName, (err) => {
      if (err && !res.headersSent) next(err);
    });
  } catch (err) {
    next(err);
  }
});
```

**Solution 4 — Email via a job queue**

```js
// queue.js
const { Queue } = require('bullmq');
const connection = { host: '127.0.0.1', port: 6379 };
const emailQueue = new Queue('emails', { connection });
module.exports = { emailQueue, connection };
```

```js
// routes (producer)
const { emailQueue } = require('./queue');

app.post('/signup', express.json(), async (req, res) => {
  const user = await createUser(req.body); // your logic
  await emailQueue.add(
    'welcome',
    { to: user.email, name: user.name },
    { attempts: 3, backoff: { type: 'exponential', delay: 1000 }, removeOnComplete: 100 }
  );
  res.status(202).json({ id: user.id });
});
```

```js
// worker.js (consumer) — run with: node worker.js
const { Worker } = require('bullmq');
const nodemailer = require('nodemailer');
const { connection } = require('./queue');

async function buildTransport() {
  const acc = await nodemailer.createTestAccount(); // Ethereal dev account
  return nodemailer.createTransport({
    host: 'smtp.ethereal.email', port: 587, secure: false,
    auth: { user: acc.user, pass: acc.pass },
  });
}

(async () => {
  const transporter = await buildTransport();
  const worker = new Worker('emails', async (job) => {
    if (job.name === 'welcome') {
      const info = await transporter.sendMail({
        from: 'no-reply@acme.com',
        to: job.data.to,
        subject: 'Welcome!',
        text: `Hi ${job.data.name}, welcome aboard!`,
        html: `<h1>Welcome, ${job.data.name}!</h1>`,
      });
      console.log('Preview:', nodemailer.getTestMessageUrl(info));
    }
  }, { connection, concurrency: 5 });

  worker.on('failed', (job, err) => console.error(`Job ${job?.id} failed:`, err.message));
  process.on('SIGTERM', async () => { await worker.close(); process.exit(0); });
})();
```

**Solution 5 — Long-running export**

```js
// In-memory status store for the example; use a DB in production.
const reports = new Map();

const { Queue, Worker } = require('bullmq');
const connection = { host: '127.0.0.1', port: 6379 };
const reportQueue = new Queue('reports', { connection });

app.post('/reports', async (req, res) => {
  const id = crypto.randomUUID();
  reports.set(id, { id, status: 'queued', fileUrl: null });
  await reportQueue.add('build', { id });
  res.status(202).location(`/reports/${id}`).json({ id, status: 'queued', statusUrl: `/reports/${id}` });
});

app.get('/reports/:id', (req, res) => {
  const r = reports.get(req.params.id);
  if (!r) return res.sendStatus(404);
  res.json(r);
});

new Worker('reports', async (job) => {
  const r = reports.get(job.data.id);
  r.status = 'processing';
  await new Promise((resolve) => setTimeout(resolve, 5000)); // simulate slow build
  r.status = 'done';
  r.fileUrl = `/downloads/${job.data.id}.csv`;
}, { connection });
```

**Solution 6 — Scheduled cleanup, multi-instance safe**

```js
const { Queue, Worker } = require('bullmq');
const connection = { host: '127.0.0.1', port: 6379 };
const cleanupQueue = new Queue('cleanup', { connection });

// Register the repeatable job ONCE (safe to call repeatedly — keyed by repeat opts).
// Why BullMQ repeatable over node-cron:
//  - With three app instances, node-cron would fire the cleanup THREE times each hour
//    (one per process), causing redundant/possibly conflicting deletes.
//  - BullMQ enqueues exactly ONE job per tick into Redis; whichever worker is free
//    picks it up, so the cleanup runs once regardless of instance count.
//  - It also survives restarts (schedule lives in Redis) and gets retries/backoff.
async function registerCleanup() {
  await cleanupQueue.add('purge-old-exports', {}, { repeat: { pattern: '0 * * * *' } }); // hourly
}
registerCleanup();

// Worker (own process). Run as many as you like; each tick is processed once.
new Worker('cleanup', async () => {
  const cutoff = Date.now() - 24 * 60 * 60 * 1000;
  await db.exports.deleteWhere({ createdAt: { $lt: cutoff } }); // your delete logic
}, { connection });
```

## Summary / Cheat Sheet

| Topic | Key tool / API | Remember |
| --- | --- | --- |
| Upload encoding | `multipart/form-data` | Set form `enctype`; `express.json()` can't parse it |
| Parse uploads | Multer (`single`/`array`/`fields`) | Attach per-route, not globally |
| Storage engine | `diskStorage` / `memoryStorage` | Memory for forward-then-discard; always set limits |
| Limits | `limits.fileSize`, `files` | Mandatory DoS shield |
| Filter | `fileFilter` | Cheap pre-filter; mimetype is spoofable |
| Real type check | `file-type` (magic bytes) | Ignore client-claimed mime/extension |
| Safe filename | `crypto.randomUUID()`, `path.basename` | Never use `originalname` as a path |
| Serve files | `express.static`, `res.sendFile`, streams | Stream + Range for media; auth-gate private files |
| Cloud storage | S3 / Cloudinary, pre-signed URLs | Durable, CDN, bytes bypass your server |
| Email | Nodemailer + provider SMTP | `text` + `html`; 465→secure:true, 587→false |
| Email in dev | Ethereal / Mailtrap | Never send to real inboxes |
| Why offload work | Single-threaded event loop | Respond fast (`202`), process elsewhere |
| Job queue | BullMQ + Redis | Producer adds, worker processes; idempotent jobs |
| Reliability | `attempts` + exponential `backoff` | Set `removeOnComplete/Fail`; idempotency |
| Scheduling | `delay`, `repeat`, node-cron | BullMQ repeatable for multi-instance; pin timezone |
| Long tasks | 202 + polling / webhook / SSE | Persist a status record as source of truth |

You now have the full toolkit for the "heavy" side of backend work: accepting and validating binary uploads safely, delivering them efficiently, sending email reliably through queued jobs, and architecting systems where slow work happens off the request path with retries, schedules, and clear status to the client. These patterns — accept fast, process durably, report status — recur in every serious production system you'll build.
