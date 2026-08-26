---
title: "Spring Boot Monitoring Dashboard: From Zero to Production in 20 Minutes"
date: 2026-08-24
categories: [Spring Boot, DevOps]
tags: [spring-boot, monitoring, prometheus, grafana, observability]
description: "A speed-run guide to setting up complete observability with Prometheus, Grafana, Loki, and Tempo — with pre-built dashboards you can copy-paste"
mermaid: true
---
I've seen too many teams deploy Spring Boot services to production with zero observability. No metrics, no dashboards, no alerting. Then when something breaks at 2 AM, it's log-file archaeology with `grep` and prayers.

Here's the thing: setting up proper monitoring for Spring Boot doesn't take days. It takes 20 minutes if you know what you're doing. After doing this for dozens of services across banking and e-commerce platforms in Singapore, I've distilled it to a repeatable recipe. Metrics, dashboards, alerts, logs, and traces — all connected, all copy-paste ready.

Let's speed-run this.

## What We're Building

By the end of this post, you'll have:

- **Prometheus** scraping metrics from your Spring Boot app
- **Grafana** with pre-built dashboards showing JVM stats, HTTP metrics, and business KPIs
- **Alert rules** that fire when response times spike or error rates climb
- **Loki** aggregating structured logs from your application
- **Tempo** capturing distributed traces with automatic span correlation
- All of it wired together so you can jump from a metric spike to related logs to the exact trace

## Step 1: Spring Boot Dependencies (2 Minutes)

Add these to your `pom.xml`:

```xml
<dependencies>
    <!-- Actuator exposes metrics endpoint -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    
    <!-- Prometheus metrics format -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
    
    <!-- Distributed tracing with OTLP export -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-otel</artifactId>
    </dependency>
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-exporter-otlp</artifactId>
    </dependency>
    
    <!-- Structured logging with Loki appender -->
    <dependency>
        <groupId>com.github.loki4j</groupId>
        <artifactId>loki-logback-appender</artifactId>
        <version>1.5.1</version>
    </dependency>
</dependencies>
```

## Step 2: Application Configuration (3 Minutes)

`application.yml`:

```yaml
spring:
  application:
    name: order-service

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics,loggers
  endpoint:
    health:
      show-details: when_authorized
      probes:
        enabled: true
  metrics:
    tags:
      application: ${spring.application.name}
      environment: ${ENVIRONMENT:local}
    distribution:
      percentiles-histogram:
        http.server.requests: true
      percentiles:
        http.server.requests: 0.5, 0.9, 0.95, 0.99
      slo:
        http.server.requests: 50ms, 100ms, 200ms, 500ms
  tracing:
    sampling:
      probability: 1.0
  otlp:
    tracing:
      endpoint: http://localhost:4318/v1/traces

logging:
  pattern:
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
```

Key configuration decisions:

- **Percentile histograms** — Enable these for proper p50/p90/p99 calculations in Prometheus
- **SLO buckets** — Define your latency SLOs upfront (50ms, 100ms, 200ms, 500ms)
- **Trace correlation in logs** — The logging pattern includes traceId and spanId so you can jump from log line to full trace
- **100% sampling** — For local/staging. In production, use 10-20% or adaptive sampling

## Step 3: Custom Business Metrics (3 Minutes)

Don't just rely on auto-generated HTTP metrics. Add business-meaningful metrics:

