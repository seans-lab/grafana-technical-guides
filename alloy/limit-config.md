# Grafana Alloy — Configuration to Prevent Grafana Cloud Limits

**Version:** 1.0
**Audience:** Platform Engineers, SRE Teams, Observability Engineers
**Scope:** Alloy configurations to manage and prevent Grafana Cloud rate limits and ingestion limits

---

## ⚠️ Important Notice

```
┌─────────────────────────────────────────────────────────────────┐
│                     IMPORTANT NOTICE                             │
│                                                                  │
│  Grafana Cloud limits vary significantly by plan tier:          │
│                                                                  │
│  Free    → Strictest limits                                     │
│  Pro     → Higher limits, some configurable                     │
│  Advanced → Highest limits, fully configurable                  │
│                                                                  │
│  Always verify your specific limits at:                         │
│  your-stack.grafana.net → Administration → Billing & Usage      │
│                                                                  │
│  The configurations in this guide implement best practices      │
│  for limit management across all tiers.                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📋 Table of Contents

1. [Understanding Grafana Cloud Limits](#limits)
2. [Metrics (Mimir) — Limit Prevention](#metrics)
3. [Logs (Loki) — Limit Prevention](#logs)
4. [Traces (Tempo) — Limit Prevention](#traces)
5. [Profiles (Pyroscope) — Limit Prevention](#profiles)
6. [Global Alloy Configuration](#global)
7. [Complete Production Configuration](#complete)
8. [Monitoring Your Usage](#monitoring)
9. [Best Practices](#bestpractices)

---

<a name="limits"></a>
## 1. 📊 Understanding Grafana Cloud Limits

### Key Limit Categories

```
METRICS (Mimir)
├── Active series limit          → Max unique time series
├── Ingestion rate limit         → Samples per second
├── Remote write request rate    → Requests per second
├── Label count per series       → Max labels per metric
├── Label name length            → Max characters in label name
├── Label value length           → Max characters in label value
├── Metric name length           → Max characters in metric name
└── Out-of-order samples         → Time window for late samples

LOGS (Loki)
├── Ingestion rate limit         → MB/s per tenant
├── Ingestion burst size         → MB burst allowance
├── Max streams per push         → Streams per request
├── Max label count per stream   → Labels per log stream
├── Max label name length        → Characters in label name
├── Max label value length       → Characters in label value
├── Max line size                → Bytes per log line
└── Max entries per push         → Log lines per request

TRACES (Tempo)
├── Ingestion rate limit         → Spans per second
├── Ingestion burst size         → Spans burst allowance
├── Max bytes per trace          → Max trace size
├── Max search tag values        → Tag value cardinality
└── Trace duration limit         → Max trace duration

