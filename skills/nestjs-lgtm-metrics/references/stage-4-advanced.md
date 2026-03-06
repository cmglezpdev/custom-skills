# Stage 4: Advanced Patterns

For scaled apps, complex architectures, and mature teams. Built with the OpenTelemetry SDK.

## Table of Contents
1. [Create Instruments](#instruments)
2. [WebSocket Metrics](#websocket)
3. [Rate Limiter Observability](#rate-limiter)
4. [Multi-Tenant Metrics](#multi-tenant)
5. [App Lifecycle Metrics](#lifecycle)
6. [Exemplar Support (Metrics + Traces)](#exemplars)
7. [Canary Deployment Metrics](#canary)
8. [PromQL Queries](#promql)

---

## 1. Create Instruments <a id="instruments"></a>

```typescript
// src/telemetry/instruments/stage4.instruments.ts
import { Inject, Injectable } from '@nestjs/common';
import { Meter, Counter, Histogram, UpDownCounter } from '@opentelemetry/api';
import { METER_TOKEN } from '../metrics.module';

@Injectable()
export class Stage4Instruments {
  // WebSocket
  public readonly wsMessages: Counter;
  public readonly wsConnectionDuration: Histogram;

  // Rate Limiter
  public readonly rateLimitHits: Counter;

  // Multi-Tenant
  public readonly tenantRequests: Counter;

  // Lifecycle
  public readonly shutdownDuration: Histogram;

  // Mutable state for observable gauges
  private wsConnectionCounts = new Map<string, number>();
  private rateLimitRemaining = new Map<string, number>();
  private tenantResourceUsages = new Map<string, number>();
  private startupDuration: number | null = null;
  private appInfoAttributes: Record<string, string> | null = null;

  constructor(@Inject(METER_TOKEN) meter: Meter) {
    // --- WebSocket ---
    meter.createObservableGauge('app_ws_connections_active', {
      description: 'Current active WebSocket connections',
    }).addCallback((result) => {
      for (const [namespace, count] of this.wsConnectionCounts) {
        result.observe(count, { namespace });
      }
    });

    this.wsMessages = meter.createCounter('app_ws_messages_total', {
      description: 'Total WebSocket messages',
    });

    this.wsConnectionDuration = meter.createHistogram('app_ws_connection_duration_seconds', {
      description: 'Duration of WebSocket connections in seconds',
      unit: 's',
      advice: {
        explicitBucketBoundaries: [1, 5, 10, 30, 60, 300, 600, 1800, 3600],
      },
    });

    // --- Rate Limiter ---
    this.rateLimitHits = meter.createCounter('app_rate_limit_hits_total', {
      description: 'Total rate limit rejections',
    });

    meter.createObservableGauge('app_rate_limit_remaining', {
      description: 'Sampled remaining rate limit tokens',
    }).addCallback((result) => {
      for (const [limiter, remaining] of this.rateLimitRemaining) {
        result.observe(remaining, { limiter });
      }
    });

    // --- Multi-Tenant ---
    this.tenantRequests = meter.createCounter('app_tenant_requests_total', {
      description: 'Total requests by tenant tier',
    });

    meter.createObservableGauge('app_tenant_resource_usage', {
      description: 'Resource usage by tenant tier',
    }).addCallback((result) => {
      for (const [key, value] of this.tenantResourceUsages) {
        const [tier, resourceType] = key.split('::');
        result.observe(value, { tenant_tier: tier, resource_type: resourceType });
      }
    });

    // --- Lifecycle ---
    this.shutdownDuration = meter.createHistogram('app_shutdown_duration_seconds', {
      description: 'Graceful shutdown duration in seconds',
      unit: 's',
      advice: {
        explicitBucketBoundaries: [0.5, 1, 2, 5, 10, 15, 30],
      },
    });

    meter.createObservableGauge('app_startup_duration_seconds', {
      description: 'Application startup duration in seconds',
      unit: 's',
    }).addCallback((result) => {
      if (this.startupDuration !== null) {
        result.observe(this.startupDuration);
      }
    });

    // App info gauge (always 1, attributes carry metadata)
    meter.createObservableGauge('app_info', {
      description: 'Application metadata (always 1, attributes carry info)',
    }).addCallback((result) => {
      if (this.appInfoAttributes) {
        result.observe(1, this.appInfoAttributes);
      }
    });
  }

  // --- Setters for observable gauges ---

  setWsConnectionCount(namespace: string, count: number) {
    this.wsConnectionCounts.set(namespace, count);
  }

  incrementWsConnections(namespace: string) {
    const current = this.wsConnectionCounts.get(namespace) || 0;
    this.wsConnectionCounts.set(namespace, current + 1);
  }

  decrementWsConnections(namespace: string) {
    const current = this.wsConnectionCounts.get(namespace) || 0;
    this.wsConnectionCounts.set(namespace, Math.max(0, current - 1));
  }

  setRateLimitRemaining(limiter: string, remaining: number) {
    this.rateLimitRemaining.set(limiter, remaining);
  }

  setTenantResourceUsage(tenantTier: string, resourceType: string, value: number) {
    this.tenantResourceUsages.set(`${tenantTier}::${resourceType}`, value);
  }

  setStartupDuration(seconds: number) {
    this.startupDuration = seconds;
  }

  setAppInfo(version: string, commitSha: string, environment: string) {
    this.appInfoAttributes = { version, commit_sha: commitSha, environment };
  }
}
```

Register `Stage4Instruments` in the `MetricsModule`.

---

## 2. WebSocket Metrics <a id="websocket"></a>

For Socket.IO or raw WebSocket gateways in NestJS.

```typescript
// src/telemetry/ws-metrics.service.ts
import { Injectable } from '@nestjs/common';
import { Socket } from 'socket.io';
import { Stage4Instruments } from './instruments/stage4.instruments';

const connectionStartTimes = new Map<string, bigint>();

@Injectable()
export class WsMetricsService {
  constructor(private readonly instruments: Stage4Instruments) {}

  handleConnection(client: Socket, namespace: string = '/') {
    connectionStartTimes.set(client.id, process.hrtime.bigint());
    this.instruments.incrementWsConnections(namespace);
  }

  handleDisconnect(client: Socket, namespace: string = '/') {
    this.instruments.decrementWsConnections(namespace);

    const startTime = connectionStartTimes.get(client.id);
    if (startTime) {
      const duration = Number(process.hrtime.bigint() - startTime) / 1e9;
      this.instruments.wsConnectionDuration.record(duration);
      connectionStartTimes.delete(client.id);
    }
  }

  recordMessage(namespace: string, direction: 'in' | 'out', eventType: string) {
    this.instruments.wsMessages.add(1, { namespace, direction, event_type: eventType });
  }
}

// Usage in your gateway:
// @WebSocketGateway({ namespace: '/chat' })
// export class ChatGateway implements OnGatewayConnection, OnGatewayDisconnect {
//   constructor(private wsMetrics: WsMetricsService) {}
//
//   handleConnection(client: Socket) {
//     this.wsMetrics.handleConnection(client, '/chat');
//   }
//
//   handleDisconnect(client: Socket) {
//     this.wsMetrics.handleDisconnect(client, '/chat');
//   }
//
//   @SubscribeMessage('message')
//   handleMessage(client: Socket, payload: any) {
//     this.wsMetrics.recordMessage('/chat', 'in', 'message');
//     // ... handle
//     this.wsMetrics.recordMessage('/chat', 'out', 'message_ack');
//   }
// }
```

---

## 3. Rate Limiter Observability <a id="rate-limiter"></a>

Without metrics, you don't know if your rate limiter is protecting you or hurting legitimate users.

```typescript
// Wrap any rate limiter's check method
async checkRateLimit(
  key: string,
  limiterName: string,
  route: string,
): Promise<{ allowed: boolean; remaining: number }> {
  const result = await this.rateLimiter.check(key);

  if (!result.allowed) {
    this.instruments.rateLimitHits.add(1, { limiter: limiterName, route });
  }

  // Sample remaining tokens. Don't do this per-request at high scale.
  // Sample every Nth request or use a timer.
  this.instruments.setRateLimitRemaining(limiterName, result.remaining);

  return result;
}
```

For `@nestjs/throttler`, extend the guard:

```typescript
@Injectable()
export class InstrumentedThrottlerGuard extends ThrottlerGuard {
  constructor(private readonly instruments: Stage4Instruments) {
    super(/* deps */);
  }

  protected async throwThrottlingException(context: ExecutionContext): Promise<void> {
    const request = context.switchToHttp().getRequest();
    const route = request.route?.path || request.path || 'unknown';
    this.instruments.rateLimitHits.add(1, { limiter: 'default', route });
    throw new ThrottlerException();
  }
}
```

---

## 4. Multi-Tenant Metrics <a id="multi-tenant"></a>

Track by tenant tier, not tenant ID. Tenant ID as an attribute = unbounded cardinality.

```typescript
// src/telemetry/tenant-metrics.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { Stage4Instruments } from './instruments/stage4.instruments';

@Injectable()
export class TenantMetricsInterceptor implements NestInterceptor {
  constructor(private readonly instruments: Stage4Instruments) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const tenantTier = request.tenant?.tier || 'unknown';
    this.instruments.tenantRequests.add(1, { tenant_tier: tenantTier });
    return next.handle();
  }
}
```

For resource usage, update via a periodic job:

```typescript
@Cron('0 */5 * * * *')
async updateTenantResourceMetrics() {
  const tiers = ['free', 'starter', 'pro', 'enterprise'];
  for (const tier of tiers) {
    const stats = await this.tenantService.getAggregatedResourceUsage(tier);
    this.instruments.setTenantResourceUsage(tier, 'storage_bytes', stats.totalStorageBytes);
    this.instruments.setTenantResourceUsage(tier, 'api_calls_today', stats.totalApiCallsToday);
  }
}
```

---

## 5. App Lifecycle Metrics <a id="lifecycle"></a>

```typescript
// src/telemetry/lifecycle-metrics.service.ts
import { Injectable, OnApplicationBootstrap } from '@nestjs/common';
import { Stage4Instruments } from './instruments/stage4.instruments';

// Set this as early as possible in main.ts, before NestFactory.create()
export const APP_START_TIME = process.hrtime.bigint();

@Injectable()
export class LifecycleMetricsService implements OnApplicationBootstrap {
  constructor(private readonly instruments: Stage4Instruments) {}

  onApplicationBootstrap() {
    const duration = Number(process.hrtime.bigint() - APP_START_TIME) / 1e9;
    this.instruments.setStartupDuration(duration);
    this.instruments.setAppInfo(
      process.env.APP_VERSION || 'unknown',
      process.env.COMMIT_SHA || 'unknown',
      process.env.NODE_ENV || 'development',
    );
  }

  async recordShutdown(shutdownFn: () => Promise<void>) {
    const start = process.hrtime.bigint();
    await shutdownFn();
    const duration = Number(process.hrtime.bigint() - start) / 1e9;
    this.instruments.shutdownDuration.record(duration);
    // Note: This may not be pushed before the process exits.
    // Ensure your MeterProvider.shutdown() is called after this
    // to flush pending metrics.
  }
}
```

---

## 6. Exemplar Support (Connecting Metrics to Traces) <a id="exemplars"></a>

Exemplars attach a trace ID to a metric sample. In Grafana, this lets you click a spike in a metric chart and jump directly to the trace that caused it.

The OTel SDK supports exemplars natively when a TracerProvider is active in the same process. If you have tracing configured (out of scope for this skill, but relevant to mention), exemplars are attached automatically to histogram recordings when a span is active.

To verify: check Prometheus with exemplar storage enabled, and Grafana configured with the exemplar data source pointing to your trace backend (Tempo, Jaeger, etc.).

---

## 7. Canary Deployment Metrics <a id="canary"></a>

The `app_info` gauge enables canary analysis. When you deploy a new version alongside the old one, compare their metrics.

```promql
# Compare error rates between canary and stable
# Join on instance labels

# Error rate for current version
sum(rate(http_requests_total{status_code=~"5.."}[5m]))
  * on(instance) group_left(version)
  app_info{version="1.2.0"}

# vs previous version
sum(rate(http_requests_total{status_code=~"5.."}[5m]))
  * on(instance) group_left(version)
  app_info{version="1.1.0"}
```

---

## 8. Key PromQL Queries for Stage 4 <a id="promql"></a>

```promql
# Active WebSocket connections by namespace
app_ws_connections_active

# WebSocket message throughput
sum(rate(app_ws_messages_total[5m])) by (namespace, direction)

# Median WebSocket connection duration
histogram_quantile(0.5,
  sum(rate(app_ws_connection_duration_seconds_bucket[1h])) by (le)
)

# Rate limit rejection rate per route
topk(10,
  sum(rate(app_rate_limit_hits_total[5m])) by (route)
)

# Resource usage by tenant tier
app_tenant_resource_usage{resource_type="api_calls_today"}

# Startup time (detect slow deploys)
app_startup_duration_seconds

# App version distribution across instances
count by (version) (app_info)
```
