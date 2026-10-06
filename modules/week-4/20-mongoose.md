# Mongoose ODM (Complete Guide)

MongoDB stores **schemaless** documents: a collection can hold a record with twelve fields next to one with three, and the database will not complain. That flexibility is liberating during prototyping and dangerous in production. Without rules, your `users` collection slowly accumulates documents where `email` is sometimes a string, sometimes missing, sometimes spelled `e_mail`, and where `age` is occasionally the string `"forty"`. Six months later nobody can write a reliable query.

**Mongoose** is the layer that brings discipline back. It is an **Object Document Mapper (ODM)** — the document-database cousin of an ORM. You declare a **schema** describing the shape, types, defaults, and validation rules of your documents; Mongoose compiles that schema into a **model**; and from then on every document you create, read, update, or delete passes through that model, which casts types, enforces validation, runs your business logic in hooks, and gives you a rich, chainable query API on top of the raw MongoDB driver.

This module is a deep, complete tour of Mongoose. We start from *why* an ODM exists, then build up connections, schemas, validation, models and documents, the full CRUD surface, the query builder, middleware, virtuals and methods, indexes, population (Mongoose's answer to joins), subdocuments and relationships, and finish with the production concerns: timestamps, `toJSON` transforms, transactions, and error handling. Everything here targets **Node.js current LTS**, **Mongoose 8.x**, and **MongoDB 6/7**, using plain JavaScript with CommonJS.

> This module assumes you have MongoDB available — either a local `mongod`, a Docker container (`docker run -d -p 27017:27017 mongo`), or a free MongoDB Atlas cluster. Install the dependency with `npm install mongoose`.

## What You'll Learn

By the end of this module you will be able to:

- Explain what an **ODM** is and articulate the concrete trade-offs of **Mongoose vs the native MongoDB driver**.
- **Connect** to MongoDB with `mongoose.connect`, configure connection options, listen to **connection events**, and handle connection failures gracefully.
- Design **schemas** with the full range of SchemaTypes and field options (`required`, `default`, `unique`, `enum`, `min`/`max`, `match`, `trim`, `lowercase`), including **nested schemas**.
- Write **built-in, custom, and async validators**, and parse the resulting **`ValidationError`** objects.
- Compile **models**, create documents two ways (`create` vs `new` + `save`), and understand which one runs what.
- **Read** data with `find`, `findById`, `findOne`, query conditions, and Mongoose's **comparison and logical operators**.
- Drive the **query builder** — `where`, `sort`, `select`, `limit`, `skip`, `.lean()`, and `.exec()` — and know when each matters for performance.
- **Update** safely with `updateOne` and `findByIdAndUpdate` (`new: true`, `runValidators: true`), and explain why `save()` and update queries behave differently.
- **Delete** with `deleteOne` and `findByIdAndDelete`.
- Wire up **middleware/hooks** (`pre`/`post` for `save`, `validate`, and query operations) and reason correctly about `this` binding in each.
- Add **virtuals**, **instance methods**, and **static methods**.
- Declare **indexes** in schemas and understand their cost.
- Model relationships with **refs** and **`.populate()`**, including field selection and **multi-level population**.
- Embed **subdocuments** and **arrays of subdocuments**, and model **one-to-many** and **many-to-many** relationships.
- Apply **timestamps**, shape API output with **`toJSON` transforms**, run multi-document **transactions**, and build a clean **error-handling** pattern.

---

## 1. What Is an ODM, and Why Mongoose?

### The raw driver vs an ODM

You can talk to MongoDB directly with the official `mongodb` Node.js driver. It is excellent, fast, and gives you the raw collection API:

```js
// Native driver — no schema, no validation, no casting.
const { MongoClient } = require('mongodb');

async function run() {
  const client = new MongoClient('mongodb://127.0.0.1:27017');
  await client.connect();
  const db = client.db('shop');

  // Nothing stops this: age is a string, the field name is misspelled,
  // and there is no "name" field at all. Mongo accepts it happily.
  await db.collection('users').insertOne({ emial: 'a@b.com', age: 'forty' });

  await client.close();
}
run();
```

The driver does exactly what you tell it — including storing garbage. There is no concept of *what a user looks like*. An **ODM** sits on top of that driver and adds a modeling layer:

| Concern | Native `mongodb` driver | Mongoose ODM |
| --- | --- | --- |
| Schema / structure | None — totally freeform | Declared per model; enforced |
| Type casting | None — `"42"` stays a string | Auto-casts to declared types |
| Validation | You write it manually | Declarative + custom validators |
| Defaults | Set them yourself every time | `default` in schema |
| Business logic on writes | Scatter it in your code | Centralized in middleware/hooks |
| Relationships ("joins") | `$lookup` aggregation by hand | `ref` + `.populate()` |
| Query API | Good, lower-level | Chainable query builder + helpers |
| Performance ceiling | Highest (no overhead) | Slightly lower (hydration cost) |
| Boilerplate | More | Much less |

### Why choose Mongoose

- **A single source of truth for your data shape.** The schema is documentation that the runtime enforces.
- **Type safety at the boundary.** A form posts `age` as the string `"30"`; Mongoose casts it to the number `30` because the schema says so.
- **Validation where it belongs** — at the model, not duplicated across every route.
- **Lifecycle hooks** let you hash passwords, set slugs, or audit changes in one place.
- **Population** makes referenced documents feel like joins without writing aggregation pipelines.

### Why you might *not* use it (be honest in interviews)

- **Overhead.** Mongoose hydrates every result into a full document with getters, setters, virtuals, and change tracking. For read-heavy paths this matters — which is exactly why `.lean()` exists (Section 8).
- **Leaky abstraction.** Complex aggregation pipelines, geospatial work, and bulk operations sometimes drop you back to driver-level thinking.
- **A learning curve** around middleware `this` binding and the `save()`-vs-update distinction (both covered here).

> BEST PRACTICE: Use Mongoose for the structured, business-logic-heavy parts of your app, and reach for the raw driver or `.lean()` for high-throughput read paths and big aggregations. They coexist — Mongoose exposes the underlying driver collection via `Model.collection`.

---

## 2. Connecting to MongoDB

### The basic connection

`mongoose.connect()` opens a connection pool and returns a promise. You typically call it **once**, at application startup, before you start your HTTP server.

```js
// db.js — connect once, export a helper.
const mongoose = require('mongoose');

async function connectDB() {
  // The connection string (URI) tells Mongoose where the DB lives.
  // Format: mongodb://[user:pass@]host:port/dbName?options
  const uri = process.env.MONGO_URI || 'mongodb://127.0.0.1:27017/shop';

  // mongoose.connect resolves once the initial connection succeeds.
  await mongoose.connect(uri, {
    // These two are sensible explicit defaults; Mongoose 8 sets good
    // defaults already, but being explicit documents intent.
    serverSelectionTimeoutMS: 5000, // fail fast if no server in 5s
    maxPoolSize: 10,                // up to 10 concurrent sockets
  });

  console.log('MongoDB connected');
}

module.exports = connectDB;
```

> TIP: Use `127.0.0.1` instead of `localhost` in local URIs. On some systems `localhost` resolves to IPv6 `::1` first, and if `mongod` is only listening on IPv4 you get a confusing connection timeout.

### Anatomy of a connection string

```
mongodb+srv://alice:secret@cluster0.abcd.mongodb.net/shop?retryWrites=true&w=majority
└──┬───┘  └─┬─┘ └─┬──┘ └───────────┬───────────┘ └─┬─┘ └──────────┬──────────┘
 scheme    user   pass            host             db          options
```

- **`mongodb://`** — standard scheme; you list hosts explicitly.
- **`mongodb+srv://`** — DNS-seedlist scheme used by Atlas; the host resolves to a replica set automatically. Note `+srv` URIs must **not** include a port.
- **`/shop`** — the default database. If omitted, Mongoose uses `test`. Always name it.
- **Query options** — `retryWrites=true` (retry transient write failures once), `w=majority` (write acknowledged by a majority of replica set members).

### Wiring it into app startup

```js
// server.js — connect first, then listen. Never accept traffic before the DB is up.
const connectDB = require('./db');
const app = require('./app'); // your Express app

(async () => {
  try {
    await connectDB();              // wait for DB
    app.listen(3000, () => console.log('API on http://localhost:3000'));
  } catch (err) {
    console.error('Startup failed:', err.message);
    process.exit(1);                // exit so a supervisor/orchestrator restarts us
  }
})();
```

### Connection events

The connection is an `EventEmitter`. Listening to its events is how you observe and react to its lifecycle — initial connect, drops, and reconnects (Mongoose buffers operations and reconnects automatically by default).

```js
const mongoose = require('mongoose');
const conn = mongoose.connection; // the default connection

conn.on('connected',    () => console.log('Mongoose: connected'));
conn.on('open',         () => console.log('Mongoose: connection open'));
conn.on('disconnected', () => console.warn('Mongoose: disconnected'));
conn.on('reconnected',  () => console.log('Mongoose: reconnected'));
conn.on('error',        (err) => console.error('Mongoose error:', err.message));

// Gracefully close the pool on shutdown so in-flight ops finish.
process.on('SIGINT', async () => {
  await mongoose.connection.close();
  console.log('Mongoose connection closed on app termination');
  process.exit(0);
});
```

### Handling connection errors

There are **two distinct failure moments** and you must handle both:

1. **Initial connection failure** — the promise from `connect()` rejects. Handle it with `try/catch` (as in `server.js` above) and usually exit the process.
2. **Runtime errors after connecting** — a network blip, a primary stepping down. These surface on the `'error'` and `'disconnected'` events, *not* by rejecting the original promise. Log and monitor these; Mongoose retries by default.

```js
try {
  await mongoose.connect(uri, { serverSelectionTimeoutMS: 3000 });
} catch (err) {
  // e.g. wrong URI, auth failure, server unreachable within the timeout.
  console.error('Initial Mongo connection failed:', err.message);
  process.exit(1);
}
```

> WARNING — Pitfall: A common mistake is to assume `await mongoose.connect()` resolving means the connection will stay up forever. It only confirms the *initial* connect. Always attach an `'error'` and `'disconnected'` listener too, or silent outages will look like mysterious hung requests.

> BEST PRACTICE: Keep credentials out of source control. Read the URI from `process.env.MONGO_URI` (load it with `dotenv` in dev). Never hardcode an Atlas password.

---

## 3. Schemas: The Blueprint of a Document

A **schema** defines the fields a document can have, their types, and their rules. It does *not* talk to the database — it is pure configuration. You compile it into a model (Section 5), and the model talks to the DB.

```js
const mongoose = require('mongoose');
const { Schema } = mongoose; // destructure for convenience

const userSchema = new Schema({
  name:  { type: String, required: true, trim: true },
  email: { type: String, required: true, unique: true, lowercase: true },
  age:   { type: Number, min: 0, max: 120 },
  role:  { type: String, enum: ['user', 'admin'], default: 'user' },
  bio:   { type: String, maxlength: 500 },
  active:{ type: Boolean, default: true },
  tags:  [String],                 // an array of strings
  createdAt: { type: Date, default: Date.now }, // note: Date.now (no parens)
});
```

### SchemaTypes (the built-in field types)

| SchemaType | JS type stored | Notes |
| --- | --- | --- |
| `String` | string | Supports `trim`, `lowercase`, `uppercase`, `match`, `enum`, `minlength`, `maxlength` |
| `Number` | number | Supports `min`, `max` |
| `Date` | Date | Casts ISO strings and timestamps to `Date` |
| `Boolean` | boolean | Casts `'true'`/`1`/etc. with care |
| `Buffer` | Buffer | Binary data |
| `ObjectId` | `Schema.Types.ObjectId` | Document IDs and **references** to other docs |
| `Mixed` | anything | `Schema.Types.Mixed` — no casting, no change tracking (avoid) |
| `Array` | array | `[Type]` for typed arrays |
| `Map` | Map | Keys are strings; values are a declared type |
| `Decimal128` | high-precision decimal | Money — avoid float rounding |

### Field options reference

| Option | Applies to | Meaning |
| --- | --- | --- |
| `required` | all | Field must be present. Can be `true` or `[true, 'msg']` or a function |
| `default` | all | Value (or function returning a value) used when none supplied |
| `unique` | all | Builds a **unique index** (NOT a validator — see warning) |
| `enum` | String, Number | Value must be one of a fixed set |
| `min` / `max` | Number, Date | Inclusive bounds |
| `minlength` / `maxlength` | String | Length bounds |
| `match` | String | Must match a RegExp |
| `trim` | String | Strips leading/trailing whitespace (a setter) |
| `lowercase` / `uppercase` | String | Normalizes case (a setter) |
| `immutable` | all | Value cannot change after first set |
| `select` | all | `false` excludes the field from query results by default |
| `index` | all | Builds a (non-unique) index |

> WARNING — Pitfall: **`unique` is not a validator.** It is shorthand for "build a unique index in MongoDB." If two requests race to insert the same email, you do **not** get a clean `ValidationError` — you get a MongoDB **duplicate key error (code 11000)** at write time. You must catch that error code specifically (Section 16). Also, the index only enforces uniqueness *after it is built*; on an existing collection with duplicates the build fails silently unless you watch for it.

### Nested schemas (sub-objects)

Fields can be objects. There are two flavors, and the difference matters:

```js
// (A) Plain nested path — just an object literal. No separate _id, no its own schema.
const userSchema = new Schema({
  name: String,
  address: {                       // nested PATH, not a subdocument
    street: String,
    city:   { type: String, required: true },
    zip:    String,
  },
});

// (B) A nested SCHEMA (subdocument) — its own schema object, gets its own _id by default.
const addressSchema = new Schema({
  street: String,
  city:   { type: String, required: true },
  zip:    String,
});

const userSchema2 = new Schema({
  name: String,
  address: addressSchema,          // SUBDOCUMENT: has hooks, validation, _id
});
```

Use a **nested path** (A) for simple grouping. Use a **subdocument schema** (B) when the nested thing deserves its own validation, defaults, or hooks. We dig deeper into subdocuments in Section 14.

> TIP: A subtle gotcha with nested paths (A): `required` on a nested field only triggers if the *parent object* exists. If `address` is entirely absent, the nested `city.required` does not fire. Subdocument schemas behave more predictably here.

---

## 4. SchemaTypes & Validation

Validation runs automatically before a document is saved (`save()` and `create()`), and optionally on update queries (Section 9). If any rule fails, the operation rejects with a `ValidationError`.

### Built-in validators

```js
const productSchema = new Schema({
  sku: {
    type: String,
    required: [true, 'SKU is required'],     // custom message form
    match: [/^[A-Z]{3}-\d{4}$/, 'SKU must look like ABC-1234'],
    uppercase: true,
  },
  price: {
    type: Number,
    required: true,
    min: [0, 'Price cannot be negative'],    // [value, message]
  },
  category: {
    type: String,
    enum: {
      values: ['books', 'games', 'toys'],
      message: '{VALUE} is not a supported category', // {VALUE} interpolated
    },
  },
  releaseDate: { type: Date, max: ['2100-01-01', 'Date too far in future'] },
});
```

Every built-in validator accepts the `[value, message]` array form to customize the error message. `{VALUE}` and `{PATH}` are interpolated into messages.

### Custom validators (`validate`)

When the built-ins are not enough, supply a `validate` function. It returns `true` if valid, `false` if not.

```js
const userSchema = new Schema({
  username: {
    type: String,
    validate: {
      // `this` is the document being validated (when not an update).
      validator: function (value) {
        return /^[a-z0-9_]+$/.test(value); // only lowercase, digits, underscore
      },
      message: (props) => `${props.value} is not a valid username`,
    },
  },
  // You can attach multiple validators as an array.
  password: {
    type: String,
    validate: [
      { validator: (v) => v.length >= 8, message: 'Min 8 characters' },
      { validator: (v) => /\d/.test(v),  message: 'Need at least one digit' },
    ],
  },
});
```

### Async validators

If the validator returns a **Promise**, Mongoose awaits it. This is how you validate against the database or an external service — e.g. confirm an email is not already taken (with a friendlier error than the raw duplicate-key error).

```js
const userSchema = new Schema({
  email: {
    type: String,
    required: true,
    validate: {
      // Return a Promise<boolean>. Resolving false (or throwing) = invalid.
      validator: async function (value) {
        // `this.constructor` is the Model inside a document validator.
        const existing = await this.constructor.findOne({ email: value });
        // Allow if no other doc has it (or it's this same doc on update).
        return !existing || existing._id.equals(this._id);
      },
      message: (props) => `Email ${props.value} is already registered`,
    },
  },
});
```

> WARNING — Pitfall: Async validators of this kind have a **race condition** — two simultaneous signups can both pass the `findOne` check before either saves. They improve UX but are **not** a substitute for the unique index. Keep both: the async validator for a friendly message in the common case, the unique index for correctness under concurrency.

### Validation errors

A failed validation rejects with a `ValidationError` whose `.errors` is keyed by path. Inspect it to build clean API responses.

```js
try {
  await User.create({ name: '', email: 'BAD', age: 999 });
} catch (err) {
  if (err.name === 'ValidationError') {
    // err.errors is an object: { path: ValidatorError, ... }
    for (const [path, e] of Object.entries(err.errors)) {
      console.log(`${path}: ${e.message} (kind: ${e.kind})`);
    }
    // e.g. -> "age: Path `age` (999) is more than maximum allowed value (120). (kind: max)"
  }
}
```

> TIP: Manually trigger validation without saving via `await doc.validate()`, or synchronously via `doc.validateSync()`. Useful in tests and multi-step flows.

---

## 5. Models & Documents

A **model** is a schema compiled into a constructor that maps to a MongoDB **collection**. A **document** is an instance of a model — one record.

```js
// Model name (singular, PascalCase) -> Mongoose pluralizes/lowercases it
// into the collection name. "User" -> collection "users".
const User = mongoose.model('User', userSchema);
```

> TIP: To control the collection name explicitly, pass it as a third argument: `mongoose.model('User', userSchema, 'app_users')`, or set `{ collection: 'app_users' }` in the schema options. Otherwise Mongoose lowercases and pluralizes the model name.

### Creating documents: two routes

**Route 1 — `new` + `save()`.** Build the instance in memory, optionally mutate it, then persist.

```js
const user = new User({ name: 'Ada', email: 'ada@calc.dev' });
user.role = 'admin';      // mutate before saving
await user.save();        // runs validation + save hooks, writes to DB
console.log(user._id);    // the generated ObjectId is now populated
```

**Route 2 — `Model.create()`.** A shortcut that does `new` + `save()` in one call. Accepts one object or an array.

```js
const user = await User.create({ name: 'Grace', email: 'grace@navy.mil' });

// Bulk create (each is validated and saved; runs save hooks per doc).
const users = await User.create([
  { name: 'Linus', email: 'linus@kernel.org' },
  { name: 'Guido', email: 'guido@python.org' },
]);
```

| | `new` + `save()` | `Model.create()` |
| --- | --- | --- |
| Steps | Two (construct, then save) | One |
| Mutate before persist | Yes, between the two calls | No |
| Runs `validate` + `save` hooks | Yes | Yes |
| Returns | The same instance you built | The created document(s) |
| Best for | When you need to set fields conditionally | Straight inserts |

> BEST PRACTICE: For inserting **many** documents where you do not need per-document hooks, prefer `Model.insertMany(docs)`. It is far faster (one bulk write) but by default skips `save` middleware. Pass `{ runValidators: true }`... actually `insertMany` validates by default; it just does not run `save` hooks. Know that distinction.

---

## 6. Reading: `find`, `findById`, `findOne`

These are the core read operations. Each returns a **Query** object (not a Promise) until you `await` it or call `.exec()`.

```js
// find -> ARRAY of all matching docs (empty array if none).
const admins = await User.find({ role: 'admin' });

// findOne -> the FIRST matching doc, or null.
const ada = await User.findOne({ email: 'ada@calc.dev' });

// findById -> shorthand for findOne({ _id: id }); null if not found.
const byId = await User.findById('507f1f77bcf86cd799439011');

// No filter = everything (careful on big collections!).
const everyone = await User.find();
```

> WARNING — Pitfall: `findById(id)` returns `null` when nothing matches — it does **not** throw. Always handle the `null` case (e.g. respond 404) instead of blindly accessing `.name` on it.

### Query conditions & operators

Mongoose passes MongoDB query operators straight through. The filter object is the same syntax you'd use with the raw driver.

```js
// Comparison operators
await User.find({ age: { $gte: 18, $lt: 65 } });   // 18 <= age < 65
await User.find({ role: { $in: ['admin', 'mod'] } }); // role is one of
await User.find({ age: { $ne: 30 } });              // not equal
await User.find({ deletedAt: { $exists: false } }); // field absent

// Logical operators
await User.find({ $or: [{ role: 'admin' }, { age: { $gte: 60 } }] });
await User.find({ $and: [{ active: true }, { age: { $lt: 30 } }] });
await User.find({ name: { $not: /^test/i } });      // does NOT start with "test"

// Regex / text search on a string field
await User.find({ name: /ada/i });                  // case-insensitive contains

// Match inside arrays
await User.find({ tags: 'node' });                  // array contains 'node'
await User.find({ tags: { $all: ['node', 'js'] } });// contains BOTH
```

| Operator | Meaning |
| --- | --- |
| `$eq` / `$ne` | Equal / not equal |
| `$gt` / `$gte` / `$lt` / `$lte` | Greater/less than (or equal) |
| `$in` / `$nin` | In / not in a list |
| `$exists` | Field present or not |
| `$regex` (or a `/regex/`) | Pattern match |
| `$or` / `$and` / `$nor` | Logical combination |
| `$not` | Negate a condition |
| `$all` / `$elemMatch` / `$size` | Array conditions |

---

## 7. The Query Builder & Chaining

Because read methods return a **Query**, you can refine it fluently before executing. This is one of Mongoose's most ergonomic features.

```js
const results = await User.find()           // start the query
  .where('age').gte(18).lte(65)             // age between 18 and 65
  .where('role').equals('user')             // role == 'user'
  .select('name email -_id')                // include name+email, exclude _id
  .sort({ name: 1 })                        // ascending by name (1 = asc, -1 = desc)
  .skip(20)                                 // for pagination: skip first 20
  .limit(10);                               // return at most 10
// awaiting the chain executes it.
```

### The chaining methods

| Method | Purpose |
| --- | --- |
| `.where(path)` | Begin a condition on `path` (pairs with `.equals/.gt/.in/...`) |
| `.select(fields)` | Field projection — `'a b'` to include, `'-a'` to exclude |
| `.sort(spec)` | Order results: `{ field: 1 \| -1 }` or `'field -other'` |
| `.limit(n)` | Cap the number of returned documents |
| `.skip(n)` | Skip the first `n` (use with `.limit` for paging) |
| `.populate(path)` | Replace refs with the referenced documents (Section 13) |
| `.lean()` | Return plain objects instead of Mongoose documents |
| `.exec()` | Execute the query, returning a real Promise |

### `.select()` projection in depth

```js
await User.find().select('name email');   // ONLY name, email (and _id by default)
await User.find().select('-password -__v'); // everything EXCEPT these
// You cannot mix inclusion and exclusion (except for _id).
```

If a field has `select: false` in the schema (e.g. a password hash), include it explicitly with a `+`:

```js
const u = await User.findOne({ email }).select('+password'); // override schema default
```

### `.lean()` — the performance lever

By default every result is **hydrated** into a full Mongoose document: change tracking, getters/setters, virtuals, instance methods. That costs CPU and memory. When you only need to **read and serialize** data (the typical GET endpoint), `.lean()` returns plain JavaScript objects and skips all hydration — often **5–10x faster** and far lighter.

```js
// Hydrated documents (can call .save(), have virtuals, etc.)
const docs = await User.find({ active: true });

// Plain objects — NO .save(), NO virtuals, NO instance methods. Read-only, fast.
const plain = await User.find({ active: true }).lean();
```

> BEST PRACTICE: Default to `.lean()` on read-only endpoints that just return JSON. Drop it only when you need to mutate-and-save the result, call instance methods, or rely on virtuals/`toJSON` transforms.

### `.exec()` — Promise vs thenable

A Query is *thenable* (it has a `.then`), so `await User.find()` works. `.exec()` returns a genuine `Promise`. Two practical reasons to use it:

1. **Better stack traces** on errors — the rejection originates closer to your call site.
2. **Explicitness** in code review — it signals "this query runs here."

```js
const user = await User.findById(id).exec(); // returns a true Promise<Document|null>
```

---

## 8. Updating Documents

There are two philosophies for updating, and the difference is one of the most-tested Mongoose topics.

### Philosophy A — load, mutate, `save()`

Fetch the document, change it in memory, save it. This runs **full validation** and **`save` middleware**, and only writes the fields that actually changed.

```js
const user = await User.findById(id);
if (!user) throw new Error('Not found');
user.age = 31;                 // mutate
user.role = 'admin';
await user.save();             // validates everything, runs pre/post('save') hooks
```

### Philosophy B — update queries

Tell MongoDB to update directly, without loading the document into Node. Faster (one round trip, no hydration), but it **bypasses `save` hooks** and, by default, **does not run validators**.

```js
// updateOne -> returns a result summary, NOT the document.
const res = await User.updateOne(
  { _id: id },
  { $set: { age: 31 }, $inc: { loginCount: 1 } } // atomic update operators
);
console.log(res.matchedCount, res.modifiedCount);

// updateMany -> same, across all matches.
await User.updateMany({ role: 'user' }, { $set: { active: true } });
```

### `findByIdAndUpdate` / `findOneAndUpdate`

These update **and return** a document in one atomic operation. Two options you almost always want:

- **`new: true`** — return the *updated* document. **Without it, you get the document as it was BEFORE the update** (the default, which surprises everyone once).
- **`runValidators: true`** — run schema validators against the update. Off by default for update queries.

```js
const updated = await User.findByIdAndUpdate(
  id,
  { $set: { email: 'new@addr.dev' } },
  {
    new: true,            // return the document AFTER the update
    runValidators: true,  // validate the update against the schema
    context: 'query',     // required for some validators that use `this`
  }
);
```

### Atomic update operators

Inside an update you use MongoDB operators, applied atomically server-side:

| Operator | Effect |
| --- | --- |
| `$set` | Set field value(s) |
| `$unset` | Remove a field |
| `$inc` | Increment a number (negative to decrement) |
| `$mul` | Multiply a number |
| `$push` / `$addToSet` | Append to an array (`$addToSet` only if absent) |
| `$pull` | Remove matching array elements |
| `$rename` | Rename a field |

> WARNING — Pitfall: `$inc: { loginCount: 1 }` is **race-safe** because the server does the arithmetic atomically. Doing `user.loginCount += 1; await user.save()` is **not** — two concurrent requests can both read the old value and lose an increment. Use atomic operators for counters.

### `save()` vs update queries — the full comparison

| Aspect | `save()` (load-mutate-save) | `updateOne` / `findByIdAndUpdate` |
| --- | --- | --- |
| DB round trips | Two (read, then write) | One |
| Runs `validate` | Always | Only with `runValidators: true` |
| Runs `pre/post('save')` hooks | Yes | No (runs `pre/post('updateOne'/'findOneAndUpdate')` instead) |
| Returns | The document | A result summary (`update*`) or a document (`findOneAnd*`) |
| Atomic counters | Risky (read-modify-write) | Safe with `$inc` |
| Best for | Rich domain logic, validations, hooks | Targeted, high-throughput field updates |

> WARNING — Pitfall: Update-query validators only validate the **fields in the update**, not the whole document — and some custom validators that read `this` need `{ context: 'query' }` to work. If your validators depend on other fields, prefer the load-mutate-`save()` path.

---

## 9. Deleting Documents

```js
// deleteOne -> removes the first match; returns { deletedCount }.
const res = await User.deleteOne({ email: 'ada@calc.dev' });
console.log(res.deletedCount); // 1 or 0

// deleteMany -> removes all matches.
await User.deleteMany({ active: false });

// findByIdAndDelete -> removes AND returns the deleted document (or null).
const removed = await User.findByIdAndDelete(id);
if (!removed) throw new Error('Nothing to delete');

// findOneAndDelete -> same, by filter.
const removed2 = await User.findOneAndDelete({ email: 'old@x.dev' });
```

| Method | Returns | Runs delete hooks? |
| --- | --- | --- |
| `deleteOne` (model) | `{ deletedCount }` | `pre/post('deleteOne')` query hook |
| `deleteMany` | `{ deletedCount }` | `pre/post('deleteMany')` |
| `findByIdAndDelete` | deleted doc or `null` | `pre/post('findOneAndDelete')` |
| `doc.deleteOne()` (on an instance) | the doc | document `deleteOne` hook |

> BEST PRACTICE: In many apps you want **soft deletes** — set `deletedAt: new Date()` instead of physically removing data — so you can audit and recover. Combine a `deletedAt` field with a query helper or a default scope. Use hard deletes only when you truly mean it.

---

## 10. Middleware (Hooks)

**Middleware** (also called hooks) are functions Mongoose runs automatically around certain operations. They are the heart of Mongoose's "centralize your logic" promise: hash a password before every save, generate a slug, cascade deletes, log audits.

There are four hook *types*, and the most important practical split is **document hooks vs query hooks** — because `this` means something different in each.

### Document middleware: `save`, `validate`

For `save` and `validate`, the hooks run on a **document**, so `this` is the document.

```js
const bcrypt = require('bcrypt');

userSchema.pre('save', async function (next) {
  // `this` is the document about to be saved.
  if (!this.isModified('password')) return next(); // skip if unchanged
  this.password = await bcrypt.hash(this.password, 12);
  next(); // or just return (async functions can omit next)
});

userSchema.post('save', function (doc, next) {
  // Runs AFTER a successful save. `doc` is the saved document.
  console.log(`User ${doc._id} saved`);
  next();
});

// validate hooks run BEFORE save's validation step.
userSchema.pre('validate', function () {
  if (this.name) this.slug = this.name.toLowerCase().replace(/\s+/g, '-');
});
```

The order for a `save()` is: **`pre('validate')` → validation → `post('validate')` → `pre('save')` → write → `post('save')`**.

### Query middleware: `find`, `findOne`, `updateOne`, `deleteOne`, ...

For query operations, the hook runs on the **Query**, so `this` is the query, **not** a document. This is the #1 confusion in Mongoose.

```js
// Soft-delete scope: automatically hide deleted docs from all find queries.
userSchema.pre(/^find/, function () {
  // `this` is the QUERY. Add a condition to it.
  this.where({ deletedAt: { $exists: false } });
});

// Use a regex (/^find/) to cover find, findOne, findById, findOneAndUpdate, etc.
```

> WARNING — Pitfall: In a **query** hook, `this` is the Query — you cannot read document field values like `this.password`. In a **document** hook (`save`/`validate`), `this` is the document. Mixing these up is the most common Mongoose bug. Also note: `findByIdAndUpdate` triggers the **`findOneAndUpdate`** query hook, *not* the `save` hook — so password-hashing logic in a `save` hook will silently NOT run on `findByIdAndUpdate`.

### Hook registration must come before model compilation

```js
// CORRECT: register all hooks, THEN compile the model.
userSchema.pre('save', fn);
const User = mongoose.model('User', userSchema); // compile last

// WRONG: hooks added after model() is called are ignored.
```

| Hook spec | `this` is | Common use |
| --- | --- | --- |
| `pre('validate')` | document | Derive fields (slugs) before validation |
| `pre('save')` | document | Hash passwords, set computed fields |
| `post('save')` | document (as arg) | Side effects, logging, emit events |
| `pre(/^find/)` | query | Default filters, soft-delete scoping |
| `pre('findOneAndUpdate')` | query | Normalize update payloads |
| `pre('deleteOne')` | document or query | Cascade deletes |
| `pre('aggregate')` | aggregate | Inject pipeline stages |

---

## 11. Virtuals, Instance Methods & Static Methods

These three let you attach behavior and derived data to your models.

### Virtuals — computed properties that aren't stored

A **virtual** is a field that exists on the document but is **not persisted** to MongoDB. It is computed on access. Classic example: `fullName` from `firstName` + `lastName`.

```js
userSchema.virtual('fullName').get(function () {
  return `${this.firstName} ${this.lastName}`;
});

// Virtuals can have setters too.
userSchema.virtual('fullName').set(function (value) {
  const [first, last] = value.split(' ');
  this.firstName = first;
  this.lastName = last;
});

const u = await User.findById(id);
console.log(u.fullName); // "Ada Lovelace" — computed, not in the DB
```

> WARNING — Pitfall: Virtuals do **not** appear in `JSON.stringify(doc)` or API responses by default, and they are gone entirely with `.lean()`. To include them in output, enable them in schema options: `{ toJSON: { virtuals: true }, toObject: { virtuals: true } }` (Section 15).

### Instance methods — behavior on a single document

Add methods to `schema.methods`. Inside, `this` is the document.

```js
userSchema.methods.comparePassword = async function (candidate) {
  return bcrypt.compare(candidate, this.password); // `this` = the document
};

const user = await User.findOne({ email }).select('+password');
const ok = await user.comparePassword('hunter2');
```

### Static methods — behavior on the model

Add methods to `schema.statics`. Inside, `this` is the **model**, so you can run queries.

```js
userSchema.statics.findByEmail = function (email) {
  return this.findOne({ email: email.toLowerCase() }); // `this` = the Model
};

const user = await User.findByEmail('ADA@calc.dev');
```

| Feature | Defined on | `this` is | Use for |
| --- | --- | --- | --- |
| Virtual | `schema.virtual()` | document | Derived, non-stored fields |
| Instance method | `schema.methods` | document | Operations on one record |
| Static method | `schema.statics` | model | Custom queries / factory helpers |
| Query helper | `schema.query` | query | Reusable chainable conditions |

> TIP: Use **arrow functions** anywhere you do **not** need `this` (e.g. plain utilities), but use **regular `function`** for virtuals, methods, statics, and hooks — arrows capture the surrounding `this` and break Mongoose's binding.

---

## 12. Indexes

An **index** is a sorted data structure MongoDB maintains so it can find documents without scanning the whole collection. Without an index, `User.findOne({ email })` performs a **collection scan** — O(n) — which is fine for 100 docs and catastrophic for 10 million.

### Declaring indexes

```js
const userSchema = new Schema({
  email:    { type: String, unique: true },   // shorthand: unique index
  username: { type: String, index: true },     // shorthand: plain index
  city:     String,
  age:      Number,
});

// Compound index — supports queries that filter/sort on city then age.
userSchema.index({ city: 1, age: -1 });

// Partial / unique / TTL indexes via the second options arg:
userSchema.index({ email: 1 }, { unique: true });
userSchema.index(
  { createdAt: 1 },
  { expireAfterSeconds: 3600 } // TTL: auto-delete docs 1h after createdAt
);
```

### Index build behavior

By default Mongoose calls `createIndexes` on startup (controlled by the `autoIndex` option). This is convenient in dev but a problem in production: building indexes on a large live collection is expensive.

```js
// In production, turn off autoIndex and build indexes deliberately/offline.
mongoose.connect(uri, { autoIndex: false });
// or per-schema: new Schema({...}, { autoIndex: false })
```

> WARNING — Pitfall: Indexes are not free. Each one consumes RAM and slows down writes (every insert/update must maintain every index). Index the fields you actually query/sort on — not everything. The `unique` shorthand also builds an index, so it carries this cost.

> TIP: Verify your queries actually use indexes with `.explain()`: `await User.find({ email }).explain('executionStats')` and look for `IXSCAN` (good) vs `COLLSCAN` (a full scan — missing index).

---

## 13. Population (Refs & Joins)

MongoDB is not relational, but real data has relationships. Mongoose models them with **references**: store another document's `ObjectId`, then use **`.populate()`** to swap that id for the actual document at query time (Mongoose issues a second query under the hood).

### Defining a ref

```js
const postSchema = new Schema({
  title: String,
  body:  String,
  // Store the author's id, and tell Mongoose which model it refers to.
  author: { type: Schema.Types.ObjectId, ref: 'User' },
  // An array of refs — a post's tags pointing at Tag documents.
  tags: [{ type: Schema.Types.ObjectId, ref: 'Tag' }],
});
const Post = mongoose.model('Post', postSchema);
```

### Populating

```js
// Without populate: author is just an ObjectId.
const raw = await Post.findById(id);
console.log(raw.author); // ObjectId("...")

// With populate: author becomes the full User document.
const post = await Post.findById(id).populate('author');
console.log(post.author.name); // "Ada Lovelace"

// Populate multiple paths.
const post2 = await Post.findById(id).populate('author').populate('tags');
```

### Selecting fields while populating

Pull only what you need from the referenced document:

```js
const post = await Post.findById(id).populate({
  path: 'author',
  select: 'name email -_id', // only name + email from the User
});
```

### Nested / multi-level populate

You can populate a path *inside* a populated document.

```js
// A comment refers to a user; a post refers to comments.
// Populate comments, and within each comment populate its user.
const post = await Post.findById(id).populate({
  path: 'comments',
  populate: { path: 'user', select: 'name' },
});
```

### Filtering populated results with `match`

```js
const post = await Post.findById(id).populate({
  path: 'comments',
  match: { approved: true },  // only approved comments
  options: { sort: { createdAt: -1 }, limit: 5 },
});
```

> WARNING — Pitfall: `populate` is **not a database join** — Mongoose runs a separate query per populated path and stitches the results in Node. Populating a list of 1,000 posts each with an author is one query for posts + one query for the batch of distinct authors (Mongoose deduplicates). Watch for **over-population** that fetches huge documents you do not need — always `select` the minimal fields.

> TIP: If a referenced document was deleted, the populated field becomes `null`. Always guard against `post.author?.name`. Also, `.lean()` works with `populate` and is the fast path for read-only populated responses.

---

## 14. Subdocuments, Arrays & Relationships

You can either **embed** related data inside a document (subdocuments) or **reference** it (refs + populate). Choosing between them is a core data-modeling decision.

### Single subdocuments

```js
const addressSchema = new Schema({
  street: String,
  city:   { type: String, required: true },
});

const userSchema = new Schema({
  name: String,
  address: addressSchema, // embedded subdocument — lives inside the user doc
});

const u = await User.create({ name: 'Ada', address: { city: 'London' } });
u.address.city = 'Bath';  // mutate the subdoc
await u.save();           // saved as part of the parent document
```

### Arrays of subdocuments

This is how you model a parent with many embedded children (e.g. an order with line items).

```js
const itemSchema = new Schema({
  product: String,
  qty:     { type: Number, default: 1 },
  price:   Number,
});

const orderSchema = new Schema({
  customer: String,
  items: [itemSchema], // array of subdocuments
});
const Order = mongoose.model('Order', orderSchema);

const order = await Order.create({
  customer: 'Grace',
  items: [{ product: 'Book', qty: 2, price: 20 }],
});

// Subdocument arrays have helpers:
order.items.push({ product: 'Pen', price: 3 }); // add
const sub = order.items.id(someSubdocId);        // find by its _id
sub.deleteOne();                                  // remove it
await order.save();
```

### Embed vs reference — the modeling decision

| Use **embedding** when... | Use **referencing** when... |
| --- | --- |
| Child data is owned by, and read with, the parent | Child is shared across many parents |
| The child has bounded size (won't grow unboundedly) | The set can grow large/unbounded |
| You almost always need them together | You often need one without the other |
| Example: order line items, address | Example: a user's posts, post's author |

> WARNING — Pitfall: MongoDB documents have a **16 MB size limit**. Embedding an unbounded array (e.g. every comment ever on a viral post) will eventually blow that limit. Unbounded growth → reference, don't embed.

### Modeling relationships

**One-to-many (references).** The "many" side stores the parent's id.

```js
// A user has many posts. Each post references its author.
const postSchema = new Schema({
  title: String,
  author: { type: Schema.Types.ObjectId, ref: 'User' }, // the "one"
});
// To list a user's posts:
const posts = await Post.find({ author: userId });
```

**One-to-many (virtual populate).** Keep the reference on the child but query *from the parent* without storing an array — clean and unbounded-safe.

```js
userSchema.virtual('posts', {
  ref: 'Post',
  localField: '_id',     // User._id
  foreignField: 'author' // matches Post.author
});
// Enable virtuals in toJSON (Section 15), then:
const user = await User.findById(id).populate('posts');
```

**Many-to-many.** Each side stores an array of the other's ids (or use a join collection for relationship metadata).

```js
const studentSchema = new Schema({
  name: String,
  courses: [{ type: Schema.Types.ObjectId, ref: 'Course' }],
});
const courseSchema = new Schema({
  title: String,
  students: [{ type: Schema.Types.ObjectId, ref: 'Student' }],
});
// Populate either direction:
const student = await Student.findById(id).populate('courses');
```

---

## 15. Timestamps, `toJSON` Transforms, Transactions & Error Handling

### Timestamps

Set `{ timestamps: true }` and Mongoose maintains `createdAt` and `updatedAt` automatically on every save and update.

```js
const userSchema = new Schema(
  { name: String, email: String },
  { timestamps: true } // adds createdAt + updatedAt, kept current automatically
);
// Customize the field names if you like:
// { timestamps: { createdAt: 'created_at', updatedAt: 'updated_at' } }
```

### `toJSON` transforms — shaping API output

When you `res.json(user)`, Express calls `JSON.stringify`, which calls the document's `toJSON`. Configure it to drop sensitive/internal fields and surface virtuals — so you never leak a password hash or `__v` again.

```js
const userSchema = new Schema(
  { name: String, email: String, password: { type: String, select: false } },
  {
    timestamps: true,
    toJSON: {
      virtuals: true, // include virtuals in JSON output
      transform(doc, ret) {
        ret.id = ret._id;     // expose a clean `id`
        delete ret._id;       // hide Mongo's _id
        delete ret.__v;       // hide the version key
        delete ret.password;  // belt-and-suspenders: never serialize it
        return ret;
      },
    },
  }
);
```

> BEST PRACTICE: Configure `toJSON` once on the schema rather than hand-picking fields in every controller. It centralizes your output contract and prevents accidental leakage of internal fields.

### Transactions (intro)

A **transaction** groups multiple writes so they all succeed or all roll back — essential for operations like "debit one account, credit another." Transactions require a **replica set** (Atlas is one by default; a standalone local `mongod` is not, unless run as a single-node replica set).

```js
const session = await mongoose.startSession();
try {
  session.startTransaction();

  // Pass { session } to every operation in the transaction.
  await Account.updateOne({ _id: from }, { $inc: { balance: -100 } }, { session });
  await Account.updateOne({ _id: to },   { $inc: { balance:  100 } }, { session });

  await session.commitTransaction(); // both writes commit together
} catch (err) {
  await session.abortTransaction();  // any failure rolls BOTH back
  throw err;
} finally {
  session.endSession();              // always release the session
}
```

> TIP: Mongoose also offers `session.withTransaction(async () => { ... })`, which handles commit/abort/retry on transient errors for you. Prefer it for production code.

### Error-handling patterns

Distinguish error *kinds* so you can map them to correct HTTP responses:

```js
function toHttpError(err) {
  if (err.name === 'ValidationError') {
    // Schema validation failed -> 400 with field messages.
    const fields = Object.fromEntries(
      Object.entries(err.errors).map(([k, v]) => [k, v.message])
    );
    return { status: 400, body: { error: 'Validation failed', fields } };
  }
  if (err.name === 'CastError') {
    // e.g. a malformed ObjectId passed to findById -> 400.
    return { status: 400, body: { error: `Invalid ${err.path}: ${err.value}` } };
  }
  if (err.code === 11000) {
    // Duplicate key (unique index) -> 409 Conflict.
    const field = Object.keys(err.keyValue)[0];
    return { status: 409, body: { error: `${field} already exists` } };
  }
  return { status: 500, body: { error: 'Internal server error' } };
}
```

| Error | `name` / `code` | Cause | HTTP |
| --- | --- | --- | --- |
| Validation | `ValidationError` | Schema rule failed | 400 |
| Cast | `CastError` | Bad type (e.g. invalid ObjectId) | 400 |
| Duplicate key | code `11000` | Unique index violation | 409 |
| Not found | `null` return (not an error!) | No matching doc | 404 |

> WARNING — Pitfall: A "not found" is **not** a thrown error — `findById` returns `null`. Check for it explicitly and return 404; do not expect a `catch` block to handle it.

---

## Common Pitfalls & Gotchas

- **`unique` is an index, not a validator.** Duplicate inserts throw error code `11000`, not a `ValidationError`. Catch `err.code === 11000`.
- **`findByIdAndUpdate` returns the OLD doc by default.** Pass `{ new: true }` to get the updated one.
- **Update queries skip validation and `save` hooks.** Use `{ runValidators: true }`, and remember password-hashing `save` hooks do NOT run on `findOneAndUpdate`.
- **`this` differs in document vs query hooks.** Document hooks (`save`/`validate`): `this` is the doc. Query hooks (`find`, `findOneAndUpdate`): `this` is the query.
- **Arrow functions break `this`.** Never use them for hooks, virtuals, methods, or statics.
- **Virtuals vanish in JSON and under `.lean()`** unless you set `toJSON: { virtuals: true }`. `.lean()` strips them entirely.
- **`findById(badFormat)` throws a `CastError`**, while `findById(validButMissing)` returns `null`. Handle both.
- **`Date.now` vs `Date.now()`** as a default: pass the *function* `Date.now` (no parens) so it runs at create time, not once at schema-definition time.
- **`populate` is not a join** — it is extra queries stitched in Node. Over-populating large docs hurts performance; always `select`.
- **`autoIndex: true` in production** can stall startup building indexes on big collections. Disable it and build indexes deliberately.
- **Hooks added after `mongoose.model()`** are ignored. Register all middleware before compiling the model.
- **Read-modify-write on counters loses updates** under concurrency. Use atomic `$inc`.

---

## Best Practices

- **Connect once at startup**, await it before listening, and attach `'error'`/`'disconnected'` listeners. Exit on initial connect failure.
- **Keep the URI and credentials in environment variables**, never in source.
- **Default to `.lean()`** on read-only JSON endpoints; drop it only when you need documents.
- **Centralize output shaping** in a schema `toJSON` transform (hide `password`, `__v`, rename `_id` → `id`).
- **Use `select: false`** for sensitive fields and opt them in with `.select('+password')` only where needed.
- **Prefer atomic update operators** (`$inc`, `$push`, `$addToSet`) over read-modify-write for concurrency safety.
- **Validate on updates** with `{ runValidators: true }`, or prefer load-mutate-`save()` when validators depend on multiple fields.
- **Index what you query and sort on** — and verify with `.explain('executionStats')`. Avoid over-indexing.
- **Choose embed vs reference deliberately**: bounded + always-together → embed; shared/unbounded → reference (or virtual populate).
- **Use transactions** for multi-document invariants, ideally via `session.withTransaction`.
- **Map error kinds to HTTP codes** consistently (`ValidationError`→400, `CastError`→400, `11000`→409, `null`→404).
- **Register all hooks before compiling the model**, and use regular `function` for anything that relies on `this`.

---

## Key Interview Points

- **What is an ODM and why use Mongoose over the driver?** An ODM maps schemas to MongoDB documents, adding type casting, validation, defaults, hooks, and population on top of the raw driver — at the cost of some hydration overhead, mitigated by `.lean()`.
- **Is `unique` a validator?** No. It builds a unique index; violations surface as a duplicate-key error (code `11000`) at write time, not a `ValidationError`.
- **`save()` vs update queries?** `save()` does read-modify-write, runs full validation and `save` hooks, and returns the document. `updateOne`/`findByIdAndUpdate` are single round trips that skip `save` hooks and (by default) validators; pass `runValidators: true`.
- **Default return of `findByIdAndUpdate`?** The document *before* the update. Use `{ new: true }` for the updated version.
- **`this` in middleware?** Document for `save`/`validate` hooks; the Query for `find`/`findOneAndUpdate` query hooks. `findByIdAndUpdate` triggers the `findOneAndUpdate` query hook, not `save`.
- **What does `.lean()` do and when?** Returns plain objects, skipping document hydration (no virtuals/methods/change tracking) — much faster for read-only endpoints.
- **Is `populate` a join?** No — it runs additional queries and merges results in application code. Select minimal fields to avoid over-fetching.
- **Embed vs reference?** Embed bounded data read with its parent; reference shared or unbounded data. Mind the 16 MB document limit.
- **Virtuals vs instance vs static methods?** Virtuals are computed, non-stored fields; instance methods act on a document (`this` = doc); statics act on the model (`this` = model).
- **Built-in vs custom vs async validators?** Built-ins (`required`, `enum`, `min`, `match`); custom via a `validator` function; async by returning a Promise (good for friendly uniqueness messages, but not a substitute for the unique index).
- **How do indexes affect performance?** They speed reads/sorts but cost RAM and slow writes; disable `autoIndex` in production and verify usage with `.explain()`.
- **How do transactions work in Mongoose?** Via a session passed to each operation, committed/aborted together; require a replica set; `session.withTransaction` handles retries.

---

## Practice Exercises

Work these in order; each builds on the last. Set up a project with `npm init -y && npm install mongoose bcrypt` and a running MongoDB.

1. **Connect & model.** Write a `connectDB()` that connects, logs `'connected'` and `'error'` events, and exits on initial failure. Define a `User` schema with `name` (required, trimmed), `email` (required, unique, lowercase, matching an email regex), `age` (0–120), and `role` (enum `user`/`admin`, default `user`).
2. **Validation deep-dive.** Add a custom validator on `email` that asynchronously rejects a duplicate with a friendly message, and a `password` field that is `select: false`, min length 8, hashed via a `pre('save')` hook. Add an instance method `comparePassword`.
3. **CRUD + query builder.** Write functions to: create a user; find active adult users sorted by name, returning only `name` and `email`, paginated (`skip`/`limit`), using `.lean()`; update a user's role with `findByIdAndUpdate` returning the new doc and running validators; and delete a user by id, returning 404-style `null` handling.
4. **Relationships & population.** Add a `Post` model with `title`, `body`, and an `author` ref to `User`. Add a virtual `posts` on `User` (virtual populate). Write a query that loads a user with their posts populated, selecting only post titles.
5. **Subdocuments & atomic ops.** Add an `Order` model with `customer` and an array of `items` subdocuments (`product`, `qty` default 1, `price`). Write functions to push an item, remove an item by its subdoc `_id`, and atomically increment a top-level `version` counter with `$inc`.
6. **Transaction.** Model two `Account` documents and write a `transfer(from, to, amount)` that moves money inside a transaction using `session.withTransaction`, rolling back if either account would go negative.

### Solutions

#### Solution 1 — Connect & model

```js
// db.js
const mongoose = require('mongoose');

async function connectDB() {
  const uri = process.env.MONGO_URI || 'mongodb://127.0.0.1:27017/practice';
  mongoose.connection.on('connected', () => console.log('connected'));
  mongoose.connection.on('error', (e) => console.error('error', e.message));
  try {
    await mongoose.connect(uri, { serverSelectionTimeoutMS: 5000 });
  } catch (err) {
    console.error('Initial connection failed:', err.message);
    process.exit(1);
  }
}
module.exports = connectDB;
```

```js
// models/User.js
const { Schema, model } = require('mongoose');

const userSchema = new Schema({
  name:  { type: String, required: true, trim: true },
  email: {
    type: String, required: true, unique: true, lowercase: true,
    match: [/^[^\s@]+@[^\s@]+\.[^\s@]+$/, 'Invalid email format'],
  },
  age:   { type: Number, min: 0, max: 120 },
  role:  { type: String, enum: ['user', 'admin'], default: 'user' },
});

module.exports = model('User', userSchema);
```

#### Solution 2 — Validation, hashing, instance method

```js
// models/User.js (extended)
const { Schema, model } = require('mongoose');
const bcrypt = require('bcrypt');

const userSchema = new Schema({
  name:  { type: String, required: true, trim: true },
  email: {
    type: String, required: true, unique: true, lowercase: true,
    match: [/^[^\s@]+@[^\s@]+\.[^\s@]+$/, 'Invalid email format'],
    validate: {
      validator: async function (value) {
        const existing = await this.constructor.findOne({ email: value });
        return !existing || existing._id.equals(this._id);
      },
      message: (props) => `Email ${props.value} is already registered`,
    },
  },
  password: { type: String, required: true, minlength: 8, select: false },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
});

userSchema.pre('save', async function () {
  if (!this.isModified('password')) return;     // only hash when it changed
  this.password = await bcrypt.hash(this.password, 12);
});

userSchema.methods.comparePassword = function (candidate) {
  return bcrypt.compare(candidate, this.password);
};

module.exports = model('User', userSchema);
```

#### Solution 3 — CRUD + query builder

```js
const User = require('./models/User');

async function createUser(data) {
  return User.create(data); // runs validation + save hooks
}

async function listActiveAdults({ page = 1, perPage = 10 } = {}) {
  return User.find({ age: { $gte: 18 } })
    .select('name email -_id')
    .sort({ name: 1 })
    .skip((page - 1) * perPage)
    .limit(perPage)
    .lean(); // read-only -> fast plain objects
}

async function changeRole(id, role) {
  const updated = await User.findByIdAndUpdate(
    id,
    { $set: { role } },
    { new: true, runValidators: true }
  );
  if (!updated) return null; // caller maps null -> 404
  return updated;
}

async function removeUser(id) {
  const removed = await User.findByIdAndDelete(id);
  return removed; // null if it did not exist
}
```

#### Solution 4 — Relationships & virtual populate

```js
// models/Post.js
const { Schema, model } = require('mongoose');
const postSchema = new Schema({
  title: { type: String, required: true },
  body:  String,
  author:{ type: Schema.Types.ObjectId, ref: 'User', required: true },
});
module.exports = model('Post', postSchema);
```

```js
// Add to userSchema BEFORE compiling the model:
userSchema.set('toJSON', { virtuals: true });
userSchema.set('toObject', { virtuals: true });
userSchema.virtual('posts', {
  ref: 'Post',
  localField: '_id',
  foreignField: 'author',
});

// Query:
async function userWithPosts(id) {
  return User.findById(id).populate({ path: 'posts', select: 'title -author' });
}
```

#### Solution 5 — Subdocuments & atomic ops

```js
// models/Order.js
const { Schema, model } = require('mongoose');
const itemSchema = new Schema({
  product: { type: String, required: true },
  qty:     { type: Number, default: 1, min: 1 },
  price:   { type: Number, required: true, min: 0 },
});
const orderSchema = new Schema({
  customer: { type: String, required: true },
  items:    [itemSchema],
  version:  { type: Number, default: 0 },
});
const Order = model('Order', orderSchema);

async function addItem(orderId, item) {
  const order = await Order.findById(orderId);
  if (!order) return null;
  order.items.push(item);    // subdocument array helper
  await order.save();        // validates the new subdoc
  return order;
}

async function removeItem(orderId, itemId) {
  const order = await Order.findById(orderId);
  if (!order) return null;
  const sub = order.items.id(itemId);
  if (!sub) return order;    // nothing to remove
  sub.deleteOne();
  await order.save();
  return order;
}

async function bumpVersion(orderId) {
  // Atomic — race-safe, single round trip.
  return Order.findByIdAndUpdate(
    orderId,
    { $inc: { version: 1 } },
    { new: true }
  );
}

module.exports = { Order, addItem, removeItem, bumpVersion };
```

#### Solution 6 — Transaction

```js
const mongoose = require('mongoose');
const { Schema, model } = mongoose;

const accountSchema = new Schema({
  owner:   String,
  balance: { type: Number, default: 0, min: 0 },
});
const Account = model('Account', accountSchema);

async function transfer(fromId, toId, amount) {
  const session = await mongoose.startSession();
  try {
    await session.withTransaction(async () => {
      // Debit only if sufficient funds (guard with a filter condition).
      const debit = await Account.updateOne(
        { _id: fromId, balance: { $gte: amount } },
        { $inc: { balance: -amount } },
        { session }
      );
      if (debit.modifiedCount === 0) {
        throw new Error('Insufficient funds'); // aborts the transaction
      }
      await Account.updateOne(
        { _id: toId },
        { $inc: { balance: amount } },
        { session }
      );
    });
  } finally {
    session.endSession();
  }
}

module.exports = { Account, transfer };
```

> Note for Solution 6: transactions require a replica set. Locally, start one with `mongod --replSet rs0` and initiate it once with `rs.initiate()` in `mongosh`, or use MongoDB Atlas.

---

## Summary / Cheat Sheet

| Task | Code |
| --- | --- |
| Connect | `await mongoose.connect(uri, { serverSelectionTimeoutMS: 5000 })` |
| Connection events | `mongoose.connection.on('error' \| 'disconnected', fn)` |
| Define schema | `new Schema({ field: { type, required, default, ... } })` |
| Compile model | `mongoose.model('User', userSchema)` |
| Create (1 step) | `await User.create({ ... })` |
| Create (2 step) | `const u = new User({...}); await u.save()` |
| Read all | `await User.find(filter)` |
| Read one | `await User.findOne(filter)` / `findById(id)` (→ `null` if none) |
| Project fields | `.select('name email')` / `.select('-password')` |
| Sort/paginate | `.sort({ name: 1 }).skip(n).limit(m)` |
| Fast read-only | `.lean()` |
| Real Promise | `.exec()` |
| Update (return new) | `findByIdAndUpdate(id, upd, { new: true, runValidators: true })` |
| Update many | `updateMany(filter, { $set: {...} })` |
| Atomic counter | `{ $inc: { count: 1 } }` |
| Delete + return | `findByIdAndDelete(id)` (→ `null` if none) |
| Delete many | `deleteMany(filter)` |
| Doc hook | `schema.pre('save', function () { /* this = doc */ })` |
| Query hook | `schema.pre(/^find/, function () { /* this = query */ })` |
| Virtual | `schema.virtual('full').get(function(){...})` |
| Instance method | `schema.methods.foo = function(){ /* this = doc */ }` |
| Static method | `schema.statics.bar = function(){ /* this = model */ }` |
| Index | `schema.index({ city: 1, age: -1 })` |
| Ref field | `{ type: Schema.Types.ObjectId, ref: 'User' }` |
| Populate | `.populate({ path: 'author', select: 'name' })` |
| Subdoc array | `items: [itemSchema]`, then `doc.items.id(x)` / `.push()` |
| Timestamps | `new Schema({...}, { timestamps: true })` |
| JSON transform | `{ toJSON: { virtuals: true, transform(d, ret){...} } }` |
| Transaction | `session.withTransaction(async () => { ...{ session } })` |

**Error map:** `ValidationError`→400 · `CastError`→400 · code `11000`→409 · `null` return→404.

**The three things people get wrong most:** (1) `findByIdAndUpdate` returns the *old* doc without `new: true`; (2) `this` is the *query* in query hooks and the *document* in `save`/`validate` hooks; (3) `unique` is an index, not a validator (catch error `11000`).
