# Real-World Anti-Patterns Found in DaraMex

**Source**: Testing audit run on 2026-04-12 against `apps/api/` and `apps/panel/`.

These are **anti-patterns actually present in this codebase** (not hypothetical). Counts reflect what was measured. Use this file as a priority list when auditing or refactoring tests.

---

## The Big Picture — what the audit revealed

| Metric | Value | Meaning |
|--------|-------|---------|
| API test suites broken by TS errors | **129 / 187 (69%)** | Tests abandoned during refactors |
| API tests actually running | 601 (594 pass, 7 fail) | The 31% that compiles is mostly healthy |
| Panel tests | 317 (316 pass, 1 fail) | Panel is in good shape |
| `as any` occurrences | **186** | Mock type-safety collapsed |
| `toHaveBeenCalled()` without args | 192 lines across 78 files | Mock Hell in ~42% of specs |
| Unmocked `new Date()` in specs | 64 | Flakiness risk |
| `toBeDefined()` existence tests | 62 | Tests that catch nothing |
| Vague test names | 49 | Hard to know what's protected |
| `fireEvent.` in panel (instead of `userEvent`) | 12 | Legacy pattern, migrate |
| `toHaveClass(` CSS coupling | **0** | ✅ Good discipline here |

---

## THE META-LESSON — "A test that doesn't compile is not a test"

This is the biggest finding. Over two-thirds of API test suites don't compile because the code they test was refactored and the tests were left behind. Symptoms:

- `Cannot find module '@repo/schemas'` — package renamed
- `Type 'string' is not assignable to type 'BilingualMessage'` — domain added i18n, tests still pass strings
- `Type 'MongoAbility<AbilityTuple>' is not assignable to 'AppAbility'` — CASL types tightened

### Why this happened

Tests were treated as **write-once artifacts**. When you refactored production code, you updated the production code AND the consumers — but tests were "below the line" and got skipped. The CI probably doesn't fail on TS errors in tests, so they silently rot.

### Before fixing — diagnose the cause

A test suite that fails to run has THREE possible causes, and the fix is different for each. Never skip diagnosis.

| Cause | Symptom | Fix |
|-------|---------|-----|
| **Test rotten** | TS errors about types that shifted — the test imports old shapes | Update the test to match current shapes |
| **Production broken** | Type errors in production code too | Fix the production code. Do NOT patch with `as any` in the test |
| **Build/workspace incomplete** | `Cannot find module '@repo/xxx'` despite correct import paths | Run the workspace build. Do NOT modify the test |

**Reflex before touching anything:**
1. Does the import path exist in the codebase (literally)?
2. Does the imported type still have the shape the test expects?
3. Is the package actually BUILT?

Answer all three before editing. The overwhelming majority of "broken tests" in this codebase are cause #1 or #3 — the test itself is NOT the problem.

### How to prevent it

1. **CI MUST fail when tests don't compile.** Non-negotiable. If it doesn't today, fix this first.
2. **When you refactor a type/interface/schema, run the affected tests immediately** — not "later".
3. **Treat a rotten test like a rotten production bug.** If you can't fix it in the same PR as the refactor, DELETE it and re-add it properly. A deleted test is honest. A rotten test is a lie.
4. **Never land a PR that leaves a suite in broken-compilation state.** Even if the test doesn't logically break, if it can't compile the CI is lying to you.

---

## Anti-Pattern #1 — `as any` in mocks (186 occurrences)

### 🚫 What it looks like in this codebase

```ts
const repo = { save: jest.fn() } as any
const gateway = { send: jest.fn() } as unknown as IEmailGateway
const handler = new CreateTaskHandler(repo as any, logger as any)
```

### Why it's wrong

- **Port contract drift becomes invisible.** If you add `findByUserId` to `ITaskRepository`, every `as any` mock silently keeps working even though they're now missing a method.
- **Argument types are lost.** `repo.save.mockResolvedValue('lol')` doesn't complain — but the real port returns `Result<void>`.
- **Refactors become dangerous.** The type-system advantage you paid for with Clean/Hexagonal is ZERO if your tests cast it away.

### ✅ Correct fix

