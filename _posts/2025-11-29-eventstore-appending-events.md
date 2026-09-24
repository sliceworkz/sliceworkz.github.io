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

stream.append(Event.of(new CustomerRegistered("John"), Tags.none()));
```

That is an unconditional append: nothing was read to decide on, so there is no consistency boundary to check, no `OptimisticLockingException` is possible, and on PostgreSQL no lock is taken. It is exactly `stream.append(AppendCriteria.none(), event)`, which stays valid — the two spellings are interchangeable. The overloads taking `AppendCriteria` are for the [conditional appends](#optimistic-locking) below.

**Append through the stream the event belongs to.** An event is stored in exactly one stream, so a stream opened on a wildcard id — `EventStreamId.anyContext()`, or `forContext("customer").anyPurpose()` — reads across every stream it matches but refuses `append` with `IllegalArgumentException`, storing nothing. To write to a stream a wildcard reads across, open that stream by its own id: a handle is cheap, and a stream per operation is the intended usage.

The `append()` method returns a list of fully-formed `Event` objects with assigned metadata (reference, position, timestamp).

**That return value is the read-your-own-writes story.** The events come back typed, with their assigned references — the same list you would otherwise have had to query back. To act on your own append on the appending thread, write the code after the call rather than subscribing to anything:

```java
List<Event<CustomerEvent>> written = stream.append(
    Event.of(new CustomerRegistered("John"), Tags.none()));

EventReference justWritten = written.getLast().reference();
return Response.accepted().header("X-Event-Reference", justWritten.toString()).build();
```

Note that `append` deserializes the events it just wrote in order to return them. A payload that serializes but cannot be read back therefore fails *here*, as an `EventDeserializationException`, with the event already stored — see [Error Handling](/posts/eventstore-error-handling/).

## Adding Tags

Tags are key-value pairs that enable dynamic querying and correlation of events across different event types. They are central to the Dynamic Consistency Boundary (DCB) pattern.

```java
// Single tag
stream.append(Event.of(new CustomerRegistered("John"), Tags.of("customer", "123")));

// Multiple tags: alternating keys and values
stream.append(Event.of(
    new CustomerRegistered("John"),
    Tags.of("customer", "123", "region", "EU", "priority", "high")
));
```

`Tags.of(key, value, ...)` takes alternating keys and values; an odd count is rejected rather than read as a trailing flag. `Tags.of(Tag...)` builds from tags you already hold.

Tags enable querying events across different event types based on shared business identifiers:

```java
// Find all events for a specific customer, regardless of event type
List<Event<CustomerEvent>> customerEvents = stream.query(
    EventQuery.forTags(Tags.of("customer", "123"))
);
```

### `Tags` Is a Set

`Tags.of(Tag...)` eliminates a repeated tag rather than rejecting it: tags gathered from several sources — a domain tag list plus what a decorator adds — legitimately overlap, and that overlap should not turn into an append failure. A `null` element is an `IllegalArgumentException`.

Several tags under one *key* are an ordinary shape — a transfer tagged `customer:alice` and `customer:bob`. So reading a tag back by key has two methods:

```java
Tags tags = event.tags();

tags.tags("customer");   // Set<Tag>: every tag under the key, possibly several
tags.tag("region");      // Optional<Tag>: the one tag under the key; throws IllegalStateException if there are several
```

`tag(key)` refuses to guess rather than returning whichever tag hash order put first; use `tags(key)` for a key that may hold several.

You can also add tags to annotate events with any application-level metadata you need, but that you don't like to put in your event definition payloads.

### What a Tag May Contain

A tag is stored and matched as a single flattened string, `"key:value"` — that string form is the wire format, not a debugging rendering. The PostgreSQL backend stores it in a `text[]` column and answers a tag query with array containment built from the same rendering.

**Construction therefore rejects the shapes that would not survive that round trip.** `Tag.of(...)` throws `IllegalArgumentException` for:

| Rejected | Why |
|---|---|
| a `':'` in the **key** — `Tag.of("a:b", "c")` | renders to `"a:b:c"`, which is also what `Tag.of("a", "b:c")` renders to: two logical tags, one stored string |
| leading or trailing whitespace on either half | the read path strips it, so it would not come back as written |
| an empty key or value | the read path maps an empty half to `null` |
| a tag with neither key nor value | renders as `""` and reads back as nothing |

Values may contain `':'` freely — parsing splits on the first colon only — and whitespace *inside* a key or value is fine. `Tag.of(null, "v")` stays legal.

```java
Tags.of("customer", "123")            // fine
Tags.of("url", "https://example.com") // fine — colons in the value are allowed
Tag.of("na:me", "x")                  // IllegalArgumentException — colon in the key
Tag.of("name", " x ")                 // IllegalArgumentException — padded value
```

Whitespace is rejected rather than silently stripped so the mistake surfaces where it is made. Code handling untrusted input should `strip()` before constructing a tag.

What this buys is that **a tag read off an event is the tag that was appended** — which matters, because re-tagging a new event with a tag read back from an old one is an ordinary pattern.

> **Matching is exact containment, never key-prefix.** `Tag.of("customer")` and `Tag.of("customer", "123")` are two different tags, and a query for the first does **not** return events carrying the second. There is no wildcard form. Tag every event with `Tag.of("customer", id)` and query for that; a bare key is a flag, for when the presence of the tag is itself the fact.
{: .prompt-warning }

## Appended Event Metadata

When an event is appended, the EventStore enriches it with metadata:

- **EventReference**: A unique reference containing both a global `EventId` (UUIDv7) and a `position` (sequential number starting at 1)
- **Timestamp**: When the event was persisted, as an `Instant`
- **Stream**: The `EventStreamId` the event belongs to

```java
List<Event<CustomerEvent>> appended = stream.append(
    Event.of(new CustomerRegistered("John"), Tags.none())
);

