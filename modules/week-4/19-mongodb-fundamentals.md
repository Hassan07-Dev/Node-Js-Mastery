# MongoDB Fundamentals

Up to this point in the course you have built servers, parsed requests, and routed traffic — but every request handler you wrote so far either invented data on the fly or shoved it into an in-memory array that vanished the moment the process restarted. Real backends are defined by what they *remember*. The database is the long-term memory of your application, and for the JavaScript backend ecosystem, **MongoDB** is the most common first database you'll meet.

This module teaches MongoDB from first principles, assuming you have never touched a database before. We start with the conceptual divide between SQL and NoSQL, build up the document model, get you a running MongoDB instance, and then drill into querying, updating, indexing, aggregation, and data modeling. We work entirely in **`mongosh`** (the MongoDB shell) in this module so that you learn the *database itself* without the abstraction of an ODM in the way. The next module layers Mongoose on top — but you cannot use Mongoose well if you don't first understand what it is hiding from you.

By the end you should be able to design a collection, write non-trivial queries and aggregations, reason about indexes, and answer the data-modeling questions interviewers love to ask.

## What You'll Learn

- The fundamental difference between **relational (SQL)** and **non-relational (NoSQL)** databases, and the specific situations where MongoDB's document model is the right call (and where it isn't).
- The core vocabulary: **databases, collections, documents, BSON, fields**, and the special **`_id` / `ObjectId`** primary key.
- How to set up MongoDB two ways — a **local server** and a free **MongoDB Atlas** cluster — and how to explore data with **`mongosh`** and **Compass**.
- Full CRUD in the shell: `insertOne`/`insertMany`, `find`/`findOne`, `updateOne`/`updateMany`/`replaceOne`, and `deleteOne`/`deleteMany`.
- The complete query operator toolkit: **comparison**, **logical**, **element**, **evaluation (regex)**, and **array** operators.
- The update operator toolkit: `$set`, `$unset`, `$inc`, `$push`, `$pull`, `$addToSet`, and friends.
- Shaping result sets with **projection**, **sorting**, **limit**, and **skip** — and the pagination trap hidden in `skip`.
- **Indexes**: why a query can be 10,000× faster with one, how to create single/compound/unique/text indexes, and how to read `explain()` output.
- The **aggregation pipeline**: thinking in stages with `$match`, `$group`, `$project`, `$sort`, and `$lookup`.
- **Data modeling**: the central NoSQL decision of **embedding vs. referencing**, and the tradeoffs that drive it.

---

## 1. SQL vs. NoSQL and the Document Model

### What a database actually is

Before comparing types, fix the basic idea. A **database** is a program (a *server*) whose entire job is to store data durably on disk, keep it consistent, and let many clients read and write it concurrently and quickly. Your Node app is a *client* of that server. You don't read raw files — you send the database a request ("give me all users older than 30") and it answers, having done the heavy lifting of finding, filtering, and locking data safely.

Databases split broadly into two camps: **relational (SQL)** and **non-relational (NoSQL)**. MongoDB is the flagship of one important NoSQL family: the **document database**.

### The relational (SQL) model

Relational databases — PostgreSQL, MySQL, SQL Server, SQLite — store data in **tables**: rigid grids of **rows** and **columns**. You define a **schema** up front: a `users` table has exactly these columns, each with a fixed type (`name VARCHAR(100)`, `age INTEGER`, `created_at TIMESTAMP`). Every row must conform. Relationships between tables are expressed by **foreign keys** and reassembled at read time using **JOINs**. You talk to them in **SQL**, a declarative query language.

```sql
-- Relational: data spread across normalized tables, joined at query time.
SELECT u.name, o.total
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE u.age > 30;
```

Strengths: rigid schemas enforce data integrity, **JOINs** make multi-entity queries natural, and decades of maturity bring rock-solid **transactions** (ACID guarantees). The cost is rigidity — changing the shape of your data means schema migrations, and a single logical "thing" (an order with its line items, shipping address, and payments) is shattered across many tables.

### The non-relational (NoSQL) model

"NoSQL" is an umbrella for everything that isn't a traditional relational table. It includes several distinct families:

| NoSQL family | Stores data as | Examples | Typical use |
|---|---|---|---|
| **Document** | JSON-like documents | **MongoDB**, Couchbase | General-purpose app data |
| **Key-value** | Simple key → value pairs | Redis, DynamoDB | Caching, sessions, counters |
| **Column-family** | Wide sparse columns | Cassandra, HBase | Huge write-heavy time-series |
| **Graph** | Nodes and edges | Neo4j, Neptune | Social graphs, recommendations |

MongoDB is a **document database**. Instead of rows in tables, you store **documents** — self-contained, JSON-like objects — inside **collections**. A document can hold nested objects and arrays, and two documents in the same collection don't have to look identical. The schema lives in your application, not (by default) in the database.

```js
// Document: a single order with everything it needs, in one place.
{
  _id: ObjectId("..."),
  customer: { name: "Ada Lovelace", email: "ada@example.com" },
  items: [
    { sku: "BOOK-01", qty: 2, price: 19.99 },
    { sku: "PEN-07",  qty: 5, price: 1.50  }
  ],
  total: 47.48,
  status: "shipped",
  createdAt: ISODate("2026-06-01T10:00:00Z")
}
```

Notice what just happened: the order, the customer info, and the line items — three separate tables in the relational world — live in **one document**. To load the whole order, MongoDB reads one document. No JOIN.

### Side-by-side comparison

| Aspect | SQL (Relational) | MongoDB (Document) |
|---|---|---|
| Storage unit | Row in a table | Document in a collection |
| Schema | Fixed, enforced by DB | Flexible, enforced by app (optional DB validation) |
| Structure | Flat columns; types fixed | Nested objects + arrays; types per-document |
| Relationships | Foreign keys + JOINs | Embedding or manual references (`$lookup`) |
| Query language | SQL | Query documents / aggregation pipeline (JavaScript-ish) |
| Scaling model | Primarily vertical; sharding is harder | Horizontal **sharding** is first-class |
| Transactions | Mature, the default unit of work | Supported (multi-document since 4.0) but used more sparingly |
| Schema changes | Migrations, often painful | Add a field whenever; old docs simply lack it |

### When to choose MongoDB (and when not to)

MongoDB is **not** "better than SQL." It makes different tradeoffs. Choosing it well is an interview-grade skill.

**Reach for MongoDB when:**

- Your data is naturally **document-shaped** — a thing and its sub-things travel together (a blog post + its comments, a product + its variants).
- Your **schema evolves rapidly** or varies per record (a CMS, a product catalog where every category has different attributes).
- You need to **scale writes horizontally** across many machines; sharding is built into MongoDB's DNA.
- You want your data to map cleanly onto the JSON your API already speaks, minimizing impedance mismatch.

**Prefer a relational database when:**

- Your data is **highly relational** with many-to-many links you query from every direction (think accounting ledgers, complex inventory).
- You need **strong multi-entity transactional guarantees** as the everyday norm (banking, financial systems).
- You rely on **ad-hoc reporting / complex JOINs** across many entities, or your organization already standardizes on SQL and BI tooling.

> BEST PRACTICE: Don't pick a database by hype. Sketch your top 5 read queries and your top 3 write patterns *first*. The shape of those access patterns — not the shape of your data — should drive the choice. In MongoDB especially, you model for how you *read*, not for some abstract notion of "correct" structure.