```ts
let repo: jest.Mocked<ITaskRepository>

beforeEach(() => {
  repo = {
    save: jest.fn(),
    findById: jest.fn(),
    findByOrgId: jest.fn(),
  } satisfies jest.Mocked<ITaskRepository>
})
```

Or for sparse mocks where you only need one method:

```ts
const repo: Partial<jest.Mocked<ITaskRepository>> = {
  save: jest.fn(),
}
```

### When it's acceptable

Practically never. If you have to reach for `as any`, the port design is probably bad — too many responsibilities, or wrong abstraction. Split the port first.

---

## Anti-Pattern #2 — `toHaveBeenCalled()` without arguments (192 lines in 78 files — "Mock Hell")

### 🚫 What it looks like

```ts
await handler.execute({ title: 'test', orgId: 'org-1' })

expect(repo.save).toHaveBeenCalled()
expect(logger.log).toHaveBeenCalled()
expect(notifier.notify).toHaveBeenCalled()
```

### Why it's wrong

This verifies **coupling**, not **behavior**. The assertions say "these functions were called at some point with some arguments". You could:
- Pass an empty object to `repo.save` → test passes
- Pass the wrong `orgId` → test passes
- Call the logger for a completely different event → test passes

You're not testing that CreateTaskHandler works. You're testing that CreateTaskHandler calls its dependencies, which any child of the class does by existing.

### ✅ Correct fix

Verify **arguments** and **return values**:

```ts
const result = await handler.execute({ title: 'Invoice A', orgId: 'org-1' })

// 1. Assert the return value (the handler's contract)
expect(result.isOk()).toBe(true)
const taskId = result.unwrap()

// 2. Assert the mock was called with the RIGHT arguments
expect(repo.save).toHaveBeenCalledTimes(1)
const savedTask = repo.save.mock.calls[0][0]
expect(savedTask.title).toBe('Invoice A')
expect(savedTask.orgId).toBe('org-1')
expect(savedTask.id).toBe(taskId)
expect(savedTask.status).toBe('pending')
```

### Useful Jest matchers instead of the empty call

| Instead of | Use |
|------------|-----|
| `toHaveBeenCalled()` | `toHaveBeenCalledWith(...)` or `toHaveBeenCalledTimes(n)` |
| `repo.save.mock.calls[0]` | `expect(repo.save).toHaveBeenCalledWith(expect.objectContaining({...}))` |
| Multiple individual asserts | `expect.objectContaining({ title: 'x', status: 'pending' })` |

### When `toHaveBeenCalled()` alone IS OK

Only when you explicitly want to verify a side effect **didn't** happen:

```ts
expect(repo.save).not.toHaveBeenCalled()
```

This asserts the domain validation kicked in BEFORE reaching infra. That's a real business rule worth protecting.

---

## Anti-Pattern #3 — Unmocked `new Date()` (64 occurrences)

### 🚫 What it looks like

```ts
const task = Task.create({ title: 'x', createdAt: new Date() })
// ...
expect(task.createdAt).toEqual(new Date())
// ↑ This might fail if the two `new Date()` are a millisecond apart
```

Or worse:

```ts
const handler = new MarkOverdueHandler(repo, now)
await handler.execute()
expect(repo.updated).toBeGreaterThan(Date.now() - 1000)
// ↑ Flaky — passes locally, fails in CI when the machine is slow
```

### Why it's wrong

Fails **FIRST-Repeatable**. Tests must produce the same result every run. A test that depends on the current time is a test that will **eventually fail for reasons unrelated to your code** — usually at 3am when someone's on-call.

### ✅ Correct fix — two strategies

**Strategy A — Fake timers (when testing time-dependent logic)**:

```ts
beforeEach(() => {
  jest.useFakeTimers()
  jest.setSystemTime(new Date('2026-01-15T10:00:00Z'))
})

afterEach(() => {
  jest.useRealTimers()
})

it('...', () => {
  const task = Task.create({ title: 'x' })
  expect(task.createdAt).toEqual(new Date('2026-01-15T10:00:00Z'))
})
```

**Strategy B — Clock as a dependency (better for domain)**:

Design the domain to accept `now` as a parameter or inject a `Clock` service:

```ts
// Domain
class Task {
  static create(data: { title: string }, now: Date): Task {
    return new Task({ ...data, createdAt: now })
  }
}

// Test
const NOW = new Date('2026-01-15T10:00:00Z')
const task = Task.create({ title: 'x' }, NOW)
expect(task.createdAt).toEqual(NOW)
```

