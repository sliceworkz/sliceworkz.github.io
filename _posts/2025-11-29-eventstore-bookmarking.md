---
layout: post
toc: true
title: Bookmarking
description: Reader Bookmarks on an EventStream
date: 2025-11-29 06:00:00
categories: [Eventstore Documentation,Eventstore API]
tags: [bookmark]
---
# Stream Readers and Bookmarking

This guide covers how to track event processing progress using bookmarks, enabling resumable and eventually consistent event processing.

## Stream Readers and Bookmarking

In event sourcing, it's crucial to persist the point up to which a stream has been processed by a reader. This enables:

- **Resumable processing**: Restart processing from the last checkpoint after crashes or deployments
- **Eventually consistent read models**: Build derived views that catch up independently
- **Processing guarantees**: Ensure events are processed at-least-once
- **Progress monitoring**: Track how far behind real-time each reader is

Eventual consistency allows readers and processors to move independently from the write side. A new read model that still has to catch up with history can process events at its own pace:

```java
// New projection starting from scratch
CustomerSummary projection = new CustomerSummary("123");

// Process all historical events
Projector.from(stream).towards(projection).build().run();

// Now caught up with history
```

Bookmarks enable this independence by marking the last successfully processed event.

## Placing a Bookmark

The `placeBookmark()` method registers the `EventReference` of the last processed event for a named reader:

```java
stream.placeBookmark(
    "customer-summary-builder",        // Reader identifier
    event.reference(),                  // Last processed event
    Tags.of("status", "active")        // Metadata tags
);
```

**Reader Identifier**: A unique string identifying the processor (e.g., "order-projection", "email-sender-v2"). This is the key for the bookmark.

**Event Reference**: The reference of the last event successfully processed by this reader.

**Tags**: Optional metadata stored with the bookmark for observability. Common uses:
- Processing status: `Tags.of("status", "active")`
- Timestamp: `Tags.of("updated-at", Instant.now().toString())`
- Version: `Tags.of("version", "2.0")`
- Instance ID: `Tags.of("instance", hostname)`

Tags are stored only for monitoring and debugging—they don't affect bookmark retrieval.

**Creating vs. Updating**: There's no distinction. Calling `placeBookmark()` with the same reader ID creates the bookmark on first call and updates it on subsequent calls. The reader ID is the key.

```java
// First call - creates bookmark
stream.placeBookmark("my-reader", ref1, Tags.none());

// Second call - updates bookmark for same reader
stream.placeBookmark("my-reader", ref2, Tags.none());
```

## Retrieving a Bookmark

The `getBookmark()` method retrieves the last bookmarked position for a reader:

```java
Optional<EventReference> bookmark = stream.getBookmark("customer-summary-builder");

bookmark.ifPresentOrElse(
    ref -> System.out.println("Resume from position: " + ref.position()),
    () -> System.out.println("No bookmark - start from beginning")
);
```

Returns `Optional.empty()` if no bookmark exists for that reader (first run scenario).

## Retrieving All Bookmarks

`getBookmarks()` returns every bookmark on the stream, with the metadata that `getBookmark()` does not carry:

```java
public record Bookmark (
    String reader,             // the reader identifier
    EventReference reference,  // the position it reached
    Tags tags,                 // the tags supplied when it was placed
    Instant updatedAt          // when it was last placed
) { }
```

```java
List<Bookmark> bookmarks = stream.getBookmarks();

bookmarks.forEach(b -> System.out.printf(
    "%-30s position %-8d updated %s %s%n",
    b.reader(), b.reference().position(), b.updatedAt(), b.tags()));
```

This is the API behind an operational view of a system's readers. Where `getBookmark(reader)` answers "where is *this* processor", `getBookmarks()` answers "where is everything, and which of it has stopped moving":