PROFILES (Pyroscope)
├── Ingestion rate limit         → MB/s
└── Series limit                 → Active profiling series
```

---

### Limit Impact Summary

```
Limit Hit          → What Happens
─────────────────────────────────────────────────────────────
Metrics rate       → 429 Too Many Requests, samples dropped
Metrics series     → New series rejected, existing OK
Loki rate          → 429 ingestion errors, logs lost
Loki streams       → New stream pushes rejected
Tempo rate         → Spans dropped at ingestion
WAL not configured → Data loss on Alloy restart
No backpressure    → Memory exhaustion and OOM crash
```

---

<a name="metrics"></a>
## 2. 📈 Metrics (Mimir) — Limit Prevention

### 2.1 — Remote Write with Full Backpressure & Retry Configuration

```river
// ============================================================
// prometheus.remote_write — Production hardened configuration
// Prevents hitting Mimir ingestion rate limits
// ============================================================
prometheus.remote_write "grafana_cloud_metrics" {
  endpoint {
    url = env("PROMETHEUS_URL")

    basic_auth {
      username = env("PROMETHEUS_USERNAME")
      password = env("GRAFANA_CLOUD_API_KEY")
    }

    // ========================================================
    // TLS — always verify in production
    // ========================================================
    tls_config {
      insecure_skip_verify = false
    }

    // ========================================================
    // Queue configuration
    // Tuned to respect rate limits while maximising throughput
    // ========================================================
    queue_config {
      // Number of concurrent shards sending data
      // Start low and increase if you have headroom
      // Each shard = one HTTP connection to Mimir
      min_shards = 1
      max_shards = 10       // Limit parallel connections

      // Samples buffered per shard before blocking
      // Higher = more memory, better throughput
      capacity = 5000

      // Max samples per remote write request
      // Grafana Cloud default max is 500 per request
      // Keep below your plan limit
      max_samples_per_send = 500

      // How long to wait to fill a batch
      // Longer = fewer requests, better for rate limits
      batch_send_deadline = "10s"

      // Retry configuration
      // Prevents hammering Mimir when limits are hit
      min_backoff    = "1s"
      max_backoff    = "256s"   // ~4 minutes max backoff
      retry_on_http_429 = true  // Retry on rate limit response
    }

    // ========================================================
    // Metadata configuration
    // Reduce metadata write frequency to save quota
    // ========================================================
    metadata_config {
      send          = true
      send_interval = "5m"    // Default is 1m — reduce to 5m
      max_samples_per_send = 500
    }
  }

  // ========================================================
  // Write-Ahead Log (WAL)
  // Prevents data loss during Alloy restarts or
  // temporary Mimir unavailability
  // ========================================================
  wal {
    // How often to truncate the WAL
    truncate_frequency = "2h"

    // How long to keep data in WAL
    // Long enough to survive extended outages
    max_keepalive_time = "8h"

    // Minimum time to keep data in WAL
    min_keepalive_time = "1h"
  }

  // ========================================================
  // External labels — mandatory for multi-tenant routing
  // ========================================================
  external_labels = {
    cluster     = env("CLUSTER_NAME"),
    environment = env("ENVIRONMENT"),
    team        = env("TEAM_NAME"),
  }
}
```

---

### 2.2 — Aggressive Metric Relabelling to Reduce Series

```river
// ============================================================
// prometheus.relabel — Drop high-cardinality and
// unnecessary metrics BEFORE sending to Grafana Cloud
// Every dropped series = reduced active series count
// ============================================================
prometheus.relabel "reduce_cardinality" {

  // ========================================================
  // Drop metrics you do not use
  // Audit your dashboards and alerts — drop everything else
  // ========================================================

  // Drop all Go runtime metrics (usually not needed)
  rule {
    source_labels = ["__name__"]
    regex         = "go_.*"
    action        = "drop"
  }

  // Drop process metrics (usually covered by node_exporter)
  rule {
    source_labels = ["__name__"]
    regex         = "process_.*"
    action        = "drop"
  }

  // Drop scrape metadata metrics
  rule {
    source_labels = ["__name__"]
    regex         = "scrape_.*"
    action        = "drop"
  }

  // Drop promhttp metrics
  rule {
    source_labels = ["__name__"]
    regex         = "promhttp_.*"
    action        = "drop"
  }

  // Drop net_conntrack metrics (rarely needed)
  rule {
    source_labels = ["__name__"]
    regex         = "net_conntrack_.*"
    action        = "drop"
  }

  // ========================================================
  // Drop high-cardinality labels that add series
  // without adding query value
  // ========================================================

  // Remove pod hash labels (extremely high cardinality)
  rule {
    action = "labeldrop"
    regex  = "pod_template_hash"
  }

  // Remove controller revision hash
  rule {
    action = "labeldrop"
    regex  = "controller_revision_hash"
  }

  // Remove statefulset pod name (use pod label instead)
  rule {
    action = "labeldrop"
    regex  = "statefulset_kubernetes_io_pod_name"
  }

  // Remove helm chart labels (version changes = new series)
  rule {
    action = "labeldrop"
    regex  = "helm_sh_chart|app_kubernetes_io_version"
  }

  // Remove long uid labels
  rule {
    action = "labeldrop"
    regex  = ".*_uid"
  }

  // Remove node labels that add cardinality
  rule {
    action = "labeldrop"
    regex  = "beta_kubernetes_io_.*|topology_kubernetes_io_.*"
  }

  // ========================================================
  // Drop specific high-volume, low-value metrics
  // Tune these based on your actual usage
  // ========================================================

  // Drop per-bucket histogram metrics you do not query
  rule {
    source_labels = ["__name__"]
    regex         = "apiserver_request_duration_seconds_bucket"
    action        = "drop"
  }

  // Drop etcd histogram buckets (use summary instead)
  rule {
    source_labels = ["__name__"]
    regex         = "etcd_request_duration_seconds_bucket"
    action        = "drop"
  }

  // Drop kubelet volume metrics (high cardinality)
  rule {
    source_labels = ["__name__"]
    regex         = "kubelet_volume_stats_.*"
    action        = "drop"
  }

  // ========================================================
  // Truncate excessively long label values
  // Prevents label value length limit errors
  // ========================================================
  rule {
    source_labels = ["__path__"]
    regex         = "(.{0,128}).*"
    replacement   = "$1"
    target_label  = "__path__"
  }

  forward_to = [prometheus.remote_write.grafana_cloud_metrics.receiver]
}
```

---

### 2.3 — Scrape Configuration to Reduce Ingestion Rate

```river
// ============================================================
// prometheus.scrape — Tuned scrape intervals
// Longer intervals = fewer samples = lower ingestion rate
// ============================================================

// High frequency — only for SLO-critical metrics
prometheus.scrape "critical_services" {
  targets         = discovery.relabel.critical_pods.output
  forward_to      = [prometheus.relabel.reduce_cardinality.receiver]

  // 15s only for truly critical metrics
  scrape_interval = "15s"
  scrape_timeout  = "10s"

  // Limit body size to prevent memory spikes
  body_size_limit = "10MB"
}

// Standard frequency — most application metrics
prometheus.scrape "standard_services" {
  targets         = discovery.relabel.standard_pods.output
  forward_to      = [prometheus.relabel.reduce_cardinality.receiver]

  // 60s is sufficient for most application metrics
  // Doubles sampling from 30s = halves series rate
  scrape_interval = "60s"
  scrape_timeout  = "15s"

  body_size_limit = "10MB"
}

// Low frequency — infrastructure and background metrics
prometheus.scrape "infrastructure" {
  targets         = discovery.relabel.infra_pods.output
  forward_to      = [prometheus.relabel.reduce_cardinality.receiver]

  // 120s for slow-changing infrastructure metrics
  scrape_interval = "120s"
  scrape_timeout  = "30s"

  body_size_limit = "10MB"
}

