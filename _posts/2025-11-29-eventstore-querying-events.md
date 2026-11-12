---
layout: post
toc: true
title: Querying Events
description: Querying for Domain Events
date: 2025-11-29 03:00:00
categories: [Eventstore Documentation,Eventstore API]
tags: [events,query,tags,paging,head,raw stream]
---

This guide covers the various ways to query events from the EventStore, including filtering, paging, backward queries, temporal queries, raw reads and cross-stream querying.

## Filters and Queries

Two types select events, and each says one thing:

- An **`EventFilter`** says **which** events are relevant: event types, tags, and optionally the part of the history they are about. It is the business half of a read, and it is also what a consistency boundary (`AppendCriteria`) checks.
- An **`EventQuery`** says **how** a filter is read: `EventQuery.of(filter)` plus a direction (`backwards()`), a `limit(n)` and a read bound (`notAfter(ref)`). None of those says what is relevant, so none of them ever reaches an `AppendCriteria`.

A query is always built from a filter, and a plain forward read needs no query at all — `stream.query(filter)` reads what the filter selects, forward, with no limit:

```java
EventFilter relevant = EventFilter.forTypes(CustomerEvent.class).tagged("customer", "123");

List<Event<CustomerEvent>> all    = stream.query(relevant);                                    // forward, everything
List<Event<CustomerEvent>> newest = stream.query(EventQuery.of(relevant).backwards().limit(1)); // how it is read
```

**An event matches a filter if:**
1. The event's type is **any** of the types allowed by the filter (OR condition)
2. **All** tags specified by the filter are present on the event (AND condition)
3. It lies within the filter's bounds, when it has any (see [Bounding What Is Relevant](#bounding-what-is-relevant-after-and-until))

Events may have additional tags beyond those specified in the filter—the filter only requires that the specified tags are present.

```java
// CustomerRegistered OR CustomerUpdated events with region=EU tag
EventFilter filter = EventFilter.forTypes(CustomerRegistered.class, CustomerUpdated.class)
                                .tagged("region", "EU");

// This matches:
Event.of(new CustomerRegistered("John"), Tags.of("region", "EU")) // ✓ Type matches, has required tag
Event.of(new CustomerUpdated("Jane"), Tags.of("region", "EU", "premium", "true")) // ✓ Type matches, has required tag (plus extra)

// This does NOT match:
Event.of(new CustomerRegistered("Bob"), Tags.of("region", "US")) // ✗ Type matches, but wrong tag value
Event.of(new CustomerChurned("Alice"), Tags.of("region", "EU")) // ✗ Has required tag, but wrong type
Event.of(new CustomerUpdated("Dave"), Tags.none()) // ✗ Type matches, but missing required tag
```

The same filter can be used both for database-level filtering and in-process filtering, as explained further down.

## Building a Filter

There are two spellings, and they build the same filter — a filter built one way compares equal to the same filter built the other way.

**Fluently**, starting from the types or from the tags, as a chain: the types first, then the tags every matching event must carry, then its bounds, then a union where a decision spans several boundaries:

```java
// all events of the CustomerEvent hierarchy, for one customer
EventFilter.forTypes(CustomerEvent.class).tagged("customer", "123");

// any type, carrying the tag -- the usual shape of a consistency boundary
EventFilter.forTags("customer", "123");

// narrowing accumulates: each tagged(...) adds a required tag
EventFilter.forTypes(OrderPlaced.class, OrderShipped.class)
           .tagged("customer", "123")
           .tagged("region", "EU");
```

Every builder has a form for what you already hold:

