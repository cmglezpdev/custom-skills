# Positive Testing Patterns for DaraMex

Patterns that make DaraMex tests robust, readable, and refactor-safe. Unlike `real-world-antipatterns.md` which lists things to avoid, this file lists things to DO. Each pattern is accompanied by a real example from the codebase.

---

## 1. AAA Pattern — Arrange / Act / Assert (MANDATORY)

Every test MUST have three phases, separated by explicit `// Arrange`, `// Act`, `// Assert` comments. This is a **project-level convention**, not a style choice.

1. **Arrange** — set up the state (fixtures, mocks, inputs)
2. **Act** — execute the behavior under test (ideally one call)
3. **Assert** — verify the result

### The ONLY acceptable form in DaraMex

```ts
it('assigns a sortIndex less than the first element', async () => {
  // Arrange
  mockRepo.findById.mockResolvedValue(taskC);
  mockRepo.findByStatusIdOrdered.mockResolvedValue([taskA, taskB, taskC]);
  mockRepo.save.mockImplementation(async (t) => t);

  // Act
  const result = await handler.execute(
    new ReorderTaskCommand('id-c', null, TEST_AGENCY_ID, TEST_ORG_ID),
  );

  // Assert
  expect(result.isSuccess).toBe(true);
  const savedTask = mockRepo.save.mock.calls[0][0] as Task;
  expect(savedTask.sortIndex < taskA.sortIndex).toBe(true);
});
```

### Rules

- The three comments (`// Arrange`, `// Act`, `// Assert`) are REQUIRED on every `it` block — even trivial ones.
- If a phase is empty, omit that specific comment (e.g. a test with no Arrange starts directly with `// Act`).
- Blank lines between phases are OPTIONAL but recommended for further visual separation.
- Comments are preferred over blank-line-only separation because they serve as a mental anchor for developers still learning the pattern, and they SURVIVE formatter/prettier changes that might collapse blank lines.

### Rejected alternatives

- **Blank lines only**: not enough. A reader skimming the test may miss phase boundaries. Formatters can also collapse blank lines unexpectedly.
- **No separation**: anti-pattern. If AAA is not visible, the test is unreadable under pressure. Rewrite.

### Why it matters

- When a test fails, the reader jumps straight to Assert to understand the expectation, then walks back to Arrange to understand the state.
- The comments double as documentation for developers new to testing.
- Consistency across the codebase: anyone opening any spec sees the same structure.

---

## 2. Factory functions for fixtures

When you build the same domain entity in many tests, centralize its construction in a typed factory. Saves repetition and future-proofs against shape changes.

### Pattern

```ts
// From reorder-task.command.spec.ts
const makeTask = (
  id: string,
  sortIndex: string,
  statusId = 'status-1',
): Task => {
  return Task.rehydrate({
    id,
    orgId: TEST_ORG_ID,
    agencyId: TEST_AGENCY_ID,
    description: `Task ${id}`,
    startedAt: null,
    finishedAt: null,
    statusId,
    creatorId: 'creator-1',
    relatedUserIds: [],
    tagIds: [],
    attachmentIds: [],
    sortIndex,
    parentId: null,
    projectId: null,
    createdAt: FIXED_DATE,
    updatedAt: FIXED_DATE,
  });
};

const taskA = makeTask('id-a', 'd00000');
const taskB = makeTask('id-b', 'm00000');
const taskC = makeTask('id-c', 't00000');
```

### Rules

- Typed return type (`(): Task`, not `any`)
- Sensible defaults for ALL fields
- Accept parameters for the fields that vary per test
- If an entity field is added in production, you update the factory ONCE — all tests using it still pass

### When not to use it

- Extremely simple value objects where `new Money(100)` is clearer than `makeMoney(100)`.
- One-off fixtures used in a single test.

---

## 3. Constants at top of file

Promote magic values (IDs, dates) to constants at the top of the spec file:

```ts
// From reorder-task.command.spec.ts
const TEST_ORG_ID = '019d0000-0000-7000-8000-000000000001';
const TEST_AGENCY_ID = '019d0000-0000-7000-8000-000000000002';
const FIXED_DATE = new Date('2026-01-15T10:00:00.000Z');
```

### Why

- Single point of change when the format or meaning evolves.
- Readable assertions: `new ReorderTaskCommand('id-c', null, TEST_AGENCY_ID, TEST_ORG_ID)` vs raw UUIDs.
- Makes it obvious which IDs are "test placeholders" vs real data.

### Convention for dates

Always use `new Date('YYYY-MM-DDTHH:MM:SS.sssZ')` in UTC. Never `new Date()` in fixtures — that breaks FIRST-Repeatable.

---

## 4. `Task.create()` vs `Task.rehydrate()` — DDD test fixture pattern

A Clean/Hexagonal entity typically has TWO factory methods. Use the correct one in tests:

| Method | Purpose | Behavior |
|--------|---------|----------|
| `Entity.create(props)` | Create a brand new entity | Applies creation validations, emits domain events (e.g. `TaskCreated`), generates ID |
| `Entity.rehydrate(props)` | Reconstruct an existing entity from storage | No validations, no events, accepts the ID as input |

### Rule for tests

**Use `rehydrate()` for fixtures.** Reasons:

- You want full control over every field (including `id`, `createdAt`, `sortIndex`).
- You don't want domain events firing as side effects of building a fixture.
- You want to be able to construct states to test how the domain handles them.

**Use `create()` only when** testing the creation flow itself (e.g. "creating a Task with empty title throws DomainError").

### Red flag

If your entity doesn't have `rehydrate()` and you find yourself doing `new Task(...)` with `as any`, the domain design is missing this pattern. The fix is to add `rehydrate()` to the entity, not to bypass it with casts.

---

