---
layout: post
toc: true
title: Querying Events
description: Querying for Domain Events
date: 2025-11-29 03:00:00
categories: [Eventstore Documentation,Eventstore API]
tags: [events,query,tags]
---

This guide covers the various ways to query events from the EventStore, including filtering, pagination, backward queries, temporal queries, and cross-stream querying.

## EventQuery Concept

An `EventQuery` is the fundamental mechanism for selecting events from the EventStore. It wraps an `EventFilter` (which defines matching criteria based on event types, tags, and an optional temporal boundary) together with traversal semantics: a direction (forward or backward) and an optional limit.

**An event matches an EventQuery if:**
1. The event's type is **any** of the types allowed by the query (OR condition)
2. **All** tags specified by the query are present on the event (AND condition)

Events may have additional tags beyond those specified in the query—the query only requires that the specified tags are present.

```java
// Query for CustomerRegistered OR CustomerUpdated events with region=EU tag
EventQuery query = EventQuery.forEvents(
    EventTypesFilter.of(CustomerRegistered.class, CustomerUpdated.class),
    Tags.of("region", "EU")
);

// This matches:
Event.of(new CustomerRegistered("John"), Tags.of("region", "EU")) // ✓ Type matches, has required tag
Event.of(new CustomerUpdated("Jane"), Tags.of("region", "EU", "premium", "true")) // ✓ Type matches, has required tag (plus extra)

// This does NOT match:
Event.of(new CustomerRegistered("Bob"), Tags.of("region", "US")) // ✗ Type matches, but wrong tag value
Event.of(new CustomerChurned("Alice"), Tags.of("region", "EU")) // ✗ Has required tag, but wrong type
Event.of(new CustomerUpdated("Dave"), Tags.none()) // ✗ Type matches, but missing required tag
```

The same `EventQuery` object can be used both for database-level filtering and in-process filtering, as explained in the next section.

## In-Database vs In-Process Querying

The `EventQuery` object is versatile—it can be used to filter events at two different levels:

1. **Database-level filtering**: Pass the query to the event stream's `query()` method
2. **In-process filtering**: Use the query's `matches()` method in your Java code

### Database-Level Filtering (Recommended)

When you pass an `EventQuery` to the event stream, the filtering happens in the event storage (database):

```java
EventStream<CustomerEvent> stream = eventstore.getEventStream(streamId, CustomerEvent.class);

EventQuery query = EventQuery.forEvents(
    EventTypesFilter.of(CustomerRegistered.class),
    Tags.of("region", "EU")
);

// Query is executed in the database
Stream<Event<CustomerEvent>> events = stream.query(query);
events.forEach(event -> processEvent(event));
```

**Advantages:**
- Only matching events are read from the database
- Efficient—leverages database indexes and query optimization
- Minimal memory usage and network transfer
- Recommended for most use cases

### In-Process Filtering

The same `EventQuery` object can filter events in your Java application:

```java
EventStream<CustomerEvent> stream = eventstore.getEventStream(streamId, CustomerEvent.class);

EventQuery query = EventQuery.forEvents(
    EventTypesFilter.of(CustomerRegistered.class),
    Tags.of("region", "EU")
);

// Query all events from database, filter in Java
Stream<Event<CustomerEvent>> allEvents = stream.query(EventQuery.matchAll());
Stream<Event<CustomerEvent>> filtered = allEvents.filter(query::matches);
filtered.forEach(event -> processEvent(event));
```

**Disadvantages:**
- All events are read from the database
- Filtering happens in application memory
- Poor performance with large event streams
- Higher memory usage and network transfer

**Important:** While these two approaches are functionally equivalent (they return the same events), the in-process approach suffers from significant performance issues because all events must be retrieved from the database before filtering.

### Hybrid Approach: Coarse Database Filtering + Fine-Grained In-Process Filtering

Sometimes it's beneficial to retrieve a limited set of events from the database and then apply multiple fine-grained filters in Java. This allows you to reuse query results for multiple objectives without running multiple similar database queries:

