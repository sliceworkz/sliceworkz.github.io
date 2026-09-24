---
layout: post
toc: true
title: Projecting Events
description: Projecting Domain Events into a read model
date: 2025-11-29 04:00:00
categories: [Eventstore Documentation,Eventstore API]
tags: [events,projection,projector,read model]
---

This guide covers how to build projections from event streams, including event handlers, read models, and using the Projector utility.

## Projections from Your Events

Projections process events from a stream to create current-state views needed by your application. 
They produce views that make sense out of all the business events that happened until a certain moment in time.

These materialized views enable:

- **Business decisions**: Aggregating historical facts to validate new commands
- **User screens**: Building denormalized data structures for efficient display
- **REST endpoints**: Generating response payloads from event history
- **Reports**: Calculating metrics and analytics from domain events
- etc...

A projection queries relevant events and applies them sequentially to build state:

```java
// Projection builds current customer state from events
CustomerProjection projection = new CustomerProjection("123");
Projector.from(stream).into(projection).build().run();

// Use the projection result for business logic
if (!projection.isChurned()) {
    // Process order for active customer
}
```

Projections are deterministic: replaying the same events in the same order always produces the same result.

## Writing an Event Handler

`EventHandler` is the one handler interface, and `when(Event<E>)` its one method: the projector hands every matching event to it, one call per event, with its metadata — timestamp, tags, reference and stream — alongside the domain event in `data()`.

```java
public class CustomerCounter implements EventHandler<CustomerEvent> {
    private int registrationCount = 0;
    private int churnCount = 0;

    @Override
    public void when(Event<CustomerEvent> event) {
        switch (event.data()) {
            case CustomerRegistered r -> registrationCount++;
            case CustomerChurned c -> churnCount++;
            default -> {} // Ignore other events
        }
    }

    public int getCount() { return registrationCount; }
}
```

A handler that needs only the domain event switches on `event.data()`; one that needs the metadata reads it from the same argument:

```java
public class CustomerTimeline implements EventHandler<CustomerEvent> {
    private final List<TimelineEntry> timeline = new ArrayList<>();

    @Override
    public void when(Event<CustomerEvent> event) {
        timeline.add(new TimelineEntry(
            event.timestamp(),                       // an Instant
            event.reference().position(),
            event.data().getClass().getSimpleName()
        ));
    }

    public List<TimelineEntry> getTimeline() { return timeline; }
}
```

`EventHandler` is a functional interface, enabling lambda usage:

```java
EventHandler<CustomerEvent> logger = event ->
    System.out.println("Event: " + event.type().name() + " at " + event.reference());
```

The metadata is there when you need it:
- Event timestamps for temporal information that is not in your event payload
- Tags for correlation, filtering or additional information stored therein (audit logging, ...)
- References for tracking position
- Stream information for multi-stream projections

## Implementing a Readmodel

A readmodel is a projection that implements the `Projection` interface, combining an `EventQuery` with an event handler:

```java
public class CustomerSummary implements Projection<CustomerEvent> {
    private final String customerId;
    private String name;
    private boolean churned;

    public CustomerSummary(String customerId) {
        this.customerId = customerId;
    }

    @Override
    public EventQuery eventQuery() {
        // Query all events for this specific customer
        return EventQuery.forTags(Tags.of("customer", customerId));
    }

    @Override
    public void when(Event<CustomerEvent> event) {
        switch(event.data()) {
            case CustomerRegistered r -> this.name = r.name();
            case CustomerNameChanged n -> this.name = n.name();
            case CustomerChurned c -> this.churned = true;
        }
    }

    public String getName() { return name; }
    public boolean isChurned() { return churned; }
}
```

The `eventQuery()` method defines which events are relevant. The Projector will only call `when()` for matching events.

**`eventQuery()` is read once per run.** Storage is asked with that query and every event of the run is matched against it, so a projection that computes its query cannot answer differently halfway through a run — and is not asked once per event, either.

## Using the Projector

