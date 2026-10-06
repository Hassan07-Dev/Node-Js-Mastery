# JavaScript Essentials for Backend Development

Before you write a single line of Express, you need a rock-solid mental model of the JavaScript language itself. Backend code lives and dies by subtle language behavior: a misplaced `this`, an accidental shared reference, an unhandled rejected promise, a `==` comparison that silently coerces a user-supplied string. On the frontend a bug shows up as a broken button. On the backend the *same* bug corrupts a database, leaks one user's data to another, or takes down a server that thousands of clients depend on.

This module re-grounds you in the parts of JavaScript that matter most for server-side work. You already know JS syntax — `for` loops, `if` statements, basic objects — so we will not waste time there. Instead we go deep on the mechanics that bite backend developers: scoping and hoisting, value vs reference semantics, `this` binding, closures, prototypes, classes, modern syntax, the core array methods you'll use in every controller, JSON handling, and error modeling. Every concept is explained from first principles, with the *why* (not just the *how*), and with runnable Node.js examples.

> TIP: Run every code block in this module. Save a snippet to `scratch.js` and run `node scratch.js`. Reading about hoisting is fine; watching `ReferenceError: Cannot access 'x' before initialization` print to your terminal is how it actually sticks.

---

## What You'll Learn

- How `var`, `let`, and `const` differ in scope, hoisting, and reassignment — and why `var` is effectively banned in modern code.
- Block scope, the hoisting model, and the **Temporal Dead Zone** (TDZ).
- The difference between **primitive** and **reference** types, how assignment and function arguments copy them, and why this causes "spooky action at a distance" bugs.
- Type coercion, the truth about `==` vs `===`, and the full truthy/falsy table.
- Every way to define a function (declaration, expression, arrow) and modern parameter features (default, rest).
- The five `this` binding rules and how to reason about `this` in any situation — including why arrow functions are special.
- **Closures** and lexical scope, with real backend use cases (configuration, rate limiting, memoization, middleware factories).
- Objects, the **prototype chain**, prototypal inheritance, and the essential `Object.*` static methods.
- **ES6 classes**: constructors, methods, inheritance with `extends`/`super`, `static` members, getters/setters, and `#private` fields.
- **Destructuring**, **spread**, and **rest** — the idioms you'll see in every Express handler.
- Template literals, **optional chaining** (`?.`), and **nullish coalescing** (`??`).
- The workhorse array methods: `map`, `filter`, `reduce`, `forEach`, `find`, `some`, `every`, `sort`.
- `JSON.parse` / `JSON.stringify`, including the gotchas that matter for APIs.
- The `Error` object, `try`/`catch`/`finally`, `throw`, and how to design **custom error classes** for clean API error handling.

---

## 1. `var`, `let`, `const`: Scope, Hoisting, and the Temporal Dead Zone

### Why this matters on the backend

Variable scoping bugs are among the most common sources of subtle data corruption in Node. A classic example: building an array of asynchronous callbacks inside a `for` loop. With `var`, every callback closes over the *same* shared variable and you get wrong results — a bug that has shipped to production in countless codebases. Understanding scoping precisely is the difference between "it works on my machine" and "it works."

### The three declarations at a glance

| Feature | `var` | `let` | `const` |
|---|---|---|---|
| Scope | Function-scoped | Block-scoped | Block-scoped |
| Hoisted | Yes, initialized to `undefined` | Yes, but in the **TDZ** | Yes, but in the **TDZ** |
| Re-declaration in same scope | Allowed | `SyntaxError` | `SyntaxError` |
| Re-assignment | Allowed | Allowed | **Not allowed** |
| Creates property on global object | Yes (in non-module scripts) | No | No |
| Use in modern code | Avoid | When reassignment is needed | **Default choice** |

### Block scope

A *block* is any pair of curly braces: `if`, `for`, `while`, or a bare `{ }`. `let` and `const` are visible only inside the block they're declared in. `var` ignores blocks entirely — it only respects function boundaries.

```js
// block-scope.js
function demo() {
  if (true) {
    var functionScoped = "I leak out of the block";
    let blockScoped = "I stay inside the block";
  }
  console.log(functionScoped); // "I leak out of the block"
  console.log(blockScoped);    // ReferenceError: blockScoped is not defined
}
demo();
```

`var` "leaking" out of blocks is exactly the kind of thing that makes code hard to reason about — variables exist in places you didn't intend.

### Hoisting

*Hoisting* is the JS engine's behavior of processing declarations before executing code, top to bottom. Conceptually, declarations are "moved to the top" of their scope. But the three keywords are hoisted *differently*:

- `var` declarations are hoisted **and initialized to `undefined`**. You can reference the variable before its line; you just get `undefined`.
- `let` and `const` declarations are hoisted **but not initialized**. The variable exists in scope but is in the **Temporal Dead Zone** until execution reaches the declaration line. Touching it before then throws.

```js
// hoisting.js
console.log(a); // undefined  (var is hoisted and initialized to undefined)
var a = 1;

console.log(b); // ReferenceError: Cannot access 'b' before initialization
let b = 2;      // b was hoisted but lives in the TDZ until this line runs
```

> WARNING — Pitfall: The fact that `var` gives `undefined` rather than throwing makes `var` bugs *silent*. The TDZ exists precisely to turn those silent bugs into loud, immediate errors. This is a feature, not an annoyance.

### Function hoisting vs variable hoisting

Function **declarations** are fully hoisted — both the name and the body — so you can call them before they appear in the source. Function **expressions** (a function assigned to a variable) follow the variable's hoisting rules.

```js
// function-hoisting.js
sayHi();       // "hi"  — full function declaration is hoisted
function sayHi() { console.log("hi"); }

sayBye();      // TypeError: sayBye is not a function
var sayBye = function () { console.log("bye"); };
// `sayBye` the var is hoisted as undefined; calling undefined() throws.
```

### The Temporal Dead Zone (TDZ), precisely

The TDZ is the region of code from the **start of the block** to the **point where the `let`/`const` variable is initialized**. Within that region the binding exists but accessing it (read or write) throws a `ReferenceError`.

```js
// tdz.js
{
  // TDZ for `count` starts here ...
  // console.log(count); // would throw: Cannot access 'count' before initialization
  const count = 10; // ... TDZ ends here
  console.log(count); // 10
}
```

### The loop-closure classic

This is the single most famous scoping bug, and a guaranteed interview question.

```js
// loop-closure.js
// With var: all callbacks share ONE variable.
const fnsVar = [];
for (var i = 0; i < 3; i++) {
  fnsVar.push(() => i);
}
console.log(fnsVar.map((fn) => fn())); // [3, 3, 3]  ❌  i is 3 after the loop

// With let: each iteration gets a FRESH binding.
const fnsLet = [];
for (let j = 0; j < 3; j++) {
  fnsLet.push(() => j);
}
console.log(fnsLet.map((fn) => fn())); // [0, 1, 2]  ✅
```

> BEST PRACTICE: Default to `const`. Reach for `let` only when you genuinely need to reassign. Treat `var` as a legacy keyword you read in old code but never write.

---

## 2. Primitive vs Reference Types, Coercion, `==` vs `===`, Truthy/Falsy

### Primitives vs references — the core mental model

JavaScript has two categories of values:

- **Primitives**: `string`, `number`, `boolean`, `null`, `undefined`, `symbol`, `bigint`. They are **immutable** and copied **by value**.
- **References (objects)**: everything else — plain objects, arrays, functions, dates, regexes, `Map`, `Set`. Variables hold a **reference** (a pointer) to the value; they are copied **by reference**.

