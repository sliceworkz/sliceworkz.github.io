---
layout: post
toc: true
title: Eventstore Listeners
description: Notifying your application about Eventstore activity
date: 2025-11-29 06:00:00
categories: [Eventstore Documentation,Eventstore API]
tags: [listener]
---
## EventStore Listeners

Listeners enable your application to react to activity in the EventStore without constantly polling. They provide notifications when events are appended or when readers update their processing bookmarks. This reactive approach is essential for building responsive event-driven systems.

In single-instance deployments, listeners help decouple components—one part appends events while another reacts asynchronously. But listeners become truly powerful in multi-node deployments where several application instances share the same storage backend. When one node appends events, all subscribed nodes across the cluster receive notifications, enabling distributed coordination without additional messaging infrastructure.

EventStore provides two categories of listeners:

**Append Listeners** notify you when new events are written to a stream. Choose between:
- **Eventually Consistent Append Listeners**: Asynchronous, non-blocking notifications (recommended for most scenarios)
- **Consistent Append Listeners**: Synchronous, transactional notifications for immediate processing

**Bookmark Listeners** notify you when readers update their processing position, useful for monitoring progress in distributed event processing pipelines. Only eventually consistent bookmark listeners are supported.

### Consistency Trade-offs

**Eventually consistent listeners** execute asynchronously after the operation completes. They don't block the append or bookmark operation, making them ideal for high-throughput scenarios. There's a small delay between the operation and the notification, and exceptions in listener code won't affect the original operation. PostgreSQL's LISTEN/NOTIFY mechanism efficiently propagates notifications across all connected nodes with minimal latency.

**Consistent listeners** execute synchronously within the append transaction itself. They receive full typed domain events immediately and can update in-memory projections transactionally. However, they block the append operation, reduce throughput, and any exceptions can cause the append to fail. They're only suitable when immediate consistency within the same process is absolutely required.

**Recommendation**: Use eventually consistent listeners whenever possible. They scale better, isolate failures, and work seamlessly across distributed deployments. Reserve consistent listeners for rare cases where transactional guarantees within a single process are essential.

## Eventually Consistent Append Listeners

Eventually consistent append listeners receive asynchronous notifications after events are appended. They receive an `EventReference` pointing to at least the last appended event, not the full event data, and not every event appended triggers a separate notification. This lightweight notification model allows you to query only the events you care about.

These listeners are ideal for:
- Building eventually consistent read models and projections
- Triggering background jobs or workflows
- Notifying external systems of changes
- Coordinating distributed event processors

```java
// Create event stream
EventStore eventStore = PostgresEventStorage.newBuilder().buildStore();
EventStreamId streamId = EventStreamId.forContext("order");
EventStream<OrderEvent> stream = eventStore.getEventStream(streamId, OrderEvent.class);

// Track last processed position
AtomicReference<EventReference> lastProcessed = new AtomicReference<>();

// Subscribe to eventually consistent notifications
stream.subscribe((EventReference atLeastUntil) -> {
    // Query new events since last processed
    stream.query(EventQuery.matchAll(), lastProcessed.get())
        .forEach(event -> {
            System.out.println("Processing: " + event.type());
            updateReadModel(event);
        });

    // Update bookmark for resumability
    lastProcessed.set(atLeastUntil);
    stream.placeBookmark("order-processor", atLeastUntil, Tags.none());
});
```

> Multiple nodes running the same listener code will each receive notifications independently!  Make sure to put a coordination mechanism in place to avoid that every node will try to update your (shared) readmodel.
{: .prompt-warning }

## Consistent Append Listeners

Consistent append listeners receive synchronous notifications during the append transaction. Unlike eventually consistent listeners, they receive the full typed domain events immediately, as well ass all events appended.
No querying of the eventstore is thus required to process the information in the newly appended events.

These listeners are appropriate for very specific scenarios like updating an in-memory caches transactionally.

Use them sparingly—they block the append operation and reduce system throughput.

```java
EventStore eventStore = InMemoryEventStorage.newBuilder().buildStore();
EventStreamId streamId = EventStreamId.forContext("customer").withPurpose("123");
EventStream<CustomerEvent> stream = eventStore.getEventStream(streamId, CustomerEvent.class);

// In-memory cache updated transactionally
Map<String, String> cache = new ConcurrentHashMap<>();

stream.subscribe((List<Event<CustomerEvent>> events) -> {
    events.forEach(event -> {
        // Process typed events immediately
        switch (event.data()) {
            case CustomerRegistered(String id, String name) ->
                cache.put(id, name);
            case CustomerNameChanged(String id, String newName) ->
                cache.put(id, newName);
            case CustomerChurned(String id) ->
                cache.remove(id);
        }
    });
});

// Cache is updated synchronously before append completes
stream.append(AppendCriteria.none(),
    Event.of(new CustomerRegistered("123", "Alice"), Tags.none()));
```

Keep processing logic fast and simple.

## Eventually Consistent Bookmark Listeners

Bookmark listeners notify you when readers update their processing position by placing a bookmark. They're useful for monitoring distributed event processing systems, detecting lag, and coordinating multiple processors.

Use cases include:
- Monitoring progress across multiple readers
- Detecting stuck or slow processors
- Implementing health checks for processing pipelines
- Coordinating distributed event processors

```java
EventStore eventStore = PostgresEventStorage.newBuilder().buildStore();
EventStreamId streamId = EventStreamId.forContext("order");
EventStream<OrderEvent> stream = eventStore.getEventStream(streamId, OrderEvent.class);

// Monitor all readers
Map<String, EventReference> readerPositions = new ConcurrentHashMap<>();

stream.subscribe((String reader, EventReference processedUntil) -> {
    System.out.println("Reader '" + reader + "' processed up to: " +
        processedUntil.position());

    readerPositions.put(reader, processedUntil);

    // Detect processing lag
    long latestPosition = getLatestEventPosition(stream);
    long readerPosition = processedUntil.position();
    long lag = latestPosition - readerPosition;

    if (lag > 1000) {
        alertOnProcessingLag(reader, lag);
    }
});
```

**Why no consistent bookmark listeners?**

Consistent (synchronous) bookmark listeners would be problematic. Bookmarks are typically placed after processing each event or batch, potentially hundreds or thousands of times per second. Blocking these operations with synchronous notifications would severely degrade throughput and create tight coupling on readers.

Additionally, bookmark placement is often performed by autonomous background processors that shouldn't be coupled to the rest of the system, and in many deployment scenarios wouldn't even be deployed in the same process as the other components, making it impossible to synchronously notify them. The asynchronous, eventually consistent model allows readers to work independently while still providing visibility into their progress for monitoring purposes. If you need immediate awareness of bookmark updates, query the bookmark directly using `getBookmark(reader)`.