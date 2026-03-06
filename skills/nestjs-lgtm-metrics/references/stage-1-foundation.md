# Stage 1: Foundation Metrics

Full implementation for HTTP RED metrics, Node.js runtime metrics, and health check metrics using the OpenTelemetry SDK.

## Table of Contents
1. [Create Instruments](#instruments)
2. [HTTP Metrics Interceptor](#http-interceptor)
3. [Event Loop Lag Monitoring](#event-loop)
4. [Health Check Metrics](#health-checks)
5. [Testing Your Setup](#testing)

---

## 1. Create Instruments <a id="instruments"></a>

All instruments are created from the `Meter` instance provided by the `MetricsModule` (see `otel-setup.md`). Create them once in a service and reuse across the app.

```typescript
// src/telemetry/instruments/http.instruments.ts
import { Inject, Injectable } from '@nestjs/common';
import { Meter, Counter, Histogram, UpDownCounter } from '@opentelemetry/api';
import { METER_TOKEN } from '../metrics.module';

@Injectable()
export class HttpInstruments {
  public readonly requestDuration: Histogram;
  public readonly requestsTotal: Counter;
  public readonly requestsInFlight: UpDownCounter;

  constructor(@Inject(METER_TOKEN) meter: Meter) {
    this.requestDuration = meter.createHistogram('http_request_duration_seconds', {
      description: 'Duration of HTTP requests in seconds',
      unit: 's',
      advice: {
        explicitBucketBoundaries: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10],
      },
    });

    this.requestsTotal = meter.createCounter('http_requests_total', {
      description: 'Total number of HTTP requests',
    });

    // UpDownCounter is the OTel equivalent of a Prometheus gauge
    // used for tracking values that go up and down.
    this.requestsInFlight = meter.createUpDownCounter('http_requests_in_flight', {
      description: 'Number of HTTP requests currently being processed',
    });
  }
}
```

```typescript
// src/telemetry/instruments/runtime.instruments.ts
import { Inject, Injectable } from '@nestjs/common';
import { Meter, Histogram } from '@opentelemetry/api';
import { METER_TOKEN } from '../metrics.module';

@Injectable()
export class RuntimeInstruments {
  public readonly eventLoopLag: Histogram;

  constructor(@Inject(METER_TOKEN) meter: Meter) {
    this.eventLoopLag = meter.createHistogram('nodejs_eventloop_lag_seconds', {
      description: 'Event loop lag in seconds',
      unit: 's',
      advice: {
        explicitBucketBoundaries: [0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1],
      },
    });
  }
}
```

```typescript
// src/telemetry/instruments/health.instruments.ts
import { Inject, Injectable } from '@nestjs/common';
import { Meter, Histogram, ObservableGauge } from '@opentelemetry/api';
import { METER_TOKEN } from '../metrics.module';

// For the health status gauge, OTel uses ObservableGauge with a callback.
// But for simplicity and since health checks are event-driven (not polled),
// we use a regular approach: store state and report via observable callback.

@Injectable()
export class HealthInstruments {
  public readonly checkDuration: Histogram;
  private healthStatuses = new Map<string, number>();

  constructor(@Inject(METER_TOKEN) meter: Meter) {
    this.checkDuration = meter.createHistogram('app_health_check_duration_seconds', {
      description: 'Duration of health check probes in seconds',
      unit: 's',
      advice: {
        explicitBucketBoundaries: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1],
      },
    });

    // ObservableGauge reports the current health state on each metric collection.
    meter.createObservableGauge('app_health_check_status', {
      description: 'Health check status (1=healthy, 0=unhealthy)',
    }).addCallback((observableResult) => {
      for (const [checkName, status] of this.healthStatuses) {
        observableResult.observe(status, { check_name: checkName });
      }
    });
  }

  setHealthStatus(checkName: string, healthy: boolean) {
    this.healthStatuses.set(checkName, healthy ? 1 : 0);
  }
}
```

Register all instruments in the MetricsModule:

```typescript
// src/telemetry/metrics.module.ts (updated)
import { Global, Module } from '@nestjs/common';
import { metrics, Meter } from '@opentelemetry/api';
import { HttpInstruments } from './instruments/http.instruments';
import { RuntimeInstruments } from './instruments/runtime.instruments';
import { HealthInstruments } from './instruments/health.instruments';

export const METER_TOKEN = 'OTEL_METER';

@Global()
@Module({
  providers: [
    {
      provide: METER_TOKEN,
      useFactory: (): Meter => metrics.getMeter('nestjs-app', '1.0.0'),
    },
    HttpInstruments,
    RuntimeInstruments,
    HealthInstruments,
  ],
  exports: [METER_TOKEN, HttpInstruments, RuntimeInstruments, HealthInstruments],
})
export class MetricsModule {}
```

---

## 2. HTTP Metrics Interceptor <a id="http-interceptor"></a>

Captures every HTTP request. Uses the route pattern (not the full URL) to avoid cardinality explosion.

```typescript
// src/telemetry/http-metrics.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable, tap } from 'rxjs';
import { Request, Response } from 'express';
import { HttpInstruments } from './instruments/http.instruments';

@Injectable()
export class HttpMetricsInterceptor implements NestInterceptor {
  constructor(private readonly httpInstruments: HttpInstruments) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const httpContext = context.switchToHttp();
    const request = httpContext.getRequest<Request>();
    const method = request.method;

    // Use the route pattern, NOT the full URL.
    // request.route?.path gives you '/users/:id' instead of '/users/12345'
    const route = request.route?.path || request.path || 'unknown';

    this.httpInstruments.requestsInFlight.add(1, { method });
    const startTime = process.hrtime.bigint();

    return next.handle().pipe(
      tap({
        next: () => {
          const statusCode = httpContext.getResponse<Response>().statusCode;
          this.recordMetrics(method, route, statusCode, startTime);
          this.httpInstruments.requestsInFlight.add(-1, { method });
        },
        error: (error) => {
          const statusCode = error?.status || error?.statusCode || 500;
          this.recordMetrics(method, route, statusCode, startTime);
          this.httpInstruments.requestsInFlight.add(-1, { method });
        },
      }),
    );
  }

  private recordMetrics(
    method: string,
    route: string,
    statusCode: number,
    startTime: bigint,
  ): void {
    const durationSeconds = Number(process.hrtime.bigint() - startTime) / 1e9;
    const attributes = {
      method,
      route,
      status_code: String(statusCode),
    };

    this.httpInstruments.requestDuration.record(durationSeconds, attributes);
    this.httpInstruments.requestsTotal.add(1, attributes);
  }
}
```

Register globally in `main.ts`:
```typescript
// src/main.ts (add after NestFactory.create)
const metricsInterceptor = app.get(HttpMetricsInterceptor);
app.useGlobalInterceptors(metricsInterceptor);
```

**Important**: If you use `APP_GUARD` for auth, the interceptor runs after guards. Failed auth requests (401/403) are still captured because the `error` handler in `tap` catches them.

---

## 3. Event Loop Lag Monitoring <a id="event-loop"></a>

The event loop lag is the most important Node.js-specific metric. It tells you when your process is overloaded before it becomes visible in HTTP latency.

```typescript
// src/telemetry/eventloop-metrics.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { monitorEventLoopDelay } from 'node:perf_hooks';
import { RuntimeInstruments } from './instruments/runtime.instruments';

@Injectable()
export class EventLoopMetricsService implements OnModuleInit, OnModuleDestroy {
  private histogram: ReturnType<typeof monitorEventLoopDelay> | null = null;
  private interval: NodeJS.Timeout | null = null;

  constructor(private readonly runtimeInstruments: RuntimeInstruments) {}

  onModuleInit() {
    // Resolution of 20ms is sufficient for alerting.
    this.histogram = monitorEventLoopDelay({ resolution: 20 });
    this.histogram.enable();

    // Sample every 5 seconds.
    this.interval = setInterval(() => {
      if (this.histogram) {
        // Convert from nanoseconds to seconds
        const lagSeconds = this.histogram.mean / 1e9;
        this.runtimeInstruments.eventLoopLag.record(lagSeconds);
        this.histogram.reset();
      }
    }, 5000);
  }

  onModuleDestroy() {
    if (this.interval) clearInterval(this.interval);
    if (this.histogram) this.histogram.disable();
  }
}
```

Add `EventLoopMetricsService` to the MetricsModule providers.

---

## 4. Health Check Metrics <a id="health-checks"></a>

If you use `@nestjs/terminus` for health checks, wrap each check with metric instrumentation.

```typescript
// src/health/health.controller.ts
import { Controller, Get } from '@nestjs/common';
import {
  HealthCheck,
  HealthCheckService,
  TypeOrmHealthIndicator,
} from '@nestjs/terminus';
import { HealthInstruments } from '../telemetry/instruments/health.instruments';

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: TypeOrmHealthIndicator,
    private healthInstruments: HealthInstruments,
  ) {}

  @Get()
  @HealthCheck()
  async check() {
    const result = await this.health.check([
      () => this.instrumentedCheck('database', () => this.db.pingCheck('database')),
      // Add more checks as needed
    ]);
    return result;
  }

  private async instrumentedCheck(
    checkName: string,
    checkFn: () => Promise<any>,
  ) {
    const startTime = process.hrtime.bigint();
    try {
      const result = await checkFn();
      this.healthInstruments.setHealthStatus(checkName, true);
      return result;
    } catch (error) {
      this.healthInstruments.setHealthStatus(checkName, false);
      throw error;
    } finally {
      const duration = Number(process.hrtime.bigint() - startTime) / 1e9;
      this.healthInstruments.checkDuration.record(duration, { check_name: checkName });
    }
  }
}
```

---

## 5. Testing Your Setup <a id="testing"></a>

After implementing, verify:

1. **Check the Collector**: The Collector should log incoming metric batches. If using the prometheus exporter on the Collector, `curl http://otel-collector:8889/metrics` should show your app's metrics.

2. **Send some requests** and check that `http_request_duration_seconds_bucket` lines appear with your route attributes.

3. **Check Prometheus targets**: In the Prometheus UI (`http://prometheus:9090/targets`), confirm the Collector target is UP.

4. **Grafana test query**: In Grafana Explore, run:
   ```promql
   rate(http_requests_total[5m])
   ```

5. **Event loop lag baseline**: A healthy Node.js process has event loop lag under 10ms. If you see > 50ms at idle, something is blocking the loop.

**Key PromQL queries for Stage 1 dashboards:**

```promql
# Request rate by route
sum(rate(http_requests_total[5m])) by (route, method)

# Error rate (5xx)
sum(rate(http_requests_total{status_code=~"5.."}[5m]))
  /
sum(rate(http_requests_total[5m]))

# P99 latency by route
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, route))

# Event loop lag P95
histogram_quantile(0.95, sum(rate(nodejs_eventloop_lag_seconds_bucket[5m])) by (le))

# Requests in flight
sum(http_requests_in_flight)
```
