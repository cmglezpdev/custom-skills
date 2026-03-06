# OpenTelemetry SDK Setup and Collector Configuration

Complete setup for pushing application metrics from NestJS to Prometheus via an OTel Collector.

## Table of Contents
1. [Install Dependencies](#install)
2. [SDK Bootstrap](#bootstrap)
3. [NestJS Integration](#nestjs)
4. [Collector Configuration](#collector)
5. [Verify the Pipeline](#verify)

---

## 1. Install Dependencies <a id="install"></a>

```bash
npm install @opentelemetry/sdk-node \
  @opentelemetry/sdk-metrics \
  @opentelemetry/exporter-metrics-otlp-grpc \
  @opentelemetry/resources \
  @opentelemetry/semantic-conventions \
  @opentelemetry/host-metrics
```

If your Collector only accepts HTTP, replace `exporter-metrics-otlp-grpc` with `@opentelemetry/exporter-metrics-otlp-http`. gRPC is preferred for lower overhead.

---

## 2. SDK Bootstrap <a id="bootstrap"></a>

This file must be loaded **before** the NestJS app starts. It configures the global MeterProvider that all application code will use to create instruments.

```typescript
// src/telemetry/metrics.bootstrap.ts
import { OTLPMetricExporter } from '@opentelemetry/exporter-metrics-otlp-grpc';
import {
  MeterProvider,
  PeriodicExportingMetricReader,
} from '@opentelemetry/sdk-metrics';
import { Resource } from '@opentelemetry/resources';
import {
  ATTR_SERVICE_NAME,
  ATTR_SERVICE_VERSION,
  ATTR_DEPLOYMENT_ENVIRONMENT_NAME,
} from '@opentelemetry/semantic-conventions';
import { metrics } from '@opentelemetry/api';
import { HostMetrics } from '@opentelemetry/host-metrics';

export function bootstrapMetrics() {
  const resource = new Resource({
    [ATTR_SERVICE_NAME]: process.env.OTEL_SERVICE_NAME || 'nestjs-app',
    [ATTR_SERVICE_VERSION]: process.env.APP_VERSION || 'unknown',
    [ATTR_DEPLOYMENT_ENVIRONMENT_NAME]: process.env.NODE_ENV || 'development',
  });

  const metricExporter = new OTLPMetricExporter({
    // Points to your OTel Collector's OTLP gRPC receiver
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT || 'http://otel-collector:4317',
  });

  const metricReader = new PeriodicExportingMetricReader({
    exporter: metricExporter,
    // How often metrics are flushed to the Collector.
    // 15s is a good default. Lower = more network overhead.
    // Higher = longer delay before metrics appear in Prometheus.
    exportIntervalMillis: 15_000,
  });

  const meterProvider = new MeterProvider({
    resource,
    readers: [metricReader],
  });

  // Register as the global MeterProvider.
  // After this, metrics.getMeter('name') works anywhere in the app.
  metrics.setGlobalMeterProvider(meterProvider);

  // Host metrics: CPU, memory, network from the Node.js process perspective.
  // These complement container-level metrics from cAdvisor by showing
  // what the Node.js process itself sees.
  const hostMetrics = new HostMetrics({
    meterProvider,
    name: 'nestjs-host-metrics',
  });
  hostMetrics.start();

  return meterProvider;
}
```

**Load it in `main.ts` before NestFactory.create:**

```typescript
// src/main.ts
import { bootstrapMetrics } from './telemetry/metrics.bootstrap';

// MUST be called before NestFactory.create() so the MeterProvider
// is available when modules initialize and create instruments.
const meterProvider = bootstrapMetrics();

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Graceful shutdown: flush pending metrics before exit
  app.enableShutdownHooks();
  const shutdownHandler = async () => {
    await meterProvider.shutdown();
  };
  process.on('SIGTERM', shutdownHandler);
  process.on('SIGINT', shutdownHandler);

  await app.listen(3000);
}
bootstrap();
```

---

## 3. NestJS Integration <a id="nestjs"></a>

Create a metrics module that provides the OTel `Meter` instance to the rest of the app via NestJS DI.

```typescript
// src/telemetry/metrics.module.ts
import { Global, Module } from '@nestjs/common';
import { metrics, Meter } from '@opentelemetry/api';

export const METER_TOKEN = 'OTEL_METER';

@Global()
@Module({
  providers: [
    {
      provide: METER_TOKEN,
      useFactory: (): Meter => {
        // Gets the meter from the global MeterProvider set in bootstrap.
        return metrics.getMeter('nestjs-app', '1.0.0');
      },
    },
  ],
  exports: [METER_TOKEN],
})
export class MetricsModule {}
```

Register it in `app.module.ts`:
```typescript
import { MetricsModule } from './telemetry/metrics.module';

@Module({
  imports: [
    MetricsModule,
    // ... other modules
  ],
})
export class AppModule {}
```

Now any service can inject the meter:
```typescript
import { Inject, Injectable } from '@nestjs/common';
import { Meter } from '@opentelemetry/api';
import { METER_TOKEN } from './telemetry/metrics.module';

@Injectable()
export class SomeService {
  private readonly myCounter;

  constructor(@Inject(METER_TOKEN) private readonly meter: Meter) {
    this.myCounter = this.meter.createCounter('app_something_total', {
      description: 'Total somethings',
    });
  }
}
```

---

## 4. Collector Configuration <a id="collector"></a>

Minimal Collector config that receives OTLP metrics and exports to Prometheus.

### Option A: Collector exposes a `/metrics` endpoint that Prometheus scrapes

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 10s
    send_batch_size: 1024

exporters:
  prometheus:
    endpoint: 0.0.0.0:8889
    # Namespace prefix for all metrics (optional, avoid if you already prefix with app_)
    # namespace: ""
    resource_to_telemetry_conversion:
      enabled: true

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
```

Then add a Prometheus scrape target:
```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'otel-collector'
    scrape_interval: 15s
    static_configs:
      - targets: ['otel-collector:8889']
```

### Option B: Collector pushes to Prometheus via remote_write

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  batch:
    timeout: 10s

exporters:
  prometheusremotewrite:
    endpoint: http://prometheus:9090/api/v1/write
    resource_to_telemetry_conversion:
      enabled: true

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheusremotewrite]
```

Prometheus must have `--web.enable-remote-write-receiver` flag enabled for Option B.

### Docker Compose snippet

```yaml
# docker-compose.yml (relevant services only)
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    volumes:
      - ./otel-collector-config.yaml:/etc/otelcol-contrib/config.yaml
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "8889:8889"   # Prometheus exporter (Option A only)
    depends_on:
      - prometheus

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    # Uncomment for Option B (remote_write):
    # command:
    #   - '--config.file=/etc/prometheus/prometheus.yml'
    #   - '--web.enable-remote-write-receiver'
```

---

## 5. Verify the Pipeline <a id="verify"></a>

After setting up:

1. **Start the app and Collector**. Check Collector logs for `"msg":"Exporter started"` or errors.

2. **Send some requests** to your NestJS app.

3. **Check the Collector's own metrics** (if using the prometheus exporter): `curl http://otel-collector:8889/metrics`. You should see your app's metrics with their attributes.

4. **Check Prometheus targets**: In the Prometheus UI (`http://prometheus:9090/targets`), the `otel-collector` job should show as UP.

5. **Query in Grafana**: Go to Explore, select Prometheus, run:
   ```promql
   rate(http_requests_total[5m])
   ```
   You should see data points.

**Common issues:**
- No data in Prometheus: Check the Collector logs. Most often the OTLP endpoint URL is wrong in the app's bootstrap, or the Collector can't reach Prometheus.
- Metrics appear with unexpected names: The OTel SDK to Prometheus conversion follows conventions. Dots become underscores. Units may be appended. Use `resource_to_telemetry_conversion: enabled: true` in the exporter to convert OTel resource attributes (like `service.name`) into Prometheus labels.
- Histogram buckets look wrong: OTel SDK uses explicit bucket boundaries you define. If you're using the default (no boundaries specified), the SDK picks its own defaults which may not match your needs. Always specify boundaries explicitly.