## 5. Type-safe mocks with `jest.Mocked<IPort>`

### The gold standard

```ts
let mockRepo: jest.Mocked<ITaskRepository>;

beforeEach(() => {
  mockRepo = {
    findById: jest.fn(),
    findAll: jest.fn(),
    findByStatusId: jest.fn(),
    findByStatusIdOrdered: jest.fn(),
    findLastSortIndexByStatusId: jest.fn(),
    save: jest.fn(),
    findByDateRange: jest.fn(),
    delete: jest.fn(),
    deleteByStatusId: jest.fn(),
    countByStatusId: jest.fn(),
    findWithAccessFilter: jest.fn(),
    findNoStatus: jest.fn(),
  };

  handler = new ReorderTaskHandler(mockRepo);
});
```

### What this gives you

- If you add `findByProjectId(projectId: string)` to `ITaskRepository`, TypeScript REFUSES to compile the mock until you add `findByProjectId: jest.fn()`.
- All mock methods are typed: `mockRepo.save.mockResolvedValue(...)` is restricted to valid return values.
- Refactoring becomes a compile-time checked operation.

### Sparse mocks (when only some methods are needed)

Use `Partial<jest.Mocked<IPort>>`:

```ts
const repo: Partial<jest.Mocked<ITaskRepository>> = {
  save: jest.fn(),
  findById: jest.fn(),
};
```

### Never do this

```ts
const repo = {} as any
const repo = {} as ITaskRepository           // still unsafe
const repo = { save: jest.fn() } as unknown as ITaskRepository
```

These are all forms of opting out of type safety. The type-system benefit you paid for (Clean Arch) is lost.

---

## 6. Mock isolation strategies

Two valid approaches for ensuring tests don't leak state:

### Strategy A — Recreate in `beforeEach` (preferred)

```ts
beforeEach(() => {
  mockRepo = {
    findById: jest.fn(),
    save: jest.fn(),
    // ... all methods
  };
  handler = new ReorderTaskHandler(mockRepo);
});
```

Pros: Zero leak between tests.  
Cons: Slightly slower (creates new objects).

### Strategy B — `jest.clearAllMocks()` in `beforeEach`

```ts
beforeEach(() => {
  jest.clearAllMocks();
});
```

Pros: Faster if the mock object is expensive to build.  
Cons: Does NOT reset `mockReturnValue` / `mockResolvedValue` — those persist unless you call `mockReset`.

### Rule of thumb

Default to Strategy A. Switch to B only if test setup time becomes measurable (rare).

---

## 7. Passthrough mocks

When the mock needs to "return what it received":

```ts
mockRepo.save.mockImplementation(async (t) => t);
```

### When to use it

- When the handler uses the return value of the mocked call: `const saved = await repo.save(task); return Result.ok(saved)`.
- Even when the current handler doesn't use the return value, configuring a sensible passthrough is defensive — the handler may start using it in the future and the test won't suddenly break.

### Variations

```ts
// Return a transformed value
mockRepo.save.mockImplementation(async (t) => ({ ...t, id: 'new-id' }));

// Throw on certain inputs
mockRepo.save.mockImplementation(async (t) => {
  if (!t.title) throw new Error('title required');
  return t;
});
```

---

## 8. Test data that makes invariants obvious

When values participate in ordering, use values whose NATURAL string/number comparison expresses the intended order.

### Example from operations

```ts
const taskA = makeTask('id-a', 'd00000');  // first  (D)
const taskB = makeTask('id-b', 'm00000');  // middle (M)
const taskC = makeTask('id-c', 't00000');  // last   (T)
```

`'d00000' < 'm00000' < 't00000'` lexicographically, mirroring D → M → T. A reader understands the ordering at a glance.

### Anti-example

```ts
const taskA = makeTask('id-a', 'xyz123abc');   // ???
const taskB = makeTask('id-b', '000abc999');   // ???
```

You can't tell the ordering without running the comparator mentally. Don't make your reader work.

### Principle

Test data is not random noise. It's DESIGNED to make the invariant being tested self-evident.

---

## 9. Test invariants, not exact values

Prefer relational assertions (`<`, `>`, `between`) over literal comparisons — when the exact value is an implementation detail.

### Example (DO)

```ts
const savedTask = mockRepo.save.mock.calls[0][0] as Task;
expect(savedTask.sortIndex < taskA.sortIndex).toBe(true);
```

This tests the RULE: "rank-to-front produces a rank less than the first". It survives any algorithmic change in `LexoRankService` as long as the invariant holds.

### Example (DON'T)

```ts
expect(savedTask.sortIndex).toBe('c50000');
```

This tests the INTERNAL IMPLEMENTATION of LexoRank. A precision upgrade (`c500000`) or format change breaks the test even though behavior is correct.

### Exception: when exact value IS the spec

```ts
// From reorder-task.command.spec.ts — LexoRank integration tests
const expected = LexoRankService.midpoint(undefined, taskA.sortIndex);
expect(savedTask.sortIndex).toBe(expected);
```

Here the exact value is derived from the same algorithm — so the test is pinning that the handler delegates to `LexoRankService.midpoint` correctly. That's a legitimate integration assertion.

### Rule

Ask yourself: "does this value come from an algorithm whose details I control today but might change?" If yes → test the invariant. If the value is a stable business constant or derived from the same algorithm → test the literal.

### The combined-layer pattern — invariant as safety net

When the handler delegates to a pure helper (like `LexoRankService.midpoint`), consider asserting BOTH:

