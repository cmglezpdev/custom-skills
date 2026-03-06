# Grafana Dashboards and Alerting

Structure, provisioning, and alert rules for each stage of metrics, backed by Prometheus.

## Table of Contents
1. [Dashboard Hierarchy](#hierarchy)
2. [Service Overview Dashboard (Stage 1)](#service-overview)
3. [Dependency Health Dashboard (Stage 2)](#dependency-health)
4. [Business Dashboard (Stage 3)](#business)
5. [Operational Dashboard (Stage 4)](#operational)
6. [Alert Rules](#alerts)
7. [Provisioning](#provisioning)

---

## 1. Dashboard Hierarchy <a id="hierarchy"></a>

Organize dashboards by audience and urgency:

```
Folder: NestJS App
├── Service Overview          <- On-call engineer's first stop
├── Dependency Health         <- "Is it us or our dependencies?"
├── Business Metrics          <- Product managers, leadership
└── Operational Control       <- Platform team, SRE
```

Each dashboard should load in under 3 seconds. If it doesn't, you have too many panels or your queries span too much data.

**Variables to define on every dashboard:**
- `$environment` (production, staging)
- `$instance` (for multi-instance deployments)
- Time range selector (default: last 1h for operational, last 24h for business)

---

## 2. Service Overview Dashboard (Stage 1) <a id="service-overview"></a>

The dashboard your on-call engineer opens first.

**Row 1: Golden Signals (single stat panels)**

| Panel | PromQL | Type |
|---|---|---|
| Request Rate | `sum(rate(http_requests_total[5m]))` | Stat |
| Error Rate % | `sum(rate(http_requests_total{status_code=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100` | Stat (red threshold > 1%) |
| P99 Latency | `histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))` | Stat (red threshold > 1s) |
| In-Flight Requests | `sum(http_requests_in_flight)` | Stat |

**Row 2: Request Rate and Error Rate (time series)**

| Panel | PromQL |
|---|---|
| Request Rate by Route | `sum(rate(http_requests_total[5m])) by (route, method)` |
| Error Rate by Route | `sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (route)` |
| Status Code Distribution | `sum(rate(http_requests_total[5m])) by (status_code)` |

**Row 3: Latency (time series)**

| Panel | PromQL |
|---|---|
| Latency Percentiles (P50, P95, P99) | `histogram_quantile(0.5, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))` (repeat for 0.95, 0.99) |
| Latency Heatmap | Use `http_request_duration_seconds_bucket` with Grafana's heatmap panel |

**Row 4: Node.js Runtime**

| Panel | PromQL |
|---|---|
| Event Loop Lag P95 | `histogram_quantile(0.95, sum(rate(nodejs_eventloop_lag_seconds_bucket[5m])) by (le))` |
| Heap Used | `process_resident_memory_bytes` (from host-metrics) |
| Active Handles | `nodejs_active_handles_total` |

**Row 5: Health Checks**

| Panel | PromQL |
|---|---|
| Health Status | `app_health_check_status` (table panel, green/red) |
| Health Check Duration | `app_health_check_duration_seconds` by check_name |

---

## 3. Dependency Health Dashboard (Stage 2) <a id="dependency-health"></a>

Answers: "Is the problem in our app or in what we depend on?"

**Row 1: Database (App Perspective)**

| Panel | PromQL |
|---|---|
| DB Query P95 by Operation | `histogram_quantile(0.95, sum(rate(app_db_query_duration_seconds_bucket[5m])) by (le, operation))` |
| DB Pool Wait Time P99 | `histogram_quantile(0.99, sum(rate(app_db_connection_pool_wait_seconds_bucket[5m])) by (le))` |
| DB Query Error Rate | `sum(rate(app_db_query_errors_total[5m])) by (error_type)` |

**Row 2: Redis (App Perspective)**

| Panel | PromQL |
|---|---|
| Cache Hit Ratio | `sum(rate(app_cache_hit_total[5m])) by (cache_name) / (sum(rate(app_cache_hit_total[5m])) by (cache_name) + sum(rate(app_cache_miss_total[5m])) by (cache_name))` |
| Redis Op Duration P95 | `histogram_quantile(0.95, sum(rate(app_redis_operation_duration_seconds_bucket[5m])) by (le, operation))` |
| Redis Errors | `sum(rate(app_redis_operation_errors_total[5m])) by (error_type)` |

**Row 3: External APIs**

| Panel | PromQL |
|---|---|
| External API P95 by Service | `histogram_quantile(0.95, sum(rate(app_external_api_duration_seconds_bucket[5m])) by (le, service))` |
| Circuit Breaker State | `app_circuit_breaker_state` (table, colored by value) |
| External API Error Rate | `sum(rate(app_external_api_errors_total[5m])) by (service, error_type)` |

**Row 4: Background Jobs**

| Panel | PromQL |
|---|---|
| Queue Depth | `app_job_queue_depth` by queue |
| Job Duration P95 | `histogram_quantile(0.95, sum(rate(app_job_duration_seconds_bucket[5m])) by (le, queue))` |
| Job Failure Rate | `sum(rate(app_job_completed_total{status="failed"}[5m])) by (queue) / sum(rate(app_job_completed_total[5m])) by (queue)` |

**Row 5: Auth**

| Panel | PromQL |
|---|---|
| Auth Failure Rate | `sum(rate(app_auth_attempts_total{result="failure"}[5m])) / sum(rate(app_auth_attempts_total[5m]))` |
| Token Operations | `sum(rate(app_auth_token_operations_total[5m])) by (operation)` |
| Authorization Denials | `sum(rate(app_authorization_denied_total[5m])) by (resource, action)` |

---

## 4. Business Dashboard (Stage 3) <a id="business"></a>

Non-engineers should understand this. Minimal jargon. Clear titles.

**Row 1: Key Business Metrics (stat panels, big numbers)**

| Panel | PromQL |
|---|---|
| Signups Today | `sum(increase(app_user_signups_total[24h]))` |
| Revenue Today | `sum(increase(app_transaction_value_total{currency="USD"}[24h]))` |
| Tx Success Rate | `sum(rate(app_transactions_total{status="success"}[1h])) / sum(rate(app_transactions_total[1h])) * 100` |

**Row 2: User Lifecycle (time series)**

| Panel | PromQL |
|---|---|
| Signups by Source | `sum(increase(app_user_signups_total[1h])) by (source)` |
| Logins by Method | `sum(rate(app_user_logins_total[5m])) by (method)` |

**Row 3: Conversion Funnels (bar gauge or table)**

Display each step's completion rate relative to the first step.

**Row 4: Feature Adoption**

| Panel | PromQL |
|---|---|
| Feature Usage | `sum(rate(app_feature_usage_total[1h])) by (feature_name, variant)` |
| Feature Error Rate | `sum(rate(app_feature_errors_total[1h])) by (feature_name)` |

**Row 5: SLO Compliance**

| Panel | PromQL |
|---|---|
| 30-day Availability | `1 - (sum(increase(app_sli_request_availability{result="failure"}[30d])) / sum(increase(app_sli_request_availability[30d])))` |
| Error Budget Remaining | Compute from SLO target vs actual |

---

## 5. Operational Dashboard (Stage 4) <a id="operational"></a>

For platform teams managing multi-tenant, scaled deployments. Rows for: WebSocket connections, rate limiter activity, tenant resource usage, deployment version distribution, startup/shutdown times.

---

## 6. Alert Rules <a id="alerts"></a>

### Stage 1 Alerts (Critical)

```yaml
groups:
  - name: nestjs-critical
    rules:
      - alert: HighErrorRate
        expr: >
          sum(rate(http_requests_total{status_code=~"5.."}[5m]))
          / sum(rate(http_requests_total[5m]))
          > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Error rate above 5% for 2 minutes"

      - alert: HighLatencyP99
        expr: >
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
          ) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P99 latency above 2s for 5 minutes"

      - alert: EventLoopLagHigh
        expr: >
          histogram_quantile(0.95,
            sum(rate(nodejs_eventloop_lag_seconds_bucket[5m])) by (le)
          ) > 0.1
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "Event loop lag P95 above 100ms for 3 minutes"

      - alert: HealthCheckFailing
        expr: app_health_check_status == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Health check {{ $labels.check_name }} is failing"
```

### Stage 2 Alerts (Dependency)

```yaml
      - alert: DbPoolWaitHigh
        expr: >
          histogram_quantile(0.99,
            sum(rate(app_db_connection_pool_wait_seconds_bucket[5m])) by (le)
          ) > 0.5
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "Database pool wait time P99 above 500ms"

      - alert: CacheHitRateLow
        expr: >
          sum(rate(app_cache_hit_total[10m])) by (cache_name)
          / (sum(rate(app_cache_hit_total[10m])) by (cache_name)
            + sum(rate(app_cache_miss_total[10m])) by (cache_name))
          < 0.5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Cache hit rate for {{ $labels.cache_name }} below 50%"

      - alert: CircuitBreakerOpen
        expr: app_circuit_breaker_state == 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Circuit breaker for {{ $labels.service }} is OPEN"

      - alert: JobQueueBacklog
        expr: app_job_queue_depth > 500
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Job queue {{ $labels.queue }} has 500+ pending jobs"
```

### Stage 3 Alerts (Business)

```yaml
      - alert: SLOBurnRateHigh
        expr: >
          sum(rate(app_sli_request_availability{result="failure"}[1h]))
          / sum(rate(app_sli_request_availability[1h]))
          > 14 * 0.001
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "SLO burn rate is 14x budget for {{ $labels.sli_name }}"

      - alert: TransactionFailureSpike
        expr: >
          sum(rate(app_transactions_total{status="failed"}[5m]))
          / sum(rate(app_transactions_total[5m]))
          > 0.1
        for: 3m
        labels:
          severity: critical
        annotations:
          summary: "Transaction failure rate above 10%"
```

---

## 7. Provisioning <a id="provisioning"></a>

If you manage Grafana dashboards as code (recommended):

**File provisioning (docker-compose / k8s)**

```yaml
# grafana/provisioning/dashboards/dashboards.yaml
apiVersion: 1
providers:
  - name: NestJS App
    type: file
    folder: NestJS App
    options:
      path: /var/lib/grafana/dashboards/nestjs
      foldersFromFilesStructure: false
```

Place dashboard JSON files in the mounted directory. Export from Grafana UI after building manually, or use grafonnet/Jsonnet for programmatic generation.

**Dashboard JSON export tip**: Build the dashboard manually in Grafana first, then export via Share, Export. Strip the `id` field (instance-specific) and keep `uid` for idempotent provisioning.

**Alert provisioning**: Grafana alerting rules can be provisioned via YAML files in `grafana/provisioning/alerting/`. The alert examples above are in Prometheus alerting rule format. For Grafana-native alerting, convert them to Grafana's provisioning format or create them in the Grafana UI.
