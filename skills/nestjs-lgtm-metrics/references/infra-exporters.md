# Infrastructure Exporters and Community Dashboards

This reference covers the exporter setup, Prometheus scrape config, and recommended Grafana community dashboards for common infrastructure services. These metrics complement your application-level metrics. Your app metrics show how the app *experiences* these services. Infra metrics show how the services *perform* internally.

## Table of Contents
1. [Architecture Overview](#architecture)
2. [Container and Host Metrics](#containers)
3. [PostgreSQL](#postgresql)
4. [Redis](#redis)
5. [RabbitMQ](#rabbitmq)
6. [MongoDB](#mongodb)
7. [Qdrant](#qdrant)
8. [Elasticsearch / OpenSearch](#elasticsearch)
9. [Nginx / Reverse Proxy](#nginx)
10. [MinIO / S3-Compatible Storage](#minio)
11. [Complete Prometheus Scrape Config](#full-scrape-config)
12. [Dashboard Organization](#dashboard-org)

---

## 1. Architecture Overview <a id="architecture"></a>

```
┌─────────────────────────────────────────────────────────────┐
│  Infrastructure Metrics (this file)                         │
│                                                             │
│  node-exporter ─────┐                                       │
│  cAdvisor ──────────┤                                       │
│  postgres-exporter ─┤                                       │
│  redis-exporter ────┤──→ Prometheus ──→ Grafana             │
│  rabbitmq (built-in)┤                                       │
│  mongodb-exporter ──┤                                       │
│  qdrant (built-in) ─┘                                       │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│  Application Metrics (stages 1-4)                           │
│                                                             │
│  NestJS /metrics ───────→ Prometheus ──→ Grafana            │
└─────────────────────────────────────────────────────────────┘
```

**What infra exporters give you**: Server-side query throughput, memory consumption, replication lag, connection counts, disk I/O, container resource limits.

**What infra exporters cannot give you**: App-side latency (including pool wait, serialization, network), which cache strategy is effective, which queries are slow from the user's perspective, business-level error rates. That's what Stages 1-4 cover.

---

## 2. Container and Host Metrics <a id="containers"></a>

### node-exporter (Host metrics)

Exposes CPU, memory, disk, network, and filesystem metrics for the host machine. If you run on Kubernetes, the DaemonSet handles this. For docker-compose:

```yaml
# docker-compose.yml
node-exporter:
  image: prom/node-exporter:latest
  container_name: node-exporter
  restart: unless-stopped
  pid: host
  volumes:
    - /proc:/host/proc:ro
    - /sys:/host/sys:ro
    - /:/rootfs:ro
  command:
    - '--path.procfs=/host/proc'
    - '--path.sysfs=/host/sys'
    - '--path.rootfs=/rootfs'
    - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
  ports:
    - '9100:9100'
```

**Prometheus scrape config:**
```yaml
- job_name: 'node-exporter'
  scrape_interval: 15s
  static_configs:
    - targets: ['node-exporter:9100']
```

**Grafana community dashboard:** [Node Exporter Full — ID `1860`](https://grafana.com/grafana/dashboards/1860)

Covers CPU, memory, disk I/O, network, filesystem usage, system load. This is the single most useful infra dashboard.

---

### cAdvisor (Container metrics)

Exposes per-container CPU, memory, network, and disk metrics. Essential for knowing which container is eating resources.

```yaml
cadvisor:
  image: gcr.io/cadvisor/cadvisor:latest
  container_name: cadvisor
  restart: unless-stopped
  privileged: true
  volumes:
    - /:/rootfs:ro
    - /var/run:/var/run:ro
    - /sys:/sys:ro
    - /var/lib/docker/:/var/lib/docker:ro
    - /dev/disk/:/dev/disk:ro
  ports:
    - '8080:8080'
```

**Prometheus scrape config:**
```yaml
- job_name: 'cadvisor'
  scrape_interval: 15s
  static_configs:
    - targets: ['cadvisor:8080']
```

**Grafana community dashboard:** [Docker Container & Host Metrics — ID `10619`](https://grafana.com/grafana/dashboards/10619)

Alternative: [cAdvisor Exporter — ID `14282`](https://grafana.com/grafana/dashboards/14282)

**Key metrics to watch:**
- `container_cpu_usage_seconds_total` — CPU usage per container
- `container_memory_usage_bytes` — Memory per container
- `container_memory_working_set_bytes` — Actual memory pressure (what OOM killer looks at)
- `container_network_receive_bytes_total` / `container_network_transmit_bytes_total`

---

## 3. PostgreSQL <a id="postgresql"></a>

### postgres-exporter

```yaml
postgres-exporter:
  image: prometheuscommunity/postgres-exporter:latest
  container_name: postgres-exporter
  restart: unless-stopped
  environment:
    DATA_SOURCE_NAME: "postgresql://postgres_user:postgres_password@postgres:5432/your_db?sslmode=disable"
  ports:
    - '9187:9187'
  depends_on:
    - postgres
```

**Prometheus scrape config:**
```yaml
- job_name: 'postgres'
  scrape_interval: 15s
  static_configs:
    - targets: ['postgres-exporter:9187']
```

**Grafana community dashboard:** [PostgreSQL Database — ID `9628`](https://grafana.com/grafana/dashboards/9628)

Alternative (more detailed): [PostgreSQL Exporter — ID `455`](https://grafana.com/grafana/dashboards/455)

**Key metrics provided:**
- `pg_stat_activity_count` — Active connections by state (active, idle, idle in transaction)
- `pg_stat_database_tup_fetched` / `tup_returned` / `tup_inserted` / `tup_updated` / `tup_deleted` — Row-level throughput
- `pg_stat_database_blks_hit` / `blks_read` — Buffer cache hit ratio
- `pg_stat_database_deadlocks` — Deadlock count
- `pg_stat_database_conflicts` — Replication conflicts
- `pg_replication_lag` — Replication lag (if using replicas)
- `pg_database_size_bytes` — Database size
- `pg_stat_user_tables_seq_scan` / `idx_scan` — Sequential vs index scans (high seq_scan on large tables = missing indexes)
- `pg_locks_count` — Lock contention

**What this does NOT tell you (your app metrics fill this gap):**
- App-side query latency (includes pool wait, ORM overhead, network)
- Which application code path triggers slow queries
- Connection pool saturation from the app's perspective

**Custom queries** (optional, for deeper visibility): Create a `queries.yaml` to extend the exporter with custom SQL metrics:

```yaml
# custom-queries.yaml
pg_slow_queries:
  query: "SELECT count(*) as count FROM pg_stat_activity WHERE state = 'active' AND now() - query_start > interval '5 seconds'"
  metrics:
    - count:
        usage: "GAUGE"
        description: "Number of queries running longer than 5 seconds"
```

Mount and reference via `--extend.query-path=/etc/postgres-exporter/queries.yaml`.

---

## 4. Redis <a id="redis"></a>

### redis-exporter

```yaml
redis-exporter:
  image: oliver006/redis_exporter:latest
  container_name: redis-exporter
  restart: unless-stopped
  environment:
    REDIS_ADDR: "redis://redis:6379"
    # If auth: REDIS_PASSWORD: "your_password"
  ports:
    - '9121:9121'
  depends_on:
    - redis
```

**Prometheus scrape config:**
```yaml
- job_name: 'redis'
  scrape_interval: 15s
  static_configs:
    - targets: ['redis-exporter:9121']
```

**Grafana community dashboard:** [Redis Dashboard for Prometheus — ID `763`](https://grafana.com/grafana/dashboards/763)

Alternative: [Redis Exporter Quickstart — ID `11835`](https://grafana.com/grafana/dashboards/11835)

**Key metrics provided:**
- `redis_connected_clients` — Current client connections
- `redis_blocked_clients` — Clients blocked on BLPOP/BRPOP
- `redis_used_memory` / `redis_used_memory_rss` — Memory usage (rss is what the OS actually allocates)
- `redis_mem_fragmentation_ratio` — > 1.5 means significant fragmentation
- `redis_keyspace_hits_total` / `redis_keyspace_misses_total` — Global hit/miss ratio
- `redis_commands_processed_total` — Command throughput
- `redis_connected_slaves` — Replication health
- `redis_evicted_keys_total` — Keys evicted due to maxmemory (if this is rising, you're out of memory)
- `redis_rejected_connections_total` — Connection rejections
- `redis_slowlog_length` — Slow command log size

**What this does NOT tell you:**
- Which cache strategy in your app is effective (hit/miss per cache_name)
- App-side Redis operation latency (includes serialization, network)
- Whether your cache-aside, write-through, or TTL strategy is working

---

## 5. RabbitMQ <a id="rabbitmq"></a>

### Built-in Prometheus plugin

RabbitMQ 3.8+ ships with a built-in Prometheus metrics endpoint. No sidecar exporter needed.

```yaml
rabbitmq:
  image: rabbitmq:3-management
  container_name: rabbitmq
  restart: unless-stopped
  environment:
    RABBITMQ_DEFAULT_USER: guest
    RABBITMQ_DEFAULT_PASS: guest
  ports:
    - '5672:5672'    # AMQP
    - '15672:15672'  # Management UI
    - '15692:15692'  # Prometheus metrics
  volumes:
    - ./rabbitmq-enabled-plugins:/etc/rabbitmq/enabled_plugins
```

Enable the plugin:
```bash
# rabbitmq-enabled-plugins file content:
[rabbitmq_management,rabbitmq_prometheus].
```

Or enable at runtime: `rabbitmq-plugins enable rabbitmq_prometheus`

**Prometheus scrape config:**
```yaml
- job_name: 'rabbitmq'
  scrape_interval: 15s
  static_configs:
    - targets: ['rabbitmq:15692']
```

**Grafana community dashboard:** [RabbitMQ Overview — ID `10991`](https://grafana.com/grafana/dashboards/10991)

Alternative (official): [RabbitMQ Quorum Queues — ID `11340`](https://grafana.com/grafana/dashboards/11340)

**Key metrics provided:**
- `rabbitmq_queue_messages` — Messages in queue (ready + unacked)
- `rabbitmq_queue_messages_ready` — Messages waiting to be consumed
- `rabbitmq_queue_messages_unacked` — Messages delivered but not acknowledged
- `rabbitmq_queue_consumers` — Consumer count per queue
- `rabbitmq_channel_messages_published_total` — Publish rate
- `rabbitmq_channel_messages_delivered_total` — Delivery rate
- `rabbitmq_channel_messages_acknowledged_total` — Ack rate
- `rabbitmq_connections` — Total connections
- `rabbitmq_channels` — Total channels
- `rabbitmq_node_mem_used` — Node memory
- `rabbitmq_node_disk_free` — Disk free (critical for flow control)

**What this does NOT tell you:**
- App-side publish/consume latency
- End-to-end message processing time (publish → consume → business logic complete)
- Which consumer is slow or failing

---

## 6. MongoDB <a id="mongodb"></a>

### mongodb-exporter

```yaml
mongodb-exporter:
  image: percona/mongodb_exporter:0.40
  container_name: mongodb-exporter
  restart: unless-stopped
  command:
    - '--mongodb.uri=mongodb://mongo_user:mongo_password@mongodb:27017'
    - '--compatible-mode'
    - '--collect-all'
  ports:
    - '9216:9216'
  depends_on:
    - mongodb
```

**Prometheus scrape config:**
```yaml
- job_name: 'mongodb'
  scrape_interval: 15s
  static_configs:
    - targets: ['mongodb-exporter:9216']
```

**Grafana community dashboard:** [MongoDB Overview — ID `2583`](https://grafana.com/grafana/dashboards/2583)

Alternative (Percona): [MongoDB Exporter — ID `12079`](https://grafana.com/grafana/dashboards/12079)

**Key metrics provided:**
- `mongodb_ss_connections` — Current connections by state
- `mongodb_ss_opcounters` — Operation counters (insert, query, update, delete, getmore, command)
- `mongodb_ss_mem_resident` / `mongodb_ss_mem_virtual` — Memory usage
- `mongodb_ss_globalLock_currentQueue` — Queued operations (read/write)
- `mongodb_ss_wt_cache_bytes_currently_in_cache` — WiredTiger cache usage
- `mongodb_ss_repl_lag` — Replication lag
- `mongodb_dbstats_dataSize` — Database size
- `mongodb_ss_network_bytesIn` / `bytesOut` — Network throughput

---

## 7. Qdrant <a id="qdrant"></a>

### Built-in Prometheus endpoint

Qdrant exposes metrics natively on its REST API. No sidecar exporter needed.

```yaml
qdrant:
  image: qdrant/qdrant:latest
  container_name: qdrant
  restart: unless-stopped
  ports:
    - '6333:6333'  # REST API
    - '6334:6334'  # gRPC
  volumes:
    - qdrant-data:/qdrant/storage
  environment:
    QDRANT__SERVICE__ENABLE_METRICS: "true"
```

Metrics endpoint: `http://qdrant:6333/metrics`

**Prometheus scrape config:**
```yaml
- job_name: 'qdrant'
  scrape_interval: 15s
  static_configs:
    - targets: ['qdrant:6333']
  metrics_path: /metrics
```

**Grafana community dashboard:** [Qdrant Dashboard — ID `18486`](https://grafana.com/grafana/dashboards/18486)

There's limited community dashboard support compared to more established databases. You may need to build a custom dashboard. Key panels to include:

**Key metrics provided:**
- `app_info` — Qdrant version and metadata
- `collections_total` — Number of collections
- `collections_vector_total` — Total vectors stored
- `rest_responses_total` — REST API request count by method and status
- `rest_responses_duration_seconds` — REST API latency
- `grpc_responses_total` — gRPC request count
- `grpc_responses_duration_seconds` — gRPC latency
- `qdrant_search_duration_seconds` — Search (ANN query) latency

**What this does NOT tell you:**
- App-side vector search latency (includes serialization of vectors, network, result parsing)
- Which collection or search strategy your app finds slow
- Relevance quality (that's an application-level concern)

---

## 8. Elasticsearch / OpenSearch <a id="elasticsearch"></a>

### elasticsearch-exporter

```yaml
elasticsearch-exporter:
  image: quay.io/prometheuscommunity/elasticsearch-exporter:latest
  container_name: elasticsearch-exporter
  restart: unless-stopped
  command:
    - '--es.uri=http://elasticsearch:9200'
    - '--es.all'
    - '--es.indices'
  ports:
    - '9114:9114'
  depends_on:
    - elasticsearch
```

**Prometheus scrape config:**
```yaml
- job_name: 'elasticsearch'
  scrape_interval: 15s
  static_configs:
    - targets: ['elasticsearch-exporter:9114']
```

**Grafana community dashboard:** [Elasticsearch Exporter — ID `2322`](https://grafana.com/grafana/dashboards/2322)

Alternative: [Elasticsearch Cluster — ID `6483`](https://grafana.com/grafana/dashboards/6483)

**Key metrics provided:**
- `elasticsearch_cluster_health_status` — Cluster health (green/yellow/red)
- `elasticsearch_cluster_health_number_of_nodes` / `number_of_data_nodes`
- `elasticsearch_indices_docs_total` — Total documents
- `elasticsearch_indices_store_size_bytes` — Index storage size
- `elasticsearch_jvm_memory_used_bytes` / `max_bytes` — JVM heap
- `elasticsearch_indices_search_query_total` — Query count
- `elasticsearch_indices_search_query_time_seconds` — Server-side query time
- `elasticsearch_indices_indexing_index_total` — Indexing throughput
- `elasticsearch_transport_tx_size_bytes_total` — Inter-node traffic

---

## 9. Nginx / Reverse Proxy <a id="nginx"></a>

### nginx-prometheus-exporter

Requires `ngx_http_stub_status_module` enabled in Nginx.

```nginx
# nginx.conf — add this server block or location
server {
    listen 8081;
    location /stub_status {
        stub_status;
        allow 172.0.0.0/8;  # Docker network
        deny all;
    }
}
```

```yaml
nginx-exporter:
  image: nginx/nginx-prometheus-exporter:latest
  container_name: nginx-exporter
  restart: unless-stopped
  command:
    - '-nginx.scrape-uri=http://nginx:8081/stub_status'
  ports:
    - '9113:9113'
  depends_on:
    - nginx
```

**Prometheus scrape config:**
```yaml
- job_name: 'nginx'
  scrape_interval: 15s
  static_configs:
    - targets: ['nginx-exporter:9113']
```

**Grafana community dashboard:** [NGINX — ID `12708`](https://grafana.com/grafana/dashboards/12708)

**Key metrics provided:**
- `nginx_connections_active` — Active client connections
- `nginx_connections_accepted` / `handled` — Connection handling
- `nginx_http_requests_total` — Total requests
- `nginx_connections_reading` / `writing` / `waiting` — Connection states

For richer metrics, use the Nginx VTS module or switch to the NGINX Plus API exporter.

---

## 10. MinIO / S3-Compatible Storage <a id="minio"></a>

### Built-in Prometheus endpoint

MinIO exposes metrics natively.

```yaml
minio:
  image: minio/minio:latest
  container_name: minio
  restart: unless-stopped
  command: server /data --console-address ":9001"
  environment:
    MINIO_ROOT_USER: minioadmin
    MINIO_ROOT_PASSWORD: minioadmin
    MINIO_PROMETHEUS_AUTH_TYPE: public
  ports:
    - '9000:9000'
    - '9001:9001'
```

Metrics endpoint: `http://minio:9000/minio/v2/metrics/cluster`

**Prometheus scrape config:**
```yaml
- job_name: 'minio'
  scrape_interval: 15s
  metrics_path: /minio/v2/metrics/cluster
  static_configs:
    - targets: ['minio:9000']
```

**Grafana community dashboard:** [MinIO Dashboard — ID `13502`](https://grafana.com/grafana/dashboards/13502)

**Key metrics provided:**
- `minio_s3_requests_total` — S3 API request count by method
- `minio_s3_traffic_received_bytes` / `sent_bytes` — Network throughput
- `minio_bucket_usage_total_bytes` — Storage per bucket
- `minio_bucket_objects_count` — Object count per bucket
- `minio_node_disk_used_bytes` / `free_bytes` — Disk usage

---

## 11. Complete Prometheus Scrape Config <a id="full-scrape-config"></a>

Combine all exporters into a single Prometheus config. Comment out services you don't use.

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  # Application
  - job_name: 'nestjs-app'
    scrape_interval: 15s
    static_configs:
      - targets: ['nestjs-app:3000']
    metrics_path: /metrics

  # Host
  - job_name: 'node-exporter'
    scrape_interval: 15s
    static_configs:
      - targets: ['node-exporter:9100']

  # Containers
  - job_name: 'cadvisor'
    scrape_interval: 15s
    static_configs:
      - targets: ['cadvisor:8080']

  # PostgreSQL
  - job_name: 'postgres'
    scrape_interval: 15s
    static_configs:
      - targets: ['postgres-exporter:9187']

  # Redis
  - job_name: 'redis'
    scrape_interval: 15s
    static_configs:
      - targets: ['redis-exporter:9121']

  # RabbitMQ
  - job_name: 'rabbitmq'
    scrape_interval: 15s
    static_configs:
      - targets: ['rabbitmq:15692']

  # MongoDB
  - job_name: 'mongodb'
    scrape_interval: 15s
    static_configs:
      - targets: ['mongodb-exporter:9216']

  # Qdrant
  - job_name: 'qdrant'
    scrape_interval: 15s
    static_configs:
      - targets: ['qdrant:6333']
    metrics_path: /metrics

  # Elasticsearch
  - job_name: 'elasticsearch'
    scrape_interval: 15s
    static_configs:
      - targets: ['elasticsearch-exporter:9114']

  # Nginx
  - job_name: 'nginx'
    scrape_interval: 15s
    static_configs:
      - targets: ['nginx-exporter:9113']

  # MinIO
  - job_name: 'minio'
    scrape_interval: 15s
    metrics_path: /minio/v2/metrics/cluster
    static_configs:
      - targets: ['minio:9000']
```

---

## 12. Dashboard Organization <a id="dashboard-org"></a>

Recommended Grafana folder structure:

```
Folder: Infrastructure
├── Host Overview          (node-exporter, dashboard 1860)
├── Containers             (cAdvisor, dashboard 10619)
├── PostgreSQL             (postgres-exporter, dashboard 9628)
├── Redis                  (redis-exporter, dashboard 763)
├── RabbitMQ               (built-in, dashboard 10991)
├── MongoDB                (mongodb-exporter, dashboard 2583)
├── Qdrant                 (built-in or custom)
├── Elasticsearch          (elasticsearch-exporter, dashboard 2322)
├── Nginx                  (nginx-exporter, dashboard 12708)
└── MinIO                  (built-in, dashboard 13502)

Folder: NestJS App
├── Service Overview       (Stage 1 app metrics)
├── Dependency Health      (Stage 2 app metrics)
├── Business Metrics       (Stage 3 app metrics)
└── Operational Control    (Stage 4 app metrics)
```

**Importing community dashboards**: In Grafana, go to Dashboards → New → Import → Enter the dashboard ID. Select your Prometheus data source. Adjust variables if needed.

**Linking infra and app dashboards**: Use Grafana's dashboard links feature to cross-reference. From the PostgreSQL infra dashboard, link to the Dependency Health app dashboard. This lets an on-call engineer jump between "what PostgreSQL reports internally" and "what our app experiences."

### Quick Reference: Exporter and Dashboard IDs

| Service | Exporter Image | Port | Dashboard ID |
|---|---|---|---|
| Host | `prom/node-exporter` | 9100 | 1860 |
| Containers | `gcr.io/cadvisor/cadvisor` | 8080 | 10619 |
| PostgreSQL | `prometheuscommunity/postgres-exporter` | 9187 | 9628 |
| Redis | `oliver006/redis_exporter` | 9121 | 763 |
| RabbitMQ | built-in plugin | 15692 | 10991 |
| MongoDB | `percona/mongodb_exporter` | 9216 | 2583 |
| Qdrant | built-in | 6333 | 18486 |
| Elasticsearch | `prometheuscommunity/elasticsearch-exporter` | 9114 | 2322 |
| Nginx | `nginx/nginx-prometheus-exporter` | 9113 | 12708 |
| MinIO | built-in | 9000 | 13502 |