```ts
// Act
const result = await handler.execute(command);

// Assert
expect(result.isSuccess).toBe(true);

const savedTask = mockRepo.save.mock.calls[0][0] as Task;

// Layer 1 — delegation (exact value from same function)
const expected = LexoRankService.midpoint(undefined, taskA.sortIndex);
expect(savedTask.sortIndex).toBe(expected);

// Layer 2 — invariant (safety net against bugs IN the helper itself)
expect(savedTask.sortIndex < taskA.sortIndex).toBe(true);
```

Why layer 2 matters: if `LexoRankService.midpoint` itself has a bug (returns the wrong value), Layer 1 still passes because the test and the production code share the bug. Layer 2 is independent — it verifies the ACTUAL ordering invariant. It's your safety net against bugs in the helper.

This combined-layer pattern is the strongest form. Use it whenever the production code delegates math or computation to a pure helper.

---

## 10. `describe.each` and `it.each` — when to use them

`describe.each` / `it.each` run the same test structure with multiple input rows. Useful ONLY when:

1. The assertions are IDENTICAL across cases (only inputs change).
2. The test name can be meaningfully parameterized.

### Good case

```ts
describe('isAdult', () => {
  it.each([
    { age: 0,  expected: false },
    { age: 17, expected: false },
    { age: 18, expected: true  },
    { age: 99, expected: true  },
  ])('returns $expected for age $age', ({ age, expected }) => {
    expect(isAdult(age)).toBe(expected);
  });
});
```

### Bad case — different assertions per row

```ts
// DO NOT use .each when assertions differ per case
describe.each([
  { name: 'to front', afterId: null,   /* expects sortIndex < first */ },
  { name: 'to end',   afterId: 'id-c', /* expects sortIndex > last  */ },
  { name: 'between',  afterId: 'id-a', /* expects sortIndex > A && < B */ },
])('reorder $name', ...)
```

Here the assertions are fundamentally different. Writing them as `.each` forces conditionals inside the test body, which kills readability. Keep them as separate `it` blocks (as in `reorder-task.command.spec.ts`).

### Typical use cases in DaraMex

- Zod schema validation with many invalid inputs
- Pure validators (email, password strength)
- Testing the same port contract against multiple implementations

---

## 11. Clarity > DRY in tests

