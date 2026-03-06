# Usage Examples: Enriching Wide Events

How to inject `WideEventService` and enrich the wide event from controllers, services, and guards.

## Usage in Controllers

Controllers enrich the event with business context relevant to the endpoint.

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
    const user = req.user;
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

Services that perform I/O (database, HTTP, cache) enrich the event with performance context.

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