```java
@Component
public class OrderMetrics {

    private final Counter ordersCreated;
    private final Counter ordersFailed;
    private final Timer orderProcessingTime;
    private final AtomicInteger activeOrders;
    private final DistributionSummary orderValue;

    public OrderMetrics(MeterRegistry registry) {
        this.ordersCreated = Counter.builder("orders.created.total")
            .description("Total orders created")
            .tag("service", "order-service")
            .register(registry);

        this.ordersFailed = Counter.builder("orders.failed.total")
            .description("Total orders that failed processing")
            .tag("service", "order-service")
            .register(registry);

        this.orderProcessingTime = Timer.builder("orders.processing.duration")
            .description("Time taken to process an order")
            .publishPercentiles(0.5, 0.9, 0.95, 0.99)
            .register(registry);

        this.activeOrders = registry.gauge("orders.active.count", 
            new AtomicInteger(0));

        this.orderValue = DistributionSummary.builder("orders.value")
            .description("Distribution of order values")
            .baseUnit("dollars")
            .publishPercentiles(0.5, 0.75, 0.9)
            .register(registry);
    }

    public void recordOrderCreated() {
        ordersCreated.increment();
    }

    public void recordOrderFailed() {
        ordersFailed.increment();
    }

    public Timer.Sample startProcessing() {
        activeOrders.incrementAndGet();
        return Timer.start();
    }

    public void stopProcessing(Timer.Sample sample) {
        sample.stop(orderProcessingTime);
        activeOrders.decrementAndGet();
    }

    public void recordOrderValue(double amount) {
        orderValue.record(amount);
    }
}
```

Usage in your service:

```java
@Service
public class OrderService {

    private final OrderRepository repository;
    private final OrderMetrics metrics;

    public OrderService(OrderRepository repository, OrderMetrics metrics) {
        this.repository = repository;
        this.metrics = metrics;
    }

    public Order createOrder(CreateOrderRequest request) {
        Timer.Sample sample = metrics.startProcessing();
        try {
            Order order = buildOrder(request);
            Order saved = repository.save(order);
            
            metrics.recordOrderCreated();
            metrics.recordOrderValue(saved.getTotalAmount());
            
            return saved;
        } catch (Exception e) {
            metrics.recordOrderFailed();
            throw e;
        } finally {
            metrics.stopProcessing(sample);
        }
    }
}
```

## Step 4: Structured Logging with Loki (3 Minutes)

Create `logback-spring.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <springProperty scope="context" name="appName" source="spring.application.name"/>

    <!-- Console output for local development -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} %highlight(%-5level) [%thread] %cyan(%logger{36}) - %msg %X{traceId:-} %n</pattern>
        </encoder>
    </appender>

    <!-- Loki appender for log aggregation -->
    <appender name="LOKI" class="com.github.loki4j.logback.Loki4jAppender">
        <http>
            <url>http://localhost:3100/loki/api/v1/push</url>
        </http>
        <format>
            <label>
                <pattern>application=${appName},host=${HOSTNAME},level=%level</pattern>
            </label>
            <message>
                <pattern>
                    {"timestamp":"%d{yyyy-MM-dd'T'HH:mm:ss.SSS}","level":"%level","logger":"%logger","thread":"%thread","traceId":"%X{traceId:-none}","spanId":"%X{spanId:-none}","message":"%msg"}
                </pattern>
            </message>
            <sortByTime>true</sortByTime>
        </format>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="LOKI"/>
    </root>

    <!-- Reduce noise from libraries -->
    <logger name="org.apache.kafka" level="WARN"/>
    <logger name="org.hibernate.SQL" level="DEBUG"/>
</configuration>
```

The JSON-structured log messages mean you can query by any field in Grafana:

```
{application="order-service"} |= "traceId" | json | level="ERROR"
```

## Step 5: Prometheus Configuration (2 Minutes)

