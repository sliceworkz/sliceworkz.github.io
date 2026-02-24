---
layout: post
toc: true
title: Eventstore Observability
description: Eventstore Observability with Micrometer, Prometheus and Grafana
date: 2025-11-29 08:00:00
categories: [Eventstore Documentation,Eventstore deployment]
tags: [observability,monitoring,micrometer,prometheus,grafana]
---

## EventStore Observability

Understanding how your EventStore deployment performs in production is critical for maintaining a healthy event-sourced system. Observability enables you to:

- **Track operational health**: Monitor append and query rates to detect unusual activity patterns
- **Identify performance bottlenecks**: Measure operation durations to find slow queries or contention
- **Optimize resource usage**: Understand which event streams are most active and resource-intensive
- **Debug production issues**: Correlate metrics with application behavior during incident investigation
- **Capacity planning**: Use historical metrics to predict growth and plan infrastructure scaling

## Micrometer Integration

EventStore uses [Micrometer](https://micrometer.io/) as its metrics collection framework. Micrometer provides a vendor-neutral facade similar to SLF4J for logging, allowing you to emit metrics once and send them to various monitoring backends (Prometheus, Grafana Cloud, Datadog, etc.).

When creating an EventStore instance, provide a `MeterRegistry` to enable metrics collection:

```java
// Option 1: Use the global registry (simplest approach)
EventStorage storage = PostgresEventStorage.newBuilder().build();
EventStore store = EventStoreFactory.get().eventStore(storage);

// Option 2: Provide a custom registry with specific configuration
MeterRegistry registry = new SimpleMeterRegistry();
EventStore store = EventStoreFactory.get().eventStore(storage, registry);
```

### Adding Custom Tags for Drill-Down Analysis

To enable drill-down analysis by deployment context, add common tags to your registry:

```java
MeterRegistry registry = new SimpleMeterRegistry();

// Add tags for deployment context
registry.config().commonTags(
    "instance", "eventstore-01",           // Instance identifier
    "deployment", "production-eu-west",    // Deployment unit/region
    "app.version", "1.2.3",                // Application version
    "environment", "production"             // Environment name
);

EventStore store = EventStoreFactory.get().eventStore(storage, registry);
```

These tags are automatically applied to all metrics, enabling you to:
- Compare performance across different instances
- Identify version-specific issues after deployments
- Separate production from staging metrics
- Analyze regional performance differences

## Available Metrics

EventStore exposes the following metrics through Micrometer. All metrics include these automatic tags:

| Tag | Description | Example Values |
|-----|-------------|----------------|
| `context` | Event stream context | `"customer"`, `"order"`, `""` (empty for any-context) |
| `purpose` | Event stream purpose | `"123"`, `"aggregate-id"`, `""` (empty for any-purpose) |
| `typed` | Whether stream uses typed or raw events | `"true"`, `"false"` |
| `storage` | Storage backend name | `"postgres"`, `"inmemory"` |

Additionally, the `append.event` and `query.event` counters include an `eventtype` tag with the event type name, enabling per-event-type throughput analysis:

| Tag | Description | Example Values |
|-----|-------------|----------------|
| `eventtype` | The event type name (on `append.event` and `query.event` only) | `"CustomerRegistered"`, `"OrderPlaced"` |

### Counters

| Metric Name | Description | Unit |
|-------------|-------------|------|
| `sliceworkz.eventstore.stream.create` | Number of event stream objects created | count |
| `sliceworkz.eventstore.append` | Number of successful append operations | count |
| `sliceworkz.eventstore.append.event` | Total number of events appended (tagged with `eventtype`) | count |
| `sliceworkz.eventstore.append.optimisticlock` | Number of append operations rejected due to optimistic locking conflicts | count |
| `sliceworkz.eventstore.query` | Number of query operations executed | count |
| `sliceworkz.eventstore.query.event` | Total number of events retrieved by queries (tagged with `eventtype`) | count |
| `sliceworkz.eventstore.get.event` | Number of individual event lookups by ID | count |
| `sliceworkz.eventstore.bookmark.place` | Number of bookmark updates | count |
| `sliceworkz.eventstore.bookmark.get` | Number of bookmark retrievals | count |

### Timers

| Metric Name | Description | Unit |
|-------------|-------------|------|
| `sliceworkz.eventstore.append.duration` | Time taken to append events (including optimistic locking check) | milliseconds |
| `sliceworkz.eventstore.query.duration` | Time taken to execute queries | milliseconds |

### Gauges

| Metric Name | Description | Unit |
|-------------|-------------|------|
| `sliceworkz.eventstore.append.position` | Highest event position appended to this stream | position |

## Example Configuration: Prometheus

To expose metrics to Prometheus, add the Prometheus Micrometer registry dependency and configure an HTTP endpoint.

### Maven Dependencies

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
    <version>1.16.0</version>
</dependency>
<dependency>
    <groupId>io.javalin</groupId>
    <artifactId>javalin</artifactId>
    <version>6.4.0</version>
</dependency>
```

### Java Configuration with Javalin

```java
import io.javalin.Javalin;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.prometheus.PrometheusConfig;
import io.micrometer.prometheus.PrometheusMeterRegistry;

public class EventStoreApp {
    public static void main(String[] args) {
        // Create Prometheus registry
        PrometheusMeterRegistry prometheusRegistry =
            new PrometheusMeterRegistry(PrometheusConfig.DEFAULT);

        // Add common tags for drill-down
        prometheusRegistry.config().commonTags(
            "instance", System.getenv("HOSTNAME"),
            "app.version", "1.2.3"
        );

        // Create EventStore with Prometheus metrics
        EventStorage storage = PostgresEventStorage.newBuilder().build();
        EventStore eventStore = EventStoreFactory.get()
            .eventStore(storage, prometheusRegistry);

        // Expose metrics endpoint via Javalin
        Javalin app = Javalin.create().start(8080);

        app.get("/metrics", ctx -> {
            ctx.contentType("text/plain; version=0.0.4");
            ctx.result(prometheusRegistry.scrape());
        });

        // Your application logic here...
    }
}
```

### Prometheus Scrape Configuration

Add this job to your `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'eventstore'
    static_configs:
      - targets: ['localhost:8080']
    metrics_path: '/metrics'
    scrape_interval: 15s
```

## Example Reporting: Grafana

Grafana provides powerful visualization and alerting capabilities for EventStore metrics using Prometheus as a datasource.

### Setting Up Grafana with Prometheus

1. **Add Prometheus datasource** in Grafana:
   - Navigate to Configuration → Data Sources
   - Select "Prometheus"
   - Set URL to your Prometheus instance (e.g., `http://localhost:9090`)
   - Click "Save & Test"

2. **Create EventStore dashboard** with useful panels:

**Panel: Append Rate by Stream Context**
```promql
rate(sliceworkz_eventstore_append_total[5m])
```

**Panel: Query Duration (95th Percentile)**
```promql
histogram_quantile(0.95,
  rate(sliceworkz_eventstore_query_duration_seconds_bucket[5m])
)
```

**Panel: Optimistic Locking Conflict Rate**
```promql
rate(sliceworkz_eventstore_append_optimisticlock_total[5m])
```

**Panel: Events Appended per Second**
```promql
rate(sliceworkz_eventstore_append_event_total[5m])
```

**Panel: Events Appended per Second by Event Type**
```promql
rate(sliceworkz_eventstore_append_event_total[5m])
```
Group by `eventtype` label to see the per-event-type breakdown.

**Panel: Highest Event Position by Stream**
```promql
sliceworkz_eventstore_append_position
```

### Key Metrics to Monitor

- **High optimistic locking conflicts**: May indicate contention on specific aggregates requiring architectural review
- **Slow query durations**: Could signal missing indexes, inefficient queries, or database resource constraints
- **Append rate spikes**: Unusual activity patterns that might indicate bugs or attacks
- **Event position growth**: Helps predict storage requirements and identify most active streams

With Grafana, you can set up alerts on these metrics to proactively detect issues before they impact users.