// ============================================================
// Kubernetes node metrics — use honour_labels carefully
// ============================================================
prometheus.scrape "node_metrics" {
  targets         = discovery.relabel.nodes.output
  forward_to      = [prometheus.relabel.reduce_cardinality.receiver]

  scrape_interval = "60s"
  scrape_timeout  = "30s"

  // Limit metrics path to prevent scraping everything
  metrics_path    = "/metrics"

  body_size_limit = "20MB"

  // Honour labels from targets but do not allow
  // them to override existing labels
  honor_labels    = false
}
```

---

### 2.4 — Sample Limiting per Scrape Target

```river
// ============================================================
// Enforce per-target sample limits
// Prevents a single misbehaving service from consuming
// your entire metrics quota
// ============================================================
prometheus.scrape "with_sample_limits" {
  targets    = discovery.relabel.all_pods.output
  forward_to = [prometheus.relabel.reduce_cardinality.receiver]

  scrape_interval = "60s"

  // Hard limit: reject scrape if more than
  // this many samples are returned
  // Protects against metric explosion
  sample_limit = 2000

  // Limit unique label sets per scrape
  // Prevents high-cardinality series explosions
  target_limit = 1000

  // Limit number of labels per sample
  label_limit = 30

  // Max length of label name
  label_name_length_limit = 1024

  // Max length of label value
  label_value_length_limit = 2048
}
```

---

### 2.5 — Metric Aggregation to Reduce Series Count

```river
// ============================================================
// prometheus.relabel — Aggregate before sending
// Reduces series by removing instance-level granularity
// where not needed
// ============================================================
prometheus.relabel "aggregate_by_service" {

  // For request rate metrics, aggregate by service
  // instead of by individual pod instance
  // This can reduce series by 10x in large clusters

  // Remove instance label from HTTP metrics
  // (aggregate across all pods of a service)
  rule {
    source_labels = ["__name__"]
    regex         = "http_requests_total|http_request_duration.*"
    action        = "keep"
  }

  rule {
    action = "labeldrop"
    regex  = "instance"
  }

  // Keep pod label only for error-rate metrics
  // where per-pod debugging is needed
  rule {
    source_labels = ["__name__"]
    regex         = ".*_errors_total"
    action        = "keep"
  }

  forward_to = [prometheus.remote_write.grafana_cloud_metrics.receiver]
}
```

---

<a name="logs"></a>
## 3. 📜 Logs (Loki) — Limit Prevention

### 3.1 — Loki Write with Rate Limiting

```river
// ============================================================
// loki.write — Production configuration with rate limiting
// Prevents hitting Loki ingestion rate and burst limits
// ============================================================
loki.write "grafana_cloud_logs" {
  endpoint {
    url = env("LOKI_URL")

    basic_auth {
      username = env("LOKI_USERNAME")
      password = env("GRAFANA_CLOUD_API_KEY")
    }

    tls_config {
      insecure_skip_verify = false
    }

    // ========================================================
    // Backpressure and retry configuration
    // ========================================================
    // Remote timeout — do not wait forever
    remote_timeout = "10s"

    // Batch configuration
    // Controls how aggressively logs are sent
    batch_size  = 1048576   // 1MB per batch
    batch_wait  = "1s"      // Wait up to 1s to fill batch

    // Retry on failure
    min_backoff = "500ms"
    max_backoff = "5m"
    max_retries = 10
  }

  // ========================================================
  // External labels — applied to ALL log streams
  // Keep this minimal to avoid label cardinality issues
  // ========================================================
  external_labels = {
    cluster     = env("CLUSTER_NAME"),
    environment = env("ENVIRONMENT"),
  }
}
```

---

### 3.2 — Log Processing to Reduce Volume

```river
// ============================================================
// loki.process — Reduce log volume before sending
// Directly reduces ingestion rate and costs
// ============================================================
loki.process "reduce_log_volume" {

  // ========================================================
  // Stage 1: Drop debug and trace logs in production
  // These are the highest volume, lowest value logs
  // ========================================================
  stage.match {
    selector = "{environment=\"production\"}"

    stage.json {
      expressions = {
        log_level = "level",
      }
    }

    // Drop debug logs
    stage.drop {
      source    = "log_level"
      value     = "debug"
      drop_counter_reason = "debug_log_dropped"
    }

    // Drop trace logs
    stage.drop {
      source    = "log_level"
      value     = "trace"
      drop_counter_reason = "trace_log_dropped"
    }
  }

  // ========================================================
  // Stage 2: Drop health check and liveness probe logs
  // These are extremely high volume and no value
  // ========================================================
  stage.drop {
    expression = ".*GET /health.*200.*"
    drop_counter_reason = "healthcheck_dropped"
  }

  stage.drop {
    expression = ".*GET /ready.*200.*"
    drop_counter_reason = "readiness_dropped"
  }

  stage.drop {
    expression = ".*GET /livez.*200.*"
    drop_counter_reason = "liveness_dropped"
  }

  stage.drop {
    expression = ".*GET /metrics.*200.*"
    drop_counter_reason = "metrics_scrape_dropped"
  }

  // Drop favicon requests
  stage.drop {
    expression = ".*favicon.ico.*"
    drop_counter_reason = "favicon_dropped"
  }

  // ========================================================
  // Stage 3: Rate limit per stream
  // Prevents any single stream from consuming all quota
  // ========================================================
  stage.limit {
    rate            = 10000    // Lines per second per stream
    burst           = 20000    // Allow burst up to this
    by_label_name   = "pod"    // Rate limit per pod
    drop            = true     // Drop instead of blocking
  }

  // ========================================================
  // Stage 4: Truncate excessively long log lines
  // Grafana Cloud has a max line size limit
  // Default limit is often 65536 bytes (64KB)
  // ========================================================
  stage.replace {
    // Truncate any single field value over 4096 chars
    expression = `(.{4096}).+`
    replace    = "$1 [TRUNCATED]"
  }

  // ========================================================
  // Stage 5: Remove high-cardinality dynamic labels
  // Each unique label combination = a new Loki stream
  // Too many streams = stream limit errors
  // ========================================================
  stage.label_drop {
    values = [
      "filename",       // File paths are high cardinality
      "stream",         // stdout/stderr adds no value as label
    ]
  }

  // ========================================================
  // Stage 6: Sampling for verbose services
  // Only keep 10% of info logs from noisy services
  // ========================================================
  stage.match {
    selector = "{app=\"api-gateway\"}"

    stage.json {
      expressions = {
        level = "level",
      }
    }

    stage.match {
      selector = "{level=\"info\"}"

      stage.sampling {
        rate = 0.1    // Keep 10% of info logs
      }
    }
  }

  forward_to = [loki.write.grafana_cloud_logs.receiver]
}
```

---

### 3.3 — Loki Source Configuration to Control Streams

```river
// ============================================================
// loki.source.kubernetes — Controlled log collection
// Fewer labels = fewer streams = lower stream count
// ============================================================
loki.source.kubernetes "pods" {
  targets    = discovery.relabel.pod_logs.output
  forward_to = [loki.process.reduce_log_volume.receiver]
}