When you assign or pass a primitive, you copy the value. When you assign or pass an object, you copy the *reference* — both variables now point at the *same* underlying object.

```js
// value-vs-reference.js
let a = 10;
let b = a;   // copy the value
b++;
console.log(a, b); // 10 11  — independent

const obj1 = { count: 0 };
const obj2 = obj1; // copy the reference (same object!)
obj2.count = 99;
console.log(obj1.count); // 99  — both see the change
```

> WARNING — Pitfall: This is the #1 source of "spooky action at a distance" backend bugs. If you cache a config object and a request handler mutates it, every subsequent request sees the mutation. If you pass a user's data object into a function that "tweaks" it, the caller's object changes too. **Functions can mutate object arguments.**

### Function arguments are copies of references

```js
// arg-mutation.js
function addTax(order) {
  order.total = order.total * 1.1; // mutates the caller's object!
}
const cart = { total: 100 };
addTax(cart);
console.log(cart.total); // 110  — the original changed

// Safer: return a new object instead of mutating the input.
function addTaxPure(order) {
  return { ...order, total: order.total * 1.1 };
}
```

> BEST PRACTICE: Prefer pure functions that return new objects over functions that mutate their arguments. Immutability makes backend code dramatically easier to reason about, especially with concurrent requests.

### Equality: `==` vs `===`

`===` (strict equality) compares **type and value** with no coercion. `==` (loose equality) performs **type coercion** before comparing, following a notoriously surprising set of rules.

```js
// equality.js
console.log(1 === 1);       // true
console.log(1 === "1");     // false (different types, no coercion)
console.log(1 == "1");      // true  (string coerced to number)
console.log(0 == false);    // true  (false coerced to 0)
console.log("" == false);   // true  ("" -> 0, false -> 0)
console.log(null == undefined); // true  (special-cased)
console.log(null === undefined); // false
console.log(NaN === NaN);   // false  (NaN is never equal to anything)
```

The one place `==` is arguably acceptable is `x == null`, which is `true` for *both* `null` and `undefined` and nothing else — a concise nullish check. Otherwise, use `===`.

> BEST PRACTICE: Always use `===` and `!==`. Reserve `== null` for the specific "is this null or undefined?" check, and even then `x ?? defaultValue` or `x == null` should be a conscious, commented choice.

### Type coercion in arithmetic and concatenation

The `+` operator is overloaded: with a string operand it concatenates; otherwise it adds numerically. Every other arithmetic operator coerces to number.

```js
// coercion.js
console.log(1 + "2");   // "12"  (number coerced to string)
console.log("5" - 2);   // 3     (string coerced to number)
console.log("5" * "2"); // 10
console.log(true + 1);  // 2     (true -> 1)
console.log([] + []);   // ""    (arrays -> "")
console.log([] + {});   // "[object Object]"
console.log(+"42");     // 42    (unary + coerces to number)
console.log(Number("42abc")); // NaN  (strict numeric parse)
console.log(parseInt("42abc", 10)); // 42  (lenient parse)
```

> WARNING — Pitfall: Data arriving over HTTP (query strings, form bodies, URL params) is **always a string**. `req.query.page` is `"2"`, not `2`. Coerce explicitly with `Number(...)` or `parseInt(..., 10)` and validate the result, or you'll do string math like `"2" + 1 === "21"`.

### Truthy and falsy

In boolean contexts (`if`, `&&`, `||`, `!`, ternary), values are coerced to boolean. Memorize the **falsy** list — everything else is truthy.

The falsy values are exactly: `false`, `0`, `-0`, `0n` (BigInt zero), `""` (empty string), `null`, `undefined`, and `NaN`.

```js
// truthy.js
if ("") console.log("won't run");      // "" is falsy
if ("0") console.log("runs");          // "0" is a non-empty string -> truthy!
if ([]) console.log("runs");           // [] is an object -> truthy
if ({}) console.log("runs");           // {} is an object -> truthy
if (0) console.log("won't run");       // 0 is falsy
```

> WARNING — Pitfall: `if (req.body.count)` is wrong if `0` is a valid value — `0` is falsy, so legitimate zeros get rejected. Use `if (req.body.count != null)` or `if ('count' in req.body)` to test presence rather than truthiness.

---

## 3. Functions: Declarations, Expressions, Arrows, Default & Rest Params

### Three ways to create a function

```js
// function-forms.js
// 1) Function declaration — hoisted, has its own `this`, has `arguments`.
function add(a, b) {
  return a + b;
}

// 2) Function expression — assigned to a variable; follows variable hoisting.
const subtract = function (a, b) {
  return a - b;
};

// 3) Arrow function — concise; lexical `this`; no `arguments`, no `prototype`.
const multiply = (a, b) => a * b;
```

### Arrow functions vs regular functions

