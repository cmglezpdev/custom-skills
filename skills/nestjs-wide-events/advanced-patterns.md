# Advanced Patterns

Patterns for non-standard contexts: AsyncLocalStorage, queue/cron jobs, and Pino integration.

## AsyncLocalStorage Alternative

For contexts where NestJS request-scoped DI is not available (queue consumers, event handlers, raw middleware), use AsyncLocalStorage.

```typescript
// src/wide-event/wide-event.context.ts
import { AsyncLocalStorage } from 'async_hooks';
import { WideEventService } from './wide-event.service';

export const wideEventStorage = new AsyncLocalStorage<WideEventService>();

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
// Usage in a utility without DI access
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
        autoLogging: false,
        level: process.env.LOG_LEVEL || 'info',
        transport:
          process.env.NODE_ENV !== 'production'
            ? { target: 'pino-pretty', options: { colorize: true } }
            : undefined,
      },
    }),
    WideEventModule,
  ],
})
export class AppModule {}
```

Then in the interceptor, replace `this.logger.log(JSON.stringify(event))` with the Pino logger:

```typescript
// In WideEventInterceptor, inject PinoLogger:
import { PinoLogger } from 'nestjs-pino';

constructor(
  private readonly tailSampler: TailSampler,
  private readonly logger: PinoLogger,
) {}

// Emit:
this.logger.info(event, 'wide_event');
// Pino writes event fields as top-level JSON keys. No JSON.stringify needed.
```