The `Projector` utility class executes projections by querying events and applying them to the handler:

```java
EventStream<CustomerEvent> stream = eventstore.getEventStream(
    EventStreamId.forContext("customers"),
    CustomerEvent.class
);

CustomerSummary projection = new CustomerSummary("123");

Projector.from(stream)
    .into(projection)
    .build()
    .run();

// Projection now contains current state
System.out.println("Customer: " + projection.getName());
```

The Projector:
1. Queries events matching the projection's `eventQuery()`
2. Reads them in pages — batches — to avoid memory issues
3. Calls `when()` for each matching event
4. Returns metrics about the projection execution

`Projector.from(stream).into(projection)` is the whole of a projector; every other setting below is an optional call on the same builder. `from(null)` is refused at the call, and `build()` refuses a missing projection with an `IllegalStateException` naming the call to make, rather than letting it fail from inside the first batch.

A batch is read as one [`EventPage`](/posts/eventstore-querying-events/#paging-through-a-stream), deserialized whole before the first of its events is handed to `when()`. So a batch holding an event this stream cannot read hands none of its events to the projection, and fails the run — see [Error Handling](/posts/eventstore-error-handling/#failures-inside-a-projector).

## Configuring the Projector

The Projector handles complexity so your application doesn't need to. By default, it queries events in batches of 500 to prevent memory exhaustion with large streams:

```java
// Use default batch size (500)
Projector.from(stream)
    .into(projection)
    .build()
    .run();

// Configure smaller batches for memory-constrained environments
Projector.from(stream)
    .into(projection)
    .inBatchesOf(100)
    .build()
    .run();

// Configure larger batches for better throughput
Projector.from(stream)
    .into(projection)
    .inBatchesOf(1000)
    .build()
    .run();
```

`inBatchesOf(n)` refuses a batch size below 1. The Projector automatically handles pagination—your projection code remains simple regardless of stream size. The batch boundary is also where a projection can commit its own work and where the bookmark is placed — see [Batch-Aware Projections](#batch-aware-projections).

You can also configure where to start processing:

```java
// Start from a specific position
EventReference checkpoint = // ... from somewhere ...

Projector.from(stream)
    .into(projection)
    .startingAfter(checkpoint)
    .build()
    .run();
```

This could, for example, be useful if you already have a persisted or existing projection that is up-to-date to a certain point you want to update with new events that were appended to the eventstore.

## Reusing the Projector

A Projector instance tracks its position in the event stream and can be reused for incremental updates:

```java
CustomerSummary projection = new CustomerSummary("123");

Projector<CustomerEvent> projector = Projector.from(stream)
    .into(projection)
    .build();

// Initial run - process all historical events
ProjectorMetrics metrics1 = projector.run();
System.out.println("Initial: " + metrics1.eventsHandled() + " events");

// ... time passes, new events are appended to stream ...

// Incremental run - process only new events since last run
ProjectorMetrics metrics2 = projector.run();
System.out.println("Incremental: " + metrics2.eventsHandled() + " new events");

// The projection is now up-to-date
System.out.println("Current state: " + projection.getName());
```

The Projector remembers the last processed event reference and automatically resumes from that position on subsequent runs.

You can also process up to a specific point in time:

```java
// Process events up to a historical checkpoint
ProjectorMetrics metrics = projector.runUntil(historicalReference);

// Projection now reflects state as of that moment
```

This enables:
- **Point-in-time queries**: Reconstruct historical states
- **Controlled updates**: Process events in stages
- **Testing**: Verify projection behavior at specific points

## Subscribing to Stream Updates

Instead of manually calling `run()` to update projections, you can subscribe a projector to automatically receive notifications when new events are appended. Simply include `subscribe()` in the builder chain:

```java
CustomerSummary projection = new CustomerSummary("123");

Projector<CustomerEvent> projector = Projector.from(stream)
    .into(projection)
    .subscribe()
    .build();

// New events automatically trigger projection updates
stream.append(
    AppendCriteria.none(),
    Event.of(new CustomerRegistered("Alice"), Tags.of("customer", "123"))
);

// The projection is updated asynchronously
```

The `subscribe()` method configures the projector to register itself as an append listener on the stream — a `Projector` *is* an `AppendListener`. When events are appended, the projector's `eventsAppended()` method is invoked asynchronously, triggering a `run()` to process new events.

The projector keeps no `Subscription` handle of its own: it ends with its source. To end one projector's subscription without closing the stream, build it without `subscribe()` and subscribe it yourself, keeping the handle:

```java
Projector<CustomerEvent> projector = Projector.from(stream).into(projection).build();
Subscription subscription = stream.subscribe(projector);
// ...
subscription.close();   // this projector stops; other subscriptions on the stream carry on
```

This subscription-based approach is ideal for keeping read models current with minimal latency. The projector automatically handles incremental updates, processing only events since its last run.

> **A subscribed projector keeps its stream alive.** Subscribing registers the stream with the storage, which holds it while it has a live subscription — so a subscribed stream that is never closed is retained for the lifetime of the storage. Close the stream when the projection is no longer needed, or close the store, which closes them all. See [Lifecycle and Shutdown](/posts/eventstore-lifecycle/#closing-a-stream).
{: .prompt-warning }

**A failing projection does not fail the append, and does not retry itself.** `run()` throws a `ProjectorException`, which is contained and logged at ERROR by the notification machinery: the appending caller is never told, other subscribers still get their notification, and nothing replays the notification this projector failed on. It is notified again on the next append.

That is exactly why a subscribed projector doing work that must not be lost should also bookmark its progress — the bookmark, not the notification, is what makes the next run resume from the right place.

Combine subscriptions with bookmarking for resilience across restarts:

```java
Projector<CustomerEvent> projector = Projector.from(stream)
    .into(projection)
    .subscribe()
    .bookmarkAs("customer-summary")
    .build();
```

With this configuration:
- **On startup**: The projector reads the bookmark and catches up to the current position
- **During operation**: New appends trigger automatic incremental updates
- **After each update**: The bookmark is saved, enabling seamless recovery

## Initializing Projections with Savepoints

For projections that are created on-the-fly — for example, to answer a query in an API request — replaying the entire event history on every instantiation can be expensive. The **savepoint pattern** addresses this by allowing projections to initialize from a recent snapshot event rather than replaying from the beginning.

A projection can optionally implement `initQuery()` to find the most recent savepoint event before the main `eventQuery()` runs:

```java
public class StockLevelProjection implements Projection<StockEvent> {
    private final String product;
    private int level = 0;

    public StockLevelProjection(String product) {
        this.product = product;
    }

    @Override
    public EventQuery initQuery() {
        return EventQuery.forEvents(
            EventTypesFilter.of(StockCounted.class),
            Tags.of("product", product)
        ).backwards().limit(1);
    }

    @Override
    public EventQuery eventQuery() {
        return EventQuery.forEvents(
            EventTypesFilter.of(StockAdded.class, StockPicked.class),
            Tags.of("product", product)
        );
    }

    @Override
    public void when(Event<StockEvent> event) {
        switch (event.data()) {
            case StockCounted c  -> level = c.counted();
            case StockAdded a    -> level += a.quantity();
            case StockPicked p   -> level -= p.quantity();
        }
    }

    public int level() { return level; }
}
```

The `initQuery()` runs once on the first projector execution. If a savepoint is found, the projection initializes from it and the main `eventQuery()` processes only the events that occurred after. When no savepoint exists, the pattern degrades gracefully — the full stream is replayed.

A savepoint handler that throws fails the run as a `ProjectorException` naming the savepoint event, exactly as a failing batch does, and the cursor goes back to where the run started — so the next run re-runs `initQuery()` rather than starting the main query from a read model that was never initialised.

> When bookmarking is enabled, `initQuery()` is ignored. Bookmarked projections track their own position and must process every event.
{: .prompt-warning }

For a detailed discussion of the savepoint pattern — including design decisions, comparison with bookmarking, and strategies for when to create savepoints — see the dedicated [Savepoint Pattern](/posts/eventstore-savepoint-pattern/) article.

## Interpreting Metrics

The Projector returns `ProjectorMetrics` containing detailed statistics about projection execution:

```java
ProjectorMetrics metrics = projector.run();

System.out.println("Events streamed: " + metrics.eventsStreamed());
System.out.println("Events handled: " + metrics.eventsHandled());
System.out.println("Queries done: " + metrics.queriesDone());
System.out.println("Last event: " + metrics.lastEventReference());
System.out.println("Most recent: " + metrics.mostRecentEventReference());
```

### Metrics from the Last Run

`ProjectorMetrics` returned from `run()` or `runUntil()` describes that specific execution:

- **eventsStreamed**: Total events retrieved from the event source (may include filtered events)
- **eventsHandled**: Events actually processed by the projection handler
- **queriesDone**: Number of batch queries executed against the event source
- **lastEventReference**: Reference to the last processed event (cursor position — the event the projector will resume after on the next run)
- **mostRecentEventReference**: The chronologically newest event seen during this execution. For forward queries this equals `lastEventReference`. For backward queries it points to the first event encountered (which is the newest chronologically). This is useful for optimistic locking when using backward projections

```java
ProjectorMetrics metrics = projector.run();

if (metrics.eventsHandled() == 0) {
    System.out.println("No new events to process");
} else {
    System.out.println("Processed " + metrics.eventsHandled() +
                       " events in " + metrics.queriesDone() + " batches");
}
```

The difference between `eventsStreamed` and `eventsHandled` indicates filtering efficiency: events matched by the query but filtered out by the projection logic.

### Accumulated Metrics

The Projector tracks total metrics across all runs. `accumulatedMetrics()` is also the way to read a *subscribed* projector's position from another thread: it is published by every run, whereas `run()`'s return value is only seen by the thread that called it — which, for a subscribed projector, is the storage's notification thread.

```java
Projector<CustomerEvent> projector = Projector.from(stream)
    .into(projection)
    .build();

// Run 1
projector.run(); // Handles 100 events

// Run 2
projector.run(); // Handles 10 new events

// View totals
ProjectorMetrics total = projector.accumulatedMetrics();
System.out.println("Total events handled: " + total.eventsHandled()); // 110
System.out.println("Total queries: " + total.queriesDone());
System.out.println("Current position: " + total.lastEventReference());
System.out.println("Most recent event: " + total.mostRecentEventReference());
```

Accumulated metrics are useful for:
- **Monitoring**: Track total events processed over time
- **Debugging**: Identify performance issues across runs
- **Resumption**: Get the current position for checkpointing
- **Reporting**: Calculate processing statistics

## Bookmarking for Process Restart and Progress Tracking

Bookmarking allows projectors to automatically save and restore their position in the event stream. This enables projectors to:

- **Resume after restart**: Pick up exactly where they left off when your application restarts
- **Track progress**: Monitor how far a projection has processed through the event stream
- **Allow projection updates to run subsequently on different instances**: Share position with next application instance needing it
- **Enable incremental updates**: Process only new events since the last run (without relying on a local in-process variable)

A bookmark consists of a reader name (unique identifier) and an event reference (position in the stream).   
Optionally, you can add tags to add metadata (eg: application version, hostname, process id, ... that placed the bookmark)

### Basic Bookmarking

Name the reader whose bookmark records the projector's progress:

```java
CustomerSummary projection = new CustomerSummary("123");

Projector<CustomerEvent> projector = Projector.from(stream)
    .into(projection)
    .bookmarkAs("customer-summary-projection")
    .build();

// First run processes all historical events and saves bookmark
projector.run();

// Application restarts...

// Second run automatically reads bookmark and processes only new events
projector.run();
```

Projectors with different reader names keep independent positions; two built with the same name share one — which is what lets a restarted process resume where its predecessor left off. `bookmarkAs(reader, tags)` stores tags with the bookmark — a tenant, a schema version, the instance that placed it — which take no part in reading it back. A bookmarked projector ignores its projection's `initQuery()`, since the bookmark already says where to resume.

The projector automatically:
1. Reads the bookmark before each run to determine the starting position
2. Processes events from that position forward
3. Saves the updated bookmark **after every batch**, not once at the end of the run — so a catch-up interrupted halfway resumes near where it stopped instead of replaying everything it had already processed

### Continuous Projection Updates

For long-running projections that need to stay current with new events, combine bookmarking with event notifications:

```java
public class CustomerProjectionService {
    private final EventStream<CustomerEvent> stream;
    private final CustomerSummary projection;
    private final Projector<CustomerEvent> projector;
    private final Subscription subscription;

    public CustomerProjectionService(EventStore eventStore) {
        EventStreamId streamId = EventStreamId.forContext("customers");
        this.stream = eventStore.getEventStream(streamId, CustomerEvent.class);
        this.projection = new CustomerSummary("123");

        // Configure projector with bookmarking
        this.projector = Projector.from(stream)
            .into(projection)
            .bookmarkAs("customer-summary", Tags.of("customer", "123"))
            .build();

        // Subscribe to append notifications
        this.subscription = stream.subscribe(this::updateProjection);
    }

    /*
     * Called asynchronously each time events are appended to the stream.
     * Returning the reference reached tells the store this listener has caught up;
     * returning null means the same thing, and is what happens when the run
     * matched no events.
     */
    private EventReference updateProjection(EventReference atLeastUntil) {

        // Project new events, bookmark is fetched before this run (ref builder instructions above)
        ProjectorMetrics metrics = projector.run();

        if (metrics.eventsHandled() > 0) {
            System.out.println("Processed " + metrics.eventsHandled() + " new events");
            System.out.println("Current position: " + metrics.lastEventReference());
        }

        // Bookmark is automatically updated after each run
        
        return metrics.lastEventReference();
    }

    public CustomerSummary getProjection() {
        return projection;
    }
}
```

This pattern ensures your read model stays synchronized with the event stream:

1. **Initial state**: The projector reads the bookmark to resume from the last known position
2. **New events**: When events are appended, the notification triggers an update
3. **Incremental processing**: Only new events since the bookmark are processed
4. **Automatic bookmark update**: The new position is saved for the next run

### Bookmark Read Frequencies

A bookmarked projector reads its bookmark **before every run** by default. That is what a distributed deployment needs: when another instance moved the bookmark — a takeover after [leader election](/posts/eventstore-leader-election/) — the next run starts where that instance left off. Two readings differ from the default, so those are the two settings that exist:

```java
// Default: read before every run - for distributed systems
Projector.from(stream)
    .into(projection)
    .bookmarkAs("my-projection")
    .build();

// Read once, before the first run, then keep the projector's own cursor -
// for a long-lived projector that is the only writer of its bookmark (saves a lookup per run)
Projector.from(stream)
    .into(projection)
    .bookmarkAs("my-projection")
    .readBookmarkOnce()
    .build();

// Read only when asked - the projector starts from startingAfter(...) or the beginning
Projector<CustomerEvent> projector = Projector.from(stream)
    .into(projection)
    .bookmarkAs("my-projection")
    .readBookmarkOnRequest()
    .build();

// Explicitly read bookmark when needed
projector.readBookmark();
projector.run();
```

Whichever is chosen, the bookmark is still *placed* after every batch. `readBookmarkOnRequest()` is the setting for a projection that holds its own position in its own store (see [Being Exactly-Once Against Your Own Store](#being-exactly-once-against-your-own-store)), and for a test that wants to decide when the bookmark is consulted. Choosing either without `bookmarkAs(...)` is refused at `build()` — there is no bookmark to read.

**`readBookmark()` never overlaps a run.** Both take the projector's lock, so a manual read while a run is in progress — a subscribed projector runs on the storage's notification thread — waits for the run to finish and then resets the position, rather than moving a cursor the run is about to overwrite with its own progress.

## Naming a Projector for Its Observations

When the store is [observed](/posts/eventstore-observability-micrometer-prometheus-grafana/), each batch a projector runs is reported under the projection's class simple name (the full class name for an anonymous class). A framework that wraps its own components in one adapter class would otherwise report all of them under the adapter's name; `named(...)` says otherwise:

```java
Projector.from(stream)
    .into(new ReadModelAdapter(customerList))
    .named("customer-list")
    .bookmarkAs("customer-list")
    .build();
```

The name is for observation only and takes no part in bookmarking, which is keyed by the reader.

## Batch-Aware Projections

A projection that writes to a durable store of its own usually wants to commit per batch rather than per event. Implement `BatchAwareProjection` instead of `Projection` and the projector calls three extra hooks around each batch — a batch being one query execution, so up to `inBatchesOf(...)` events:

| Hook | When |
|---|---|
| `beforeBatch()` | once before the first event of a batch — begin a transaction, take a resource |
| `afterBatch(Optional<EventReference>)` | after every event of the batch was handled — commit |
| `cancelBatch()` | when anything threw while processing the batch — roll back |

```java
class CustomerListProjection implements BatchAwareProjection<CustomerEvent> {

    private final EntityManager em;
    private EntityTransaction tx;

    @Override
    public void beforeBatch() {
        tx = em.getTransaction();
        tx.begin();
    }

    @Override
    public void when(Event<CustomerEvent> event) {
        if (event.data() instanceof CustomerRegistered reg) {
            em.persist(new CustomerEntity(reg.id(), reg.name()));
        }
    }

    @Override
    public void afterBatch(Optional<EventReference> lastEventReference) {
        tx.commit();
    }

    @Override
    public void cancelBatch() {
        if (tx != null && tx.isActive()) {
            tx.rollback();
        }
    }

    @Override
    public EventQuery eventQuery() {
        return EventQuery.forEvents(EventTypesFilter.of(CustomerRegistered.class), Tags.none());
    }
}
```

`beforeBatch()` is not called for a batch in which nothing matched the projection's query.

### What the Batch Boundary Guarantees

The projection commits its own work in `afterBatch`; the bookmark saying how far it has come lives in the event store. No transaction spans the two, so the **ordering** is the whole guarantee:

- **The batch is committed first and bookmarked second.** A crash in that window costs a re-projection of one batch, never a silent skip. At-least-once, deliberately in that direction.
- **Throwing from `afterBatch` means the batch did not land.** The projector treats a failed commit exactly like a failure during processing: it is reported as a `ProjectorException`, and the projector's cursor is taken back to where the batch started, so those events are offered again on the next run. Never swallow a failed commit here — a projection reporting success for work it did not persist loses those events with nothing raised anywhere.
- **A batch is ended exactly once.** `cancelBatch()` is *not* called after an `afterBatch` that threw, since that projection has already released what it held. And a `cancelBatch()` that throws is logged and attached to the original failure as a **suppressed** exception rather than replacing it, so a poison event whose rollback also failed is still reported as the poison event.

### Being Exactly-Once Against Your Own Store

`afterBatch` is handed the batch's last `EventReference` for one specific reason: where re-projecting a batch would *duplicate* rather than merely repeat work, the projection should hold its own position.

```java
@Override
public void afterBatch(Optional<EventReference> lastEventReference) {
    // same transaction, same store: position and data commit together
    lastEventReference.ifPresent(ref ->
        em.merge(new ProjectionCursor("customer-list", ref.toString())));
    tx.commit();
}
```

Resume from that cursor instead of from the event store's bookmark, and the projection's own store becomes the authority on how far it has come — the only way to be exactly-once across two stores that cannot share a transaction. The alternative is to make `when` idempotent, for example by ignoring any event whose position is not higher than the one already recorded alongside the read model.