```java
// which readers have fallen behind the head of the stream?
EventReference head = stream.query(EventQuery.matchAll().backwards().limit(1))
                            .findFirst().map(Event::reference).orElse(null);

stream.getBookmarks().stream()
    .filter(b -> head != null && b.reference().happenedBefore(head))
    .forEach(b -> LOGGER.warn("reader {} is behind, last moved at {}", b.reader(), b.updatedAt()));

// which readers have not moved in an hour?
Instant cutoff = Instant.now().minus(Duration.ofHours(1));
stream.getBookmarks().stream()
    .filter(b -> b.updatedAt().isBefore(cutoff))
    .forEach(b -> alertOnStalledReader(b.reader(), b.updatedAt()));
```

`updatedAt` is stamped at placement time, and `tags` are exactly the tags passed to `placeBookmark(...)` — which is what makes them worth populating with a hostname, an instance id or an application version. A reader that has stopped moving is much easier to chase down when its bookmark says which process last touched it.

Bookmarks placed before the metadata was recorded read back with `Tags.none()` and an epoch `updatedAt`, so a filter on `updatedAt` should tolerate that rather than treat it as a stalled reader.

## Removing a Bookmark

`removeBookmark(reader)` deletes a reader's bookmark and returns the reference it held, or `Optional.empty()` if there was none:

```java
Optional<EventReference> removed = stream.removeBookmark("customer-analytics");
```

The usual reason to do this is a **projection rebuild**: dropping the read model and the bookmark together makes the next run replay from the beginning of the stream.

```java
readModel.truncate();
stream.removeBookmark("customer-analytics");
projector.run();                    // replays from the start
```

## Typical Usage Scenario

A complete example showing bookmark retrieval, event processing with append listeners, and periodic bookmark updates:

```java
EventStream<CustomerEvent> stream = eventstore.getEventStream(
    EventStreamId.forContext("customer").anyPurpose(),
    CustomerEvent.class
);

String readerName = "customer-analytics";

// Retrieve last bookmark
Optional<EventReference> bookmark = stream.getBookmark(readerName);
EventReference startAfter = bookmark.orElse(null);

if (startAfter == null) {
    System.out.println("No bookmark found - processing from beginning");
} else {
    System.out.println("Resuming from position: " + startAfter.position());
}

// Register listener for new events appended to stream
stream.subscribe((EventReference atLeastUntil) -> {
    // Process new events since last bookmark
    stream.query(EventQuery.matchAll(), startAfter, Limit.to(100))
        .forEach(event -> {
            processEvent(event);

            // Update bookmark after processing
            stream.placeBookmark(
                readerName,
                event.reference(),
                Tags.of("processed-at", Instant.now().toString())
            );
        });
    return null;
});

// Initial catch-up: process all existing events
stream.query(EventQuery.matchAll(), startAfter)
    .forEach(event -> {
        processEvent(event);
        stream.placeBookmark(readerName, event.reference(), Tags.none());
    });
```

When the application starts:
- **No bookmark exists**: Process from the beginning of the stream
- **Bookmark exists**: Resume from the bookmarked position

The append listener ensures new events are processed without delay.

> This simple/naive implementation does one write operation (placeBookmark) per single event that is processed, which is far from ideal.  
The goal here was to show the basic setup, read further for ways to optimize this.
{: .prompt-warning }


## Typical Usage Scenario with Batching

A single reader instance works through an event stream, placing bookmarks as savepoints:

```java
String readerName = "order-fulfillment";
EventStream<OrderEvent> stream = eventstore.getEventStream(
    EventStreamId.forContext("orders").anyPurpose(),
    OrderEvent.class
);

// Retrieve bookmark
Optional<EventReference> lastProcessed = stream.getBookmark(readerName);

// Query events after bookmark
EventQuery query = EventQuery.matchAll();
EventReference after = lastProcessed.orElse(null);
int batchSize = 500;

while (true) {
    List<Event<OrderEvent>> batch = stream.query(
        query,
        after,
        Limit.to(batchSize)
    ).toList();

    if (batch.isEmpty()) {
        break; // Caught up
    }

    // Process batch
    batch.forEach(event -> processEvent(event));

    // Update bookmark after batch
    EventReference lastInBatch = batch.getLast().reference();
    stream.placeBookmark(
        readerName,
        lastInBatch,
        Tags.of("batch-size", String.valueOf(batch.size()))
    );

    after = lastInBatch;
}
```