| Aspect | Regular function | Arrow function |
|---|---|---|
| Own `this` | Yes (set by call site) | No — inherits `this` from enclosing scope |
| `arguments` object | Yes | No (use rest params instead) |
| Usable as constructor (`new`) | Yes | No (throws `TypeError`) |
| Has `prototype` property | Yes | No |
| Can be a generator | Yes (`function*`) | No |
| Hoisted | Declarations yes; expressions no | No (it's an expression) |
| Implicit return | No | Yes, when body is a single expression |

```js
// arrow-implicit-return.js
const square = (x) => x * x;          // implicit return of the expression
const makeUser = (name) => ({ name }); // wrap object literal in parens!
const noop = () => {};                 // empty body, returns undefined
```

> WARNING — Pitfall: `const f = () => { id: 1 }` does **not** return `{ id: 1 }`. The `{` is parsed as a function body, `id:` becomes a label, and the function returns `undefined`. To return an object literal, wrap it: `() => ({ id: 1 })`.

### Default parameters

Default values kick in when an argument is `undefined` (note: **not** when it's `null`).

```js
// default-params.js
function connect(host = "localhost", port = 5432, opts = {}) {
  return `${host}:${port}`;
}
console.log(connect());                 // "localhost:5432"
console.log(connect("db.internal"));    // "db.internal:5432"
console.log(connect(undefined, 3306));  // "localhost:3306" (skip with undefined)

// Defaults are evaluated at call time and can reference earlier params.
function range(start, end, step = (end - start) / 10) {
  return { start, end, step };
}
console.log(range(0, 100)); // { start: 0, end: 100, step: 10 }
```

### Rest parameters

The rest parameter `...args` collects "the remaining arguments" into a **real array** (unlike the old `arguments` object, which is array-*like*). It must be the last parameter.

```js
// rest-params.js
function sum(...numbers) {
  // numbers is a true array, so array methods work directly.
  return numbers.reduce((acc, n) => acc + n, 0);
}
console.log(sum(1, 2, 3, 4)); // 10

function logRequest(method, path, ...middlewares) {
  console.log(method, path, `(${middlewares.length} middlewares)`);
}
logRequest("GET", "/users", () => {}, () => {}); // GET /users (2 middlewares)
```

> BEST PRACTICE: Prefer rest parameters over the legacy `arguments` object. Rest params are real arrays, are explicit in the signature, and work in arrow functions.

---

## 4. The `this` Keyword and All Binding Rules

`this` is the most misunderstood feature in JavaScript, and it's a frequent interview topic. The key insight: in regular functions, **`this` is determined by *how the function is called*, not where it's defined**. There are five rules; check them in priority order.

### Rule 1 — `new` binding

When a function is called with `new`, a brand-new object is created and `this` points to it.

```js
// this-new.js
function User(name) {
  this.name = name; // `this` is the new object
}
const u = new User("Ada");
console.log(u.name); // "Ada"
```

### Rule 2 — Explicit binding: `call`, `apply`, `bind`

You can force `this` with `call` / `apply` (invoke immediately) or `bind` (return a new permanently-bound function).

```js
// this-explicit.js
function greet(greeting) {
  return `${greeting}, ${this.name}`;
}
const person = { name: "Grace" };

console.log(greet.call(person, "Hello"));   // call: args listed individually
console.log(greet.apply(person, ["Hi"]));   // apply: args as an array
const boundGreet = greet.bind(person);      // bind: returns a new function
console.log(boundGreet("Hey"));             // "Hey, Grace"
```

### Rule 3 — Implicit binding: method call

When a function is called as a property of an object (`obj.method()`), `this` is the object **left of the dot**.

```js
// this-implicit.js
const counter = {
  count: 0,
  increment() {
    this.count++; // `this` is `counter`
    return this.count;
  },
};
console.log(counter.increment()); // 1
console.log(counter.increment()); // 2
```

### Rule 4 — Default binding

A plain function call with no object context. In **strict mode** (and inside ES modules, which are always strict), `this` is `undefined`. In sloppy mode it falls back to the global object.

```js
// this-default.js
"use strict";
function whoAmI() {
  return this; // undefined in strict mode
}
console.log(whoAmI()); // undefined
```

> WARNING — Pitfall: The "lost `this`" bug. Extracting a method detaches it from its object, so implicit binding is gone and default binding applies.
>
> ```js
> const inc = counter.increment; // detached!
> // inc(); // TypeError: Cannot read properties of undefined (reading 'count')
> ```
> This bites constantly when passing methods as callbacks: `app.get("/x", obj.handler)` loses `this`. Fix with `obj.handler.bind(obj)` or an arrow wrapper `(...a) => obj.handler(...a)`.

### Rule 5 — Arrow functions: lexical `this`

Arrow functions have **no `this` of their own**. They capture `this` from the enclosing lexical scope at definition time. This makes them perfect for callbacks inside methods.

```js
// this-arrow.js
const timer = {
  seconds: 0,
  start() {
    // Arrow keeps `this` === timer because it's lexically inside start().
    setInterval(() => {
      this.seconds++;
      // A regular function here would have its own `this` (undefined/global).
    }, 1000);
  },
};
```

### Priority order

When multiple rules could apply, resolve in this order: **`new`** > **explicit (`bind`/`call`/`apply`)** > **implicit (method)** > **default**. Arrow functions ignore all of these and use lexical scope.

> WARNING — Pitfall: Never use an arrow function as an object **method** if it needs `this` to refer to the object — the arrow will capture the *outer* (module/global) scope, not the object.
> ```js
> const bad = { val: 1, get() { return this.val; } };       // works
> const broken = { val: 1, get: () => this.val };           // `this` is NOT broken
> ```

---

## 5. Closures and Lexical Scope

### What a closure is

A **closure** is a function bundled together with references to its surrounding lexical environment. In plain terms: an inner function "remembers" the variables of the outer function even after the outer function has returned. This is possible because JavaScript uses **lexical scoping** — a function's scope is determined by where it's *written*, not where it's *called*.

```js
// closure-basics.js
function makeCounter() {
  let count = 0; // private to this closure
  return function () {
    count++;     // remembers `count` across calls
    return count;
  };
}
const next = makeCounter();
console.log(next()); // 1
console.log(next()); // 2
// `count` is unreachable from outside — true privacy via closure.
```

### Why closures matter on the backend

Closures are not academic trivia — they power core backend patterns. Here are four you'll use constantly.

**1) Private state / encapsulation** (data that can't be tampered with from outside):

```js
// closure-private.js
function createBankAccount(initialBalance) {
  let balance = initialBalance; // private
  return {
    deposit(amount) { balance += amount; return balance; },
    getBalance() { return balance; },
    // No way to set balance to an arbitrary value from outside.
  };
}
```

**2) Configuration via factory functions** (dependency injection without a framework):

```js
// closure-factory.js
// A logger factory that "bakes in" a prefix.
function createLogger(prefix) {
  return (message) => console.log(`[${prefix}] ${message}`);
}
const dbLog = createLogger("DB");
dbLog("connection established"); // [DB] connection established
```

**3) Express middleware factories** — the single most common closure pattern in Node:

```js
// closure-middleware.js
// Returns a middleware configured with a required role.
function requireRole(role) {
  // The returned function closes over `role`.
  return (req, res, next) => {
    if (req.user?.role !== role) {
      return res.status(403).json({ error: "Forbidden" });
    }
    next();
  };
}
// Usage: app.delete("/users/:id", requireRole("admin"), handler);
```

**4) Memoization** (caching expensive results in a private cache):

```js
// closure-memoize.js
function memoize(fn) {
  const cache = new Map(); // private cache, lives in the closure
  return (...args) => {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}
const slowSquare = (n) => { /* imagine expensive work */ return n * n; };
const fastSquare = memoize(slowSquare);
```

