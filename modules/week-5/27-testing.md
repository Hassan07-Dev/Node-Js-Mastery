# Testing Node & Express Applications

You can now build a complete Express API backed by MongoDB. The question every team will ask before they let your code touch production is: **how do you know it works — and how will you know it still works after the next change?** The answer is automated tests. Testing is not a chore bolted on at the end; it is the discipline that lets you refactor fearlessly, ship on Fridays, and onboard new teammates who can change code they don't fully understand because the test suite catches their mistakes.

This module turns you into a confident, fluent tester of Node and Express applications. We start from first principles — *why* we test and *what kinds* of tests exist — then go deep on **Jest** (the dominant JS test runner), the art of **mocking**, testing **asynchronous** code, **HTTP-level integration tests** with **supertest**, **database integration tests** with **mongodb-memory-server**, how to **structure** suites and kill **flakiness**, what **code coverage** actually tells you, a practical **TDD** loop, and finally the **built-in `node:test` runner** as a zero-dependency alternative. Everything here is runnable on Node.js current LTS with plain JavaScript.

## What You'll Learn

By the end of this module you will be able to:

- Explain **why automated testing exists**, articulate the **testing pyramid**, and precisely distinguish **unit**, **integration**, and **end-to-end (e2e)** tests.
- Set up **Jest** from scratch and write tests with `describe` / `it` / `test`, `expect`, and the most important **matchers**.
- Use **setup/teardown hooks** (`beforeAll`, `beforeEach`, `afterEach`, `afterAll`) correctly, and understand their execution order.
- **Mock** dependencies with `jest.fn()`, `jest.mock()`, and `jest.spyOn()`, including mocking whole modules and **fake timers**.
- Test **asynchronous code** robustly using `async/await`, `.resolves`, `.rejects`, and avoid the classic async-test traps.
- Write **HTTP-level integration tests** for Express routes with **supertest** — without binding to a real port.
- Run **integration tests against a real-ish MongoDB** using **mongodb-memory-server**, with proper isolation between tests.
- **Structure** a test suite, build **fixtures and factories**, and systematically **eliminate flaky tests**.
- Read and reason about **code coverage** and explain what "good coverage" really means.
- Follow a disciplined **Test-Driven Development (TDD)** red–green–refactor loop.
- Use the **built-in `node:test` runner** and `node:assert` as a dependency-free alternative to Jest.

---

## 1. Why Test, and the Testing Pyramid

### 1.1 Why automated tests exist

Before frameworks, "testing" meant a developer manually clicking through the app or hitting endpoints with `curl`. That works once. It does not scale to a codebase touched by ten people over three years, where any change can silently break something five files away. **Automated tests** encode your expectations as executable code so a machine can re-verify all of them in seconds, on every change.

The concrete payoffs:

- **Regression safety.** A test you wrote in January catches a bug you'd otherwise introduce in July. This is the single biggest reason tests pay for themselves.
- **Refactoring confidence.** You can restructure internals aggressively as long as the tests still pass — the tests pin the *behavior*, freeing you to change the *implementation*.
- **Executable specification.** A well-named test reads like a requirement: `it('returns 404 when the user does not exist')`. New teammates learn the system by reading tests.
- **Faster feedback.** Catching a bug in a 50ms unit test is orders of magnitude cheaper than catching it in QA, and astronomically cheaper than catching it in production.
- **Design pressure.** Code that is hard to test is usually badly coupled. The friction of writing a test is an early warning that your design needs work.

> BEST PRACTICE: Treat a failing test as a *gift*. It found a problem before your users did. The goal of testing is not to prove your code is perfect; it is to *find defects cheaply* and lock in correct behavior.

### 1.2 The testing pyramid

Not all tests are equal. They trade off **speed/cost** against **realism/confidence**. The **testing pyramid** (popularized by Mike Cohn) is a heuristic for how many of each kind you should have.

```
            /\
           /  \      e2e  (few)        slow, brittle, very realistic
          /----\
         /      \    integration       medium speed, medium realism
        /--------\
       /          \  unit (many)        fast, cheap, isolated
      /------------\
```

- **Many unit tests** at the base: fast, isolated, cheap to write and maintain.
- **Fewer integration tests** in the middle: they exercise several units together (e.g., a route + its controller + a real DB).
- **A handful of e2e tests** at the top: they drive the whole system the way a user would.

The pyramid shape exists because of cost. Each layer up is slower to run, harder to debug when it fails, and more prone to flakiness. If your suite is "inverted" (lots of slow e2e tests, few unit tests), it will be slow, flaky, and painful to maintain — an **ice-cream cone** anti-pattern.

> TIP: The pyramid is a guideline, not a law. Some teams favor a "testing trophy" (heavier on integration tests) because integration tests catch the bugs that actually happen in web apps — wiring mistakes between layers. The principle that survives every variation: **most of your tests should be fast, and you should have few slow ones.**

### 1.3 Unit vs integration vs e2e

| Aspect | Unit | Integration | End-to-End (e2e) |
| --- | --- | --- | --- |
| **Scope** | One function/module in isolation | Several modules working together | Whole system, outside-in |
| **Dependencies** | Mocked/stubbed | Some real (e.g., real DB) | All real (HTTP server, DB, maybe browser) |
| **Speed** | Milliseconds | Tens–hundreds of ms | Seconds |
| **Stability** | Very stable | Stable-ish | Most prone to flakiness |
| **Failure clarity** | Pinpoints the exact unit | Points to a layer | "Something broke somewhere" |
| **Example** | `calculateTotal(cart)` returns correct sum | `POST /orders` writes a row to the DB and returns 201 | Sign up, log in, place an order through real HTTP |

A **unit** is the smallest testable piece — usually a single pure function or a single class method — with its collaborators replaced by test doubles. **Integration** tests verify that units *cooperate* correctly: that your controller actually calls the model, that the model actually queries the DB, that the route is wired to the right handler. **E2e** tests verify the system as a black box from the outside (real HTTP requests, real database) and answer "does the whole thing work for a user?"

For an Express + Mongoose API, the sweet spot is: **unit-test your pure business logic and service functions**, **integration-test your routes against an in-memory MongoDB via supertest**, and keep e2e tests minimal (a few critical user journeys, often run only in CI).

> WARNING — Pitfall: People argue endlessly about whether a "route + in-memory DB" test is "integration" or "e2e." Don't waste energy on labels. What matters is: *what does it cover, how fast is it, and how likely is it to flake?* Optimize for those.

---

## 2. Jest Setup and Basics

**Jest** is a batteries-included test framework: a test runner, an assertion library (`expect`), a mocking system, and a coverage tool — all in one package. It is the de facto standard for Node and JS testing, which is why we lead with it.

### 2.1 Installing and configuring Jest