**Restart Behavior**: If the process crashes and restarts, the bookmark is a persisted pointer indicating where to resume. The reader queries events after the bookmark and continues processing.

**Integration with Query Parameters**: The bookmark works naturally with the `after` parameter of event queries. Simply pass the bookmarked reference as the `after` parameter.

**Multi-Server Failover**: In a multi-server setup with failover, another logical instance of the same reader can take over by retrieving the same bookmark:

```java
// Server 1 goes down while processing
// Bookmark persisted at position 1000

// Server 2 takes over
Optional<EventReference> bookmark = stream.getBookmark("order-fulfillment");
// Resumes from position 1000
```

Multiple instances can't process the same reader concurrently (sequential processing), but failover enables high availability.

**Idempotent Processing with Larger Batches**: When combined with idempotent event handling, bookmarks can be updated less frequently for better performance:

```java
int batchSize = 1000;
int bookmarkEvery = 100; // Update bookmark every 100 events
int processedCount = 0;

stream.query(query, after, Limit.to(batchSize)).forEach(event -> {
    processEventIdempotently(event); // Idempotent handler
    processedCount++;

    if (processedCount % bookmarkEvery == 0) {
        stream.placeBookmark(readerName, event.reference(), Tags.none());
    }
});

// Place final bookmark
if (processedCount > 0) {
    stream.placeBookmark(readerName, lastEventRef, Tags.none());
}
```

If processing fails mid-batch, idempotent handlers allow re-processing without side effects.


> Idempotency can be as simple as storing the Event position (available in the EventReference) along with the read model, and to ignore handling of any events that carry a position that is not higher than the one already stored.
{: .prompt-info }

## Multiple Reader Instances

Event streams are typically processed **sequentially** because events in a stream have causal relationships and ordering requirements. Parallelizing their processing toward a useful result is not trivial.

**Exceptions**: Scenarios that can be parallelized include:
- **MapReduce algorithms**: Independent aggregations that can be combined
- **CRDTs (Conflict-free Replicated Data Types)**: Structures designed for commutative operations

For most use cases, a **single reader instance** works its way sequentially through all events in a stream.

However, **multiple independent read models** can be built in parallel:

```java
// Three different readers processing the same stream independently

// Reader 1: Customer analytics
stream.placeBookmark("customer-analytics", ref1, Tags.none());

// Reader 2: Email notification sender
stream.placeBookmark("email-sender", ref2, Tags.none());

// Reader 3: Reporting dashboard
stream.placeBookmark("dashboard-updater", ref3, Tags.none());
```

Each reader:
- Processes events at its own pace
- Maintains its own bookmark
- Can lag behind or catch up independently
- Builds a different view or performs different actions

These are the "readers" placing their bookmarks. They operate independently on the same event stream.

**Key Insight**: While a single reader must process events sequentially, many different readers can process the same stream concurrently, each maintaining their own bookmark.

## Listening for Bookmark Updates - Monitoring Reader Progress

Register a listener to receive real-time notifications when bookmarks are updated:

```java
stream.subscribe((String reader, EventReference processedUntil) -> {
    System.out.println(
        reader + " processed up to position " + processedUntil.position()
    );

    // Record metrics
    metricsService.recordReaderPosition(reader, processedUntil.position());
});
```

The listener is invoked asynchronously (eventually consistent) whenever any reader places or updates a bookmark.

Some use Cases for Bookmark Listeners:

- **Observability** - verify processors are still running and Eventual Consistency is keeping up with events being appended
- **Monitoring delays in processing**
- **Waiting for a Reader to process up to a certain point**
- **Coordinating Multiple Readers**
- etc ...

Bookmark listeners enable powerful monitoring and coordination patterns for eventually consistent event processing systems. For a point-in-time view of every reader rather than a notification per change, use `getBookmarks()` as shown above.