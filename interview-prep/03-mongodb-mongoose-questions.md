# MongoDB & Mongoose Interview Questions & Answers

This question bank prepares you to discuss MongoDB and Mongoose at the depth a strong backend candidate is expected to reach. It assumes you already know JavaScript but are relatively new to databases and server-side persistence, so the early questions build the mental model from first principles before the later ones push into modeling trade-offs, the aggregation pipeline, transactions, and performance.

**How to use it:**

1. Read each question, cover the answer, and try to answer out loud as if a real interviewer were across the table. Speaking is different from recognizing.
2. Then read the model answer. The goal is not to memorize it word-for-word but to absorb the *structure* of a good answer: definition, the "why", a concrete example, and the trade-off.
3. Pay attention to the `Follow-up:` lines — interviewers rarely ask one question and move on. They drill. If you can pre-empt the drill, you sound senior.
4. Run the code. Every snippet is runnable against a local MongoDB (`mongodb://127.0.0.1:27017`) or Atlas. Predicting output in your head and then verifying is the single best way to make this stick.
5. The final **Rapid-Fire Quick Answers** table is for the morning of the interview — one-line refreshers, nothing more.

Stack assumed: Node.js current LTS, MongoDB 6+/7+, Mongoose 8.x, plain JavaScript with CommonJS `require` (ES Module variants shown where relevant).

---

## Beginner

### 1. **What is the difference between SQL and NoSQL databases, and where does MongoDB fit?**

SQL (relational) databases like PostgreSQL and MySQL store data in **tables** with a fixed **schema** of typed columns, and rows are linked across tables via **foreign keys**. You query them with the declarative SQL language, and they classically guarantee **ACID** transactions across multiple rows and tables.

NoSQL is an umbrella term for databases that intentionally relax one or more of those properties to gain flexibility or horizontal scalability. There are four broad families:

| Family | Example | Stores data as |
|---|---|---|
| Document | MongoDB, CouchDB | JSON-like documents |
| Key-Value | Redis, DynamoDB | Opaque values keyed by a string |
| Column-family | Cassandra, HBase | Wide rows grouped into column families |
| Graph | Neo4j | Nodes and edges |

MongoDB is a **document database**. Instead of rows in tables, it stores **documents** (BSON, a binary superset of JSON) inside **collections**. Documents in the same collection do not have to share the same shape, so the schema is "flexible" — enforced by your application (or Mongoose), not rigidly by the database engine by default.

The key mental shift for someone coming from SQL: in MongoDB you often **store related data together** inside one document rather than splitting it across normalized tables, because there are no cheap JOINs at the storage layer. You design around how you *read* the data.

> TIP: A clean one-liner for interviews: "SQL is schema-on-write with relations and JOINs; MongoDB is schema-flexible documents where you model data the way your application reads it."

**Follow-up:** "Does MongoDB support transactions / ACID?" (Yes since 4.0 for multi-document transactions on replica sets — covered later.)

---

### 2. **When would you choose MongoDB over a relational database, and when would you avoid it?**

**Choose MongoDB when:**

- Your data is naturally hierarchical or document-shaped (a product with nested variants, a user with embedded preferences, a CMS article with blocks).
- The schema evolves rapidly and you want to add fields without migrations.
- Your read patterns are well known and you can model data so a typical query hits a single document (no JOINs needed).
- You need easy horizontal scaling via **sharding** for very high write/read throughput.
- You want to start fast — the document model maps almost 1:1 to objects in your code.

**Avoid / reconsider MongoDB when:**

- Your data is highly relational with many many-to-many relationships that must be queried from every direction (an accounting ledger, a complex ERP).
- You need strong multi-entity transactional guarantees as the *default* mode of operation (banking core ledgers).
- You rely heavily on ad-hoc reporting joins across many tables — SQL is far more ergonomic there.
- Your team and tooling are deeply invested in SQL and the problem is a comfortable relational fit.

> WARNING — Pitfall: "MongoDB is faster than SQL" is **not** a correct answer. Performance depends entirely on access patterns and indexing. The honest differentiator is the **data model** and **scaling story**, not raw speed.

**Follow-up:** "Can you give a concrete bad fit for MongoDB?" — A double-entry accounting system where every transaction touches multiple accounts and consistency is non-negotiable across them.

---

### 3. **Explain documents, collections, and databases in MongoDB. How do they map to SQL concepts?**

- A **document** is a single record: an ordered set of key/value pairs, stored as BSON. It looks like a JSON object and can contain nested objects and arrays.
- A **collection** is a group of documents — analogous to a table, but without an enforced column schema.
- A **database** holds collections.

| MongoDB | Relational SQL |
|---|---|
| Database | Database |
| Collection | Table |
| Document | Row |
| Field | Column |
| Embedded document / array | (JOINed table or JSON column) |
| `_id` | Primary key |
| Index | Index |
| `$lookup` (aggregation) | JOIN |

A sample document:

```json
{
  "_id": { "$oid": "65f1c2a9e3b1a2c4d5e6f7a8" },
  "name": "Ada Lovelace",
  "email": "ada@example.com",
  "roles": ["admin", "editor"],
  "address": { "city": "London", "zip": "EC1" },
  "createdAt": { "$date": "2024-01-10T09:00:00Z" }
}
```

Notice `roles` is an array and `address` is a nested document — in SQL those would typically be separate tables.

**Follow-up:** "Do all documents in a collection need the same fields?" — No. That is the point of a flexible schema, though in practice you usually enforce consistency with Mongoose.

---

### 4. **What is BSON and how does it differ from JSON?**

**BSON** ("Binary JSON") is the binary-encoded serialization format MongoDB uses to store documents on disk and transmit them over the wire. JSON is the human-readable text format you usually *write* in your app; BSON is what MongoDB actually stores.

Differences that matter:

| Aspect | JSON | BSON |
|---|---|---|
| Encoding | Text | Binary |
| Parse speed | Slower (text parsing) | Faster (typed, length-prefixed) |
| Types | string, number, boolean, null, object, array | All of those **plus** `ObjectId`, `Date`, `Binary`, `Decimal128`, `Int32`, `Int64`, `Double`, `Timestamp`, `RegExp` |
| Numbers | One number type | Distinguishes `int`, `long`, `double`, `decimal` |
| Field order | Not guaranteed meaningful | Preserved |

The practical takeaways:

- BSON has **richer types**, which is why a MongoDB `Date` round-trips correctly and why money should use `Decimal128` rather than a floating-point `double`.
- Because BSON is length-prefixed and typed, MongoDB can skip over fields quickly when scanning, which helps performance.

```js
// In Node, the driver/Mongoose converts JS values to BSON for you.
const { Decimal128, ObjectId } = require("mongodb");

const doc = {
  _id: new ObjectId(),          // BSON ObjectId, not a plain string
  price: Decimal128.fromString("19.99"), // exact decimal, safe for money
  createdAt: new Date(),        // stored as BSON Date
};
```