Initialize a project and add Jest as a **dev dependency** (it's only needed during development, never in production):

```bash
# In your project root
npm init -y                 # create package.json if you don't have one
npm install --save-dev jest # install Jest as a devDependency
```

Wire up an npm script so you (and CI) run tests with one command:

```json
// package.json (excerpt)
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage"
  }
}
```

Now `npm test` runs Jest. By default Jest discovers any file matching `*.test.js` / `*.spec.js`, or any file inside a `__tests__/` folder.

You usually want a small config file. Use `jest.config.js`:

```js
// jest.config.js
/** @type {import('jest').Config} */
module.exports = {
  testEnvironment: 'node',   // we're testing a server, not a browser DOM
  verbose: true,             // print each test name as it runs
  clearMocks: true,          // automatically reset mock state between tests
  coverageDirectory: 'coverage',
  // Tell Jest where your tests live (optional; default discovery is fine too)
  testMatch: ['**/__tests__/**/*.test.js', '**/*.test.js'],
};
```

> TIP: `testEnvironment: 'node'` is important for backend code. The default in older Jest setups could be `jsdom`, which simulates a browser and is both slower and wrong for server code. Modern Jest defaults to `node`, but set it explicitly so nobody has to wonder.

#### A note on Jest and ES Modules (ESM)

Jest was built in the CommonJS era. If your project uses `require`/`module.exports` (CommonJS), everything "just works." If your project uses ESM (`import`/`export` with `"type": "module"`), Jest needs extra setup because its mocking magic predates native ESM. The common approaches:

- Run Jest under Babel (`babel-jest`) to transpile ESM to CJS for tests.
- Use Jest's **experimental native ESM** support by running with `node --experimental-vm-modules` and importing the mock API via `import { jest } from '@jest/globals'`.

This module uses **CommonJS** for all examples, which is the path of least resistance with Jest. (Section 12 covers `node:test`, which has first-class ESM support if you live in ESM-land.)

### 2.2 `describe`, `test` / `it`, and your first test

Jest exposes a few **globals** in every test file (no import needed under the default setup):

- `test(name, fn)` — defines a single test. `it` is an exact alias of `test`; use whichever reads better.
- `describe(name, fn)` — groups related tests into a suite. Purely organizational; you can nest them.
- `expect(value)` — starts an assertion; you chain a **matcher** onto it.

Let's test a tiny pure function. First the code under test:

```js
// src/math.js
function add(a, b) {
  return a + b;
}

function divide(a, b) {
  if (b === 0) {
    throw new Error('Cannot divide by zero'); // we'll test this branch too
  }
  return a / b;
}

module.exports = { add, divide };
```

Now the test file, placed next to it as `src/math.test.js`:

```js
// src/math.test.js
const { add, divide } = require('./math');

describe('add', () => {
  it('adds two positive numbers', () => {
    expect(add(2, 3)).toBe(5); // toBe = strict equality (===)
  });

  it('handles negative numbers', () => {
    expect(add(-4, 1)).toBe(-3);
  });
});

describe('divide', () => {
  it('divides evenly', () => {
    expect(divide(10, 2)).toBe(5);
  });

  it('throws when dividing by zero', () => {
    // Pass a FUNCTION to expect, so Jest can call it and catch the throw.
    expect(() => divide(1, 0)).toThrow('Cannot divide by zero');
  });
});
```

Run it:

```bash
npm test
```

You'll see each `it` reported as a passing test grouped under its `describe`. The naming convention "subject → behavior" (`describe('divide') > it('throws when dividing by zero')`) makes failures read like sentences.

### 2.3 The Arrange–Act–Assert pattern

Almost every good test has three phases. Keeping them visually separate makes tests scannable:

```js
it('applies a 10% discount to the cart total', () => {
  // Arrange — set up inputs and the system under test
  const cart = { items: [{ price: 100 }, { price: 50 }] };

  // Act — perform the one action being tested
  const total = applyDiscount(cart, 0.1);

  // Assert — verify the outcome
  expect(total).toBe(135);
});
```

> BEST PRACTICE: One logical assertion-target per test. A test should be able to fail for exactly one reason. If you're asserting five unrelated things, you probably have five tests hiding in a trench coat.

### 2.4 `expect` and common matchers

A **matcher** is the comparison applied to the value inside `expect()`. Here are the ones you'll use constantly.

| Matcher | Use it for | Example |
| --- | --- | --- |
| `toBe(x)` | Strict equality (`===`); primitives & identity | `expect(2 + 2).toBe(4)` |
| `toEqual(x)` | Deep value equality for objects/arrays | `expect(user).toEqual({ id: 1 })` |
| `toStrictEqual(x)` | Like `toEqual` but also checks `undefined` props & types | `expect(a).toStrictEqual(b)` |
| `toBeTruthy()` / `toBeFalsy()` | Boolean-ish checks | `expect(value).toBeTruthy()` |
| `toBeNull()` / `toBeUndefined()` / `toBeDefined()` | Specific nullish checks | `expect(x).toBeNull()` |
| `toBeGreaterThan(n)` / `toBeLessThan(n)` | Numeric comparisons | `expect(age).toBeGreaterThan(17)` |
| `toBeCloseTo(n, digits)` | Floating-point comparisons | `expect(0.1 + 0.2).toBeCloseTo(0.3)` |
| `toContain(item)` | Array/string membership | `expect(['a','b']).toContain('b')` |
| `toHaveLength(n)` | Length of array/string | `expect(list).toHaveLength(3)` |
| `toHaveProperty(path, val?)` | Object has a (nested) key | `expect(res.body).toHaveProperty('data.id')` |
| `toMatch(regexOrStr)` | String matches regex/substring | `expect(msg).toMatch(/error/i)` |
| `toMatchObject(obj)` | Subset match of an object | `expect(user).toMatchObject({ active: true })` |
| `toThrow(err?)` | A function throws | `expect(() => f()).toThrow(TypeError)` |
| `.not` | Negate any matcher | `expect(x).not.toBe(5)` |

The single most common beginner mistake is `toBe` vs `toEqual`:

```js
it('toBe vs toEqual', () => {
  const a = { id: 1 };
  const b = { id: 1 };

  // expect(a).toBe(b);     // ❌ FAILS — different object references
  expect(a).toEqual(b);     // ✅ PASSES — same structure & values

  expect(3).toBe(3);        // ✅ primitives compare by value, so toBe works
});
```

`toBe` uses `Object.is` (essentially `===`). Two different objects with identical contents are *not* the same reference, so `toBe` fails. Use `toEqual` (recursive structural equality) for objects and arrays.

```js
it('toEqual vs toStrictEqual on undefined props', () => {
  expect({ a: undefined, b: 2 }).toEqual({ b: 2 });        // ✅ ignores undefined
  expect({ a: undefined, b: 2 }).not.toStrictEqual({ b: 2 }); // ❌ strict cares
});
```

> WARNING — Pitfall: `expect(() => f()).toThrow()` requires a **function**, not the result of calling `f()`. If you write `expect(f()).toThrow()`, `f()` runs *before* `expect` sees it, the error escapes, and your test errors out instead of asserting. Always wrap the call in an arrow function.

---

## 3. Setup and Teardown Hooks

Tests should be **independent**: each should run alone or in any order and produce the same result. Hooks let you create the preconditions each test needs and clean up afterward so state doesn't leak.

| Hook | Runs | Typical use |
| --- | --- | --- |
| `beforeAll(fn)` | Once, before all tests in the scope | Expensive one-time setup: connect to DB, start a server |
| `beforeEach(fn)` | Before *every* test in the scope | Reset state: clear collections, rebuild fixtures |
| `afterEach(fn)` | After *every* test in the scope | Tidy up per-test side effects, restore mocks |
| `afterAll(fn)` | Once, after all tests in the scope | Tear down: close DB connection, stop server |

"Scope" means the enclosing `describe` (or the file's top level). Hooks declared inside a `describe` apply only to that block; hooks at the top level apply to the whole file.

### 3.1 Execution order

Order is the thing people get wrong. For a single test inside a nested `describe`, hooks fire **outer-to-inner for `before*`** and **inner-to-outer for `after*`** — like nested function calls wrapping each test.

```js
// hooks-order.test.js
beforeAll(() => console.log('1: outer beforeAll'));
afterAll(() => console.log('8: outer afterAll'));
beforeEach(() => console.log('3: outer beforeEach'));
afterEach(() => console.log('6: outer afterEach'));

describe('group', () => {
  beforeAll(() => console.log('2: inner beforeAll'));
  afterAll(() => console.log('7: inner afterAll'));
  beforeEach(() => console.log('4: inner beforeEach'));
  afterEach(() => console.log('5: inner afterEach'));

  it('runs the test body', () => console.log('-- test body --'));
});

/* Output order:
   1: outer beforeAll
   2: inner beforeAll
   3: outer beforeEach
   4: inner beforeEach
   -- test body --
   5: inner afterEach
   6: outer afterEach
   7: inner afterAll
   8: outer afterAll
*/
```

### 3.2 Async hooks

Hooks can be async. Return a promise (or use `async`/`await`) and Jest will wait for it before proceeding.

```js
let db;

beforeAll(async () => {
  db = await connectToDatabase();   // Jest waits for the connection
});

afterAll(async () => {
  await db.close();                 // and waits for cleanup
});

beforeEach(async () => {
  await db.collection('users').deleteMany({}); // fresh slate per test
});
```

> BEST PRACTICE: Prefer `beforeEach` for *resetting state* and `beforeAll` only for *expensive, immutable* setup (like opening one DB connection). If you set shared mutable state in `beforeAll` and tests mutate it, you've coupled your tests and reintroduced order-dependence.

> WARNING — Pitfall: If a `beforeAll` opens a resource (server, DB connection, timer) you **must** close it in `afterAll`, or Jest will warn `A worker process has failed to exit gracefully` and may hang. Run with `--detectOpenHandles` to find the culprit.

---

## 4. Mocking: Doubles, Spies, and Fakes

To unit-test a function in isolation, you replace its real collaborators with **test doubles** — fake versions you control. Jest's mocking system is its superpower. Let's get the vocabulary straight first, because interviewers probe it:

| Term | Meaning |
| --- | --- |
| **Stub** | A double that returns canned values; no behavior verification |
| **Mock** | A double you make assertions on (was it called? with what?) |
| **Spy** | Wraps a *real* function to record calls; can optionally pass through or replace |
| **Fake** | A lightweight working implementation (e.g., an in-memory DB) |

In Jest these blur together because a single `jest.fn()` can act as a stub *and* a mock — it records calls (mock) and can return canned values (stub).

### 4.1 `jest.fn()` — the mock function

`jest.fn()` creates a function that remembers every call made to it and lets you control what it returns.

```js
it('jest.fn records calls and controls return values', () => {
  const mock = jest.fn();

  mock('hello', 42);
  mock('world');

  // Inspect the recorded calls via the .mock property:
  expect(mock).toHaveBeenCalled();               // called at least once
  expect(mock).toHaveBeenCalledTimes(2);         // exactly twice
  expect(mock).toHaveBeenCalledWith('hello', 42); // with these args (some call)
  expect(mock).toHaveBeenLastCalledWith('world'); // the most recent call
  expect(mock.mock.calls[0]).toEqual(['hello', 42]); // raw call log
});

it('jest.fn can return canned values', () => {
  const getUser = jest.fn().mockReturnValue({ id: 1, name: 'Ada' });
  expect(getUser()).toEqual({ id: 1, name: 'Ada' });

  // Different value on successive calls:
  const seq = jest.fn()
    .mockReturnValueOnce('first')
    .mockReturnValueOnce('second')
    .mockReturnValue('rest');
  expect(seq()).toBe('first');
  expect(seq()).toBe('second');
  expect(seq()).toBe('rest');
  expect(seq()).toBe('rest');
});

it('jest.fn can resolve/reject for async code', async () => {
  const fetchData = jest.fn().mockResolvedValue({ ok: true });
  await expect(fetchData()).resolves.toEqual({ ok: true });

  const failing = jest.fn().mockRejectedValue(new Error('boom'));
  await expect(failing()).rejects.toThrow('boom');
});
```

Use `jest.fn()` directly to inject a fake collaborator via **dependency injection**:

```js
// src/notify.js
function notifyUser(user, sender) {
  // sender is injected, so we can pass a mock in tests
  if (!user.email) throw new Error('No email');
  return sender(user.email, 'Welcome!');
}
module.exports = { notifyUser };
```

```js
// src/notify.test.js
const { notifyUser } = require('./notify');

it('sends a welcome email to the user', () => {
  const sender = jest.fn().mockReturnValue('sent');
  const result = notifyUser({ email: 'ada@x.com' }, sender);

  expect(result).toBe('sent');
  expect(sender).toHaveBeenCalledWith('ada@x.com', 'Welcome!');
});
```

### 4.2 `jest.spyOn()` — spying on real methods

When you can't inject a dependency (it's a method on an imported object), `jest.spyOn(object, 'method')` wraps the real method so you can observe — and optionally override — it.

```js
// src/report.js
const logger = require('./logger'); // has a .warn(msg) method

function process(value) {
  if (value < 0) logger.warn('negative value');
  return Math.abs(value);
}
module.exports = { process };
```

```js
// src/report.test.js
const { process } = require('./report');
const logger = require('./logger');

it('warns on negative input', () => {
  // Replace logger.warn with a spy, suppressing the real implementation:
  const warnSpy = jest.spyOn(logger, 'warn').mockImplementation(() => {});

  expect(process(-5)).toBe(5);
  expect(warnSpy).toHaveBeenCalledWith('negative value');

  warnSpy.mockRestore(); // put the real method back
});
```

> TIP: `mockImplementation(() => {})` silences the real method (handy for muting `console.log` noise). Omit it to *spy while still calling through* to the real implementation. Always `mockRestore()` a spy (or set `restoreMocks: true` in config) so it doesn't leak into other tests.

### 4.3 `jest.mock()` — mocking whole modules

The big one. `jest.mock('module-path')` replaces an *entire* module with an auto-mock (every export becomes a `jest.fn()`). This is how you isolate a unit from its imported dependencies — a real database driver, an HTTP client, the filesystem.

```js
// src/services/userService.js
const db = require('../db'); // a module with async findById(id)

async function getUserName(id) {
  const user = await db.findById(id);
  if (!user) throw new Error('User not found');
  return user.name;
}
module.exports = { getUserName };
```

```js
// src/services/userService.test.js
const { getUserName } = require('./userService');
const db = require('../db');

jest.mock('../db'); // auto-mock: db.findById becomes a jest.fn()

describe('getUserName', () => {
  afterEach(() => jest.clearAllMocks()); // reset call history & impls

  it('returns the user name when found', async () => {
    db.findById.mockResolvedValue({ id: 1, name: 'Ada' }); // arrange mock
    await expect(getUserName(1)).resolves.toBe('Ada');
    expect(db.findById).toHaveBeenCalledWith(1);
  });

  it('throws when the user is missing', async () => {
    db.findById.mockResolvedValue(null);
    await expect(getUserName(99)).rejects.toThrow('User not found');
  });
});
```

`jest.mock()` calls are **hoisted** to the top of the file by Jest's transform — they run before the `require` statements, which is why the import gets the mocked version even though the call appears below it.

You can also provide a **factory** for full control over the mock's shape:

```js
// Mock only what you need, with a custom factory:
jest.mock('../db', () => ({
  findById: jest.fn(),
  insert: jest.fn(),
}));
```

And you can do a **partial mock**, keeping real exports while overriding a few:

```js
jest.mock('../utils', () => {
  const actual = jest.requireActual('../utils'); // import the REAL module
  return {
    ...actual,                 // keep everything real
    now: jest.fn(() => 0),     // override just utils.now()
  };
});
```

> WARNING — Pitfall: Because `jest.mock` is hoisted above imports, you **cannot** reference an out-of-scope variable inside its factory unless the variable name begins with `mock` (Jest's special exception). This trips up everyone once:
>
> ```js
> const fakeUser = { id: 1 };
> jest.mock('../db', () => ({ findById: () => fakeUser })); // ❌ ReferenceError
>
> const mockUser = { id: 1 };
> jest.mock('../db', () => ({ findById: () => mockUser })); // ✅ allowed
> ```

### 4.4 Mocking timers

Code that uses `setTimeout`, `setInterval`, or `Date` is hard to test because you don't want to *actually* wait. **Fake timers** let you fast-forward time synchronously.

```js
// src/debounce.js
function delayedGreeting(name, cb) {
  setTimeout(() => cb(`Hi, ${name}`), 1000); // fires after 1s
}
module.exports = { delayedGreeting };
```

```js
// src/debounce.test.js
const { delayedGreeting } = require('./debounce');

describe('delayedGreeting', () => {
  beforeEach(() => jest.useFakeTimers());   // swap in fake timer control
  afterEach(() => jest.useRealTimers());    // restore real timers

  it('calls back after 1 second — without waiting', () => {
    const cb = jest.fn();
    delayedGreeting('Ada', cb);

    expect(cb).not.toHaveBeenCalled();      // nothing yet
    jest.advanceTimersByTime(1000);         // fast-forward 1s instantly
    expect(cb).toHaveBeenCalledWith('Hi, Ada');
  });
});
```

Useful fake-timer controls:

- `jest.advanceTimersByTime(ms)` — move the clock forward by `ms`, firing due timers.
- `jest.runAllTimers()` — run every pending timer to completion (careful with recursive `setInterval`).
- `jest.runOnlyPendingTimers()` — run timers currently queued, but not ones they schedule.

You can also freeze the wall clock:

```js
it('freezes Date with fake timers', () => {
  jest.useFakeTimers().setSystemTime(new Date('2026-01-01T00:00:00Z'));
  expect(new Date().toISOString()).toBe('2026-01-01T00:00:00.000Z');
  jest.useRealTimers();
});
```

### 4.5 Keeping mocks clean between tests

Mock state (call history, implementations) persists unless you reset it, which causes phantom cross-test failures. Three tools, in increasing aggressiveness:

| Method | What it resets |
| --- | --- |
| `jest.clearAllMocks()` | Call history only (`mock.calls`, `mock.results`) |
| `jest.resetAllMocks()` | Call history **and** implementations (back to no-op `jest.fn()`) |
| `jest.restoreAllMocks()` | Restores originals for spies created with `spyOn` |

Set them globally in config so you never forget:

```js
// jest.config.js
module.exports = {
  clearMocks: true,    // clear call history before every test
  // resetMocks: true, // also wipe implementations (use deliberately)
  restoreMocks: true,  // restore spies after every test
};
```

---

## 5. Testing Asynchronous Code

Almost all Node code is async. Testing it correctly hinges on one rule: **the test runner must know when your async work is done**, or it will declare success before your assertions run.

### 5.1 The async patterns

**Preferred: `async`/`await`.** Make the test function `async` and `await` the work. Jest waits for the returned promise.

```js
// src/fetchUser.js
async function fetchUser(id, api) {
  const res = await api.get(`/users/${id}`);
  return res.data;
}
module.exports = { fetchUser };
```

```js
// src/fetchUser.test.js
const { fetchUser } = require('./fetchUser');

it('returns the user data', async () => {
  const api = { get: jest.fn().mockResolvedValue({ data: { id: 1 } }) };
  const user = await fetchUser(1, api);    // await the async work
  expect(user).toEqual({ id: 1 });
});
```

**`.resolves` / `.rejects` matchers.** Cleaner for one-liner assertions on a promise. Always `await` (or `return`) them.

```js
it('resolves to the user (resolves matcher)', async () => {
  const api = { get: jest.fn().mockResolvedValue({ data: { id: 2 } }) };
  await expect(fetchUser(2, api)).resolves.toEqual({ id: 2 });
});

it('rejects when the API errors (rejects matcher)', async () => {
  const api = { get: jest.fn().mockRejectedValue(new Error('500')) };
  await expect(fetchUser(3, api)).rejects.toThrow('500');
});
```

**Returning a promise (no `async`).** Equivalent — Jest waits for any promise you return:

```js
it('also works by returning the promise', () => {
  const api = { get: jest.fn().mockResolvedValue({ data: { id: 4 } }) };
  return expect(fetchUser(4, api)).resolves.toEqual({ id: 4 });
});
```

**Callback-style code: the `done` callback.** For old-school callbacks, accept `done` and call it when finished. Call `done(err)` on failure.

```js
it('handles a callback-style API', (done) => {
  legacyLoad((err, data) => {
    try {
      expect(err).toBeNull();
      expect(data).toBe('ok');
      done();             // signal completion
    } catch (e) {
      done(e);            // forward assertion failures
    }
  });
});
```

### 5.2 The classic async traps

> WARNING — Pitfall #1 — Forgetting to await. If you don't `await` (or `return`) the promise, the test function returns immediately and Jest marks it **passed** before assertions run. A genuinely broken async function then shows as green:
>
> ```js
> it('silently passes — BAD', () => {
>   expect(fetchUser(1, brokenApi)).resolves.toEqual({ id: 1 }); // not awaited!
> }); // test ends; the assertion never gets a chance to fail
> ```
>
> The fix is always `await` (or `return`) the promise/`.resolves`/`.rejects` chain.

> WARNING — Pitfall #2 — Testing rejection without asserting it happened. If you only `await` a rejecting promise inside a try/catch but never assert the catch ran, a function that *doesn't* throw passes silently. Use `expect.assertions(n)` to force a count:
>
> ```js
> it('definitely throws', async () => {
>   expect.assertions(1); // guarantees exactly 1 assertion runs
>   try {
>     await mayThrow();
>   } catch (err) {
>     expect(err.message).toMatch(/bad/);
>   }
> });
> ```
>
> If `mayThrow()` doesn't reject, the catch never runs, no assertion fires, and `expect.assertions(1)` fails the test — exactly what you want. (Or just use `await expect(...).rejects.toThrow()`, which is simpler.)

> TIP: Mixing fake timers with promises requires care. If your code does `await new Promise(r => setTimeout(r, 1000))`, advancing fake timers must be interleaved with the microtask queue. `jest.advanceTimersByTimeAsync(1000)` (async variant) handles this correctly.

---

## 6. Testing Express Routes with supertest

Unit tests prove your functions are correct. But a web app's value is in its **HTTP surface**: the right method + path returns the right status, headers, and body. **supertest** lets you fire HTTP requests at your Express app and assert on the responses — *without* manually starting a server on a real port.

The key insight: **export your `app` separately from the code that calls `app.listen()`.** supertest binds to the app object directly (on an ephemeral port it manages), so you never touch port 3000 in tests.

### 6.1 Structuring the app for testability

```js
// src/app.js  — the Express app, with NO listen() call
const express = require('express');

const app = express();
app.use(express.json()); // parse JSON request bodies

// In-memory store just to make the example self-contained
const users = [];
let nextId = 1;

app.get('/health', (req, res) => res.json({ status: 'ok' }));

app.get('/users/:id', (req, res) => {
  const user = users.find((u) => u.id === Number(req.params.id));
  if (!user) return res.status(404).json({ error: 'User not found' });
  res.json(user);
});

app.post('/users', (req, res) => {
  const { name, email } = req.body;
  if (!name || !email) {
    return res.status(400).json({ error: 'name and email are required' });
  }
  const user = { id: nextId++, name, email };
  users.push(user);
  res.status(201).json(user); // 201 Created
});

module.exports = app; // export the app for tests
```

```js
// src/server.js  — the ONLY place that binds a port (run in production)
const app = require('./app');

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Listening on ${PORT}`));
```

### 6.2 Writing supertest tests

```bash
npm install --save-dev supertest
```

```js
// src/app.test.js
const request = require('supertest');
const app = require('./app'); // import the app, NOT the server