> WARNING — Pitfall: Closures keep their captured variables alive in memory. If a closure captures a large object (or a request/response) and is stored long-term (e.g., in a module-level cache or an event listener that's never removed), you create a **memory leak**. Be deliberate about what long-lived closures capture.

> TIP: A simple rate limiter is a closure over a `Map` of timestamps per IP. Closures + `Map` + `setTimeout` is the recipe for a surprising amount of in-memory backend state.

---

## 6. Objects, Prototypes, and Prototypal Inheritance

### Objects and property access

```js
// objects.js
const user = {
  name: "Linus",
  "favorite color": "green", // quoted key needs bracket access
  greet() { return `Hi, ${this.name}`; }, // method shorthand
};
console.log(user.name);               // dot access
console.log(user["favorite color"]);  // bracket access (dynamic / unusual keys)

const key = "name";
console.log(user[key]); // "Linus" — computed access with a variable
```

### The prototype chain

Every JavaScript object has an internal link to another object called its **prototype**. When you access a property, the engine looks on the object itself; if not found, it walks up the **prototype chain** until it finds the property or reaches `null`. This is how inheritance works in JS — there are no classes under the hood, just objects linked to other objects.

```js
// prototype-chain.js
const animal = {
  eats: true,
  walk() { return `${this.name} walks`; },
};
const dog = Object.create(animal); // dog's prototype is animal
dog.name = "Rex";
dog.barks = true;

console.log(dog.barks);     // true  (own property)
console.log(dog.eats);      // true  (inherited from animal)
console.log(dog.walk());    // "Rex walks" (method found up the chain)
console.log(Object.getPrototypeOf(dog) === animal); // true
```

### Constructor functions and `prototype`

Before `class` syntax, "classes" were built with constructor functions plus a shared `prototype` object. Methods go on the prototype so all instances share one copy rather than each instance carrying its own.

```js
// constructor-function.js
function Vehicle(wheels) {
  this.wheels = wheels; // per-instance data
}
// Shared method on the prototype — one function, all instances use it.
Vehicle.prototype.describe = function () {
  return `Vehicle with ${this.wheels} wheels`;
};
const car = new Vehicle(4);
console.log(car.describe()); // "Vehicle with 4 wheels"
```

### Essential `Object.*` static methods

These show up everywhere in backend code — serializing, validating, cloning, iterating over request payloads.

```js
// object-methods.js
const config = { host: "localhost", port: 5432, ssl: true };

console.log(Object.keys(config));    // ["host", "port", "ssl"]
console.log(Object.values(config));  // ["localhost", 5432, true]
console.log(Object.entries(config)); // [["host","localhost"], ["port",5432], ["ssl",true]]

// Iterate key/value pairs.
for (const [key, value] of Object.entries(config)) {
  console.log(`${key} = ${value}`);
}

// Merge / shallow-copy (later sources win).
const withDefaults = Object.assign({}, { port: 80 }, config); // { port:5432, host, ssl }

// Build an object from pairs (inverse of Object.entries).
const fromPairs = Object.fromEntries([["a", 1], ["b", 2]]); // { a: 1, b: 2 }

// Freeze to prevent mutation (shallow!).
const frozen = Object.freeze({ env: "production" });
frozen.env = "dev"; // silently ignored (throws in strict mode)
console.log(frozen.env); // "production"
```

| Method | Returns | Use case |
|---|---|---|
| `Object.keys(o)` | Array of own enumerable keys | Iterating / counting fields |
| `Object.values(o)` | Array of own enumerable values | Summing / scanning values |
| `Object.entries(o)` | Array of `[key, value]` pairs | `for...of` iteration, transforms |
| `Object.fromEntries(p)` | Object from pairs | Building objects from `Map`/filters |
| `Object.assign(t, ...s)` | Mutated target | Shallow merge / clone |
| `Object.freeze(o)` | Frozen object | Immutable config |
| `Object.create(proto)` | New object with given prototype | Manual prototypal inheritance |
| `Object.getPrototypeOf(o)` | The prototype | Inspecting the chain |

> WARNING — Pitfall: `Object.freeze` and the spread operator `{...obj}` are **shallow**. Nested objects are still mutable / shared. For deep immutability or deep cloning, use `structuredClone(obj)` (built into modern Node) or a library.

---

## 7. ES6 Classes

`class` is **syntactic sugar over the prototype system** — it does not introduce a new object model. But it's cleaner, and it's what you'll use for models, services, and custom errors. Mongoose models, custom error hierarchies, and service layers all lean on classes.

### Constructor, methods, and instances

```js
// class-basics.js
class User {
  constructor(name, email) {
    this.name = name;   // instance fields set in the constructor
    this.email = email;
  }
  // Methods live on User.prototype — shared across all instances.
  describe() {
    return `${this.name} <${this.email}>`;
  }
}
const u = new User("Ada", "ada@example.com");
console.log(u.describe()); // "Ada <ada@example.com>"
console.log(typeof User);  // "function" — a class IS a function under the hood
```

### Inheritance with `extends` and `super`

`super(...)` calls the parent constructor; `super.method()` calls the parent's method. You **must** call `super()` before using `this` in a subclass constructor.

```js
// class-inheritance.js
class Animal {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} makes a sound`; }
}
class Dog extends Animal {
  constructor(name, breed) {
    super(name);       // call parent constructor FIRST
    this.breed = breed;
  }
  speak() {
    // Extend, don't just replace, parent behavior.
    return `${super.speak()} — specifically, a bark`;
  }
}
const d = new Dog("Rex", "Lab");
console.log(d.speak()); // "Rex makes a sound — specifically, a bark"
console.log(d instanceof Animal); // true
```

### Static members

`static` methods and fields belong to the **class itself**, not instances. They're ideal for factories and utilities.

```js
// class-static.js
class Temperature {
  constructor(celsius) { this.celsius = celsius; }
  static fromFahrenheit(f) {           // factory — called on the class
    return new Temperature((f - 32) * 5 / 9);
  }
  static get ABSOLUTE_ZERO() { return -273.15; } // static getter constant
}
const t = Temperature.fromFahrenheit(98.6);
console.log(t.celsius.toFixed(1));        // "37.0"
console.log(Temperature.ABSOLUTE_ZERO);   // -273.15
```

### Getters and setters

Computed properties that look like fields but run code on access/assignment — great for derived values and validation.

```js
// class-accessors.js
class Account {
  #balance = 0; // private field (see below)
  get balance() {                 // read like a property: account.balance
    return this.#balance;
  }
  set balance(value) {            // write like a property: account.balance = 50
    if (value < 0) throw new RangeError("Balance cannot be negative");
    this.#balance = value;
  }
}
const a = new Account();
a.balance = 100;          // invokes the setter
console.log(a.balance);   // 100 — invokes the getter
// a.balance = -5;        // throws RangeError
```

### Private fields (`#`)

Fields prefixed with `#` are **truly private** — inaccessible outside the class body, even via bracket notation. This is real encapsulation, enforced by the language (unlike the old `_underscore` convention, which was just a hint).

```js
// class-private.js
class PasswordHasher {
  #salt; // private; not visible as a property from outside
  constructor(salt) { this.#salt = salt; }
  hash(password) {
    return `${password}::${this.#salt}`; // toy example
  }
}
const h = new PasswordHasher("s3cr3t");
console.log(h.hash("pw"));   // "pw::s3cr3t"
// console.log(h.#salt);     // SyntaxError — truly inaccessible
// console.log(h["#salt"]);  // undefined — not a real string-keyed property
```

| Member type | Declared with | Belongs to | Accessed as |
|---|---|---|---|
| Instance field | `this.x =` or `x = ...` | Each instance | `obj.x` |
| Instance method | `method() {}` | Prototype (shared) | `obj.method()` |
| Static member | `static x` / `static m()` | The class | `Class.x` / `Class.m()` |
| Getter/Setter | `get`/`set` | Prototype | `obj.prop` (no parens) |
| Private field | `#x` | Each instance (private) | `this.#x` inside class only |

> BEST PRACTICE: Use `#private` fields for genuinely internal state (hashes, connection handles, caches). Reserve getters/setters for derived/validated properties — don't wrap every field in a trivial getter/setter just for the sake of it.

---

## 8. Destructuring, Spread, and Rest

These three features are everywhere in modern Express code. Mastering them makes handlers concise and readable.

### Object destructuring

Pull properties into variables by name. Supports defaults, renaming, and nesting.

```js
// destructure-object.js
const req = {
  body: { username: "ada", password: "pw" },
  query: { page: "2" },
};

// Basic
const { username, password } = req.body;

// Rename + default
const { page: pageStr, limit = "10" } = req.query;

// Nested
const { body: { username: u } } = req;

// Destructuring in a function parameter — extremely common in Express.
function createUser({ name, email, role = "user" }) {
  return { name, email, role };
}
console.log(createUser({ name: "Bob", email: "b@x.com" }));
// { name: "Bob", email: "b@x.com", role: "user" }
```

### Array destructuring

Positional. Supports skipping, defaults, and swapping.

```js
// destructure-array.js
const [first, second] = [10, 20, 30];     // first=10, second=20
const [, , third] = [10, 20, 30];         // skip with commas; third=30
const [head, ...tail] = [1, 2, 3, 4];     // head=1, tail=[2,3,4] (rest)
let a = 1, b = 2;
[a, b] = [b, a];                          // swap without a temp variable
```

### Spread operator

Expands an iterable/object into individual elements/properties. Used for copying, merging, and passing arrays as arguments.

