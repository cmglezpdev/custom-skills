---
name: daramex-testing
description: >
  DaraMex-specific testing rules and enterprise-grade testing philosophy for the monorepo.
  Covers NestJS (Jest) in apps/api/ and React (Vitest + Testing Library) in apps/panel/,
  applied to Clean/Hexagonal Architecture. Use this skill whenever writing, reviewing,
  refactoring, or evaluating tests in DaraMex: unit tests, integration tests, controller
  tests with supertest, React component tests, mocks, or when asked to audit, fix, or
  improve the test suite. Also trigger when the user mentions "testing", "tests", "specs",
  "TDD", "mocks", "coverage", "what to test", or any variation in Spanish ("pruebas",
  "tests", "qué testear", "mocks").
license: Apache-2.0
metadata:
  author: gentleman-programming
  version: "2.2"
---

# DaraMex Testing Rules

Project-specific testing overlay for the DaraMex monorepo. Use **alongside** the generic `tdd`, `vitest`, and `nestjs-best-practices` skills — this skill provides the DaraMex-specific decisions layered on top.

**API stack**: NestJS + Jest 30 + ts-jest 29 + supertest (Clean/Hexagonal)
**Panel stack**: React + Vitest 4 + @testing-library/react 16 + jsdom
**Mode**: Strict TDD (RED → GREEN → TRIANGULATE) — see `tdd` skill

---

## The 11 Non-Negotiable Rules

1. **A test that doesn't compile is not a test. It's a lie.** When you refactor production code, you update the tests IN THE SAME PR. If you can't, DELETE the test — a deleted test is honest, a rotten test is false coverage.
2. **A bad test is worse than no test.** It gives false confidence and slows refactors. Delete brittle tests — don't "fix" them.
3. **Test behavior, not implementation.** If refactoring without changing behavior breaks the test, the test is wrong.
4. **Pyramid shape** — many unit, some integration, few E2E. Never inverted.
5. **Every test name must describe a behavior or rule**, not "should work" or "should create".
6. **AAA comments MANDATORY** — every `it` must have literal `// Arrange`, `// Act`, `// Assert` comments. Blank lines alone are NOT sufficient. This is a project-level convention, not a style choice.
7. **Never use `as any` in mocks** — always typed (`jest.Mocked<IPort>` or `Partial<IPort>`).
8. **Verify arguments AND return values**, not just "was called". `toHaveBeenCalled()` alone is a code smell.
9. **One logical assertion per test**. If you need 2 `describe.each`, split into two `it`.
10. **No `new Date()` or `Math.random()` unmocked** — breaks FIRST-Repeatable.
11. **Every bug fix follows the 4-step workflow**: (1) write a test that reproduces the bug and FAILS, (2) apply the minimal fix, (3) confirm the test PASSES, (4) commit test + fix together in the same PR. Fixing before writing the test is REJECTED. See `patterns.md` section 15 for the full workflow.

---

## Testing Pyramid — what goes where

| Layer | Target | % of suite | Tool | What it verifies |
|-------|--------|-----------|------|------------------|
| **Unit** | Pure functions, domain entities, value objects, use case handlers (with mocked ports) | ~70% | Jest / Vitest | Business rules, edge cases, branches |
| **Integration** | Repositories against real DB, controllers via supertest, adapters against fake HTTP | ~20% | Jest + supertest / Vitest + MSW | Wiring, DB contracts, HTTP contracts |
| **E2E** | Complete user flows through the deployed app | ~10% | supertest (API) | Critical flows only (auth, checkout, onboarding) |

Anti-pattern: **Ice cream cone 🍦** (few units, many E2E). Rejected.

---

## Clean/Hexagonal layers — what to test in each

DaraMex API uses `domain/` + `application/` + `infrastructure/` in every module. Each layer has a distinct testing approach:

### 🔹 Domain layer → PURE unit tests

- **No NestJS, no mocks, no DB, no HTTP.** Just plain TypeScript.
- Test: entities, value objects, domain services, domain events.
- Every branch, every `throw`, every state transition gets a test.
- Fastest and most valuable tests you'll write.

### 🔹 Application layer → unit tests with ports mocked