describe('Users API', () => {
  it('GET /health returns ok', async () => {
    const res = await request(app).get('/health'); // fire the request

    expect(res.status).toBe(200);
    expect(res.body).toEqual({ status: 'ok' });
  });

  it('POST /users creates a user and returns 201', async () => {
    const res = await request(app)
      .post('/users')
      .send({ name: 'Ada', email: 'ada@x.com' }) // JSON body
      .set('Accept', 'application/json');

    expect(res.status).toBe(201);
    expect(res.body).toMatchObject({ name: 'Ada', email: 'ada@x.com' });
    expect(res.body).toHaveProperty('id'); // server-assigned id
  });

  it('POST /users rejects missing fields with 400', async () => {
    const res = await request(app).post('/users').send({ name: 'NoEmail' });

    expect(res.status).toBe(400);
    expect(res.body.error).toMatch(/required/);
  });

  it('GET /users/:id returns 404 for unknown id', async () => {
    const res = await request(app).get('/users/99999');
    expect(res.status).toBe(404);
    expect(res.body).toEqual({ error: 'User not found' });
  });
});
```

supertest also offers a chainable, built-in expectation style if you prefer (it throws on mismatch):

```js
it('supports supertest .expect() assertions', async () => {
  await request(app)
    .get('/health')
    .expect('Content-Type', /json/)   // header matches regex
    .expect(200, { status: 'ok' });   // status AND body in one call
});
```

> BEST PRACTICE: Combine both styles when it helps clarity: use supertest's `.expect(200)` to fail fast on the status, then pull `res.body` into `expect()` for richer body assertions. Many teams standardize on `await request(app)...` + Jest `expect` for consistency with the rest of the suite.

> TIP: To test authenticated routes, set headers on the request: `.set('Authorization', 'Bearer ' + token)`. To test cookies/sessions across requests, use `const agent = request.agent(app)`; the agent persists cookies between calls, simulating a logged-in browser.

---

## 7. Integration Tests Against a Real-ish DB with mongodb-memory-server

Mocking the database is fine for unit tests, but it doesn't catch real bugs: a bad Mongoose query, a missing index, a schema validation rule that fires unexpectedly, a wrong projection. For confidence that your data layer actually works, you want to run against **real MongoDB** — but you don't want a shared, stateful, network-dependent database in your test suite (slow, flaky, requires setup).

**`mongodb-memory-server`** is the answer: it downloads a real `mongod` binary and runs it **in memory**, spun up fresh per test run. You get genuine MongoDB behavior with the speed and isolation of an ephemeral instance — no Docker, no external service.

```bash
npm install --save-dev mongodb-memory-server
npm install mongoose
```

### 7.1 A Mongoose model to test

```js
// src/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true, lowercase: true },
  age: { type: Number, min: 0 },
}, { timestamps: true });