`docker/prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert-rules.yml"

scrape_configs:
  - job_name: 'spring-boot-apps'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['host.docker.internal:8080']
        labels:
          application: 'order-service'
          environment: 'local'

  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

## Step 6: Alert Rules (3 Minutes)

`docker/prometheus/alert-rules.yml`:

```yaml
groups:
  - name: spring-boot-alerts
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
          /
          sum(rate(http_server_requests_seconds_count[5m]))
          > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected (> 5%)"
          description: "Error rate is {{ $value | humanizePercentage }} over the last 5 minutes"

      # Slow response times
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95, 
            sum(rate(http_server_requests_seconds_bucket[5m])) by (le, uri)
          ) > 0.5
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "P95 latency above 500ms"
          description: "P95 latency for {{ $labels.uri }} is {{ $value }}s"

      # JVM memory pressure
      - alert: HighMemoryUsage
        expr: |
          jvm_memory_used_bytes{area="heap"}
          /
          jvm_memory_max_bytes{area="heap"}
          > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "JVM heap usage above 85%"
          description: "Heap usage is {{ $value | humanizePercentage }}"

      # Application down
      - alert: ApplicationDown
        expr: up{job="spring-boot-apps"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Application is down"
          description: "{{ $labels.application }} has been down for more than 1 minute"

      # High order failure rate (business metric)
      - alert: HighOrderFailureRate
        expr: |
          rate(orders_failed_total[5m])
          /
          (rate(orders_created_total[5m]) + rate(orders_failed_total[5m]))
          > 0.1
        for: 3m
        labels:
          severity: critical
        annotations:
          summary: "Order failure rate above 10%"
          description: "{{ $value | humanizePercentage }} of orders are failing"
```

## Step 7: Grafana Dashboard (4 Minutes)

Here's a pre-built Grafana dashboard JSON. Save this as `docker/grafana/dashboards/spring-boot-overview.json`:

```json
{
  "dashboard": {
    "title": "Spring Boot Application Overview",
    "uid": "spring-boot-overview",
    "timezone": "browser",
    "refresh": "10s",
    "panels": [
      {
        "title": "Request Rate",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 },
        "targets": [
          {
            "expr": "sum(rate(http_server_requests_seconds_count{application=\"$application\"}[5m])) by (uri, method)",
            "legendFormat": "{{method}} {{uri}}"
          }
        ]
      },
      {
        "title": "Response Time (P95)",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 12, "x": 12, "y": 0 },
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket{application=\"$application\"}[5m])) by (le, uri))",
            "legendFormat": "{{uri}}"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "stat",
        "gridPos": { "h": 4, "w": 6, "x": 0, "y": 8 },
        "targets": [
          {
            "expr": "sum(rate(http_server_requests_seconds_count{application=\"$application\", status=~\"5..\"}[5m])) / sum(rate(http_server_requests_seconds_count{application=\"$application\"}[5m]))"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percentunit",
            "thresholds": {
              "steps": [
                { "value": 0, "color": "green" },
                { "value": 0.01, "color": "yellow" },
                { "value": 0.05, "color": "red" }
              ]
            }
          }
        }
      },
      {
        "title": "JVM Heap Usage",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 12 },
        "targets": [
          {
            "expr": "jvm_memory_used_bytes{application=\"$application\", area=\"heap\"}",
            "legendFormat": "Used"
          },
          {
            "expr": "jvm_memory_max_bytes{application=\"$application\", area=\"heap\"}",
            "legendFormat": "Max"
          },
          {
            "expr": "jvm_memory_committed_bytes{application=\"$application\", area=\"heap\"}",
            "legendFormat": "Committed"
          }
        ],
        "fieldConfig": {
          "defaults": { "unit": "bytes" }
        }
      },
      {
        "title": "Active Threads",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 12, "x": 12, "y": 12 },
        "targets": [
          {
            "expr": "jvm_threads_live_threads{application=\"$application\"}",
            "legendFormat": "Live"
          },
          {
            "expr": "jvm_threads_peak_threads{application=\"$application\"}",
            "legendFormat": "Peak"
          }
        ]
      },
      {
        "title": "Orders Created vs Failed",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 20 },
        "targets": [
          {
            "expr": "rate(orders_created_total{application=\"$application\"}[5m])",
            "legendFormat": "Created"
          },
          {
            "expr": "rate(orders_failed_total{application=\"$application\"}[5m])",
            "legendFormat": "Failed"
          }
        ]
      },
      {
        "title": "Database Connection Pool",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 12, "x": 12, "y": 20 },
        "targets": [
          {
            "expr": "hikaricp_connections_active{application=\"$application\"}",
            "legendFormat": "Active"
          },
          {
            "expr": "hikaricp_connections_idle{application=\"$application\"}",
            "legendFormat": "Idle"
          },
          {
            "expr": "hikaricp_connections_max{application=\"$application\"}",
            "legendFormat": "Max"
          }
        ]
      }
    ],
    "templating": {
      "list": [
        {
          "name": "application",
          "type": "query",
          "query": "label_values(jvm_info, application)",
          "current": { "text": "order-service", "value": "order-service" }
        }
      ]
    }
  }
}
```

Grafana provisioning (`docker/grafana/provisioning/datasources/datasources.yml`):

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: true
    jsonData:
      derivedFields:
        - datasourceUid: tempo
          matcherRegex: '"traceId":"(\w+)"'
          name: TraceID
          url: '$${__value.raw}'

  - name: Tempo
    type: tempo
    access: proxy
    url: http://tempo:3200
    uid: tempo
    editable: true
    jsonData:
      tracesToLogs:
        datasourceUid: loki
        filterByTraceID: true
        filterBySpanID: true
```