> BEST PRACTICE: Store currency as `Decimal128` (Mongoose: `type: mongoose.Schema.Types.Decimal128`), never as a JS `Number`, to avoid floating-point rounding errors.

**Follow-up:** "Why not store money as a float?" — Binary floating point cannot represent values like `0.1` exactly, so sums drift.

---

### 5. **What is the `_id` field and what is an `ObjectId`?**

Every MongoDB document has a primary key field called `_id` that is **unique within its collection** and is **automatically indexed**. If you do not supply one on insert, MongoDB generates an `ObjectId`.

An `ObjectId` is a 12-byte value composed of:

1. **4 bytes** — a Unix timestamp (seconds) of creation.
2. **5 bytes** — a random value unique to the machine/process.
3. **3 bytes** — an incrementing counter.

Consequences worth knowing:

- ObjectIds are **roughly time-ordered**, so sorting by `_id` approximately sorts by creation time, and you can extract the creation time without a separate field.
- They are generated **client-side** by the driver, so you do not need a round-trip to the database to know a new document's id.
- They are globally unlikely to collide, which suits distributed/sharded systems far better than an auto-increment integer.

```js
const { ObjectId } = require("mongodb");
const id = new ObjectId("65f1c2a9e3b1a2c4d5e6f7a8");
console.log(id.getTimestamp()); // -> Date when the id (and usually the doc) was created
```

> WARNING — Pitfall: A 24-character hex *string* is not the same as an `ObjectId`. Querying `{ _id: "65f1..." }` with a string will not match a document whose `_id` is an `ObjectId`. In Mongoose, casting usually handles this for `_id`, but for other ObjectId fields you must pass an actual `ObjectId` or a string that Mongoose can cast.

**Follow-up:** "Can `_id` be something other than an ObjectId?" — Yes, any unique value (a string, a number, even a sub-document) as long as it is unique; you just lose the automatic ordering/timestamp benefits.

---

### 6. **What are the basic CRUD operations in MongoDB and what do they return?**

CRUD = Create, Read, Update, Delete. Using the native driver / `mongosh` style:

```js
// CREATE
db.users.insertOne({ name: "Ada", age: 36 });
db.users.insertMany([{ name: "Grace" }, { name: "Linus" }]);

// READ
db.users.findOne({ name: "Ada" });           // first match or null
db.users.find({ age: { $gte: 18 } });          // a cursor over many docs

// UPDATE
db.users.updateOne({ name: "Ada" }, { $set: { age: 37 } });
db.users.updateMany({ age: null }, { $set: { age: 0 } });
db.users.replaceOne({ name: "Ada" }, { name: "Ada", age: 37 });

// DELETE
db.users.deleteOne({ name: "Ada" });
db.users.deleteMany({ active: false });
```

Two things candidates often get wrong:

1. **`find()` returns a cursor, not an array.** You iterate it (or call `.toArray()`). Mongoose's `find()` resolves to an array when awaited, which hides this.
2. **Updates need an update operator** like `$set`. Passing a raw object `{ age: 37 }` to `updateOne` in modern drivers throws, because it is ambiguous with a replacement. `replaceOne` is the explicit "swap the whole document" operation.

The Mongoose equivalents:

```js
await User.create({ name: "Ada", age: 36 });             // Create
const u = await User.findOne({ name: "Ada" });            // Read
await User.updateOne({ name: "Ada" }, { $set: { age: 37 } }); // Update
await User.deleteOne({ name: "Ada" });                    // Delete
```

**Follow-up:** "What does `updateOne` return?" — A result object with `matchedCount`, `modifiedCount`, `upsertedId`, etc., not the updated document. Use `findOneAndUpdate` to get the document back.

---

### 7. **What is Mongoose and what problem does it solve? What does "ODM" mean?**

**Mongoose** is an **ODM** — Object Data Modeling library — for MongoDB in Node.js. An ODM is the document-database equivalent of an ORM (Object-Relational Mapper): it maps your MongoDB documents to JavaScript objects and gives you a structured way to interact with them.

MongoDB itself is schema-flexible, which is powerful but means *nothing stops you* from inserting a typo'd field or the wrong type. Mongoose adds a layer on top of the native driver that provides:

- **Schemas** — you declare the expected shape and types of documents.
- **Models** — constructors built from schemas that you use to query and create documents.
- **Validation** — required fields, min/max, enums, custom validators, all enforced before save.
- **Type casting** — a string `"36"` going into a `Number` field is cast to `36`.
- **Middleware (hooks)** — run logic before/after save, validate, remove, query, etc.
- **Query helpers, virtuals, instance/static methods, population** — ergonomics around relationships and computed fields.

```js
const mongoose = require("mongoose");

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  age: { type: Number, min: 0 },
});

const User = mongoose.model("User", userSchema);
// Now User.create / find / updateOne etc. all enforce the schema.
```

> TIP: The honest trade-off to mention: Mongoose adds structure, safety, and developer ergonomics, but also a layer of abstraction and some overhead. Teams that want maximum control and minimal magic sometimes use the native driver directly.

