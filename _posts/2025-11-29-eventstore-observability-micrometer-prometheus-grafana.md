---
layout: post
toc: true
title: Eventstore Observability
description: Observing the Eventstore with the metrics or tracing library of your choice, through the EventStoreObserver SPI, and asking a storage for its health — with Micrometer, Prometheus and Grafana as the worked example
date: 2025-11-29 08:00:00
categories: [Eventstore Documentation,Eventstore deployment]
tags: [observability,monitoring,observer,health,tracing,micrometer,prometheus,grafana]
---

## EventStore Observability

Understanding how your EventStore deployment performs in production is critical for maintaining a healthy event-sourced system. Observability enables you to:

- **Track operational health**: Monitor append and query rates, conflicts and de-duplications
- **Identify performance bottlenecks**: Measure operation durations, and how much of each is spent in the storage
- **Trace a request end to end**: See an append, its key-store lookups and its JDBC calls nested under one span
- **Detect the silent failure**: Know when append notifications have stopped, which nothing else will tell you
- **Capacity planning**: Use historical figures to predict growth and plan infrastructure scaling

## An SPI of Its Own, and No Metrics Library

**The store reports what it does to an `EventStoreObserver`, in its own terms, and names no metrics or tracing library.** An application that wants meters, spans, log lines or an audit trail provides an observer that turns observations into them. `sliceworkz-eventstore-api` therefore carries no Micrometer, no OpenTelemetry — nothing but SLF4J — and you bind the library you already use.

The alternative — a meter facade naming counters, timers and gauges — was rejected because it can express nothing but meters: a tracer needs to know where an operation starts and ends and what it answered, which is exactly what an observation is.

**Observation is opt-in.** A store or storage given no observer uses `EventStoreObserver.NOOP`, which records nothing and allocates nothing.

## Configuring an Observer

The observer travels with the storage, the same way the shredding codec does. Give it to the storage builder, and every store built on that storage reports to it:

```java
EventStoreObserver observer = new MicrometerObserver(registry);   // an example observer, see below

EventStore store = PostgresEventStorage.newBuilder()
    .observer(observer)
    .buildStore();

// the in-memory backends take one the same way
EventStore store = InMemoryEventStorage.newBuilder().observer(observer).buildStore();
```

Or give it to one store built on a storage you hold, which takes precedence over the storage's own:

```java
EventStore store = EventStore.on(storage).observer(observer).build();
```

A `Projector` finds the observer through its source, so a projector is observed exactly when its stream is, with nothing to configure on the projector.

## Operations: a Scope From Start to Close

Every operation the store performs on behalf of a caller is reported through `start(Observation)`, which describes what is about to be done and returns an `Observation.Scope`. The store then reports exactly one of `completed(outcome)` or `failed(throwable)`, and closes the scope in a `finally` — **all synchronously, on the caller's thread**. So an observer may make a span current between `start` and `close`, and the JDBC calls, key-store lookups and a projector batch's page query nest beneath it with nothing propagated.

| Observation | Reported for | Completes with |
|---|---|---|
| `Append` | an append: the types submitted, whether it is conditional, how many idempotency keys | `Appended`, `Conflicted` or `Duplicated` |
| `Query` | a query or a page, a projector's pages among them | `Read`: stored events read per stored type, events returned |
| `GetEvent` | `getEventById` | `Found` |
| `Head` | `head()` | `HeadRead` |
| `PlaceBookmark` / `GetBookmark` / `ListBookmarks` | the bookmark operations — `PlaceBookmark` carries the last event handled — empty for a read position placed alone — and, when the placement records one, the read position; `GetBookmark` covers `getBookmark` and `findBookmark` alike | `Done` / `Found` / `Counted` |
| `ProjectorBatch` | one projector batch; phase `INIT` for the savepoint read, `BATCH` for every page | `Projected`: stored events read, events handled, the last reference, whether it bookmarked |
| `Erase` | an erasure, with its reason | `Erased`: keys shredded, categories erased |

The records carry the caller's own arguments, and the stream as a `StreamInfo` — storage name, `EventStreamId`, and whether the stream is typed. `Observation` and `Outcome` are sealed: an observer switching over them with a `default` branch keeps compiling when an operation is added.