```java
EventStream<CustomerEvent> stream = eventstore.getEventStream(streamId, CustomerEvent.class);

// Coarse filter: Get all customer events for EU region
EventQuery broadQuery = EventQuery.forEvents(
    EventTypesFilter.any(),  // All event types
    Tags.of("region", "EU")
);

List<Event<CustomerEvent>> euEvents = stream.query(broadQuery).toList();

// Now apply multiple fine-grained filters in-process
EventQuery registrationsQuery = EventQuery.forEvents(
    EventTypesFilter.of(CustomerRegistered.class),
    Tags.of("region", "EU")
);

EventQuery premiumQuery = EventQuery.forEvents(
    EventTypesFilter.any(),
    Tags.of("region", "EU", "premium", "true")
);

EventQuery churnQuery = EventQuery.forEvents(
    EventTypesFilter.of(CustomerChurned.class),
    Tags.of("region", "EU")
);

// Reuse the same event list with different filters
List<Event<CustomerEvent>> registrations = euEvents.stream()
    .filter(e -> registrationsQuery.matches(e))
    .toList();

List<Event<CustomerEvent>> premiumCustomers = euEvents.stream()
    .filter(e -> premiumQuery.matches(e))
    .toList();

List<Event<CustomerEvent>> churned = euEvents.stream()
    .filter(e -> churnQuery.matches(e))
    .toList();

System.out.println("EU Registrations: " + registrations.size());
System.out.println("EU Premium: " + premiumCustomers.size());
System.out.println("EU Churned: " + churned.size());
```

**When to use this approach:**
- You need to apply multiple related queries to the same dataset
- The coarse query retrieves a manageable number of events
- You want to avoid multiple database round-trips
- Fine-grained filtering logic is complex or changes frequently

**When to avoid:**
- The coarse query returns too many events (memory concerns)
- You only need one specific filter (use database-level filtering instead)

This hybrid approach can balance efficiency and flexibility by retrieving a relevant subset once and filtering it multiple ways in memory, 
as long as you make sure the number of retrieved events is low enough to do so, or if you query in batches (see further)

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

### `query()` Returns a Stream, but It Is Already in Memory

This is the single most important thing to know about querying. `query()` hands back a `java.util.stream.Stream`, but **storage has finished reading by the time it returns**: the whole result set is fetched and the stream iterates a list.

Three consequences follow directly:

- **`findFirst()`, `.limit(10)` and `takeWhile` on the returned stream discard work already done.** They are cheap, but they save nothing at the database.
- **An unbounded query against a large stream is an `OutOfMemoryError`, not a slow stream.** There is no back-pressure to arrive at. Bound the read with `EventQuery.limit(n)`, which is the limit storage is actually given.
- **Nothing needs closing.** No database resource is held open behind the stream, so it is safe to abandon half-consumed. (`EventSource.close()` is about subscriptions, not queries.)

```java
// reads everything matching into heap before returning
List<Event<CustomerEvent>> all = stream.query(EventQuery.matchAll()).toList();

// reads 500 stored events
List<Event<CustomerEvent>> page = stream.query(EventQuery.matchAll().limit(500)).toList();
```

