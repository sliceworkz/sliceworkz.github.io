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
Projector.from(stream).towards(projection).build().run();

// Use the projection result for business logic
if (!projection.isChurned()) {
    // Process order for active customer
}
```

Projections are deterministic: replaying the same events in the same order always produces the same result.

## Writing a Simple Event Handler

The `EventHandler` interface processes domain events without metadata. Implement the `when()` method to handle events:

```java
public class CustomerCounter implements EventHandler<CustomerEvent> {
    private int registrationCount = 0;

    @Override
    public void when(CustomerEvent event) {
        if (event instanceof CustomerRegistered) {
            registrationCount++;
        }
    }

    public int getCount() { return registrationCount; }
}
```

Use pattern matching with switch expressions for cleaner code:

```java
@Override
public void when(CustomerEvent event) {
    switch(event) {
        case CustomerRegistered r -> registrationCount++;
        case CustomerChurned c -> churnCount++;
        default -> {} // Ignore other events
    }
}
```

`EventHandler` is a functional interface, enabling lambda usage:

```java
EventHandler<CustomerEvent> logger = event ->
    System.out.println("Event: " + event.getClass().getSimpleName());
```

## Writing an Event Handler with Access to Metadata

The `EventWithMetaDataHandler` interface provides access to the full `Event` wrapper, including timestamp, tags, reference, and stream information:

```java
public class CustomerTimeline implements EventWithMetaDataHandler<CustomerEvent> {
    private List<TimelineEntry> timeline = new ArrayList<>();

    @Override
    public void when(Event<CustomerEvent> event) {
        timeline.add(new TimelineEntry(
            event.timestamp(),
            event.reference().position(),
            event.data().getClass().getSimpleName()
        ));
    }

    public List<TimelineEntry> getTimeline() { return timeline; }
}
```

**Use `EventWithMetaDataHandler` when you need:**
- Event timestamps for temporal information that is not in your event payload
- Tags for correlation, filtering or additional information stored therein (audit logging log, ...)
- References for tracking position
- Stream information for multi-stream projections

**Use `EventHandler` when:**
- You only need the business event data
- Building simple state aggregations
- Metadata is irrelevant to the projection logic

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
        return EventQuery.forEvents(
            EventTypesFilter.any(),
            Tags.of("customer", customerId)
        );
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

For projections that don't need metadata, use `ProjectionWithoutMetaData`:

```java
public class OrderTotal implements ProjectionWithoutMetaData<OrderEvent> {
    private BigDecimal total = BigDecimal.ZERO;

    @Override
    public EventQuery eventQuery() {
        return EventQuery.forEvents(
            EventTypesFilter.of(OrderPlaced.class),
            Tags.none()
        );
    }

    @Override
    public void when(OrderEvent event) {
        if (event instanceof OrderPlaced placed) {
            total = total.add(placed.amount());
        }
    }

    public BigDecimal getTotal() { return total; }
}
```

## Using the Projector

The `Projector` utility class executes projections by querying events and applying them to the handler:

```java
EventStream<CustomerEvent> stream = eventstore.getEventStream(
    EventStreamId.forContext("customers"),
    CustomerEvent.class
);

CustomerSummary projection = new CustomerSummary("123");

Projector.from(stream)
    .towards(projection)
    .build()
    .run();

// Projection now contains current state
System.out.println("Customer: " + projection.getName());
```

The Projector:
1. Queries events matching the projection's `eventQuery()`
2. Streams them in batches to avoid memory issues
3. Calls `when()` for each matching event
4. Returns metrics about the projection execution

## Configuring the Projector

The Projector handles complexity so your application doesn't need to. By default, it queries events in batches of 500 to prevent memory exhaustion with large streams:

```java
// Use default batch size (500)
Projector.from(stream)
    .towards(projection)
    .build()
    .run();

// Configure smaller batches for memory-constrained environments
Projector.from(stream)
    .towards(projection)
    .inBatchesOf(100)
    .build()
    .run();

// Configure larger batches for better throughput
Projector.from(stream)
    .towards(projection)
    .inBatchesOf(1000)
    .build()
    .run();
```

The Projector automatically handles pagination—your projection code remains simple regardless of stream size.

You can also configure where to start processing:

```java
// Start from a specific position
EventReference checkpoint = // ... from somewhere ...