Unlike production code, DRY (Don't Repeat Yourself) is NOT a primary virtue in tests. Repetition is often a feature, not a bug.

### Why

- Tests are documentation. A reader opens ONE test and should understand it without jumping to helpers.
- Abstractions over test code can hide bugs — a helper function with a subtle logic error silently corrupts dozens of tests.
- Tests change less frequently than production code. The "maintenance burden" of repetition is smaller than the "readability burden" of abstraction.

### When repetition IS acceptable

- Three tests that each spin up a similar 4-line Arrange. Keep the repetition.
- Four tests that each call `handler.execute()` with a different command. Keep the repetition.

### When abstraction IS useful in tests

- Fixture factories (`makeTask`) — centralizes entity construction
- Top-level constants (`TEST_ORG_ID`) — single source of truth for magic values
- `beforeEach` setup — when literally every test needs the same starting state

The difference: abstractions that ELIMINATE incidental complexity are good. Abstractions that HIDE test intent are bad.

---

## 12. Casts in tests — `as T` vs `as any`

Not all casts are equal.

### Acceptable: cast to a more specific type when type inference is too generic

```ts
// From reorder-task.command.spec.ts
const savedTask = mockRepo.save.mock.calls[0][0] as Task;
expect(savedTask.sortIndex < taskA.sortIndex).toBe(true);
```

Jest types `mock.calls` generically; `Task` is specific. The cast is pragmatic and preserves type checking downstream.

### Unacceptable: cast to `any` or `unknown`

```ts
const handler = new ReorderTaskHandler(repo as any);       // ❌
const result = (await handler.execute(cmd)) as any;        // ❌
```

These turn off type checking entirely. Replace with typed mocks (`jest.Mocked<IPort>`) or properly-typed return values.

### Rule

- `as T` where T is more specific than the inferred type → OK in tests
- `as any` or `as unknown as T` → replace with proper typing

---

## 13. Diagnosing test failures (the 3 causes)

Before fixing a failing test, identify WHICH of these it is:

1. **Test rotten** — production code was refactored; the test still imports old shapes/types. Symptom: TS errors about type mismatches, missing properties, removed functions.
   - Fix: update the test to match current shapes.

2. **Production broken** — the test is correct; the production code doesn't compile.
   - Fix: fix the production code. Never patch with `as any` in the test.

3. **Build/workspace incomplete** — the test and production are both correct, but a workspace package isn't built.
   - Symptom: `Cannot find module '@repo/something'` even though the import path is correct.
   - Fix: run the workspace build (`pnpm build --filter @repo/something` or equivalent). DO NOT touch the test.

### Reflex before touching anything

When you see `Test suite failed to run`, ask three questions BEFORE editing:

1. Does the import path exist in the codebase (literally)?
2. Does the imported type still have the shape the test expects?
3. Is the package actually BUILT?

Answer all three before modifying anything.

| Cause | Symptom | Fix |
|-------|---------|-----|
| **Test rotten** | TS errors about types that shifted — the test imports old shapes | Update the test to match current shapes |
| **Production broken** | Type errors in production code too | Fix the production code. Do NOT patch with `as any` in the test |
| **Build/workspace incomplete** | `Cannot find module '@repo/xxx'` despite correct import paths | Run the workspace build. Do NOT modify the test |

---

## 14. Writing strong assertions

A test is only as strong as its weakest assertion. "It passes" does NOT mean "it protects". Most test suites rot because the asserts themselves were weak from day one — they verified something, but not the thing that mattered.

### The 3-question weakness test

Before committing a test, ask yourself:

1. **"If I delete the entire implementation under test, does this test still pass?"**
   - If yes → your assertions are weak. They depend on something other than the behavior.
2. **"If I introduce a subtle bug (wrong field assigned, off-by-one, swapped arguments), would this assertion catch it?"**
   - If no → strengthen with more specific expectations.
3. **"Is this single assertion enough on its own, or does my test need layered asserts?"**
   - Most tests need multiple asserts to cover result + state + side effects.

### Common weak assertions and how to strengthen them

| Weak pattern | Why it's weak | How to strengthen |
|--------------|---------------|-------------------|
| `expect(result.isSuccess).toBe(true)` alone | Handler could return `Result.ok` with wrong or empty payload and test still passes | Verify the returned value AND the persisted state: `expect(savedTask.id).toBe(expectedId)`, `expect(savedTask.status).toBe('pending')` |
| `expect(fn).toHaveBeenCalled()` alone | See anti-pattern #2 in `real-world-antipatterns.md` — Mock Hell | Use `toHaveBeenCalledWith(expect.objectContaining({...}))` or inspect `mock.calls[0][0]` |
| `expect(result).toBeTruthy()` | `{}`, `[]`, `'x'`, `1` are all truthy. Catches almost nothing. | Assert the specific shape: `toEqual`, `toMatchObject`, or concrete field asserts |
| `expect(list).toHaveLength(3)` as sole assert | Right count + wrong content still passes | Combine with `expect(list[0]).toMatchObject({...})` or `expect(list).toEqual([...])` |
| `expect(obj.id).toBeDefined()` | TypeScript already enforces this if typed correctly | Assert the actual value, or delete the test |
| `expect(() => fn()).not.toThrow()` alone | "Doesn't crash" is a very low bar | Assert what fn RETURNED / what STATE changed |

### The layered-assert template for happy paths

A strong happy-path test for an application handler typically asserts **four layers**:

```ts
it('persists a new Task and returns its id on success', async () => {
  // Arrange
  mockRepo.save.mockImplementation(async (t) => t);

  // Act
  const result = await handler.execute({ title: 'Invoice A', orgId: 'org-1' });

  // Assert — Layer 1: result type
  expect(result.isSuccess).toBe(true);

  // Assert — Layer 2: result content
  const taskId = result.unwrap();
  expect(taskId).toMatch(/^[0-9a-f-]{36}$/); // or whatever your id format is

  // Assert — Layer 3: side effect (was the right thing saved?)
  expect(mockRepo.save).toHaveBeenCalledTimes(1);
  const savedTask = mockRepo.save.mock.calls[0][0] as Task;
  expect(savedTask.title).toBe('Invoice A');
  expect(savedTask.orgId).toBe('org-1');
  expect(savedTask.status).toBe('pending');

  // Assert — Layer 4: absence of WRONG side effects
  expect(mockNotifier.notify).not.toHaveBeenCalled(); // didn't email anyone yet
});
```

Four layers sounds like a lot, but each one catches a different class of bug:
- Layer 1 catches: "handler returns Fail when it should succeed"
- Layer 2 catches: "handler returns Ok but with wrong id / data"
- Layer 3 catches: "handler succeeds but persists the wrong object"
- Layer 4 catches: "handler accidentally triggers an unrelated side effect"

### The layered-assert template for error paths (4 layers)

Error paths need FOUR layers. All four are MANDATORY when the code provides enough information to verify them:

```ts
it('returns 404 with AFTER_ID_NOT_FOUND code when afterId is not in the list', async () => {
  // Arrange
  const orderedList = [taskA, taskB, taskC];
  mockRepo.findById.mockResolvedValue(taskA);
  mockRepo.findByStatusIdOrdered.mockResolvedValue(orderedList);

  // Act
  const command = new ReorderTaskCommand('id-a', 'non-existent-id', TEST_AGENCY_ID, TEST_ORG_ID);
  const result = await handler.execute(command);

  // Assert — Layer 1: failure type
  expect(result.isFailure).toBe(true);

  // Assert — Layer 2: HTTP status (correct semantic for the error class)
  expect(result.error.httpStatus).toBe(404);

  // Assert — Layer 3: specific error code (distinguishes this 404 from other 404s)
  expect(result.error.errorCode).toBe('OPERATIONS.TASK.AFTER_ID_NOT_FOUND');

  // Assert — Layer 4: short-circuit — downstream writes did NOT happen
  expect(mockRepo.save).not.toHaveBeenCalled();
});
```

### Why four layers, not two

- **Layer 1** catches: "handler returned success when it should have failed"
- **Layer 2** catches: "handler failed with the WRONG HTTP status (e.g. 500 instead of 404)"
- **Layer 3** catches: "two different errors collapsed into the same 404 — clients can't distinguish"
- **Layer 4** catches: "validation executed but the wrong flow ran anyway (data leaks, wasted I/O, double writes)"

Missing Layer 3 is the most common error-path weakness in this codebase. If your error code system exists (DaraMex uses strings like `'OPERATIONS.TASK.AFTER_ID_NOT_FOUND'`), verifying the specific code is required. Dropping this layer means two different errors producing the same HTTP status become indistinguishable in tests.

### Choosing the right HTTP status

| Code | Meaning | When |
|------|---------|------|
| 400 Bad Request | Request is malformed or contains invalid inputs | `afterId === id`, negative counts, malformed UUIDs |
| 404 Not Found | Request is well-formed but references something that doesn't exist | Resource id missing, afterId not in list |
| 409 Conflict | Request is valid but the current state prohibits it | Duplicate creation, stale update |
| 422 Unprocessable Entity | Request is structurally valid but violates business rules | Transfer exceeds balance, expired coupon |

Don't default to 400 for everything. The wrong code lies to your clients about the nature of the error.

### When a single assertion IS enough

There's one legitimate case: a pure function test where the return IS the full behavior.

```ts
it('returns true when age is 18 or older', () => {
  // Act
  const result = isAdult(18);

  // Assert
  expect(result).toBe(true);
});
```

No state, no side effects, no layers. Pure functions are simple by design — embrace it.

### The reflex to build

When writing a test, instinctively ask: **"What TWO or THREE things need to be true for this behavior to be correct?"** List them. That's your assert count. If you wrote only one, you probably missed something.

### Relation to regression tests

Every bug fix ships with a regression test. And regression tests are BORN from strong assertions: the test captures the EXACT condition that the bug violated. If your regression test's assert is weak, the same bug can return.

---

## 15. The bug-fix workflow (MANDATORY)

Every bug fix in DaraMex MUST follow this 4-step workflow. This is a **project-level convention**, not a style preference. The codebase owner set it explicitly.

### The workflow

1. **STOP — do not touch production code yet.** When a bug is reported or discovered, resist the urge to jump to the fix.

2. **Think: what test would FAIL right now because of this bug?**
   - Which input triggers the bug?
   - What behavior is wrong?
   - What use cases are affected by this bug?

3. **Write those tests FIRST.** Add them to the spec file.
   - Run them → they MUST FAIL (RED). This proves the bug is real and your test reproduces it.
   - If the test passes before fixing, your test does NOT reproduce the bug — rewrite it.

4. **Now fix the production code.**
   - Run the tests → they MUST PASS (GREEN).
   - Run the full affected suite → everything stays GREEN.

5. **Commit the test and the fix TOGETHER in the same PR.**
   - The test stays in the codebase forever as a regression guard.
   - Future developers (including you) cannot reintroduce the bug without CI catching it.

### Visual timeline

```
Bug discovered
      ↓
  [Write failing test] ←── RED  (proves the bug exists)
      ↓
  [Apply fix in prod]
      ↓
  [Test passes now]    ←── GREEN (proves the fix works)
      ↓
  [Commit fix + test]  ←── Regression test stays FOREVER
```

### Why this order, not the other

**Wrong order** ("fix first, test after"):
- You never confirm that your fix actually addresses the bug — the test might pass for a different reason.
- You may skip the test entirely because "it already works now".
- If the test you write after doesn't actually fail without the fix, it's a decorative test that catches nothing.

**Correct order** (this workflow):
- RED first PROVES the test is tied to the bug.
- GREEN after PROVES the fix resolves it.
- The test's value is demonstrated before it's accepted.

### Concrete example — MessageMapper.toDto() omitting `feedback`

**Bug**: After the user likes/dislikes an AI message, a refresh loses the feedback state. Root cause: `MessageMapper.toDto()` was not copying the `feedback` field from the entity to the DTO.

Step-by-step:

```ts
// Step 2-3: Write the failing test FIRST
it('includes the feedback field when mapping a Message to DTO', () => {
  // Arrange
  const message = Message.rehydrate({
    id: MESSAGE_ID,
    content: 'Hello',
    feedback: 'like',
    // ...other fields
  });

  // Act
  const dto = MessageMapper.toDto(message);

  // Assert
  expect(dto.feedback).toBe('like');
});
```

Run the test → FAILS (`expected 'like', received undefined`). Bug reproduced.

```ts
// Step 4: Fix MessageMapper.toDto()
return {
  id: message.id,
  content: message.content,
  feedback: message.feedback,  // ← this line was missing
  // ...other fields
};
```

Run the test → PASSES. Run the full mapper suite → still all GREEN. Commit.

Now, three months from now, if anyone refactors `MessageMapper.toDto()` and forgets `feedback`, CI blocks the PR.

### Connection to Strict TDD

This project has Strict TDD enabled. The bug-fix workflow IS the RED → GREEN → REFACTOR loop applied to bug fixes:
- **RED**: the test that reproduces the bug
- **GREEN**: the minimal fix that makes the test pass
- **REFACTOR** (optional): clean up the fix if it introduced duplication or unclear code

For features, Strict TDD uses the same loop; see the companion `tdd` skill for the feature-building variant.

### Checklist before committing a bug fix

- [ ] I wrote a test that reproduces the bug (it fails without my fix)
- [ ] My fix is minimal — doesn't expand scope beyond the bug
- [ ] All previously passing tests still pass (no regressions elsewhere)
- [ ] Test and fix are in the SAME commit / PR
- [ ] Test name describes the SPECIFIC behavior the bug violated (not "should not crash")
- [ ] If the bug touched a domain rule, there's at least one domain-level test covering it

If any checkbox is unchecked, do NOT merge. Go back and fill the gap.

---

## 16. Multi-tenancy testing — which layer owns the validation?

DaraMex is multi-tenant. There are multiple tenant boundaries (org, agency, user). When testing handlers, the first question is always: **who is responsible for validating tenancy?**

### Clean Architecture responsibility map

| Layer | Tenant responsibility | What to test |
|-------|----------------------|--------------|
| **Endpoint / decorator `@OrgId`** | Validate the user has permission over the requested tenant. Reject cross-tenant access. | Integration tests with `supertest`: 403 on cross-tenant requests, correct headers/cookies, etc. |
| **Guard / middleware** | Authenticate, enforce access rules. | Integration tests (included in controller specs). |
| **Handler (application)** | TRUST the inputs. Use the tenant IDs correctly when calling infra (queries, commands). | Unit tests: verify handler calls `repo.findByX(orgId, ...)` with the RIGHT orgId. Do NOT re-validate permissions here. |
| **Repository (infrastructure)** | Apply the tenant scope to the actual SQL / ORM query. | Integration tests: verify the query includes the scope. |

### Principle: test what the handler USES, not what it ignores

If the command carries `agencyId` but the handler doesn't destructure or use it, **there is nothing to assert about it**. Testing the absence of use is noise.

### Practical checklist for a handler test

Before claiming "multi-tenancy is tested", verify:

- [ ] Every tenant ID the handler USES is verified via `toHaveBeenCalledWith(...)` in at least one test.
- [ ] Tenant IDs the handler IGNORES (passes through the command constructor but doesn't destructure) are NOT asserted — they're someone else's responsibility.
- [ ] The WRONG tenant scope is verified NOT to leak: `expect(repo.findX).not.toHaveBeenCalledWith(OTHER_ORG_ID, ...)`.
- [ ] Permission validation is NOT tested at this layer — that's for the integration test of the endpoint.

### Common mistake: testing the wrong layer

```ts
// ❌ WRONG — handler test trying to verify access rules
it('rejects when user is from a different org', async () => {
  // ...
});
```

Handlers don't know about users or access rules. That logic lives in the decorator / guard. Testing it here is testing your MIDDLEWARE via the handler, which is a leaky abstraction.

The right place for that test: `infrastructure/http/<feature>.controller.spec.ts` with `supertest` and an authenticated (or un-authenticated) request.

### Testing scoping by INPUT, not OUTPUT

A weak scoping test verifies the output is right. A strong one verifies the INPUT to downstream calls:

```ts
// ❌ Weak — could pass even if handler queried the wrong scope
expect(result.isSuccess).toBe(true);
expect(savedTask.sortIndex > taskB.sortIndex).toBe(true);

// ✅ Strong — directly verifies the query was scoped correctly
expect(mockRepo.findByStatusIdOrdered).toHaveBeenCalledTimes(1);
expect(mockRepo.findByStatusIdOrdered).toHaveBeenCalledWith(TEST_ORG_ID, 'status-1');
expect(mockRepo.findAll).not.toHaveBeenCalled();
```

The first assertion set is satisfied by any correct-looking output. The second directly proves the handler queried only the correct tenant scope, not just "by coincidence the rank was right".

---

## 17. Test comments are code — maintain or delete

Comments inside tests are DOCUMENTATION. When you refactor test code, you refactor the comments too. A stale comment is WORSE than no comment because it lies to the next reader (often you, in 6 months).

### Classic stale-comment trap

```ts
// rank should be > taskB (last in status-1), not influenced by taskD/taskE (status-2)
expect(savedTask.sortIndex > taskB.sortIndex).toBe(true);
```

If `taskD` and `taskE` were removed from the file during a refactor but this comment stayed, the test now mentions fixtures that don't exist. A reader looking at this comment may search the file for `taskD` to understand the scenario — and find nothing.

### Rule

- When you delete a fixture, grep the whole spec file for its name. Update or delete any comment that references it.
- When you change the scenario a test covers, rewrite the comment to match.
- When in doubt: **delete the comment**. Clear code + clear test name is better than any stale comment.
- Treat comments as first-class code in test files, same as production.

### Useful comment patterns in tests

Comments DO belong in tests when they:

- Explain the SCENARIO being simulated (`// Move C after A — expected between A and B`)
- Explain an unusual choice (`// statusId is the tenant boundary here, not orgId`)
- Explain why a specific value matters (`// 'd00000' ensures lexicographic ordering without leading zeros`)

Comments DO NOT belong in tests when they:

- Describe what the next line of code does (code is self-explanatory)
- Repeat the test name (redundant)
- Reference variables or code that no longer exist (stale)

---

## 18. Distinguishing `not.toHaveBeenCalled()` from `not.toHaveBeenCalledWith(...)`

Both are negative assertions but they protect different invariants.

### `not.toHaveBeenCalled()`
"This function was NEVER called, with ANY argument."

Use when:
- Validation should short-circuit BEFORE any call to the mock.
- The invariant is "no work was performed".

```ts
// Domain validation rejected the input; infra was never touched
expect(mockRepo.save).not.toHaveBeenCalled();
```

### `not.toHaveBeenCalledWith(X, Y)`
"This function may or may not have been called, but NOT with these specific arguments."

Use when:
- The function IS expected to be called, but only with certain arguments.
- You want to assert tenant isolation or contract boundaries.

```ts
// findByStatusIdOrdered WAS called (for status-1), but MUST NOT be called for status-2
expect(mockRepo.findByStatusIdOrdered).toHaveBeenCalledWith(TEST_ORG_ID, 'status-1');
expect(mockRepo.findByStatusIdOrdered).not.toHaveBeenCalledWith(TEST_ORG_ID, 'status-2');
```

### Common mistake
Using `not.toHaveBeenCalled()` when you should use `not.toHaveBeenCalledWith(...)`. The former will FAIL if the function is called at all; the latter allows legitimate calls while forbidding specific bad ones.

Always pick the form that expresses exactly the invariant you want to protect — no more, no less.

---

## 19. Mutable shared fixtures — the silent test pollution

One of the hardest test pollution bugs to find: fixtures declared at `describe` scope that get mutated by the code under test. The tests APPEAR to pass, but they depend on execution order. Adding or removing a single test can break seemingly unrelated tests.

### The trap

```ts
describe('ReorderTaskHandler', () => {
  // ❌ Fixtures created ONCE at describe scope
  const taskA = makeTask('id-a', 'd00000');
  const taskB = makeTask('id-b', 'm00000');
  const taskC = makeTask('id-c', 't00000');

  beforeEach(() => {
    mockRepo = { ... }; // only mocks are fresh
    handler = new ReorderTaskHandler(mockRepo);
    // taskA, taskB, taskC keep any mutations from the previous test
  });

  // Test 1 reorders taskB → mutates taskB.sortIndex
  // Test 2 reorders taskA → mutates taskA.sortIndex
  // Test 3 uses taskA or taskB → now with mutated state, test FAILS mysteriously
});
```

### Why it's silent

The handler calls `task.reorder(rank)`, and `Task.reorder()` mutates the entity (`this._props.sortIndex = rank`). Each test leaves behind a different sortIndex on whichever task it reordered. When the next test reads the "fixed" `taskB.sortIndex`, it's actually reading a value set by the previous test.

The tests pass or fail depending on **execution order**. Adding a new test can reveal the pollution (as happened in DaraMex when a new "between" test was added to the LexoRank integration suite).

### The fix — fresh fixtures in `beforeEach`

```ts
describe('ReorderTaskHandler', () => {
  let taskA: Task;
  let taskB: Task;
  let taskC: Task;

  beforeEach(() => {
    // Fresh fixtures per test — immune to mutations from previous tests
    taskA = makeTask('id-a', 'd00000');
    taskB = makeTask('id-b', 'm00000');
    taskC = makeTask('id-c', 't00000');

    mockRepo = { ... };
    handler = new ReorderTaskHandler(mockRepo);
  });
});
```

### The smell test

Any fixture whose class has a method that mutates state (setter, `reorder()`, `markAsDone()`, any method returning `void` that changes the entity) MUST be recreated per test. If in doubt, recreate.

### Detection strategy

Run the test file in random order to expose order dependencies:

```bash
# Jest
pnpm exec jest --testPathPatterns="your-file" --randomize

# Vitest
pnpm vitest --sequence.shuffle
```

If tests pass in sorted order but fail in random order → you have shared mutable state. Fix by moving fixtures into `beforeEach`.

### Related architectural concern — domain mutability

If your entity has methods that mutate in place (`task.reorder(rank)` returning `void`), you inherit the test pollution problem at every test site. A cleaner DDD design returns a new entity:

```ts
// Mutable (current DaraMex style)
task.reorder(rank); // void — mutates in place

// Immutable (DDD-strict style)
const reordered = task.reorder(rank); // returns a new Task
```

Immutable entities eliminate this class of test pollution entirely. They're a larger architectural shift, but consider this pattern when designing new domain aggregates.

### Rule

- Any fixture used across multiple tests where the code under test MIGHT mutate it → `beforeEach`.
- Plain value objects with no mutating methods → safe at describe scope.
- When unsure → `beforeEach`. The cost is negligible.

---

## 20. Defensive entity invariant tests — where TypeScript ends, runtime begins

Type-level constraints (`Omit<...>`, optional fields, `readonly`) are NOT security. They're development-time hints. A cast (`as`, `as unknown as`) silently bypasses them. An entity that holds tenant-critical or identity-critical invariants MUST enforce them at runtime too.

### The pattern — tenant immutability in DaraMex entities

Any field that should be immutable after creation (tenant IDs, creator ID, created timestamps, unique business keys) gets a **defensive test** AND a **runtime guard**.

### Writing the defensive tests FIRST (Strict TDD)

Before writing or modifying the entity, write tests that WILL fail if the entity doesn't enforce the invariant:

```ts
describe('Task entity › update() — immutable tenant invariants (defensive)', () => {
  it('does not modify orgId when update() is called with a type cast bypass', () => {
    // Arrange
    const task = Task.new(makeTaskProps({ orgId: TEST_ORG_ID }));
    const originalOrgId = task.orgId;

    // Act — simulate a caller bypassing TypeScript with a cast
    task.update({ orgId: 'hacker-org-id' } as unknown as TaskUpdateProps);

    // Assert — orgId must remain unchanged
    expect(task.orgId).toBe(originalOrgId);
  });

  // Repeat for agencyId, creatorId, and any other tenant/identity-critical field
});
```

If any of these tests FAIL → real bug. Apply the runtime guard fix.

### The runtime guard pattern for BaseEntity subclasses

`BaseEntity` provides `getUpdatableKeys()` as an extension point explicitly for this. Subclasses override it to EXCLUDE immutable fields:

```ts
export class Task extends BaseEntity<TaskProps, TaskUpdateProps> {
  /**
   * Tenant-critical fields that must NEVER change after creation.
   * Enforced at runtime via getUpdatableKeys() — complements the TypeScript-level
   * exclusion in TaskUpdateProps against accidental or malicious type casts.
   */
  private static readonly IMMUTABLE_KEYS: readonly (keyof TaskProps)[] = [
    'orgId',
    'agencyId',
    'creatorId',
  ] as const;

  protected override getUpdatableKeys(): (keyof TaskProps)[] {
    return super.getUpdatableKeys().filter(
      (k) => !(Task.IMMUTABLE_KEYS as readonly string[]).includes(k as string),
    );
  }
}
```

### Checklist for ANY BaseEntity subclass in DaraMex

- [ ] Does the entity have tenant fields (`orgId`, `agencyId`)? → They MUST be in `IMMUTABLE_KEYS`
- [ ] Does the entity have an owner / creator field? → It MUST be in `IMMUTABLE_KEYS`
- [ ] Does the entity expose an update path? → It MUST override `getUpdatableKeys()` if there are any immutable fields
- [ ] Are there defensive tests using `as unknown as UpdateProps` casts? → Write them. If they pass, great. If they fail, you just found a bug.

### Audit recipe for the existing codebase

```bash
# Find entities that extend BaseEntity
rg -l "extends BaseEntity" apps/api/src

# Check which ones override getUpdatableKeys
rg -l "override getUpdatableKeys" apps/api/src

# The difference is your audit backlog
```

The gap size tells you the severity. If many entities extend BaseEntity and few override, the category of bug is widespread and should be treated as a systematic security sweep, not per-entity fixes.

### Why this fix was minimal (TDD principle)

The tests covered 3 fields: `orgId`, `agencyId`, `creatorId`. The fix excluded exactly those 3, even though `TaskUpdateProps` type-level also excludes `sortIndex`, `status`, `tags`, and others.

**Rule**: extend the fix only to what tests cover. If more fields need protecting, write more defensive tests first. This keeps the fix honest — every line of the fix is justified by a RED test.

---

## 21. Rich domain model — moving invariants INTO the entity

An "anemic" domain entity is one that has getters and simple mutators but no business rules. The rules live elsewhere (Zod schemas, handlers, "nowhere"). This defeats the purpose of having a domain model — you pay the cost of entities (mapping, rehydration) without the benefit (encapsulated invariants).

A **rich domain entity** encodes its own rules: a Task knows a parent can't be itself; an Order knows it can't be completed with zero items; a User knows an email must be syntactically valid.

### The three layers of enforcement in a rich entity

1. **Creation** — `Entity.new()` validates. Invalid data cannot be constructed.
2. **Mutation** — `entity.update()` validates the NEW state. The entity never exists in an invalid state, even transiently.
3. **Transitions** — domain methods (`start()`, `finish()`, `cancel()`, `approve()`) encapsulate state changes with their own preconditions.

### The `validateProps` + validate-before-apply pattern

```ts
export class Task extends BaseEntity<TaskProps, TaskUpdateProps> {
  static new(props: TaskProps): Task {
    const normalized = {
      ...props,
      relatedUserIds: props.relatedUserIds ?? [],
      // ... other defaults
    };
    // Fail fast — validate BEFORE construction. Atomic: invalid data never becomes an instance.
    Task.validateProps(normalized);
    return new Task(normalized);
  }

  update(props: TaskUpdateProps): this {
    const { relatedUserIds, tagIds, attachmentIds, ...restProps } = props;

    // Build the PROPOSED state and validate BEFORE applying any mutation.
    const proposedProps: TaskProps = {
      ...this._props,
      ...restProps,
      ...(relatedUserIds !== undefined && {
        relatedUserIds: [...new Set(relatedUserIds)],
      }),
      // ... other dedups
    };
    Task.validateProps(proposedProps, this.id);

    // Now apply — safe because we know the result is valid.
    super.update(restProps);
    // ... apply array dedups
    this.touch();
    return this;
  }

  private static validateProps(props: TaskProps, id?: string): void {
    if (id !== undefined && props.parentId != null && props.parentId === id) {
      throw new Error('Task cannot be its own parent (parentId === id)');
    }
    if (props.sortIndex === '') {
      throw new Error('Task sortIndex cannot be empty');
    }
    // ... more rules
  }
}
```

### Why validate-before-apply (atomicity)

Compare with the naive "mutate first, validate after" pattern:

```ts
// ❌ BAD — invalid transient state
update(props): this {
  super.update(props);              // mutates immediately
  Task.validateInvariants(this);    // if this throws, entity is already corrupted
  this.touch();
  return this;
}
```

If `validateInvariants` throws, the entity is left in an inconsistent state even though the caller sees an exception. Callers who catch and continue work with broken data. `validate-before-apply` guarantees that either the mutation succeeds fully OR the entity stays untouched.

### Domain state transitions — `start()`, `finish()`, etc.

When an entity has lifecycle states (started → finished, pending → approved, etc.), express each transition as a method with its own preconditions:

```ts
start(at: Date): void {
  if (this._props.startedAt !== null) {
    throw new Error('Task has already been started');
  }
  this._props.startedAt = at;
  this.touch();
}

finish(at: Date): void {
  if (this._props.startedAt === null) {
    throw new Error('Task cannot be finished before it is started');
  }
  if (at.getTime() < this._props.startedAt.getTime()) {
    throw new Error('Task finish date cannot be before started date');
  }
  this._props.finishedAt = at;
  this.touch();
}
```

Callers at the application layer say `task.start(now)` and `task.finish(now)`, expressing business INTENT rather than setting fields. The entity enforces the rules.

### `rehydrate()` bypass — pragmatic trade-off

`Task.rehydrate()` (inherited from `BaseEntity`) does NOT call `validateProps`. This is intentional:

- **Why bypass**: allows loading legacy data from the DB that may pre-date a new invariant. Without bypass, adding a rule like "description cannot be empty" would break every read of an existing bad row.
- **Trade-off**: the DB is trusted. If bad data exists, it can be loaded and inspected.
- **Alternative**: strict mode that validates even on rehydrate — opt-in via flag. Consider if data migration is finished and strictness is desired.

### Testing pattern for rich domain entities

For each invariant, write tests in TWO places (minimum):
1. `Task.new()` rejects invalid input
2. `task.update()` rejects invalid mutation

For state transitions, test:
1. The happy path (transition works)
2. The precondition failure (transition rejected when invalid)
3. The side effects (touch, state flags)

Example structure:

```ts
describe('invariant: parentId cannot equal own id', () => {
  it('throws when update() attempts to set parentId equal to own id', () => {
    const task = Task.new(makeTaskProps());
    expect(() => task.update({ parentId: task.id })).toThrow(/own parent/i);
  });
});

describe('start() — domain state transition', () => {
  it('sets startedAt to the given date', () => { /* happy path */ });
  it('throws when called on a task that has already been started', () => { /* precondition */ });
  it('updates updatedAt via touch', () => { /* side effect */ });
});
```

### Detecting anemic models in code review

Red flags that suggest your entity is anemic:

- The entity has only getters (no behavior)
- Business rules live in `handlers/` or in Zod schemas only
- You can construct an entity with obviously-invalid data without any error
- The entity's constructor does no work beyond assigning props
- State transitions happen via direct field mutation from handlers (`task.startedAt = new Date()`)

If you see these, the domain is anemic. Plan to migrate it using the `validateProps` + transition-method pattern above.