| You hold | Spelling |
|---|---|
| event classes | `EventFilter.forTypes(CustomerRegistered.class, CustomerRenamed.class)` |
| stored type names (a raw stream, an import, a browsing tool) | `EventFilter.forTypes(EventType.named("CustomerRegistered"))` — taken literally, nothing resolved |
| one tag as key and value | `EventFilter.forTags("customer", "123")`, `.tagged("customer", "123")` |
| `Tag` objects made elsewhere (an entity's tag, a rule tag) | `EventFilter.forTags(customerTag)`, `.tagged(customerTag, regionTag)` |
| a `Tags` set | `EventFilter.forTags(Tags.of("region", "EU", "priority", "high"))`, `.tagged(tags)` |

**From its two halves**, when you already hold a types filter and a set of tags:

```java
EventFilter.of(EventTypesFilter.of(OrderPlaced.class, OrderShipped.class),
               Tags.of("customer", "123", "region", "EU"));
```

`EventFilter.matchAll()` and `EventFilter.matchNone()` complete the set; `EventFilter.of(items...)` builds a filter directly from `EventFilterItem`s, and `EventFilter.of()` with none matches nothing.

**A sealed interface stands for every event type under it.** `forTypes(CustomerEvent.class)` names the whole hierarchy, a nested sealed interface names its own branch, and the filter resolves them into the stored names of the records when it is built. A non-sealed interface is refused. See [Defining Events](/posts/eventstore-defining-events/#a-sealed-interface-names-its-whole-hierarchy).

**Name current types, never legacy ones.** On a stream that registers upcasting, a filter naming a `@LegacyEvent` class is refused with `IllegalArgumentException`: name the current type it is read as, and its legacy events come along. See [Querying with legacy Events](#querying-with-legacy-events).

### Building a Query

Wrap the filter when the read is anything other than forward and complete:

```java
EventQuery.of(filter)                       // forward, no limit, no read bound
EventQuery.of(filter).backwards()           // newest first
EventQuery.of(filter).limit(500)            // read at most 500 stored events
EventQuery.of(filter).notAfter(head)        // read nothing recorded after head
```

There is deliberately no shorthand that skips the filter, and no union of queries — see [Complex Event Filters](#complex-event-filters).

## In-Database vs In-Process Querying

The same filter can select events at two different levels:

1. **Database-level filtering**: Pass the filter (or a query built on it) to the event stream's `query()` method
2. **In-process filtering**: Ask the filter whether an event matches: `filter.matches(event)`

### Database-Level Filtering (Recommended)

When you pass a filter or an `EventQuery` to the event stream, the filtering happens in the event storage (database):

```java
EventStream<CustomerEvent> stream = eventstore.getEventStream(streamId, CustomerEvent.class);

EventFilter filter = EventFilter.forTypes(CustomerRegistered.class).tagged("region", "EU");

// Filtering is executed in the database
List<Event<CustomerEvent>> events = stream.query(filter);
events.forEach(event -> processEvent(event));
```

**Advantages:**
- Only matching events are read from the database
- Efficient—leverages database indexes and query optimization
- Minimal memory usage and network transfer
- Recommended for most use cases

### In-Process Filtering

An `EventFilter` can test events in your Java code:

```java
EventFilter filter = EventFilter.forTypes(CustomerRegistered.class).tagged("region", "EU");

// Query all events from database, filter in Java
List<Event<CustomerEvent>> filtered = stream.query(EventFilter.matchAll()).stream()
    .filter(filter::matches)
    .toList();
```

Whether an event matches, whether the filter matches everything or nothing, whether any of its items is bounded, and its items themselves are all questions for the filter: `filter.matches(e)`, `filter.isMatchAll()`, `filter.isMatchNone()`, `filter.isBounded()`, `filter.items()`. A query answers only what it adds to its `filter()`: `isBackwards()`, `limit()` and `notAfter()`.

**Disadvantages:**
- All events are read from the database
- Filtering happens in application memory
- Poor performance with large event streams
- Higher memory usage and network transfer

**Important:** While these two approaches are functionally equivalent (they return the same events), the in-process approach suffers from significant performance issues because all events must be retrieved from the database before filtering.

### Hybrid Approach: Coarse Database Filtering + Fine-Grained In-Process Filtering

Sometimes it's beneficial to retrieve a limited set of events from the database and then apply multiple fine-grained filters in Java. This allows you to reuse query results for multiple objectives without running multiple similar database queries:

```java
// Coarse filter: Get all customer events for EU region
List<Event<CustomerEvent>> euEvents = stream.query(EventFilter.forTags("region", "EU"));

// Now apply multiple fine-grained filters in-process
EventFilter registrations = EventFilter.forTypes(CustomerRegistered.class).tagged("region", "EU");
EventFilter premium       = EventFilter.forTags(Tags.of("region", "EU", "premium", "true"));
EventFilter churned       = EventFilter.forTypes(CustomerChurned.class).tagged("region", "EU");

// Reuse the same event list with different filters
long registrationCount = euEvents.stream().filter(registrations::matches).count();
long premiumCount      = euEvents.stream().filter(premium::matches).count();
long churnCount        = euEvents.stream().filter(churned::matches).count();
```

**When to use this approach:**
- You need to apply multiple related filters to the same dataset
- The coarse query retrieves a manageable number of events
- You want to avoid multiple database round-trips
- Fine-grained filtering logic is complex or changes frequently

**When to avoid:**
- The coarse query returns too many events (memory concerns)
- You only need one specific filter (use database-level filtering instead)

This hybrid approach can balance efficiency and flexibility by retrieving a relevant subset once and filtering it multiple ways in memory,
as long as you make sure the number of retrieved events is low enough to do so, or if you page through them (see further).

## Querying all Domain Events in a Stream

The simplest query retrieves all events from a stream using `EventFilter.matchAll()`:

```java
EventStream<CustomerEvent> stream = eventstore.getEventStream(
    EventStreamId.forContext("customer").withPurpose("123"),
    CustomerEvent.class
);

List<Event<CustomerEvent>> allEvents = stream.query(EventFilter.matchAll());
allEvents.forEach(event -> System.out.println(event));
```

**Important**: This approach can lead to performance and memory problems if the stream contains a large number of events. For long streams, page through them (described below) in manageable chunks.

### `query()` Returns a List, Read in Full

This is the single most important thing to know about querying. `query()` hands back a `List`, and **storage has finished reading by the time it returns**: the whole result set is fetched, and every event in it is deserialized and upcast. The type says what a query costs.

Three consequences follow directly:

- **An unbounded query against a large stream is an `OutOfMemoryError`, not a slow read.** There is no back-pressure to arrive at. Bound the read with `EventQuery.of(filter).limit(n)`, which is the limit storage is actually given — nothing applied to the returned list bounds anything.
- **A poison event fails the call itself.** An `EventDeserializationException` comes from `query()`, and a query holding one hands out none of its events. See [Error Handling](/posts/eventstore-error-handling/).
- **Nothing needs closing.** No database resource is held open behind the list. (`EventSource.close()` is about subscriptions, not queries.)

```java
// reads everything matching into heap before returning
List<Event<CustomerEvent>> all = stream.query(EventFilter.matchAll());

// reads 500 stored events
List<Event<CustomerEvent>> page = stream.query(EventQuery.of(EventFilter.matchAll()).limit(500));
```

`.stream()` is one call away when you want stream operations over events already read.

**A full replay is a loop, not one unbounded query.** `Projector` already reads in batches of 500, carrying a cursor between them, and is the right tool for a stream of unknown size. By hand, page with `page(q.limit(n), cursor)` — see [Paging Through a Stream](#paging-through-a-stream). The unlimited path exists for callers who know their result set is small, or genuinely want it all at once; it is not a way to process a large stream incrementally.

> On PostgreSQL, deserialization is roughly 2 µs per event, and on an ordinary 500-event page it is most of the wait — more than the database's own share. Bounding a read is worth more than it looks, and tuning the database is the wrong first move for a read returning thousands of events. See [Stream Design and Performance](/posts/eventstore-stream-design-and-performance/).
{: .prompt-info }

## Querying Domain Events on Type and Tags

Events can be filtered by combining event type filters with tags. The matching semantics are:

- **Event Types**: The event must match **any** of the specified types (OR condition)
- **Tags**: The event must contain **all** specified tags (AND condition)

Note that events can have additional tags beyond those specified in the filter—the filter only requires that the specified tags are present.

```java
// Match any event type
EventFilter anyType = EventFilter.forTags("customer", "123");

// Match a single type
EventFilter singleType = EventFilter.forTypes(CustomerChurned.class);

// Match multiple types (OR condition)
EventFilter multipleTypes = EventFilter.forTypes(
    CustomerRegistered.class,
    CustomerNameChanged.class,
    CustomerChurned.class
);

// Match a whole sealed hierarchy
EventFilter everyCustomerEvent = EventFilter.forTypes(CustomerEvent.class);

// Multiple tags (ALL must be present)
EventFilter multipleTags = EventFilter.forTags(Tags.of("customer", "123", "region", "EU", "priority", "high"));
// Matches events that have AT LEAST these three tags
// (events can have additional tags)
```

> **Tag matching is exact containment, never key-prefix.** A filter on `Tag.of("customer")` does not return events tagged `customer:123` — see [What a Tag May Contain](/posts/eventstore-appending-events/#what-a-tag-may-contain).
{: .prompt-info }

## Paging Through a Stream

For large event streams, read in pages. `page(query, cursor)` answers an `EventPage`: the events, plus what it took from storage to produce them.

```java
EventQuery query = EventQuery.of(EventFilter.matchAll()).limit(500);
EventReference cursor = null;
EventPage<CustomerEvent> page;

do {
    page = stream.page(query, cursor);
    page.events().forEach(event -> processEvent(event));
    cursor = page.lastStoredEventReference().orElse(null);
} while ( page.storedEventCount() == 500 );   // a page shorter than its limit is the last one
```

| `EventPage` | Meaning |
|---|---|
| `events()` | the events read, upcast and filtered, in the query's direction |
| `storedEventCount()` | how many *stored* events were read to produce them — what the limit counts |
| `lastStoredEventReference()` | the reference of the last stored event read, whole: the cursor of the next page |
| `isExhausted()` | no stored event was read at all — past the cursor, the stream is empty |

**Why a page and not just the events.** A limit counts stored events, and an [upcaster](/posts/eventstore-defining-events/#multi-event-upcasting) may turn one stored event into several or into none. So a page holding *no* events may sit in the middle of a stream, and the reference to continue from may be on no event returned. Paging on `events().getLast().reference()` gets both of those wrong; the page's own account gets them right. `isExhausted()` is not the same as `events().isEmpty()` for exactly that reason.

`query(query, cursor)` is the same read without that account, read whole exactly as a page is. Where upcasting cannot produce more or fewer events than were stored, it is enough:

```java
List<Event<CustomerEvent>> next = stream.query(EventQuery.of(EventFilter.matchAll()).limit(100), lastRef);
```

The cursor tells the store where to start. For forward queries it acts as an "after" cursor; for backward queries as a "before" cursor. It doesn't affect which events match the query, only where the scan begins.

**What a cursor is for.** Most reads need none: a decider pins its read with [`notAfter`](#bounding-a-read-notafter), and a read model or any other reader working through a long history is driven by a `Projector`, which keeps the cursor itself. The cursor overloads are for what is left to application code — *what is new since a reference I already hold*: a monitor polling a stream for what was appended since it last looked, or a test collecting what a command appended after its fixture. Give such a query a limit and continue from the last event it returned; `query(filter, cursor)` exists for the case where what follows the cursor is known to be little. The cursor is where an iteration stands, not part of what is asked, which is why it is an argument of the read and not a property of the `EventQuery`.

### What a Limit Actually Counts

**`limit(n)` means "read n stored events", and it is pushed into the storage query** — a SQL `LIMIT` on PostgreSQL, a short-circuiting `Stream.limit` in memory. It is not applied to the result after the fact, which is exactly what makes it bound memory as well as output.

The limit is a property of the query, and nothing else: no read takes a `Limit` beside the query's own. A cursor does not change it — `query(q.limit(500), cursor)` and `page(q.limit(500), cursor)` read 500, the same as `query(q)` would with the limit set on `q`. A query with no limit reads to the end of a stream, deliberately.

**Without upcasting, n stored events are n events back. With it, they are not.** An upcaster may turn one stored event into several or into none, and the limit is spent before it runs:

```java
// over an event that upcasts into two, this returns TWO events —
// having read exactly one stored event
stream.query(EventQuery.of(EventFilter.matchAll()).limit(1));

// over an event that upcasts into none, it returns ZERO —
// also having read exactly one stored event
```

Trimming the surplus would return a fragment of a stored event and leave a cursor pointing into its middle, so the store does not do it. Where you need exactly n events, take a `subList` of the returned list — cheap, since those events are already read:

```java
List<Event<CustomerEvent>> read = stream.query(EventQuery.of(EventFilter.matchAll()).limit(10));
List<Event<CustomerEvent>> exactlyTen = read.subList(0, Math.min(10, read.size()));
```

`Projector` counts stored events for the same reason.

## Querying backwards

Backward queries return events in reverse chronological order (newest first). The direction is set on the `EventQuery` itself using `backwards()`, and a limit can be set using `limit()`:

```java
// Get the last 10 events
List<Event<CustomerEvent>> recentEvents = stream.query(
    EventQuery.of(EventFilter.matchAll()).backwards().limit(10)
);

// Find the last CustomerRegistered event
Optional<Event<CustomerEvent>> lastRegistration = stream.query(
    EventQuery.of(EventFilter.forTypes(CustomerRegistered.class)).backwards().limit(1)
).stream().findFirst();
```

### Backward Pagination

```java
EventQuery backwards = EventQuery.of(EventFilter.matchAll()).backwards().limit(100);
EventReference beforeRef = null;
EventPage<CustomerEvent> page;

do {
    page = stream.page(backwards, beforeRef);

    // Process events (already in reverse order)
    page.events().forEach(event -> processEvent(event));

    // Continue before the oldest stored event of this page
    beforeRef = page.lastStoredEventReference().orElse(null);
} while ( page.storedEventCount() == 100 );
```

## The Head of a Stream

`head()` answers the reference of the newest stored event in the stream, without reading it:

```java
Optional<EventReference> head = stream.head();
```

It is not the same as the `backwards().limit(1)` query idiom, and is better at the job that idiom was usually doing:

- **It never deserializes, upcasts or decrypts.** A head this stream cannot map, one that upcasts into nothing, or one holding personal data under a key store that is down cannot make it fail or lie. The typed query fails on the first, reads the second as an empty stream, and pays a key-store round trip for the third.
- **It is what a query would see.** On PostgreSQL it sits behind the same visibility barrier as every read, so it never runs ahead of the reads it bounds.
- **It names a stored event, whole.** A boundary at it includes every event that stored event upcasts into.
- **An absent head is an empty stream**, and that is a valid answer — "I decided on nothing" — never to be replaced by some other reference.

Its main use is to pin a consistency boundary *before* a decision is read: bound every read with `notAfter(head)` and hand the same reference to `AppendCriteria`. See [Optimistic Locking](/posts/eventstore-appending-events/#optimistic-locking).

## Bounding a Read: `notAfter`

`EventQuery.notAfter(reference)` reads nothing recorded after a reference. It is the **read bound**: a property of *how* a filter is read, not of which events are relevant. Its main use is to pin every read of a decision at one moment — the head taken before reading — and it is equally what reconstructs how things stood at any past moment:

```java
EventFilter customer123 = EventFilter.forTags("customer", "123");

// Get all events, and pick a moment in time
List<Event<CustomerEvent>> allEvents = stream.query(customer123);
EventReference momentInTime = allEvents.get(5).reference(); // 6th event

// Read the events up to that moment
List<Event<CustomerEvent>> pastEvents = stream.query(EventQuery.of(customer123).notAfter(momentInTime));
// Returns only the first six events
```

This enables time-travel queries to reconstruct how a projection looked at any point in history:

```java
// Reconstruct customer state as it was at a stored reference
CustomerSummary historicalState = new CustomerSummary("123");
Projector.from(stream).into(historicalState).build().runUntil(checkpoint);
```

`notAfter` replaces any read bound already set, and `notAfter(null)` removes it.

### How the Read Bound Behaves

Four properties are worth being precise about:

**It is inclusive.** The event named by the reference is returned.

**It is direction-independent.** The bound is compared over the total `(tx, position)` order of *stored* events, not against the direction of travel, so `.backwards()` returns exactly the same events as forward — just newest first. A backwards read starts at the bound and walks into history; it never reads what was recorded after it:

```java
EventQuery upTo = EventQuery.of(customer123).notAfter(checkpoint);

List<Event<CustomerEvent>> forward  = stream.query(upTo);
List<Event<CustomerEvent>> backward = stream.query(upTo.backwards());
// same events, opposite order
```

**It names a stored event, whole.** The `index` on a reference distinguishes the events one stored event upcasts into, and a bound compares stored events: every event the stored event at the bound upcasts into is at or before it, whatever its index. A reference obtained without upcasting — `head()`, a bookmark read back from PostgreSQL — therefore bounds a typed read without cutting the newest stored event in pieces.

**It never reaches a consistency boundary.** `AppendCriteria.of(query, head)` takes the query's *filter* only; its direction, limit and read bound say how the facts were read, not which facts are relevant. So the query a decider read with is exactly the one to hand to the append — the read is bounded at the head, the check looks past it. See [Optimistic Locking](/posts/eventstore-appending-events/#optimistic-locking).

Note that the ordering compared is the tuple, not the position alone. Positions and transactions are assigned independently, so an event can hold a lower position and a higher transaction than one that committed before it. Comparing positions would silently drop such events.

## Bounding What Is Relevant: `after` and `until`

Some decisions are about part of the history by their nature: *the stock movements after the last count*, *the invoices recorded up to the year close*. That is a **business bound**, and it lives on the filter, per item:

```java
EventFilter movements = EventFilter.forTypes(StockAdded.class, StockPicked.class)
                                   .tagged("product", product)
                                   .after(lastCount);                 // exclusive

EventFilter invoices  = EventFilter.forTypes(InvoiceIssued.class)
                                   .tagged("customer", customer)
                                   .until(yearClosed);                // inclusive

// one read over both, each item keeping its own bounds
List<Event<ShopEvent>> facts = stream.query(movements.or(invoices));
```

- **`after(ref)` is exclusive, `until(ref)` is inclusive**, both in recording order over stored events, and both select the same events in either direction.
- **They apply to every item of the filter they are called on**, and replace whatever bound an item had; `null` removes it. `afterIfLater(ref)` and `untilIfEarlier(ref)` *narrow* instead: an item bounded tighter keeps its own bound.
- **A bound set before a union stays with the items it was set on.** That is what lets one read unite facts bounded at different points of the history — something a single read bound cannot express.
- On a match-all filter they produce one item of any type bounded that way; a match-none filter stays match-none.

Because item bounds are part of what is relevant, they *do* reach a consistency boundary — sensibly. When an `AppendCriteria` is made, each item's bounds are judged against its expected reference: an item `until` a reference at or before it can match nothing appended later and is left out of the check, and an `after` at or before it is dropped, since everything after the expected reference is after it too. See [Appending Events](/posts/eventstore-appending-events/#items-bounded-before-the-reference).

> **`notAfter` or `until`?** If the bound is *where you happened to read up to* — the head pinned for a decision, a checkpoint — it is a read bound: `EventQuery.notAfter`. If it is *part of what the decision is about* — a period closed, a savepoint taken — it belongs to the filter: `EventFilter.until` / `after`.
{: .prompt-tip }

## Querying by Event ID

A specific event can be retrieved directly by its `EventId`:

```java
EventId eventId = EventId.of("550e8400-e29b-41d4-a716-446655440000");

Optional<List<Event<CustomerEvent>>> found = stream.getEventById(eventId);

found.ifPresent(events -> events.forEach(e ->
    System.out.println("Found event: " + e.data() + " at " + e.reference())));
```

`getEventById` answers in two levels:

- **The `Optional`** says whether *this stream* holds a stored event with that id. An id the storage does not hold, or holds in a stream this one does not read across, is **absent**.
- **The list** is what that stored event reads as through the stream's mappings: one event normally; several, each with the same id/position/tx and a distinct index, when a legacy event upcasts into several; or **none**, when it upcasts into nothing — present with an empty list.

So `isPresent()` is the presence check, on a typed stream and a raw one alike, and an upcast-to-nothing legacy event is not reported as missing. Like `query()`, it throws an `EventDeserializationException` for a stored event the stream cannot read.

## Raw Streams: Reading Without Domain Classes

`getRawEventStream(id)` opens a stream with no event root classes, so no type mapping. Every stored event reads as the JSON document it is stored as:

```java
EventSource<String> raw = eventstore.getRawEventStream(EventStreamId.forContext("customer"));

for ( Event<String> event : raw.query(EventQuery.of(EventFilter.matchAll()).limit(100)) ) {
    System.out.println(event.type() + " " + event.data());   // data() is the stored JSON document
}
```

- **`data()` is a `String`**: the stored document, parsed by nothing on the way out. It is the same document that was appended, though not necessarily byte for byte — PostgreSQL hands back its `jsonb` rendering. A caller that wants to look inside parses it with the JSON library of its choice.
- **Nothing is upcast**, so a legacy event comes back under its stored type in its stored shape. **Nothing is decrypted**, so a [`Shreddable`](/posts/eventstore-erasing-personal-data/) value comes back as its sealed envelope.
- **It is an `EventSource`, not an `EventStream`**, because a raw stream cannot append: an append is admitted only for a type the stream maps, and a raw stream maps none. Query, page, head, `getEventById`, subscriptions and bookmarks all work, over any stream id, concrete or wildcard.

What it is for: reading the stored event an `EventDeserializationException` names, following every append in a store, and the presence check before an [import](/posts/eventstore-importing-events/) — without the domain classes, with no mapping that could fail on the way.

```java
// the event a typed stream cannot read, by the reference its exception carries
EventSource<String> everything = eventstore.getRawEventStream(EventStreamId.anyContext());
String json = everything.getEventById(reference.id()).orElseThrow().getFirst().data();
```

A stream deliberately typed *wider* than one hierarchy, and able to append, is not a raw stream: open it with the `Set` overload and the roots it should carry — `eventstore.getEventStream(id, Set.of(CustomerEvent.class, OrderEvent.class))`, typed as you assign it, `EventStream<Object>` for instance.

## Querying over EventStreams

EventStreams can be queried across multiple contexts or purposes using wildcard stream identifiers. **A wildcard stream is a source, not a sink**: it reads across every stream it matches and refuses `append`, since an event is stored in exactly one stream. Open the stream you want to write to by its own id.

### Query across all purposes in a context

```java
// Get all events for all customers
EventStream<CustomerEvent> allCustomers = eventstore.getEventStream(
    EventStreamId.forContext("customer").anyPurpose(),
    CustomerEvent.class
);

List<Event<CustomerEvent>> allCustomerEvents = allCustomers.query(EventQuery.of(EventFilter.matchAll()).limit(500));
```

> Where a context is split into a stream per entity, a context-wide read like this is a cross-entity read. It is served by its own index on PostgreSQL, so it pages efficiently in order — but reading **one** entity this way, by tag through the wildcard, rather than through its own stream, gives up everything the split bought. See [Stream Design and Performance](/posts/eventstore-stream-design-and-performance/).
{: .prompt-warning }

### Query across all contexts

```java
// Get events from any context with a specific purpose
EventSource<String> specificPurpose = eventstore.getRawEventStream(
    EventStreamId.anyContext().withPurpose("analytics")
);

List<Event<String>> events = specificPurpose.query(EventQuery.of(EventFilter.matchAll()).limit(500));
```

### Query across all contexts and purposes

```java
// Every event in the entire event store, typed by the hierarchies you register
EventStream<Object> everything = eventstore.getEventStream(
    EventStreamId.anyContext(),
    Set.of(CustomerEvent.class, OrderEvent.class)
);

List<Event<Object>> page = everything.query(EventQuery.of(EventFilter.matchAll()).limit(500));
```

**Use case example**: Global event monitoring or cross-context analytics — raw, so no event type can fail to map:

```java
// Find all events tagged with a specific correlation ID across the entire store
EventSource<String> global = eventstore.getRawEventStream(EventStreamId.anyContext());

List<Event<String>> correlated = global.query(EventFilter.forTags("correlationId", "abc-123"));
```

`EventStreamId.covers(other)` says whether an id's scope contains a stream — it is what scopes every read, and what decides which subscribers a notification is relevant to.

## Querying with legacy Events

When a stream is configured with legacy event types, legacy events are transparently upcast during queries. Application code only needs to work with current event types:

```java
// Current event definitions
sealed interface CustomerEvent {
    record CustomerRegisteredV2(Name name, Email email) implements CustomerEvent {}
    record CustomerRenamed(Name name) implements CustomerEvent {}
}

// Define legacy events separately
sealed interface CustomerLegacyEvent {
    @LegacyEvent(upcaster = CustomerRegisteredUpcaster.class)
    record CustomerRegistered(String name) implements CustomerLegacyEvent {}
}

// Get stream specifying both current and legacy types
EventStream<CustomerEvent> stream = eventstore.getEventStream(
    EventStreamId.forContext("customer").withPurpose("123"),
    CustomerEvent.class,
    CustomerLegacyEvent.class
);

// Query by current type - includes upcast legacy events
List<Event<CustomerEvent>> registrations = stream.query(
    EventFilter.forTypes(CustomerEvent.CustomerRegisteredV2.class)
);

// All events are typed as CustomerEvent (never CustomerLegacyEvent)
for (Event<CustomerEvent> event : registrations) {
    CustomerEvent currentEvent = event.data();
    // Legacy CustomerRegistered events are automatically upcast to CustomerRegisteredV2
}
```

**Key points:**
- Queries use **current event types** only; a filter naming a legacy type is refused with `IllegalArgumentException`
- Legacy events whose upcast chain ends in the queried type are automatically included, however many versions back
- The upcasting is transparent—application code never sees legacy event types
- `event.type()` is the current type; `event.storedType()` names the legacy type it was read from
- When a legacy event is upcast into multiple current events, all appear in query results in their correct position with distinct index values
- When a legacy event is upcast to zero events (filtering), it is silently skipped

## Complex Event Filters

For advanced scenarios, you can combine filters using `or()`. This creates a UNION of filters, allowing you to retrieve events that match any of several different patterns — in a single read.

### Understanding Filter Matching Semantics

An `EventFilter` is a union of `EventFilterItem`s, and matching combines AND and OR logic:

**Within a single item:**
- **Event Types**: The event must match **ANY** of the specified types (OR condition)
- **Tags**: The event must contain **ALL** specified tags (AND condition)
- **Bounds**: The event must lie within the item's `after`/`until`, if it has any

**Across multiple items:**
- If **ANY** item matches, the event matches the overall filter (OR condition)

This gives you powerful flexibility to express complex selection criteria.

### Basic Filter Combination

Combine two filters to match events that satisfy either:

```java
// Filter 1: All CustomerRegistered events
EventFilter newCustomers = EventFilter.forTypes(CustomerRegistered.class);

// Filter 2: All events for VIP customers
EventFilter vipActivity = EventFilter.forTags("customerType", "VIP");

// Combined: CustomerRegistered events OR any VIP customer events
EventFilter combined = newCustomers.or(vipActivity);

List<Event<CustomerEvent>> events = stream.query(combined);
```

The combined filter will return:
- All `CustomerRegistered` events (regardless of tags)
- All events (any type) with the tag `customerType=VIP`

No duplicates are returned: events matching multiple items in the filter are returned once.
As always, events are returned in stream order.

### Narrowing a Union

`tagged(...)` distributes over a union: `a.or(b).tagged(t)` is `a.tagged(t).or(b.tagged(t))`. That makes it easy to scope a combination of facts to one entity:

```java
EventFilter courseFacts = EventFilter.forTypes(CourseDefined.class, CourseCapacityUpdated.class)
                                     .or(EventFilter.forTypes(StudentSubscribedToCourse.class))
                                     .tagged("course", "CS101");
```

A match-all narrowed by a tag is `forTags(tag)`; a match-none stays match-none. The same distribution holds for `after`, `until` and their `IfLater`/`IfEarlier` forms: called on a union, they bound every item of it.

### Combining Filters with Different Types and Tags

Create complex selection criteria by combining filters with different event types and tag requirements:

```java
// Events related to a specific student
EventFilter studentEvents = EventFilter.forTypes(StudentRegistered.class, StudentSubscribedToCourse.class)
                                       .tagged("student", "S123");

// Events related to a specific course
EventFilter courseEvents = EventFilter.forTypes(CourseDefined.class, CourseCapacityUpdated.class, StudentSubscribedToCourse.class)
                                      .tagged("course", "CS101");

// Combined: All events relevant to this student-course interaction
EventFilter relevantFacts = studentEvents.or(courseEvents);
```

The combined filter matches events where **any** of these conditions are true:
- Event is `StudentRegistered` OR `StudentSubscribedToCourse` AND has tag `student=S123`
- Event is `CourseDefined` OR `CourseCapacityUpdated` OR `StudentSubscribedToCourse` AND has tag `course=CS101`

Notice that `StudentSubscribedToCourse` events with **either** tag will be included.

### Combination Rules

`or` on filters always succeeds. The items of both sides are kept side by side, each with its own types, tags **and bounds**, so filters narrowed to different entities, or bounded at different points of the history, unite freely. A match-all on either side makes the union match-all; a match-none contributes nothing.

There is **no union of queries**. A direction, a limit or a read bound cannot be united meaningfully: two `backwards().limit(1)` savepoint queries each ask for "the most recent event matching *me*"; united into `(A OR B) limit 1` they would ask for "the most recent event matching either", which answers neither. So unite the *filters*, and wrap the union in one query where the read needs a direction, limit or bound — or run the queries separately when each needs its own limit.

### Practical Use Case: Dynamic Consistency Boundary

Filter combination is particularly useful for Dynamic Consistency Boundaries where business decisions depend on multiple types of facts:

```java
public class SubscribeToCourseCommand {
    private final String studentId;
    private final String courseId;

    public EventFilter relevantFacts() {
        // Student-specific facts
        EventFilter student = EventFilter.forTypes(StudentRegistered.class, StudentSubscribedToCourse.class)
                                         .tagged("student", studentId);

        // Course-specific facts
        EventFilter course = EventFilter.forTypes(CourseDefined.class, CourseCapacityUpdated.class, StudentSubscribedToCourse.class)
                                        .tagged("course", courseId);

        // All relevant facts for this business decision
        return student.or(course);
    }

    public void execute(EventStream<LearningEvent> stream) {
        EventFilter relevant = relevantFacts();

        // Pin the boundary, then load current state as of that boundary
        EventReference head = stream.head().orElse(null);
        List<Event<LearningEvent>> facts = stream.query(EventQuery.of(relevant).notAfter(head));

        // Make business decision based on facts
        SubscriptionDecision decision = SubscriptionDecision.from(facts);
        if (decision.hasCapacity()) {
            // Append new event with optimistic locking: the same filter, the head as reference
            stream.append(
                AppendCriteria.of(relevant, head),
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

> On PostgreSQL, an OR-of-facts read like this one is not served by the tag index — the index answers a single tag containment, not a disjunction — so it walks the stream in order and filters. Keep the number of OR-ed items small on hot paths. A union whose items carry *different* bounds is read as one ordered branch per item, merged in order, so each item uses its own range instead of one walk over the widest; items sharing their bounds (every filter without any) keep the single statement.
{: .prompt-info }

### Matching Examples

To clarify the matching semantics, consider these examples:

**Example 1: Simple combination**

```java
EventFilter f = EventFilter.forTypes(CustomerRegistered.class).tagged("region", "EU")
                           .or(EventFilter.forTypes(OrderPlaced.class).tagged("priority", "high"));
```

This matches events where:
- Event type is `CustomerRegistered` AND has tag `region=EU`, **OR**
- Event type is `OrderPlaced` AND has tag `priority=high`

**Example 2: Multiple types and tags per item**

```java
EventFilter f = EventFilter.forTypes(CustomerRegistered.class, CustomerUpdated.class)
                           .tagged("region", "EU").tagged("verified", "true")
                           .or(EventFilter.forTypes(OrderPlaced.class, OrderShipped.class).tagged("priority", "high"));
```

This matches events where:
- Event type is `CustomerRegistered` OR `CustomerUpdated` AND has BOTH tags `region=EU` and `verified=true`, **OR**
- Event type is `OrderPlaced` OR `OrderShipped` AND has tag `priority=high`

**Example 3: Any type with specific tags**

```java
EventFilter f = EventFilter.forTags("correlationId", "abc-123")
                           .or(EventFilter.forTypes(ErrorOccurred.class));
```

This matches events where:
- ANY event type with tag `correlationId=abc-123`, **OR**
- Event type is `ErrorOccurred` (regardless of tags)

This pattern is useful for debugging: retrieve all events in a specific correlation chain plus any error events.

**Example 4: Items bounded at different points**

```java
EventFilter f = EventFilter.forTypes(StockAdded.class, StockPicked.class).tagged("product", "P1").after(lastCount)
                           .or(EventFilter.forTypes(PriceChanged.class).tagged("product", "P1"));
```

This matches events where:
- Event type is `StockAdded` OR `StockPicked` AND has tag `product=P1` AND was recorded after `lastCount`, **OR**
- Event type is `PriceChanged` AND has tag `product=P1`, whenever it was recorded
