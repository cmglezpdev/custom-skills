# Testing Tooling Cheatsheet — Jest 30 & Vitest 4

A curated list of testing utilities most developers underuse. Organized by category. Every example follows the DaraMex `// Arrange / // Act / // Assert` convention.

Jest commands shown with `jest.X`. Vitest equivalents shown with `vi.X`. Most of the API is shared; differences are called out explicitly.

---

## 1. Parameterized tests — run the same structure with many inputs

Parameterized tests let you write one `it` block and drive it with a table of inputs and expected outputs. They eliminate copy-paste across test cases and make the intent of edge-case coverage obvious.

### `it.each` / `describe.each` (Jest + Vitest)

Runs the test once per row. The array is **spread** into positional arguments.

```ts
it.each([
  ['', 'title is required'],
  ['   ', 'title is required'],
  ['a'.repeat(256), 'title too long'],
])('rejects task creation when title is "%s"', async (title, expectedError) => {
  // Arrange
  const command = new CreateTaskCommand(title, TEST_AGENCY_ID);

  // Act
  const result = await handler.execute(command);

  // Assert
  expect(result.isFailure).toBe(true);
  expect(result.error.message).toContain(expectedError);
});
```

With objects (preferred when rows have many fields — avoids positional confusion):

```ts
it.each([
  { input: '', error: 'title is required' },
  { input: '   ', error: 'title is required' },
])('rejects when title is "$input"', async ({ input, error }) => {
  // Arrange
  const command = new CreateTaskCommand(input, TEST_AGENCY_ID);

  // Act
  const result = await handler.execute(command);

  // Assert
  expect(result.isFailure).toBe(true);
  expect(result.error.message).toContain(error);
});
```

### `it.for` / `describe.for` (Vitest only)

Vitest 4 added `.for` as an alternative to `.each`. Key difference: **arrays are NOT spread** — the entire row is passed as a single argument and must be destructured in the callback. This also provides access to `TestContext` as the second argument (required for concurrent snapshots).

```ts
// Panel (Vitest only)
it.for([
  ['pending', false],
  ['done', true],
  ['archived', true],
])('shows completion badge when status is %s', ([status, expectBadge]) => {
  // Arrange
  render(<TaskCard status={status} />);

  // Act — implicit render

  // Assert
  const badge = screen.queryByRole('img', { name: /completed/i });
  expectBadge
    ? expect(badge).toBeInTheDocument()
    : expect(badge).not.toBeInTheDocument();
});
```

Concurrent snapshot variant (the primary reason `.for` exists):

```ts
test.concurrent.for([
  [1, 1, 2],
  [1, 2, 3],
])('add(%i, %i) -> %i', ([a, b, expected], { expect }) => {
  // Arrange — data from table

  // Act
  const result = add(a, b);

  // Assert
  expect(result).toBe(expected); // uses TestContext expect for concurrent safety
});
```

### Tagged template literal form (rare but elegant)

Jest supports a template-literal table syntax for `it.each`. Use it when aligning columns makes the test data obviously readable:

```ts
it.each`
  a    | b    | expected
  ${1} | ${1} | ${2}
  ${1} | ${2} | ${3}
`('$a + $b equals $expected', ({ a, b, expected }) => {
  // Arrange — data from table

  // Act
  const result = add(a, b);

  // Assert
  expect(result).toBe(expected);
});
```

### When to use

- Same assertion logic, different input/output combinations.
- Boundary conditions: empty string, max length, negative values, zero.
- Enum-driven behavior: every status, every role.

### When NOT to use

- Different assertions per case — split into separate `it` blocks.
- When each case needs a unique Arrange phase (different mocks per case).
- When test names would all be identical (template interpolation is required).

---

## 2. Lifecycle hooks — set up and tear down

Hooks organize shared setup and teardown. They also define a clear ownership boundary: what is shared vs what is per-test.

### `beforeEach` / `afterEach`

Run before/after **every** `it` in the nearest enclosing `describe`. Use for fresh instances, state reset, or per-test mocks.