// ============================================================
// discovery.relabel — Control which labels become
// Loki stream labels
// CRITICAL: Every unique label combination = new stream
// ============================================================
discovery.relabel "pod_logs" {
  targets = discovery.kubernetes.pods.targets

  // ========================================================
  // Only keep the labels you ACTUALLY filter by in Loki
  // Every additional label exponentially increases streams
  // ========================================================

  // Keep: namespace (low cardinality, high value)
  rule {
    source_labels = ["__meta_kubernetes_namespace"]
    target_label  = "namespace"
  }

  // Keep: app name (medium cardinality, essential)
  rule {
    source_labels = [
      "__meta_kubernetes_pod_label_app_kubernetes_io_name"
    ]
    target_label = "app"
  }

  // Keep: pod name for debugging (high cardinality — consider
  // removing if stream count is too high)
  rule {
    source_labels = ["__meta_kubernetes_pod_name"]
    target_label  = "pod"
  }

  // ========================================================
  // DROP these — they increase stream count significantly
  // ========================================================

  // Do NOT keep: pod template hash
  // Do NOT keep: deployment revision
  // Do NOT keep: full container image with digest
  // Do NOT keep: node name (use namespace + pod instead)
  // Do NOT keep: uid fields

  // ========================================================
  // Namespace allow-list — only collect from app namespaces
  // Drop system namespace logs (handled by platform Alloy)
  // ========================================================
  rule {
    source_labels = ["__meta_kubernetes_namespace"]
    regex         = "kube-system|kube-public|monitoring|flux-system"
    action        = "drop"
  }

  // Only collect from pods with logging annotation
  rule {
    source_labels = [
      "__meta_kubernetes_pod_annotation_logs_grafana_com_scrape"
    ]
    regex  = "true"
    action = "keep"
  }
}
```

---

### 3.4 — Deduplication to Reduce Repeated Log Lines

```river
// ============================================================
// loki.process — Deduplicate repeated log lines
// Prevents log storms from filling ingestion quota
// ============================================================
loki.process "deduplicate_logs" {

  // Detect and collapse repeated identical log lines
  // Replaces N identical consecutive lines with one line
  // and a "repeated N times" marker
  stage.decolorize {}   // Remove ANSI colour codes first

  // Use eventlogmessage to deduplicate
  stage.match {
    selector = "{}"

    stage.template {
      source   = "dedupe_key"
      template = "{{ .message }}"
    }

    // Rate limit based on message content
    stage.limit {
      rate          = 100
      burst         = 200
      by_label_name = "message"
      drop          = true
    }
  }

  forward_to = [loki.write.grafana_cloud_logs.receiver]
}
```

---

<a name="traces"></a>
## 4. 🔍 Traces (Tempo) — Limit Prevention

### 4.1 — Trace Sampling to Control Span Ingestion Rate

```river
// ============================================================
// Trace sampling strategies to prevent hitting Tempo limits
// Choose the strategy that fits your use case
// ============================================================

// ============================================================
// Strategy 1: Probabilistic Sampling (Simple)
// Use when: You want a fixed percentage of traces
// ============================================================
otelcol.processor.probabilistic_sampler "fixed_rate" {
  // Keep 10% of all traces
  // Adjust based on your Tempo ingestion limit
  sampling_percentage = 10

  output {
    traces = [otelcol.processor.batch.default.input]
  }
}

// ============================================================
// Strategy 2: Tail-based Sampling (Recommended)
// Use when: You want to keep ALL error traces but
// sample successful traces
// This ensures you never miss important traces
// ============================================================
otelcol.processor.tail_sampling "intelligent" {

  // Wait for all spans of a trace before deciding
  decision_wait = "10s"

  // Max traces held in memory waiting for decision
  // Lower = less memory but may miss late spans
  num_traces = 50000

  // ========================================================
  // Policy 1: Always keep ERROR traces (100%)
  // ========================================================
  policy {
    name = "errors-policy"
    type = "status_code"

    status_code {
      status_codes = ["ERROR"]
    }
  }

  // ========================================================
  // Policy 2: Always keep SLOW traces (> 1 second)
  // ========================================================
  policy {
    name = "slow-traces-policy"
    type = "latency"

    latency {
      threshold_ms = 1000
    }
  }

  // ========================================================
  // Policy 3: Sample 5% of successful fast traces
  // ========================================================
  policy {
    name = "success-sample-policy"
    type = "probabilistic"

    probabilistic {
      sampling_percentage = 5
    }
  }

  // ========================================================
  // Policy 4: Always keep traces from critical services
  // ========================================================
  policy {
    name = "critical-services-policy"
    type = "string_attribute"

    string_attribute {
      key    = "service.name"
      values = ["payments-api", "trading-engine"]
    }
  }

  output {
    traces = [otelcol.processor.batch.default.input]
  }
}