> WARNING — Pitfall: "Schemaless" does not mean "no schema." It means the schema is enforced by your application code (and your Mongoose models) instead of the database. Without discipline, a "flexible" collection rots into a swamp of inconsistent documents that are a nightmare to query. Flexibility is a tool, not a license.

---

## 2. Core Concepts: Databases, Collections, Documents, BSON, `_id`

MongoDB's hierarchy is three levels deep. Map it to relational terms as a scaffold (but remember the analogy is imperfect):

| MongoDB | Relational analogue | What it is |
|---|---|---|
| **Database** | Database | A named container for collections |
| **Collection** | Table | A named group of related documents |
| **Document** | Row | One JSON-like record (a BSON object) |
| **Field** | Column | A key/value pair inside a document |

A single MongoDB **server** (or cluster) hosts many **databases**. Each database holds many **collections**. Each collection holds many **documents**.

### Documents

A document is a set of **field: value** pairs, where values can be strings, numbers, booleans, dates, `null`, arrays, or **nested documents** (objects). This nesting is the superpower — a document can contain a whole tree of data.

```js
{
  _id: ObjectId("6650f1a2c3d4e5f6a7b8c9d0"),
  username: "ada",                       // String
  age: 36,                               // Number
  isActive: true,                        // Boolean
  roles: ["admin", "editor"],            // Array of strings
  profile: {                             // Nested document (embedded object)
    firstName: "Ada",
    lastName: "Lovelace",
    social: { twitter: "@ada" }          // ...nested further
  },
  lastLogin: ISODate("2026-06-18T08:30:00Z"), // Date
  metadata: null                         // Null
}
```

Documents in the *same* collection are not required to share fields. You can have one user with a `profile` and another without — MongoDB does not object. (Whether you *should* allow that is a modeling decision.)

### Collections

A collection is just a bucket of documents. It is created lazily — the first time you insert a document into a collection name that doesn't exist, MongoDB creates the collection (and the database) automatically. There is no `CREATE TABLE` step required.

### BSON: what's actually on disk

When you write JSON-looking documents, MongoDB stores them as **BSON** — *Binary JSON*. BSON is a binary-encoded superset of JSON that exists for three reasons:

1. **More types than JSON.** JSON has no date, no 64-bit integer, no binary blob, no `ObjectId`. BSON adds `Date`, `int32`, `int64`, `Decimal128`, `Binary`, `ObjectId`, `Timestamp`, and more. That's why you see `ISODate(...)` and `ObjectId(...)` in the shell — those are BSON types, not plain JSON.
2. **Speed.** Binary encoding is faster to parse and skip-scan than text JSON, and stores length prefixes so the engine can jump around without reading every byte.
3. **Traversability.** Length headers let MongoDB find a nested field quickly.

| Type | JSON has it? | BSON has it? | Shell representation |
|---|---|---|---|
| String, Boolean, Null, Array, Object | Yes | Yes | `"x"`, `true`, `null`, `[]`, `{}` |
| Double (float) | Yes (just "number") | Yes | `3.14` |
| 32/64-bit Integer | No | Yes | `NumberInt(5)`, `NumberLong(5)` |
| High-precision decimal | No | Yes | `NumberDecimal("19.99")` |
| Date | No | Yes | `ISODate("2026-06-19")` |
| ObjectId | No | Yes | `ObjectId("...")` |
| Binary data | No | Yes | `BinData(...)` |

> TIP: For money, use `NumberDecimal` (Decimal128), never a floating-point `Double`. Floats can't represent `0.1 + 0.2` exactly, and rounding errors in financial data are a classic, embarrassing bug.

### `_id` and `ObjectId`

**Every document has a primary key field named `_id`.** It is unique within its collection and immutable once set. If you don't provide one on insert, MongoDB generates an **`ObjectId`** for you automatically.

An `ObjectId` is a 12-byte value, not a random UUID. Its bytes encode:

- **4 bytes** — a Unix timestamp (seconds) of creation.
- **5 bytes** — a random value unique to the machine/process.
- **3 bytes** — an incrementing counter.

Two useful consequences fall out of that structure:

```js
// You can extract the creation time straight from the _id — no separate field needed.
const id = ObjectId("6650f1a2c3d4e5f6a7b8c9d0");
id.getTimestamp();  // ISODate("2024-05-24T...") — when the doc was created

// Because the leading bytes are a timestamp, sorting by _id roughly sorts by insert time.
db.users.find().sort({ _id: -1 }).limit(10);  // ~10 most recently inserted docs
```

You are free to supply your own `_id` (a string, a number, anything unique) if you have a natural key:

```js
db.countries.insertOne({ _id: "US", name: "United States" }); // custom string _id
db.countries.insertOne({ _id: "US", name: "Duplicate" });     // ERROR: duplicate key
```

> WARNING — Pitfall: An `ObjectId` is **not** a string. `ObjectId("abc...") === "abc..."` is `false`. When a query "finds nothing" even though the document clearly exists, the cause is almost always that you passed a string `_id` where an `ObjectId` was needed. Wrap it: `db.users.find({ _id: ObjectId("...") })`.

> BEST PRACTICE: Don't expose raw `ObjectId`s in URLs if you'd rather not leak creation timestamps or document counts. For most apps it's fine, but it is a (small) information leak to be aware of.

---

## 3. Setup: Local MongoDB vs. Atlas, `mongosh`, and Compass

You need a running MongoDB server to follow along. There are two paths; do whichever is easier for you. Atlas (cloud) is the fastest way to a working cluster and is what most teams use in production; local is great for offline tinkering.

### Option A — MongoDB Atlas (managed cloud, free tier)

