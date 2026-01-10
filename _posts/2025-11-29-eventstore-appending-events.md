---
layout: post
toc: true
title: Appending Events
description: Appending Domain Events to the Eventstore
date: 2025-11-29 02:00:00
categories: [Eventstore Documentation,Eventstore API]
tags: [events,append,tags,optimistic locking,dcb,idempotency]
---

This guide covers how to append events to an event stream, including adding tags, understanding event metadata, optimistic locking, and implementing idempotency.

## Appending an Event to a Stream

Events are appended to an `EventStream` using the `append()` method. Before appending, events are created as `EphemeralEvent` instances—lightweight representations without stream association, reference, or timestamp.

```java
EventStream<CustomerEvent> stream = eventstore.getEventStream(
    EventStreamId.forContext("customer").withPurpose("123"),
    CustomerEvent.class
);

stream.append(
    AppendCriteria.none(),
    Event.of(new CustomerRegistered("John"), Tags.none())
);
```

The `append()` method returns a list of fully-formed `Event` objects with assigned metadata (reference, position, timestamp).

## Adding Tags

Tags are key-value pairs that enable dynamic querying and correlation of events across different event types. They are central to the Dynamic Consistency Boundary (DCB) pattern.

```java
// Single tag
stream.append(
    AppendCriteria.none(),
    Event.of(new CustomerRegistered("John"), Tags.of("customer", "123"))
);

// Multiple tags
stream.append(
    AppendCriteria.none(),
    Event.of(
        new CustomerRegistered("John"),
        Tags.of("customer", "123", "region", "EU", "priority", "high")
    )
);
```

Tags enable querying events across different event types based on shared business identifiers:

```java
// Find all events for a specific customer, regardless of event type
Stream<Event<CustomerEvent>> customerEvents = stream.query(
    EventQuery.forEvents(EventTypesFilter.any(), Tags.of("customer", "123"))
);
```

You can also add tags to annotate events with any application-level metadata you need, but that you don't like to put in your event definition payloads.


## Appended Event Metadata

When an event is appended, the EventStore enriches it with metadata:

- **EventReference**: A unique reference containing both a global `EventId` (UUID) and a `position` (sequential number starting at 1)
- **Timestamp**: When the event was persisted
- **Stream**: The `EventStreamId` the event belongs to

```java
List<Event<CustomerEvent>> appended = stream.append(
    AppendCriteria.none(),
    Event.of(new CustomerRegistered("John"), Tags.none())
);

Event<CustomerEvent> event = appended.get(0);
EventReference ref = event.reference();

System.out.println("Event ID: " + ref.id());
System.out.println("Position: " + ref.position());  // Sequential: 1, 2, 3, ...
System.out.println("Timestamp: " + event.timestamp());
```

### EventReference: A unique reference to your event

The `EventReference` combines identity and ordering:
- **EventId**: Globally unique identifier (UUID-based)
- **Position**: Sequential position within the stream (starts at 1, unique over all streams stored in the same storage)

This way, the `EventReference` provides a great reference to:
- determine where you were in processing events in your stream (querying a next batch of Events after that reference next time)
- version a projection built from events, to determine up until which event
- passing a reference to a client to demand a view that has been updated to at least the information that was submitted by that specific client (consistency)
- compare the sequence in which two events have happened, based on their `position` in the stream
- etc... 

EventReference is also crucial for implementing optimistic locking in the DCB pattern. It allows you to note the last relevant event when making a decision, then verify no new relevant facts have emerged when appending the result.

```java
List<Event<CustomerEvent>> events = stream.query(
    EventQuery.forEvents(EventTypesFilter.any(), Tags.of("customer", "123"))
).toList();

EventReference lastRef = events.getLast().reference();
// Use this reference for optimistic locking
```

## Appending Multiple Events to a Stream

Multiple events can be appended in a single atomic operation by passing a list:

```java
stream.append(
    AppendCriteria.none(),
    List.of(
        Event.of(new CustomerRegistered("John"), Tags.of("customer", "123")),
        Event.of(new CustomerAddressChanged("Main St"), Tags.of("customer", "123")),
        Event.of(new CustomerEmailChanged("john@example.com"), Tags.of("customer", "123"))
    )
);
```

All events in the list are appended atomically: either all succeed or all fail. Each event receives a consecutive position number within the stream.

**Important**: When using optimistic locking with batch appends, the `AppendCriteria` check is performed once before appending any events. If the check passes, all events are appended together.

## Optimistic Locking