module.exports = mongoose.model('User', userSchema);
```

### 7.2 Reusable in-memory DB setup

Factor the boilerplate into a helper so every DB test file uses the same lifecycle:

```js
// test/setupTestDB.js
const mongoose = require('mongoose');
const { MongoMemoryServer } = require('mongodb-memory-server');

let mongod;

function setupTestDB() {
  // Start the in-memory Mongo and connect ONCE for the file.
  beforeAll(async () => {
    mongod = await MongoMemoryServer.create();
    const uri = mongod.getUri();
    await mongoose.connect(uri); // modern Mongoose needs no extra options
  });

  // Wipe every collection between tests for full isolation.
  afterEach(async () => {
    const { collections } = mongoose.connection;
    for (const key of Object.keys(collections)) {
      await collections[key].deleteMany({});
    }
  });

  // Disconnect and stop the server so Jest can exit cleanly.
  afterAll(async () => {
    await mongoose.disconnect();
    await mongod.stop();
  });
}

module.exports = { setupTestDB };
```

### 7.3 Testing the model directly

```js
// src/models/User.test.js
const mongoose = require('mongoose');
const User = require('./User');
const { setupTestDB } = require('../../test/setupTestDB');

setupTestDB(); // installs the beforeAll/afterEach/afterAll hooks

