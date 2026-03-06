# Stage 2: Application Intelligence Metrics

Metrics that expose how your app experiences its dependencies and internal behaviors, using the OpenTelemetry SDK.

## Table of Contents
1. [Create Instruments](#instruments)
2. [Database Query Metrics](#database)
3. [Redis Operation Metrics](#redis)
4. [External API Call Metrics](#external-api)
5. [Auth Metrics](#auth)
6. [Background Job Metrics](#jobs)
7. [PromQL Queries](#promql)

---

## 1. Create Instruments <a id="instruments"></a>

All Stage 2 instruments in one injectable service. Inject the `Meter` from the global `MetricsModule`.

```typescript
// src/telemetry/instruments/stage2.instruments.ts
import { Inject, Injectable } from '@nestjs/common';
import { Meter, Counter, Histogram, UpDownCounter } from '@opentelemetry/api';
import { METER_TOKEN } from '../metrics.module';

@Injectable()
export class Stage2Instruments {
  // Database
  public readonly dbQueryDuration: Histogram;
  public readonly dbQueryErrors: Counter;
  public readonly dbPoolWait: Histogram;

  // Redis
  public readonly redisOpDuration: Histogram;
  public readonly redisOpErrors: Counter;
  public readonly cacheHit: Counter;
  public readonly cacheMiss: Counter;

  // External APIs
  public readonly externalApiDuration: Histogram;
  public readonly externalApiErrors: Counter;
  // For circuit breaker state, we use an ObservableGauge registered separately.

  // Auth
  public readonly authAttempts: Counter;
  public readonly authTokenOps: Counter;
  public readonly authzDenied: Counter;

  // Jobs
  public readonly jobDuration: Histogram;
  public readonly jobCompleted: Counter;
  public readonly jobAttempts: Counter;

  // Mutable state for observable gauges
  private circuitBreakerStates = new Map<string, number>();
  private queueDepths = new Map<string, number>();

  constructor(@Inject(METER_TOKEN) meter: Meter) {
    // --- Database ---
    this.dbQueryDuration = meter.createHistogram('app_db_query_duration_seconds', {
      description: 'Duration of database queries as observed by the application',
      unit: 's',
      advice: {
        explicitBucketBoundaries: [0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
      },
    });

    this.dbQueryErrors = meter.createCounter('app_db_query_errors_total', {
      description: 'Total database query errors observed by the application',
    });

    this.dbPoolWait = meter.createHistogram('app_db_connection_pool_wait_seconds', {
      description: 'Time spent waiting for a database connection from the pool',
      unit: 's',
      advice: {
        explicitBucketBoundaries: [0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1],
      },
    });

    // --- Redis ---
    this.redisOpDuration = meter.createHistogram('app_redis_operation_duration_seconds', {
      description: 'Duration of Redis operations as observed by the application',
      unit: 's',
      advice: {
        explicitBucketBoundaries: [0.0005, 0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25],
      },
    });

    this.redisOpErrors = meter.createCounter('app_redis_operation_errors_total', {
      description: 'Total Redis operation errors observed by the application',
    });

    this.cacheHit = meter.createCounter('app_cache_hit_total', {
      description: 'Total cache hits by cache name',
    });

    this.cacheMiss = meter.createCounter('app_cache_miss_total', {
      description: 'Total cache misses by cache name',
    });

    // --- External APIs ---
    this.externalApiDuration = meter.createHistogram('app_external_api_duration_seconds', {
      description: 'Duration of external API calls',
      unit: 's',
      advice: {
        explicitBucketBoundaries: [0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10, 30],
      },
    });

    this.externalApiErrors = meter.createCounter('app_external_api_errors_total', {
      description: 'Total external API call errors',
    });

    // Circuit breaker as ObservableGauge (polled on collection)
    meter.createObservableGauge('app_circuit_breaker_state', {
      description: 'Circuit breaker state (0=closed, 0.5=half-open, 1=open)',
    }).addCallback((result) => {
      for (const [service, state] of this.circuitBreakerStates) {
        result.observe(state, { service });
      }
    });

    // --- Auth ---
    this.authAttempts = meter.createCounter('app_auth_attempts_total', {
      description: 'Total authentication attempts',
    });

    this.authTokenOps = meter.createCounter('app_auth_token_operations_total', {
      description: 'Total auth token operations',
    });

    this.authzDenied = meter.createCounter('app_authorization_denied_total', {
      description: 'Total authorization denials',
    });

    // --- Jobs ---
    this.jobDuration = meter.createHistogram('app_job_duration_seconds', {
      description: 'Duration of background job processing',
      unit: 's',
      advice: {
        explicitBucketBoundaries: [0.1, 0.5, 1, 5, 10, 30, 60, 120, 300],
      },
    });

    this.jobCompleted = meter.createCounter('app_job_completed_total', {
      description: 'Total completed background jobs',
    });

    this.jobAttempts = meter.createCounter('app_job_attempts_total', {
      description: 'Total job processing attempts (tracks retries)',
    });

    // Queue depth as ObservableGauge
    meter.createObservableGauge('app_job_queue_depth', {
      description: 'Current number of jobs waiting in each queue',
    }).addCallback((result) => {
      for (const [queue, depth] of this.queueDepths) {
        result.observe(depth, { queue });
      }
    });
  }

  // --- Setters for observable gauges ---

  setCircuitBreakerState(service: string, state: 'closed' | 'half-open' | 'open') {
    const value = { closed: 0, 'half-open': 0.5, open: 1 }[state];
    this.circuitBreakerStates.set(service, value);
  }

  setQueueDepth(queue: string, depth: number) {
    this.queueDepths.set(queue, depth);
  }
}
```

Register `Stage2Instruments` in the `MetricsModule` providers and exports.

---

## 2. Database Query Metrics <a id="database"></a>

### TypeORM Subscriber

TypeORM subscribers intercept every write operation. This gives you app-side timing without modifying every repository.

```typescript
// src/telemetry/subscribers/typeorm-metrics.subscriber.ts
import {
  EntitySubscriberInterface,
  EventSubscriber,
  InsertEvent,
  UpdateEvent,
  RemoveEvent,
} from 'typeorm';
import { Injectable } from '@nestjs/common';
import { Stage2Instruments } from '../instruments/stage2.instruments';

@Injectable()
@EventSubscriber()
export class TypeOrmMetricsSubscriber implements EntitySubscriberInterface {
  constructor(private readonly instruments: Stage2Instruments) {}

  beforeInsert(event: InsertEvent<any>) {
    (event as any).__startTime = process.hrtime.bigint();
  }

  afterInsert(event: InsertEvent<any>) {
    this.recordDuration('insert', event.metadata.tableName, (event as any).__startTime);
  }

  beforeUpdate(event: UpdateEvent<any>) {
    (event as any).__startTime = process.hrtime.bigint();
  }

  afterUpdate(event: UpdateEvent<any>) {
    this.recordDuration('update', event.metadata.tableName, (event as any).__startTime);
  }

  beforeRemove(event: RemoveEvent<any>) {
    (event as any).__startTime = process.hrtime.bigint();
  }

  afterRemove(event: RemoveEvent<any>) {
    this.recordDuration('delete', event.metadata.tableName, (event as any).__startTime);
  }

  private recordDuration(operation: string, entity: string, startTime: bigint) {
    if (!startTime) return;
    const duration = Number(process.hrtime.bigint() - startTime) / 1e9;
    this.instruments.dbQueryDuration.record(duration, { operation, entity });
  }
}
```

### Pool Wait Time (the hidden metric)

Neither postgres-exporter nor TypeORM default logging gives you this. When your pool is saturated, requests wait silently.

```typescript
// Wrap DataSource pool acquisition to measure actual wait duration.
// TypeORM + pg specific. Adapt for your driver.
async function instrumentedGetConnection(
  dataSource: DataSource,
  instruments: Stage2Instruments,
) {
  const start = process.hrtime.bigint();
  const queryRunner = dataSource.createQueryRunner();
  await queryRunner.connect();
  const waitDuration = Number(process.hrtime.bigint() - start) / 1e9;
  instruments.dbPoolWait.record(waitDuration);
  return queryRunner;
}
```

---

## 3. Redis Operation Metrics <a id="redis"></a>

Wrap your Redis client or cache service.

```typescript
// src/telemetry/instrumented-cache.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRedis } from '@nestjs-modules/ioredis'; // or your Redis module
import Redis from 'ioredis';
import { Stage2Instruments } from './instruments/stage2.instruments';

@Injectable()
export class InstrumentedCacheService {
  constructor(
    @InjectRedis() private readonly redis: Redis,
    private readonly instruments: Stage2Instruments,
  ) {}

  /**
   * cacheName identifies WHICH caching strategy this is part of.
   * keyPrefix groups related keys for the attribute (e.g., 'user', 'session', 'config').
   */
  async get(cacheName: string, key: string): Promise<string | null> {
    const keyPrefix = this.extractPrefix(key);
    const start = process.hrtime.bigint();

    try {
      const value = await this.redis.get(key);
      const duration = Number(process.hrtime.bigint() - start) / 1e9;
      this.instruments.redisOpDuration.record(duration, { operation: 'get', key_prefix: keyPrefix });

      if (value !== null) {
        this.instruments.cacheHit.add(1, { cache_name: cacheName });
      } else {
        this.instruments.cacheMiss.add(1, { cache_name: cacheName });
      }
      return value;
    } catch (error) {
      this.instruments.redisOpErrors.add(1, {
        operation: 'get',
        error_type: error.constructor?.name || 'UnknownError',
      });
      throw error;
    }
  }

  async set(key: string, value: string, ttlSeconds?: number): Promise<void> {
    const keyPrefix = this.extractPrefix(key);
    const start = process.hrtime.bigint();

    try {
      if (ttlSeconds) {
        await this.redis.setex(key, ttlSeconds, value);
      } else {
        await this.redis.set(key, value);
      }
      const duration = Number(process.hrtime.bigint() - start) / 1e9;
      this.instruments.redisOpDuration.record(duration, { operation: 'set', key_prefix: keyPrefix });
    } catch (error) {
      this.instruments.redisOpErrors.add(1, {
        operation: 'set',
        error_type: error.constructor?.name || 'UnknownError',
      });
      throw error;
    }
  }

  async del(key: string): Promise<void> {
    const keyPrefix = this.extractPrefix(key);
    const start = process.hrtime.bigint();

    try {
      await this.redis.del(key);
      const duration = Number(process.hrtime.bigint() - start) / 1e9;
      this.instruments.redisOpDuration.record(duration, { operation: 'del', key_prefix: keyPrefix });
    } catch (error) {
      this.instruments.redisOpErrors.add(1, {
        operation: 'del',
        error_type: error.constructor?.name || 'UnknownError',
      });
      throw error;
    }
  }

  /**
   * Extract a prefix from the key for labeling.
   * e.g., 'user:123:profile' -> 'user'
   * Keep this deterministic and low-cardinality.
   */
  private extractPrefix(key: string): string {
    const parts = key.split(':');
    return parts[0] || 'unknown';
  }
}
```

---

## 4. External API Call Metrics <a id="external-api"></a>

Wrap your HTTP client (Axios, fetch, etc.).

```typescript
// src/telemetry/instrumented-http.service.ts
import { Injectable } from '@nestjs/common';
import { HttpService } from '@nestjs/axios';
import { firstValueFrom } from 'rxjs';
import { AxiosRequestConfig } from 'axios';
import { Stage2Instruments } from './instruments/stage2.instruments';

@Injectable()
export class InstrumentedHttpService {
  constructor(
    private readonly httpService: HttpService,
    private readonly instruments: Stage2Instruments,
  ) {}

  /**
   * @param service - The service name (e.g., 'payment-gateway', 'email-service')
   * @param endpoint - A low-cardinality endpoint label (e.g., '/charge', '/send')
   */
  async request<T>(
    service: string,
    endpoint: string,
    config: AxiosRequestConfig,
  ): Promise<T> {
    const start = process.hrtime.bigint();

    try {
      const response = await firstValueFrom(this.httpService.request<T>(config));
      const duration = Number(process.hrtime.bigint() - start) / 1e9;
      this.instruments.externalApiDuration.record(duration, {
        service,
        endpoint,
        status_code: String(response.status),
      });
      return response.data;
    } catch (error) {
      const duration = Number(process.hrtime.bigint() - start) / 1e9;
      const statusCode = error.response?.status || 0;
      this.instruments.externalApiDuration.record(duration, {
        service,
        endpoint,
        status_code: String(statusCode),
      });
      this.instruments.externalApiErrors.add(1, {
        service,
        error_type: statusCode ? `http_${statusCode}` : error.code || 'unknown',
      });
      throw error;
    }
  }

  /**
   * Call this whenever your circuit breaker transitions state.
   * Integrate with your circuit breaker library (opossum, cockatiel, etc.)
   */
  setCircuitBreakerState(service: string, state: 'closed' | 'half-open' | 'open') {
    this.instruments.setCircuitBreakerState(service, state);
  }
}
```

---

## 5. Auth Metrics <a id="auth"></a>

```typescript
// src/auth/auth-metrics.service.ts
import { Injectable } from '@nestjs/common';
import { Stage2Instruments } from '../telemetry/instruments/stage2.instruments';

@Injectable()
export class AuthMetricsService {
  constructor(private readonly instruments: Stage2Instruments) {}

  recordAuthAttempt(method: string, result: 'success' | 'failure' | 'locked') {
    this.instruments.authAttempts.add(1, { method, result });
  }

  recordTokenOperation(operation: 'issue' | 'refresh' | 'revoke' | 'expire') {
    this.instruments.authTokenOps.add(1, { operation });
  }

  recordAuthorizationDenied(resource: string, action: string) {
    this.instruments.authzDenied.add(1, { resource, action });
  }
}
```

---

## 6. Background Job Metrics <a id="jobs"></a>

```typescript
// src/telemetry/job-metrics.service.ts
import { Injectable } from '@nestjs/common';
import { Stage2Instruments } from './instruments/stage2.instruments';

@Injectable()
export class JobMetricsService {
  constructor(private readonly instruments: Stage2Instruments) {}

  recordJobCompletion(
    queue: string,
    jobType: string,
    status: 'completed' | 'failed' | 'stalled',
    durationSeconds: number,
  ) {
    this.instruments.jobDuration.record(durationSeconds, { queue, job_type: jobType });
    this.instruments.jobCompleted.add(1, { queue, job_type: jobType, status });
  }

  recordJobAttempt(queue: string, jobType: string) {
    this.instruments.jobAttempts.add(1, { queue, job_type: jobType });
  }

  updateQueueDepth(queue: string, depth: number) {
    this.instruments.setQueueDepth(queue, depth);
  }
}

// Integration with BullMQ processor:
// @Processor('email-queue')
// export class EmailProcessor extends WorkerHost {
//   constructor(private jobMetrics: JobMetricsService) { super(); }
//
//   async process(job: Job) {
//     this.jobMetrics.recordJobAttempt('email', job.name);
//     const start = process.hrtime.bigint();
//     try {
//       await this.handleEmail(job);
//       const duration = Number(process.hrtime.bigint() - start) / 1e9;
//       this.jobMetrics.recordJobCompletion('email', job.name, 'completed', duration);
//     } catch (error) {
//       const duration = Number(process.hrtime.bigint() - start) / 1e9;
//       this.jobMetrics.recordJobCompletion('email', job.name, 'failed', duration);
//       throw error;
//     }
//   }
// }
```

---

## 7. Key PromQL Queries for Stage 2 <a id="promql"></a>

```promql
# App-side DB query P95 latency by entity
histogram_quantile(0.95,
  sum(rate(app_db_query_duration_seconds_bucket[5m])) by (le, entity, operation)
)

# DB pool wait time P99 (the hidden bottleneck)
histogram_quantile(0.99,
  sum(rate(app_db_connection_pool_wait_seconds_bucket[5m])) by (le)
)

# Cache hit ratio per cache strategy
sum(rate(app_cache_hit_total[5m])) by (cache_name)
  /
(sum(rate(app_cache_hit_total[5m])) by (cache_name)
  + sum(rate(app_cache_miss_total[5m])) by (cache_name))

# External API error rate by service
sum(rate(app_external_api_errors_total[5m])) by (service)

# Circuit breakers currently open
app_circuit_breaker_state == 1

# Auth failure rate
sum(rate(app_auth_attempts_total{result="failure"}[5m]))
  /
sum(rate(app_auth_attempts_total[5m]))

# Job failure rate by queue
sum(rate(app_job_completed_total{status="failed"}[5m])) by (queue)
  /
sum(rate(app_job_completed_total[5m])) by (queue)

# Queue depth (stale jobs building up)
app_job_queue_depth > 100
```