```js
// spread.js
const base = { host: "localhost", port: 80 };
const override = { port: 8080, ssl: true };
const merged = { ...base, ...override }; // { host, port:8080, ssl:true } — later wins

const arr1 = [1, 2];
const arr2 = [3, 4];
const combined = [...arr1, ...arr2]; // [1, 2, 3, 4]
const copy = [...arr1];              // shallow copy

console.log(Math.max(...[5, 1, 9, 3])); // spread array into args -> 9
```

### Rest vs spread — same syntax, opposite jobs

The `...` token means **rest** (collect many into one) on the *left* of `=` or in a parameter list, and **spread** (expand one into many) on the *right* of `=` or in a call/literal.

```js
// rest-vs-spread.js
const [x, ...rest] = [1, 2, 3]; // REST: collect -> rest = [2, 3]
const expanded = [0, ...rest];  // SPREAD: expand -> [0, 2, 3]
```

> WARNING — Pitfall: Spread is a **shallow** copy. `const copy = { ...original }` copies top-level properties, but nested objects are shared by reference. Mutating `copy.nested.x` mutates `original.nested.x`. Use `structuredClone()` for a deep copy.

---

## 9. Template Literals, Optional Chaining, and Nullish Coalescing

### Template literals

Backtick strings with `${}` interpolation and multi-line support — far better than `+` concatenation for building log messages, SQL-ish strings, and HTML.

```js
// template-literals.js
const name = "Ada";
const count = 3;
const msg = `User ${name} has ${count} item${count === 1 ? "" : "s"}`;
console.log(msg); // "User Ada has 3 items"

const multiline = `Line 1
Line 2`; // preserves the newline

// Expressions are allowed inside ${}
console.log(`Total: $${(19.99 * count).toFixed(2)}`); // "Total: $59.97"
```

### Optional chaining (`?.`)

Safely access deeply nested properties (or call methods) that might not exist. If the value before `?.` is `null` or `undefined`, the whole expression short-circuits to `undefined` instead of throwing.

```js
// optional-chaining.js
const res = { data: { user: { profile: null } } };

console.log(res.data?.user?.profile?.avatar); // undefined (no crash)
// Without ?., res.data.user.profile.avatar throws: Cannot read 'avatar' of null

// Optional call — only calls if the method exists.
const obj = {};
obj.maybeFn?.(); // no error; would crash with obj.maybeFn()

// Optional dynamic access.
const key = "x";
console.log(res?.data?.[key]); // undefined
```

> WARNING — Pitfall: Don't over-apply `?.`. If a value is *supposed* to exist and its absence is a real bug, you want the crash, not a silent `undefined` that propagates into your database. Use `?.` for genuinely optional data, not as a blanket "make errors go away" tool.

### Nullish coalescing (`??`)

Returns the right-hand side only when the left is `null` or `undefined` — **not** for other falsy values like `0` or `""`. This fixes the classic bug where `||` wrongly overrides legitimate `0` and `""`.

```js
// nullish-coalescing.js
const port = process.env.PORT ?? 3000; // use 3000 only if PORT is null/undefined

// The crucial difference vs ||:
const count = 0;
console.log(count || 10); // 10  ❌  — 0 is falsy, gets replaced (likely a bug)
console.log(count ?? 10); // 0   ✅  — 0 is a valid value, preserved

const name = "";
console.log(name || "Anonymous"); // "Anonymous"  — replaces empty string
console.log(name ?? "Anonymous"); // ""            — keeps empty string
```

> BEST PRACTICE: Use `??` for default values whenever `0`, `""`, or `false` are legitimate inputs (which on the backend is *most* of the time — think quantities, prices, flags). Reserve `||` for cases where any falsy value should genuinely trigger the fallback.

`??` pairs beautifully with optional chaining: `const city = user?.address?.city ?? "Unknown";`.

---

## 10. Essential Array Methods

Backend code is constantly transforming collections: rows from a database, items in a cart, results from an API. These methods are your daily tools. The key distinction: **`map`/`filter`/`reduce` return new values (functional, non-mutating)**, while `forEach` returns nothing and `sort` mutates in place.

### `map` — transform each element

```js
// arr-map.js
const users = [{ id: 1, name: "Ada" }, { id: 2, name: "Bob" }];
const names = users.map((u) => u.name);          // ["Ada", "Bob"]
const ids = users.map((u) => u.id);              // [1, 2]
// Common: shape DB documents into API responses.
const dtos = users.map(({ id, name }) => ({ id, displayName: name }));
```

### `filter` — keep elements that pass a test

```js
// arr-filter.js
const nums = [1, 2, 3, 4, 5, 6];
const evens = nums.filter((n) => n % 2 === 0);   // [2, 4, 6]
const active = users.filter((u) => u.id > 1);    // [{ id: 2, name: "Bob" }]
```

### `reduce` — collapse to a single value

The most powerful and most misunderstood. It threads an accumulator through the array. **Always pass an initial value** as the second argument.

```js
// arr-reduce.js
const cart = [{ price: 10, qty: 2 }, { price: 5, qty: 4 }];
const total = cart.reduce((sum, item) => sum + item.price * item.qty, 0); // 40

// reduce can build objects too — e.g., group/index by a key.
const list = [{ id: "a" }, { id: "b" }];
const byId = list.reduce((acc, item) => {
  acc[item.id] = item;
  return acc;
}, {}); // { a: { id: "a" }, b: { id: "b" } }
```

### `forEach` — side effects, no return

Use for side effects (logging, pushing to an external sink). It returns `undefined` and cannot be chained.

```js
// arr-foreach.js
["a", "b"].forEach((item, index) => console.log(index, item));
// 0 a
// 1 b
```

> WARNING — Pitfall: `forEach` ignores `async`/`await` — it does **not** wait for async callbacks. To process items asynchronously, use a `for...of` loop (sequential) or `Promise.all(arr.map(async ...))` (concurrent). Never `await` inside `forEach` expecting it to pause.

### `find` and `findIndex` — first match

```js
// arr-find.js
const found = users.find((u) => u.id === 2); // { id: 2, name: "Bob" } (or undefined)
const idx = users.findIndex((u) => u.id === 2); // 1 (or -1)
```

### `some` and `every` — boolean tests

```js
// arr-some-every.js
const hasAdmin = users.some((u) => u.role === "admin");  // true if ANY match
const allActive = users.every((u) => u.active === true);  // true if ALL match
```

### `sort` — order elements (mutates!)

`sort` mutates the original array and, by default, sorts as **strings**. For numbers you must pass a comparator.

```js
// arr-sort.js
console.log([10, 1, 2].sort());           // [1, 10, 2]  ❌  string sort!
console.log([10, 1, 2].sort((a, b) => a - b)); // [1, 2, 10] ✅ ascending
console.log([10, 1, 2].sort((a, b) => b - a)); // [10, 2, 1]    descending

// To avoid mutating the original, copy first (or use toSorted in modern Node):
const original = [3, 1, 2];
const sorted = [...original].sort((a, b) => a - b);
// const sorted = original.toSorted((a, b) => a - b); // Node 20+, non-mutating
```

| Method | Returns | Mutates? | Typical use |
|---|---|---|---|
| `map` | New array (same length) | No | Transform / reshape |
| `filter` | New array (subset) | No | Select matching items |
| `reduce` | Single accumulated value | No | Sum, group, build object |
| `forEach` | `undefined` | No | Side effects only |
| `find` | First match or `undefined` | No | Look up one item |
| `findIndex` | Index or `-1` | No | Locate position |
| `some` | Boolean | No | "Any match?" |
| `every` | Boolean | No | "All match?" |
| `sort` | The same array | **Yes** | Order (pass comparator!) |

