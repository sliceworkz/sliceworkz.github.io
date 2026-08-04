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

EventStore provides two listener interfaces, both eventually consistent:

**`EventStreamEventuallyConsistentAppendListener`** notifies you when new events are written to a stream.

**`EventStreamEventuallyConsistentBookmarkListener`** notifies you when readers update their processing position — useful for monitoring progress in distributed event processing pipelines.

Both are subscribed through the same `subscribe(...)` method on an `EventStream`, and both are functional interfaces, so a lambda is enough.

## Why Everything Is Eventually Consistent

**No listener runs in a transaction, and none can veto an append.** `EventStorage.append` commits before it returns — on PostgreSQL by issuing the `COMMIT` inside it, in memory by having the events in the log — so by the time anything is notified, the events are durable and every other reader can already see them. A notification is an announcement, never a vote.

That is not a limitation to work around, because the case a synchronous listener would serve is already covered: **`append()` returns the events it wrote**, typed, with their assigned references — the same list a caller would otherwise have queried back.

```java
List<Event<CustomerEvent>> written = stream.append(
    AppendCriteria.none(),
    Event.of(new CustomerRegistered("123", "Alice"), Tags.of("customer", "123")));

// read-your-own-writes: no subscription involved
cache.put("123", "Alice");
EventReference justWritten = written.getLast().reference();
```

To react to your own append on the appending thread, write the code after the call. Subscriptions exist for what *other* threads, processes and nodes append.

## Eventually Consistent Append Listeners

An append listener receives an `EventReference` pointing to **at least** the last appended event — not the event data itself, and not one notification per event. This lightweight model lets you query exactly the events you care about, and it collapses a burst of appends into a single wake-up.

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
            lastProcessed.set(event.reference());
        });

    // Update bookmark for resumability
    stream.placeBookmark("order-processor", atLeastUntil, Tags.none());
    return lastProcessed.get();
});
```

> Multiple nodes running the same listener code will each receive notifications independently!  Make sure to put a coordination mechanism in place to avoid that every node will try to update your (shared) readmodel.
{: .prompt-warning }

### What the Return Value Means

The listener returns the reference it has now reached. The store keeps delivering until the listener has caught up with the target it was notified about, so this return value is how it learns that it has.

**Returning `null`, or a reference behind the target, both mean "caught up".** Nothing is lost by that: the next append carries a later reference, which is after this one and so still delivered.

That matters because the ordinary case returns null. `Projector.eventsAppended` returns `run().lastEventReference()`, which is null whenever the run's query matched no events — so any subscribed projector whose event type has not occurred yet returns null on every unrelated append to its stream.

### A Listener's Failure Is Contained

Each subscriber's exception is caught, logged at ERROR, and the next subscriber still gets the notification. Three consequences to design around:

- **The appending caller is never told.** The events are already committed; a listener cannot un-append them.
- **The other subscribers are unaffected.** A throwing listener does not starve the ones behind it, and both keep receiving notifications afterwards.
- **Nothing replays what a failing listener missed.** It is notified again on the next append; the notification it failed on is gone.

**A listener that must not lose progress belongs behind a `Projector` reading from a bookmark**, which resumes from a persisted position rather than from whatever the last notification happened to be. See [Projecting Events](/posts/eventstore-projecting-events/#bookmarking-for-process-restart-and-progress-tracking).

### Subscriptions Have a Lifecycle

Subscribing registers the stream with the storage, which then holds it **strongly** until the stream is closed. That is what keeps live updates working after the caller drops the variable — and it means nothing releases the subscription on your behalf:

```java
try ( EventStream<OrderEvent> stream = eventStore.getEventStream(streamId, OrderEvent.class) ) {
    stream.subscribe(reference -> { updateReadModel(); return reference; });
    // ... application runs ...
}   // subscription ended, registration released
```

A stream you only query and append through registers nothing and needs no cleanup at all. See [Lifecycle and Shutdown](/posts/eventstore-lifecycle/) for the full contract.

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

Bookmark listeners get the same failure containment as append listeners: an exception is logged and the next subscriber still runs.

**Why bookmark notifications are asynchronous too.** Bookmarks are placed after processing each event or batch, potentially hundreds or thousands of times per second. Blocking those operations with synchronous notifications would severely degrade throughput and couple readers tightly to whoever is watching them.

Bookmark placement is also typically done by autonomous background processors that shouldn't be coupled to the rest of the system, and in many deployments wouldn't even run in the same process, making a synchronous notification impossible in the first place. The asynchronous model lets readers work independently while still providing visibility into their progress. If you need the current position on demand rather than on change, query it directly with `getBookmark(reader)` — or `getBookmarks()` for all readers at once.

## Delivery Guarantees in Practice

**Notifications are a hint, not a feed.** They tell you that events exist *at least* up to a reference. By the time you query, more may have arrived — which is fine, because you query for what you need rather than consuming what you were handed.

**A burst collapses.** Only the newest reference in a burst is delivered; earlier ones are dropped because reaching the newest implies having passed them. This is why a listener must query rather than assume one notification equals one event.

**On PostgreSQL, one notification is emitted per stream per statement**, not per row — so a 1000-event append wakes a subscriber once per stream it touched. See [PostgreSQL EventStorage](/posts/eventstore-configuring-postgresql-storage/#append-notifications).

**Across nodes, delivery depends on LISTEN/NOTIFY being established.** If the monitoring connection is unavailable, notifications stop while queries keep working — which is exactly the silent failure the `sliceworkz.eventstore.notifications.up` gauge exists to make visible. See [Eventstore Observability](/posts/eventstore-observability-micrometer-prometheus-grafana/).