**The outcomes of stream operations carry `storageTime`**, the share of the operation spent inside the `EventStorage`. The scope spans the whole call, so the rest is serialization, upcasting and unsealing — which on an ordinary page is most of the wait.

### A Completion Is an Answer, Not Only a Success

An admitted append answers one of three things:

- **`Appended`** — the batch was stored.
- **`Conflicted`** — a DCB conflict: nothing stored, and the caller receives the `OptimisticLockingException` to re-decide on. It is the store working as intended, so it is an answer rather than a failure. The lock check runs first, so a stale retry answers `Conflicted` too.
- **`Duplicated`** — every idempotency key in the batch was stored before: a retry, swallowed whole. A batch is stored whole or not at all, so there is no partly de-duplicated answer.

`failed` is for an operation that could not answer: a storage error, a poison event, a key store that is down, an `IdempotencyKeyConflictException`, a projection that threw — with the throwable the caller receives. Argument refusals (an append through a wildcard stream, a legacy type in a filter, a repeated key) happen before an observation starts and are not reported.

## Lifecycle and Health

The observer's other methods report what is *held*, so an observer can count it and release what it registered for it. Each has an empty default, so you override what you care about:

| Method | Reported when |
|---|---|
| `storageStarted(storage)` / `storageClosed(storage)` | a backend finished `build()` / was closed — channels are reported down first |
| `subscriptionOpened(stream)` / `subscriptionClosed(stream)` | once per subscription, so opened minus closed is the live count — and a count that only rises is a subscribed stream nobody closes |
| `notificationChannelChanged(storage, channel, listening)` | a LISTEN/NOTIFY channel went up or down (PostgreSQL) |
| `visibilityStallChanged(storage, stalled)` | reads went behind the [visibility barrier](/posts/eventstore-configuring-postgresql-storage/#what-can-stall-reads-the-visibility-barrier), or came out from behind it (PostgreSQL) |
| `streamOpened(stream)` | a stream handle was opened |

There is deliberately no `streamClosed`: a handle used only to query and append holds nothing and is never closed, so a counterpart would climb forever. What a stream holds is a subscription.

### Notification Channel Health

This is the signal to alert on. When append notifications stop, the store is **degraded, not broken**: queries, appends and bookmarks all keep working, but nothing wakes a subscriber, so every subscribed projection quietly stops advancing. Nothing throws, and nothing else changes.

The PostgreSQL backend reports each channel — `NotificationChannel.EVENT_APPENDED` and `BOOKMARK_PLACED` — as **down from its constructor**, so an observer knows the channel from the moment the storage exists; up once its listener is registered; down whenever it loses it (a dropped connection, or one found silently dead by the liveness probe) and up again when it recovers; and down on close. Every transition is reported exactly once. The in-memory backends notify in-process, have no channels, and never call it.

For a health endpoint rather than a metric, [`health()`](#asking-a-storage-for-its-health) reports each channel as a check of its own.

### Visibility Stalls

A channel reported up does not rule out a read stall caused by a long-running write transaction elsewhere in the PostgreSQL cluster: every call the store makes keeps succeeding, and reads simply stop advancing. The PostgreSQL backend holds an append notification back until the events it announces are readable, and once one has been held for 10 seconds it calls `visibilityStallChanged(storage, true)`; once it is delivered, `false`. A stall still in progress when the storage closes is reported cleared, so a gauge built on it does not keep showing a stall for a storage that is gone. The in-memory backends have no barrier and never call it.

> Only a store something is appended to can notice a stall: a stall during which nothing is appended withholds nothing. Monitoring on the database stays the complete answer — see [What Can Stall Reads](/posts/eventstore-configuring-postgresql-storage/#what-can-stall-reads-the-visibility-barrier).
{: .prompt-warning }

## Asking a Storage for Its Health

The observer is told what happens; a health endpoint asks what the state is *now*. `EventStorage.health()` answers a `StorageHealth`: one `HealthCheck` per aspect the backend can judge, each with a name, a `HealthStatus`, a message and details, and a `status()` that is the worst of them.

```java
StorageHealth health = storage.health();                 // waits at most DEFAULT_HEALTH_TIMEOUT, 2s
StorageHealth quick  = storage.health(Duration.ofMillis(500));

health.status();                                          // UP, UNKNOWN, DEGRADED or DOWN
for ( HealthCheck check : health.checks() ) {
    log.info("{}: {} {} {}", check.name(), check.status(), check.message(), check.details());
}
health.check(HealthCheck.OPEN);                           // Optional<HealthCheck>, by name
```

`HealthStatus` is ordered `UP < UNKNOWN < DEGRADED < DOWN`, and the overall status is computed from the checks rather than stated beside them, so a storage can never report itself `UP` beside a check that is `DOWN`. A check's details are JSON scalars — strings, numbers and booleans — in the order the backend gave them, so a `StorageHealth` renders into a health endpoint's response as it is.

It is built for health endpoints, and held to rules an ordinary operation is not:

- **It never throws on account of the storage.** A closed storage answers `DOWN` under the `open` check and asks nothing else; a check that fails is reported as a failed check. Only a timeout that is not positive is refused — that is the caller's mistake.
- **It returns within its timeout**, whatever the database does. A check that has not answered in time is reported as such and left to finish; the next call waits on that same check rather than starting a second one beside it, so a database that stopped answering is asked once, not once per probe.
- **It is not observed and not cached.** A probe is not an operation a caller performed, and every call checks again — pace it in the endpoint that calls it.

### What Each Backend Reports

Every backend reports `HealthCheck.OPEN` (`"open"`), `DOWN` once the storage is closed. The other checks are named by constants on the backend that reports them:

| Check | Backend | Reports |
|---|---|---|
| `InMemoryFsEventStorage.HEALTH_DIRECTORIES` (`directories`) | file-backed in-memory | `DOWN` when the events or bookmarks directory is gone or not writable — every append and bookmark is written to a file as it happens |
| `PostgresEventStorage.HEALTH_DATABASE` (`database`) | PostgreSQL | a pooled connection and `SELECT 1`, with `roundTripMicros`. `DOWN` when it fails (`error`, `sqlState`) or does not answer in time (`timeoutMillis`, `inFlightMillis`). It says the database answers, not that this store's tables are intact — those are checked when the storage is built |
| `HEALTH_EVENT_APPENDED_NOTIFICATIONS` (`eventAppendedNotifications`), `HEALTH_BOOKMARK_PLACED_NOTIFICATIONS` (`bookmarkPlacedNotifications`) | PostgreSQL | `DEGRADED` while the channel's monitor holds no `LISTEN` |
| `HEALTH_VISIBILITY` (`visibility`) | PostgreSQL | `DEGRADED` once an append notification has been withheld behind the visibility barrier for 10 seconds, with `withheldStreams` and `withheldForMillis` |
| `HEALTH_CONNECTION_POOL` (`connectionPool`), `HEALTH_MONITORING_CONNECTION_POOL` (`monitoringConnectionPool`) | PostgreSQL | for a HikariCP pool, whoever created it: `active`, `idle`, `total`, `max` and `waiting`, `DEGRADED` while anyone waits for a connection. The monitoring pool is reported only when the monitors have a pool of their own |

The plain in-memory backend reports `open` alone. A third-party `EventStorage` that does not implement the method answers `UNKNOWN`: it has checked nothing, and does not claim to be up.

### Mapping It onto Your Endpoint

**`DEGRADED` means reads and writes work, and something the application relies on does not**: subscribers are not woken, reads are held behind the barrier, callers queue for a connection. None of those is fixed by restarting the process or taking it out of a load balancer — the channel reconnects by itself, the stall is a transaction elsewhere in the cluster, the queue is load. So a liveness probe that restarts on `DEGRADED` turns a database-side condition into an outage of every instance at once. **`DOWN`** — the storage is closed, or the database does not answer — is the status a readiness probe is for. Which of the degraded checks should still fail readiness is your call; the store states what it sees:

```java
app.get("/health/ready", ctx -> {
    StorageHealth health = storage.health();
    ctx.status(health.status() == HealthStatus.DOWN ? 503 : 200)
       .json(health);
});
```

## What an Observer Must Honour

- **Thread-safe.** A store is used from many threads at once, and so is its observer.
- **Cheap.** Every call is on the caller's thread, inside the operation it describes: a slow observer is a slow store. Anything expensive belongs on a thread of the observer's own.
- **Never throwing.** An operation never fails because its observation did: the library wraps every observer with `EventStoreObserver.contained(...)`, which catches what escapes — a `RuntimeException`, or a `LinkageError` from an observer whose library is missing at runtime — and logs it, at ERROR the first time with its stack trace and at DEBUG after. That is a guard, not a licence: an observer that throws loses what it was recording.

### Cardinality Is the Observer's Concern

Every observation carries the stream id as it is, and `purpose` can be an entity id where a context is [split into a stream per entity](/posts/eventstore-stream-design-and-performance/#when-a-stream-per-entity-is-worth-it). A tracer or a log line wants that purpose uncapped. A **metrics** registry does not: it never evicts a meter, so an uncapped per-entity tag is a leak that fails nothing — the process just gets steadily heavier. An observer turning the purpose into a metrics tag must bound the values it admits, which is why the cap lives where the tag value is chosen, in the observer, and not in the store.

## Example: a Micrometer Observer

As an example of what you could build on the SPI, here is an observer that reports to Micrometer. It fits on a page: it times every operation by kind and answer, and exposes channel health and visibility stalls as gauges. Adapt it to the meters, tags and naming conventions your application already uses:

```java
import io.micrometer.core.instrument.*;
import org.sliceworkz.eventstore.observability.*;

public final class MicrometerObserver implements EventStoreObserver {

    private static final int MAX_PURPOSES = 1000;

    private final MeterRegistry registry;
    private final Set<String> admittedPurposes = ConcurrentHashMap.newKeySet();
    private final Map<String, AtomicInteger> channels = new ConcurrentHashMap<>();
    private final Map<String, AtomicInteger> stalls = new ConcurrentHashMap<>();

    public MicrometerObserver ( MeterRegistry registry ) {
        this.registry = registry;
    }

    @Override
    public <O extends Outcome> Observation.Scope<O> start ( Observation<O> observation ) {
        long started = System.nanoTime();
        return new Observation.Scope<>() {
            private String answer = "unknown";

            @Override public void completed ( O outcome ) {
                answer = switch ( outcome ) {
                    case Outcome.Conflicted c -> "conflicted";
                    case Outcome.Duplicated d -> "duplicated";
                    default -> "ok";
                };
            }

            @Override public void failed ( Throwable failure ) {
                answer = "error";
            }

            @Override public void close ( ) {
                Timer.builder("eventstore.operation")
                     .tag("operation", observation.getClass().getSimpleName())
                     .tag("outcome", answer)
                     .tags(streamTags(observation))
                     .register(registry)
                     .record(System.nanoTime() - started, TimeUnit.NANOSECONDS);
            }
        };
    }

    @Override
    public void notificationChannelChanged ( String storage, NotificationChannel channel, boolean listening ) {
        channels.computeIfAbsent(storage + "/" + channel, key -> {
            AtomicInteger state = new AtomicInteger();
            Gauge.builder("eventstore.notifications.up", state, AtomicInteger::get)
                 .tag("storage", storage).tag("channel", channel.name().toLowerCase())
                 .register(registry);
            return state;
        }).set(listening ? 1 : 0);
    }

    @Override
    public void visibilityStallChanged ( String storage, boolean stalled ) {
        stalls.computeIfAbsent(storage, key -> {
            AtomicInteger state = new AtomicInteger();
            Gauge.builder("eventstore.visibility.stalled", state, AtomicInteger::get)
                 .tag("storage", storage)
                 .register(registry);
            return state;
        }).set(stalled ? 1 : 0);
    }

    private Tags streamTags ( Observation<?> observation ) {
        if ( !(observation instanceof Observation.OnStream<?> onStream) ) {
            return Tags.of("storage", observation.storage());
        }
        EventStreamId id = onStream.stream().stream();
        String purpose = id.purpose() == null ? "" : id.purpose();
        // bound the purpose: the first MAX_PURPOSES seen keep their own value, the rest are pooled
        if ( !admittedPurposes.contains(purpose)
                && ( admittedPurposes.size() >= MAX_PURPOSES || !admittedPurposes.add(purpose) ) ) {
            purpose = "_other";
        }
        return Tags.of("storage", observation.storage(),
                       "context", id.context() == null ? "" : id.context(),
                       "purpose", purpose);
    }
}
```

(`Tags` here is Micrometer's `io.micrometer.core.instrument.Tags`, not the eventstore's.) Because the scope is synchronous, the same shape works for a tracer: open a span in `start`, make it current, set its status in `completed`/`failed`, and end it in `close`.

To count events rather than operations, read the outcome: `Outcome.Appended.storedPerType()` and `Outcome.Read.readPerStoredType()` break an operation down by event type, and `Observation.Append.submittedPerType()` against a `Duplicated` answer says how many submitted events a retry swallowed.

### Connection Pool Metrics

HikariCP has its own metrics seam, and the PostgreSQL builder hands it to the pools the storage uses, whether the builder created them or you supplied them:

```java
PostgresEventStorage.newBuilder()
    .observer(new MicrometerObserver(registry))
    .poolMetrics(new MicrometerMetricsTrackerFactory(registry))   // HikariCP's own Micrometer tracker
    .buildStore();
```

## Example Configuration: Prometheus

To expose metrics to Prometheus, add the Prometheus Micrometer registry dependency and configure an HTTP endpoint.

### Maven Dependencies

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
    <version>1.17.1</version>
</dependency>
<dependency>
    <groupId>io.javalin</groupId>
    <artifactId>javalin</artifactId>
    <version>7.2.3</version>
</dependency>
```

### Java Configuration with Javalin

```java
import io.javalin.Javalin;
import io.micrometer.prometheusmetrics.PrometheusConfig;
import io.micrometer.prometheusmetrics.PrometheusMeterRegistry;
import com.zaxxer.hikari.metrics.micrometer.MicrometerMetricsTrackerFactory;

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

        try ( EventStore eventStore = PostgresEventStorage.newBuilder()
                  .observer(new MicrometerObserver(prometheusRegistry))
                  .poolMetrics(new MicrometerMetricsTrackerFactory(prometheusRegistry))
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

With the observer above, Grafana panels over Prometheus look like this.

**Panel: Append Rate by Stream Context**
```promql
sum by (context) (rate(eventstore_operation_seconds_count{operation="Append"}[5m]))
```

**Panel: Query Duration (95th Percentile)** — enable histograms on the timer (`.publishPercentileHistogram()`) for this one
```promql
histogram_quantile(0.95,
  sum by (le) (rate(eventstore_operation_seconds_bucket{operation="Query"}[5m])))
```

**Panel: Optimistic Locking Conflict Rate**
```promql
sum(rate(eventstore_operation_seconds_count{operation="Append",outcome="conflicted"}[5m]))
  / sum(rate(eventstore_operation_seconds_count{operation="Append"}[5m]))
```

**Panel: Retries Swallowed by Idempotency**
```promql
sum(rate(eventstore_operation_seconds_count{operation="Append",outcome="duplicated"}[5m]))
```

**Panel: Notification Health**
```promql
min by (channel) (eventstore_notifications_up)
```

### Key Signals to Monitor

- **Notifications down**: `eventstore_notifications_up == 0` means read models have stopped advancing on their own, while everything else keeps working. This is the one failure that is otherwise silent, and the first thing to alert on:
  ```promql
  min_over_time(eventstore_notifications_up[1m]) == 0
  ```
- **High optimistic locking conflicts**: a boundary shared by many writers — widen it; see [Stream Design and Performance](/posts/eventstore-stream-design-and-performance/)
- **Slow queries**: compare the operation's duration with its `storageTime` first — a read returning thousands of events spends most of its time deserializing, not in the database
- **Subscriptions only rising**: `subscriptionOpened` minus `subscriptionClosed` climbing without bound is a subscribed stream nobody closes
- **A `purpose="_other"` series appearing**: the cardinality cap in your observer has been reached, so per-purpose breakdowns are no longer complete

## Testing With an Observer

The testing module publishes `RecordingObserver`, which keeps everything it is told and checks the contract a store owes an observer along the way — see [Testing Your Application](/posts/eventstore-testing/#asserting-what-the-store-reports-recordingobserver).