**Follow-up:** "Could you use MongoDB without Mongoose?" — Yes, via the official `mongodb` driver. You then enforce schema yourself (or with MongoDB's native JSON Schema validation).

---

### 8. **What is the difference between a Mongoose schema and a model?**

- A **schema** is a *blueprint*. It defines fields, their types, validation rules, defaults, virtuals, methods, and middleware. A schema does **not** talk to the database by itself.
- A **model** is a *compiled constructor* created from a schema and bound to a specific collection. The model is what you actually use to create, query, update, and delete documents. Each instance of a model is a **document**.

```js
// Schema = the design
const bookSchema = new mongoose.Schema({
  title: String,
  pages: Number,
});

// Model = the usable class, bound to the "books" collection (pluralized, lowercased)
const Book = mongoose.model("Book", bookSchema);

// Document = an instance of the model
const b = new Book({ title: "SICP", pages: 657 });
await b.save();
```

The flow is: **Schema -> Model -> Document.**

Note Mongoose pluralizes and lowercases the model name to derive the collection name: `mongoose.model("Book", ...)` uses the `books` collection. You can override with a third argument: `mongoose.model("Book", bookSchema, "my_books")`.

**Follow-up:** "What collection name does `mongoose.model('Person')` use?" — `people` (Mongoose uses a pluralization library, so it handles irregular plurals).

---

### 9. **How do you connect to MongoDB with Mongoose, and what are the connection-handling best practices?**

```js
const mongoose = require("mongoose");

async function connectDB() {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      // Modern Mongoose (6+) sets sensible defaults; these are optional examples:
      serverSelectionTimeoutMS: 5000, // fail fast if no server in 5s
      maxPoolSize: 10,                // size of the connection pool
    });
    console.log("MongoDB connected");
  } catch (err) {
    console.error("MongoDB connection error:", err.message);
    process.exit(1); // app cannot run without the DB — exit so the orchestrator restarts it
  }
}

// React to connection lifecycle events
mongoose.connection.on("disconnected", () => console.warn("Mongo disconnected"));
mongoose.connection.on("error", (err) => console.error("Mongo error:", err));

connectDB();
```

Best practices to mention:

- **Connect once at startup**, not per request. Mongoose maintains a **connection pool** and reuses sockets; opening a connection per request destroys performance.
- **Store the URI in an environment variable**, never hardcoded — it contains credentials.
- **Fail fast on startup**: if the DB is unreachable at boot, exit so your process manager restarts cleanly, rather than serving broken requests.
- Mongoose **buffers** model operations issued before the connection is ready and flushes them once connected, so you do not have to manually await the connection before defining models. (You can disable buffering with `bufferCommands: false` if you prefer hard failures.)
- For serverless (Lambda, Vercel), **cache the connection** across invocations on the global object so you do not reconnect on every cold-warm cycle.

> WARNING — Pitfall: Calling `mongoose.connect()` inside a route handler is a classic beginner mistake. It opens a new pool on every request and exhausts connections.

**Follow-up:** "What is a connection pool and why does it matter?" — A set of reusable open sockets so requests do not pay TCP/auth handshake cost each time, and concurrency is bounded.

---

### 10. **What does Mongoose schema validation give you, and what are the built-in validators?**

Validation runs automatically on `save()` (and on `create()`), and can be triggered on updates via `runValidators`. It rejects documents that violate the rules with a `ValidationError`.

Built-in validators:

| Validator | Applies to | Example |
|---|---|---|
| `required` | all types | `{ type: String, required: true }` |
| `min` / `max` | Number, Date | `{ type: Number, min: 0, max: 120 }` |
| `minLength` / `maxLength` | String | `{ type: String, minLength: 8 }` |
| `enum` | String, Number | `{ type: String, enum: ["draft", "published"] }` |
| `match` | String (regex) | `{ type: String, match: /@/ }` |
| `validate` | custom | see below |

Custom and async validators:

```js
const userSchema = new mongoose.Schema({
  email: {
    type: String,
    required: [true, "Email is required"], // custom message form
    validate: {
      validator: (v) => /^\S+@\S+\.\S+$/.test(v),
      message: (props) => `${props.value} is not a valid email`,
    },
  },
  age: { type: Number, min: [18, "Must be 18+"] },
});
```

```js
try {
  await User.create({ email: "nope", age: 5 });
} catch (err) {
  // err.name === "ValidationError"
  // err.errors.email.message, err.errors.age.message available
}
```

> WARNING — Pitfall: `unique: true` is **not a validator** — it builds a unique *index* on the field. If duplicates already exist or the index has not finished building, you will not get a clean Mongoose `ValidationError`; you get a MongoDB duplicate-key error (code `11000`) at write time. Handle that error explicitly.

**Follow-up:** "Does validation run on `findOneAndUpdate`?" — Not by default; you must pass `runValidators: true`.

---

## Intermediate

### 11. **Explain the most common MongoDB query and update operators.**

**Query (comparison) operators** filter which documents match:

| Operator | Meaning |
|---|---|
| `$eq`, `$ne` | equal / not equal |
| `$gt`, `$gte`, `$lt`, `$lte` | greater/less than (or equal) |
| `$in`, `$nin` | value in / not in an array of options |
| `$exists` | field present or not |
| `$regex` | string matches pattern |
| `$and`, `$or`, `$nor`, `$not` | logical combinators |
| `$elemMatch` | array element matching multiple conditions |

```js
// Active users aged 18–30 whose role is admin OR editor
db.users.find({
  active: true,
  age: { $gte: 18, $lte: 30 },
  role: { $in: ["admin", "editor"] },
});
```

**Update operators** modify matched documents:

| Operator | Effect |
|---|---|
| `$set` | set field value |
| `$unset` | remove a field |
| `$inc` | increment a number |
| `$mul` | multiply |
| `$push` | append to an array |
| `$addToSet` | append only if not present |
| `$pull` | remove matching array elements |
| `$pop` | remove first/last array element |
| `$rename` | rename a field |
| `$min` / `$max` | set only if smaller/larger |
| `$currentDate` | set to current date |

```js
db.products.updateOne(
  { _id: id },
  {
    $set: { name: "New name" },
    $inc: { stock: -1 },           // sell one unit
    $addToSet: { tags: "sale" },   // no duplicate tags
  }
);
```

> WARNING — Pitfall: `$push` allows duplicates; `$addToSet` does not. Mixing them up is a common bug when maintaining a set of unique values like tags or member ids.

**Follow-up:** "Difference between `$elemMatch` and a plain array query?" — `{ scores: { $gt: 80, $lt: 90 } }` matches if *some* element is >80 and *some* (possibly different) element is <90; `$elemMatch` requires a *single* element to satisfy all conditions at once.

---

### 12. **What is the difference between `find`, `findOne`, `findById`, and the `findOneAnd*` family in Mongoose?**

- `find(filter)` returns **an array** of all matches (empty array if none).
- `findOne(filter)` returns **one document or `null`**.
- `findById(id)` is sugar for `findOne({ _id: id })`; it casts the id and returns one doc or `null`.
- `findOneAndUpdate` / `findByIdAndUpdate` / `findOneAndDelete` find a document **and** mutate it in a single atomic operation, returning the document.

```js
const all = await User.find({ active: true });   // []  or [docs]
const one = await User.findOne({ email });        // doc or null
const byId = await User.findById(req.params.id);  // doc or null

const updated = await User.findByIdAndUpdate(
  id,
  { $set: { name: "New" } },
  { new: true, runValidators: true } // see Q13
);
```

Key distinctions:

- The `findOneAnd*` operations are **atomic at the document level** — no other operation can interleave between the find and the modify. Doing `findOne` then editing then `save()` is **not** atomic and is vulnerable to lost updates under concurrency.
- They return the matched document; the plain `updateOne` returns only counts.

**Follow-up:** "When would you still prefer `findOne` + `save()` over `findOneAndUpdate`?" — When you need Mongoose document middleware/`save` hooks, full validation semantics, or complex multi-field logic that is awkward to express as update operators.

---

### 13. **What do the `new` and `runValidators` options do on `findByIdAndUpdate`/`findOneAndUpdate`? What are the default behaviors people get wrong?**

This is one of the most common "gotcha" interview questions for Mongoose.

By default, `findOneAndUpdate` and `findByIdAndUpdate`:

1. Return the document **as it was *before* the update** (the old version).
2. Do **not** run schema validators.
3. Apply `$set`-style updates without running `save` middleware (`pre('save')` does **not** fire; only `pre('findOneAndUpdate')` / query middleware fires).

The options fix these:

- **`new: true`** — return the **updated** document instead of the original.
- **`runValidators: true`** — run schema validators against the update operation.

```js
// WRONG — returns the OLD document and skips validators
const u1 = await User.findByIdAndUpdate(id, { age: -5 });
// u1 is the document BEFORE update; the invalid age: -5 is saved anyway

// RIGHT — returns updated doc and enforces validation
const u2 = await User.findByIdAndUpdate(
  id,
  { age: 30 },
  {
    new: true,           // return the modified document
    runValidators: true, // enforce min/max/required etc. on the update
    context: "query",    // needed for some custom validators that reference `this`
  }
);
```

Additional nuances worth mentioning:

- With `runValidators`, validators run against the **update payload**, not a full document, so a custom validator that reads *other* fields via `this` may not behave as on `save` unless you pass `context: "query"`.
- `required` validators are **not** applied to update operations by default even with `runValidators`, because the update may legitimately not touch that field. Use `runValidators` together with care, or validate on the application side.
- Use `upsert: true` to insert if no document matches; combine with `setDefaultsOnInsert: true` so schema defaults apply to the inserted doc.

> WARNING — Pitfall: Forgetting `new: true` and then wondering why the API response shows stale data is an extremely common bug. Forgetting `runValidators` is how invalid data sneaks into an otherwise validated collection.

**Follow-up:** "Does `pre('save')` run on `findByIdAndUpdate`?" — No. Only query middleware (`pre('findOneAndUpdate')`) runs.

---

### 14. **What are indexes in MongoDB, how do they work, and what are the trade-offs?**

An **index** is a separate, ordered data structure (a B-tree) that lets MongoDB find matching documents without scanning every document in the collection. Without an index, a query does a **COLLSCAN** (collection scan), reading every document — O(n). With a suitable index, it does an **IXSCAN**, navigating the tree — roughly O(log n) to locate, plus the matches.

Types of indexes:

| Type | Use |
|---|---|
| Single field | `{ email: 1 }` |
| Compound | `{ status: 1, createdAt: -1 }` — multiple fields in order |
| Multikey | automatically used when indexing an array field |
| Text | full-text search on strings |
| Geospatial (`2dsphere`) | location queries |
| Hashed | sharding by hash |
| TTL | auto-expire documents after N seconds |
| Partial / sparse | index only some documents |
| Unique | enforce uniqueness |

```js
// In Mongoose, declare on the schema
userSchema.index({ email: 1 }, { unique: true });
userSchema.index({ status: 1, createdAt: -1 }); // compound

// TTL: expire session docs 1 hour after createdAt
sessionSchema.index({ createdAt: 1 }, { expireAfterSeconds: 3600 });
```

Trade-offs:

- Indexes make reads fast but **slow down writes**, because every insert/update must also update each affected index.
- They consume **memory and disk**. The best performance comes when indexes fit in RAM (the "working set").
- More indexes is not better — only create indexes that real queries use.

The **ESR rule** for compound indexes (very interview-worthy): order fields as **E**quality, then **S**ort, then **R**ange. A compound index `{ status: 1, createdAt: -1 }` efficiently serves `find({ status: "active" }).sort({ createdAt: -1 })`.

> BEST PRACTICE: Use `db.collection.find(...).explain("executionStats")` to confirm a query uses an IXSCAN and not a COLLSCAN. If you see `COLLSCAN` on a hot query, you are missing an index.

**Follow-up:** "Why might MongoDB ignore your index?" — Low selectivity (a scan is cheaper), wrong field order vs. the query, the query not being a prefix of a compound index, or a type mismatch preventing index use.

---

### 15. **What is a covered query?**

A **covered query** is one where MongoDB can answer entirely from the index, without fetching the actual documents. It happens when:

1. All fields in the query filter are part of the index, **and**
2. All fields returned (the projection) are also in the index, **and**
3. No returned field is an array (multikey), and `_id` is explicitly excluded if it is not in the index.

```js
// Index on { email: 1, name: 1 }
db.users.createIndex({ email: 1, name: 1 });

// Covered: filter and projection both live entirely in the index
db.users.find(
  { email: "ada@example.com" },
  { _id: 0, name: 1 } // exclude _id since it's not in this index
);
```

Because the documents themselves are never read from disk, covered queries are extremely fast. `explain()` shows `totalDocsExamined: 0` for a covered query — that zero is the tell.

**Follow-up:** "Why must you exclude `_id`?" — `_id` is returned by default and is not part of your `{ email, name }` index, so including it forces a document fetch and breaks coverage.

---

### 16. **What is the aggregation pipeline? Walk through the common stages.**

The **aggregation pipeline** processes documents through a sequence of **stages**, where each stage transforms the stream and passes results to the next — conceptually like Unix pipes or array methods chained together. It is how you do grouping, joins, reshaping, and analytics that a simple `find` cannot express.

Common stages:

| Stage | Purpose | SQL analogy |
|---|---|---|
| `$match` | filter documents | `WHERE` |
| `$group` | group and aggregate | `GROUP BY` |
| `$project` | reshape / select fields | `SELECT` |
| `$sort` | order results | `ORDER BY` |
| `$limit` / `$skip` | pagination | `LIMIT` / `OFFSET` |
| `$lookup` | join another collection | `JOIN` |
| `$unwind` | flatten an array into multiple docs | — |
| `$count` | count documents | `COUNT(*)` |
| `$addFields` / `$set` | add computed fields | computed column |

Example — total revenue per customer for paid orders, top 5 spenders:

```js
const result = await Order.aggregate([
  { $match: { status: "paid" } },                          // 1. filter
  { $group: {                                              // 2. group by customer
      _id: "$customerId",
      totalSpent: { $sum: "$amount" },
      orderCount: { $sum: 1 },
  } },
  { $sort: { totalSpent: -1 } },                            // 3. highest first
  { $limit: 5 },                                            // 4. top 5
  { $project: { _id: 0, customerId: "$_id", totalSpent: 1, orderCount: 1 } },
]);
```

Key principles:

- Put `$match` (and `$sort` that can use an index) **as early as possible** so later stages process fewer documents and can leverage indexes.
- `$group`'s `_id` is the grouping key; accumulators like `$sum`, `$avg`, `$min`, `$max`, `$push`, `$addToSet` compute per group.
- `$unwind` is essential before grouping by individual array elements.

> TIP: Aggregation results are **plain objects, not Mongoose documents** — no virtuals, no instance methods, no getters. Treat them like raw data.

**Follow-up:** "How would you join with `$lookup`?" — Shown in Q24.

---

### 17. **Embedding vs. referencing: how do you decide how to model relationships?**

MongoDB has no JOINs at the storage layer, so you model relationships in one of two ways:

**Embedding** — nest the related data inside the parent document:

```js
// One blog post document containing its comments
{
  _id: 1,
  title: "Indexing 101",
  comments: [
    { author: "Ada", text: "Great post" },
    { author: "Grace", text: "Thanks!" },
  ],
}
```

**Referencing** — store the related document's `_id` and join at read time:

```js
// post document
{ _id: 1, title: "Indexing 101", author: ObjectId("...") }
// separate users collection
{ _id: ObjectId("..."), name: "Ada" }
```

Decision guide:

| Favor EMBEDDING when... | Favor REFERENCING when... |
|---|---|
| Data is read together almost always | Data is queried independently |
| Strong "contains"/ownership relationship | Many-to-many relationship |
| Child data is bounded in size | The "many" side is large or unbounded |
| You want a single atomic read/write | The same entity is shared by many parents |
| Updates to children are infrequent | Children change often and independently |

Rules of thumb interviewers like to hear:

- **"What is read together should be stored together."** Model around your dominant query.
- Watch the **16 MB document size limit** and **unbounded arrays**. A post with potentially infinite comments should *not* embed them — that array grows without bound and the document keeps getting rewritten.
- A hybrid is common: embed a *summary* (e.g., last 3 comments + count) and reference the full set.

> WARNING — Pitfall: The "unbounded array" anti-pattern (embedding an ever-growing list) is a classic production failure mode. Each push rewrites the document, fragments storage, and eventually risks the 16 MB ceiling.

**Follow-up:** "How would you model a many-to-many like students and courses?" — Reference on both sides (arrays of ids), or use a join collection of enrollment documents if the relationship itself carries data (grade, enrolledAt).

---

### 18. **What is population in Mongoose and how does it relate to refs?**

**Population** is Mongoose's mechanism for automatically replacing a referenced `ObjectId` (a "ref") with the actual document(s) from another collection — effectively an application-level join performed by issuing additional queries.

You declare the relationship with `ref`:

```js
const postSchema = new mongoose.Schema({
  title: String,
  author: { type: mongoose.Schema.Types.ObjectId, ref: "User" }, // ref points to the User model
  tags: [{ type: mongoose.Schema.Types.ObjectId, ref: "Tag" }],
});
const Post = mongoose.model("Post", postSchema);
```

Then populate at query time:

```js
const post = await Post.findById(id)
  .populate("author", "name email")  // only pull name + email from User
  .populate("tags");

console.log(post.author.name); // the User document is now embedded in the result
```

How it works under the hood: Mongoose runs the main query, collects the referenced ids, then runs a **second query** (`$in` on `_id`) against the referenced collection, and stitches the results together. It is **not** a server-side join — it is multiple round-trips orchestrated by Mongoose.

You can also do nested and filtered population:

```js
await Post.find()
  .populate({
    path: "author",
    select: "name",
    populate: { path: "company", select: "name" }, // nested populate
  });
```

> TIP: Always `select` only the fields you need when populating. Pulling entire referenced documents you do not use wastes bandwidth and memory.

**Follow-up:** "Is `populate` a JOIN?" — No; it is one or more extra queries executed by Mongoose, not a single server-side join. For a true server-side join use aggregation `$lookup`.

---

### 19. **What are virtuals in Mongoose? Give a real use case.**

A **virtual** is a property that is **computed**, not stored in MongoDB. It is defined on the schema and derived from other fields when accessed. Virtuals are perfect for values you can compute on the fly and do not want to persist.

```js
const userSchema = new mongoose.Schema({
  firstName: String,
  lastName: String,
});

// Read-only virtual
userSchema.virtual("fullName").get(function () {
  return `${this.firstName} ${this.lastName}`;
});

// Writable virtual (split a full name back into parts)
userSchema.virtual("fullName").set(function (value) {
  const [first, last] = value.split(" ");
  this.firstName = first;
  this.lastName = last;
});

const u = new User({ firstName: "Ada", lastName: "Lovelace" });
console.log(u.fullName); // "Ada Lovelace" — computed, not stored
```

A second major use is **virtual populate** — defining a reference relationship in reverse without storing an array of ids:

```js
// A User has many Posts, but we store author on Post, not posts on User
userSchema.virtual("posts", {
  ref: "Post",
  localField: "_id",
  foreignField: "author",
});
await User.findById(id).populate("posts");
```

> WARNING — Pitfall: Virtuals are **not included in `toJSON()` / `toObject()` output by default**. If your API response is missing `fullName`, enable them: `new Schema({...}, { toJSON: { virtuals: true }, toObject: { virtuals: true } })`. Also, you cannot **query** by a virtual — it does not exist in the database.

**Follow-up:** "Why can't you query by a virtual?" — Because it is computed in JS and never stored, MongoDB has nothing to match against.

---

### 20. **What is the difference between instance methods and static methods in Mongoose?**

- **Instance methods** operate on a **single document** (`this` is the document). Use them for behavior that belongs to one record.
- **Static methods** operate on the **model** (`this` is the model). Use them for collection-wide queries or factory-style helpers.

```js
const userSchema = new mongoose.Schema({ name: String, email: String });

// Instance method — called on a document
userSchema.methods.getInitials = function () {
  return this.name.split(" ").map((p) => p[0]).join("");
};

// Static method — called on the model
userSchema.statics.findByEmail = function (email) {
  return this.findOne({ email }); // `this` is the User model
};

const User = mongoose.model("User", userSchema);

const u = await User.findByEmail("ada@example.com"); // static
console.log(u.getInitials());                         // instance
```

There is a third related concept — **query helpers** — which extend chainable query builders:

```js
userSchema.query.active = function () {
  return this.where({ active: true });
};
await User.find().active(); // chainable custom query
```

> WARNING — Pitfall: Do **not** use arrow functions for methods/statics/hooks. Arrow functions do not bind their own `this`, so `this` will not be the document or model.

**Follow-up:** "When would you choose a static over a query helper?" — A static returns a standalone result; a query helper extends a chain. Use the helper when you want composability with other query methods.

---

## Advanced

### 21. **What is `lean()` in Mongoose, when should you use it, and what do you give up?**

By default, every document a Mongoose query returns is a full **Mongoose document** — a class instance wrapped with getters/setters, virtuals, change tracking, and methods. That hydration has real CPU and memory cost.

`.lean()` tells Mongoose to skip hydration and return **plain JavaScript objects** straight from the driver:

```js
// Heavy: full Mongoose documents
const users = await User.find();

// Light: plain objects, much faster and smaller
const users = await User.find().lean();
```

Benefits:

- Significantly **faster** and **lower memory** — often several times faster for large result sets, because no document instances are constructed.
- Ideal for **read-only** endpoints, list views, reports, and anything you serialize straight to JSON.

What you lose with `lean()`:

- No `save()` or other document methods (it is a plain object).
- No virtuals, getters/setters, or `instanceof` checks (unless you opt back in via plugins).
- No change tracking / no document middleware on the result.
- Custom getters and schema-level transforms do not apply.

> BEST PRACTICE: Use `.lean()` for GET endpoints that only read and serialize data. Do **not** use it when you intend to mutate and `save()` the result, or when you depend on virtuals/methods.

`.lean()` still works with `populate()`, and you can combine it with projection (`.select()`) for maximum efficiency.

**Follow-up:** "Can you still populate with lean?" — Yes; the populated fields come back as plain nested objects rather than Mongoose documents.

---

### 22. **What is the N+1 query problem in this context, and how do you avoid it?**

The **N+1 problem** is when you run **1** query to fetch a list of N parent records, and then **N** additional queries — one per parent — to fetch each parent's related data. Total = N+1 queries, which destroys performance as N grows.

A naive example:

```js
// 1 query for posts
const posts = await Post.find();

// + N queries, one per post — this is the N+1 trap
for (const post of posts) {
  post.authorDoc = await User.findById(post.author);
}
```

If there are 100 posts, that is 101 round-trips.

**Fixes:**

1. **`populate()`** — Mongoose batches the referenced ids and issues a single `$in` query for all authors (turning N+1 into 2 queries):

```js
const posts = await Post.find().populate("author", "name");
```

2. **Aggregation `$lookup`** — a single server-side query that joins the collections (1 round-trip):

```js
const posts = await Post.aggregate([
  { $lookup: { from: "users", localField: "author", foreignField: "_id", as: "author" } },
  { $unwind: "$author" },
]);
```

3. **Manual batching** — collect all ids and do one `find({ _id: { $in: ids } })`, then map in memory. This is essentially what `populate` does and is useful when you need custom control (the "dataloader" pattern).

```js
const posts = await Post.find().lean();
const authorIds = [...new Set(posts.map((p) => String(p.author)))];
const authors = await User.find({ _id: { $in: authorIds } }).lean();
const byId = new Map(authors.map((a) => [String(a._id), a]));
for (const p of posts) p.author = byId.get(String(p.author));
```

> WARNING — Pitfall: N+1 often hides inside ORM/ODM convenience. Awaiting a query inside a loop is the red flag. If you `await` per iteration over a fetched list, stop and batch.

**Follow-up:** "Is `populate` always better than `$lookup`?" — Not always. `populate` is simpler and respects Mongoose schema/virtuals, but it costs an extra round-trip and pulls full documents into Node. `$lookup` does the join server-side in one trip and is better for large/analytical joins, at the cost of returning plain objects.

---

### 23. **Explain MongoDB transactions. What do you need for them and what are the constraints?**

A **multi-document transaction** lets you group several operations across one or more documents/collections so they either **all commit or all roll back** — the ACID "all-or-nothing" guarantee that SQL gives you by default. MongoDB single-document operations have always been atomic; *multi-document* transactions were added in **4.0** (replica sets) and **4.2** (sharded clusters).

Requirements:

- A **replica set** or sharded cluster — transactions do **not** work on a standalone `mongod`. (Atlas and `mongod --replSet` provide this; a single-node replica set is fine for local dev.)
- A **session** that ties the operations together.

Mongoose example with the convenient `withTransaction` helper (handles commit/abort/retry):

```js
const session = await mongoose.startSession();
try {
  await session.withTransaction(async () => {
    // Move $100 from A to B — both updates succeed or both roll back
    await Account.updateOne(
      { _id: aId },
      { $inc: { balance: -100 } },
      { session } // every op MUST pass the session
    );
    await Account.updateOne(
      { _id: bId },
      { $inc: { balance: 100 } },
      { session }
    );
  });
} finally {
  await session.endSession();
}
```

Constraints and gotchas worth mentioning:

- **Every operation inside the transaction must pass the `session`**, or it runs outside the transaction and will not roll back.
- Transactions have a default **60-second time limit**; keep them short.
- They carry **performance overhead** and can cause **write conflicts** under contention (the `withTransaction` helper auto-retries on transient errors).
- Prefer the **single-document atomicity** of a well-designed (embedded) model when possible. The need for multi-document transactions can be a signal that your data model could embed related data instead.

> BEST PRACTICE: Reach for transactions for genuinely cross-document invariants (transfers, inventory + order, multi-collection consistency). For everything that fits in one document, rely on single-document atomic updates with operators like `$inc`/`$set`.

**Follow-up:** "Why don't transactions work on a standalone mongod?" — They depend on the replica set oplog/commit protocol; a standalone has no replication machinery to coordinate the commit.

---

### 24. **How does `$lookup` work and how is it different from `populate`?**

`$lookup` is an aggregation stage that performs a **left outer join** to another collection **on the database server**, in a single query.

```js
const orders = await Order.aggregate([
  {
    $lookup: {
      from: "users",          // the collection to join (raw collection name, pluralized)
      localField: "customerId", // field on Order
      foreignField: "_id",      // field on the joined users
      as: "customer",           // results placed in this array field
    },
  },
  { $unwind: "$customer" },     // turn the 1-element array into an object
  { $project: { amount: 1, "customer.name": 1 } },
]);
```

There is also a more powerful **pipeline form** of `$lookup` that lets you add conditions, projections, and sub-stages to the joined documents:

```js
{
  $lookup: {
    from: "orders",
    let: { uid: "$_id" },
    pipeline: [
      { $match: { $expr: { $eq: ["$customerId", "$$uid"] } } },
      { $match: { status: "paid" } },
      { $project: { amount: 1 } },
    ],
    as: "paidOrders",
  },
}
```

| Aspect | `populate` (Mongoose) | `$lookup` (aggregation) |
|---|---|---|
| Where it runs | Application (Node) | Database server |
| Round-trips | 2+ (extra query per populated path) | 1 |
| Result type | Mongoose documents (virtuals, methods) | Plain objects |
| Schema awareness | Yes (ref, schema) | No — you reference raw collection names |
| Best for | Simple relational reads, small fan-out | Large joins, analytics, server-side filtering/grouping |

> TIP: Use the `from` value as the **actual collection name** (usually the lowercased plural Mongoose generated), not the Mongoose model name. Getting this wrong silently returns empty arrays.

**Follow-up:** "Can `$lookup` join on something other than equality?" — Yes, the pipeline form with `$expr` supports arbitrary conditions, not just equality of two fields.

---

### 25. **Walk through Mongoose middleware (hooks). What is the difference between document and query middleware, and what is a common security/performance use?**

**Middleware** (also called **hooks**) are functions Mongoose runs **before** (`pre`) or **after** (`post`) certain operations. They let you inject logic — hashing a password, setting timestamps, cascading deletes, logging — without scattering it across your codebase.

There are several middleware *types*, and the critical distinction is **document** vs **query** middleware, because `this` means different things and not every operation triggers every hook:

| Middleware type | Hooks on | `this` refers to |
|---|---|---|
| Document | `save`, `validate`, `remove` (and `deleteOne`/`updateOne` as document ops) | the document |
| Query | `find`, `findOne`, `findOneAndUpdate`, `updateOne`, `deleteOne`, `countDocuments`, etc. | the query object |
| Aggregate | `aggregate` | the aggregation object |
| Model | `insertMany` | the model |

A classic, must-know example — **hashing a password before save**:

```js
const bcrypt = require("bcrypt");

userSchema.pre("save", async function (next) {
  // `this` is the document. Only re-hash if the password actually changed.
  if (!this.isModified("password")) return next();
  this.password = await bcrypt.hash(this.password, 12);
  next(); // or simply return/await with async hooks in modern Mongoose
});
```

Another common one — **soft-delete filtering via query middleware** so deleted records never leak:

```js
// Automatically exclude soft-deleted docs from every find*
userSchema.pre(/^find/, function (next) {
  // `this` is the query
  this.where({ isDeleted: { $ne: true } });
  next();
});
```

> WARNING — Pitfall: `pre('save')` does **not** run on `findOneAndUpdate` / `updateOne` / `updateMany`, because those are *query* operations, not document saves. If you hash passwords only in `pre('save')` but allow password changes via `findByIdAndUpdate`, you will store **plaintext passwords**. Either route those updates through `save()`, or add a `pre('findOneAndUpdate')` hook too.

> WARNING — Pitfall: Use `function () {}` not arrow functions in hooks, or `this` will be wrong.

**Follow-up:** "How do you cascade-delete related documents?" — A `pre`/`post` hook on the delete operation that removes dependents (be careful: document `deleteOne` hooks vs. query `deleteOne` hooks differ in what `this` is and whether they fire on `deleteMany`).

---

### 26. **How would you implement efficient pagination on a large collection, and why is `skip` problematic?**

The naive approach is **offset pagination** with `skip` + `limit`:

```js
// Page 3 of 20-per-page
await Post.find().sort({ createdAt: -1 }).skip(40).limit(20);
```

This works but degrades on large collections: to skip 40,000 documents, MongoDB must **walk past all 40,000** before returning the next 20. `skip` is O(offset), so deep pages get progressively slower. It can also show **duplicates or gaps** if documents are inserted/deleted between page loads.

The scalable alternative is **cursor / keyset pagination** ("seek method"): instead of an offset, remember the sort key of the last item seen and ask for items *after* it:

```js
// First page
const page1 = await Post.find()
  .sort({ _id: -1 })   // _id is monotonic-ish and unique; great cursor
  .limit(20);

const lastId = page1[page1.length - 1]._id;

// Next page: everything with _id less than the last one we saw
const page2 = await Post.find({ _id: { $lt: lastId } })
  .sort({ _id: -1 })
  .limit(20);
```

Because `{ _id: { $lt: lastId } }` uses the index directly, every page is **O(limit)** regardless of depth, and pages stay stable under concurrent inserts.

| Approach | Pros | Cons |
|---|---|---|
| `skip`/`limit` | Random page access, simple, "page N" | Slow on deep pages, can duplicate/skip rows |
| Keyset cursor | Constant-time per page, stable | No arbitrary jump to "page 50", needs a unique sort key |

> BEST PRACTICE: Use `skip`/`limit` only for small datasets or admin tables where deep paging is rare. For infinite-scroll feeds and large collections, use keyset pagination on an indexed, unique sort field (often `_id`, or `createdAt` + `_id` as a tiebreaker).

**Follow-up:** "What if you sort by a non-unique field like `createdAt`?" — Add `_id` as a tiebreaker in both the sort and the cursor condition, e.g. `{ createdAt: { $lt: c } }` OR (`createdAt` equal AND `_id < lastId`), so the cursor is deterministic.

---

### 27. **Predict the output: the `findOneAndUpdate` "old document" gotcha.**

Given:

```js
const userSchema = new mongoose.Schema({ name: String, visits: Number });
const User = mongoose.model("User", userSchema);

await User.create({ _id: id, name: "Ada", visits: 0 });

const result = await User.findOneAndUpdate(
  { _id: id },
  { $inc: { visits: 1 } }
  // note: no options passed
);

console.log(result.visits);
```

**What prints, and why?**

It prints **`0`**, not `1`.

By default `findOneAndUpdate` returns the document **before** the update is applied. The increment *does* happen in the database (`visits` becomes `1` on disk), but the returned object reflects the pre-update state.

To get `1`, pass `{ new: true }`:

```js
const result = await User.findOneAndUpdate(
  { _id: id },
  { $inc: { visits: 1 } },
  { new: true } // return the post-update document
);
console.log(result.visits); // 1
```

This is the most-tested Mongoose gotcha precisely because the database state and the returned value diverge, which surprises people. (Note: the **native MongoDB driver's** `findOneAndUpdate` uses `returnDocument: "after" | "before"` instead, defaulting to `"before"` — same concept, different option name.)

**Follow-up:** "How does this behave in the raw driver?" — Same default (returns the original), but the option is `{ returnDocument: "after" }`, not `{ new: true }`.

---

### 28. **How do you optimize a slow MongoDB query? Walk me through your process.**

A structured, senior-sounding answer follows a clear method:

1. **Reproduce and measure.** Identify the exact query and its real parameters. Enable the **slow query log** / database profiler (`db.setProfilingLevel(1, { slowms: 100 })`) to find offenders in production.

2. **Run `explain("executionStats")`.** This is the core tool. Look at:
   - `winningPlan.stage`: is it `IXSCAN` (good) or `COLLSCAN` (bad — full scan)?
   - `totalDocsExamined` vs `nReturned`: a big ratio means you are scanning far more than you return — a missing or poor index.
   - `totalKeysExamined`: how much of the index was walked.
   - `executionTimeMillis`: the wall-clock cost.

```js
const plan = await Post.find({ status: "active" })
  .sort({ createdAt: -1 })
  .explain("executionStats");
// Inspect plan.executionStats.executionStages.stage etc.
```

3. **Add or fix indexes.** Create an index matching the filter + sort, applying the **ESR rule** (Equality, Sort, Range field order). Verify the new plan now uses `IXSCAN` and `totalDocsExamined` drops toward `nReturned`.

4. **Reduce data movement.**
   - **Project** only needed fields (`.select()`), aim for **covered queries**.
   - Use **`.lean()`** for read-only endpoints to skip Mongoose hydration.
   - Avoid pulling huge documents/arrays you do not use.

5. **Fix query shape.**
   - Eliminate **N+1** (batch with `populate`/`$in`/`$lookup`).
   - Replace deep `skip` pagination with **keyset pagination**.
   - Avoid leading-wildcard regexes (`/foo$/` cannot use an index) and unindexed sorts that spill to disk.

6. **Push work to the database.** Use the aggregation pipeline (with early `$match`/`$sort`) instead of fetching everything and computing in Node.

7. **Scale if needed.** Once the query and indexes are sound, consider read replicas for read scaling and **sharding** for horizontal write/data scaling — but only after fixing the query, since scaling a bad query just spreads the pain.

> BEST PRACTICE: The single most valuable habit is reading `explain()` output. "I'd run `explain('executionStats')`, check for COLLSCAN, and compare `totalDocsExamined` to `nReturned`" is the answer interviewers want to hear.

**Follow-up:** "What does a `totalDocsExamined` much larger than `nReturned` tell you?" — The query is scanning many documents to return few; you need a more selective index, or the index field order does not match the query/sort.

---

### 29. **What is the difference between `updateOne`, `replaceOne`, and `save()`, and what are the concurrency implications?**

- **`updateOne(filter, update)`** applies *update operators* (`$set`, `$inc`, ...) atomically to the matched document on the server. It does not load the document into Node. Safe under concurrency for the fields it touches because the operators apply server-side.
- **`replaceOne(filter, doc)`** swaps the *entire* document (except `_id`) with the provided one. Any field you omit is **removed**. Atomic at the document level.
- **`save()`** (Mongoose) sends the changes of an in-memory document you previously loaded. Mongoose tracks which paths changed and, by default, only writes those (`$set` on dirty paths). It runs full document middleware and validation.

The concurrency concern is the **read-modify-write race / lost update**:

```js
// RISKY: two requests can both read balance=100, both write 90, losing one debit
const acct = await Account.findById(id);
acct.balance -= 10;
await acct.save();
```

If two requests interleave between the read and the save, one update is lost. Two safe patterns:

```js
// 1. Atomic operator — let the server do the arithmetic
await Account.updateOne({ _id: id, balance: { $gte: 10 } }, { $inc: { balance: -10 } });

// 2. Optimistic concurrency — version the document and retry on conflict
const schema = new mongoose.Schema({ balance: Number }, { optimisticConcurrency: true });
// save() now checks the __v version and throws VersionError if it changed underneath you
```

> WARNING — Pitfall: `save()` on a stale, in-memory document is the classic lost-update bug. For counters, balances, and inventory, prefer atomic update operators or optimistic concurrency over read-modify-write.

**Follow-up:** "What is `__v`?" — The `versionKey`. Mongoose increments it to detect concurrent modification of arrays and, with `optimisticConcurrency`, to guard whole-document saves.

---

### 30. **How does Mongoose handle schema-less / flexible data, defaults, timestamps, and `strict` mode?**

Even though MongoDB is flexible, Mongoose enforces your schema by default. Several features control how that flexibility is managed:

**`strict` mode (default `true`)** — fields not in the schema are **silently dropped** on save rather than stored:

```js
const s = new mongoose.Schema({ name: String }); // strict: true by default
const u = new User({ name: "Ada", hacker: "rogue field" });
await u.save();
// "hacker" is NOT persisted — Mongoose ignores unknown fields
```

You can set `strict: false` to allow arbitrary fields, or `strict: "throw"` to error on unknown fields. There is also `strictQuery` controlling whether unknown fields in *filters* are stripped.

**Mixed type for genuinely free-form data:**

```js
const schema = new mongoose.Schema({ metadata: mongoose.Schema.Types.Mixed });
// Anything goes in metadata, but Mongoose can't track changes to it automatically:
doc.metadata.foo = "bar";
doc.markModified("metadata"); // REQUIRED so Mongoose knows to persist the change
await doc.save();
```

**Defaults** — static or function-based:

```js
const schema = new mongoose.Schema({
  role: { type: String, default: "user" },
  createdAt: { type: Date, default: Date.now }, // pass the function, don't call it
  apiKey: { type: String, default: () => crypto.randomUUID() },
});
```

**Timestamps** — let Mongoose maintain `createdAt`/`updatedAt`:

```js
const schema = new mongoose.Schema({ name: String }, { timestamps: true });
// Mongoose adds and maintains createdAt and updatedAt automatically,
// including on findOneAndUpdate/updateOne.
```

> WARNING — Pitfall: With `Mixed` type, Mongoose cannot detect in-place mutations. You must call `doc.markModified('path')` or the change will silently not save. Also note `default: Date.now` (reference, evaluated per document) vs `default: Date.now()` (called once at schema definition — every doc gets the same fixed timestamp — a real bug).

**Follow-up:** "Why does `default: Date.now()` give every document the same time?" — Because `Date.now()` is invoked once when the schema is defined; pass the function `Date.now` so Mongoose calls it per document.

---

## Rapid-Fire Quick Answers

| Question | One-line answer |
|---|---|
| What is BSON? | Binary JSON — MongoDB's typed storage format with extra types like `ObjectId`, `Date`, `Decimal128`. |
| What is an `ObjectId` made of? | 12 bytes: 4 timestamp + 5 random + 3 incrementing counter; roughly time-ordered. |
| Collection vs table? | Same idea, but a collection has no enforced column schema. |
| Default return of `findOneAndUpdate`? | The document **before** the update; pass `{ new: true }` to get the updated one. |
| What does `runValidators: true` do? | Runs schema validators on update operations (off by default for `findOneAndUpdate`). |
| `$push` vs `$addToSet`? | `$push` allows duplicates; `$addToSet` only adds if not already present. |
| What does `.lean()` return? | Plain JS objects (no Mongoose document, no virtuals/methods) — faster, lighter, read-only. |
| What is the N+1 problem? | 1 query for a list + N queries for each item's relations; fix with `populate`/`$in`/`$lookup`. |
| `populate` vs `$lookup`? | `populate` = extra app-side queries returning Mongoose docs; `$lookup` = one server-side join returning plain objects. |
| What is a virtual? | A computed, non-persisted field; excluded from JSON unless `toJSON: { virtuals: true }`. |
| Instance vs static method? | Instance acts on one document (`this` = doc); static acts on the model (`this` = model). |
| Does `pre('save')` fire on `findOneAndUpdate`? | No — that is query middleware, not document middleware. |
| When are transactions available? | On replica sets/sharded clusters (4.0+), not standalone; every op must pass the `session`. |
| What does `unique: true` do? | Builds a unique index — it is **not** a validator; duplicates throw error code 11000. |
| COLLSCAN vs IXSCAN in `explain`? | COLLSCAN = full scan (bad); IXSCAN = index used (good); compare `totalDocsExamined` to `nReturned`. |
| ESR index rule? | Order compound-index fields as Equality, then Sort, then Range. |
| Why avoid deep `skip` pagination? | `skip` is O(offset); use keyset (cursor) pagination on an indexed unique field instead. |
| `default: Date.now` vs `Date.now()`? | Pass the function `Date.now` (per-doc); calling `Date.now()` freezes one timestamp at schema-definition time. |