describe('User model', () => {
  it('saves a valid user with timestamps', async () => {
    const user = await User.create({ name: 'Ada', email: 'Ada@X.com' });

    expect(user._id).toBeDefined();
    expect(user.email).toBe('ada@x.com');     // lowercase transform applied
    expect(user.createdAt).toBeInstanceOf(Date);
  });

  it('rejects a user with no name (schema validation)', async () => {
    await expect(User.create({ email: 'x@x.com' }))
      .rejects.toThrow(mongoose.Error.ValidationError);
  });

  it('enforces the unique email index', async () => {
    await User.create({ name: 'A', email: 'dup@x.com' });
    // Duplicate key error has MongoServerError code 11000:
    await expect(User.create({ name: 'B', email: 'dup@x.com' }))
      .rejects.toMatchObject({ code: 11000 });
  });

  it('rejects a negative age (min constraint)', async () => {
    await expect(User.create({ name: 'A', email: 'a@x.com', age: -1 }))
      .rejects.toThrow(/age/);
  });
});
```

> WARNING — Pitfall: The `unique: true` constraint is enforced by a **database index**, not by Mongoose validation. The index is built asynchronously after the model is created. In tests this is usually fine because the in-memory server builds it quickly, but if a uniqueness test is flaky, call `await User.init()` (or `await User.syncIndexes()`) in `beforeAll` to guarantee indexes are built before you rely on them.

### 7.4 Full-stack integration: route + DB together

The highest-value test combines supertest (HTTP) with the in-memory DB. This exercises routing, body parsing, the controller, validation, *and* the real database in one shot — the most realistic test you can write without a full e2e harness.

```js
// src/app.test.js  (DB-backed version)
const request = require('supertest');
const app = require('./app');               // app wired to Mongoose models
const User = require('./models/User');
const { setupTestDB } = require('../test/setupTestDB');

setupTestDB();

describe('Users API (with real DB)', () => {
  it('POST /users persists to the database', async () => {
    const res = await request(app)
      .post('/users')
      .send({ name: 'Ada', email: 'ada@x.com' });

    expect(res.status).toBe(201);

    // Verify it actually landed in the DB, not just the response:
    const inDb = await User.findById(res.body.id ?? res.body._id);
    expect(inDb).not.toBeNull();
    expect(inDb.name).toBe('Ada');
  });

  it('GET /users returns seeded users', async () => {
    // Seed directly via the model (fast, bypasses HTTP for setup):
    await User.create([
      { name: 'Ada', email: 'ada@x.com' },
      { name: 'Linus', email: 'linus@x.com' },
    ]);

    const res = await request(app).get('/users');

    expect(res.status).toBe(200);
    expect(res.body).toHaveLength(2);
  });
});
```

> BEST PRACTICE: Seed test data through the **model layer** (`User.create(...)`) rather than the HTTP API when the data is just a *precondition*. Reserve HTTP calls for the behavior you're actually testing. This keeps tests fast and failures unambiguous.

> TIP: The very first `MongoMemoryServer.create()` downloads the `mongod` binary, which can be slow and may need network access in CI. Cache the download directory between CI runs, or pre-pull the binary in a setup step. Once cached, startup is fast.

---

## 8. Structuring Tests, Fixtures/Factories, and Avoiding Flaky Tests

### 8.1 Organizing the suite

Two common layouts, both fine — pick one and be consistent:

- **Co-located:** `src/math.js` + `src/math.test.js` next to each other. Easy to find tests; encourages writing them.
- **Centralized:** a top-level `tests/` (or `__tests__/`) mirroring `src/`. Keeps source dirs clean; common for integration/e2e suites.

A pragmatic hybrid: co-locate unit tests, and put cross-cutting integration/e2e tests under `tests/integration/`. Keep shared helpers (like `setupTestDB.js`) in a `test/` folder.

> BEST PRACTICE: Name tests by **behavior**, not implementation. `it('returns 404 when the order does not exist')` survives refactors; `it('calls findOrderById then returns res.status(404)')` breaks the moment you rename a function. Test *what*, not *how*.

### 8.2 Fixtures and factories

A **fixture** is a fixed, reusable piece of test data. A **factory** is a function that *builds* test data, letting each test override just the fields it cares about. Factories beat hardcoded fixtures because they reduce duplication and make each test's intent obvious (you only specify what's relevant).

```js
// test/factories/user.factory.js
let counter = 0;

// Build a plain user object; override any field per test.
function buildUser(overrides = {}) {
  counter += 1;
  return {
    name: `User ${counter}`,
    email: `user${counter}@example.com`, // unique by default → no dup-key flakes
    age: 30,
    ...overrides, // caller's values win
  };
}

module.exports = { buildUser };
```

```js
// usage in a test
const { buildUser } = require('../test/factories/user.factory');

it('rejects underage users', async () => {
  // Only the relevant field is specified; the rest are sensible defaults:
  const res = await request(app).post('/users').send(buildUser({ age: -5 }));
  expect(res.status).toBe(400);
});
```

For richer fake data (names, emails, addresses), libraries like `@faker-js/faker` pair well with factories. Keep generated data **deterministic where it matters** — seed the faker RNG if a test depends on specific values.

> TIP: Make factory emails/usernames **unique by default** (a counter or `faker`), so tests that create multiple records don't trip uniqueness constraints. A huge fraction of "random" DB-test flakiness is just collisions on a unique field.

### 8.3 Killing flaky tests

A **flaky** test passes and fails non-deterministically without code changes. Flakes are corrosive: people start ignoring red builds, which defeats the entire point of testing. Root causes and fixes:

| Cause | Symptom | Fix |
| --- | --- | --- |
| **Shared state between tests** | Passes alone, fails in suite (or vice versa) | Reset state in `beforeEach`/`afterEach`; never rely on order |
| **Order dependence** | Fails when test order changes | Make each test self-sufficient; don't share mutable globals |
| **Real time / timeouts** | Fails on slow CI machines | Use fake timers; avoid `setTimeout` waits; don't assert on wall-clock |
| **Unhandled async** | Intermittent "passed" on broken code | Always `await`; use `expect.assertions(n)` for error paths |
| **Network / external services** | Fails when a service is down/slow | Mock external calls; use in-memory DB instead of shared DB |
| **Non-deterministic data** | Sometimes-duplicate keys, random sort order | Seed RNG; sort before asserting; unique factory data |
| **Leaked resources** | Suite hangs; "worker failed to exit" | Close servers/DB/timers in `afterAll`; run `--detectOpenHandles` |

```js
// Don't assert on insertion order from the DB — sort explicitly:
const users = await User.find().sort({ name: 1 }); // deterministic order
expect(users.map((u) => u.name)).toEqual(['Ada', 'Linus']);
```

> WARNING — Pitfall: Never write `await new Promise(r => setTimeout(r, 500))` "to let things settle." It's the #1 source of flakiness — too short on a busy CI box, too slow everywhere else. Wait on the actual condition (await the promise, poll for the state) instead of guessing a duration.

> BEST PRACTICE: If a test flakes, **fix it or delete it** — never `retry` it into submission. A retried flaky test is hiding a real race condition or leak that will eventually bite you in production.

---

## 9. Code Coverage and What "Good Coverage" Means

**Code coverage** measures how much of your source code runs during the test suite. Jest produces it with one flag:

```bash
npx jest --coverage
```

You'll see a table with four metrics:

| Metric | Question it answers |
| --- | --- |
| **Statements** | What % of statements executed? |
| **Branches** | What % of `if`/`else`/`?:`/`switch` paths executed? |
| **Functions** | What % of functions were called? |
| **Lines** | What % of lines executed? |

**Branch coverage is the most informative.** 100% line coverage can still miss the `else` branch of a condition if both halves live on lines that ran for other reasons. A test can execute a line without exercising its logic.

### 9.1 What "good" coverage is

Coverage tells you what is **definitely untested** (the red lines). It does **not** tell you what is *well* tested. You can have 100% coverage with assertions that prove nothing:

```js
it('useless 100%-coverage test', () => {
  add(2, 3); // line runs → counts as covered → but asserts NOTHING
});
```

So:

- **Coverage is a floor, not a ceiling.** Low coverage proves you have gaps; high coverage doesn't prove quality.
- **Chase ~80% as a healthy target** for most codebases; push critical modules (payments, auth, money math) toward 100% *branch* coverage; don't obsess over the last few percent of boilerplate.
- **Cover behavior, not lines.** Aim to test every meaningful *path and edge case*; coverage is the byproduct, not the goal.

You can enforce a minimum in config so coverage can't silently regress:

```js
// jest.config.js
module.exports = {
  collectCoverage: true,
  collectCoverageFrom: ['src/**/*.js', '!src/server.js'], // exclude bootstrap
  coverageThreshold: {
    global: { branches: 80, functions: 80, lines: 80, statements: 80 },
  },
};
```

> WARNING — Pitfall: Mandating "100% coverage" as a hard rule produces *worse* tests — developers write meaningless tests to hit lines (like the example above) instead of meaningful ones. Use thresholds to prevent regressions, and rely on **code review** to judge whether the assertions are actually worth anything.

---

## 10. A Brief TDD Workflow Walkthrough

**Test-Driven Development (TDD)** inverts the usual order: you write the test *first*, watch it fail, then write the minimum code to pass. The loop is **Red → Green → Refactor**:

1. **Red.** Write one small failing test for the next bit of behavior. Run it; confirm it fails *for the right reason*.
2. **Green.** Write the simplest code that makes it pass — even something crude. No gold-plating.
3. **Refactor.** Clean up the code (and tests) now that you have a safety net. Tests stay green.

Why bother? TDD forces you to define "done" before you start, produces inherently testable designs (you can't write code that's hard to test if the test came first), guarantees coverage of the behavior you build, and gives a tight feedback loop that keeps you focused.

### 10.1 Worked example — `slugify(title)`

**Step 1 — Red.** Specify the first behavior.

```js
// src/slugify.test.js
const { slugify } = require('./slugify');