// ============================================================
// Strategy 3: Per-service rate limiting
// Use when: Specific services are too noisy
// ============================================================
otelcol.processor.filter "drop_noisy_services" {
  error_mode = "ignore"

  traces {
    span = [
      // Drop health check traces entirely
      "attributes[\"http.target\"] == \"/health\"",
      "attributes[\"http.target\"] == \"/ready\"",
      "attributes[\"http.target\"] == \"/metrics\"",
      // Drop traces from known noisy background jobs
      "attributes[\"messaging.destination\"] == \"heartbeat-queue\"",
    ]
  }

  output {
    traces = [otelcol.processor.probabilistic_sampler.fixed_rate.input]
  }
}
```

---

### 4.2 — Batch Processor to Optimise Tempo Requests

```river
// ============================================================
// otelcol.processor.batch — Batch traces before sending
// Reduces number of HTTP requests to Tempo
// Prevents hitting request rate limits
// ============================================================
otelcol.processor.batch "default" {
  // Wait up to 5 seconds to fill a batch
  timeout = "5s"

  // Target batch size in spans
  send_batch_size = 1000

  // Hard limit on batch size
  send_batch_max_size = 2000

  output {
    traces = [otelcol.exporter.otlphttp.grafana_cloud_traces.input]
  }
}
```

---

### 4.3 — Trace Attribute Management

```river
// ============================================================
// otelcol.processor.attributes — Clean up trace attributes
// Reduces payload size and prevents attribute limits
// ============================================================
otelcol.processor.attributes "clean_traces" {

  // Add mandatory attributes
  action {
    key    = "cluster"
    value  = env("CLUSTER_NAME")
    action = "upsert"
  }

  action {
    key    = "environment"
    value  = env("ENVIRONMENT")
    action = "upsert"
  }

  // Remove high-cardinality attributes that
  // add no query value but increase payload size
  action {
    key    = "http.request.header.authorization"
    action = "delete"
  }

  action {
    key    = "http.request.header.cookie"
    action = "delete"
  }

  action {
    key    = "http.response.header.set-cookie"
    action = "delete"
  }

  // Remove internal implementation details
  action {
    key    = "thread.id"
    action = "delete"
  }

  action {
    key    = "thread.name"
    action = "delete"
  }

  output {
    traces = [otelcol.processor.batch.default.input]
  }
}
```

---

### 4.4 — Tempo Exporter with Retry

```river
// ============================================================
// otelcol.exporter.otlphttp — Tempo exporter with
// retry and timeout configuration
// ============================================================
otelcol.exporter.otlphttp "grafana_cloud_traces" {
  client {
    endpoint = env("TEMPO_URL")

    auth = otelcol.auth.basic.grafana_cloud.handler

    tls {
      insecure             = false
      insecure_skip_verify = false
    }

    // Timeout for individual requests
    timeout = "30s"

    // Retry on failure
    retry_on_failure {
      enabled          = true
      initial_interval = "1s"
      max_interval     = "30s"
      max_elapsed_time = "5m"
    }

    // Sending queue to buffer during outages
    sending_queue {
      enabled       = true
      num_consumers = 4
      queue_size    = 1000
    }
  }
}

otelcol.auth.basic "grafana_cloud" {
  username = env("TEMPO_USERNAME")
  password = env("GRAFANA_CLOUD_API_KEY")
}
```

---

<a name="profiles"></a>
## 5. 🔥 Profiles (Pyroscope) — Limit Prevention

```river
// ============================================================
// pyroscope.write — Profile ingestion with rate control
// ============================================================
pyroscope.write "grafana_cloud_profiles" {
  endpoint {
    url = env("PYROSCOPE_URL")

    basic_auth {
      username = env("PYROSCOPE_USERNAME")
      password = env("GRAFANA_CLOUD_API_KEY")
    }
  }

  external_labels = {
    cluster     = env("CLUSTER_NAME"),
    environment = env("ENVIRONMENT"),
  }
}

// ============================================================
// pyroscope.scrape — Selective profiling
// Only profile services that need it
// Profiling everything = quota exhausted quickly
// ============================================================
pyroscope.scrape "selective_profiling" {
  targets    = discovery.relabel.profile_targets.output
  forward_to = [pyroscope.write.grafana_cloud_profiles.receiver]

  // Only scrape these profile types to reduce volume
  profiling_config {
    // CPU profiling — most valuable, keep enabled
    profile.process_cpu {
      enabled = true
      delta   = true
    }

    // Memory profiling — enable only if investigating
    profile.memory {
      enabled = false    // Disable by default — high volume
    }

    // Block profiling — disable by default
    profile.block {
      enabled = false
    }

    // Mutex profiling — disable by default
    profile.mutex {
      enabled = false
    }

    // Goroutine profiling — enable selectively
    profile.goroutine {
      enabled = false
    }
  }

  // Scrape profiles every 60s (default is 15s)
  // 4x reduction in profile ingestion rate
  scrape_interval = "60s"
  scrape_timeout  = "15s"
}

// ============================================================
// discovery.relabel — Only profile specific services
// Not every service needs continuous profiling
// ============================================================
discovery.relabel "profile_targets" {
  targets = discovery.kubernetes.pods.targets

  // Only profile pods with profiling annotation
  rule {
    source_labels = [
      "__meta_kubernetes_pod_annotation_profiles_grafana_com_scrape"
    ]
    regex  = "true"
    action = "keep"
  }

  // Only profile production namespace
  rule {
    source_labels = ["__meta_kubernetes_namespace"]
    regex         = env("PROFILING_NAMESPACES")
    action        = "keep"
  }
}
```

---

<a name="global"></a>
## 6. ⚙️ Global Alloy Configuration

### 6.1 — Memory and Resource Management

```river
// ============================================================
// logging — Reduce Alloy's own log volume
// ============================================================
logging {
  level  = "warn"     // warn in production, info for debugging
  format = "json"
}

// ============================================================
// tracing — Disable Alloy's own internal tracing
// Reduces resource usage and self-generated trace volume
// ============================================================
tracing {
  sampling_fraction = 0.0
}
```

---

### 6.2 — OTLP Receiver with Backpressure

```river
// ============================================================
// otelcol.receiver.otlp — Receiver with limits enforced
// Prevents Alloy from being overwhelmed by application
// telemetry and running out of memory
// ============================================================
otelcol.receiver.otlp "with_limits" {
  grpc {
    endpoint = "0.0.0.0:4317"

    // Limit max message size to prevent OOM
    max_recv_msg_size_mib = 4    // 4MB max per gRPC message

    // Keepalive settings
    keepalive {
      server_parameters {
        max_connection_idle     = "11s"
        max_connection_age      = "12s"
        max_connection_age_grace = "5s"
        time                    = "10s"
        timeout                 = "3s"
      }
    }
  }

  http {
    endpoint = "0.0.0.0:4318"

    // Limit max request body size
    max_request_body_size = "4194304"    // 4MB
  }

  output {
    metrics = [otelcol.processor.memory_limiter.default.input]
    logs    = [otelcol.processor.memory_limiter.default.input]
    traces  = [otelcol.processor.memory_limiter.default.input]
  }
}