```ts
describe('CreateTaskHandler', () => {
  let mockRepo: jest.Mocked<ITaskRepository>;
  let handler: CreateTaskHandler;

  beforeEach(() => {
    // Arrange (shared)
    mockRepo = {
      save: jest.fn(),
      findById: jest.fn(),
    } as jest.Mocked<ITaskRepository>;

    handler = new CreateTaskHandler(mockRepo);
  });

  it('saves the task on valid input', async () => {
    // Arrange
    mockRepo.save.mockResolvedValue(undefined);
    const command = new CreateTaskCommand('Fix login bug', TEST_AGENCY_ID);

    // Act
    const result = await handler.execute(command);

    // Assert
    expect(result.isSuccess).toBe(true);
  });
});
```

### `beforeAll` / `afterAll`

Run once per `describe` block. Use ONLY for expensive, side-effect-free setup that is safe to share across all tests in the suite.

```ts
describe('TaskRepository (integration)', () => {
  let app: INestApplication;
  let repo: TaskRepository;

  beforeAll(async () => {
    // Arrange (one-time boot — expensive)
    const module = await Test.createTestingModule({
      imports: [TypeOrmModule.forFeature([TaskEntity])],
      providers: [TaskRepository],
    }).compile();

    app = module.createNestApplication();
    await app.init();
    repo = module.get(TaskRepository);
  });

  afterAll(async () => {
    await app.close();
  });

  it('persists a new task', async () => {
    // Arrange
    const task = Task.create({ title: 'Refactor auth', agencyId: TEST_AGENCY_ID }).value;

    // Act
    await repo.save(task);
    const found = await repo.findById(task.id);

    // Assert
    expect(found?.title).toBe('Refactor auth');
  });
});
```

### Nested describe blocks — execution order

Hooks compose with nesting. Outer hooks run before inner hooks:

```
beforeAll (outer)
  beforeEach (outer)
    beforeEach (inner)
      it
    afterEach (inner)
  afterEach (outer)
afterAll (outer)
```

Use nested `describe` + `beforeEach` to set up common state for a sub-group without polluting sibling groups.

### Rule: `beforeAll` vs `beforeEach`

| Need | Use |
|------|-----|
| Fresh state per test (mocks, objects) | `beforeEach` |
| Expensive one-time setup (DB, app boot, HTTP server) | `beforeAll` |
| Cleanup of side effects (truncate DB table) | `afterEach` |
| Shutdown (close connections, HTTP server) | `afterAll` |

**Never** use `beforeAll` for mutable shared state (counters, mock call history). Tests will bleed into each other.

---

## 3. Mocks and spies

Mocks replace real dependencies. Spies wrap real implementations for observation. Know the difference — misusing them is the single biggest source of test rot in DaraMex.

### `jest.fn()` / `vi.fn()` — create a mock function

Creates a function that does nothing by default (returns `undefined`). You control its behavior.

```ts
// Arrange
const save = jest.fn<Promise<void>, [Task]>();
save.mockResolvedValue(undefined);
```

### `jest.spyOn(obj, 'method')` / `vi.spyOn()` — wrap a REAL method

Wraps an existing method on an object. The original runs unless you override it. **Restore after use.**

```ts
// Arrange
const spy = jest.spyOn(console, 'error').mockImplementation(() => {});

// Act
service.doSomethingThatLogs();

// Assert
expect(spy).toHaveBeenCalledWith(expect.stringContaining('invalid input'));
spy.mockRestore();
```

### When to use each

- **`jest.fn()`**: you own the dependency (port/interface) — replace it entirely.
- **`jest.spyOn()`**: you do NOT own the object (third-party, global, real class) — observe and optionally override.

### Typed mocks — `jest.Mocked<T>` / `vi.Mocked<T>`

Makes every property of `T` a `jest.Mock`. Gives autocomplete on `.mockResolvedValue`, `.mock.calls`, etc.

```ts
// Arrange
const mockRepo: jest.Mocked<ITaskRepository> = {
  save: jest.fn(),
  findById: jest.fn(),
  findByStatusIdOrdered: jest.fn(),
  delete: jest.fn(),
};
```

NEVER use `as any`. It defeats TypeScript and hides refactor breaks.

### Partial typed mocks — `Partial<jest.Mocked<T>>`

Use when an interface has many methods but only some are exercised by the test.