Projector.from(stream)
    .towards(projection)
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
    .towards(projection)
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
    .towards(projection)
    .subscribe()
    .build();

// New events automatically trigger projection updates
stream.append(
    AppendCriteria.none(),
    Event.of(new CustomerRegistered("Alice"), Tags.of("customer", "123"))
);

// The projection is updated asynchronously
```

The `subscribe()` method configures the projector to register itself as an eventually consistent append listener on the stream. When events are appended, the projector's `eventsAppended()` method is invoked asynchronously, triggering a `run()` to process new events.

This subscription-based approach is ideal for keeping read models current with minimal latency. The projector automatically handles incremental updates, processing only events since its last run.

> **A subscribed projector keeps its stream alive.** Subscribing registers the stream with the storage, which holds it until the stream is closed — so a subscribed stream that is never closed is retained for the lifetime of the storage. Close the stream when the projection is no longer needed, or close the store, which closes them all. See [Lifecycle and Shutdown](/posts/eventstore-lifecycle/#closing-a-stream).
{: .prompt-warning }

**A failing projection does not fail the append, and does not retry itself.** `run()` throws a `ProjectorException`, which is contained and logged at ERROR by the notification machinery: the appending caller is never told, other subscribers still get their notification, and nothing replays the notification this projector failed on. It is notified again on the next append.

That is exactly why a subscribed projector doing work that must not be lost should also bookmark its progress — the bookmark, not the notification, is what makes the next run resume from the right place.

Combine subscriptions with bookmarking for resilience across restarts:

```java
Projector<CustomerEvent> projector = Projector.from(stream)
    .towards(projection)
    .subscribe()
    .bookmarkProgress()
        .withReader("customer-summary")
        .readBeforeEachExecution()
        .done()
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

The Projector tracks total metrics across all runs:

```java
Projector<CustomerEvent> projector = Projector.from(stream)
    .towards(projection)
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

Configure bookmarking using the fluent API:

```java
CustomerSummary projection = new CustomerSummary("123");

Projector<CustomerEvent> projector = Projector.from(stream)
    .towards(projection)
    .bookmarkProgress()
        .withReader("customer-summary-projection")
        .done()
    .build();

// First run processes all historical events and saves bookmark
projector.run();

// Application restarts...

// Second run automatically reads bookmark and processes only new events
projector.run();
```

The projector automatically:
1. Reads the bookmark before each run to determine the starting position
2. Processes events from that position forward
3. Saves the updated bookmark after processing new events

### Continuous Projection Updates

For long-running projections that need to stay current with new events, combine bookmarking with event notifications:

```java
public class CustomerProjectionService {
    private final EventStream<CustomerEvent> stream;
    private final CustomerSummary projection;
    private final Projector<CustomerEvent> projector;

    public CustomerProjectionService(EventStore eventStore) {
        EventStreamId streamId = EventStreamId.forContext("customers");
        this.stream = eventStore.getEventStream(streamId, CustomerEvent.class);
        this.projection = new CustomerSummary("123");

        // Configure projector with bookmarking
        this.projector = Projector.from(stream)
            .towards(projection)
            .bookmarkProgress()
                .withReader("customer-summary")
                .withTags(Tags.of("customer", "123"))
                .readBeforeEachExecution()
                .done()
            .build();

        // Subscribe to append notifications
        stream.subscribe(this::updateProjection);
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

Control when bookmarks are read to match your use case:

```java
// Read before each execution (default) - for distributed systems
Projector.from(stream)
    .towards(projection)
    .bookmarkProgress()
        .withReader("my-projection")
        .readBeforeEachExecution()  // Default behavior
        .done()
    .build();

// Read once at creation - for single-instance applications
Projector.from(stream)
    .towards(projection)
    .bookmarkProgress()
        .withReader("my-projection")
        .readAtCreationOnly()
        .done()
    .build();

// Read before first execution - for delayed initialization
Projector.from(stream)
    .towards(projection)
    .bookmarkProgress()
        .withReader("my-projection")
        .readBeforeFirstExecution()
        .done()
    .build();

// Manual control - explicit bookmark management
Projector<CustomerEvent> projector = Projector.from(stream)
    .towards(projection)
    .bookmarkProgress()
        .withReader("my-projection")
        .readOnManualTriggerOnly()
        .done()
    .build();

// Explicitly read bookmark when needed
projector.readBookmark();
projector.run();
```
