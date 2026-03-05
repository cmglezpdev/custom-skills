---
name: nestjs-wide-events
description: Implement structured, wide-event logging in NestJS applications following the canonical log line / wide event pattern. Use this skill whenever the user asks about logging, observability, debugging, or tracing in a NestJS app. Also trigger when the user mentions log lines, structured logging, canonical log lines, wide events, request context, observability, or asks how to improve their NestJS logging setup. Use this even if the user just says "add logging" to a NestJS project, since the wide event pattern should be the default, not scattered console.log calls.
---

# NestJS Wide Events Logging

This skill implements the **wide event** (aka canonical log line) pattern in NestJS applications. The philosophy: emit ONE context-rich structured event per request per service, instead of scattering dozens of log lines across your codebase.

## Why This Matters

Traditional logging (`console.log("Payment failed")`) optimizes for writing, not querying. When debugging production issues, you need to correlate events across the full request lifecycle. Wide events solve this by capturing everything about a request in a single structured JSON object with high cardinality (many unique values per field) and high dimensionality (many fields per event).

## Core Principles

1. **One event per request per service.** Build a context object throughout the request lifecycle. Emit it once at the end.
2. **Business context is mandatory.** User ID is not enough. Include subscription tier, account age, feature flags, cart value, anything you would need at 2am during an incident.
3. **High cardinality is the point.** Fields like `user_id`, `request_id`, `order_id` with millions of unique values are what make debugging possible. Do not limit yourself to low-cardinality fields like `http_method`.
4. **High dimensionality wins.** 50 fields on one event beats 5 fields across 10 events. More dimensions means more questions you can answer without a second query.
5. **Structured JSON only.** Never emit plain string logs. Every field is a queryable key-value pair.
6. **Tail sampling controls cost.** Always keep errors, slow requests, and VIP users. Randomly sample the rest.
7. **No log-level abuse.** Do not use `debug`, `info`, `warn` as organizational categories for scattered log lines. The wide event IS the log. Use level only to indicate severity of the outcome.

## Architecture in NestJS

Read `references/implementation-patterns.md` for full code. Here is the high-level design:

### Components

- **`WideEventInterceptor`** (global NestJS interceptor): Creates the wide event context at request start, captures it at request end, emits the single event. This is the backbone.
- **`WideEventService`** (request-scoped injectable): Holds the mutable event context for the current request. Any service, guard, or pipe can inject it and enrich the event with business context.
- **`WideEventModule`** (NestJS module): Wires everything together. Import it at root level.
- **`TailSampler`** (utility): Decides whether to persist the event after it is built. Always keeps errors, slow requests, flagged users. Samples the rest.
- **`WideEventExceptionFilter`** (global exception filter): Enriches the event with error details before the interceptor emits.

### Request Lifecycle Flow

```
Request arrives
  → WideEventInterceptor.intercept() starts timer, seeds event with HTTP metadata
    → Guards, pipes, etc. can inject WideEventService and add context
      → Controller handler runs, injects WideEventService, adds business context
        → Downstream services inject WideEventService, add timing, results, errors
      → Response or exception
    → WideEventExceptionFilter catches errors, enriches event
  → WideEventInterceptor.intercept() finalizes: duration, status, outcome
  → TailSampler decides: persist or drop
  → Single JSON event emitted via Logger
```

### What Goes in the Wide Event

Every wide event SHOULD include these field groups. Not all will apply to every endpoint, but aim for coverage:

**Request envelope** (always present, set by interceptor automatically):
- `timestamp`, `request_id`, `trace_id`
- `method`, `path`, `query_params` (sanitized), `status_code`, `duration_ms`
- `ip`, `user_agent`
- `service`, `version`, `deployment_id`, `region`, `node_env`

**User context** (set by auth guard or controller):
- `user.id`, `user.subscription`, `user.account_age_days`, `user.role`
- `user.org_id`, `user.org_name`

**Business context** (set by controller/service, domain-specific):
- For checkout: `cart.id`, `cart.item_count`, `cart.total_cents`, `cart.coupon`
- For payments: `payment.method`, `payment.provider`, `payment.latency_ms`, `payment.attempt`
- For any domain: the key entities being operated on and their relevant attributes

**Error context** (set by exception filter):
- `error.type`, `error.code`, `error.message`, `error.retriable`
- `error.stack` (only in non-production or for 500s)
- Domain-specific error fields (`error.stripe_decline_code`, etc.)

**Performance context** (set by services doing I/O):
- `db.query_count`, `db.total_ms`, `db.slowest_query_ms`
- `cache.hit`, `cache.key`, `cache.latency_ms`
- `external_calls[].service`, `external_calls[].duration_ms`, `external_calls[].status`

**Feature flags** (set by feature flag guard or service):
- `feature_flags.*` as flat boolean fields

**Outcome** (set by interceptor):
- `outcome`: `"success"` | `"error"` | `"client_error"`
- `level`: Derived from status code. 5xx = `error`, 4xx = `warn`, 2xx/3xx = `info`

### Key Implementation Rules

1. **Use `REQUEST` scope for `WideEventService`.** Each request gets its own instance. This is how concurrent requests avoid contaminating each other's context.
2. **The interceptor is the only place that calls `logger.info(event)`.** No other code emits log lines. Services only enrich the event object.
3. **Sanitize sensitive fields.** Strip passwords, tokens, PII from query params and bodies before attaching them. Define a sanitization allowlist, not a denylist.
4. **Use `cls-hooked` or `AsyncLocalStorage` if you cannot use request-scoped injection everywhere.** Some contexts (event handlers, queue consumers) do not have access to the NestJS DI request scope. AsyncLocalStorage bridges this gap.
5. **Nest related fields under namespaces.** `user.id` not `userId`. `payment.latency_ms` not `paymentLatencyMs`. This keeps the event scannable and queryable with dot notation.
6. **Timings are always `_ms` suffix, integers.** Not seconds, not floats. Consistent units prevent confusion.
7. **For queue consumers, background jobs, and cron tasks:** emit a wide event per job execution, not per request. Same pattern, different trigger.

### Tail Sampling Rules

```
ALWAYS keep (100%):
- status_code >= 500
- any event with error field populated
- duration_ms > p99 threshold (configure per endpoint)
- user.subscription === 'enterprise' or similar VIP flag
- feature_flags containing any active rollout being monitored

SAMPLE the rest (configurable, default 5%):
- Healthy, fast, unremarkable requests
```

### What NOT to Do

- Do NOT scatter `this.logger.log()` / `console.log()` calls through your services. This is the old model. It creates noise and makes correlation impossible.
- Do NOT use log levels as a way to categorize different stages of request processing. There is one event. It has one level based on outcome.
- Do NOT put high-volume debug tracing into the wide event. If you need function-level tracing for development, use a separate debug-only tracer that is disabled in production.
- Do NOT log request/response bodies wholesale. Sanitize and extract only the fields you need.
- Do NOT treat OpenTelemetry as a substitute for wide events. OTel is a transport layer. You still decide what to capture. Ideally, your wide event IS your enriched OTel span.

## Quick Start

When adding logging to a new or existing NestJS app:

1. Read `references/implementation-patterns.md` for full code of all components.
2. Create `WideEventModule` with `WideEventService` (request-scoped) and `WideEventInterceptor` (global).
3. Register the module at the app root.
4. In each controller and service that handles meaningful business logic, inject `WideEventService` and call `event.enrich({...})` with domain context.
5. Add `TailSampler` and configure sampling rules.
6. Remove all existing scattered log statements. Replace them with enrichments to the wide event.