```ts
const mockRepo = {
  save: jest.fn(),
} as jest.Mocked<ITaskRepository>; // cast only what you provide
```

### Return value configuration

```ts
// Single return value
mockRepo.findById.mockResolvedValue(task);

// Different value each call (first call → taskA, second call → taskB)
mockRepo.findById
  .mockResolvedValueOnce(taskA)
  .mockResolvedValueOnce(taskB);

// Rejection
mockRepo.save.mockRejectedValue(new Error('DB timeout'));

// Dynamic — based on input
mockRepo.findById.mockImplementation(async (id) =>
  id === task.id ? task : null
);
```

### Call inspection — the anti-Mock-Hell toolkit

```ts
// BAD — code smell, passes with any behavior
expect(mockRepo.save).toHaveBeenCalled();

// GOOD — verifies call count
expect(mockRepo.save).toHaveBeenCalledTimes(1);

// GOOD — verifies exact arguments
expect(mockRepo.save).toHaveBeenCalledWith(
  expect.objectContaining({ title: 'Fix login bug' })
);

// GOOD — verifies nth call specifically
expect(mockRepo.save).toHaveBeenNthCalledWith(1, task);

// GOOD — verifies last call
expect(mockRepo.notify).toHaveBeenLastCalledWith(userId, 'task.created');

// GOOD — assert NOT called (legit)
expect(mockRepo.delete).not.toHaveBeenCalled();

// Direct inspection — when mock matchers aren't expressive enough
const [savedTask] = mockRepo.save.mock.calls[0];
expect(savedTask.sortIndex).toBeLessThan(firstTask.sortIndex);
```

### Reset strategies

| Method | Clears call history | Removes implementation | Restores original |
|--------|---------------------|----------------------|-------------------|
| `mockClear()` | Yes | No | No |
| `mockReset()` | Yes | Yes | No |
| `mockRestore()` | Yes | Yes | Yes (spyOn only) |

Global equivalents (in `afterEach` or Jest config):

```ts
jest.clearAllMocks();   // clear call history on all mocks
jest.resetAllMocks();   // clear + remove implementations
jest.restoreAllMocks(); // restore all spyOn to originals
```

Configure in `jest.config.ts` to run automatically:

```ts
// jest.config.ts
export default {
  clearMocks: true,    // equivalent to jest.clearAllMocks() after each test
  restoreMocks: true,  // equivalent to jest.restoreAllMocks() after each test
};
```

---

## 4. Fake timers — mocking Date and setTimeout

`new Date()` without a fixed value is a FIRST-Repeatable violation. Fake timers replace the system clock and timer functions so your tests are deterministic.

### API

```ts
// Setup (in beforeEach or at top of it)
jest.useFakeTimers();
vi.useFakeTimers(); // Vitest

// Set the current time
jest.setSystemTime(new Date('2026-01-15T10:00:00Z'));
vi.setSystemTime(new Date('2026-01-15T10:00:00Z'));

// Advance time
jest.advanceTimersByTime(5000); // advance 5 seconds
vi.advanceTimersByTime(5000);

// Run all pending timers immediately
jest.runAllTimers();
vi.runAllTimers();

// Restore real timers
jest.useRealTimers();
vi.useRealTimers();
```

### Recipe: test something that depends on "now"

```ts
describe('Task.markAsDone', () => {
  beforeEach(() => {
    jest.useFakeTimers();
  });

  afterEach(() => {
    jest.useRealTimers();
  });

  it('rejects completion when doneAt is in the future', () => {
    // Arrange
    jest.setSystemTime(new Date('2026-01-15T10:00:00Z'));
    const task = Task.create({ title: 'Deploy', agencyId: TEST_AGENCY_ID }).value;
    const futureDate = new Date('2026-01-16T10:00:00Z');

    // Act
    const result = task.markAsDone(futureDate);

    // Assert
    expect(result.isFailure).toBe(true);
    expect(result.error.message).toContain('cannot be in the future');
  });

  it('accepts completion when doneAt is now', () => {
    // Arrange
    const now = new Date('2026-01-15T10:00:00Z');
    jest.setSystemTime(now);
    const task = Task.create({ title: 'Deploy', agencyId: TEST_AGENCY_ID }).value;

    // Act
    const result = task.markAsDone(now);

    // Assert
    expect(result.isSuccess).toBe(true);
  });
});
```