> BEST PRACTICE: Chain non-mutating methods for readable data pipelines: `data.filter(isValid).map(toDTO).sort(byName)`. Reach for `reduce` only when `map`/`filter` can't express the transform — it's powerful but harder to read.

---

## 11. JSON: `parse` and `stringify`

JSON is the lingua franca of web APIs. Express receives JSON request bodies and sends JSON responses, so you must understand the conversion both ways — and its limits.

```js
// json-basics.js
const obj = { name: "Ada", age: 36, active: true, tags: ["a", "b"] };

const json = JSON.stringify(obj);
console.log(json); // '{"name":"Ada","age":36,"active":true,"tags":["a","b"]}'

const parsed = JSON.parse(json);
console.log(parsed.name); // "Ada" — back to a real object
```

### Pretty-printing and the replacer

`JSON.stringify(value, replacer, space)` takes an optional **replacer** (filter keys or transform values) and **space** (indentation).

```js
// json-stringify-args.js
const data = { id: 1, password: "secret", name: "Ada" };

// Pretty print with 2-space indentation.
console.log(JSON.stringify(data, null, 2));

// Whitelist keys with an array replacer (e.g., strip sensitive fields).
console.log(JSON.stringify(data, ["id", "name"]));
// '{"id":1,"name":"Ada"}'  — password excluded
```

### `toJSON` hook

If an object has a `toJSON()` method, `JSON.stringify` uses its return value. This is how `Date` serializes to an ISO string, and how Mongoose lets you customize document serialization.

```js
// json-tojson.js
console.log(JSON.stringify({ when: new Date(0) }));
// '{"when":"1970-01-01T00:00:00.000Z"}'  — Date.toJSON() ran

const account = {
  balance: 100,
  password: "secret",
  toJSON() {
    const { password, ...safe } = this; // omit password on serialization
    return safe;
  },
};
console.log(JSON.stringify(account)); // '{"balance":100}'
```

### What JSON cannot represent

| JS value | `JSON.stringify` result |
|---|---|
| `undefined` (in object) | Key is **omitted** entirely |
| `undefined` (in array) | Becomes `null` |
| Function | Omitted (object) / `null` (array) |
| `NaN`, `Infinity` | `null` |
| `Date` | ISO string (via `toJSON`) |
| `BigInt` | **Throws** `TypeError` |
| `Map`, `Set` | `{}` (no custom serialization) |
| Circular reference | **Throws** `TypeError` |

```js
// json-limits.js
console.log(JSON.stringify({ a: undefined, b: () => {}, c: NaN }));
// '{"c":null}'  — a and b dropped, NaN -> null

const circular = {};
circular.self = circular;
// JSON.stringify(circular); // TypeError: Converting circular structure to JSON
```

> WARNING — Pitfall: `JSON.parse` **throws** on invalid JSON. Always wrap parsing of untrusted input (a request body, a file, a queue message) in `try/catch`. In Express, `express.json()` middleware does this for you and surfaces a 400-style error, but anywhere you call `JSON.parse` manually, you own the error handling.

> TIP: The `parse(stringify(obj))` deep-clone trick works for plain JSON-safe data but silently drops functions, `undefined`, and `Date` objects (Dates become strings). For a correct deep clone, use `structuredClone(obj)`.

---

## 12. Errors: `Error`, `try`/`catch`/`finally`, `throw`, and Custom Error Classes

Robust error handling is what separates a toy script from a production server. A backend that crashes on the first malformed request, or that returns a 200 with a half-broken body, is worse than useless. You need to **detect**, **classify**, and **respond** to errors deliberately.

### The `Error` object

`Error` instances carry a `name`, a `message`, and a `stack` (the call trace). Built-in subclasses include `TypeError`, `RangeError`, `SyntaxError`, and `ReferenceError`.

```js
// error-object.js
const err = new Error("Something failed");
console.log(err.name);    // "Error"
console.log(err.message); // "Something failed"
console.log(err.stack);   // "Error: Something failed\n    at ..." (full trace)
```

### `throw`, `try`, `catch`, `finally`

`throw` raises an error and unwinds the stack until a `catch` handles it. `finally` runs **regardless** of whether an error was thrown — ideal for cleanup (closing connections, releasing locks).

```js
// try-catch-finally.js
function parseConfig(raw) {
  try {
    const config = JSON.parse(raw); // may throw SyntaxError
    if (!config.port) throw new Error("Missing 'port'");
    return config;
  } catch (err) {
    console.error("Failed to parse config:", err.message);
    return null; // graceful fallback
  } finally {
    console.log("parseConfig finished"); // always runs
  }
}
parseConfig('{"port": 3000}'); // logs "parseConfig finished", returns config
parseConfig("not json");        // logs the error, then "finished", returns null
```

> WARNING — Pitfall: Always `throw` an `Error` instance, never a string or plain object. `throw "boom"` gives you no stack trace and breaks the common `catch (err) { err.message }` / `instanceof` patterns. Likewise, an empty `catch {}` that swallows errors silently is one of the worst things you can do — at minimum, log it.

### Custom error classes

For real APIs you want **typed errors** so a central handler can map them to the right HTTP status. Extend `Error`, set `this.name`, and (in V8/Node) call `Error.captureStackTrace` to keep your constructor out of the trace.

```js
// custom-errors.js
class AppError extends Error {
  constructor(message, statusCode = 500) {
    super(message);
    this.name = this.constructor.name; // e.g. "AppError"
    this.statusCode = statusCode;
    this.isOperational = true; // expected, handled error (vs a programmer bug)
    Error.captureStackTrace(this, this.constructor); // clean stack trace
  }
}

class NotFoundError extends AppError {
  constructor(resource = "Resource") {
    super(`${resource} not found`, 404);
  }
}

class ValidationError extends AppError {
  constructor(message, fields) {
    super(message, 400);
    this.fields = fields; // attach extra context
  }
}

// Usage
function getUser(id, db) {
  const user = db.get(id);
  if (!user) throw new NotFoundError("User");
  return user;
}

try {
  getUser(99, { get: () => null });
} catch (err) {
  console.log(err instanceof AppError);   // true
  console.log(err instanceof NotFoundError); // true
  console.log(err.statusCode, err.message);  // 404 "User not found"
}
```

This `instanceof`-driven design is the foundation of a clean Express error handler:

```js
// error-handler-preview.js  (Express pattern you'll build later)
function errorHandler(err, req, res, next) {
  const status = err.statusCode ?? 500;
  const message = err.isOperational ? err.message : "Internal Server Error";
  res.status(status).json({ error: message });
}
```

> BEST PRACTICE: Distinguish **operational errors** (expected: bad input, not found, unauthorized — handle and respond) from **programmer errors** (bugs: `undefined is not a function` — log, alert, and consider restarting the process). Never leak internal error details or stack traces to API clients in production.

> WARNING — Pitfall: In async code, a thrown error inside a Promise becomes a **rejection**, not a synchronous throw. A `try/catch` only catches it if you `await` the promise. Errors thrown in async callbacks that aren't awaited become *unhandled rejections* — a top cause of mysterious Node crashes. (Async error handling gets a full treatment in the async module; just know the foundation here.)

---

## Common Pitfalls & Gotchas

