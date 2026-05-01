# Testing Fundamentals

Conceptual foundation for writing enterprise-grade tests. Read this before `examples.md`.

---

## 1. The Testing Pyramid

```
         /\
        /E2E\         ← Few, slow, expensive, fragile
       /------\
      /  Int   \      ← Some, moderate speed, verify wiring
     /----------\
    /    Unit    \    ← Many, fast, isolated
   /--------------\
```

### The building analogy

| Type | Building equivalent | Software equivalent |
|------|---------------------|---------------------|
| Unit | Check each brick (resistance, cracks, firing) | Check each function/method in isolation |
| Integration | Check walls (bricks + cement + rebar) hold load | Check multiple pieces work together: repo + DB, handler + mapper |
| E2E | Walk people through the finished building | Simulate a real user end-to-end: login → action → result |

### Tradeoffs per layer

| Property | Unit | Integration | E2E |
|----------|------|-------------|-----|
| Speed | milliseconds | seconds | seconds to minutes |
| Cost to write | low | medium | high |
| Cost to maintain | low | medium | VERY high |
| Error precision | surgical (exact line) | reasonable (which component) | vague ("something broke") |
| Confidence provided | low/medium | high | very high (real simulation) |

**Rule of thumb**: `Unit >> Integration >> E2E` in count. Typical healthy distribution: 70 / 20 / 10.

### Anti-pattern: the ice cream cone 🍦

Inverted pyramid — few units, many E2E — gives vague error signals, slow feedback, and flaky CI. Rejected.

---

## 2. FIRST Principles + Extensions

Classic acronym: **FIRST**.

| Letter | Meaning | Practical |
|--------|---------|-----------|
| **F** — Fast | A unit test runs in milliseconds. If it takes 2s touching a real DB, it's NOT a unit test. |
| **I** — Independent | Test does not depend on execution order or other tests' state. Run one alone → it passes. |
| **R** — Repeatable | Run 1000 times → same result. No unmocked `Date` or `Math.random`. |
| **S** — Self-validating | Test outputs PASS or FAIL. No human inspection of logs "to see if it looks right". |
| **T** — Timely | Write tests BEFORE or ALONGSIDE the code, not three months later. |

### Extensions the industry considers equally critical

| Property | Meaning |
|----------|---------|
| **Behavior-focused** | Test WHAT the code does, not HOW. Refactor without breaking test = test is healthy. |
| **Readable** | A teammate who has never seen the code understands the test in 30 seconds. |
| **Trustworthy** | When it fails, you trust there's a real bug. A flaky test is WORSE than no test. |

---

## 3. What to test vs what NOT to test

### ✅ DO test

- **Business rules**: "if user has no balance, cannot purchase"
- **Edge cases**: null, empty, boundary values, cutoff dates
- **Conditional branches**: every `if`, `switch`, `throw`
- **Public contracts**: the public API of a module/class
- **Bugs you fixed** (regression tests): a test that fails without the fix
- **Data transformations**: mappers, parsers, formatters
- **Validations**: Zod schemas, domain rules

### ❌ DON'T test

- **Framework code** — NestJS `@Controller` is already tested by Nest
- **Trivial getters/setters** — `get name() { return this._name }` is noise
- **Third-party libraries** — don't test `bcrypt` hashes; test YOUR code calls it with the right args
- **Private implementation details** — test the public method, not `_privateHelper`
- **Generated code** — DTOs, types

### The smell test

Every time you write a test, ask: **"what REAL bug does this test catch?"**. If you can't answer in one sentence, probably it adds nothing.

**Bad example** (catches no bug):
```ts
it('should create a user', () => {
  const user = new User({ name: 'Juan' })
  expect(user.name).toBe('Juan') // TypeScript already guarantees this
})
```

**Good example** (catches a real bug):
```ts
it('rejects user creation when name is empty', () => {
  expect(() => new User({ name: '' })).toThrow('Name is required')
})
```

---

## 4. Mock Hell — the #1 anti-pattern in NestJS

```ts
// ❌ MAL — tests coupling, not behavior
it('creates a task', async () => {
  const repo = { save: jest.fn().mockResolvedValue(true) }
  const logger = { log: jest.fn() }
  const handler = new CreateTaskHandler(repo as any, logger as any)

  await handler.execute({ title: 'test' })

  expect(repo.save).toHaveBeenCalled()
  expect(logger.log).toHaveBeenCalled()
})
```

Problems:
- `toHaveBeenCalled()` only verifies the function was called, NOT with what
- Testing `logger.log` couples to implementation (remove the log → test breaks, no bug)
- `as any` kills type safety
- No failure path covered

Fix: assert ARGUMENTS and RETURN VALUES. See `examples.md` for the correct version.

---

## 5. Regression tests — the 1 test you OWE every bug

Every time you fix a bug:

1. Before fixing, write a test that reproduces the bug → it FAILS.
2. Apply the fix → test PASSES.
3. The test stays forever.

This is non-negotiable. Without it, the same bug will return in 3 months when someone refactors.