---

## 5. Partial and fuzzy matchers — less brittle assertions

Asserting exact equality on large objects is brittle — every new field breaks the test. Partial matchers let you focus on what actually matters.

### Core matchers

```ts
// Object subset — only checks specified properties
expect(task).toMatchObject({ title: 'Fix bug', status: 'pending' });

// Object subset in call args
expect(mockRepo.save).toHaveBeenCalledWith(
  expect.objectContaining({ title: 'Fix bug', status: 'pending' })
);

// Array — checks array contains these elements (order irrelevant)
expect(tags).toEqual(expect.arrayContaining(['urgent', 'backend']));

// String contains substring
expect(errorMessage).toEqual(expect.stringContaining('required'));

// String matches regex
expect(errorMessage).toEqual(expect.stringMatching(/must be .+ characters/));

// Type-only match — any string, any number, any Date, etc.
expect(task.id).toEqual(expect.any(String));
expect(task.createdAt).toEqual(expect.any(Date));

// Any non-null/undefined value
expect(result.value).toEqual(expect.anything());
```

### Recipe: verify save was called with the right business fields, ignoring generated fields

```ts
it('saves a task with correct domain fields', async () => {
  // Arrange
  mockRepo.save.mockResolvedValue(undefined);
  const command = new CreateTaskCommand('Fix login bug', TEST_AGENCY_ID);

  // Act
  await handler.execute(command);

  // Assert
  expect(mockRepo.save).toHaveBeenCalledWith(
    expect.objectContaining({
      title: 'Fix login bug',
      status: 'pending',
      agencyId: TEST_AGENCY_ID,
    })
    // id, createdAt, sortIndex are NOT asserted — they're generated
  );
});
```

---

## 6. Exception and error assertions

### Synchronous throws

```ts
it('throws when title is empty', () => {
  // Arrange
  const invalidTitle = '';

  // Act + Assert (combined when testing a throw)
  expect(() => Task.create({ title: invalidTitle, agencyId: TEST_AGENCY_ID }))
    .toThrow('title is required');
});
```

### Async throws

```ts
it('rejects when repo throws', async () => {
  // Arrange
  mockRepo.save.mockRejectedValue(new Error('DB connection lost'));
  const command = new CreateTaskCommand('Fix bug', TEST_AGENCY_ID);

  // Act + Assert
  await expect(handler.execute(command)).rejects.toThrow('DB connection lost');
});
```

### Async success assertion

```ts
it('resolves with the created task id', async () => {
  // Arrange
  mockRepo.save.mockResolvedValue(undefined);
  const command = new CreateTaskCommand('Fix bug', TEST_AGENCY_ID);

  // Act + Assert
  await expect(handler.execute(command)).resolves.toMatchObject({
    isSuccess: true,
  });
});
```

### `expect.assertions(n)` — guard against silently skipped async asserts

Without this, an async test that never reaches its `expect` still passes (the promise resolves but no assertion ran).

```ts
it('calls notify after save', async () => {
  // Arrange
  expect.assertions(2); // MUST hit exactly 2 assertions or the test fails
  mockRepo.save.mockResolvedValue(undefined);
  const command = new CreateTaskCommand('Fix bug', TEST_AGENCY_ID);

  // Act
  const result = await handler.execute(command);

  // Assert
  expect(result.isSuccess).toBe(true);
  expect(mockNotifier.notify).toHaveBeenCalledTimes(1);
});
```

### Recipe: Result pattern (preferred over try/catch in DaraMex)

```ts
it('returns failure with 400 status on invalid command', async () => {
  // Arrange
  const command = new CreateTaskCommand('', TEST_AGENCY_ID); // empty title

  // Act
  const result = await handler.execute(command);

  // Assert
  expect(result.isFailure).toBe(true);
  expect(result.error.httpStatus).toBe(400);
  expect(result.error.message).toContain('title is required');
});
```

Never use try/catch to test failure in DaraMex — the Result pattern makes it unnecessary.

---