// ============================================================
// otelcol.processor.memory_limiter — CRITICAL
// Prevents Alloy from running out of memory during
// traffic spikes or when upstream is slow
// Must be FIRST processor in every pipeline
// ============================================================
otelcol.processor.memory_limiter "default" {
  // Start dropping data when memory exceeds this
  limit_mib = 800

  // Start GC aggressively at this threshold
  spike_limit_mib = 200

  // Check memory every 1 second
  check_interval = "1s"

  output {
    metrics = [otelcol.processor.batch.default.input]
    logs    = [otelcol.processor.batch.default.input]
    traces  = [otelcol.processor.tail_sampling.intelligent.input]
  }
}
```

---

<a name="complete"></a>
## 7. 📄 Complete Production Configuration

```river
// ============================================================
// config.alloy — Complete Production Configuration
// Designed to prevent all common Grafana Cloud limit issues
//
// Environment variables required:
//   GRAFANA_CLOUD_API_KEY
//   PROMETHEUS_URL
//   PROMETHEUS_USERNAME
//   LOKI_URL
//   LOKI_USERNAME
//   TEMPO_URL
//   TEMPO_USERNAME
//   CLUSTER_NAME
//   ENVIRONMENT
//   TEAM_NAME
// ============================================================

// ============================================================
// GLOBAL SETTINGS
// ============================================================
logging {
  level  = "warn"
  format = "json"
}

tracing {
  sampling_fraction = 0.0
}

// ============================================================
// KUBERNETES SERVICE DISCOVERY
// ============================================================
discovery.kubernetes "pods" {
  role = "pod"
}

discovery.kubernetes "services" {
  role = "service"
}

discovery.kubernetes "nodes" {
  role = "node"
}

// ============================================================
// METRICS PIPELINE
// ============================================================

// --- Service Discovery Relabelling ---
discovery.relabel "metrics_pods" {
  targets = discovery.kubernetes.pods.targets

  // Only scrape annotated pods
  rule {
    source_labels = [
      "__meta_kubernetes_pod_annotation_prometheus_io_scrape"
    ]
    regex  = "true"
    action = "keep"
  }

  // Drop system namespaces
  rule {
    source_labels = ["__meta_kubernetes_namespace"]
    regex         = "kube-system|kube-public|monitoring|flux-system"
    action        = "drop"
  }

  // Custom port annotation
  rule {
    source_labels = [
      "__meta_kubernetes_pod_annotation_prometheus_io_port"
    ]
    regex        = "(.+)"
    replacement  = "$1"
    target_label = "__metrics_port__"
  }

  // Namespace label
  rule {
    source_labels = ["__meta_kubernetes_namespace"]
    target_label  = "namespace"
  }

  // Pod label
  rule {
    source_labels = ["__meta_kubernetes_pod_name"]
    target_label  = "pod"
  }

  // App label
  rule {
    source_labels = [
      "__meta_kubernetes_pod_label_app_kubernetes_io_name"
    ]
    target_label = "app"
  }

  // Drop high-cardinality k8s labels
  rule {
    action = "labeldrop"
    regex  = "pod_template_hash|controller_revision_hash|statefulset_kubernetes_io_pod_name"
  }
}

// --- Scrape ---
prometheus.scrape "pods" {
  targets         = discovery.relabel.metrics_pods.output
  forward_to      = [prometheus.relabel.drop_unnecessary.receiver]
  scrape_interval = "60s"
  scrape_timeout  = "15s"
  sample_limit    = 2000
  label_limit     = 30
  body_size_limit = "10MB"
}

// --- Drop unnecessary metrics ---
prometheus.relabel "drop_unnecessary" {

  // Drop Go runtime metrics
  rule {
    source_labels = ["__name__"]
    regex         = "go_.*|process_.*|promhttp_.*|scrape_.*"
    action        = "drop"
  }

  // Drop high-cardinality histogram buckets
  rule {
    source_labels = ["__name__"]
    regex         = "apiserver_request_duration_seconds_bucket|etcd_request_duration_seconds_bucket"
    action        = "drop"
  }

  // Drop unused kubelet metrics
  rule {
    source_labels = ["__name__"]
    regex         = "kubelet_volume_stats_.*"
    action        = "drop"
  }

  // Drop high-cardinality labels
  rule {
    action = "labeldrop"
    regex  = "pod_template_hash|controller_revision_hash|.*_uid|helm_sh_chart"
  }

  forward_to = [prometheus.remote_write.grafana_cloud.receiver]
}

// --- Remote Write ---
prometheus.remote_write "grafana_cloud" {
  endpoint {
    url = env("PROMETHEUS_URL")

    basic_auth {
      username = env("PROMETHEUS_USERNAME")
      password = env("GRAFANA_CLOUD_API_KEY")
    }

    tls_config {
      insecure_skip_verify = false
    }

    queue_config {
      min_shards           = 1
      max_shards           = 10
      capacity             = 5000
      max_samples_per_send = 500
      batch_send_deadline  = "10s"
      min_backoff          = "1s"
      max_backoff          = "256s"
      retry_on_http_429    = true
    }

    metadata_config {
      send          = true
      send_interval = "5m"
    }
  }

  wal {
    truncate_frequency = "2h"
    max_keepalive_time = "8h"
    min_keepalive_time = "1h"
  }

  external_labels = {
    cluster     = env("CLUSTER_NAME"),
    environment = env("ENVIRONMENT"),
    team        = env("TEAM_NAME"),
  }
}

// ============================================================
// LOGS PIPELINE
// ============================================================