**Strategy B is superior** because:
- Forces you to think about time explicitly (side-effect becomes parameter)
- Easier to reason about
- Doesn't pollute the test globals

---

## Anti-Pattern #4 — `toBeDefined()` existence tests (62 occurrences)

### 🚫 What it looks like

```ts
it('should have a createTask method', () => {
  expect(handler.execute).toBeDefined()
})

it('should instantiate', () => {
  const handler = new CreateTaskHandler(repo)
  expect(handler).toBeDefined()
})
```

### Why it's wrong

**TypeScript already guarantees this.** If `handler.execute` didn't exist, your code wouldn't compile. Testing that the method exists catches nothing — you could delete the method's body and the test still passes.

### ✅ Correct fix

**Delete the test.** If you want to verify behavior, test what the method DOES, not that it EXISTS:

```ts
// Before: existence test (useless)
expect(handler.execute).toBeDefined()

// After: behavior test (useful)
const result = await handler.execute({ title: 'x', orgId: 'org-1' })
expect(result.isOk()).toBe(true)
```

### When `toBeDefined()` IS acceptable

Only for things that CAN legitimately be undefined:

```ts
const user = await repo.findById('non-existent')
expect(user).toBeUndefined()  // meaningful — tests the "not found" case
```

Or:

```ts
const result = await handler.execute(input)
const dto = result.unwrap()
expect(dto.optionalField).toBeDefined() // meaningful if optionalField is `string | undefined`
```

The rule: `toBeDefined` is only useful when **undefined was a real possibility**.

---

## Anti-Pattern #5 — Vague test names (49 occurrences)

### 🚫 What it looks like

```ts
it('should create a task', async () => { ... })
it('should work', async () => { ... })
it('should render', () => { ... })
it('should return true', () => { ... })
```

### Why it's wrong

Test names are **documentation**. When they fail, the first signal you get is the name. "should create a task" tells you nothing about which rule was violated.

A good test name answers: **"What rule or behavior does this protect?"**

### ✅ Correct fix — name pattern

Use the structure: **"[verb] [expected behavior] [when / under what condition]"**

| 🚫 Before | ✅ After |
|-----------|----------|
| `should create a task` | `persists a new Task with pending status and returns its id` |
| `should work` | `converts BilingualMessage to English when locale is "en"` |
| `should render` | `shows an empty state message when the tasks array is empty` |
| `should return true` | `returns true when the user has the required permission` |
| `should fail` | `returns Result.fail with DomainError when title is empty` |

### Describe + it combination

Structure them so reading `describe` + `it` together forms a sentence:

```ts
describe('CreateTaskHandler', () => {
  describe('when input is valid', () => {
    it('persists the task and returns its id', () => {...})
    it('emits a TaskCreated domain event', () => {...})
  })

  describe('when title is empty', () => {
    it('returns Result.fail with a DomainError', () => {...})
    it('does not touch the repository', () => {...})
  })
})
```

Reading this: "CreateTaskHandler, when input is valid, persists the task and returns its id". That's readable documentation.

---

## Anti-Pattern #6 — `fireEvent.` instead of `userEvent` (12 occurrences in panel)

### 🚫 What it looks like

```ts
import { fireEvent } from '@testing-library/react'

fireEvent.click(button)
fireEvent.change(input, { target: { value: 'hello' } })
```

### Why it's wrong

`fireEvent` dispatches a single synthetic event. Real users don't do that:
- Clicking involves `mouseDown`, `mouseUp`, `click`, plus focus
- Typing fires `keyDown`, `keyPress`, `input`, `keyUp` per character
- Hovering includes `mouseEnter`, `mouseMove`

Components that rely on these sequences (most modern UI libraries, focus management, compound components) will misbehave under `fireEvent` but pass tests. You'll ship bugs.

### ✅ Correct fix

```ts
import userEvent from '@testing-library/user-event'

const user = userEvent.setup()

await user.click(button)
await user.type(input, 'hello')
await user.hover(tooltip)
await user.keyboard('{Escape}')
```

### Migration checklist