## 7. Snapshots — when they help, when they hurt

Snapshots capture the serialized output of a value and compare it on future runs. They sound powerful but are almost always misused.

### `toMatchSnapshot()` — file-based

Creates a `.snap` file alongside the test. Updated with `jest --updateSnapshot` / `vitest --update`.

### `toMatchInlineSnapshot()` — string in the test file

Vitest/Jest writes the snapshot inline into the test file on first run.

```ts
it('serializes task mapper output', () => {
  // Arrange
  const task = Task.rehydrate({ id: 'fixed-id', title: 'Deploy', status: 'pending' });

  // Act
  const dto = TaskMapper.toDTO(task);

  // Assert
  expect(dto).toMatchInlineSnapshot(`
    {
      "id": "fixed-id",
      "status": "pending",
      "title": "Deploy",
    }
  `);
});
```

### When snapshots are useful

- Large but **stable** output (e.g. generated SQL, CLI output, serialized mappers).
- Inline snapshots for short, stable strings — they document the expected shape right in the test.

### When snapshots are dangerous

- Output includes timestamps, random IDs, or any non-deterministic values.
- UI component snapshots — prefer Testing Library role/text queries instead.
- When you hit "update snapshot" without reading the diff — the snapshot becomes a rubber stamp.

### DaraMex position

Use inline snapshots sparingly for **mapper output**. Avoid file-based snapshots unless there is an explicit justification. Never use snapshots for domain entity assertions.

---

## 8. Module mocking (Vitest focus, Jest equivalent noted)

Module mocks replace an entire import at the module registry level. Use them for infrastructure adapters, env config, or when `jest.fn()` on an injected dependency is not enough.

### `vi.mock(path, factory)` — auto-mock a module (hoisted to top of file)

```ts
// Panel test file
vi.mock('@/lib/http-client', () => ({
  httpClient: {
    get: vi.fn(),
    post: vi.fn(),
  },
}));

it('fetches tasks from the API', async () => {
  // Arrange
  const mockGet = vi.mocked(httpClient.get);
  mockGet.mockResolvedValue({ data: [{ id: '1', title: 'Fix bug' }] });

  // Act
  const result = await fetchTasks();

  // Assert
  expect(result).toHaveLength(1);
  expect(result[0].title).toBe('Fix bug');
});
```

### `vi.hoisted(() => ...)` — hoist values used by `vi.mock` factory

```ts
const mockSave = vi.hoisted(() => vi.fn());

vi.mock('@/lib/storage', () => ({
  storage: { save: mockSave },
}));

it('saves data on submit', async () => {
  // Arrange
  mockSave.mockResolvedValue(undefined);

  // Act
  await submitForm({ title: 'Task' });

  // Assert
  expect(mockSave).toHaveBeenCalledWith(expect.objectContaining({ title: 'Task' }));
});
```

### `vi.importActual(path)` — partial mock: keep most, override one export

```ts
vi.mock('@/utils/date', async () => {
  const actual = await vi.importActual<typeof import('@/utils/date')>('@/utils/date');
  return {
    ...actual,
    getNow: vi.fn(() => new Date('2026-01-15T10:00:00Z')), // mock only this
  };
});
```

### `vi.stubGlobal` / `vi.stubEnv`

```ts
// Stub a global (e.g. fetch)
vi.stubGlobal('fetch', vi.fn().mockResolvedValue({ json: () => ({ ok: true }) }));

// Stub an env variable
vi.stubEnv('NODE_ENV', 'production');
```

Remember to call `vi.unstubAllGlobals()` / `vi.unstubAllEnvs()` in `afterEach`.

### Jest equivalents

```ts
jest.mock('path/to/module', () => ({ fn: jest.fn() }));
jest.requireActual('path/to/module'); // partial mock
jest.resetModules(); // clear module registry between tests
```

---

## 9. React Testing Library (panel only)

RTL's philosophy: test what the user sees, not the implementation. Every RTL query is implicitly an accessibility check.

### Query priority order (follow this — always)

