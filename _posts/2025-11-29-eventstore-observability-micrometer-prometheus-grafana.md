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
| `sliceworkz.eventstore.bookmark.list` | Number of `getBookmarks()` calls | count |

### Timers

| Metric Name | Description | Unit |
|-------------|-------------|------|
| `sliceworkz.eventstore.append.duration` | Time taken to append events (including optimistic locking check) | milliseconds |
| `sliceworkz.eventstore.query.duration` | Time taken to execute queries | milliseconds |

### Gauges

| Metric Name | Description | Unit |
|-------------|-------------|------|
| `sliceworkz.eventstore.append.position` | Highest event position appended, per tag set | position |
| `sliceworkz.eventstore.notifications.up` | Whether a LISTEN/NOTIFY channel is established (1) or not (0) | boolean |

**`append.position`** reads `NaN` until something is appended. Its state is held per tag set **on the store**, not per stream — a gauge cannot be re-registered, and Micrometer holds gauge state weakly, so a per-stream holder would leave the series permanently `NaN` as soon as the first stream for that tag set was collected. That matters in practice, because obtaining a stream per operation is the recommended usage.

**`notifications.up`** is specific to the PostgreSQL backend and tagged `storage` and `channel` (`event_appended` / `bookmark_placed`) rather than by stream. It is registered by the storage constructor, so the series exists reading 0 from the moment the storage does — a gauge that only appears once notifications work would be no use for alerting on notifications *not* working.

This is the metric that makes the store's one silent failure mode visible. When notifications stop, the store is **degraded, not broken**: queries, appends and bookmarks all keep working, but nothing wakes a subscriber, so projections only advance when something runs them explicitly. Nothing throws and nothing else in these metrics changes.

```promql
# alert: notifications down for more than a minute
min_over_time(sliceworkz_eventstore_notifications_up[1m]) == 0
```

The same state is available to a health endpoint via `PostgresEventStorageImpl.isNotificationsAvailable()`, at the cost of a downcast from `EventStorage`. It returns to 1 on its own once the database is reachable again — the monitors retry with backoff.

> This gauge does **not** reveal a read stall caused by a long-running write transaction elsewhere in the PostgreSQL cluster. During such a stall every call the store makes keeps succeeding, so nothing here moves. Detecting that is done on the database — see [What Can Stall Reads](/posts/eventstore-configuring-postgresql-storage/#what-can-stall-reads-the-visibility-barrier).
{: .prompt-warning }

## Bounding Meter Cardinality: the `purpose` Tag

`context` is a code-level concept, so its cardinality is a property of your application. **`purpose` is not.** It is an optional secondary identifier, and the natural way to use it is per entity — `forContext("customer").withPurpose("123")` — which gives it one value per customer.

**Nothing evicts a meter.** A Micrometer registry keeps every meter it has ever registered, so the cost follows the number of distinct purposes the process has *ever seen*, not how many streams are alive. Dropping the stream handle releases none of it.

Measured per distinct purpose, on an in-memory store with two event types: **15 meters, ~5.5 KB of heap, 18 Prometheus series** and ~2.4 KB of scrape body. At 10.000 purposes that is 150.000 meters, 53 MB and a 23 MB scrape. Nothing fails — the numbers stay correct and the process just gets steadily heavier.

**So the `purpose` tag is capped.** A store tags the first `MeterOptions.maxPurposeTagValues()` distinct purposes it sees (**default 1000**) and reports every purpose after that as `_other`, logging one WARN naming the purpose that tripped the cap. Below the cap nothing changes — that is exactly where a per-purpose breakdown is worth having — and above it the meters stay flat while the events are still counted, pooled under `_other`.

Admission is first-come-first-served and **permanent**: a purpose that got its own tag value keeps it for the life of the store, so a dashboard built on that series does not lose it when traffic widens. The flip side is that *which* purposes get through is arrival order, and not stable across restarts.

### Configuring It

```java
// purpose is an entity id here: never break down by it
EventStoreFactory.get().eventStore(storage, registry, MeterOptions.withoutPurposeBreakdown());

// a broad but genuinely bounded set of purposes
EventStoreFactory.get().eventStore(storage, registry, MeterOptions.withMaxPurposeTagValues(5000));

// same thing through the storage builders
InMemoryEventStorage.newBuilder()
    .meterOptions(MeterOptions.withoutPurposeBreakdown())
    .buildStore();

PostgresEventStorage.newBuilder()
    .meterRegistry(registry)
    .meterOptions(MeterOptions.withMaxPurposeTagValues(5000))
    .buildStore();
```

`MeterOptions.withUnlimitedPurposeTagValues()` removes the bound, which is only safe where purpose is low-cardinality by construction. Existing two-argument factory calls keep working unchanged and get the default cap.

> **A Micrometer `MeterFilter` is not a substitute.** A filter runs at registration, while the store keys its `append.position` gauge state on the tags it *asked* for — so with the meters denied outright, a registry holding zero meters still leaves the store growing by roughly 730 bytes per distinct purpose. The cap is applied where the tag value is chosen, which bounds the meters, the `eventtype` cross product and that internal map in one place.
{: .prompt-info }

**`context` is deliberately not capped.** It names a bounded context and comes from the code, not from the traffic. A store whose *context* is per-entity has the same problem with none of the protection — don't do that.

## Example Configuration: Prometheus

To expose metrics to Prometheus, add the Prometheus Micrometer registry dependency and configure an HTTP endpoint.

### Maven Dependencies

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
    <version>1.17.0</version>
</dependency>
<dependency>
    <groupId>io.javalin</groupId>
    <artifactId>javalin</artifactId>
    <version>7.2.2</version>
</dependency>
```

### Java Configuration with Javalin

```java
import io.javalin.Javalin;
import io.micrometer.prometheusmetrics.PrometheusConfig;
import io.micrometer.prometheusmetrics.PrometheusMeterRegistry;

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

        // Create EventStore with Prometheus metrics.
        // The builder passes the registry on to the storage as well, so HikariCP
        // pool metrics land in the same registry.
        try ( EventStore eventStore = PostgresEventStorage.newBuilder()
                  .meterRegistry(prometheusRegistry)
                  .meterOptions(MeterOptions.withoutPurposeBreakdown())
                  .buildStore() ) {

            // Expose metrics endpoint via Javalin
            Javalin app = Javalin.create().start(8080);

            app.get("/metrics", ctx -> {
                ctx.contentType("text/plain; version=0.0.4");
                ctx.result(prometheusRegistry.scrape());
            });

            // Your application logic here...
        }
    }
}
```

> The registry passed to `.meterRegistry(...)` is also handed to any HikariCP `DataSource` the builder created, so connection pool metrics appear alongside the event store's own.
{: .prompt-tip }

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

**Panel: Notification Health**
```promql
sliceworkz_eventstore_notifications_up
```
Group by `channel` to see the append and bookmark channels separately.

### Key Metrics to Monitor

- **Notifications down**: `notifications_up == 0` means read models have stopped advancing on their own, while everything else keeps working. This is the one failure that is otherwise silent, and the first thing to alert on
- **High optimistic locking conflicts**: May indicate contention on specific aggregates requiring architectural review
- **Slow query durations**: Could signal missing indexes, inefficient queries, or database resource constraints
- **Append rate spikes**: Unusual activity patterns that might indicate bugs or attacks
- **Event position growth**: Helps predict storage requirements and identify most active streams
- **A `purpose="_other"` series appearing**: the cardinality cap has been reached, so per-purpose breakdowns are no longer complete

With Grafana, you can set up alerts on these metrics to proactively detect issues before they impact users.