Event<CustomerEvent> event = appended.get(0);
EventReference ref = event.reference();

System.out.println("Event ID: " + ref.id());
System.out.println("Position: " + ref.position());  // Sequential: 1, 2, 3, ...
System.out.println("Timestamp: " + event.timestamp());   // an Instant
```

**The timestamp is an `Instant`**: the moment the store persisted the event, on the storage's clock — the JVM's in memory, the database server's on PostgreSQL, where the column is a `timestamptz`. It is the same kind of value as a bookmark's `updatedAt` and the clocks on a lease, so they compare without conversion. An instant has no zone: the day or wall-clock time it falls on is your rendering, made with the zone you mean — `event.timestamp().atZone(ZoneId.of("Europe/Brussels"))`.

### EventReference: A unique reference to your event

The `EventReference` combines identity and ordering:
- **EventId**: Globally unique identifier (UUIDv7 — time-ordered per [RFC 9562](https://www.rfc-editor.org/rfc/rfc9562#section-5.7), improving B-tree index locality for append-only workloads)
- **Position**: Sequential position within the stream (starts at 1, unique over all streams stored in the same storage)
- **Tx**: The transaction during which this event was appended. Primary sort criterion before position
- **Index**: Sub-event index within a single stored event (0 for regular events, 0..N for events produced by multi-event upcasting)

Events are ordered by `(tx, position, index)`. For non-upcasted events the index is always 0. When a legacy event is upcasted into multiple current events, each sub-event shares the same id/position/tx but receives a distinct index (0, 1, 2, ...).

This way, the `EventReference` provides a great reference to:
- determine where you were in processing events in your stream (querying a next batch of Events after that reference next time)
- version a projection built from events, to determine up until which event
- passing a reference to a client to demand a view that has been updated to at least the information that was submitted by that specific client (consistency)
- compare the sequence in which two events have happened, based on their `position` in the stream
- etc...

### Serializing and Deserializing EventReferences

`EventReference` supports string serialization via `toString()` and `fromString()`, useful for passing references through APIs, storing them in external systems, or logging:

```java
EventReference ref = events.getLast().reference();

// Serialize to string
String serialized = ref.toString();
// Format: "<id>:<tx>:<position>" e.g. "550e8400-e29b-41d4-a716-446655440000:10:42"

// Deserialize back
EventReference restored = EventReference.fromString(serialized);
```

For events produced by multi-event upcasting, the index is included as a fourth component:

```java
// Format with index: "<id>:<tx>:<position>:<index>"
// e.g. "550e8400-e29b-41d4-a716-446655440000:10:42:3"
```

When the index is 0 (the common case for non-upcasted events), it is omitted from the string representation.

`fromString()` throws `IllegalArgumentException` if the input is null, blank, has the wrong number of components, or contains non-numeric values where numbers are expected.

EventReference is also crucial for implementing optimistic locking in the DCB pattern. It allows you to note the last relevant event when making a decision, then verify no new relevant facts have emerged when appending the result.

```java
EventReference head = stream.head().orElse(null);
// Use this reference for optimistic locking
```

## Appending Multiple Events to a Stream

Multiple events can be appended in a single atomic operation by passing a list:

```java
stream.append(List.of(
    Event.of(new CustomerRegistered("John"), Tags.of("customer", "123")),
    Event.of(new CustomerAddressChanged("Main St"), Tags.of("customer", "123")),
    Event.of(new CustomerEmailChanged("john@example.com"), Tags.of("customer", "123"))
));
```

All events in the list are appended atomically: either all succeed or all fail. Each event receives a consecutive position number within the stream.

The parameter is a `List<? extends EphemeralEvent<? extends E>>`, so a list you mapped from your domain events fits as it is, with no type witness:

```java
List<CustomerEvent> decided = customer.rename("Jane");