**A full replay is a loop, not one unbounded query.** `Projector` already reads in batches of 500, carrying a cursor between them, and is the right tool for a stream of unknown size. By hand, page with `query(q.limit(n), cursor)` and advance the cursor to the last reference of each page — the pattern shown under [Querying in batches](#querying-in-batches). The unlimited path exists for callers who know their result set is small; it is not a way to process a large stream incrementally.

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

The cursor parameter is a technical optimization that tells the store where to start scanning. For forward queries it acts as an "after" cursor; for backward queries it acts as a "before" cursor. It doesn't affect which events match the query, only where the scan begins.

### What a Limit Actually Counts

**`limit(n)` means "read n stored events", and it is pushed into the storage query** — a SQL `LIMIT` on PostgreSQL, a short-circuiting `Stream.limit` in memory. It is not applied to the result after the fact, which is exactly what makes it bound memory as well as output.

A cursor does not change this. `query(q.limit(500), cursor)` reads 500, the same as `query(q)` would with the limit set on `q`. To read to the end of a stream deliberately, pass `Limit.none()` to the three-argument overload.

**Without upcasting, n stored events are n events back. With it, they are not.** An `@Upcast` method may turn one stored event into several or into none, and the limit is spent before it runs:

```java
// over an event that upcasts into two, this returns TWO events —
// having read exactly one stored event
stream.query(EventQuery.matchAll().limit(1)).toList();

// over an event that upcasts into none, it returns ZERO —
// also having read exactly one stored event
```

Trimming the surplus would return a fragment of a stored event and leave a cursor pointing into its middle, so the store does not do it. Where you need exactly n events, apply `.limit(n)` to the returned `Stream` — cheap, since those events are already in memory:

```java
List<Event<CustomerEvent>> exactlyTen =
    stream.query(EventQuery.matchAll().limit(10)).limit(10).toList();
```

`Projector` counts stored events for the same reason.

## Querying backwards

Backward queries return events in reverse chronological order (newest first). The direction is set on the `EventQuery` itself using `backwards()`, and a limit can be set using `limit()`:

```java
// Get the last 10 events
Stream<Event<CustomerEvent>> recentEvents = stream.query(
    EventQuery.matchAll().backwards().limit(10)
);

// Find the last CustomerRegistered event
Optional<Event<CustomerEvent>> lastRegistration = stream.query(
    EventQuery.forEvents(
        EventTypesFilter.of(CustomerRegistered.class),
        Tags.none()
    ).backwards().limit(1)
).findFirst();
```

### Backward Pagination

```java
EventReference beforeRef = null;
EventQuery backwardsQuery = EventQuery.matchAll().backwards();
while (true) {
    Stream<Event<CustomerEvent>> batch = stream.query(
        backwardsQuery,
        beforeRef,
        Limit.to(100)
    );

    List<Event<CustomerEvent>> events = batch.toList();
    if (events.isEmpty()) {
        break;
    }

    // Process events (already in reverse order)
    events.forEach(event -> processEvent(event));

    // Update cursor to continue before the oldest event in this batch
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

### How the `until` Boundary Behaves

Three properties are worth being precise about:

**It is inclusive.** The event named by the reference is returned.

**It is direction-independent.** `until` is compared over the total `(tx, position, index)` order, not against the direction of travel, so `.backwards()` returns exactly the same events as forward — just newest first:

```java
EventQuery upTo = EventQuery.forEvents(EventTypesFilter.any(), Tags.of("customer", "123"))
                            .until(checkpoint);

List<Event<CustomerEvent>> forward  = stream.query(upTo).toList();
List<Event<CustomerEvent>> backward = stream.query(upTo.backwards()).toList();
// same events, opposite order
```

**It is part of the filter, so it also bounds a consistency boundary.** An `AppendCriteria` built from a query carrying an `until` will not raise `OptimisticLockingException` for an event past that boundary — such an event is, by construction, not a new relevant fact for a decision taken as of that moment.

Note that the ordering compared is the tuple, not the position alone. Positions and transactions are assigned independently, so an event can hold a lower position and a higher transaction than one that committed before it. Comparing positions would silently drop such events.

## Querying by Event ID

A specific event can be retrieved directly by its `EventId`:

```java
EventId eventId = EventId.of("550e8400-e29b-41d4-a716-446655440000");

List<Event<CustomerEvent>> events = stream.getEventById(eventId);

if (!events.isEmpty()) {
    System.out.println("Found event: " + events.getFirst().data());
    System.out.println("Position: " + events.getFirst().reference().position());
}
```

`getEventById` returns a `List` rather than an `Optional`. For regular (non-upcasted) events the list contains exactly one element. When a stored event is upcasted into multiple sub-events via multi-event upcasting, the list contains all sub-events sharing that ID — each with the same id/position/tx but a distinct index.

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
- When a legacy event is upcasted into multiple current events (multi-event upcasting), all sub-events appear in query results in their correct position with distinct index values
- When a legacy event is upcasted to zero events (filtering), it is silently skipped

## Complex Event Queries

For advanced scenarios, you can combine multiple query criteria using the `combineWith()` method. This creates a UNION of queries, allowing you to retrieve events that match any of several different patterns.

### Understanding Query Matching Semantics

Event queries follow specific matching rules that combine AND and OR logic:

**Within a single query item:**
- **Event Types**: The event must match **ANY** of the specified types (OR condition)
- **Tags**: The event must contain **ALL** specified tags (AND condition)

**Across multiple query items:**
- If **ANY** item matches, the event matches the overall query (OR condition)

This gives you powerful flexibility to express complex selection criteria.

### Basic Query Combination

Combine two queries to match events that satisfy either query:

```java
// Query 1: All CustomerRegistered events
EventQuery newCustomers = EventQuery.forEvents(
    EventTypesFilter.of(CustomerRegistered.class),
    Tags.none()
);

// Query 2: All events for VIP customers
EventQuery vipActivity = EventQuery.forEvents(
    EventTypesFilter.any(),
    Tags.of("customerType", "VIP")
);

// Combined: CustomerRegistered events OR any VIP customer events
EventQuery combined = newCustomers.combineWith(vipActivity);

Stream<Event<CustomerEvent>> events = stream.query(combined);
```

The combined query will return:
- All `CustomerRegistered` events (regardless of tags)
- All events (any type) with the tag `customerType=VIP`

No duplicates are returned (Events matching multiple items in the complex query are returned once.
As always, events are returend in order of their position in the stream.

### Combining Queries with Different Types and Tags

Create complex selection criteria by combining queries with different event types and tag requirements:

```java
// Events related to a specific student
EventQuery studentEvents = EventQuery.forEvents(
    EventTypesFilter.of(StudentRegistered.class, StudentSubscribedToCourse.class),
    Tags.of("student", "S123")
);

// Events related to a specific course
EventQuery courseEvents = EventQuery.forEvents(
    EventTypesFilter.of(CourseDefined.class, CourseCapacityUpdated.class, StudentSubscribedToCourse.class),
    Tags.of("course", "CS101")
);

// Combined: All events relevant to this student-course interaction
EventQuery relevantFacts = studentEvents.combineWith(courseEvents);
```

The combined query matches events where **any** of these conditions are true:
- Event is `StudentRegistered` OR `StudentSubscribedToCourse` AND has tag `student=S123`
- Event is `CourseDefined` OR `CourseCapacityUpdated` OR `StudentSubscribedToCourse` AND has tag `course=CS101`

Notice that `StudentSubscribedToCourse` events with **either** tag will be included.

### Query Combination Rules

When combining queries, certain rules apply:

**Compatible "until" references:**

```java
// Both queries without "until" - OK
EventQuery q1 = EventQuery.forEvents(EventTypesFilter.of(CustomerRegistered.class), Tags.none());
EventQuery q2 = EventQuery.forEvents(EventTypesFilter.of(OrderPlaced.class), Tags.none());
EventQuery combined = q1.combineWith(q2); // Success

// Both queries with same "until" - OK
EventReference checkpoint = EventReference.of(someId, 100L);
EventQuery q3 = EventQuery.forEvents(EventTypesFilter.of(CustomerRegistered.class), Tags.none())
    .until(checkpoint);
EventQuery q4 = EventQuery.forEvents(EventTypesFilter.of(OrderPlaced.class), Tags.none())
    .until(checkpoint);
EventQuery combinedHistorical = q3.combineWith(q4); // Success

// Different "until" references - ERROR
EventQuery q5 = EventQuery.forEvents(EventTypesFilter.of(CustomerRegistered.class), Tags.none())
    .until(EventReference.of(someId, 100L));
EventQuery q6 = EventQuery.forEvents(EventTypesFilter.of(OrderPlaced.class), Tags.none())
    .until(EventReference.of(otherId, 200L));
// q5.combineWith(q6) throws IllegalArgumentException
```

Both queries must have:
- No "until" reference, **or**
- The same "until" reference

Attempting to combine queries with different "until" references throws an `IllegalArgumentException`.

**Compatible direction:**

Both queries must have the same direction:

```java
// Same direction - OK
EventQuery q7 = EventQuery.forEvents(EventTypesFilter.of(CustomerRegistered.class), Tags.none())
    .backwards();
EventQuery q8 = EventQuery.forEvents(EventTypesFilter.of(OrderPlaced.class), Tags.none())
    .backwards();
EventQuery combinedBackward = q7.combineWith(q8); // Success

// Different direction - ERROR
EventQuery q9 = EventQuery.forEvents(EventTypesFilter.of(CustomerRegistered.class), Tags.none());
EventQuery q10 = EventQuery.forEvents(EventTypesFilter.of(OrderPlaced.class), Tags.none())
    .backwards();
// q9.combineWith(q10) throws IllegalArgumentException
```

**No limits at all:**

A query carrying a limit cannot be combined, even with a query carrying the same limit:

```java
EventQuery q11 = EventQuery.forEvents(EventTypesFilter.of(CustomerRegistered.class), Tags.none())
    .limit(10);
EventQuery q12 = EventQuery.forEvents(EventTypesFilter.of(OrderPlaced.class), Tags.none())
    .limit(10);
// q11.combineWith(q12) throws IllegalArgumentException — identical limits are still refused
```

The reason is that a shared limit over a union does not preserve what either query meant on its own. Two `backwards().limit(1)` queries each ask for "the most recent event matching *me*"; combined into `(A OR B) limit 1` they ask for "the most recent event matching either", which answers neither. Since there is no correct way to fold them, they are refused rather than quietly reinterpreted.

Combine the queries without limits and apply the limit afterwards, or run them separately.

### Practical Use Case: Dynamic Consistency Boundary

Query combination is particularly useful for Dynamic Consistency Boundaries where business decisions depend on multiple types of facts:

```java
public class SubscribeToCourseCommand {
    private final String studentId;
    private final String courseId;

    public EventQuery relevantFacts() {
        // Query for student-specific facts
        EventQuery studentQuery = EventQuery.forEvents(
            EventTypesFilter.of(StudentRegistered.class, StudentSubscribedToCourse.class),
            Tags.of("student", studentId)
        );

        // Query for course-specific facts
        EventQuery courseQuery = EventQuery.forEvents(
            EventTypesFilter.of(CourseDefined.class, CourseCapacityUpdated.class, StudentSubscribedToCourse.class),
            Tags.of("course", courseId)
        );

        // Combine to get all relevant facts for this business decision
        return studentQuery.combineWith(courseQuery);
    }

    public void execute(EventStream<LearningEvent> stream) {
        // Load current state based on relevant facts
        EventQuery query = relevantFacts();
        List<Event<LearningEvent>> facts = stream.query(query).toList();

        // Make business decision based on facts
        CourseAggregate course = buildCourseState(facts);
        if (course.hasCapacity()) {
            EventReference lastRelevantFact = facts.getLast().reference();

            // Append new event with optimistic locking
            stream.append(
                AppendCriteria.of(query, lastRelevantFact),
                Event.of(
                    new StudentSubscribedToCourse(studentId, courseId),
                    Tags.of("student", studentId, "course", courseId)
                )
            );
        }
    }
}
```

This pattern ensures that if **any** new relevant fact emerges (either about the student or the course) between reading facts and appending the new event, the append will fail with an `OptimisticLockingException`.

### Matching Examples

To clarify the matching semantics, consider these examples:

**Example 1: Simple combination**

```java
EventQuery q = EventQuery.forEvents(
    EventTypesFilter.of(CustomerRegistered.class),
    Tags.of("region", "EU")
).combineWith(
    EventQuery.forEvents(
        EventTypesFilter.of(OrderPlaced.class),
        Tags.of("priority", "high")
    )
);
```

This matches events where:
- Event type is `CustomerRegistered` AND has tag `region=EU`, **OR**
- Event type is `OrderPlaced` AND has tag `priority=high`

**Example 2: Multiple types and tags per item**

```java
EventQuery q = EventQuery.forEvents(
    EventTypesFilter.of(CustomerRegistered.class, CustomerUpdated.class),
    Tags.of("region", "EU", "verified", "true")
).combineWith(
    EventQuery.forEvents(
        EventTypesFilter.of(OrderPlaced.class, OrderShipped.class),
        Tags.of("priority", "high")
    )
);
```

This matches events where:
- Event type is `CustomerRegistered` OR `CustomerUpdated` AND has BOTH tags `region=EU` and `verified=true`, **OR**
- Event type is `OrderPlaced` OR `OrderShipped` AND has tag `priority=high`

**Example 3: Any type with specific tags**

```java
EventQuery q = EventQuery.forEvents(
    EventTypesFilter.any(),
    Tags.of("correlationId", "abc-123")
).combineWith(
    EventQuery.forEvents(
        EventTypesFilter.of(ErrorOccurred.class),
        Tags.none()
    )
);
```

This matches events where:
- ANY event type with tag `correlationId=abc-123`, **OR**
- Event type is `ErrorOccurred` (regardless of tags)

This pattern is useful for debugging: retrieve all events in a specific correlation chain plus any error events.

## Batch-Merging Many Queries

`combineWith` folds two queries you chose yourself. `EventQuery.merge(...)` solves the other problem: you have been handed *x* queries — one per projection, one per decision, one per subscriber — and want to run as few database queries as possible without changing what any of them means.

```java
List<EventQuery> queries = projections.stream().map(Projection::eventQuery).toList();

MergedEventQueries merged = EventQuery.merge(queries);

System.out.println(queries.size() + " queries folded into " + merged.mergedCount());
```

`merge` returns a `MergedEventQueries` that records **which merged query absorbed each original**, so results can be re-filtered per original after running the merged set:

```java
for ( EventQuery mergedQuery : merged.mergedQueries() ) {
    List<Event<CustomerEvent>> events = stream.query(mergedQuery).toList();

    // one database round-trip serves every original in this group
    for ( EventQuery original : merged.originalsFor(mergedQuery) ) {
        List<Event<CustomerEvent>> forThisOne =
            events.stream().filter(original::matches).toList();
        dispatch(original, forThisOne);
    }
}
```

`merged.mergedFor(original)` goes the other way, for a caller holding one original query and wanting to know which merged query covers it.

### The Merge Rules

**Unlimited queries are grouped by `(direction, until)`** and folded via `combineWith`. Forward and backward queries never merge with each other, and neither do queries with different `until` boundaries.

**Limited queries are always kept separate.** Each becomes its own merged query, for the reason described above: a shared limit over a union does not preserve per-query semantics.

**A match-all in a group dominates it.** If any query in a group is match-all, the merged query is match-all with that group's direction and until. This guarantees the merged query is a superset of every original, which is what keeps per-original re-filtering correct.

**Duplicate equal queries collapse.** Two identical queries share one entry in the mapping and route identically.

Passing a query to `mergedFor(...)` that was not part of the merge input — or to `originalsFor(...)` a query that is not one of the merged results — throws `IllegalArgumentException`.

### When This Is Worth It

The saving is one database round-trip per group, at the cost of an in-memory filter pass per original. That trade favours merging when many subscribers watch overlapping slices of the same stream, and not when a handful of queries are already disjoint. A useful sanity check is the ratio `merge` reports:

```java
MergedEventQueries merged = EventQuery.merge(queries);
if ( merged.mergedCount() == queries.size() ) {
    // nothing merged — just run them individually
}
```