it('lowercases and hyphenates a simple title', () => {
  expect(slugify('Hello World')).toBe('hello-world');
});
```

```js
// src/slugify.js  — stub so the import resolves; test should FAIL
function slugify(title) {
  return ''; // deliberately wrong
}
module.exports = { slugify };
```

Run: it fails (`'' !== 'hello-world'`). Good — the test can detect wrongness.

**Step 2 — Green.** Simplest passing implementation.

```js
function slugify(title) {
  return title.toLowerCase().replace(/\s+/g, '-');
}
```

Green. Now add the next behavior as a new failing test.

```js
it('strips punctuation', () => {
  expect(slugify('Hello, World!')).toBe('hello-world');
});

it('collapses multiple spaces and trims', () => {
  expect(slugify('  Node   &   Express  ')).toBe('node-express');
});
```

These fail. **Green** again with a fuller implementation:

```js
function slugify(title) {
  return title
    .toLowerCase()
    .trim()
    .replace(/[^a-z0-9\s-]/g, '') // drop non-alphanumerics
    .replace(/\s+/g, '-')          // spaces → hyphens
    .replace(/-+/g, '-');          // collapse repeated hyphens
}
```

**Step 3 — Refactor.** The code is already clean; if it weren't, you'd tidy it now while tests stay green. Add one more edge case to lock behavior:

```js
it('handles an already-slugified string idempotently', () => {
  expect(slugify('already-a-slug')).toBe('already-a-slug');
});
```

Each cycle is tiny and fast. After a few loops you have a fully-specified, fully-tested function and you never had to "go back and add tests."

> TIP: TDD shines for pure logic with clear inputs/outputs (parsers, validators, calculators, business rules). It's more awkward for exploratory or UI-glue code. Use it where it pays; don't treat it as a religion.

---

## 11. The Built-in `node:test` Runner

Since Node.js 18 (stable in 20+), Node ships a **built-in test runner** — `node:test` plus the `node:assert` assertion library. Zero dependencies, native ESM support, and a `--test` CLI. It's a compelling lightweight alternative to Jest, especially for libraries that don't want a heavy toolchain.

### 11.1 Basics

```js
// src/math.node.test.js
const { test, describe, it } = require('node:test'); // built-in
const assert = require('node:assert/strict');        // strict assertions
const { add, divide } = require('./math');

describe('add', () => {
  it('adds two numbers', () => {
    assert.equal(add(2, 3), 5);          // strict ===, like Jest's toBe
  });
});

test('divide throws on zero', () => {
  assert.throws(() => divide(1, 0), /Cannot divide by zero/);
});

test('deep equality', () => {
  assert.deepEqual({ id: 1 }, { id: 1 }); // like Jest's toEqual
});
```

Run it:

```bash
node --test                 # discovers *.test.js / test/ files
node --test --watch         # watch mode
node --test --experimental-test-coverage   # built-in coverage
```

### 11.2 Hooks, async, and mocking

The API deliberately mirrors Jest's mental model:

```js
const { test, describe, before, beforeEach, after, mock } = require('node:test');
const assert = require('node:assert/strict');

describe('user service', () => {
  before(async () => { /* one-time setup, can be async */ });
  beforeEach(() => { /* reset per test */ });
  after(async () => { /* teardown */ });

  it('awaits async work', async () => {
    const result = await Promise.resolve(42);
    assert.equal(result, 42);
  });

  it('mocks a function with node:test', () => {
    const fn = mock.fn((x) => x * 2);      // built-in mock function
    assert.equal(fn(21), 42);
    assert.equal(fn.mock.calls.length, 1); // call inspection
    assert.deepEqual(fn.mock.calls[0].arguments, [21]);
  });
});
```

`node:test` also supports `mock.method(obj, 'name', impl)` (like `jest.spyOn`), `mock.timers` (fake timers), and `t.test(...)` subtests via the test context object `t`.

### 11.3 supertest works here too

supertest is framework-agnostic — it doesn't care which runner you use, since it just needs your Express `app`:

```js
const { test } = require('node:test');
const assert = require('node:assert/strict');
const request = require('supertest');
const app = require('./app');