stream.append(criteria, decided.stream()
    .map(e -> Event.of(e, Tags.of("customer", "123")))
    .toList());
```

**Important**: When using optimistic locking with batch appends, the `AppendCriteria` check is performed once before appending any events. If the check passes, all events are appended together.

## Optimistic Locking

The EventStore implements optimistic locking through the DCB pattern using `AppendCriteria`. This ensures that business decisions based on historical facts remain valid at the moment of appending new events.

### How It Works

1. Take the stream's `head()` — the reference of its newest stored event — *before* reading
2. Query relevant events using an `EventQuery`, bounded with `until(head)`
3. Make a business decision based on those facts
4. Append new events with `AppendCriteria` containing the same query — **without** the `until` — and the head
5. If new events matching the query exist after the head, `OptimisticLockingException` is thrown

```java
EventQuery relevant = EventQuery.forTags(Tags.of("customer", "123"));

// Step 1: pin the boundary. An absent head is an empty stream, and a valid boundary
EventReference head = stream.head().orElse(null);

// Step 2 & 3: query the relevant facts as of that head, and decide
List<Event<CustomerEvent>> facts = stream.query(relevant.until(head));

// Step 4 & 5: conditional append -- the query unbounded, the head as the expected last event
try {
    stream.append(
        AppendCriteria.of(relevant, head),
        Event.of(new CustomerNameChanged("Jane"), Tags.of("customer", "123"))
    );
} catch (OptimisticLockingException e) {
    // New relevant facts emerged - take the head again, re-read, re-decide
}
```

### Why the Head, and Not the Last Relevant Event

The classic formulation notes the reference of the last *relevant* event from the query result and hands that to `AppendCriteria`. That still works, and is correct. Taking the head first is better in three ways:

- **One boundary for several reads.** A decision that takes more than one query — a decision model plus an existence check, say — bounds each of them with `until(head)`, so nothing can land between the reads unseen. With the last relevant event of each read there is no single moment the decision was taken at.
- **No special case for an empty history.** `head()` is empty for an empty stream, and `null` is a valid "I decided on nothing" reference. There is no `getLast()` to throw on an empty result.
- **A cheaper check on PostgreSQL.** The lock check walks the stream forward *from the reference* looking for a matching event. From the head that is a walk over the few events appended during the decision; from the last relevant event of a long-idle entity it is a walk over every stream event since — see [Stream Design and Performance](/posts/eventstore-stream-design-and-performance/#the-consistency-check-and-a-stale-cursor).

`head()` never deserializes, upcasts or decrypts: an event this stream cannot map, one that upcasts into nothing, or one sealed under a key store that is down cannot make it fail. It is what a query would see — on PostgreSQL it sits behind the same visibility barrier as every read — and the event at the head need not match your filter: it is a cursor for the check, and only matching events after it count.

> **Hand `AppendCriteria` the query without its `until`.** An `until` bounds a consistency boundary exactly as it bounds a read: `AppendCriteria.of(relevant.until(head), head)` — which is what reusing "the query I read with" produces, and reads as the more careful of the two — deems nothing after the head relevant. Every append is admitted and no `OptimisticLockingException` is ever raised: optimistic locking is off, and nothing tells you so.
{: .prompt-danger }

### First Append (Empty Stream)

When appending to an empty stream, or when no previous relevant events exist, there is no `EventReference` to pass — `head()` is empty, and `null` is what you pass:

```java
stream.append(
    AppendCriteria.of(EventQuery.forTags(Tags.of("customer", "123")), null),  // decided on an empty result
    Event.of(new CustomerRegistered("John"), Tags.of("customer", "123"))
);
```

**This is not the same as `AppendCriteria.none()`, and the difference is load-bearing.** An absent reference under a *real* filter means "I decided on an empty stream", which is still a consistency boundary: if any matching event exists, that is a new relevant fact and the append correctly fails with `OptimisticLockingException`. `AppendCriteria.none()` carries no filter at all and skips the check entirely.

```java
// checked: fails if any event tagged customer=123 exists
AppendCriteria.of(EventQuery.forTags(Tags.of("customer", "123")), null);

