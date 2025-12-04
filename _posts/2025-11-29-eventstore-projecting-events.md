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

## Interpreting Metrics

The Projector returns `ProjectorMetrics` containing detailed statistics about projection execution:

```java
ProjectorMetrics metrics = projector.run();

System.out.println("Events streamed: " + metrics.eventsStreamed());
System.out.println("Events handled: " + metrics.eventsHandled());
System.out.println("Queries done: " + metrics.queriesDone());
System.out.println("Last event: " + metrics.lastEventReference());
```

### Metrics from the Last Run

`ProjectorMetrics` returned from `run()` or `runUntil()` describes that specific execution:

- **eventsStreamed**: Total events retrieved from the event source (may include filtered events)
- **eventsHandled**: Events actually processed by the projection handler
- **queriesDone**: Number of batch queries executed against the event source
- **lastEventReference**: Reference to the last processed event

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
```

Accumulated metrics are useful for:
- **Monitoring**: Track total events processed over time
- **Debugging**: Identify performance issues across runs
- **Resumption**: Get the current position for checkpointing
- **Reporting**: Calculate processing statistics