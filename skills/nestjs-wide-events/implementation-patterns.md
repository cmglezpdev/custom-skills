# NestJS Wide Events: Implementation Patterns

Complete code reference for implementing the wide event pattern in NestJS. Copy and adapt these to your project.

## Table of Contents

1. [WideEventService](#wideeventservice)
2. [WideEventInterceptor](#wideeventinterceptor)
3. [WideEventExceptionFilter](#wideeventexceptionfilter)
4. [TailSampler](#tailsampler)
5. [WideEventModule](#wideeventmodule)
6. [App Bootstrap](#app-bootstrap)
7. [Usage in Controllers](#usage-in-controllers)
8. [Usage in Services](#usage-in-services)
9. [Auth Guard Integration](#auth-guard-integration)
10. [AsyncLocalStorage Alternative](#asynclocalstorage-alternative)
11. [Queue / Cron Job Pattern](#queue--cron-job-pattern)
12. [Pino Integration](#pino-integration)
13. [Testing](#testing)

---

## WideEventService

Request-scoped service that holds the mutable event context. Every piece of code that wants to add context to the current request's wide event injects this.

```typescript
// src/wide-event/wide-event.service.ts
import { Injectable, Scope } from '@nestjs/common';

export interface WideEvent {
  timestamp: string;
  request_id: string;
  trace_id?: string;

  // Request envelope
  method?: string;
  path?: string;
  route?: string;
  query_params?: Record<string, unknown>;
  status_code?: number;
  duration_ms?: number;
  ip?: string;
  user_agent?: string;

  // Infrastructure
  service?: string;
  version?: string;
  deployment_id?: string;
  region?: string;
  node_env?: string;

  // User context
  user?: {
    id: string;
    subscription?: string;
    account_age_days?: number;
    role?: string;
    org_id?: string;
    org_name?: string;
    [key: string]: unknown;
  };

  // Error context
  error?: {
    type: string;
    code?: string;
    message: string;
    retriable?: boolean;
    stack?: string;
    [key: string]: unknown;
  };

  // Performance
  db?: {
    query_count?: number;
    total_ms?: number;
    slowest_query_ms?: number;
  };

  cache?: {
    hit?: boolean;
    key?: string;
    latency_ms?: number;
  };

  external_calls?: Array<{
    service: string;
    duration_ms: number;
    status?: number;
    error?: string;
  }>;

  // Feature flags
  feature_flags?: Record<string, boolean>;

  // Outcome
  outcome?: 'success' | 'error' | 'client_error';
  level?: 'info' | 'warn' | 'error';

  // Open-ended business context
  [key: string]: unknown;
}

@Injectable({ scope: Scope.REQUEST })
export class WideEventService {
  private event: WideEvent;

  constructor() {
    this.event = {
      timestamp: new Date().toISOString(),
      request_id: this.generateRequestId(),
    };
  }

  /**
   * Add fields to the wide event. Deep merges nested objects one level deep.
   * Call this from anywhere in the request lifecycle to attach context.
   */
  enrich(fields: Partial<WideEvent> & Record<string, unknown>): void {
    for (const [key, value] of Object.entries(fields)) {
      if (
        value !== null &&
        typeof value === 'object' &&
        !Array.isArray(value) &&
        this.event[key] &&
        typeof this.event[key] === 'object' &&
        !Array.isArray(this.event[key])
      ) {
        // Shallow merge for nested objects (user, error, db, cache, etc.)
        (this.event as any)[key] = { ...(this.event[key] as any), ...value };
      } else {
        (this.event as any)[key] = value;
      }
    }
  }

  /**
   * Append an external call record to the external_calls array.
   */
  addExternalCall(call: {
    service: string;
    duration_ms: number;
    status?: number;
    error?: string;
  }): void {
    if (!this.event.external_calls) {
      this.event.external_calls = [];
    }
    this.event.external_calls.push(call);
  }

  /**
   * Return the finalized event. Called only by the interceptor at the end.
   */
  finalize(): WideEvent {
    return { ...this.event };
  }

  private generateRequestId(): string {
    return `req_${Date.now().toString(36)}_${Math.random().toString(36).slice(2, 8)}`;
  }
}
```

---

## WideEventInterceptor

Global interceptor. Seeds the event at request start, finalizes and emits at request end. This is the ONLY place that emits the log line.

```typescript
// src/wide-event/wide-event.interceptor.ts
import {
  CallHandler,
  ExecutionContext,
  Injectable,
  Logger,
} from '@nestjs/common';
import { Observable, tap, catchError, throwError } from 'rxjs';
import { WideEventService } from './wide-event.service';
import { TailSampler } from './tail-sampler';
import { Request, Response } from 'express';

@Injectable()
export class WideEventInterceptor {
  private readonly logger = new Logger('WideEvent');

  constructor(private readonly tailSampler: TailSampler) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const startTime = Date.now();
    const httpCtx = context.switchToHttp();
    const req = httpCtx.getRequest<Request>();
    const res = httpCtx.getResponse<Response>();

    // Resolve the request-scoped WideEventService from the current context.
    // Because the interceptor itself is singleton-scoped (for performance),
    // we pull the request-scoped service from the request object where
    // NestJS attaches the DI container for the current request.
    const wideEventService: WideEventService = req['wideEventService'];
    if (!wideEventService) {
      // Fallback: if middleware did not attach it, proceed without wide event
      return next.handle();
    }

    // Seed with HTTP metadata
    wideEventService.enrich({
      method: req.method,
      path: req.originalUrl,
      route: req.route?.path,
      ip: req.ip || req.socket?.remoteAddress,
      user_agent: req.get('user-agent'),
      trace_id: req.get('x-trace-id') || req.get('traceparent')?.split('-')[1],
      service: process.env.SERVICE_NAME || 'unknown',
      version: process.env.SERVICE_VERSION || 'unknown',
      deployment_id: process.env.DEPLOYMENT_ID,
      region: process.env.REGION,
      node_env: process.env.NODE_ENV,
    });

    const emit = () => {
      const duration_ms = Date.now() - startTime;
      const status_code = res.statusCode;

      wideEventService.enrich({
        status_code,
        duration_ms,
        outcome:
          status_code >= 500
            ? 'error'
            : status_code >= 400
              ? 'client_error'
              : 'success',
        level:
          status_code >= 500 ? 'error' : status_code >= 400 ? 'warn' : 'info',
      });

      const event = wideEventService.finalize();

      if (this.tailSampler.shouldSample(event)) {
        // This is the ONE log line. Nothing else in the app should call logger directly.
        this.logger.log(JSON.stringify(event));
      }
    };

    return next.handle().pipe(
      tap(() => emit()),
      catchError((err) => {
        // Error enrichment may have already happened via the exception filter.
        // We still emit here because the interceptor is the final checkpoint.
        emit();
        return throwError(() => err);
      }),
    );
  }
}
```

---

## WideEventExceptionFilter

Catches exceptions and enriches the wide event with error details before the interceptor emits.

```typescript
// src/wide-event/wide-event-exception.filter.ts
import {
  ArgumentsHost,
  Catch,
  ExceptionFilter,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { Request, Response } from 'express';
import { WideEventService } from './wide-event.service';

@Catch()
export class WideEventExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost): void {
    const ctx = host.switchToHttp();
    const req = ctx.getRequest<Request>();
    const res = ctx.getResponse<Response>();
    const wideEventService: WideEventService | undefined =
      req['wideEventService'];

    const status =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;

    const errorPayload: Record<string, unknown> = {
      type:
        exception instanceof Error ? exception.constructor.name : 'UnknownError',
      message:
        exception instanceof Error ? exception.message : String(exception),
      retriable: false,
    };

    if (exception instanceof HttpException) {
      const response = exception.getResponse();
      if (typeof response === 'object' && response !== null) {
        errorPayload.code = (response as any).error || (response as any).code;
      }
    }

    // Include stack only in non-production for 500s
    if (
      status >= 500 &&
      exception instanceof Error &&
      process.env.NODE_ENV !== 'production'
    ) {
      errorPayload.stack = exception.stack;
    }

    if (wideEventService) {
      wideEventService.enrich({ error: errorPayload });
    }

    // Send the HTTP response
    const body =
      exception instanceof HttpException
        ? exception.getResponse()
        : { statusCode: status, message: 'Internal server error' };

    res.status(status).json(body);
  }
}
```

---

## TailSampler

Decides whether to persist an event. Always keeps errors, slow requests, VIP users.

```typescript
// src/wide-event/tail-sampler.ts
import { Injectable } from '@nestjs/common';
import { WideEvent } from './wide-event.service';

export interface SamplingConfig {
  /** Sample rate for healthy requests, 0.0 to 1.0. Default 0.05 (5%) */
  defaultRate: number;
  /** Duration threshold in ms. Requests slower than this are always kept. */
  slowThreshold_ms: number;
  /** User subscription tiers that are always kept */
  vipSubscriptions: string[];
  /** Feature flag keys that, if enabled, force the event to be kept */
  monitoredFlags: string[];
}

const DEFAULT_CONFIG: SamplingConfig = {
  defaultRate: 0.05,
  slowThreshold_ms: 2000,
  vipSubscriptions: ['enterprise', 'premium'],
  monitoredFlags: [],
};

@Injectable()
export class TailSampler {
  private config: SamplingConfig;

  constructor(config?: Partial<SamplingConfig>) {
    this.config = { ...DEFAULT_CONFIG, ...config };
  }

  shouldSample(event: WideEvent): boolean {
    // Always keep errors
    if (event.status_code && event.status_code >= 500) return true;
    if (event.error) return true;

    // Always keep slow requests
    if (
      event.duration_ms &&
      event.duration_ms > this.config.slowThreshold_ms
    ) {
      return true;
    }

    // Always keep VIP users
    if (
      event.user?.subscription &&
      this.config.vipSubscriptions.includes(
        event.user.subscription as string,
      )
    ) {
      return true;
    }

    // Always keep monitored feature flag rollouts
    if (event.feature_flags) {
      for (const flag of this.config.monitoredFlags) {
        if (event.feature_flags[flag]) return true;
      }
    }

    // Random sample the rest
    return Math.random() < this.config.defaultRate;
  }
}
```

---

## WideEventModule

Wires everything together. Import at the app root.

```typescript
// src/wide-event/wide-event.module.ts
import { Global, MiddlewareConsumer, Module, NestModule } from '@nestjs/common';
import { WideEventService } from './wide-event.service';
import { TailSampler } from './tail-sampler';
import { WideEventMiddleware } from './wide-event.middleware';

@Global()
@Module({
  providers: [
    WideEventService,
    {
      provide: TailSampler,
      useFactory: () =>
        new TailSampler({
          defaultRate: parseFloat(process.env.SAMPLING_RATE || '0.05'),
          slowThreshold_ms: parseInt(
            process.env.SLOW_THRESHOLD_MS || '2000',
            10,
          ),
          vipSubscriptions: (
            process.env.VIP_SUBSCRIPTIONS || 'enterprise,premium'
          ).split(','),
          monitoredFlags: (process.env.MONITORED_FLAGS || '').split(',').filter(Boolean),
        }),
    },
  ],
  exports: [WideEventService, TailSampler],
})
export class WideEventModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(WideEventMiddleware).forRoutes('*');
  }
}
```

---

## Middleware to Attach WideEventService to Request

Because the interceptor is singleton-scoped but `WideEventService` is request-scoped, we use middleware to resolve and attach the service to the request object.

```typescript
// src/wide-event/wide-event.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { ModuleRef } from '@nestjs/core';
import { Request, Response, NextFunction } from 'express';
import { WideEventService } from './wide-event.service';

@Injectable()
export class WideEventMiddleware implements NestMiddleware {
  constructor(private moduleRef: ModuleRef) {}

  async use(req: Request, _res: Response, next: NextFunction) {
    // Resolve a new request-scoped instance of WideEventService
    const wideEventService = await this.moduleRef.resolve(
      WideEventService,
      undefined, // contextId: undefined triggers new scope per call
      { strict: false },
    );
    // Attach to request so the singleton interceptor can access it
    (req as any)['wideEventService'] = wideEventService;
    next();
  }
}
```

---

## App Bootstrap

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { WideEventInterceptor } from './wide-event/wide-event.interceptor';
import { WideEventExceptionFilter } from './wide-event/wide-event-exception.filter';
import { TailSampler } from './wide-event/tail-sampler';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    // Disable default NestJS logger noise if you want clean wide-event-only output
    // logger: false,
  });

  const tailSampler = app.get(TailSampler);

  app.useGlobalInterceptors(new WideEventInterceptor(tailSampler));
  app.useGlobalFilters(new WideEventExceptionFilter());

  await app.listen(process.env.PORT || 3000);
}
bootstrap();
```

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { WideEventModule } from './wide-event/wide-event.module';
// ... your other modules

@Module({
  imports: [
    WideEventModule,
    // ... your other modules
  ],
})
export class AppModule {}
```

---

## Usage in Controllers

Controllers inject `WideEventService` and enrich the event with business context relevant to the endpoint.

```typescript
// src/checkout/checkout.controller.ts
import { Controller, Post, Body, Req } from '@nestjs/common';
import { WideEventService } from '../wide-event/wide-event.service';
import { CheckoutService } from './checkout.service';
import { CheckoutDto } from './checkout.dto';

@Controller('checkout')
export class CheckoutController {
  constructor(
    private readonly wideEvent: WideEventService,
    private readonly checkoutService: CheckoutService,
  ) {}

  @Post()
  async checkout(@Body() dto: CheckoutDto, @Req() req: any) {
    // Enrich with business context
    const user = req.user; // from auth guard
    this.wideEvent.enrich({
      user: {
        id: user.id,
        subscription: user.subscription,
        account_age_days: this.daysSince(user.createdAt),
      },
    });

    const cart = await this.checkoutService.getCart(user.id);
    this.wideEvent.enrich({
      cart: {
        id: cart.id,
        item_count: cart.items.length,
        total_cents: cart.total,
        coupon_applied: cart.coupon?.code,
      },
    });

    const result = await this.checkoutService.processPayment(cart, user);

    this.wideEvent.enrich({
      payment: {
        method: result.method,
        provider: result.provider,
        attempt: result.attemptNumber,
      },
    });

    return { orderId: result.orderId };
  }

  private daysSince(date: Date): number {
    return Math.floor((Date.now() - date.getTime()) / 86400000);
  }
}
```

---

## Usage in Services

Services that perform I/O (database, HTTP, cache) should enrich the event with performance context.

```typescript
// src/payment/payment.service.ts
import { Injectable } from '@nestjs/common';
import { WideEventService } from '../wide-event/wide-event.service';

@Injectable()
export class PaymentService {
  constructor(private readonly wideEvent: WideEventService) {}

  async charge(amount: number, method: string): Promise<ChargeResult> {
    const start = Date.now();
    try {
      const result = await this.stripeClient.charges.create({ amount, ... });

      this.wideEvent.addExternalCall({
        service: 'stripe',
        duration_ms: Date.now() - start,
        status: 200,
      });

      return result;
    } catch (err) {
      this.wideEvent.addExternalCall({
        service: 'stripe',
        duration_ms: Date.now() - start,
        error: err.code || err.message,
      });

      // Enrich error context with domain-specific fields
      this.wideEvent.enrich({
        error: {
          type: 'PaymentError',
          code: err.code,
          stripe_decline_code: err.decline_code,
          retriable: err.code !== 'card_declined',
        },
      });

      throw err;
    }
  }
}
```

---

## Auth Guard Integration

Enrich the wide event with user identity as early as possible.

```typescript
// src/auth/auth.guard.ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { WideEventService } from '../wide-event/wide-event.service';

@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private readonly wideEvent: WideEventService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const req = context.switchToHttp().getRequest();
    const user = await this.validateToken(req.headers.authorization);

    if (!user) {
      this.wideEvent.enrich({
        user: { id: 'anonymous' },
        outcome: 'client_error',
      });
      return false;
    }

    req.user = user;
    this.wideEvent.enrich({
      user: {
        id: user.id,
        subscription: user.subscription,
        role: user.role,
        org_id: user.orgId,
      },
    });

    return true;
  }

  private async validateToken(header: string): Promise<any> {
    // Your JWT/session validation logic
  }
}
```

---

## AsyncLocalStorage Alternative

For contexts where NestJS request-scoped DI is not available (queue consumers, event handlers, raw middleware), use AsyncLocalStorage.

```typescript
// src/wide-event/wide-event.context.ts
import { AsyncLocalStorage } from 'async_hooks';
import { WideEventService } from './wide-event.service';

export const wideEventStorage = new AsyncLocalStorage<WideEventService>();

/**
 * Get the current request's WideEventService from AsyncLocalStorage.
 * Returns undefined if called outside a tracked context.
 */
export function getWideEvent(): WideEventService | undefined {
  return wideEventStorage.getStore();
}
```

```typescript
// Usage in the middleware (alternative approach)
import { wideEventStorage } from './wide-event.context';

async use(req: Request, _res: Response, next: NextFunction) {
  const wideEventService = await this.moduleRef.resolve(WideEventService);
  (req as any)['wideEventService'] = wideEventService;
  wideEventStorage.run(wideEventService, () => next());
}
```

```typescript
// Usage in a utility that does not have DI access
import { getWideEvent } from './wide-event.context';

export async function fetchFromExternalApi(url: string) {
  const start = Date.now();
  const res = await fetch(url);
  getWideEvent()?.addExternalCall({
    service: new URL(url).hostname,
    duration_ms: Date.now() - start,
    status: res.status,
  });
  return res;
}
```

---

## Queue / Cron Job Pattern

For non-HTTP contexts, create the wide event manually per job execution.

```typescript
// src/jobs/email-digest.processor.ts
import { Processor, Process } from '@nestjs/bull';
import { Job } from 'bull';
import { Logger } from '@nestjs/common';
import { TailSampler } from '../wide-event/tail-sampler';
import { WideEvent } from '../wide-event/wide-event.service';

@Processor('email-digest')
export class EmailDigestProcessor {
  private readonly logger = new Logger('WideEvent');

  constructor(private readonly tailSampler: TailSampler) {}

  @Process()
  async handle(job: Job) {
    const start = Date.now();
    const event: Partial<WideEvent> = {
      timestamp: new Date().toISOString(),
      request_id: `job_${job.id}`,
      service: process.env.SERVICE_NAME,
      version: process.env.SERVICE_VERSION,
      // Job-specific context
      job: {
        id: job.id,
        type: 'email_digest',
        queue: 'email-digest',
        attempt: job.attemptsMade + 1,
      },
    };

    try {
      const result = await this.processDigest(job.data);

      Object.assign(event, {
        duration_ms: Date.now() - start,
        outcome: 'success',
        level: 'info',
        emails_sent: result.sent,
        emails_failed: result.failed,
      });
    } catch (err) {
      Object.assign(event, {
        duration_ms: Date.now() - start,
        outcome: 'error',
        level: 'error',
        error: {
          type: err.constructor.name,
          message: err.message,
          retriable: true,
        },
      });
      throw err;
    } finally {
      if (this.tailSampler.shouldSample(event as WideEvent)) {
        this.logger.log(JSON.stringify(event));
      }
    }
  }

  private async processDigest(data: any) {
    // ... your job logic
  }
}
```

---

## Pino Integration

For production, replace the default NestJS logger with Pino for structured JSON output and better performance.

```typescript
// Install: npm install nestjs-pino pino pino-pretty

// src/main.ts
import { Logger } from 'nestjs-pino';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, { bufferLogs: true });
  app.useLogger(app.get(Logger));
  // ...
}
```

```typescript
// src/app.module.ts
import { LoggerModule } from 'nestjs-pino';

@Module({
  imports: [
    LoggerModule.forRoot({
      pinoHttp: {
        // Disable pino-http's own request logging. We handle it via wide events.
        autoLogging: false,
        level: process.env.LOG_LEVEL || 'info',
        transport:
          process.env.NODE_ENV !== 'production'
            ? { target: 'pino-pretty', options: { colorize: true } }
            : undefined,
      },
    }),
    WideEventModule,
    // ...
  ],
})
export class AppModule {}
```

Then in the interceptor, replace `this.logger.log(JSON.stringify(event))` with the Pino logger, which natively outputs JSON:

```typescript
// In WideEventInterceptor, inject PinoLogger:
import { PinoLogger } from 'nestjs-pino';

constructor(
  private readonly tailSampler: TailSampler,
  private readonly logger: PinoLogger,
) {}

// Then emit:
this.logger.info(event, 'wide_event');
// Pino writes the event fields as top-level JSON keys. No JSON.stringify needed.
```

---

## Testing

Test that wide events are correctly enriched without needing a real HTTP server.

```typescript
// src/wide-event/__tests__/wide-event.service.spec.ts
import { WideEventService } from '../wide-event.service';

describe('WideEventService', () => {
  let service: WideEventService;

  beforeEach(() => {
    service = new WideEventService();
  });

  it('should merge nested objects shallowly', () => {
    service.enrich({ user: { id: 'u1', subscription: 'free' } });
    service.enrich({ user: { subscription: 'premium' } });
    const event = service.finalize();
    expect(event.user).toEqual({ id: 'u1', subscription: 'premium' });
  });

  it('should accumulate external calls', () => {
    service.addExternalCall({ service: 'stripe', duration_ms: 100 });
    service.addExternalCall({ service: 'sendgrid', duration_ms: 50 });
    const event = service.finalize();
    expect(event.external_calls).toHaveLength(2);
  });

  it('should include timestamp and request_id by default', () => {
    const event = service.finalize();
    expect(event.timestamp).toBeDefined();
    expect(event.request_id).toMatch(/^req_/);
  });
});
```

```typescript
// src/wide-event/__tests__/tail-sampler.spec.ts
import { TailSampler } from '../tail-sampler';
import { WideEvent } from '../wide-event.service';

describe('TailSampler', () => {
  const sampler = new TailSampler({ defaultRate: 0 }); // 0% random sampling

  it('should always keep 500 errors', () => {
    const event = { status_code: 500 } as WideEvent;
    expect(sampler.shouldSample(event)).toBe(true);
  });

  it('should always keep events with error field', () => {
    const event = {
      status_code: 200,
      error: { type: 'Timeout', message: 'timed out' },
    } as WideEvent;
    expect(sampler.shouldSample(event)).toBe(true);
  });

  it('should always keep slow requests', () => {
    const event = { status_code: 200, duration_ms: 5000 } as WideEvent;
    expect(sampler.shouldSample(event)).toBe(true);
  });

  it('should drop healthy requests when rate is 0', () => {
    const event = { status_code: 200, duration_ms: 50 } as WideEvent;
    expect(sampler.shouldSample(event)).toBe(false);
  });
});
```