- Command handlers, query handlers, use cases.
- Dependencies (`IXxxRepository`, `IXxxGateway`) are **interfaces** — mock with `jest.Mocked<IXxx>`.
- Do NOT mock domain classes (they're pure, use them as-is).
- Verify: `Result.ok` / `Result.fail`, arguments passed to ports, domain validations stop I/O early.

### 🔹 Infrastructure layer → integration tests

- Adapters (TypeORM repos, HTTP gateways, ElevenLabs/OpenAI clients, etc.).
- Controllers — use `@nestjs/testing` `Test.createTestingModule` + `supertest`.
- DB: prefer a test container or in-memory SQLite; assert real round-trips.
- External HTTP: mock with `nock` or MSW, assert the adapter sends correct requests and parses responses correctly.

### 🔹 E2E → full flows

- Reserved for **critical user journeys** only.
- Boot the full app. Use real HTTP. Hit the real DB in a test schema.
- Do not duplicate CRUD integration tests here.

---

## Panel (React) — Testing Library rules

1. **Query by role first** (`getByRole('button', { name: /submit/i })`), then by text, then by label. NEVER by class, id, or test-id unless unavoidable.
2. **`userEvent` over `fireEvent`** — simulates real user interactions.
3. **Do not test hooks directly** unless the hook is the public API of a package. Test the component that uses the hook.
4. **Do not test `className`, internal state, or DOM structure** — test what the user sees and can do.
5. **Assert accessible output**: `screen.getByText`, `getByRole`, `findByLabelText`. These double as a11y guarantees.
6. **Every `onClick`, `onChange`, `onSubmit` prop** is a contract — test that it's called with correct args.

---

## Anti-patterns to hunt down in this codebase

When auditing or refactoring DaraMex tests, flag and kill:

| Anti-pattern | How to spot it | Fix |
|--------------|----------------|-----|
| **Mock Hell** | Only `toHaveBeenCalled()` asserts; tests pass with any behavior | Assert arguments + return values |
| **Existence tests** | `expect(method).toBeDefined()` | Delete — TypeScript already guarantees this |
| **Constructor tests** | `expect(obj.name).toBe('x')` right after `new Obj({ name: 'x' })` | Delete unless there's validation logic |
| **CSS class assertions** | `toHaveClass('foo')` | Replace with role/text queries |
| **`as any` in mocks** | `repo as any`, `service as any` | Replace with `jest.Mocked<IPort>` |
| **Shared mutable state** | Mocks defined at `describe` scope, reused across `it` without reset | Move to `beforeEach` with fresh instances |
| **Test the logger/metrics** | `expect(logger.log).toHaveBeenCalled()` | Delete — that's implementation detail |
| **`new Date()` without fixed arg** | Flaky timestamp comparisons | Use fixed dates or `jest.useFakeTimers()` |
| **Unclear test names** | "should work", "should create", "returns true" | Rewrite to describe the rule: "rejects creation when X is empty" |

---

## Commands

```bash
# API (NestJS + Jest)
pnpm --filter api test                                    # all tests
pnpm --filter api test -- path/to/file.spec.ts            # single file
pnpm --filter api test -- --testPathPatterns="users"      # by pattern
pnpm --filter api test:cov                                # with coverage
pnpm --filter api test:e2e                                # e2e only
pnpm --filter api exec npx jest --forceExit               # when hangs

# Panel (React + Vitest)
pnpm --filter panel test                                  # all tests
pnpm --filter panel test -- path/to/file.test.tsx         # single file
pnpm --filter panel test -- --coverage                    # with coverage
pnpm --filter panel test -- --ui                          # Vitest UI
```

---

## Resources

- **Fundamentals** → `references/fundamentals.md` — Testing pyramid, FIRST principles, what to test vs what not to test.
- **Examples** → `references/examples.md` — Paired MAL/BIEN examples for every layer. Grows over time.
- **Real-world anti-patterns** → `references/real-world-antipatterns.md` — **READ THIS**. Specific anti-patterns measured in this codebase (as of 2026-04 audit): 186 `as any` in mocks, 192 Mock Hell lines, 129/187 suites broken by abandonment during refactors. Includes fix strategies prioritized by ROI.
- **Positive patterns** → `references/patterns.md` — The positive counterpart to anti-patterns. Covers AAA, fixture factories, `jest.Mocked<IPort>`, `Task.create()` vs `Task.rehydrate()`, passthrough mocks, invariant-based asserts, `describe.each` usage, and test failure diagnosis (rotten vs broken vs unbuilt).
- **Tooling cheatsheet** → `references/tooling-cheatsheet.md` — Jest 30 + Vitest 4 APIs most developers underuse: parameterized tests (`it.each`, `it.for`), typed mocks, fake timers, partial matchers, module mocking, React Testing Library queries, NestJS `Test.createTestingModule` + supertest.
- **Companion skills**:
  - `tdd` — red-green-refactor loop (Strict TDD is mandatory in this repo)
  - `vitest` — Vitest-specific patterns for the panel
  - `nestjs-best-practices` — architectural rules that inform what to test

---

## Quick Checklist Before Merging Tests

- [ ] **The test compiles** — running it doesn't produce `Test suite failed to run` with TS errors
- [ ] Every `it` name describes a rule or behavior, not "should work"
- [ ] AAA comments present (`// Arrange`, `// Act`, `// Assert`) — MANDATORY, not optional
- [ ] No `as any` in mocks — all ports typed
- [ ] Asserts verify arguments AND return values, not only `toHaveBeenCalled`
- [ ] Every error branch is tested (not just happy path) — with 4 layers: failure type, HTTP status, error code, short-circuit
- [ ] `result` from `handler.execute()` is always captured and asserted — never `await handler.execute(...)` with the result discarded
- [ ] No test depends on another test's execution order
- [ ] No timestamp or randomness without a fixed seed/value
- [ ] Pyramid respected (don't add an E2E if an integration test would cover it)
- [ ] React tests use role/text queries, not CSS classes
- [ ] Bug fixes ship with a regression test
