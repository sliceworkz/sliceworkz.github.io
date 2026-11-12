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

This guide covers the savepoint pattern — a technique for avoiding full event replays when initializing projections, using the `initQuery()` method of a projection.

## The Problem: Full Replays

Projections build read models by processing events sequentially. When a projection is initialized from scratch, it replays every event from the beginning of the stream:

```java
StockLevelProjection projection = new StockLevelProjection("WIDGET-42");
Projector.from(stream).into(projection).build().run();
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
        // Find the last stock count (savepoint) for this product: newest first, one event
        return EventQuery.of(EventFilter.forTypes(StockCounted.class).tagged("product", product))
                         .backwards().limit(1);
    }

    @Override
    public EventFilter eventFilter() {
        // Only process movements — savepoints are handled exclusively by initQuery
        return EventFilter.forTypes(StockAdded.class, StockPicked.class).tagged("product", product);
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
- **`eventFilter()`** selects the movements (`StockAdded`, `StockPicked`) — but **not** `StockCounted`. It is read forward, which is what the default `eventQuery()` does with it, so there is no query to write
- **`when()`** handles all three event types, since it receives events from both queries

## How the Projector Executes It

When the Projector runs a projection that has an `initQuery()`, the execution proceeds in two phases:

1. **Phase 1 — Initialization**: The `initQuery()` is executed. If a savepoint is found, it is passed to the `when()` handler to initialize state. The savepoint's event reference becomes the cursor position.

2. **Phase 2 — Delta processing**: The main query — `eventFilter()` read forward — is executed starting from the cursor position set by the initQuery. Only events after the savepoint are processed.

On subsequent runs of the same Projector instance, the initQuery is skipped — the projector already has a cursor position and continues from there.

If the handler throws on the savepoint — a savepoint written by a buggy version, say — the run fails as a `ProjectorException` naming the savepoint event, and the cursor goes back to where the run started. The next run therefore re-runs `initQuery()` rather than starting the delta query from a projection that was never initialised. `initQuery()` returning `null` is tolerated and means the same as the default, `EventQuery.of(EventFilter.matchNone())`: no savepoint phase.

## A Full Example

Here is a complete walkthrough showing the savepoint pattern in action:

```java
EventStore eventstore = InMemoryEventStorage.newBuilder().buildStore();

EventStreamId streamId = EventStreamId.forContext("warehouse");
EventStream<StockEvent> stream = eventstore.getEventStream(streamId, StockEvent.class);

String product = "WIDGET-42";
Tags tags = Tags.of("product", product);

// Simulate stock movements
stream.append(Event.of(new StockAdded(product, 100), tags));
stream.append(Event.of(new StockPicked(product, 10), tags));
stream.append(Event.of(new StockPicked(product, 5), tags));
stream.append(Event.of(new StockAdded(product, 50), tags));
stream.append(Event.of(new StockPicked(product, 20), tags));
```

### First run — no savepoint exists

```java
StockLevelProjection projection = new StockLevelProjection(product);
ProjectorMetrics metrics = Projector.from(stream).into(projection).build().run();

System.out.println("Stock level: " + projection.level());           // 115
System.out.println("Events handled: " + metrics.eventsHandled());   // 5
```

No savepoint exists yet, so the `initQuery()` returns nothing and the main query replays all five movements from the beginning. The pattern degrades gracefully — no special handling needed.

### Append a savepoint

```java
stream.append(Event.of(new StockCounted(product, 115), tags));
```

This records the current stock level as a domain event. It is the application's choice when to do this — after a batch of operations, on a schedule, or triggered by a manual stock count.

### Second run — savepoint found

```java
// More movements after the savepoint
stream.append(Event.of(new StockAdded(product, 30), tags));
stream.append(Event.of(new StockPicked(product, 12), tags));

StockLevelProjection projection2 = new StockLevelProjection(product);
ProjectorMetrics metrics2 = Projector.from(stream).into(projection2).build().run();

System.out.println("Stock level: " + projection2.level());              // 133
System.out.println("Events handled: " + metrics2.eventsHandled());      // 3
System.out.println("Queries done: " + metrics2.queriesDone());          // 2
```

This time the `initQuery()` finds the `StockCounted(115)` savepoint. The projection initializes with `level = 115`, then processes only the two movements after it. Three events handled total (1 savepoint + 2 movements), two queries done (1 initQuery + 1 main query) — instead of replaying all eight events from the beginning.

### Fixing a bad savepoint

```java
stream.append(Event.of(new StockCounted(product, 133), tags));
stream.append(Event.of(new StockPicked(product, 3), tags));

StockLevelProjection projection3 = new StockLevelProjection(product);
ProjectorMetrics metrics3 = Projector.from(stream).into(projection3).build().run();

System.out.println("Stock level: " + projection3.level());              // 130
System.out.println("Events handled: " + metrics3.eventsHandled());      // 2
```

The `initQuery()` always finds the most recent savepoint. If a previous savepoint was incorrect, just append a corrected one — the next projection run picks it up automatically.

## Design Decisions

### Keep initQuery and eventFilter separate

The `StockCounted` event type appears **only** in the `initQuery()`, not in the `eventFilter()`. This is intentional:

- The main query never processes savepoint events, which protects against buggy historical savepoints
- There is no risk of double-processing a savepoint that happens to fall within the main query's range
- Each has a clear responsibility: initQuery initializes, eventFilter selects the deltas

### Savepoints are pure domain events

Savepoints are ordinary events appended to the stream with `stream.append()`. There is no special framework API or savepoint-specific storage. This means:

- You decide when savepoints are created
- You decide what data they contain
- They are visible in queries like any other event
- They participate in the normal event history

### Graceful degradation

When no savepoint exists — for example, on the very first run before any savepoint has been appended — the `initQuery()` returns nothing and the main read replays from the beginning. No special configuration or fallback logic is needed.

## Savepoints in a Decision

The same shape works without a `Projector`, when a command reads its facts by hand. A savepoint is then a business bound on the filter: the movements recorded *after* the last count are relevant, nothing before it. Since 0.13 each item of a filter carries its own bounds, so that is one filter, and the read and the consistency check agree on it:

```java
EventReference head = stream.head().orElse(null);

// the newest count, read up to the head
Optional<Event<StockEvent>> lastCount = stream.query(
        EventQuery.of(EventFilter.forTypes(StockCounted.class).tagged("product", product))
                  .backwards().limit(1).notAfter(head))
    .stream().findFirst();

// the movements after it -- after(null) is no bound, so no count means every movement
EventFilter movements = EventFilter.forTypes(StockAdded.class, StockPicked.class)
                                   .tagged("product", product)
                                   .after(lastCount.map(Event::reference).orElse(null));

List<Event<StockEvent>> delta = stream.query(EventQuery.of(movements).notAfter(head));
// ... decide on the count plus the delta, then append with AppendCriteria.of(movements, head)
```

The `after` bound costs the consistency check nothing: it lies before the head, so `AppendCriteria` drops it and checks for any movement after the head, as it should. A new `StockCounted` appended meanwhile does not invalidate the decision — add `StockCounted` to the checked filter with `.or(...)` if it should.

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