// unchecked: appends unconditionally
AppendCriteria.none();
```

A boundary over a current event type also counts the legacy events that [upcast](/posts/eventstore-defining-events/#approach-2-upcasting) into it, exactly as a query for that type returns them — so a decision read through the query and the lock check never disagree about a legacy event.

`AppendCriteria.isNone()` is what distinguishes them — it is derived from the filter, independently of the reference. And `expectedLastEventReference()` is **never null** whichever factory produced the criteria: a null is normalised to `Optional.empty()`, so it is always safe to call `.isPresent()` on it.

## Idempotency

Idempotency ensures that duplicate command submissions don't create duplicate events.
When handling for example incoming REST calls or asynchronous messages (JMS, Kafka, ...), it could happen that your correctly process and append the information, but that you're not able to acknowledge proper processing to the client or messaging system due to a system or connection failure.
In that case, it is to be expected that the client assumes processing hasn't happened yet, and that it resubmits the same information.  Idempotency in your system then allows to detect and silently ignore the duplicate processing, while confirming (again) to the client that reception and processing has happened correctly.

### Idempotent append()

The EventStore provides built-in idempotency support through idempotency keys. When appending an event with an idempotency key, if an event with the same key already exists **on the same stream**, the append operation is silently ignored and returns an empty list.

```java
String requestId = "req-2025-01-15-abc123";

// First submission - succeeds and returns the appended event
List<Event<CustomerEvent>> events = stream.append(
    Event.of(new CustomerRegistered("John"), Tags.none())
        .withIdempotencyKey(requestId)
);
assertEquals(1, events.size());

// Duplicate submission - silently ignored, returns empty list
events = stream.append(
    Event.of(new CustomerRegistered("John"), Tags.none())
        .withIdempotencyKey(requestId)
);
assertEquals(0, events.size());
```

#### A Batch Is the Unit of De-duplication

A command that produces several events gives each of them a key, derived from the command's id — keys are per event, because that is what storage holds, and the events of one batch must carry *distinct* keys:

```java
String commandId = "cmd-4711";

stream.append(criteria, List.of(
    Event.of(new OrderPlaced("o-1"), tags).withIdempotencyKey(commandId + "/1"),
    Event.of(new OrderLineAdded("sku-9", 2), tags).withIdempotencyKey(commandId + "/2")));
```

A batch is stored atomically, so a genuine retry finds every key or none. That gives three cases:

| The batch… | Outcome |
|---|---|
| has every key stored already | a **retry**: swallowed whole, nothing written, an empty list returned, nobody notified |
| has no key stored yet | appended normally |
| mixes stored and new keys | refused with **`IdempotencyKeyConflictException`**, nothing stored |

The mixed case cannot be a retry: one of its events collides with a *different* event holding its key, and the rest are unknown to the store. Both silent answers would lie — storing the unknown events lets the caller believe the colliding fact landed too, and swallowing the batch loses the unknown events — so it throws. `IdempotencyKeyConflictException` (in `…eventstore.stream`) extends `RuntimeException` directly rather than `EventStorageException`, because it is never worth retrying: it is a bug in how keys are derived.

A batch that **repeats a key within itself** is refused with `IllegalArgumentException` before anything is written. An event without a key rides along with the keyed ones in a retry; the one blind spot is a batch reusing its keys with *different* unkeyed events, which cannot be told from a retry — so a command should key every event it emits.

**The lock check runs first.** A conditional append that conflicts raises `OptimisticLockingException` whether or not its keys are duplicates; de-duplication is only seen by an append that was admitted.

#### Keys Are Scoped Per Event Stream

Deduplication is scoped to the logical stream — context **plus** purpose — not to the storage as a whole. The same key used on two unrelated streams does not collide:

```java
EventStream<OrderEvent>   orders    = eventstore.getEventStream(EventStreamId.forContext("order"),    OrderEvent.class);
EventStream<InvoiceEvent> invoices  = eventstore.getEventStream(EventStreamId.forContext("invoice"), InvoiceEvent.class);

// both succeed: same key, different streams
orders.append(Event.of(new OrderPlaced("o-1"), Tags.none()).withIdempotencyKey("msg-42"));
invoices.append(Event.of(new InvoiceIssued("i-1"), Tags.none()).withIdempotencyKey("msg-42"));
```

That is what makes a key drawn from an upstream source — a message id, a request id — usable directly, without prefixing it per context to avoid accidental collisions. It also means deduplication behaviour does not depend on how storage instances or table prefixes happen to be wired at runtime.

On PostgreSQL this is enforced by a partial unique index on `(stream_context, stream_purpose, idempotency_key)`; see [PostgreSQL EventStorage](/posts/eventstore-configuring-postgresql-storage/#using-the-initialization-script).

#### Reading the Key Back

The idempotency key is persisted and surfaced on the SPI-level `StoredEvent` when reading. It is deliberately **not** exposed on the public `Event` record — application code decides idempotency at the boundary where the key comes from, and does not need it back on every event it handles. The key does survive an [import between stores](/posts/eventstore-importing-events/).

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
        EventQuery.forTags(Tags.of("idempotency", idempotencyKey)),
        null  // Expect no prior event with this key
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