- `fireEvent.click(x)` → `await user.click(x)`
- `fireEvent.change(input, { target: { value: 'v' } })` → `await user.clear(input); await user.type(input, 'v')`
- `fireEvent.submit(form)` → `await user.click(submitButton)` (simulates a real user)
- Always `await` the user action
- Always call `userEvent.setup()` once per test (or in `beforeEach`)

### When `fireEvent` IS OK

When you explicitly need to test a single low-level event that `userEvent` doesn't provide:

```ts
fireEvent.scroll(container, { target: { scrollY: 500 } })
```

These cases are rare.

---

## Anti-Pattern #7 — Tests abandoned during refactors (129 / 187 suites = 69%)

### 🚫 What it looks like

A PR refactors `apps/api/src/shared/errors/app-error.ts` to accept `BilingualMessage` instead of `string`. Production code gets updated. Tests don't:

```ts
// test/modules/operations/.../handler.spec.ts — BROKEN, doesn't compile
Result.fail({ errorCode: 'STORAGE.DELETE_FAILED', message: 'Error' })
//                                                           ^^^^^^
// Type 'string' is not assignable to type 'BilingualMessage'.
```

The test passes in the IDE-author's head. The CI may not even run TS checks on the test directory. The test rots silently.

### Why it's the most dangerous anti-pattern

- You have **no coverage** on that code path, but the test file LOOKS like you do.
- When the next refactor lands, the broken test will be "fixed" by someone who doesn't remember the original intent.
- The fix usually ends up being a loosening: `as any`, `@ts-ignore`, removing assertions.

### ✅ Correct fix — process, not code

1. **Add a CI gate**: running `pnpm --filter api test` must fail if any suite fails to compile. If the current CI doesn't do this, fix that BEFORE anything else.

2. **When refactoring a type/interface/schema**, the sequence is:
   ```
   (a) Change the production type
   (b) TypeScript errors appear in tests → FIX THEM IN THE SAME PR
   (c) Re-run the affected test suites → they pass
   (d) Only then merge
   ```

3. **When a test can't be salvaged in the refactor PR**:
   - Delete it. Leave a comment in the PR: "Deleted `foo.spec.ts` — no longer applicable to new shape; replacement TODO in issue #NNN".
   - Open an issue for a replacement.
   - A deleted test is HONEST. A rotten test is a LIE.

### Fixing this codebase's 129 broken suites

Strategy (for a future phase, not now):

1. Run `pnpm --filter api test 2>&1 | grep "FAIL test/"` to list the broken suites
2. Group them by error pattern:
   - Missing module → fix imports first
   - Type tightening (BilingualMessage, CASL) → wrap/convert in mocks
   - Renamed APIs → search-replace in tests
3. Pick one pattern, fix ALL suites sharing it in one PR
4. Repeat until green
5. Add the CI gate so this can't happen again

---

## Priority order for remediation

When auditing or fixing tests in this codebase, tackle anti-patterns in this order:

| Priority | Anti-pattern | Why this order |
|----------|-------------|----------------|
| 1 | Tests that don't compile | Blocks 69% of the suite — highest ROI |
| 2 | `as any` in mocks | Undermines all other test quality; blocking issue for refactors |
| 3 | `toHaveBeenCalled()` alone | Core assertion quality; fixes Mock Hell |
| 4 | Unmocked `new Date()` | Flakiness source |
| 5 | Vague names | Readability / debugging speed |
| 6 | `toBeDefined()` existence tests | Delete them, fast win |
| 7 | `fireEvent.` → `userEvent` | Low count (12), low urgency |

---

## Quick grep commands to audit these patterns

```bash
# Tests that don't compile (from api/ dir)
pnpm --filter api test 2>&1 | grep "Test suite failed to run" -A 5

# as any in specs
rg "as any" apps/api/test apps/panel/test

# Mock Hell
rg "\.toHaveBeenCalled\(\)" apps/api/test apps/panel/test

# Unmocked dates
rg "new Date\(\)" apps/api/test apps/panel/test

# Existence tests
rg "\.toBeDefined\(\)" apps/api/test apps/panel/test

# Vague names
rg "it\('should (create|work|render|return)" apps/api/test apps/panel/test

# Legacy fireEvent
rg "fireEvent\." apps/panel/test
```