test('GET /health returns ok', async () => {
  const res = await request(app).get('/health');
  assert.equal(res.status, 200);
  assert.deepEqual(res.body, { status: 'ok' });
});
```

### 11.4 Jest vs `node:test`

| Aspect | Jest | `node:test` |
| --- | --- | --- |
| **Install** | `npm i -D jest` | Built in — zero deps |
| **Assertions** | Rich `expect` matchers | `node:assert` (leaner) |
| **Mocking** | Very powerful (`jest.mock`, auto-mock) | Capable but more manual |
| **ESM support** | Needs config/flags | Native, first-class |
| **Snapshots** | Built in | Built in (Node 22+) |
| **Watch / coverage** | Mature, polished | Built in, improving fast |
| **Ecosystem** | Huge (plugins, matchers) | Smaller, growing |
| **Startup speed** | Heavier | Lighter |

> BEST PRACTICE: Choose **Jest** for application projects that benefit from its mocking power, rich matchers, and ecosystem (the common case for an Express + Mongoose API). Choose **`node:test`** for libraries, CLIs, or minimalist projects where you want zero dependencies and native ESM. Both are excellent; the skills transfer directly between them.

---

## Common Pitfalls & Gotchas

- **`toBe` on objects.** `toBe` is reference equality; two equal-looking objects fail it. Use `toEqual`/`toStrictEqual` for structures.
- **Calling the function before `toThrow`.** `expect(f()).toThrow()` runs `f()` first and throws outside the matcher. Always wrap: `expect(() => f()).toThrow()`.
- **Forgetting to `await` async assertions.** Un-awaited `.resolves`/`.rejects` or promises make broken code pass silently. `await` (or `return`) everything async.
- **Asserting an error path without `expect.assertions(n)`.** A try/catch that never catches passes silently. Guard the count, or use `.rejects.toThrow()`.
- **Leaked handles.** A `beforeAll` that opens a DB/server without an `afterAll` to close it makes Jest hang or warn. Run `jest --detectOpenHandles`.
- **Dirty mocks between tests.** Mock call history/implementations persist. Use `clearMocks`/`restoreMocks` in config, or `jest.clearAllMocks()` in `afterEach`.
- **`jest.mock` factory referencing outer variables.** Hoisting forbids it unless the variable name starts with `mock`. Either inline the value or use the `mock` prefix.
- **Testing against a shared real database.** Slow, order-dependent, flaky. Use `mongodb-memory-server` for isolated, fast, real-MongoDB behavior.
- **Forgetting to separate `app` from `listen`.** supertest needs the bare `app`; calling `listen()` on import causes port conflicts and open handles in tests.
- **Sleeping with `setTimeout` to "fix" races.** Always flaky. Wait on the real condition, or use fake timers.
- **Worshiping coverage numbers.** 100% coverage with weak assertions tests nothing. Coverage finds gaps; it doesn't measure quality.
- **Relying on DB insertion order.** Mongo doesn't guarantee return order without a sort. Always `.sort()` before order-sensitive assertions.

## Best Practices

- **Keep tests independent and order-agnostic.** Reset all state in `beforeEach`/`afterEach`. A test should pass alone and in any order.
- **Follow Arrange–Act–Assert** and aim for one logical assertion-target per test, so a failure has exactly one cause.
- **Name tests by behavior** (`it('returns 404 when ...')`), not by implementation details that break on refactor.
- **Mock at the boundaries.** Mock I/O (network, third-party APIs, time) and unit-test pure logic; let integration tests use a real in-memory DB.
- **Separate `app` from `server`.** Export `app` for supertest; call `listen()` only in the production entry point.
- **Use factories for test data**, with unique-by-default fields to avoid constraint collisions.
- **Make the suite fast.** Fast suites get run constantly; slow ones get skipped. Favor unit + in-memory-DB integration over heavy e2e.
- **Enforce coverage thresholds** to prevent regression, but judge test *quality* in code review.
- **Treat flakiness as a bug.** Fix the root cause; never paper over it with retries or sleeps.
- **Run tests in CI on every push/PR.** A green local suite that no one runs in CI provides false confidence.
- **Write the test when you fix a bug** (regression test) so the same defect can never silently return.

## Key Interview Points

- **What is the testing pyramid?** Many fast unit tests at the base, fewer integration tests in the middle, a handful of slow e2e tests on top — balancing speed/cost against realism/confidence. The principle: most tests should be fast.
- **Unit vs integration vs e2e?** Unit = one module in isolation with mocked deps; integration = several modules together (often with a real DB); e2e = the whole system as a black box. Cost and realism increase up the stack.
- **`toBe` vs `toEqual`?** `toBe` is `Object.is`/`===` (identity & primitives); `toEqual` is recursive structural equality for objects/arrays. `toStrictEqual` additionally checks `undefined` props and types.
- **How do you test that something throws?** Pass a function to `expect`: `expect(() => f()).toThrow()`. For async, `await expect(p).rejects.toThrow()`.
- **How do you test async code reliably?** Make the test `async` and `await` the work, or use `.resolves`/`.rejects` (awaited). The runner must know when async work completes; un-awaited promises cause false passes.
- **`jest.fn` vs `jest.spyOn` vs `jest.mock`?** `jest.fn()` creates a standalone mock function; `jest.spyOn(obj, 'm')` wraps a real method to observe/override it (restorable); `jest.mock('path')` replaces an entire module with auto-mocks.
- **Why is `jest.mock` hoisted, and what's the `mock`-prefix rule?** Jest hoists `jest.mock` above imports so the mock is in place when the module loads; its factory can't reference outer-scope variables unless their names start with `mock`.
- **Stub vs mock vs spy vs fake?** Stub returns canned values; mock is asserted on (called? with what?); spy wraps a real function to record calls; fake is a lightweight working implementation.
- **How do you test Express routes without a running server?** supertest binds directly to the exported `app` object on an ephemeral port — which is why you must export `app` separately from `app.listen()`.
- **How do you integration-test the database layer?** `mongodb-memory-server` runs a real in-memory `mongod`, giving genuine MongoDB behavior with per-run isolation and no external dependency. Wipe collections between tests.
- **What does code coverage tell you (and not)?** It shows which code definitely ran (so it finds untested gaps), especially via **branch** coverage. It does *not* prove tests assert anything meaningful; high coverage with weak assertions is worthless.
- **What is TDD?** Red–Green–Refactor: write a failing test, write minimal code to pass, then refactor under the test's protection. It produces testable designs and guaranteed coverage of built behavior.
- **What causes flaky tests and how do you fix them?** Shared/ordered state, real-time waits, unhandled async, external services, non-deterministic data, leaked handles. Fix with isolation, fake timers, awaiting, mocking I/O, deterministic data, and proper teardown.
- **`node:test` vs Jest?** `node:test` is built-in (zero deps), native ESM, leaner; Jest has richer matchers, more powerful mocking, snapshots, and a huge ecosystem. Both run supertest tests.

## Practice Exercises

Work through these in order; each builds on the last. Try them before peeking at the solutions.

1. **Matchers warm-up.** Write a function `summarize(items)` that takes an array of `{ name, price }` and returns `{ count, total, names }` (an array of names). Then write unit tests covering: an empty array, a single item, and multiple items. Use at least four different matchers.

2. **Mocking a dependency.** Write `getWeather(city, client)` that calls `client.fetch(city)` (returns a promise resolving to `{ tempC }`) and returns a string `"<city>: <tempF>°F"` converting Celsius to Fahrenheit. Unit-test it with a mocked `client` — cover a successful fetch and a rejected fetch.

3. **Fake timers.** Write `pollUntil(checkFn, intervalMs)` that calls `checkFn()` every `intervalMs` until it returns `true`, then resolves. Test it using fake timers (no real waiting), asserting `checkFn` is called the right number of times.

4. **Express route with supertest.** Given a small Express app with `POST /todos` (validates `title` is present, returns 201 with the created todo and an `id`) and `GET /todos` (returns the list), write supertest integration tests for the happy paths and the validation failure.

5. **DB integration with mongodb-memory-server.** Add a Mongoose `Todo` model (`title` required, `done` boolean defaulting to `false`). Rewrite the app to persist with Mongoose, then write integration tests using `mongodb-memory-server` that verify creation persists to the DB and that `GET /todos` returns seeded data.

6. **TDD a validator.** Using strict Red–Green–Refactor, build `validatePassword(pw)` that returns `{ valid: boolean, errors: string[] }`. Rules: at least 8 chars, at least one digit, at least one uppercase letter. Write each test first, watch it fail, then implement.

### Solutions

**Solution 1 — Matchers warm-up**

```js
// summarize.js
function summarize(items) {
  return {
    count: items.length,
    total: items.reduce((sum, i) => sum + i.price, 0),
    names: items.map((i) => i.name),
  };
}
module.exports = { summarize };
```

```js
// summarize.test.js
const { summarize } = require('./summarize');

describe('summarize', () => {
  it('handles an empty array', () => {
    const result = summarize([]);
    expect(result.count).toBe(0);
    expect(result.total).toBe(0);
    expect(result.names).toEqual([]);     // deep equality for arrays
  });

  it('handles a single item', () => {
    const result = summarize([{ name: 'Pen', price: 2 }]);
    expect(result).toEqual({ count: 1, total: 2, names: ['Pen'] });
  });

  it('handles multiple items', () => {
    const result = summarize([
      { name: 'Pen', price: 2 },
      { name: 'Pad', price: 3.5 },
    ]);
    expect(result.count).toBe(2);
    expect(result.total).toBeCloseTo(5.5);   // float-safe
    expect(result.names).toContain('Pad');   // membership
    expect(result.names).toHaveLength(2);    // length
  });
});
```

**Solution 2 — Mocking a dependency**

```js
// getWeather.js
async function getWeather(city, client) {
  const { tempC } = await client.fetch(city);
  const tempF = Math.round((tempC * 9) / 5 + 32);
  return `${city}: ${tempF}°F`;
}
module.exports = { getWeather };
```

```js
// getWeather.test.js
const { getWeather } = require('./getWeather');