The magic here is the `derivedFields` configuration in Loki — it automatically creates clickable links from traceId values in your logs to the full trace in Tempo. And Tempo's `tracesToLogs` does the reverse: from a trace span, you can jump directly to the related logs.

## Step 8: Verify Everything Works (2 Minutes)

Start the infrastructure:

```bash
docker compose up -d prometheus grafana loki tempo
```

Start your Spring Boot app:

```bash
./mvnw spring-boot:run
```

Verify metrics are exposed:

```bash
curl http://localhost:8080/actuator/prometheus | head -20
```

Check Prometheus is scraping (open `http://localhost:9090/targets`):

- Status should show "UP" for your application target

Open Grafana at `http://localhost:3000` (admin/admin):

- The Spring Boot Overview dashboard should already be available via provisioning

## The Complete Observability Loop

Here's how all the pieces connect in an incident scenario:

1. **Alert fires** — Grafana shows "High Error Rate" alert from Prometheus
2. **Check the dashboard** — You see the error spike on the timeseries panel, correlated with a latency increase
3. **Drill into logs** — Click the time range, switch to Loki datasource, filter by `level="ERROR"`
4. **Find the trace** — Click the traceId link in the log line, jumps to Tempo
5. **See the full picture** — The trace shows which downstream service is slow or failing

This loop — metric to log to trace — is what separates "we have monitoring" from "we have observability."

## Production Checklist

Before deploying this to production, adjust these settings:

- **Sampling rate** — Reduce from 1.0 to 0.1 or 0.2 (10-20% of requests traced)
- **Retention** — Set Prometheus retention to 15-30 days, Loki to 7-14 days
- **Storage** — Use remote storage (S3/GCS) for Loki and Tempo in production
- **Authentication** — Enable auth on Grafana, use service accounts for Prometheus
- **Resource limits** — Prometheus needs 2-4GB RAM for 30-day retention with 10+ services
- **Alert routing** — Connect Alertmanager to Slack/PagerDuty for real notifications

## Common Pitfalls

- **Cardinality explosion** — Don't tag metrics with user IDs, request IDs, or any high-cardinality values. Your Prometheus will OOM.
- **Missing histogram buckets** — If your SLO is 200ms but all your buckets are above 1s, you can't measure your SLO accurately.
- **Log volume** — Structured JSON logs are larger. Set Loki retention carefully or you'll fill your disk.
- **Trace context propagation** — If you use RestTemplate or WebClient, trace context propagates automatically. If you use raw HttpClient, you need to propagate headers manually.

## Final Thoughts

20 minutes. That's all it takes to go from zero observability to a production-grade monitoring setup. The initial investment is minimal, but the payoff the first time something breaks at 2 AM is immeasurable.

The most important piece isn't any single tool — it's the correlation between metrics, logs, and traces. When you can jump from "error rate spiked" to "here's the exact request that failed and why" in under 30 seconds, you've achieved real observability.
