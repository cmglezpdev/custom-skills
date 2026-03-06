# Stage 3: Business Metrics

Metrics that make the business observable through the same system engineers use. Built with the OpenTelemetry SDK.

## Table of Contents
1. [Cardinality Rules](#cardinality)
2. [Create Instruments](#instruments)
3. [Business Events Service](#business-events)
4. [Conversion Funnel Tracking](#funnels)
5. [SLI Metrics for SLOs](#sli)
6. [Feature Usage Tracking](#features)
7. [PromQL Queries](#promql)

---

## 1. Cardinality Rules (Read This First) <a id="cardinality"></a>

Business metrics are where cardinality explosions happen. Every unique attribute combination creates a time series. Every time series costs storage and query performance in Prometheus.

**Hard rules:**
- Never use `user_id`, `email`, `session_id`, or any per-user identifier as an attribute. These belong in logs or traces, not metrics.
- Never use free-text fields as attributes.
- Cap every attribute to a known, bounded set of values. If you can't enumerate possible values, it's not a metric attribute.
- `plan` is fine (free, pro, enterprise). `company_name` is not (unbounded).
- `source` is fine if bounded (organic, google, referral, direct). Not fine if it's a raw UTM parameter.
- `feature_name` is fine if you maintain a registry. Not fine if developers add arbitrary strings.

**Cardinality budget rule of thumb:**
- Total unique time series per metric < 1000 for most apps
- If a single metric would produce > 10,000 series, redesign attributes or move to logs

**If you need per-user business data**: Use structured logs (Loki) with `user_id` as a label there. Logs handle high cardinality. Metrics don't.

---

## 2. Create Instruments <a id="instruments"></a>

```typescript
// src/telemetry/instruments/stage3.instruments.ts
import { Inject, Injectable } from '@nestjs/common';
import { Meter, Counter, Histogram } from '@opentelemetry/api';
import { METER_TOKEN } from '../metrics.module';

@Injectable()
export class Stage3Instruments {
  // User Lifecycle
  public readonly userSignups: Counter;
  public readonly userLogins: Counter;
  public readonly userActions: Counter;

  // Transactions
  public readonly transactions: Counter;
  public readonly transactionValue: Counter;
  public readonly conversionFunnel: Counter;

  // Features
  public readonly featureUsage: Counter;
  public readonly featureErrors: Counter;

  // SLIs
  public readonly sliAvailability: Counter;
  public readonly sliLatency: Histogram;

  constructor(@Inject(METER_TOKEN) meter: Meter) {
    this.userSignups = meter.createCounter('app_user_signups_total', {
      description: 'Total user signups',
    });

    this.userLogins = meter.createCounter('app_user_logins_total', {
      description: 'Total user logins',
    });

    this.userActions = meter.createCounter('app_user_actions_total', {
      description: 'Total user actions by type',
    });

    this.transactions = meter.createCounter('app_transactions_total', {
      description: 'Total business transactions',
    });

    this.transactionValue = meter.createCounter('app_transaction_value_total', {
      description: 'Cumulative transaction value (use rate() to get per-second)',
    });

    this.conversionFunnel = meter.createCounter('app_conversion_funnel_step_total', {
      description: 'Conversion funnel step completions',
    });

    this.featureUsage = meter.createCounter('app_feature_usage_total', {
      description: 'Feature usage events',
    });

    this.featureErrors = meter.createCounter('app_feature_errors_total', {
      description: 'Feature-specific errors',
    });

    this.sliAvailability = meter.createCounter('app_sli_request_availability', {
      description: 'SLI availability counter (increment per request, attribute success/failure)',
    });

    this.sliLatency = meter.createHistogram('app_sli_request_latency_seconds', {
      description: 'SLI latency histogram with tight buckets around SLO threshold',
      unit: 's',
      advice: {
        // Adjust to your actual SLO. If your SLO is "99% under 200ms",
        // you need good resolution around 200ms.
        explicitBucketBoundaries: [0.05, 0.1, 0.15, 0.2, 0.25, 0.3, 0.5, 1, 2],
      },
    });
  }
}
```

Register `Stage3Instruments` in the `MetricsModule` providers and exports.

---

## 3. Business Events Service <a id="business-events"></a>

A single service that all business-event-producing code calls. Centralizes metric emission and enforces cardinality rules.

```typescript
// src/telemetry/business-metrics.service.ts
import { Injectable } from '@nestjs/common';
import { Stage3Instruments } from './instruments/stage3.instruments';

// Enforce bounded attribute values via types
type SignupSource = 'organic' | 'google' | 'referral' | 'direct' | 'other';
type Plan = 'free' | 'starter' | 'pro' | 'enterprise';
type LoginMethod = 'password' | 'google' | 'github' | 'saml' | 'api_key';
type TransactionStatus = 'success' | 'failed' | 'refunded' | 'pending';

@Injectable()
export class BusinessMetricsService {
  constructor(private readonly instruments: Stage3Instruments) {}

  // --- User Lifecycle ---

  recordSignup(source: SignupSource, plan: Plan) {
    this.instruments.userSignups.add(1, { source, plan });
  }

  recordLogin(method: LoginMethod) {
    this.instruments.userLogins.add(1, { method });
  }

  /**
   * action_type MUST come from a known, bounded set.
   * Define your action types as a union type or enum.
   */
  recordUserAction(actionType: string) {
    this.instruments.userActions.add(1, { action_type: actionType });
  }

  // --- Transactions ---

  recordTransaction(
    type: string,
    status: TransactionStatus,
    value?: number,
    currency: string = 'USD',
  ) {
    this.instruments.transactions.add(1, { type, status });
    if (value && status === 'success') {
      this.instruments.transactionValue.add(value, { currency });
    }
  }

  // --- Features ---

  recordFeatureUsage(featureName: string, variant: string = 'default') {
    this.instruments.featureUsage.add(1, { feature_name: featureName, variant });
  }

  recordFeatureError(featureName: string) {
    this.instruments.featureErrors.add(1, { feature_name: featureName });
  }
}
```

**Usage in your service layer:**
```typescript
@Injectable()
export class UsersService {
  constructor(
    private readonly businessMetrics: BusinessMetricsService,
  ) {}

  async register(dto: CreateUserDto) {
    const user = await this.usersRepository.save(dto);
    this.businessMetrics.recordSignup(
      this.classifySource(dto.referrer),
      dto.plan || 'free',
    );
    return user;
  }

  private classifySource(referrer?: string): SignupSource {
    if (!referrer) return 'direct';
    if (referrer.includes('google')) return 'google';
    return 'other';
  }
}
```

---

## 4. Conversion Funnel Tracking <a id="funnels"></a>

Each step is a counter. Drop-off is computed in PromQL.

```typescript
// src/telemetry/funnel-metrics.service.ts
import { Injectable } from '@nestjs/common';
import { Stage3Instruments } from './instruments/stage3.instruments';

export const ONBOARDING_FUNNEL = {
  name: 'onboarding',
  steps: {
    SIGNUP: 'signup',
    EMAIL_VERIFIED: 'email_verified',
    PROFILE_COMPLETED: 'profile_completed',
    FIRST_ACTION: 'first_action',
    FIRST_WEEK_RETAINED: 'first_week_retained',
  },
} as const;

export const CHECKOUT_FUNNEL = {
  name: 'checkout',
  steps: {
    CART_CREATED: 'cart_created',
    CHECKOUT_STARTED: 'checkout_started',
    PAYMENT_SUBMITTED: 'payment_submitted',
    PAYMENT_CONFIRMED: 'payment_confirmed',
    ORDER_COMPLETED: 'order_completed',
  },
} as const;

@Injectable()
export class FunnelMetricsService {
  constructor(private readonly instruments: Stage3Instruments) {}

  recordFunnelStep(funnelName: string, step: string) {
    this.instruments.conversionFunnel.add(1, { funnel: funnelName, step });
  }
}
```

---

## 5. SLI Metrics for SLO Tracking <a id="sli"></a>

SLIs are scoped to specific user-facing journeys, not all traffic.

```typescript
// src/telemetry/sli-metrics.service.ts
import { Injectable } from '@nestjs/common';
import { Stage3Instruments } from './instruments/stage3.instruments';

@Injectable()
export class SliMetricsService {
  constructor(private readonly instruments: Stage3Instruments) {}

  recordRequest(sliName: string, success: boolean, durationSeconds: number) {
    this.instruments.sliAvailability.add(1, {
      sli_name: sliName,
      result: success ? 'success' : 'failure',
    });
    this.instruments.sliLatency.record(durationSeconds, { sli_name: sliName });
  }
}
```

**SLO burn rate alerting** (PromQL):
```promql
(
  sum(rate(app_sli_request_availability{sli_name="api-read", result="failure"}[1h]))
  /
  sum(rate(app_sli_request_availability{sli_name="api-read"}[1h]))
) > (14 * 0.001)
```

---

## 6. Feature Usage Tracking <a id="features"></a>

For A/B tests and feature flags:

```typescript
async evaluateFeature(featureName: string, userId: string): Promise<string> {
  const variant = await this.featureFlagService.evaluate(featureName, userId);
  this.businessMetrics.recordFeatureUsage(featureName, variant);
  return variant;
}
```

Combine with conversion funnel metrics to measure feature impact on business outcomes.

---

## 7. Key PromQL Queries for Stage 3 <a id="promql"></a>

```promql
# Signup rate by source (hourly)
sum(increase(app_user_signups_total[1h])) by (source)

# Revenue per second
sum(rate(app_transaction_value_total{currency="USD"}[5m]))

# Conversion funnel drop-off (onboarding)
sum(increase(app_conversion_funnel_step_total{funnel="onboarding", step="email_verified"}[24h]))
  /
sum(increase(app_conversion_funnel_step_total{funnel="onboarding", step="signup"}[24h]))

# SLO compliance (30-day availability)
1 - (
  sum(increase(app_sli_request_availability{sli_name="api-read", result="failure"}[30d]))
  /
  sum(increase(app_sli_request_availability{sli_name="api-read"}[30d]))
)

# Feature adoption rate
sum(rate(app_feature_usage_total{feature_name="new-checkout"}[1h])) by (variant)

# Transaction failure rate by type
sum(rate(app_transactions_total{status="failed"}[5m])) by (type)
  /
sum(rate(app_transactions_total[5m])) by (type)
```