[MongoDB Atlas](https://www.mongodb.com/atlas) hosts MongoDB for you. The free **M0** tier is enough for learning.

1. Create an account and a free **M0 cluster** (pick a cloud region near you).
2. Under **Database Access**, create a database user with a username and password.
3. Under **Network Access**, add your IP address to the allowlist (for learning you may temporarily allow `0.0.0.0/0`, i.e. anywhere — **never do this in production**).
4. Click **Connect → Drivers / Shell** to copy your connection string. It looks like:

```bash
mongodb+srv://<user>:<password>@cluster0.abcde.mongodb.net/?retryWrites=true&w=majority
```

The `mongodb+srv://` scheme is the **SRV connection string** — it uses DNS to discover all the cluster's nodes automatically.

> WARNING — Pitfall: The connection string contains your password. Never commit it to git. Put it in an environment variable (e.g. `MONGODB_URI` in a `.env` file that is `.gitignore`d). We'll wire this into Node properly in the Mongoose module.

### Option B — Local MongoDB Community Server

Install the **MongoDB Community Edition** on your machine. On macOS with Homebrew it's the quickest:

```bash
# macOS (Homebrew)
brew tap mongodb/brew
brew install mongodb-community            # installs the mongod server
brew services start mongodb-community     # run mongod in the background

# Verify it's listening on the default port 27017
nc -z localhost 27017 && echo "MongoDB is up"
```

On Windows, download the MSI installer from mongodb.com and install it as a service. On Linux, follow the distro-specific apt/yum instructions in the docs. The server program is **`mongod`** (the "d" is for daemon); it listens on port **27017** by default. Your local connection string is simply:

```bash
mongodb://localhost:27017
```

### `mongosh` — the MongoDB Shell

**`mongosh`** is the official interactive shell. It is a full Node.js-based JavaScript REPL wired to a MongoDB connection — meaning you can declare variables, write loops, and use JS expressions, not just run queries. Install it (it ships with the server install, or `brew install mongosh` / download separately) and connect:

```bash
# Connect to a local server
mongosh "mongodb://localhost:27017"

# Connect to Atlas
mongosh "mongodb+srv://<user>:<password>@cluster0.abcde.mongodb.net/"
```

Once inside, a few essential meta-commands:

```js
show dbs                 // list all databases
use shop                 // switch to (and lazily create) the "shop" database
db                       // print the current database name
show collections         // list collections in the current db
db.users.find()          // query the "users" collection in current db
db.dropDatabase()        // delete the current database (careful!)
db.users.drop()          // delete the "users" collection
help                     // built-in help
exit                     // leave the shell
```

The global **`db`** always refers to whatever database `use` last selected. Because `mongosh` is JavaScript, you can do this:

```js
// mongosh is a real JS REPL — store a result and reuse it.
const admin = db.users.findOne({ username: "ada" });
print(admin.email);

// Loop to insert test data
for (let i = 0; i < 5; i++) {
  db.counters.insertOne({ n: i, squared: i * i });
}
```

### Compass — the GUI

**MongoDB Compass** is the official desktop GUI. Paste the same connection string and you get a visual browser: click through databases and collections, view documents as expandable JSON, build queries with a form, see index lists, and run aggregation pipelines stage-by-stage with live previews. It's excellent for *exploring* data and *prototyping* aggregations. Use `mongosh` when you want repeatability and scripting; use Compass when you want to see and click.

| Tool | Best for |
|---|---|
| `mongosh` | Scripting, automation, learning the raw API, CI |
| Compass | Visual exploration, building/debugging aggregations, viewing indexes |
| A driver (Node) | Your actual application code |

---

## 4. CRUD in the Shell

CRUD = **C**reate, **R**ead, **U**pdate, **D**elete. Everything you do to data is one of these four. We'll use a `shop` database with a `products` collection. Start with `use shop`.

### Create — `insertOne` and `insertMany`

```js
use shop  // switch to (and create) the shop database

// insertOne: add a single document. Returns the generated _id.
db.products.insertOne({
  name: "Mechanical Keyboard",
  price: 89.99,
  category: "electronics",
  tags: ["input", "rgb"],
  stock: 12,
  inStock: true
});
// => { acknowledged: true, insertedId: ObjectId("...") }

// insertMany: add many documents at once (one round-trip, much faster than a loop).
db.products.insertMany([
  { name: "Wireless Mouse",  price: 39.99, category: "electronics", tags: ["input"],          stock: 0,  inStock: false },
  { name: "USB-C Cable",     price: 9.99,  category: "accessories", tags: ["cable", "usb-c"], stock: 200, inStock: true  },
  { name: "Laptop Stand",    price: 24.50, category: "accessories", tags: ["ergonomic"],      stock: 30, inStock: true  },
  { name: "4K Monitor",      price: 329.0, category: "electronics", tags: ["display", "4k"],  stock: 5,  inStock: true  }
]);
// => { acknowledged: true, insertedIds: { '0': ObjectId(...), '1': ObjectId(...), ... } }
```

> TIP: `insertMany` runs **ordered** by default — if document #3 fails (e.g. duplicate key), documents #1 and #2 are inserted but #4+ are skipped. Pass `{ ordered: false }` to attempt every document regardless and collect all errors at the end. Use unordered for bulk imports where you want maximum throughput.

### Read — `find` and `findOne`

`find` returns a **cursor** (a lazy iterator over matching documents); `findOne` returns a single document object (or `null`).

```js
// Everything (empty filter = match all)
db.products.find();

// Filter: exact equality. Multiple fields are ANDed together.
db.products.find({ category: "electronics", inStock: true });

// findOne: first match, or null. Use when you expect at most one.
db.products.findOne({ name: "USB-C Cable" });

// Match on a nested field with dot notation (quote the key).
db.users.find({ "profile.social.twitter": "@ada" });

// Count matching documents
db.products.countDocuments({ category: "accessories" }); // => 2
```

> TIP: In `mongosh`, append `.pretty()` to a `find()` to print indented JSON in older shells — modern `mongosh` already pretties output automatically, so you rarely need it now.

### Update — `updateOne`, `updateMany`, `replaceOne`

Updates take **two** arguments: a **filter** (which documents) and an **update document** (what to change, expressed with update operators — covered in depth in §6). The single most common beginner mistake is forgetting the `$set` operator.

```js
// updateOne: change the FIRST matching document.
db.products.updateOne(
  { name: "Wireless Mouse" },          // filter
  { $set: { price: 34.99, stock: 50, inStock: true } } // update operators
);
// => { matchedCount: 1, modifiedCount: 1, ... }

// updateMany: change ALL matching documents.
db.products.updateMany(
  { category: "accessories" },
  { $inc: { price: 1.00 } }            // raise every accessory price by $1
);

// replaceOne: swap the ENTIRE document (except _id). No operators — a literal replacement.
db.products.replaceOne(
  { name: "USB-C Cable" },
  { name: "USB-C Cable 2m", price: 12.99, category: "accessories" }
  // NOTE: any fields not listed here (tags, stock, inStock) are GONE.
);

// upsert: update if found, otherwise insert. Great for "create-or-update".
db.products.updateOne(
  { name: "HDMI Cable" },
  { $set: { price: 7.99, category: "accessories" } },
  { upsert: true }   // no HDMI Cable existed? it gets created.
);
```

> WARNING — Pitfall: `db.products.updateOne({ name: "X" }, { price: 5 })` — without `$set` — does **not** set the price to 5. In MongoDB 4.x it threw an error; in modern versions an update document with no operators is treated as a **full replacement**, silently wiping every other field. Always use `$set` (or another operator) unless you truly mean to replace.

### Delete — `deleteOne` and `deleteMany`

```js
// deleteOne: remove the FIRST matching document.
db.products.deleteOne({ name: "Laptop Stand" });
// => { acknowledged: true, deletedCount: 1 }

// deleteMany: remove ALL matching documents.
db.products.deleteMany({ inStock: false });

// deleteMany with empty filter: WIPES THE ENTIRE COLLECTION. Be very sure.
db.products.deleteMany({});
```

> WARNING — Pitfall: `deleteMany({})` and `updateMany({}, ...)` with an empty filter hit *every* document in the collection. There is no "are you sure?" prompt and no recycle bin. Always double-check your filter before running a `Many` operation. In production, this is how junior engineers wipe tables. Test the filter with a `find()` first.

### CRUD return-value summary

| Method | Returns |
|---|---|
| `insertOne` | `{ acknowledged, insertedId }` |
| `insertMany` | `{ acknowledged, insertedIds }` |
| `find` | a **cursor** |
| `findOne` | a document or `null` |
| `updateOne` / `updateMany` | `{ matchedCount, modifiedCount, upsertedId? }` |
| `replaceOne` | `{ matchedCount, modifiedCount }` |
| `deleteOne` / `deleteMany` | `{ acknowledged, deletedCount }` |

> TIP: Note the distinction between `matchedCount` (how many documents the filter found) and `modifiedCount` (how many actually changed). If you `$set` a field to the value it already has, it matches but doesn't modify.

---

## 5. Query Operators

So far our filters used exact equality. The real power is in **query operators** — special `$`-prefixed keys that express conditions richer than "equals." The general shape is `{ field: { $operator: value } }`.

Seed data for this section:

```js
db.users.insertMany([
  { name: "Ada",   age: 36, city: "London",   roles: ["admin", "editor"], score: 95, email: "ada@x.com" },
  { name: "Linus", age: 54, city: "Portland",  roles: ["admin"],          score: 88 },
  { name: "Grace", age: 41, city: "New York",  roles: ["editor"],         score: 72, email: "grace@x.com" },
  { name: "Alan",  age: 41, city: "London",    roles: [],                 score: 60, email: "alan@x.com" },
  { name: "Edsger",age: 29, city: "Austin",    roles: ["viewer", "editor"], score: 81 }
]);
```

### Comparison operators

| Operator | Meaning | Example |
|---|---|---|
| `$eq` | equals (the implicit default) | `{ age: { $eq: 41 } }` |
| `$ne` | not equal | `{ city: { $ne: "London" } }` |
| `$gt` | greater than | `{ age: { $gt: 40 } }` |
| `$gte` | greater than or equal | `{ score: { $gte: 80 } }` |
| `$lt` | less than | `{ age: { $lt: 30 } }` |
| `$lte` | less than or equal | `{ age: { $lte: 36 } }` |
| `$in` | matches any value in an array | `{ city: { $in: ["London", "Austin"] } }` |
| `$nin` | matches none of the values | `{ city: { $nin: ["London"] } }` |

```js
// Users aged between 30 and 50 (inclusive). Combine operators on one field.
db.users.find({ age: { $gte: 30, $lte: 50 } });

// Users in London or Austin
db.users.find({ city: { $in: ["London", "Austin"] } });

// $in also works against an ARRAY field: "has at least one of these roles".
db.users.find({ roles: { $in: ["admin", "viewer"] } }); // Ada, Linus, Edsger
```

### Logical operators

Combine multiple conditions. Note that listing several fields in one filter object is already an implicit **AND**; the explicit operators are for ORs and more complex logic.

| Operator | Meaning |
|---|---|
| `$and` | all conditions true (usually implicit) |
| `$or` | at least one condition true |
| `$not` | inverts a single operator expression |
| `$nor` | none of the conditions true |

```js
// $or: in London OR has a high score
db.users.find({
  $or: [
    { city: "London" },
    { score: { $gte: 90 } }
  ]
});

// Explicit $and — needed when the SAME field has two conditions that can't be
// expressed in one sub-object, or when combining $or branches with other clauses.
db.users.find({
  $and: [
    { $or: [{ city: "London" }, { city: "Austin" }] },
    { age: { $lt: 40 } }
  ]
});

// $not: NOT (age > 40)  ->  age <= 40 (and also matches docs missing the field!)
db.users.find({ age: { $not: { $gt: 40 } } });

// $nor: neither admin role nor score >= 90
db.users.find({
  $nor: [
    { roles: "admin" },
    { score: { $gte: 90 } }
  ]
});
```

> WARNING — Pitfall: `$not` and `$ne` also match documents where the field is **missing**. `{ age: { $not: { $gt: 40 } } }` returns users with `age <= 40` *and* users with no `age` field at all, because "not greater than 40" is vacuously true for a non-existent value. If you mean "exists and is <= 40," add `{ age: { $exists: true } }`.

### Element operators

Query based on a field's existence or BSON type.

| Operator | Meaning | Example |
|---|---|---|
| `$exists` | field is present (or absent) | `{ email: { $exists: true } }` |
| `$type` | field is a specific BSON type | `{ score: { $type: "int" } }` |

```js
// Users who have an email field at all
db.users.find({ email: { $exists: true } }); // Ada, Grace, Alan

// Users MISSING an email field
db.users.find({ email: { $exists: false } }); // Linus, Edsger

// Type checking — find docs where "score" is stored as a string by mistake
db.users.find({ score: { $type: "string" } });

// $type accepts string aliases ("double", "string", "object", "array", "bool",
// "date", "int", "long", "decimal", "objectId") or numeric type codes.
db.users.find({ roles: { $type: "array" } });
```

### Evaluation operators — `$regex`

`$regex` matches string fields against a regular expression.

```js
// Names starting with "A" (case-sensitive)
db.users.find({ name: { $regex: /^A/ } });

// Case-insensitive: emails containing "x.com"
db.users.find({ email: { $regex: "x\\.com", $options: "i" } });

// Shorthand: a bare regex literal also works
db.users.find({ name: /^A/i });
```

> WARNING — Pitfall: A `$regex` that is *not* anchored to the start of the string (e.g. `/foo/` or `/.*foo/`) **cannot use an index efficiently** — MongoDB must scan every document. Only left-anchored, case-sensitive patterns like `/^foo/` use an index. For full-text search needs, prefer a **text index** (§8) or Atlas Search rather than abusing `$regex`.

### Array operators

Querying arrays is where MongoDB documents really shine — and where the semantics surprise newcomers.

| Operator | Meaning |
|---|---|
| `$all` | array contains **all** of the listed values |
| `$elemMatch` | **at least one** array element matches **all** the given conditions |
| `$size` | array has exactly N elements |

```js
// First, the implicit array-match rules:

// Match if the roles array CONTAINS "admin" (membership, not equality).
db.users.find({ roles: "admin" }); // Ada, Linus

// Match the array EXACTLY (same elements, same order). Rarely what you want.
db.users.find({ roles: ["admin", "editor"] }); // only Ada

// $all: contains BOTH admin AND editor (order/extra elements don't matter).
db.users.find({ roles: { $all: ["admin", "editor"] } }); // Ada

// $size: exactly 2 roles
db.users.find({ roles: { $size: 2 } }); // Ada, Edsger
```

`$elemMatch` is the subtle one. Consider an array of *sub-documents*:

```js
db.students.insertMany([
  { name: "A", grades: [{ subject: "math", score: 90 }, { subject: "art", score: 60 }] },
  { name: "B", grades: [{ subject: "math", score: 50 }, { subject: "art", score: 95 }] }
]);

// WRONG INTENT: "a math grade above 80". This actually matches student B too!
// Because the two conditions can be satisfied by DIFFERENT array elements:
// B has subject=math (in elem 0) AND score>80 (in elem 1).
db.students.find({ "grades.subject": "math", "grades.score": { $gt: 80 } }); // A and B

// RIGHT: $elemMatch requires ONE SINGLE element to satisfy ALL conditions.
db.students.find({
  grades: { $elemMatch: { subject: "math", score: { $gt: 80 } } }
}); // only A
```

> BEST PRACTICE: Whenever you filter an array of sub-documents on **two or more conditions that must hold for the same element**, use `$elemMatch`. Without it, conditions can be spread across different elements and you'll get false positives. This is a favorite interview trap.

---

## 6. Update Operators

Update operators (used inside `updateOne`/`updateMany`) modify fields in place without you having to read, mutate, and write back the whole document. This is both faster and atomic — MongoDB applies the operator as a single, indivisible operation, which avoids lost-update race conditions.

### Field update operators

| Operator | Effect |
|---|---|
| `$set` | set a field to a value (creating it if absent) |
| `$unset` | remove a field entirely |
| `$inc` | atomically add a number (use negatives to subtract) |
| `$mul` | multiply a numeric field |
| `$rename` | rename a field |
| `$min` / `$max` | set only if the new value is smaller/larger |
| `$currentDate` | set a field to the current date/time |

```js
// $set: change/add fields. Dot notation reaches into nested docs.
db.users.updateOne(
  { name: "Ada" },
  { $set: { city: "Cambridge", "profile.verified": true } }
);

// $unset: drop a field (the value you give is ignored — convention is "").
db.users.updateOne({ name: "Ada" }, { $unset: { email: "" } });

// $inc: atomic counter. Add 1 to loginCount, subtract 5 from credits.
db.users.updateOne(
  { name: "Linus" },
  { $inc: { loginCount: 1, credits: -5 } }
);

// $rename: "city" becomes "location" for everyone.
db.users.updateMany({}, { $rename: { city: "location" } });

// $currentDate: stamp the update time.
db.users.updateOne({ name: "Grace" }, { $currentDate: { lastModified: true } });
```

> BEST PRACTICE: For counters (likes, views, stock levels), always use `$inc` rather than reading the value into your app, adding 1, and writing it back. The read-modify-write approach has a race condition: two concurrent requests can both read `10`, both write `11`, and you lose an increment. `$inc` is atomic on the server and immune to this.

### Array update operators

| Operator | Effect |
|---|---|
| `$push` | append an element (allows duplicates) |
| `$addToSet` | append only if not already present (set semantics) |
| `$pull` | remove all elements matching a condition |
| `$pop` | remove the first (`-1`) or last (`1`) element |
| `$each` | modifier: push/add multiple values at once |

```js
// $push: add a role (even if it duplicates an existing one).
db.users.updateOne({ name: "Grace" }, { $push: { roles: "admin" } });

// $push + $each: add several at once.
db.users.updateOne(
  { name: "Grace" },
  { $push: { roles: { $each: ["viewer", "auditor"] } } }
);

// $addToSet: add "editor" ONLY if not already in the array (no duplicates).
db.users.updateOne({ name: "Linus" }, { $addToSet: { roles: "editor" } });

// $pull: remove every "viewer" from the roles array.
db.users.updateOne({ name: "Grace" }, { $pull: { roles: "viewer" } });

// $pull with a condition: remove all roles starting with "a".
db.users.updateOne({ name: "Edsger" }, { $pull: { roles: { $regex: /^a/ } } });

// $pop: remove the last element (1) or the first (-1).
db.users.updateOne({ name: "Grace" }, { $pop: { roles: 1 } });
```

> TIP: `$push` allows duplicates; `$addToSet` enforces uniqueness. Use `$addToSet` for things like tags, role lists, or "users who liked this," where the same value should never appear twice.

---

## 7. Projection, Sorting, Limit, and Skip

A `find()` returns full documents by default. **Projection**, **sort**, **limit**, and **skip** shape *what* comes back and *how much* — crucial for performance and for building APIs.

### Projection — choosing fields

The optional **second argument** to `find` (or a `.project()` on the cursor) selects which fields to return. `1` = include, `0` = exclude.

```js
// Include only name and score (the _id is included by default).
db.users.find({}, { name: 1, score: 1 });

// Include name and score, and explicitly drop _id.
db.users.find({}, { name: 1, score: 1, _id: 0 });

// Exclusion projection: return everything EXCEPT roles and email.
db.users.find({}, { roles: 0, email: 0 });
```

> WARNING — Pitfall: You cannot mix inclusion and exclusion in the same projection (except for `_id`, which you may always exclude). `{ name: 1, email: 0 }` throws an error. Choose one mode: either list the fields you want (inclusion) or the fields you don't (exclusion).

> BEST PRACTICE: Always project down to the fields your API response actually needs. Returning a 50-field document when the client needs 3 wastes bandwidth, leaks internal fields, and prevents **covered queries** (where the index alone answers the query without touching documents — see §8).

### Sorting, limit, and skip

These are **cursor methods** chained onto a `find()`. They are applied on the server.

```js
// Sort by score descending (-1), then by name ascending (1) as a tie-breaker.
db.users.find().sort({ score: -1, name: 1 });

// Top 3 highest scorers.
db.users.find().sort({ score: -1 }).limit(3);

// skip + limit: classic (naive) pagination — page 2, 10 per page.
db.users.find().sort({ score: -1 }).skip(10).limit(10);
```

> WARNING — Pitfall: `skip` does **not** jump straight to the Nth document — the server must walk and discard all the skipped documents first. `skip(100000)` reads and throws away 100,000 docs every time. For deep pagination this is brutally slow. Use **range-based ("keyset") pagination** instead: remember the last value seen and query for the next page.

```js
// Keyset pagination: far faster than skip for deep pages.
// Page 1:
const page1 = db.users.find().sort({ _id: 1 }).limit(10).toArray();
const lastId = page1[page1.length - 1]._id;
// Page 2: ask for everything AFTER the last id we saw.
db.users.find({ _id: { $gt: lastId } }).sort({ _id: 1 }).limit(10);
```

> TIP: The order of cursor methods in your code doesn't matter (`.sort().limit()` vs `.limit().sort()`) — MongoDB always applies the logical order **sort → skip → limit**. But for clarity, write them in that order.

---

## 8. Indexes

This is the single most impactful performance topic in MongoDB, and a guaranteed interview subject.

### Why indexes matter

Without an index, answering `db.users.find({ email: "ada@x.com" })` forces a **collection scan (`COLLSCAN`)**: MongoDB examines *every document* in the collection to check the email. On 10 documents that's instant. On 10 million documents it's a disaster. An **index** is a separate, sorted data structure (a B-tree) that maps field values to the documents that contain them — exactly like the index at the back of a textbook lets you find a topic without reading every page.

With an index on `email`, MongoDB does an **index scan (`IXSCAN`)**: it binary-searches the sorted index to the matching entry and jumps straight to the document. The difference between scanning a million documents and doing a handful of B-tree hops is the difference between a 2-second query and a 2-millisecond one.

The tradeoff: indexes **cost write performance and disk space**. Every insert/update/delete must also update every relevant index. So you index the fields you *query and sort on*, not every field.

> The `_id` field is **automatically indexed** with a unique index on every collection. You never create that one yourself.

### Creating indexes

```js
// Single-field index. 1 = ascending order, -1 = descending.
db.users.createIndex({ email: 1 });

// Compound index: indexes the COMBINATION of fields, in this order.
db.users.createIndex({ city: 1, score: -1 });

// Unique index: rejects duplicate values (great for emails, usernames).
db.users.createIndex({ email: 1 }, { unique: true });

// List all indexes on a collection
db.users.getIndexes();

// Drop an index by name
db.users.dropIndex("email_1");
```

### Compound indexes and the prefix rule

A compound index on `{ city: 1, score: -1 }` can efficiently serve queries that filter on:

- `city` alone, OR
- `city` **and** `score` together.

It **cannot** efficiently serve a query on `score` alone. This is the **prefix rule**: a compound index supports queries that use a **left-prefix** of its fields. Think of a phone book sorted by (last name, first name): you can look up "everyone named Smith" or "John Smith" instantly, but not "everyone named John" — those Johns are scattered throughout.

| Query | Uses `{ city:1, score:-1 }` index? |
|---|---|
| `{ city: "London" }` | Yes (left prefix) |
| `{ city: "London", score: { $gt: 80 } }` | Yes (full prefix) |
| `{ score: { $gt: 80 } }` | No — `score` is not a prefix |
| `.sort({ city: 1, score: -1 })` | Yes — index already in this order |

> BEST PRACTICE: Order compound-index fields by the **ESR rule** — **E**quality fields first, then **S**ort fields, then **R**ange fields. A query like `find({ city: "London", age: { $gt: 30 } }).sort({ score: -1 })` is best served by an index `{ city: 1, score: -1, age: 1 }` (equality `city`, sort `score`, range `age`).

### Text indexes

For word-based searching across string fields, create a **text index**. It tokenizes, lowercases, and stems words.

```js
// A text index over the name and description fields.
db.products.createIndex({ name: "text", description: "text" });

// Use $text + $search to query it. Searches for the WORD "wireless".
db.products.find({ $text: { $search: "wireless keyboard" } });

// Sort by relevance using the textScore meta field.
db.products.find(
  { $text: { $search: "wireless" } },
  { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } });
```

> TIP: A collection can have **only one** text index, but it may span multiple fields. For richer full-text needs (fuzzy matching, autocomplete, multi-language relevance), use **Atlas Search** (Lucene-backed) rather than the basic text index.

### Reading `explain()`

`explain()` tells you *how* MongoDB plans to run a query — most importantly, whether it uses an index. Append it to a query, with `"executionStats"` for real timing.

```js
db.users.find({ email: "ada@x.com" }).explain("executionStats");
```

The fields to read first:

- **`winningPlan.stage`** — `IXSCAN` (good, used an index) vs `COLLSCAN` (bad, scanned everything).
- **`executionStats.nReturned`** — how many documents matched.
- **`executionStats.totalDocsExamined`** — how many documents MongoDB had to inspect.
- **`executionStats.totalKeysExamined`** — index entries inspected.

```js
// Healthy: nReturned == totalDocsExamined, stage IXSCAN.
// 1 returned, 1 examined — perfectly selective index.

// Unhealthy: nReturned == 1 but totalDocsExamined == 1000000, stage COLLSCAN.
// You scanned a million docs to return one — you NEED an index here.
```

> BEST PRACTICE: The golden ratio to chase is `totalDocsExamined` ≈ `nReturned`. If MongoDB examines far more documents than it returns, the query is doing wasted work and likely needs a better index. A `COLLSCAN` in `executionStats` on a large collection is a red flag in code review.

---

## 9. The Aggregation Pipeline

Plain `find()` filters and returns documents. The **aggregation pipeline** *transforms and computes over* them — grouping, summing, reshaping, joining. It's MongoDB's answer to SQL's `GROUP BY`, aggregate functions, and JOINs combined, and it's how you build analytics, reports, and dashboards.

### The mental model: a conveyor belt of stages

`aggregate()` takes an **array of stages**. Documents flow in at the top; each stage transforms the stream and passes its output to the next. Think of Unix pipes (`cat | grep | sort | head`) — each stage does one job.

```js
db.collection.aggregate([
  { stage1 },
  { stage2 },
  { stage3 }
]);
```

Seed data — an `orders` collection:

```js
db.orders.insertMany([
  { customer: "Ada",   item: "Keyboard", category: "electronics", qty: 1, price: 90, status: "shipped" },
  { customer: "Ada",   item: "Mouse",    category: "electronics", qty: 2, price: 35, status: "shipped" },
  { customer: "Linus", item: "Cable",    category: "accessories", qty: 5, price: 10, status: "pending" },
  { customer: "Grace", item: "Monitor",  category: "electronics", qty: 1, price: 330, status: "shipped" },
  { customer: "Grace", item: "Stand",    category: "accessories", qty: 1, price: 25, status: "shipped" }
]);
```

### `$match` — filter

`$match` filters documents using the same syntax as `find`. Put it **first** so later stages process fewer documents (and so it can use an index).

```js
db.orders.aggregate([
  { $match: { status: "shipped" } }
]);
```

### `$group` — aggregate

`$group` collapses many documents into groups by a key (`_id`), computing accumulator values per group. The `_id` field of the output is the grouping key (use `null` to group everything into one bucket).

```js
// Total revenue and order count per category.
db.orders.aggregate([
  { $match: { status: "shipped" } },
  {
    $group: {
      _id: "$category",                          // group key (the $ means "field value")
      revenue: { $sum: { $multiply: ["$qty", "$price"] } }, // sum qty*price per group
      orderCount: { $sum: 1 },                   // count docs in the group
      avgPrice: { $avg: "$price" }               // average price
    }
  }
]);
// => { _id: "electronics", revenue: 490, orderCount: 3, avgPrice: ... }, ...
```

Common accumulators: `$sum`, `$avg`, `$min`, `$max`, `$first`, `$last`, `$push` (collect into an array), `$addToSet` (collect uniques).

> TIP: Inside aggregation expressions, a field reference is written as a **string starting with `$`** — `"$price"` means "the value of the price field." A bare `"price"` would be the literal string. This `$`-prefix-for-field is the #1 aggregation syntax confusion.

### `$project` — reshape

`$project` chooses, renames, and computes fields — like projection in `find`, but with computed expressions.

```js
db.orders.aggregate([
  {
    $project: {
      _id: 0,
      customer: 1,
      item: 1,
      lineTotal: { $multiply: ["$qty", "$price"] }, // computed field
      isBulk: { $gte: ["$qty", 3] }                 // computed boolean
    }
  }
]);
```

### `$sort`, `$limit`, `$skip`

```js
// Top 2 categories by revenue.
db.orders.aggregate([
  { $group: { _id: "$category", revenue: { $sum: { $multiply: ["$qty", "$price"] } } } },
  { $sort: { revenue: -1 } },
  { $limit: 2 }
]);
```

### `$lookup` — the JOIN

`$lookup` performs a left outer join to another collection — this is how you "reference" data across collections (§10). Given a `customers` collection:

```js
db.customers.insertMany([
  { _id: "Ada",   tier: "gold",   country: "UK" },
  { _id: "Linus", tier: "silver", country: "US" },
  { _id: "Grace", tier: "gold",   country: "US" }
]);

// Join each order to its customer document.
db.orders.aggregate([
  {
    $lookup: {
      from: "customers",          // the OTHER collection
      localField: "customer",     // field in orders
      foreignField: "_id",        // field in customers to match
      as: "customerInfo"          // output array field
    }
  },
  // $lookup always produces an ARRAY (since a match could be 0..many).
  // Unwind it to flatten the single match into an object.
  { $unwind: "$customerInfo" },
  { $project: { item: 1, customer: 1, tier: "$customerInfo.tier" } }
]);
```

> WARNING — Pitfall: `$lookup` puts results in an **array** named by `as`, even when there's exactly one match — because a join can return zero or many. Use `$unwind` to flatten it, or index into `arrayField[0]`. Forgetting this is why `$lookup` output "looks weird."

> BEST PRACTICE: Always lead an aggregation with `$match` (and `$project` to drop unused fields) **before** expensive stages like `$group`, `$lookup`, and `$sort`. Filtering early means later stages handle far fewer, smaller documents — and an early `$match` can use an index, while one buried mid-pipeline cannot.

A complete, realistic pipeline tying it together:

```js
// "For shipped orders, total revenue per customer tier, highest first."
db.orders.aggregate([
  { $match: { status: "shipped" } },                          // 1. filter early
  { $lookup: { from: "customers", localField: "customer",
               foreignField: "_id", as: "c" } },              // 2. join customers
  { $unwind: "$c" },                                          // 3. flatten join
  { $group: {                                                 // 4. aggregate
      _id: "$c.tier",
      revenue: { $sum: { $multiply: ["$qty", "$price"] } },
      customers: { $addToSet: "$customer" }
  }},
  { $sort: { revenue: -1 } }                                  // 5. order
]);
```

---

## 10. Data Modeling: Embedding vs. Referencing

This is the heart of MongoDB design and the topic interviewers probe hardest, because it's where MongoDB diverges most from SQL. There is no JOIN-by-default and no foreign-key enforcement, so *you* decide how related data is laid out. There are two strategies.

### Embedding (denormalization)

Put related data **inside** the parent document as a nested object or array.

```js
// Embedded: a blog post with its comments INSIDE it.
{
  _id: ObjectId("..."),
  title: "Intro to MongoDB",
  author: "Ada",
  comments: [
    { user: "Linus", text: "Great post!", at: ISODate("2026-06-01") },
    { user: "Grace", text: "Thanks!",     at: ISODate("2026-06-02") }
  ]
}
```

**Pros:** One read fetches everything (no JOIN). Atomic updates to the whole document. Great read performance. Data that's always accessed together stays together.

**Cons:** Documents can grow unbounded (the **16 MB document size limit** is a hard ceiling). Duplicated data can get out of sync. Hard to query embedded items in isolation across all parents.

### Referencing (normalization)

Store related data in a **separate collection** and link by `_id`, joining with `$lookup` (or a second query) when needed.

```js
// Referenced: posts and comments in separate collections.
// posts:
{ _id: ObjectId("post1"), title: "Intro to MongoDB", author: "Ada" }
// comments:
{ _id: ObjectId("c1"), postId: ObjectId("post1"), user: "Linus", text: "Great post!" }
{ _id: ObjectId("c2"), postId: ObjectId("post1"), user: "Grace", text: "Thanks!" }
```

**Pros:** No duplication; one source of truth. Parent documents stay small. Sub-items can grow without bound and be queried independently.

**Cons:** Reading the full picture needs a `$lookup` or extra round-trips. No DB-enforced referential integrity (a `postId` can point to a deleted post).

### How to decide

| Question | Lean **Embed** if... | Lean **Reference** if... |
|---|---|---|
| Are they read together? | Almost always together | Often read separately |
| Relationship cardinality | One-to-few (a few, bounded) | One-to-many/very-many (unbounded) |
| Does the child grow forever? | No, bounded | Yes (comments, logs, events) |
| Is the child a shared entity? | No, owned by one parent | Yes, referenced by many parents |
| Update frequency of child | Rarely changes | Changes often / independently |
| Could it blow the 16 MB limit? | No | Risk of large/growing arrays |

The guiding heuristics, in plain terms:

1. **"Data that is queried together should be stored together."** Model for your read patterns first.
2. **One-to-few → embed. One-to-many → reference.** (A user's two or three addresses: embed. A user's millions of activity-log entries: reference.)
3. **High-cardinality, unbounded, or independently-queried children → reference.** Never embed an array that grows without limit; you'll hit 16 MB and your updates will get slower as the array grows.
4. **Hybrid is allowed and common.** Embed a *summary* (e.g. `author: { id, name }` on a post) for fast reads, and keep the full entity in its own collection as the source of truth. This is the **extended/subset reference pattern** and is the pragmatic real-world default.

```js
// Hybrid: embed a tiny denormalized summary for display, reference for the full record.
{
  _id: ObjectId("..."),
  title: "Intro to MongoDB",
  author: { _id: ObjectId("u1"), name: "Ada" }, // summary embedded for fast list views
  // ...full author lives in the users collection, fetched only when needed
}
```

> BEST PRACTICE: Start by **embedding** unless you have a concrete reason to reference (unbounded growth, shared ownership, independent querying, or the 16 MB limit). Embedding is MongoDB's natural grain and usually yields the simplest, fastest reads. Reach for references when one of those specific pressures forces your hand.

> WARNING — Pitfall: MongoDB does **not** cascade deletes or enforce that a reference points to a real document. If you delete a `user` document, any `posts` that reference it become orphans pointing at nothing. Your *application* must maintain referential integrity — either clean up references on delete, or tolerate dangling ones gracefully.

---

## Common Pitfalls & Gotchas

- **Comparing `_id` as a string.** `ObjectId("...") !== "..."`. Always wrap string ids in `ObjectId()` before querying `_id`. The "query returns nothing but the doc exists" bug is almost always this.
- **Updating without `$set`.** `updateOne(filter, { field: value })` is treated as a full replacement and wipes every other field. Always use an operator (`$set`, `$inc`, …).
- **`updateMany({})` / `deleteMany({})` with an empty filter.** This silently hits every document in the collection. Test the filter with `find()` first.
- **`$ne` and `$not` also match missing fields.** "Not equal to X" is true for documents lacking the field entirely. Add `$exists: true` if you mean "present and not X."
- **`$elemMatch` confusion.** Multiple conditions on an array of sub-documents without `$elemMatch` can be satisfied by *different* elements, causing false positives.
- **`$lookup` returns an array.** Even a single join match comes back wrapped in an array; `$unwind` it or read `[0]`.
- **Forgetting the `$` on field references in aggregation.** `"$price"` is the field's value; `"price"` is the literal string `"price"`.
- **Unanchored `$regex`.** A regex not anchored with `^` cannot use an index and forces a full scan.
- **Deep `skip()` pagination.** `skip(N)` walks and discards N documents every call. Use keyset pagination for large offsets.
- **Mixing inclusion and exclusion in a projection.** `{ a: 1, b: 0 }` errors (except for `_id`).
- **Floats for money.** Use `NumberDecimal`, not `Double`, for currency.
- **Unbounded embedded arrays.** Arrays that grow forever eventually hit the 16 MB document limit and slow down every update. Reference instead.
- **Over-indexing.** Every index slows writes and consumes RAM/disk. Index for real query patterns, then verify with `explain()` — don't index "just in case."

## Best Practices

- **Model for your queries, not your data.** Write down your top read and write patterns before designing collections, then shape documents to serve them.
- **Embed by default; reference when growth, sharing, or independent access demands it.** One-to-few embeds, one-to-many references.
- **Always project** down to the fields the client needs; never ship whole documents by reflex.
- **Index the fields you filter and sort on,** order compound indexes by the **ESR rule** (Equality, Sort, Range), and verify with `explain("executionStats")` that `totalDocsExamined` ≈ `nReturned`.
- **Use atomic update operators** (`$inc`, `$push`, `$addToSet`) instead of read-modify-write to avoid race conditions.
- **Put `$match` (and `$project`) first** in aggregation pipelines so downstream stages process fewer, smaller documents and can leverage indexes.
- **Enforce uniqueness with a `unique` index,** not just application logic — concurrent requests can both pass an app-level "does this email exist?" check.
- **Keep connection strings and credentials in environment variables,** never in code or git.
- **Use `NumberDecimal` for money** and store dates as BSON `Date`, not strings, so range queries and sorting work correctly.
- **Add schema validation** (JSON Schema validators, or Mongoose in the next module) so "flexible" doesn't become "inconsistent."

## Key Interview Points

- **"What's the difference between SQL and NoSQL / when would you use MongoDB?"** SQL = rigid tabular schema with JOINs and strong transactions; MongoDB = flexible JSON-like documents, horizontal scaling, no JOIN by default. Choose MongoDB for document-shaped, rapidly-evolving, write-scaling data; choose SQL for highly relational data needing complex JOINs and transactional integrity.
- **"What is BSON and why not just JSON?"** Binary JSON — adds types JSON lacks (Date, ObjectId, Decimal128, 64-bit ints, Binary), parses faster, and uses length prefixes for fast traversal.
- **"What's an ObjectId made of?"** 12 bytes: 4-byte creation timestamp + 5-byte random (machine/process) + 3-byte incrementing counter. Therefore roughly time-sortable and carries a creation time via `getTimestamp()`.
- **"Embedding vs. referencing — how do you decide?"** Embed one-to-few data read together and bounded in size; reference one-to-many/unbounded, shared, or independently-queried data. Watch the 16 MB document limit. Hybrid (subset reference) is common.
- **"How do indexes work and what's a compound index prefix?"** An index is a sorted B-tree mapping values to documents, turning COLLSCAN into IXSCAN. A compound index serves queries on a **left-prefix** of its fields; order fields by ESR (Equality, Sort, Range).
- **"How do you diagnose a slow query?"** `explain("executionStats")`: check for `COLLSCAN` vs `IXSCAN`, and compare `totalDocsExamined` to `nReturned` — a large gap means a missing/poor index.
- **"What does the aggregation pipeline do and what are the key stages?"** A staged transform of documents: `$match` (filter), `$group` (aggregate), `$project` (reshape), `$sort`, `$lookup` (join). Put `$match` first for index use and to shrink the working set.
- **"Why use `$inc` instead of reading, adding, and writing back?"** It's a single atomic server-side operation immune to the lost-update race condition that plagues read-modify-write.
- **"What's the gotcha with `$lookup`?"** It always returns an array (zero-to-many semantics); flatten with `$unwind`. Also, there's no DB-enforced referential integrity — your app must prevent/handle orphans.
- **"Does MongoDB support transactions?"** Yes — multi-document ACID transactions since 4.0 (on replica sets/sharded clusters), but single-document operations are already atomic, so transactions are used more sparingly than in SQL.

## Practice Exercises

Use the `users`, `products`, `orders`, and `customers` collections seeded throughout this module (re-seed if needed). Try each before reading the solution.

1. **(CRUD)** Insert a new product named `"Webcam"`, price `59.99`, category `"electronics"`, with tags `["video", "usb"]` and stock `15`. Then update it to mark it `inStock: true` and raise its price by `$10` using an atomic operator.
2. **(Query operators)** Find all `users` who are either in `"London"` **or** have a `score` of at least `90`, returning only their `name` and `score` (no `_id`), sorted by score descending.
3. **(Array operators)** Given the `students` collection (with `grades` arrays of `{subject, score}`), find every student who has a `"math"` grade strictly above `80`, ensuring the subject and score match the **same** grade element.
4. **(Indexing)** Write the command to create a unique index on `users.email`, then write the command to discover whether `db.users.find({ city: "London" }).sort({ score: -1 })` uses an index — and state which index would make it efficient.
5. **(Aggregation)** Using `orders`, compute the **total revenue (`qty * price`)** and **number of orders** per `customer`, but only for `status: "shipped"` orders, returning the top 2 customers by revenue.
6. **(Aggregation + join)** Using `orders` and `customers`, compute total shipped revenue per customer **tier** (`gold`/`silver`), highest first, and also list the distinct customers contributing to each tier.

### Solutions

**1. CRUD**

```js
// Insert
db.products.insertOne({
  name: "Webcam",
  price: 59.99,
  category: "electronics",
  tags: ["video", "usb"],
  stock: 15
});

// Atomic update: set inStock true, and raise price by 10 with $inc (not read-modify-write).
db.products.updateOne(
  { name: "Webcam" },
  { $set: { inStock: true }, $inc: { price: 10 } }
);
// price is now 69.99
```

**2. Query operators + projection + sort**

```js
db.users.find(
  { $or: [{ city: "London" }, { score: { $gte: 90 } }] }, // filter
  { name: 1, score: 1, _id: 0 }                            // projection
).sort({ score: -1 });                                     // sort
```

**3. Array `$elemMatch`**

```js
// $elemMatch forces subject AND score to be satisfied by the SAME array element.
db.students.find({
  grades: { $elemMatch: { subject: "math", score: { $gt: 80 } } }
});
// Without $elemMatch you'd wrongly match students who have a math grade in one
// element and a >80 grade in a DIFFERENT element.
```

**4. Indexing + explain**

```js
// Unique index on email.
db.users.createIndex({ email: 1 }, { unique: true });

// Check the plan for the city+sort query.
db.users.find({ city: "London" }).sort({ score: -1 }).explain("executionStats");
// Look at winningPlan.stage: COLLSCAN means no usable index.

// The efficient index: equality on city, then sort on score (ESR rule).
db.users.createIndex({ city: 1, score: -1 });
// Now the query can satisfy both the filter and the sort from the index alone.
```

**5. Aggregation — group + sort + limit**

```js
db.orders.aggregate([
  { $match: { status: "shipped" } },
  {
    $group: {
      _id: "$customer",
      revenue: { $sum: { $multiply: ["$qty", "$price"] } },
      orderCount: { $sum: 1 }
    }
  },
  { $sort: { revenue: -1 } },
  { $limit: 2 }
]);
```

**6. Aggregation with `$lookup`**

```js
db.orders.aggregate([
  { $match: { status: "shipped" } },                         // filter early
  {
    $lookup: {                                               // join to customers
      from: "customers",
      localField: "customer",
      foreignField: "_id",
      as: "c"
    }
  },
  { $unwind: "$c" },                                         // flatten the join array
  {
    $group: {
      _id: "$c.tier",                                        // group by tier
      revenue: { $sum: { $multiply: ["$qty", "$price"] } },
      customers: { $addToSet: "$customer" }                  // distinct contributors
    }
  },
  { $sort: { revenue: -1 } }
]);
```

## Summary / Cheat Sheet

| Topic | Key takeaway / syntax |
|---|---|
| SQL vs NoSQL | Tables+JOINs+rigid schema vs documents+embedding+flexible schema |
| Hierarchy | Database → Collection → Document → Field |
| BSON | Binary JSON; adds Date, ObjectId, Decimal128, ints, Binary |
| `_id` / ObjectId | Auto primary key; 12 bytes = timestamp + random + counter |
| Insert | `insertOne(doc)`, `insertMany([...], { ordered: false })` |
| Read | `find(filter, projection)` → cursor; `findOne(filter)` |
| Update | `updateOne/Many(filter, { $set: {...} }, { upsert: true })` |
| Replace | `replaceOne(filter, fullDoc)` — wipes unlisted fields |
| Delete | `deleteOne/Many(filter)` — empty `{}` deletes ALL |
| Comparison | `$eq $ne $gt $gte $lt $lte $in $nin` |
| Logical | `$and $or $not $nor` (multiple fields = implicit AND) |
| Element | `$exists`, `$type` |
| Evaluation | `$regex` (anchor with `^` to use an index) |
| Array query | `$all`, `$elemMatch` (same element!), `$size` |
| Update ops | `$set $unset $inc $mul $rename`; arrays: `$push $addToSet $pull $pop` |
| Projection | `{ field: 1 }` include / `{ field: 0 }` exclude (don't mix) |
| Cursor shaping | `.sort({f:-1}).skip(n).limit(n)` — order applied sort→skip→limit |
| Pagination | Prefer keyset (`{_id:{$gt:last}}`) over deep `skip()` |
| Index | `createIndex({f:1})`; unique/compound/text; ESR ordering |
| Diagnose | `explain("executionStats")`: IXSCAN good, COLLSCAN bad; docsExamined≈nReturned |
| Aggregation | `aggregate([{$match},{$group},{$project},{$sort},{$lookup}])` |
| Field ref in agg | `"$field"` (with `$`) = value; bare string = literal |
| `$lookup` | Left outer join; returns array → `$unwind` it |
| Modeling | Embed one-to-few/read-together; reference one-to-many/unbounded/shared |
| Limits | 16 MB max document size; no cascade deletes / FK enforcement |