The EventStore implements optimistic locking through the DCB pattern using `AppendCriteria`. This ensures that business decisions based on historical facts remain valid at the moment of appending new events.

### How It Works

1. Query relevant events using an `EventQuery`
2. Make a business decision based on those facts
3. Note the reference of the last relevant event
4. Append new events with `AppendCriteria` containing the same query and last reference
5. If new events matching the query exist after the reference, `OptimisticLockingException` is thrown

```java
// Step 1: Query relevant facts
EventQuery relevantQuery = EventQuery.forEvents(
    EventTypesFilter.any(),
    Tags.of("customer", "123")
);
List<Event<CustomerEvent>> relevantEvents = stream.query(relevantQuery).toList();

// Step 2 & 3: Make decision and note last reference
EventReference lastRef = relevantEvents.getLast().reference();

// Step 4 & 5: Conditional append
try {
    stream.append(
        AppendCriteria.of(relevantQuery, Optional.of(lastRef)),
        Event.of(new CustomerNameChanged("Jane"), Tags.of("customer", "123"))
    );
} catch (OptimisticLockingException e) {
    // New relevant facts emerged - retry with updated information
}
```

### First Append (Empty Stream)

When appending to an empty stream or when no previous relevant events exist, not EventReference can be obtained and passed:

```java
stream.append(
    AppendCriteria.of(
        EventQuery.forEvents(EventTypesFilter.any(), Tags.of("customer", "123")),
        Optional.empty()  // No last reference expected
    ),
    Event.of(new CustomerRegistered("John"), Tags.of("customer", "123"))
);
```

## Idempotency

Idempotency ensures that duplicate command submissions don't create duplicate events.
When handling for example incoming REST calls or asynchronous messages (JMS, Kafka, ...), it could happen that your correctly process and append the information, but that you're not able to acknowledge proper processing to the client or messaging system due to a system or connection failure.
In that case, it is to be expected that the client assumes processing hasn't happened yet, and that it resubmits the same information.  Idempotency in your system then allows to detect and silently ignore the duplicate processing, while confirming (again) to the client that reception and processing has happened correctly.

### Idempotent append()

The EventStore provides built-in idempotency support through idempotency keys. When appending an event with an idempotency key, if an event with the same key already exists, the append operation is silently ignored and returns an empty list.

```java
String requestId = "req-2025-01-15-abc123";

// First submission - succeeds and returns the appended event
List<Event<CustomerEvent>> events = stream.append(
    AppendCriteria.none(),
    Event.of(new CustomerRegistered("John"), Tags.none())
        .withIdempotencyKey(requestId)
);
assertEquals(1, events.size());

// Duplicate submission - silently ignored, returns empty list
events = stream.append(
    AppendCriteria.none(),
    Event.of(new CustomerRegistered("John"), Tags.none())
        .withIdempotencyKey(requestId)
);
assertEquals(0, events.size());
```

**Important**: Idempotency keys can only be used when appending a single event. When appending multiple events in a batch, none of them may have an idempotency key.

### DCB-style Idempotency

An alternative way to implement idempotency is by tagging events with an idempotency tag and using optimistic locking to prevent duplicate appends.

#### Implementation Pattern

```java
public void appendIdempotently(
        EventStream<CustomerEvent> stream,
        CustomerEvent eventData,
        String idempotencyKey) {

    // Add idempotency tag to the event
    EphemeralEvent<CustomerEvent> event = Event.of(
        eventData,
        Tags.of("idempotency", idempotencyKey)
    );

    // Define criteria: no event with this idempotency key should exist
    AppendCriteria criteria = AppendCriteria.of(
        EventQuery.forEvents(
            EventTypesFilter.any(),
            Tags.of("idempotency", idempotencyKey)
        ),
        Optional.empty()  // Expect no prior event with this key
    );

    try {
        stream.append(criteria, event);
    } catch (OptimisticLockingException e) {
        // Event with this idempotency key already exists
        // Safe to ignore - this is a duplicate submission
        System.out.println("Event already processed: " + idempotencyKey);
    }
}
```

#### Usage Example

```java
String requestId = "req-2024-01-15-abc123";

// First submission - succeeds
appendIdempotently(
    stream,
    new CustomerRegistered("John"),
    requestId
);

// Duplicate submission - OptimisticLockingException thrown and ignored
appendIdempotently(
    stream,
    new CustomerRegistered("John"),
    requestId
);
```

The `OptimisticLockingException` indicates the event was already appended previously, making the operation idempotent. The exception can be safely caught and ignored, as it signals successful deduplication rather than an error condition.
