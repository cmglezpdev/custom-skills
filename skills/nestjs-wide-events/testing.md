# Testing Wide Events

Test that wide events are correctly enriched without needing a real HTTP server.

## WideEventService Tests

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

## TailSampler Tests

```typescript
// src/wide-event/__tests__/tail-sampler.spec.ts
import { TailSampler } from '../tail-sampler';
import { WideEvent } from '../wide-event.service';

describe('TailSampler', () => {
  const sampler = new TailSampler({ defaultRate: 0 });

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