// --- Log Source Discovery ---
discovery.relabel "log_pods" {
  targets = discovery.kubernetes.pods.targets

  // Only collect opted-in pods
  rule {
    source_labels = [
      "__meta_kubernetes_pod_annotation_logs_grafana_com_scrape"
    ]
    regex  = "true"
    action = "keep"
  }

  // Drop system namespaces
  rule {
    source_labels = ["__meta_kubernetes_namespace"]
    regex         = "kube-system|kube-public|monitoring|flux-system"
    action        = "drop"
  }

  // Stream labels — keep minimal
  rule {
    source_labels = ["__meta_kubernetes_namespace"]
    target_label  = "namespace"
  }

  rule {
    source_labels = [
      "__meta_kubernetes_pod_label_app_kubernetes_io_name"
    ]
    target_label = "app"
  }

  rule {
    source_labels = ["__meta_kubernetes_pod_name"]
    target_label  = "pod"
  }

  // Drop labels that increase stream count
  rule {
    action = "labeldrop"
    regex  = "filename|stream|pod_template_hash"
  }

  // Log path
  rule {
    source_labels = [
      "__meta_kubernetes_pod_uid",
      "__meta_kubernetes_pod_container_name"
    ]
    separator    = "/"
    target_label = "__path__"
    replacement  = "/var/log/pods/*$1/*.log"
  }
}

// --- Log Source ---
loki.source.kubernetes "pods" {
  targets    = discovery.relabel.log_pods.output
  forward_to = [loki.process.reduce_volume.receiver]
}

// --- Log Processing ---
loki.process "reduce_volume" {

  // Drop debug and trace logs in production
  stage.match {
    selector = "{environment=\"production\"}"

    stage.json {
      expressions = { level = "level" }
    }

    stage.drop {
      source    = "level"
      value     = "debug"
      drop_counter_reason = "debug_dropped"
    }

    stage.drop {
      source    = "level"
      value     = "trace"
      drop_counter_reason = "trace_dropped"
    }
  }

  // Drop health check logs
  stage.drop {
    expression  = "GET /(health|ready|livez|metrics).*200"
    drop_counter_reason = "healthcheck_dropped"
  }

  // Rate limit per stream
  stage.limit {
    rate          = 10000
    burst         = 20000
    by_label_name = "pod"
    drop          = true
  }

  // Truncate long lines
  stage.replace {
    expression = `(.{8192}).+`
    replace    = "$1 [TRUNCATED]"
  }

  // Drop high-cardinality stream labels
  stage.label_drop {
    values = ["filename", "stream"]
  }

  forward_to = [loki.write.grafana_cloud.receiver]
}

// --- Loki Write ---
loki.write "grafana_cloud" {
  endpoint {
    url = env("LOKI_URL")

    basic_auth {
      username = env("LOKI_USERNAME")
      password = env("GRAFANA_CLOUD_API_KEY")
    }

    tls_config {
      insecure_skip_verify = false
    }

    batch_size  = 1048576
    batch_wait  = "1s"
    min_backoff = "500ms"
    max_backoff = "5m"
    max_retries = 10
  }

  external_labels = {
    cluster     = env("CLUSTER_NAME"),
    environment = env("ENVIRONMENT"),
  }
}

// ============================================================
// TRACES PIPELINE
// ============================================================

// --- OTLP Receiver ---
otelcol.receiver.otlp "default" {
  grpc {
    endpoint             = "0.0.0.0:4317"
    max_recv_msg_size_mib = 4
  }

  http {
    endpoint              = "0.0.0.0:4318"
    max_request_body_size = "4194304"
  }

  output {
    traces = [otelcol.processor.memory_limiter.default.input]
  }
}

// --- Memory Limiter (MUST BE FIRST) ---
otelcol.processor.memory_limiter "default" {
  limit_mib       = 800
  spike_limit_mib = 200
  check_interval  = "1s"

  output {
    traces = [otelcol.processor.filter.drop_noise.input]
  }
}

// --- Drop noisy traces ---
otelcol.processor.filter "drop_noise" {
  error_mode = "ignore"

  traces {
    span = [
      "attributes[\"http.target\"] == \"/health\"",
      "attributes[\"http.target\"] == \"/ready\"",
      "attributes[\"http.target\"] == \"/metrics\"",
    ]
  }

  output {
    traces = [otelcol.processor.tail_sampling.default.input]
  }
}

// --- Tail-based Sampling ---
otelcol.processor.tail_sampling "default" {
  decision_wait = "10s"
  num_traces    = 50000

  // Always keep errors
  policy {
    name = "errors"
    type = "status_code"
    status_code { status_codes = ["ERROR"] }
  }

  // Always keep slow traces
  policy {
    name = "slow-traces"
    type = "latency"
    latency { threshold_ms = 1000 }
  }

  // Sample 5% of everything else
  policy {
    name = "sample-rest"
    type = "probabilistic"
    probabilistic { sampling_percentage = 5 }
  }

  output {
    traces = [otelcol.processor.batch.default.input]
  }
}

// --- Batch ---
otelcol.processor.batch "default" {
  timeout              = "5s"
  send_batch_size      = 1000
  send_batch_max_size  = 2000

  output {
    traces = [otelcol.exporter.otlphttp.grafana_cloud.input]
  }
}

// --- Trace Exporter ---
otelcol.exporter.otlphttp "grafana_cloud" {
  client {
    endpoint = env("TEMPO_URL")
    auth     = otelcol.auth.basic.grafana_cloud.handler

    tls {
      insecure             = false
      insecure_skip_verify = false
    }

    timeout = "30s"

    retry_on_failure {
      enabled          = true
      initial_interval = "1s"
      max_interval     = "30s"
      max_elapsed_time = "5m"
    }

    sending_queue {
      enabled       = true
      num_consumers = 4
      queue_size    = 1000
    }
  }
}

otelcol.auth.basic "grafana_cloud" {
  username = env("TEMPO_USERNAME")
  password = env("GRAFANA_CLOUD_API_KEY")
}
```

---

<a name="monitoring"></a>
## 8. 📊 Monitoring Your Usage

### 8.1 — Alloy Self-Metrics for Limit Monitoring

```promql
-- =============================================
-- METRICS — Monitor remote write health
-- =============================================

