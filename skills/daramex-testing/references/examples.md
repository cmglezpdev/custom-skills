# Testing Examples — MAL vs BIEN

Living document of paired examples showing bad test patterns alongside their enterprise-grade counterparts.

Each example follows the same structure:
- 🚫 **MAL** — typical anti-pattern
- **Problems** — why it's bad
- ✅ **BIEN** — the correct version
- **Why it's enterprise-grade** — lessons to internalize

---

## 1. Domain Entity — `Task.markAsDone`

### 🚫 MAL

```ts
describe('Task', () => {
  it('should create a task', () => {
    const task = new Task({ title: 'Test' })
    expect(task.title).toBe('Test')
  })

  it('should have a markAsDone method', () => {
    const task = new Task({ title: 'Test' })
    task.markAsDone()
    expect(task.markAsDone).toBeDefined()
  })
})
```

**Problems:**
- "should create a task" tests the constructor. TypeScript already guarantees this. Catches no bug.
- "should have a markAsDone method" tests method existence. You could delete the body and it passes. USELESS.
- No behavior tested.
- No edge cases.
- Vague names — don't describe any rule.

### ✅ BIEN

```ts
describe('Task.markAsDone', () => {
  it('transitions a pending task to done and records the completion time', () => {
    // Arrange
    const task = Task.create({ title: 'Ship invoice', status: 'pending' })
    const now = new Date('2026-01-15T10:00:00Z')

    // Act
    task.markAsDone(now)

    // Assert
    expect(task.status).toBe('done')
    expect(task.completedAt).toEqual(now)
  })

  it('throws DomainError when trying to mark an already-done task', () => {
    const task = Task.create({ title: 'x', status: 'done' })

    expect(() => task.markAsDone(new Date())).toThrow(
      new DomainError('Task is already done'),
    )
  })

  it('throws DomainError when completion time is in the future', () => {
    const task = Task.create({ title: 'x', status: 'pending' })
    const future = new Date('2099-01-01')

    expect(() => task.markAsDone(future)).toThrow(DomainError)
  })
})
```

**Why it's enterprise-grade:**
1. `describe` scopes to the method under test, not the whole class.
2. `it` names describe behavior + context ("transitions a pending task to done AND records completion time").
3. AAA visible with blank lines / comments.
4. Covers happy path + both error branches.
5. Fixed dates → reproducible.
6. Tests behavior, not existence.

---

## 2. Application Handler — `CreateTaskHandler`

### 🚫 MAL — Mock Hell

```ts
describe('CreateTaskHandler', () => {
  it('should create a task', async () => {
    const repo = { save: jest.fn().mockResolvedValue(undefined) }
    const logger = { log: jest.fn() }
    const handler = new CreateTaskHandler(repo as any, logger as any)

    await handler.execute({ title: 'x', orgId: 'org-1' })

    expect(repo.save).toHaveBeenCalled()
    expect(logger.log).toHaveBeenCalled()
  })
})
```

**Problems:**
- `toHaveBeenCalled()` without args → you could pass an empty object to the repo and it passes.
- Testing `logger.log` → coupled to implementation; remove the log and the test breaks even though behavior is fine.
- `as any` kills type safety.
- No failure case. That's probably where your bugs actually live.
- Vague name.

### ✅ BIEN

```ts
describe('CreateTaskHandler', () => {
  let repo: jest.Mocked<ITaskRepository>
  let handler: CreateTaskHandler

  beforeEach(() => {
    repo = {
      save: jest.fn(),
      findById: jest.fn(),
    } satisfies jest.Mocked<ITaskRepository>
    handler = new CreateTaskHandler(repo)
  })

  it('persists a new Task with pending status and returns its id on success', async () => {
    repo.save.mockResolvedValue(Result.ok(undefined))

    const result = await handler.execute({ title: 'Invoice A', orgId: 'org-1' })

    expect(result.isOk()).toBe(true)
    const taskId = result.unwrap()

    expect(repo.save).toHaveBeenCalledTimes(1)
    const savedTask = repo.save.mock.calls[0][0]
    expect(savedTask.title).toBe('Invoice A')
    expect(savedTask.orgId).toBe('org-1')
    expect(savedTask.id).toBe(taskId)
    expect(savedTask.status).toBe('pending')
  })

  it('returns Result.fail with RepositoryError when persistence fails', async () => {
    repo.save.mockResolvedValue(Result.fail(new RepositoryError('DB down')))

    const result = await handler.execute({ title: 'x', orgId: 'org-1' })

    expect(result.isFail()).toBe(true)
    expect(result.unwrapErr()).toBeInstanceOf(RepositoryError)
  })

  it('does not touch the repository when title is empty (domain validation)', async () => {
    const result = await handler.execute({ title: '', orgId: 'org-1' })

    expect(result.isFail()).toBe(true)
    expect(repo.save).not.toHaveBeenCalled()
  })
})
```

