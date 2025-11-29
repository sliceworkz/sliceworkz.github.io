---
layout: post
toc: true
title: Querying Events
description: Querying for Domain Events
date: 2025-11-29 03:00:00
categories: [Eventstore Documentation]
tags: [events,query,tags]
---

This guide covers the various ways to query events from the EventStore, including filtering, pagination, backward queries, temporal queries, and cross-stream querying.

## Querying all Domain Events in a Stream

The simplest query retrieves all events from a stream using `EventQuery.matchAll()`:

```java
EventStream<CustomerEvent> stream = eventstore.getEventStream(
    EventStreamId.forContext("customer").withPurpose("123"),
    CustomerEvent.class
);

Stream<Event<CustomerEvent>> allEvents = stream.query(EventQuery.matchAll());
allEvents.forEach(event -> System.out.println(event));
```

**Important**: This approach can lead to performance and memory problems if the stream contains a large number of events. For long streams, use batched queries (described below) to retrieve events in manageable chunks.

## Querying Domain Events on Type and Tags

Events can be filtered by combining event type filters with tags. The matching semantics are:

- **Event Types**: The event must match **any** of the specified types (OR condition)
- **Tags**: The event must contain **all** specified tags (AND condition)

Note that events can have additional tags beyond those specified in the query—the query only requires that the specified tags are present.

```java
// Query for specific event types with specific tags
EventQuery query = EventQuery.forEvents(
    EventTypesFilter.of(CustomerRegistered.class, CustomerNameChanged.class),
    Tags.of("region", "EU")
);

Stream<Event<CustomerEvent>> events = stream.query(query);
```

### Event Type filtering

```java
// Match any event type
EventQuery anyType = EventQuery.forEvents(
    EventTypesFilter.any(),
    Tags.of("customer", "123")
);

// Match a single type
EventQuery singleType = EventQuery.forEvents(
    EventTypesFilter.of(CustomerChurned.class),
    Tags.none()
);

// Match multiple types (OR condition)
EventQuery multipleTypes = EventQuery.forEvents(
    EventTypesFilter.of(
        CustomerRegistered.class,
        CustomerNameChanged.class,
        CustomerChurned.class
    ),
    Tags.none()
);
```

### Tag filtering

```java
// Single tag
EventQuery singleTag = EventQuery.forEvents(
    EventTypesFilter.any(),
    Tags.of("customer", "123")
);

// Multiple tags (ALL must be present)
EventQuery multipleTags = EventQuery.forEvents(
    EventTypesFilter.any(),
    Tags.of("customer", "123", "region", "EU", "priority", "high")
);
// Matches events that have AT LEAST these three tags
// (events can have additional tags)
```

## Querying in batches

For large event streams, use `Limit` to retrieve events in batches. This prevents memory exhaustion and improves performance:

```java
EventQuery query = EventQuery.matchAll();
EventReference lastRef = null;
int batchSize = 100;

while (true) {
    // Query next batch starting after the last reference
    Stream<Event<CustomerEvent>> batch = stream.query(
        query,
        lastRef,
        Limit.to(batchSize)
    );

    List<Event<CustomerEvent>> events = batch.toList();
    if (events.isEmpty()) {
        break; // No more events
    }

    // Process batch
    events.forEach(event -> processEvent(event));

    // Update reference to last event in this batch
    lastRef = events.getLast().reference();
}
```

The `after` parameter is a technical optimization that tells the store where to start scanning. It doesn't affect which events match the query, only where the scan begins.

## Querying backwards

Backward queries return events in reverse chronological order (newest first). This is useful for finding the most recent events or the last occurrence of a business fact:

```java
// Get the last 10 events
Stream<Event<CustomerEvent>> recentEvents = stream.queryBackwards(
    EventQuery.matchAll(),
    Limit.to(10)
);

// Find the last CustomerRegistered event
Optional<Event<CustomerEvent>> lastRegistration = stream.queryBackwards(
    EventQuery.forEvents(
        EventTypesFilter.of(CustomerRegistered.class),
        Tags.none()
    ),
    Limit.to(1)
).findFirst();
```

### Backward Pagination

```java
EventReference beforeRef = null;
while (true) {
    Stream<Event<CustomerEvent>> batch = stream.queryBackwards(
        EventQuery.matchAll(),
        beforeRef,
        Limit.to(100)
    );

    List<Event<CustomerEvent>> events = batch.toList();
    if (events.isEmpty()) {
        break;
    }

    // Process events (already in reverse order)
    events.forEach(event -> processEvent(event));

    // Update to continue before the first event in this batch
    beforeRef = events.getLast().reference();
}
```

## Querying until a certain moment in time

The `until` parameter allows querying events up to a specific point in history. This is fundamental to event sourcing, enabling reconstruction of system state as it existed at any past moment:

```java
// Get all events up to a specific reference
List<Event<CustomerEvent>> allEvents = stream.query(
    EventQuery.forEvents(EventTypesFilter.any(), Tags.of("customer", "123"))
).toList();

EventReference momentInTime = allEvents.get(5).reference(); // 6th event

// Query events up to that moment
EventQuery historicalQuery = EventQuery.forEvents(
    EventTypesFilter.any(),
    Tags.of("customer", "123")
).until(momentInTime);

Stream<Event<CustomerEvent>> pastEvents = stream.query(historicalQuery);
// Returns only events from position 1 through 6
```

This enables time-travel queries to reconstruct how an aggregate or projection looked at any point in history:

```java
// Reconstruct customer state as it was at position 10
CustomerAggregate historicalState = new CustomerAggregate();
stream.query(
    EventQuery.forEvents(EventTypesFilter.any(), Tags.of("customer", "123"))
        .until(EventReference.of(someEventId, 10L))
).forEach(event -> historicalState.apply(event));
```

## Querying by Event ID

A specific event can be retrieved directly by its `EventId`:

```java
EventId eventId = EventId.fromString("550e8400-e29b-41d4-a716-446655440000");

Optional<Event<CustomerEvent>> event = stream.getEventById(eventId);

event.ifPresent(e -> {
    System.out.println("Found event: " + e.data());
    System.out.println("Position: " + e.reference().position());
});
```

You can also query just the reference (without loading the full event):

```java
Optional<EventReference> ref = stream.queryReference(eventId);
ref.ifPresent(r -> System.out.println("Event at position: " + r.position()));
```

## Querying untyped Event data

For scenarios where event types are not statically known or when working with heterogeneous events, obtain an untyped stream using `Object` as the type parameter:

```java
EventStream<Object> untypedStream = eventstore.getEventStream(
    EventStreamId.forContext("customer").withPurpose("123")
);

Stream<Event<Object>> events = untypedStream.query(EventQuery.matchAll());

events.forEach(event -> {
    Object data = event.data();
    System.out.println("Event type: " + data.getClass().getName());
    System.out.println("Event data: " + data);
});
```

This is useful for:
- Generic event processors that don't care about specific types
- Diagnostic or monitoring tools
- Cross-cutting concerns like auditing or event forwarding

## Querying over EventStreams

EventStreams can be queried across multiple contexts or purposes using wildcard stream identifiers:

### Query across all purposes in a context

```java
// Get all events for all customers
EventStream<CustomerEvent> allCustomers = eventstore.getEventStream(
    EventStreamId.forContext("customer").anyPurpose(),
    CustomerEvent.class
);

Stream<Event<CustomerEvent>> allCustomerEvents = allCustomers.query(
    EventQuery.matchAll()
);
```

### Query across all contexts

```java
// Get events from any context with a specific purpose
EventStream<Object> specificPurpose = eventstore.getEventStream(
    EventStreamId.anyContext().withPurpose("analytics")
);

Stream<Event<Object>> events = specificPurpose.query(EventQuery.matchAll());
```

### Query across all contexts and purposes

```java
// Get all events in the entire event store
EventStream<Object> everything = eventstore.getEventStream(
    EventStreamId.anyContext().anyPurpose()
);

Stream<Event<Object>> allEvents = everything.query(EventQuery.matchAll());
```

**Use case example**: Global event monitoring or cross-context analytics:

```java
// Find all events tagged with a specific correlation ID across the entire store
EventStream<Object> globalStream = eventstore.getEventStream(
    EventStreamId.anyContext().anyPurpose()
);

Stream<Event<Object>> correlatedEvents = globalStream.query(
    EventQuery.forEvents(
        EventTypesFilter.any(),
        Tags.of("correlationId", "abc-123")
    )
);
```

## Querying with historical Events

When a stream is configured with historical event types, legacy events are transparently upcasted during queries. Application code only needs to work with current event types:

```java

// Current event definitions
sealed interface CustomerEvent {
    record CustomerRegisteredV2(Name name, Email email) implements CustomerEvent {}
    record CustomerRenamed(Name name) implements CustomerEvent {}
}

// Define historical events separately
sealed interface CustomerHistoricalEvent {
    @LegacyEvent(upcast = CustomerRegisteredUpcaster.class)
    record CustomerRegistered(String name) implements CustomerHistoricalEvent {}
}


// Get stream specifying both current and historical types
EventStream<CustomerEvent> stream = eventstore.getEventStream(
    EventStreamId.forContext("customer").withPurpose("123"),
    CustomerEvent.class,
    CustomerHistoricalEvent.class
);

// Query by current type - includes upcasted historical events
Stream<Event<CustomerEvent>> registrations = stream.query(
    EventQuery.forEvents(
        EventTypesFilter.of(CustomerEvent.CustomerRegisteredV2.class),
        Tags.none()
    )
);

// All events are typed as CustomerEvent (never CustomerHistoricalEvent)
registrations.forEach(event -> {
    CustomerEvent currentEvent = event.data();
    // Legacy CustomerRegistered events are automatically upcasted
    // to CustomerRegisteredV2
});
```

**Key points:**
- Queries use **current event types** only
- Historical events matching the upcasted target type are automatically included
- The upcasting is transparent—application code never sees historical event types
- No special handling needed in query logic for legacy events