-- Samples dropped due to queue full
rate(prometheus_remote_storage_samples_dropped_total[$__rate_interval])

-- Failed remote write samples (429s from Mimir)
rate(prometheus_remote_storage_failed_samples_total[$__rate_interval])

-- Remote write queue lag (how far behind are we?)
prometheus_remote_storage_samples_pending

-- WAL size (should not grow unbounded)
prometheus_tsdb_wal_storage_size_bytes

-- Samples dropped due to sample_limit
rate(prometheus_target_scrapes_exceeded_sample_limit_total[$__rate_interval])


-- =============================================
-- LOGS — Monitor Loki pipeline health
-- =============================================

-- Log lines dropped by rate limiter
rate(loki_process_dropped_lines_total[$__rate_interval])

-- Log delivery errors
rate(loki_write_sent_entries_total[$__rate_interval])

-- Log entries dropped (any reason)
sum by (reason) (
  rate(loki_process_dropped_lines_total[$__rate_interval])
)


-- =============================================
-- TRACES — Monitor sampling and export
-- =============================================

-- Spans received vs spans exported (sampling ratio)
rate(otelcol_receiver_accepted_spans_total[$__rate_interval])
/
rate(otelcol_exporter_sent_spans_total[$__rate_interval])

-- Export failures
rate(otelcol_exporter_send_failed_spans_total[$__rate_interval])

-- Memory limiter drops
rate(otelcol_processor_dropped_spans_total{
  processor="memory_limiter"
}[$__rate_interval])


-- =============================================
-- GRAFANA CLOUD — Usage against limits
-- Query these on your Grafana Cloud stack
-- =============================================

-- Active series usage vs limit
grafanacloud_instance_active_series

-- Ingestion rate vs limit
rate(grafanacloud_instance_samples_rate[$__rate_interval])

-- Log ingestion rate
grafanacloud_instance_logs_bytes_rate
```

---

### 8.2 — Recommended Alerts for Limit Monitoring

```yaml
# prometheusrule-alloy-limits.yaml
groups:
  - name: alloy-limit-prevention
    rules:

      - alert: AlloyRemoteWriteDroppingSamples
        expr: |
          rate(prometheus_remote_storage_samples_dropped_total[5m]) > 0
        for: 5m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Alloy is dropping metric samples"
          description: >
            Alloy remote write queue is dropping samples.
            This means metrics are being permanently lost.
            Check Mimir rate limits and queue configuration.

      - alert: AlloyRemoteWrite429Errors
        expr: |
          rate(prometheus_remote_storage_failed_samples_total[5m]) > 0
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "Alloy receiving 429 rate limit responses from Mimir"

      - alert: AlloyLokiDroppingLogs
        expr: |
          sum(rate(loki_process_dropped_lines_total[5m])) > 100
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "Alloy dropping more than 100 log lines per second"

      - alert: AlloyMemoryLimiterDropping
        expr: |
          rate(otelcol_processor_dropped_spans_total{
            processor="memory_limiter"
          }[5m]) > 0
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "Alloy memory limiter is dropping spans"
          description: >
            Alloy is under memory pressure.
            Consider increasing memory limits or
            reducing trace ingestion volume.

      - alert: AlloyWALGrowing
        expr: |
          prometheus_tsdb_wal_storage_size_bytes > 1073741824
        for: 15m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "Alloy WAL size exceeds 1GB"
          description: >
            The Alloy WAL is growing, indicating remote write
            is falling behind. Check Mimir connectivity
            and ingestion rate limits.
```

---

<a name="bestpractices"></a>
## 9. ✅ Best Practices Summary

```
Metrics
✅ Always configure retry_on_http_429 = true
✅ Always configure WAL to prevent data loss
✅ Set sample_limit on every scrape job
✅ Drop metrics you do not query (audit dashboards/alerts)
✅ Drop high-cardinality labels before remote write
✅ Use 60s scrape interval for most metrics (not 15s or 30s)
✅ Set max_shards conservatively (start at 10)
✅ Use metadata send_interval of 5m not 1m

Logs
✅ Always configure a rate limiter per stream
✅ Drop debug and trace logs in production
✅ Drop health check endpoint logs
✅ Keep Loki stream labels minimal (3-5 max)
✅ Truncate log lines longer than 8KB
✅ Use sampling for verbose services
✅ Deduplicate repeated log lines

Traces
✅ NEVER send 100% of traces to Tempo
✅ Use tail-based sampling to keep errors and slow traces
✅ Always filter health check spans
✅ Set memory_limiter as the FIRST processor
✅ Configure retry and sending_queue on exporter
✅ Remove sensitive attributes (auth headers etc.)

General
✅ Monitor Alloy self-metrics for drops
✅ Alert on remote write failures immediately
✅ Review Grafana Cloud usage dashboard weekly
✅ Configure WAL for all pipelines
✅ Test backpressure handling before production
✅ Set resource limits on Alloy pod to prevent OOM
```

---

## 📚 Reference Resources

| Resource | Location |
|----------|----------|
| Grafana Alloy Components | `grafana.com/docs/alloy/latest/reference/components` |
| Grafana Cloud Limits | `grafana.com/docs/grafana-cloud/account-management/limits` |
| Mimir Limits | `grafana.com/docs/mimir/latest/configure/about-grafana-mimir-limits` |
| Loki Limits | `grafana.com/docs/loki/latest/configure/limits` |
| Tempo Limits | `grafana.com/docs/tempo/latest/configuration/limits` |
| Alloy WAL Configuration | `grafana.com/docs/alloy/latest/reference/components/prometheus.remote_write` |
| OTel Tail Sampling | `grafana.com/docs/alloy/latest/reference/components/otelcol.processor.tail_sampling` |
| OTel Memory Limiter | `grafana.com/docs/alloy/latest/reference/components/otelcol.processor.memory_limiter` |

---

*Always verify your specific plan limits at your Grafana Cloud stack administration page before tuning these values.*