**Why it's enterprise-grade:**
1. `jest.Mocked<ITaskRepository>` — adding a method to the port forces the mock update (type-safe).
2. Verifies mock ARGUMENTS (`savedTask.title`) — catches "saves wrong object".
3. Verifies RETURN value (`result.unwrap()` / `result.unwrapErr()`) — that's the handler's contract.
4. Three tests = three branches: happy path + infra failure + domain validation.
5. `not.toHaveBeenCalled()` proves validation short-circuits BEFORE infra — important business rule.
6. `beforeEach` guarantees FIRST-Independent.

---

## 3. React Component — `<TaskList />`

### 🚫 MAL

```tsx
describe('TaskList', () => {
  it('should render the component', () => {
    const { container } = render(<TaskList tasks={[]} />)
    expect(container.firstChild).toHaveClass('task-list')
  })

  it('should use useState for selected task', () => {
    // mocks useState somehow
  })

  it('should call setTasks on prop change', () => {
    const setTasks = vi.fn()
    render(<TaskList tasks={[]} setTasks={setTasks} />)
    // checks internal state...
  })
})
```

**Problems:**
- `toHaveClass('task-list')` tests CSS. Change Tailwind class → test breaks though user sees the same thing.
- Testing `useState` → the user doesn't know what useState is. Implementation detail.
- Testing `setTasks` internal — pure coupling.
- No real interaction.

### ✅ BIEN

```tsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'

describe('<TaskList />', () => {
  it('renders each task title from props', () => {
    render(<TaskList tasks={[{ id: '1', title: 'Buy milk', status: 'pending' }]} />)

    expect(screen.getByText('Buy milk')).toBeInTheDocument()
  })

  it('shows an empty state message when there are no tasks', () => {
    render(<TaskList tasks={[]} />)

    expect(screen.getByText(/no tasks yet/i)).toBeInTheDocument()
  })

  it('calls onTaskClick with the task id when the user clicks a task', async () => {
    const onTaskClick = vi.fn()
    const user = userEvent.setup()

    render(
      <TaskList
        tasks={[{ id: 't-1', title: 'Buy milk', status: 'pending' }]}
        onTaskClick={onTaskClick}
      />,
    )

    await user.click(screen.getByRole('button', { name: /buy milk/i }))

    expect(onTaskClick).toHaveBeenCalledWith('t-1')
  })
})
```

**Why it's enterprise-grade:**
1. Queries by what the USER sees (role, text) — roles > CSS, text > IDs.
2. `userEvent` simulates real interactions (keyboard, hover, delay) instead of fake pixel events.
3. Public props (`onTaskClick`) are the contract — those get tested.
4. No CSS, no hooks, no internals. Refactor `useState` → `useReducer` → Zustand: tests don't break. Only breaks when real user-visible behavior breaks.
5. Regex queries (`/no tasks yet/i`) tolerate minor copy changes.

---

## Rules extracted from these examples

- If you can delete the method body and the test PASSES → test is garbage.
- If refactor (no behavior change) BREAKS the test → test was coupled to implementation (garbage).
- If the test only says "was called" → missing "with WHAT args" and "returned WHAT".
- If the React test checks `className` or hooks → it tests how it's made, not what it does.
- If tests share state across `it` → fails FIRST-Independent. Use `beforeEach`.

---

## See also

For **anti-patterns measured in this specific codebase** (with counts, real examples, and prioritized remediation), read `real-world-antipatterns.md` in this directory.

---

<!-- New examples will be appended below as we cover new patterns. -->