| Priority | Query | When |
|----------|-------|------|
| 1 | `getByRole` | Interactive elements, landmarks |
| 2 | `getByLabelText` | Form fields |
| 3 | `getByPlaceholderText` | Inputs without labels |
| 4 | `getByText` | Non-interactive text |
| 5 | `getByDisplayValue` | Current value of select/input |
| 6 | `getByAltText` | Images |
| 7 | `getByTitle` | Title attribute |
| 8 | `getByTestId` | **Last resort only** |

### Query variants

```ts
// getBy* — throws if not found. Use when element MUST be present.
const button = screen.getByRole('button', { name: /submit/i });

// queryBy* — returns null. Use to assert absence.
expect(screen.queryByRole('alert')).not.toBeInTheDocument();

// findBy* — async, polls until found. Use for elements appearing after async.
const heading = await screen.findByRole('heading', { name: /task created/i });
```

### `within(container)` — scope queries

```ts
const list = screen.getByRole('list');
const items = within(list).getAllByRole('listitem');
expect(items).toHaveLength(3);
```

### `userEvent` (strongly preferred over `fireEvent`)

```ts
import userEvent from '@testing-library/user-event';

it('submits the form on Enter', async () => {
  // Arrange
  const user = userEvent.setup(); // create a user instance per test
  const onSubmit = vi.fn();
  render(<TaskForm onSubmit={onSubmit} />);

  // Act
  await user.type(screen.getByLabelText(/title/i), 'Fix login bug');
  await user.keyboard('{Enter}');

  // Assert
  expect(onSubmit).toHaveBeenCalledWith(
    expect.objectContaining({ title: 'Fix login bug' })
  );
});
```

Key `userEvent` methods:

```ts
await user.click(element);
await user.type(element, 'text');
await user.keyboard('{Enter}');
await user.hover(element);
await user.clear(element);
await user.selectOptions(selectEl, 'value');
```

### `waitFor` / `waitForElementToBeRemoved`

```ts
// Wait for an assertion to pass (polls until timeout)
await waitFor(() => {
  expect(screen.getByRole('alert')).toHaveTextContent('Saved');
});

// Wait for an element to disappear
await waitForElementToBeRemoved(() => screen.queryByRole('progressbar'));
```

### `renderHook` — testing hooks that are a public API

Only use when the hook IS the public API (a library, a shared package). Never to test component internals.

```ts
it('returns false when list is empty', () => {
  // Arrange + Act
  const { result } = renderHook(() => useTaskSelection([]));

  // Assert
  expect(result.current.hasSelection).toBe(false);
});
```

---

## 10. NestJS testing utilities (API only)

### `Test.createTestingModule` — the foundation

```ts
import { Test, TestingModule } from '@nestjs/testing';

const module: TestingModule = await Test.createTestingModule({
  providers: [
    CreateTaskHandler,
    { provide: TASK_REPOSITORY, useValue: mockRepo },
  ],
}).compile();

const handler = module.get(CreateTaskHandler);
```

### `overrideProvider` — swap a dependency for a test double

```ts
const module = await Test.createTestingModule({
  imports: [TasksModule],
})
  .overrideProvider(TASK_REPOSITORY)
  .useValue(mockRepo)
  .compile();
```

### `overrideGuard` — bypass guards in controller tests

```ts
.overrideGuard(JwtAuthGuard)
.useValue({ canActivate: () => true })
```

### Recipe: minimal controller integration test with supertest

```ts
describe('TasksController (integration)', () => {
  let app: INestApplication;
  let mockRepo: jest.Mocked<ITaskRepository>;

  beforeAll(async () => {
    // Arrange
    mockRepo = { findById: jest.fn(), save: jest.fn() } as jest.Mocked<ITaskRepository>;

    const module = await Test.createTestingModule({
      imports: [TasksModule],
    })
      .overrideProvider(TASK_REPOSITORY)
      .useValue(mockRepo)
      .overrideGuard(JwtAuthGuard)
      .useValue({ canActivate: () => true })
      .compile();

    app = module.createNestApplication();
    await app.init();
  });

  afterAll(() => app.close());

  it('GET /tasks/:id returns 404 when task not found', async () => {
    // Arrange
    mockRepo.findById.mockResolvedValue(null);

    // Act
    const response = await request(app.getHttpServer())
      .get('/tasks/non-existent-id')
      .expect(404);

    // Assert
    expect(response.body.message).toContain('not found');
  });
});
```

