---
layout: post
toc: true
title: The Savepoint Pattern
description: Efficient projection initialization using savepoint events
date: 2026-02-14 01:00:00
categories: [Eventstore Documentation,Eventstore Application Scenarios]
tags: [projection,projector,read model,savepoint,initQuery]
---

# The Savepoint Pattern

This guide covers the savepoint pattern — a technique for avoiding full event replays when initializing projections, using the `initQuery()` capability introduced in EventStore 0.6.3.

## The Problem: Full Replays

Projections build read models by processing events sequentially. When a projection is initialized from scratch, it replays every event from the beginning of the stream:

```java
StockLevelProjection projection = new StockLevelProjection("WIDGET-42");
Projector.from(stream).towards(projection).build().run();
// Processes ALL events from the beginning of time
```

For streams with thousands or millions of events, this can be slow. Every time a new projection instance is created — after a restart, on a new server, or simply when handling a request — the entire history must be replayed.

Bookmarking solves this for long-lived projections that persist their position. But not every projection needs or wants that infrastructure. Some projections are short-lived, created on-the-fly to answer a specific question and then discarded. For these, replaying the full stream on every instantiation is wasteful.

## Savepoints: A Domain-Driven Solution

A **savepoint** is a regular domain event that summarizes the state of a projection at a point in time. It is not a framework construct — it is a pure business event, appended by your application whenever you decide it makes sense.

Consider a stock keeping scenario with these events:

```java
sealed interface StockEvent {

    record StockAdded(String product, int quantity) implements StockEvent { }

    record StockPicked(String product, int quantity) implements StockEvent { }

    /**
     * Savepoint event that summarizes the stock count at a certain moment in time.
     * This is a pure domain event — no special framework support needed.
     */
    record StockCounted(String product, int counted) implements StockEvent { }

}
```

`StockAdded` and `StockPicked` are individual stock movements. `StockCounted` is the savepoint — it captures the stock level at a moment in time. Think of it as a physical stock count in a warehouse: someone counts what is on the shelf and records the number.

With a savepoint in the stream, a new projection does not need to replay every movement from the beginning. It can jump to the most recent savepoint, initialize from there, and then process only the movements that came after.

## Using initQuery

The `Projection` interface provides an optional `initQuery()` method that enables this pattern:

```java
static class StockLevelProjection implements Projection<StockEvent> {

    private final String product;
    private int level = 0;

    public StockLevelProjection(String product) {
        this.product = product;
    }

    @Override
    public EventQuery initQuery() {
        // Find the last stock count (savepoint) for this product
        return EventQuery.forEvents(
            EventTypesFilter.of(StockCounted.class),
            Tags.of("product", product)
        ).backwards().limit(1);
    }

    @Override
    public EventQuery eventQuery() {
        // Only process movements — savepoints are handled exclusively by initQuery
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

    public int level() {
        return level;
    }

}
```

The key elements:

- **`initQuery()`** returns a backward query with limit 1 — it finds the most recent `StockCounted` event for this product
- **`eventQuery()`** returns the movements (`StockAdded`, `StockPicked`) — but **not** `StockCounted`
- **`when()`** handles all three event types, since it receives events from both queries

## How the Projector Executes It

When the Projector runs a projection that has an `initQuery()`, the execution proceeds in two phases:

1. **Phase 1 — Initialization**: The `initQuery()` is executed. If a savepoint is found, it is passed to the `when()` handler to initialize state. The savepoint's event reference becomes the cursor position.

2. **Phase 2 — Delta processing**: The `eventQuery()` is executed starting from the cursor position set by the initQuery. Only events after the savepoint are processed.

On subsequent runs of the same Projector instance, the initQuery is skipped — the projector already has a cursor position and continues from there.

## A Full Example

Here is a complete walkthrough showing the savepoint pattern in action:

```java
EventStore eventstore = InMemoryEventStorage.newBuilder().buildStore();

EventStreamId streamId = EventStreamId.forContext("warehouse");
EventStream<StockEvent> stream = eventstore.getEventStream(streamId, StockEvent.class);

String product = "WIDGET-42";
Tags tags = Tags.of("product", product);

// Simulate stock movements
stream.append(AppendCriteria.none(), Event.of(new StockAdded(product, 100), tags));
stream.append(AppendCriteria.none(), Event.of(new StockPicked(product, 10), tags));
stream.append(AppendCriteria.none(), Event.of(new StockPicked(product, 5), tags));
stream.append(AppendCriteria.none(), Event.of(new StockAdded(product, 50), tags));
stream.append(AppendCriteria.none(), Event.of(new StockPicked(product, 20), tags));
```

### First run — no savepoint exists