- **`var` in loops with closures** captures one shared binding → use `let` for a fresh binding per iteration.
- **Reference aliasing**: assigning or passing an object copies the *reference*; mutating it affects every holder. Spread/clone before mutating shared data.
- **Shallow copies**: `{...obj}`, `Object.assign`, and `[...arr]` only copy the top level. Use `structuredClone` for deep copies.
- **`||` for defaults** wrongly replaces `0`, `""`, and `false`. Use `??` when those are valid values.
- **Truthiness checks on `0`/`""`**: `if (value)` rejects legitimate `0` and `""`. Test presence (`value != null`, `'key' in obj`) instead.
- **`==` coercion surprises**: `0 == ""`, `[] == false`, `null == undefined` are all `true`. Use `===`.
- **`this` loss**: extracting a method (`const f = obj.method`) detaches `this`. Bind it or wrap in an arrow.
- **Arrow as a method**: `{ get: () => this.x }` captures the *outer* `this`, not the object. Use a regular method.
- **`forEach` + `async`**: does not await. Use `for...of` or `Promise.all(map(...))`.
- **`sort` without a comparator** sorts numbers as strings (`[1, 10, 2]`), and it **mutates** the array.
- **`JSON.stringify` drops** `undefined`, functions, and throws on `BigInt`/circular refs; `JSON.parse` throws on bad input — wrap it.
- **Throwing non-Errors**: `throw "msg"` loses the stack trace. Always throw `Error` instances.
- **HTTP input is strings**: coerce and validate `req.query`/`req.params` before doing math or comparisons.

---

## Best Practices

- **Default to `const`**, use `let` only when reassigning, never write `var`.
- **Prefer immutability**: return new objects/arrays instead of mutating inputs, especially across request boundaries.
- **Use `===`** everywhere; allow `== null` only as a deliberate, commented nullish check.
- **Use `??` for defaults** and `?.` for genuinely optional access — but don't use `?.` to paper over real bugs.
- **Favor pure functions**: same input → same output, no side effects. They're trivial to test and reason about.
- **Use modern syntax for clarity**: destructuring in parameters, template literals over concatenation, array methods over manual loops where they read better.
- **Model errors as classes** with status codes and an operational flag; centralize error-to-response mapping.
- **Encapsulate** with `#private` fields and closures; expose only what callers need.
- **Validate and coerce all external input** at the boundary before it touches business logic or the database.
- **Run code, read stack traces.** The fastest way to internalize these behaviors is to reproduce them in a `node` REPL.

---

## Key Interview Points

- **What's the difference between `var`, `let`, and `const`?** `var` is function-scoped and hoisted to `undefined`; `let`/`const` are block-scoped and hoisted into the TDZ (accessing before declaration throws). `const` can't be reassigned (but the object it points to can still be mutated).
- **What is the Temporal Dead Zone?** The span from the start of a block to a `let`/`const`'s initialization where the binding exists but accessing it throws a `ReferenceError`. It turns "use before declare" into a loud error.
- **Primitive vs reference types?** Primitives are immutable and copied by value; objects are copied by reference, so two variables can point at one object and one can mutate it for the other. Function arguments are copies of references — functions can mutate object args.
- **`==` vs `===`?** `==` coerces types before comparing (many surprising results); `===` compares type and value with no coercion. Always use `===`.
- **List the falsy values.** `false`, `0`, `-0`, `0n`, `""`, `null`, `undefined`, `NaN`. Everything else (including `"0"`, `[]`, `{}`) is truthy.
- **Explain `this` and its binding rules.** `this` is set by how a function is *called*: `new` (new object) > explicit (`call`/`apply`/`bind`) > implicit (method on object) > default (`undefined` in strict mode). Arrow functions have no own `this` — they capture it lexically.
- **What is a closure and why is it useful?** A function plus its captured lexical environment; the inner function remembers outer variables after the outer returns. Used for private state, factories, middleware, memoization. Risk: memory leaks from long-lived captures.
- **What is the prototype chain?** Objects link to a prototype; property lookup walks up the chain until found or `null`. It's the mechanism behind JS inheritance; `class` is sugar over it.
- **`class` vs constructor functions?** `class` is syntactic sugar over prototypes, adding cleaner syntax, `extends`/`super`, `static`, and true `#private` fields. No new object model underneath.
- **`??` vs `||`?** `||` returns the right side for any falsy left value; `??` only for `null`/`undefined`. Use `??` to preserve valid `0`/`""`/`false`.
- **`map` vs `forEach`?** `map` returns a new transformed array (chainable, non-mutating); `forEach` returns `undefined` and is for side effects. Neither awaits async callbacks.
- **How does `reduce` work?** It threads an accumulator through the array via a reducer `(acc, item) => newAcc`, returning a single value; always supply an initial value.
- **Does `sort` mutate? How do you sort numbers?** Yes, it mutates and defaults to string comparison; pass a comparator `(a, b) => a - b` for numbers, and copy first (`[...arr].sort` or `toSorted`) to avoid mutation.
- **What can't JSON represent?** `undefined`, functions (dropped), `NaN`/`Infinity` (→ `null`), `BigInt` and circular refs (throw), `Map`/`Set` (→ `{}`). `Date` becomes an ISO string via `toJSON`.
- **How do you design errors for an API?** Custom classes extending `Error` with `statusCode` and an `isOperational` flag; a central handler maps them to responses. Always throw `Error` instances; never leak stack traces to clients.

---

## Practice Exercises

Try each before reading the solution. Run your code with `node`.

1. **Hoisting & TDZ.** Without running it, predict the output of the following, then explain *why* each line behaves as it does:
   ```js
   console.log(x);
   var x = 5;
   console.log(y);
   let y = 10;
   ```

2. **Reference semantics.** Write a function `cloneAndUpdate(obj, key, value)` that returns a **new** object with `key` set to `value`, leaving the original object completely unchanged. Prove with an assertion that the original is untouched, including for a nested property.

3. **`this` binding.** Given the object below, explain why `getCount` fails when detached, and fix it **two** different ways (one with `bind`, one without changing the call site by using an arrow-based design):
   ```js
   const counter = {
     count: 0,
     getCount() { return this.count; },
   };
   const fn = counter.getCount;
   // fn(); // breaks — fix it
   ```

4. **Closures.** Implement `createRateLimiter(maxCalls, windowMs)` that returns a function `attempt()` returning `true` if the call is allowed within the rolling window and `false` if the limit is exceeded. Keep all state private.

5. **Array pipeline + reduce.** Given an array of orders `{ id, customer, amount, status }`, write a single expression (or short function) that returns total revenue from `"paid"` orders, **grouped by customer**, as an object like `{ alice: 150, bob: 90 }`.

6. **Custom errors + safe JSON.** Write `safeParseUser(raw)` that `JSON.parse`s a string, validates that it has a non-empty `email` and a numeric `age >= 0`, and throws a `ValidationError` (custom class extending `Error` with a `400` status and a `fields` array) on any failure — while turning malformed JSON into a `ValidationError` too. Return the parsed user on success.

### Solutions

**Solution 1 — Hoisting & TDZ.**

```js
console.log(x); // undefined  — `var x` is hoisted and initialized to undefined
var x = 5;
console.log(y); // ReferenceError: Cannot access 'y' before initialization
let y = 10;     // `y` is hoisted but sits in the TDZ until this line executes
```