---

## 11. Concurrency and filtering

### Focus and skip — run subsets during development

```ts
it.only('this is the only test that runs', () => { /* ... */ });
describe.only('only this suite runs', () => { /* ... */ });

it.skip('not ready yet', () => { /* ... */ });
describe.skip('skip this whole suite', () => { /* ... */ });
```

**Never commit `.only`** — CI will either fail or silently skip everything else.

### `it.todo` — mark intent without implementing

```ts
it.todo('rejects creation when agency is suspended');
```

Shows in test output as a reminder. NOT a substitute for writing the test.

### `it.failing` (Vitest only) — track a known bug

```ts
it.failing('calculates correct sortIndex when list has duplicates', () => {
  // Arrange
  const tasks = [taskA, taskA]; // known duplicate bug

  // Act
  const result = calculateSortIndex(tasks, null);

  // Assert — this currently fails, that's expected
  expect(result).toBeGreaterThan(taskA.sortIndex);
});
```

The test passes when the assertion fails (tracking the known bug). When you fix the bug, remove `.failing`.

### `it.concurrent` / `describe.concurrent` — parallel tests within a file

Vitest supports this natively. Jest requires `--runInBand` to be off (default).

```ts
describe.concurrent('independent computations', () => {
  it('computation A', async () => { /* runs in parallel */ });
  it('computation B', async () => { /* runs in parallel */ });
});
```

Only use concurrent when tests have NO shared mutable state.

### Command-line filtering

```bash
# API (Jest)
pnpm --filter api test -- --testPathPatterns="tasks"    # file pattern
pnpm --filter api test -- -t "rejects creation"         # test name pattern

# Panel (Vitest)
pnpm --filter panel test -- path/to/file.test.tsx       # single file
pnpm --filter panel test -- --reporter=verbose          # detailed output
```

---

## 12. Coverage

### Commands

```bash
pnpm --filter api test:cov          # Jest — outputs lcov + text summary
pnpm --filter panel test -- --coverage  # Vitest — outputs same formats
```

### Thresholds

Jest (`jest.config.ts`):

```ts
coverageThreshold: {
  global: {
    branches: 80,
    functions: 80,
    lines: 80,
    statements: 80,
  },
},
```

Vitest (`vitest.config.ts`):

```ts
coverage: {
  thresholds: {
    branches: 80,
    functions: 80,
    lines: 80,
    statements: 80,
  },
},
```

**Do NOT chase 100%.** Coverage tells you which lines ran, not whether you tested the right behaviors. A test that increments a counter to hit a line is worse than no test.

The goal: every **meaningful branch** (error path, edge case, domain rule) is covered. Getters, constructors that do nothing, and trivial delegations can stay uncovered.

---

## Anti-tool patterns — things that LOOK useful but aren't

### `toHaveBeenCalled()` alone — Rule 8 violation

```ts
// BAD — passes even if save is called with garbage data
expect(mockRepo.save).toHaveBeenCalled();

// GOOD — verifies it was called with the right domain data
expect(mockRepo.save).toHaveBeenCalledWith(
  expect.objectContaining({ title: 'Fix bug', status: 'pending' })
);
```

### Snapshot testing everything

Updating snapshots on every CI failure without reading the diff defeats the purpose. A snapshot should document stable, intentional output — not be a rubber stamp.

### `beforeAll` for per-test state

```ts
// BAD — shared mock state leaks between tests
beforeAll(() => {
  mockRepo.save.mockResolvedValue(undefined);
});

// GOOD — fresh mock state per test
beforeEach(() => {
  mockRepo = { save: jest.fn() } as jest.Mocked<ITaskRepository>;
});
```

### `it.todo` as a parking lot

Writing `it.todo('verify X')` and moving on is procrastination. Either write the test or track the behavior in an issue. A TODO list of 20 untested behaviors is not progress.

### `it.only` committed to a branch

One `it.only` silently skips your entire test suite. CI may pass while 90% of tests never run. Configure a pre-commit hook or CI lint step to catch this: `grep -r "it.only\|test.only\|describe.only" apps/`.
