# Core Setup: Wide Event Components

Complete code for the core wide event infrastructure. Copy and adapt these to your NestJS project.

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

    const wideEventService: WideEventService = req['wideEventService'];
    if (!wideEventService) {
      return next.handle();
    }

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
        this.logger.log(JSON.stringify(event));
      }
    };

    return next.handle().pipe(
      tap(() => emit()),
      catchError((err) => {
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
  defaultRate: number;
  slowThreshold_ms: number;
  vipSubscriptions: string[];
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
    if (event.status_code && event.status_code >= 500) return true;
    if (event.error) return true;

    if (
      event.duration_ms &&
      event.duration_ms > this.config.slowThreshold_ms
    ) {
      return true;
    }

    if (
      event.user?.subscription &&
      this.config.vipSubscriptions.includes(event.user.subscription as string)
    ) {
      return true;
    }

    if (event.feature_flags) {
      for (const flag of this.config.monitoredFlags) {
        if (event.feature_flags[flag]) return true;
      }
    }

    return Math.random() < this.config.defaultRate;
  }
}
```

---

## WideEventMiddleware

Because the interceptor is singleton-scoped but `WideEventService` is request-scoped, middleware resolves and attaches the service to the request object.

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
    const wideEventService = await this.moduleRef.resolve(
      WideEventService,
      undefined,
      { strict: false },
    );
    (req as any)['wideEventService'] = wideEventService;
    next();
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
          slowThreshold_ms: parseInt(process.env.SLOW_THRESHOLD_MS || '2000', 10),
          vipSubscriptions: (process.env.VIP_SUBSCRIPTIONS || 'enterprise,premium').split(','),
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

## App Bootstrap

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { WideEventInterceptor } from './wide-event/wide-event.interceptor';
import { WideEventExceptionFilter } from './wide-event/wide-event-exception.filter';
import { TailSampler } from './wide-event/tail-sampler';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

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

@Module({
  imports: [
    WideEventModule,
    // ... your other modules
  ],
})
export class AppModule {}
```