```java
StockLevelProjection projection = new StockLevelProjection(product);
ProjectorMetrics metrics = Projector.from(stream).towards(projection).build().run();

System.out.println("Stock level: " + projection.level());           // 115
System.out.println("Events handled: " + metrics.eventsHandled());   // 5
```

No savepoint exists yet, so the `initQuery()` returns nothing and the `eventQuery()` replays all five movements from the beginning. The pattern degrades gracefully — no special handling needed.

### Append a savepoint

```java
stream.append(AppendCriteria.none(), Event.of(new StockCounted(product, 115), tags));
```

This records the current stock level as a domain event. It is the application's choice when to do this — after a batch of operations, on a schedule, or triggered by a manual stock count.

### Second run — savepoint found

```java
// More movements after the savepoint
stream.append(AppendCriteria.none(), Event.of(new StockAdded(product, 30), tags));
stream.append(AppendCriteria.none(), Event.of(new StockPicked(product, 12), tags));

StockLevelProjection projection2 = new StockLevelProjection(product);
ProjectorMetrics metrics2 = Projector.from(stream).towards(projection2).build().run();

System.out.println("Stock level: " + projection2.level());              // 133
System.out.println("Events handled: " + metrics2.eventsHandled());      // 3
System.out.println("Queries done: " + metrics2.queriesDone());          // 2
```

This time the `initQuery()` finds the `StockCounted(115)` savepoint. The projection initializes with `level = 115`, then processes only the two movements after it. Three events handled total (1 savepoint + 2 movements), two queries done (1 initQuery + 1 eventQuery) — instead of replaying all eight events from the beginning.

### Fixing a bad savepoint

```java
stream.append(AppendCriteria.none(), Event.of(new StockCounted(product, 133), tags));
stream.append(AppendCriteria.none(), Event.of(new StockPicked(product, 3), tags));

StockLevelProjection projection3 = new StockLevelProjection(product);
ProjectorMetrics metrics3 = Projector.from(stream).towards(projection3).build().run();

System.out.println("Stock level: " + projection3.level());              // 130
System.out.println("Events handled: " + metrics3.eventsHandled());      // 2
```

The `initQuery()` always finds the most recent savepoint. If a previous savepoint was incorrect, just append a corrected one — the next projection run picks it up automatically.

## Design Decisions

### Keep initQuery and eventQuery separate

The `StockCounted` event type appears **only** in the `initQuery()`, not in the `eventQuery()`. This is intentional:

- The main query never processes savepoint events, which protects against buggy historical savepoints
- There is no risk of double-processing a savepoint that happens to fall within the eventQuery range
- Each query has a clear responsibility: initQuery initializes, eventQuery processes deltas

### Savepoints are pure domain events

Savepoints are ordinary events appended to the stream with `stream.append()`. There is no special framework API or savepoint-specific storage. This means:

- You decide when savepoints are created
- You decide what data they contain
- They are visible in queries like any other event
- They participate in the normal event history

### Graceful degradation

When no savepoint exists — for example, on the very first run before any savepoint has been appended — the `initQuery()` returns nothing and the main `eventQuery()` replays from the beginning. No special configuration or fallback logic is needed.

## Savepoints vs Bookmarking

Both savepoints and bookmarking address the problem of avoiding full replays, but they serve different purposes:

| | Savepoint Pattern | Bookmarking |
|---|---|---|
| **State storage** | Domain events in the stream | Separate bookmark record |
| **First run** | Jumps to last savepoint | Replays from beginning |
| **Created by** | Application logic | Projector framework |
| **Use case** | On-demand projections, live models | Long-lived persistent projections |
| **Requires persistence** | No (savepoints are in the stream) | Yes (bookmark storage) |

> When bookmarking is enabled on the Projector, the `initQuery()` is ignored. Bookmarked projections track their own position and must process every event to stay consistent. If both are configured, a warning is logged at build time.
{: .prompt-warning }

The savepoint pattern is particularly well-suited for projections that are created on-the-fly — for example, to answer a specific query in an API request. Instead of maintaining a long-lived bookmarked projection, you instantiate a projection, let it catch up from the latest savepoint, and discard it when you are done.

## When to Create Savepoints

The choice of when to append savepoint events is entirely up to the application. Some common strategies:

- **Periodic**: Append a savepoint every N events or on a time schedule
- **On demand**: Append a savepoint after significant operations (e.g., after a physical stock count)
- **Threshold-based**: Append a savepoint when the delta since the last savepoint exceeds a threshold

Since savepoints are domain events, they can also carry business meaning. A `StockCounted` event is not just a technical optimization — it represents a real business activity (a stock count). This makes the savepoint pattern a natural fit for many domains.

> The number of events between savepoints determines the tradeoff: more frequent savepoints mean faster initialization but more events in the stream. Find the right balance for your use case.
{: .prompt-info }