describe('getWeather', () => {
  it('converts Celsius to Fahrenheit on success', async () => {
    const client = { fetch: jest.fn().mockResolvedValue({ tempC: 20 }) };
    await expect(getWeather('Paris', client)).resolves.toBe('Paris: 68°F');
    expect(client.fetch).toHaveBeenCalledWith('Paris');
  });

  it('propagates a fetch error', async () => {
    const client = { fetch: jest.fn().mockRejectedValue(new Error('down')) };
    await expect(getWeather('Paris', client)).rejects.toThrow('down');
  });
});
```

**Solution 3 — Fake timers**

```js
// pollUntil.js
function pollUntil(checkFn, intervalMs) {
  return new Promise((resolve) => {
    const id = setInterval(() => {
      if (checkFn()) {
        clearInterval(id);
        resolve();
      }
    }, intervalMs);
  });
}
module.exports = { pollUntil };
```

```js
// pollUntil.test.js
const { pollUntil } = require('./pollUntil');

describe('pollUntil', () => {
  beforeEach(() => jest.useFakeTimers());
  afterEach(() => jest.useRealTimers());

  it('resolves once checkFn returns true', async () => {
    // Returns false twice, then true on the 3rd call:
    const checkFn = jest.fn()
      .mockReturnValueOnce(false)
      .mockReturnValueOnce(false)
      .mockReturnValue(true);

    const promise = pollUntil(checkFn, 100);

    // Advance three intervals; the 3rd check returns true and resolves.
    await jest.advanceTimersByTimeAsync(300);
    await promise; // should be resolved now

    expect(checkFn).toHaveBeenCalledTimes(3);
  });
});
```

**Solution 4 — Express route with supertest**

```js
// todoApp.js
const express = require('express');
const app = express();
app.use(express.json());

const todos = [];
let nextId = 1;

app.post('/todos', (req, res) => {
  const { title } = req.body;
  if (!title) return res.status(400).json({ error: 'title is required' });
  const todo = { id: nextId++, title, done: false };
  todos.push(todo);
  res.status(201).json(todo);
});

app.get('/todos', (req, res) => res.json(todos));

module.exports = app;
```

```js
// todoApp.test.js
const request = require('supertest');
const app = require('./todoApp');

describe('Todos API (in-memory)', () => {
  it('creates a todo (201) and lists it', async () => {
    const created = await request(app).post('/todos').send({ title: 'Test' });
    expect(created.status).toBe(201);
    expect(created.body).toMatchObject({ title: 'Test', done: false });
    expect(created.body).toHaveProperty('id');

    const list = await request(app).get('/todos');
    expect(list.status).toBe(200);
    expect(list.body).toContainEqual(
      expect.objectContaining({ title: 'Test' }),
    );
  });

  it('rejects a todo with no title (400)', async () => {
    const res = await request(app).post('/todos').send({});
    expect(res.status).toBe(400);
    expect(res.body.error).toMatch(/required/);
  });
});
```

**Solution 5 — DB integration with mongodb-memory-server**

```js
// models/Todo.js
const mongoose = require('mongoose');
const todoSchema = new mongoose.Schema({
  title: { type: String, required: true },
  done: { type: Boolean, default: false },
});
module.exports = mongoose.model('Todo', todoSchema);
```

```js
// todoApp.db.js
const express = require('express');
const Todo = require('./models/Todo');
const app = express();
app.use(express.json());

app.post('/todos', async (req, res, next) => {
  try {
    const todo = await Todo.create({ title: req.body.title });
    res.status(201).json(todo);
  } catch (err) {
    if (err.name === 'ValidationError') {
      return res.status(400).json({ error: err.message });
    }
    next(err);
  }
});

app.get('/todos', async (req, res) => {
  const todos = await Todo.find().sort({ _id: 1 }); // deterministic order
  res.json(todos);
});

module.exports = app;
```

```js
// todoApp.db.test.js
const request = require('supertest');
const mongoose = require('mongoose');
const { MongoMemoryServer } = require('mongodb-memory-server');
const app = require('./todoApp.db');
const Todo = require('./models/Todo');

let mongod;

beforeAll(async () => {
  mongod = await MongoMemoryServer.create();
  await mongoose.connect(mongod.getUri());
});
afterEach(async () => {
  await Todo.deleteMany({});
});
afterAll(async () => {
  await mongoose.disconnect();
  await mongod.stop();
});

describe('Todos API (real in-memory DB)', () => {
  it('POST /todos persists to the database', async () => {
    const res = await request(app).post('/todos').send({ title: 'Buy milk' });
    expect(res.status).toBe(201);

    const inDb = await Todo.findById(res.body._id);
    expect(inDb).not.toBeNull();
    expect(inDb.title).toBe('Buy milk');
    expect(inDb.done).toBe(false);
  });

  it('POST /todos with no title returns 400', async () => {
    const res = await request(app).post('/todos').send({});
    expect(res.status).toBe(400);
  });

  it('GET /todos returns seeded data', async () => {
    await Todo.create([{ title: 'A' }, { title: 'B' }]);
    const res = await request(app).get('/todos');
    expect(res.status).toBe(200);
    expect(res.body).toHaveLength(2);
    expect(res.body.map((t) => t.title)).toEqual(['A', 'B']);
  });
});
```

**Solution 6 — TDD a validator**

Follow Red–Green–Refactor. Write each test, watch it fail, then implement just enough.

```js
// validatePassword.test.js
const { validatePassword } = require('./validatePassword');

describe('validatePassword', () => {
  it('accepts a strong password', () => {
    expect(validatePassword('Abcdef12')).toEqual({ valid: true, errors: [] });
  });

  it('rejects a short password', () => {
    const result = validatePassword('Ab1');
    expect(result.valid).toBe(false);
    expect(result.errors).toContain('Must be at least 8 characters');
  });

  it('rejects a password with no digit', () => {
    const result = validatePassword('Abcdefgh');
    expect(result.errors).toContain('Must contain a digit');
  });

  it('rejects a password with no uppercase letter', () => {
    const result = validatePassword('abcdef12');
    expect(result.errors).toContain('Must contain an uppercase letter');
  });

  it('accumulates multiple errors', () => {
    const result = validatePassword('abc');
    expect(result.valid).toBe(false);
    expect(result.errors).toHaveLength(2); // too short + no digit
  });
});
```

```js
// validatePassword.js  (the Green implementation)
function validatePassword(pw) {
  const errors = [];
  if (pw.length < 8) errors.push('Must be at least 8 characters');
  if (!/\d/.test(pw)) errors.push('Must contain a digit');
  if (!/[A-Z]/.test(pw)) errors.push('Must contain an uppercase letter');
  return { valid: errors.length === 0, errors };
}
module.exports = { validatePassword };
```

## Summary / Cheat Sheet

| Topic | Key takeaway |
| --- | --- |
| **Why test** | Regression safety, refactoring confidence, executable spec, fast feedback, design pressure |
| **Pyramid** | Many unit → fewer integration → few e2e; keep most tests fast |
| **Test types** | Unit (isolated, mocked), integration (real-ish DB), e2e (whole system) |
| **Structure** | `describe` groups; `it`/`test` define; Arrange–Act–Assert; one reason to fail |
| **Matchers** | `toBe` (identity) vs `toEqual` (structure); `toThrow` needs a function; `.not` negates |
| **Hooks** | `beforeAll`/`afterAll` (once), `beforeEach`/`afterEach` (per test); outer→inner before, inner→outer after |
| **Mocks** | `jest.fn()` (standalone), `jest.spyOn()` (wrap real, restore), `jest.mock()` (whole module, hoisted) |
| **Mock hygiene** | `clearMocks`/`resetMocks`/`restoreMocks`; the `mock`-prefix rule in factories |
| **Timers** | `useFakeTimers()` + `advanceTimersByTime(ms)`; never sleep with real `setTimeout` |
| **Async** | `async`/`await`, or `await expect(p).resolves/.rejects`; always await; `expect.assertions(n)` for error paths |
| **HTTP tests** | supertest on exported `app`; separate `app` from `app.listen()` |
| **DB tests** | `mongodb-memory-server` for real in-memory Mongo; wipe collections in `afterEach` |
| **Fixtures** | Factories with unique-by-default fields beat hardcoded fixtures |
| **Flakiness** | Caused by shared state, real time, unhandled async, external deps; fix the root cause |
| **Coverage** | Branch coverage matters most; ~80% target; finds gaps, doesn't prove quality |
| **TDD** | Red → Green → Refactor; tiny loops; test-first yields testable design |
| **node:test** | Built-in, zero-dep, native ESM; `node --test`; mirrors Jest's mental model |

> BEST PRACTICE — the one rule to remember: **a test should fail for exactly one reason, run fast, and stay green for the right reason.** Everything in this module serves that goal.