`var x` is hoisted with an initial value of `undefined`, so the first log succeeds with `undefined`. `let y` is hoisted too, but it remains uninitialized in the Temporal Dead Zone until control reaches its declaration; reading it before that throws.

**Solution 2 — Reference semantics.**

```js
// solution-2.js
function cloneAndUpdate(obj, key, value) {
  // structuredClone gives a deep copy so nested objects aren't shared.
  const copy = structuredClone(obj);
  copy[key] = value;
  return copy;
}

const original = { name: "Ada", meta: { active: true } };
const updated = cloneAndUpdate(original, "name", "Grace");

console.assert(original.name === "Ada", "original.name must be unchanged");
console.assert(updated.name === "Grace", "updated.name must be set");

// Prove nested independence:
updated.meta.active = false;
console.assert(original.meta.active === true, "nested original must be unchanged");
console.log("All assertions passed");
```

A shallow `{ ...obj, [key]: value }` would pass the top-level check but **fail** the nested test, because `meta` would still be shared. `structuredClone` makes the copy fully independent.

**Solution 3 — `this` binding.**

```js
// solution-3.js
const counter = {
  count: 5,
  getCount() { return this.count; },
};

// Fix A: bind the method to its object.
const boundFn = counter.getCount.bind(counter);
console.log(boundFn()); // 5

// Fix B: define the public method as an arrow that closes over the object,
// so `this` is irrelevant — it reads `count` from the closure.
function makeCounter(start = 0) {
  let count = start;
  return {
    increment: () => ++count,        // arrow: lexical, no `this` dependence
    getCount: () => count,           // safe to detach
  };
}
const c = makeCounter(5);
const detached = c.getCount; // detaching is fine here
console.log(detached()); // 5
```

The original fails because `const fn = counter.getCount` strips the `obj.method()` call context; calling `fn()` uses default binding, where `this` is `undefined` (strict mode), so `this.count` throws. `bind` permanently fixes `this`; the closure-based design sidesteps `this` entirely.

**Solution 4 — Closures (rate limiter).**

```js
// solution-4.js
function createRateLimiter(maxCalls, windowMs) {
  let timestamps = []; // private state captured by the closure
  return function attempt() {
    const now = Date.now();
    // Drop timestamps outside the rolling window.
    timestamps = timestamps.filter((t) => now - t < windowMs);
    if (timestamps.length >= maxCalls) return false; // limit hit
    timestamps.push(now);
    return true; // allowed
  };
}

const limiter = createRateLimiter(2, 1000); // 2 calls per second
console.log(limiter()); // true
console.log(limiter()); // true
console.log(limiter()); // false  — exceeded within the window
```

`timestamps` is unreachable from outside; only `attempt` can see and mutate it. Each call prunes expired entries, giving a true rolling window.

**Solution 5 — Array pipeline + reduce.**

```js
// solution-5.js
const orders = [
  { id: 1, customer: "alice", amount: 100, status: "paid" },
  { id: 2, customer: "bob", amount: 90, status: "paid" },
  { id: 3, customer: "alice", amount: 50, status: "paid" },
  { id: 4, customer: "bob", amount: 30, status: "pending" }, // excluded
];

const revenueByCustomer = orders
  .filter((o) => o.status === "paid")           // keep only paid
  .reduce((acc, o) => {
    acc[o.customer] = (acc[o.customer] ?? 0) + o.amount; // ?? handles first time
    return acc;
  }, {});

console.log(revenueByCustomer); // { alice: 150, bob: 90 }
```

`filter` removes non-paid orders, then `reduce` accumulates per-customer totals into an object. `acc[o.customer] ?? 0` initializes a customer's running total on first encounter (using `??` so a legitimate `0` would be preserved).

**Solution 6 — Custom errors + safe JSON.**

```js
// solution-6.js
class ValidationError extends Error {
  constructor(message, fields = []) {
    super(message);
    this.name = "ValidationError";
    this.statusCode = 400;
    this.fields = fields; // which fields failed
    Error.captureStackTrace?.(this, this.constructor);
  }
}

function safeParseUser(raw) {
  let user;
  try {
    user = JSON.parse(raw);
  } catch {
    // Malformed JSON -> a 400 validation error, not a crash.
    throw new ValidationError("Invalid JSON payload", ["_body"]);
  }

  const failed = [];
  if (typeof user.email !== "string" || user.email.trim() === "") {
    failed.push("email");
  }
  if (typeof user.age !== "number" || Number.isNaN(user.age) || user.age < 0) {
    failed.push("age");
  }
  if (failed.length > 0) {
    throw new ValidationError("User validation failed", failed);
  }

  return user;
}

// Demonstrations:
console.log(safeParseUser('{"email":"a@b.com","age":30}')); // { email, age }

try {
  safeParseUser("not json");
} catch (e) {
  console.log(e instanceof ValidationError, e.statusCode, e.fields);
  // true 400 [ '_body' ]
}

try {
  safeParseUser('{"email":"","age":-1}');
} catch (e) {
  console.log(e.statusCode, e.fields); // 400 [ 'email', 'age' ]
}
```

This is the exact shape of validation you'll wire into Express: parse defensively, collect *all* failing fields (rather than failing on the first), and throw a typed error a central handler can turn into a clean `400` JSON response.

---

## Summary / Cheat Sheet

| Concept | Key takeaway |
|---|---|
| **Declarations** | `const` by default, `let` to reassign, never `var`. |
| **Scope** | `let`/`const` block-scoped; `var` function-scoped and leaky. |
| **Hoisting / TDZ** | `var` → `undefined`; `let`/`const` → TDZ throws until initialized. |
| **Primitives** | `string, number, boolean, null, undefined, symbol, bigint` — copied by value. |
| **References** | Objects/arrays/functions — copied by reference; mutations are shared. |
| **Equality** | Use `===`; `==` coerces. `== null` matches `null` + `undefined`. |
| **Falsy** | `false, 0, -0, 0n, "", null, undefined, NaN`. All else truthy. |
| **Functions** | Declaration (hoisted), expression, arrow (lexical `this`, no `arguments`). |
| **Params** | Defaults trigger on `undefined`; rest `...args` is a real array. |
| **`this`** | `new` > explicit (`bind`/`call`/`apply`) > implicit (method) > default. Arrows: lexical. |
| **Closures** | Inner fn remembers outer scope → private state, factories, middleware, memoization. |
| **Prototypes** | Property lookup walks the chain; `class` is sugar over it. |
| **Classes** | `constructor`, methods, `extends`/`super`, `static`, get/set, `#private`. |
| **Destructuring** | Pull fields by name/position; supports defaults, rename, nesting. |
| **Spread/Rest** | `...` expands (right side) or collects (left side); spread is shallow. |
| **`?.` / `??`** | Safe nested access; default only on `null`/`undefined` (preserves `0`/`""`). |
| **Array methods** | `map`/`filter`/`reduce` return new; `forEach` side effects; `sort` mutates + needs comparator. |
| **JSON** | `stringify` drops `undefined`/functions, throws on `BigInt`/circular; `parse` throws on bad input. |
| **Errors** | Throw `Error` instances; custom classes with `statusCode`/`isOperational`; `finally` for cleanup. |

You now have the language foundation that everything else in this course builds on. When you write Express handlers, Mongoose models, middleware, and async data flows, you'll see every one of these concepts in action — `this` binding in class-based services, closures in middleware factories, destructuring in request handling, custom errors in your error pipeline, and reference semantics dictating how data moves through your app. Keep this module open as a reference; you'll come back to it.
