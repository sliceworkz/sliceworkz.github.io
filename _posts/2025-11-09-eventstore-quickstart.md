---
layout: post
toc: true
title: Eventstore Quickstart Guide
description: Gettings started with the Sliceworkz EventStore
icon: fas fa-rocket
categories: [Eventstore Documentation]
tags: [quickstart,eventstore,maven]
pin: true
order: 2
---


This guide will help you get started with the EventStore library for Java. 

EventStore is a DCB-compliant event storage library that provides dynamic consistency boundaries through tag-based event queries and optimistic locking.

# Setup

## Prerequisites

- Java 21 or higher
- Maven 3.6.3 or higher

## Installation - import the BOM

Sliceworkz Eventstore is available in <a href="https://mvnrepository.com/artifact/org.sliceworkz">maven central</a>.

All EventStore modules are bundled in a Bill-Of-Material pom file.
Add the EventStore BOM to your project pom.xml to manage dependency versions:

```xml
...
<properties>
    <sliceworkz.eventstore.version>0.11.1</sliceworkz.eventstore.version>
</properties>
...
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.sliceworkz</groupId>
            <artifactId>sliceworkz-eventstore-bom</artifactId>
            <version>${sliceworkz.eventstore.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
...
```

### Add dependencies to your project

For development and testing with in-memory storage:

```xml
...
<dependencies>
    <dependency>
        <groupId>org.sliceworkz</groupId>
        <artifactId>sliceworkz-eventstore-api</artifactId>
    </dependency>
    <dependency>
        <groupId>org.sliceworkz</groupId>
        <artifactId>sliceworkz-eventstore-infra-inmem</artifactId>
    </dependency>
</dependencies>
...
```

Every storage backend brings `sliceworkz-eventstore-impl` in at runtime, so `EventStore.on(storage).build()` finds an implementation without you naming one.

For local development with file persistence (events survive restarts without requiring PostgreSQL):

```xml
...
<dependency>
    <groupId>org.sliceworkz</groupId>
    <artifactId>sliceworkz-eventstore-infra-inmem-fs</artifactId>
</dependency>
...
```

For production use with PostgreSQL, you can replace the inmemory-storage with this one. The JDBC driver is declared `provided` by the backend, so add it yourself, at the version your platform ships:

```xml
...
<dependency>
    <groupId>org.sliceworkz</groupId>
    <artifactId>sliceworkz-eventstore-infra-postgres</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <version>42.7.13</version>
</dependency>
...
```

> **What lands on your classpath.** `sliceworkz-eventstore-api` brings SLF4J and nothing else of note — no metrics library, and no Jackson beyond the optional `jackson-annotations` artifact Jackson 2 and 3 share. Jackson 3 (`tools.jackson.*`) arrives with the implementation and the backends; it is a different groupId and package from Jackson 2, so an application on Jackson 2 runs both side by side without conflict. Every jar declares an `Automatic-Module-Name` (`org.sliceworkz.eventstore`, `org.sliceworkz.eventstore.infra.postgres`, …), so `requires` clauses keep working on the module path whatever the jar file is called.
{: .prompt-info }

> **PostgreSQL version support.** The oldest supported PostgreSQL is **16**; **18+** is what the library is built around, using the native server-side `uuidv7()` function for event ids. On 16 and 17 those ids are generated in Java instead, which requires the optional `com.github.f4b6a3:uuid-creator` dependency to be added explicitly to your application. The right implementation is selected automatically at startup from the connected server's major version. See the [PostgreSQL EventStorage guide](/posts/eventstore-configuring-postgresql-storage/#postgresql-version-support) for the dependency snippet and details.
{: .prompt-info }

For testing your application against the event store, add the testing module in `test` scope — see [Testing Your Application](/posts/eventstore-testing/):

```xml
...
<dependency>
    <groupId>org.sliceworkz</groupId>
    <artifactId>sliceworkz-eventstore-testing</artifactId>
    <scope>test</scope>
</dependency>
...
```


## Quick Start Example

Using the eventstore in basic scenarios is quite straightforward.  We'll go over the different aspects underneath.

A good starting point for further discovery is to look at some small <a href="https://github.com/sliceworkz/eventstore/tree/develop/sliceworkz-eventstore-examples/src/main/java/org/sliceworkz/eventstore/examples">example applications</a> that come with the library

### 1. Define Your Domain Events

At the basis of your event-sourced applications will be the Event definitions.  The recommended way to express type-safe event hierarchies is to use sealed interfaces with record implementations:

```java
public sealed interface CustomerEvent {
    record CustomerRegistered(String name) implements CustomerEvent { }
    record CustomerNameChanged(String name) implements CustomerEvent { }
    record CustomerChurned() implements CustomerEvent { }
}
```

### 2. Create an EventStore

Creating an Eventstore requires selecting an EventStorage.

For development/testing with in-memory storage:

```java
import org.sliceworkz.eventstore.EventStore;
import org.sliceworkz.eventstore.infra.inmem.InMemoryEventStorage;

EventStore eventstore = InMemoryEventStorage.newBuilder().buildStore();
```

For local development with file persistence:

```java
import org.sliceworkz.eventstore.EventStore;
import org.sliceworkz.eventstore.infra.inmem.fs.InMemoryFsEventStorage;

EventStore eventstore = InMemoryFsEventStorage.newBuilder()
    .directory("eventstore-data")
    .buildStore();
```

This stores events and bookmarks as human-readable JSON files on disk. Events are kept in memory for fast access and automatically reloaded on restart. The default directory is `eventstore-data`.

For production with PostgreSQL:

```java
import org.sliceworkz.eventstore.EventStore;
import org.sliceworkz.eventstore.infra.postgres.PostgresEventStorage;

EventStore eventstore = PostgresEventStorage.newBuilder()
    .name("mystore")
    .prefix("myapp_")
    .buildStore();
```

> **Note:** PostgreSQL needs connection settings — a `db.properties` file, or a `DataSource` you pass in (see [PostgreSQL Configuration](#postgresql-configuration) below). Have a look at the example <a href="https://github.com/sliceworkz/eventstore/tree/develop/sliceworkz-eventstore-infra-postgres/src/main/quickstart">quickstart configuration</a> for a template.

> The default `DatabaseInitMode` is `ENSURE`, which creates missing tables, indexes, functions and triggers idempotently and leaves your data alone. `.recreateDatabase()`{:.filepath} (`DatabaseInitMode.RECREATE`) drops and recreates everything, **deleting every event** — handy for a test run, never for anything else. For production, the recommended way of working is to connect the DB with a user that only has DML rights, use `.validateDatabase()`{:.filepath} to verify the schema, and create the database schema upfront with the DDL found in the <a href="https://github.com/sliceworkz/eventstore/tree/develop/sliceworkz-eventstore-infra-postgres/src/main/quickstart">quickstart configuration</a>
{: .prompt-warning }


### 3. Get an EventStream

An Eventstore gives access to EventStreams, which are identified by a 2-part (context and purpose).
It also takes the sealed interface class as a parameter to allow typed access to the events in the stream. That class fixes the stream's type parameter: `getEventStream(id, CustomerEvent.class)` is an `EventStream<CustomerEvent>`, and assigning it to an `EventStream<OrderEvent>` does not compile.

All events in the EventStore are of course organised sequentially, an EventStream is actually a subset of the Events in the overall Event history managed by the Eventstore.


This example opens a stream in the `customer` context and gives typed access to the CustomerEvents in it:

```java
import org.sliceworkz.eventstore.stream.EventStream;
import org.sliceworkz.eventstore.stream.EventStreamId;

EventStreamId streamId = EventStreamId.forContext("customer").withPurpose("123");
EventStream<CustomerEvent> stream = eventstore.getEventStream(streamId, CustomerEvent.class);
```

> **A stream is not an aggregate.** It is tempting to read this as "the stream for customer 123" and to rebuild one object per entity from it — the classic aggregate. Resist that. Consistency comes from the **tags on the events**, not from the boundary of the stream, which is what lets a single decision span facts about several entities at once. The [Dynamic Consistency Boundary](#dynamic-consistency-boundary---optimistic-locking-with-tags) section below shows the shape to aim for, and it is the one this library is built around.
{: .prompt-warning }

> **The recommended design is one stream per bounded context**, entities told apart by tags. A conditional append only checks the stream it appends to, so the facts one decision spans have to share a stream — and in a single context stream they always do. Use a purpose to separate *kinds* of stream within a context; a stream per entity is an optimisation with a steep price, weighed in [Stream Design and Performance](/posts/eventstore-stream-design-and-performance/).
{: .prompt-info }

### 4. Append Events

You're now ready to append Events to your EventStream.
For now, we'll just add them after whatever Events already exist, without any conditions: no decision was read, so there is no consistency boundary to check.

These are simple appends without optimistic locking:

```java
import org.sliceworkz.eventstore.events.Event;
import org.sliceworkz.eventstore.events.Tags;

stream.append(Event.of(new CustomerRegistered("John"), Tags.none()));
stream.append(Event.of(new CustomerNameChanged("Jane"), Tags.none()));
```

An append without criteria is exactly `stream.append(AppendCriteria.none(), ...)`, which stays valid — use whichever reads better.

### 5. Query Events

Of course, an EventStream can also be queried for the Events that are in it.

Querying all events in the stream:

```java
import org.sliceworkz.eventstore.query.EventQuery;
import java.util.List;

List<Event<CustomerEvent>> allEvents = stream.query(EventQuery.matchAll());
allEvents.forEach(System.out::println);
```

`query()` returns a `List`, read in full: storage has finished reading by the time it comes back. Bound a read over a large stream with `.limit(n)` — see [Querying Events](/posts/eventstore-querying-events/).

Query with filters, in this example only returning Events of a certain type:

```java
List<Event<CustomerEvent>> registrations = stream.query(
    EventQuery.forTypes(CustomerRegistered.class)
);
```

This ends our very basic setup of an eventsourced application with Sliceworkz EventStore.
Before diving into some more advanced scenarios, let's explain some of the required concepts.

## A Quick tour of the Core Concepts

### EventStore

The Eventstore is the main entry point for interacting with the event storage system (inmemory or postgres-database). 
It Provides access to EventStreams.

A storage builder's `buildStore()` gives you one directly. When you hold the `EventStorage` yourself, `EventStore.on(storage).build()` turns it into a store.

You would typically create a single EventStore object per application.

### EventStream

A type-safe stream of events identified by an `EventStreamId`. Supports both reading (via `query()`) and writing (via `append()`).

### Event

An immutable record containing the domain event data, tags, reference, timestamp, and stream information. 

### Tags

Key-value pairs attached to events that enable dynamic querying across different event types. This is the foundation of <a href="https://dcb.events">Dynamic Consistency Boundary</a> approach.

### EventQuery

Defines which events to retrieve from storage and how to traverse them. An `EventQuery` wraps an `EventFilter` (matching criteria: event types, tags, and an optional "until" reference) together with a direction (forward or backward) and an optional limit. Build one fluently — `EventQuery.forTypes(CustomerEvent.class).tagged("customer", "123")`, where a sealed root stands for its whole hierarchy — or from its two halves with `EventQuery.forEvents(types, tags)`.

### AppendCriteria

Controls optimistic locking when appending events. Contains an `EventFilter` (the matching criteria extracted from an `EventQuery`) and an optional reference to the last known event. If new matching events exist after the reference, the append fails with `OptimisticLockingException`.


## Dynamic Consistency Boundary - Optimistic Locking with Tags

One of the most powerful features of EventStore is the ability to use tags and optimistic locking to implement Dynamic Consistency Boundaries.

### Scenario: Multiple Customers in One Stream

Instead of one stream per customer, use a single stream with tags to identify each customer:

```java
sealed interface CustomerEvent {
    record CustomerRegistered(String id, String name) implements CustomerEvent { }
    record CustomerNameChanged(String id, String name) implements CustomerEvent { }
    record CustomerChurned(String id) implements CustomerEvent { }
}

// Single stream for all customers
EventStreamId streamId = EventStreamId.forContext("customers");
EventStream<CustomerEvent> stream = eventstore.getEventStream(streamId, CustomerEvent.class);

// Append events with customer tags
stream.append(Event.of(new CustomerRegistered("123", "John"), Tags.of("customer", "123")));
stream.append(Event.of(new CustomerRegistered("456", "Alice"), Tags.of("customer", "456")));
```

### Query by Tag

Retrieve events for a specific customer:

```java
import java.util.List;

List<Event<CustomerEvent>> customer123Events = stream.query(
    EventQuery.forTags(Tags.of("customer", "123"))
);
```

### Conditional Append with Optimistic Locking

Ensure no new relevant events exist before appending:

```java
import org.sliceworkz.eventstore.events.EventReference;
import org.sliceworkz.eventstore.stream.AppendCriteria;
import org.sliceworkz.eventstore.stream.OptimisticLockingException;
import java.util.List;

EventQuery customer123 = EventQuery.forTags(Tags.of("customer", "123"));

// 1. Pin the boundary at the stream head, BEFORE reading.
//    An absent head is an empty stream -- a valid boundary, so no special case is needed
EventReference head = stream.head().orElse(null);

// 2. Query the relevant facts, bounded at that head
List<Event<CustomerEvent>> events = stream.query(customer123.until(head));

// 3. Make business decision based on events
// ... process events and decide to change name ...

// 4. Append with optimistic lock: the query UNBOUNDED, the head as the expected last event
try {
    stream.append(
        AppendCriteria.of(customer123, head),
        Event.of(new CustomerNameChanged("123", "Jane"), Tags.of("customer", "123"))
    );
} catch (OptimisticLockingException e) {
    // A new fact about customer 123 was appended since the head was taken
    // Retry: take the head again, query again, decide again, append
}
```

`head()` is the reference of the newest stored event in the stream, answered without reading it. Taking it *before* the read gives every read of the decision the same boundary, and gives the lock check a cursor at the stream head — which on PostgreSQL is markedly cheaper to check than the reference of the last relevant event.

> **Hand `AppendCriteria` the query without its `until`.** `AppendCriteria.of(customer123.until(head), head)` — reusing the query the read used — deems nothing after the head relevant, so the check never finds a new fact and every append is admitted. Nothing fails to tell you: optimistic locking is simply off.
{: .prompt-danger }

### The DCB Pattern

This pattern implements the <a href="https://dcb.events/specification">Dynamic Consistency Boundary specification</a>:

1. **Pin** the boundary: take the stream's `head()` (or, for a single read, note the reference of the last relevant event)
2. **Query** relevant events with an `EventQuery`, bounded with `until(head)`
3. **Decide** based on the events retrieved
4. **Append** new events with `AppendCriteria` containing the same query — without the `until` — and that reference
5. If new events matching the query exist after the reference, the append **fails**

This ensures your business decisions are based on complete information and prevents conflicts.
Any new Events that were appended to the Eventstore that wouldn't have influenced your decision are not blocking the append.

## Event Subscriptions

Subscriptions allow your application to be notified of new Events being appended (by another thread, or another process or server in case of postgres/database-storage)

This would be useful to update any read models, for example, and realizes eventual consistency.

An example of subscribing to newly appended events:

```java
import org.sliceworkz.eventstore.events.Event;
import org.sliceworkz.eventstore.events.EventReference;
import org.sliceworkz.eventstore.query.EventQuery;
import org.sliceworkz.eventstore.stream.EventSource;
import org.sliceworkz.eventstore.stream.Subscription;
import java.util.List;
import java.util.concurrent.atomic.AtomicReference;

// A raw, read-only source over every stream: each event's data is the stored JSON document
EventSource<String> everything = eventstore.getRawEventStream(EventStreamId.anyContext());

// Start after the current head
AtomicReference<EventReference> lastSeen = new AtomicReference<>(everything.head().orElse(null));

// Subscribe to new appends. The listener is a functional interface:
// it receives the reference appended to at least, and returns the one it reached.
Subscription subscription = everything.subscribe(atLeastUntil -> {
    List<Event<String>> newEvents = everything.query(EventQuery.matchAll(), lastSeen.get());

    newEvents.forEach(e -> System.out.println(e.type() + " " + e.data()));

    if (!newEvents.isEmpty()) {
        lastSeen.set(newEvents.getLast().reference());
    }
    return lastSeen.get();
});

// ... later: end this one subscription
subscription.close();
```

> **Note:** Notifications only tell you that Events were appended to **at least** a certain reference.  By the time you reach out to query new Events, it is perfectly possible that more have been appended.  Additionally, not all Event adds are notified individually per se.

> Subscribing registers the stream with the storage, which then holds it while it has a live subscription. Close the `Subscription` handle `subscribe(...)` returns, or the stream, when you are done with it — or close the store, which closes them all. See [Lifecycle and Shutdown](/posts/eventstore-lifecycle/).
{: .prompt-info }


## PostgreSQL Configuration

Create a `db.properties` file in your working directory (or at the root of the classpath, e.g. `src/main/resources`):

```properties
db.pooled.url=jdbc:postgresql://<host>/<db>
db.pooled.username=<user>
db.pooled.password=<password>
db.pooled.leakDetectionThreshold=2000
db.pooled.maximumPoolSize=25
db.pooled.datasource.sslmode=require
db.pooled.datasource.channelBinding=require

db.nonpooled.url=jdbc:postgresql://<host>/<db>
db.nonpooled.username=<user>
db.nonpooled.password=<password>
db.nonpooled.leakDetectionThreshold=70000
db.nonpooled.maximumPoolSize=2
db.nonpooled.datasource.sslmode=require
db.nonpooled.datasource.channelBinding=require
```

Keys in a section are HikariCP properties; keys under `datasource.` go to the JDBC driver. The builder looks for the file at a fixed set of locations and never walks into parent directories — see [where `db.properties` is found](/posts/eventstore-configuring-postgresql-storage/#configuring-an-eventstore-managed-datasource-dbproperties). You can also hand it over directly with `.configuration(Path.of(...))` or `.configuration(properties)`.


> **Note:** Eventstore uses up to two different types of connections to your database (pooled and nonpooled).
The nonpooled ones are used to monitor any event appends on the database by another process.
This relies on the Postgres LISTEN/NOTIFY mechanism, which doesn't work on a behind a pgbouncer or other server-side connection pooling mechanism.


Or configure programmatically:

```java
import javax.sql.DataSource;
import com.zaxxer.hikari.HikariDataSource;

HikariDataSource dataSource = new HikariDataSource();
dataSource.setJdbcUrl("jdbc:postgresql://localhost:5432/eventstore");
dataSource.setUsername("postgres");
dataSource.setPassword("postgres");

PostgresEventStorage storage = PostgresEventStorage.newBuilder()
    .dataSource(dataSource)
    .prefix("myapp_")		// if you want your tables prefixed
    .recreateDatabase()		// drop/create the tables, deleting every event - only in DEV/test
    .build();

EventStore eventstore = EventStore.on(storage).build();
```

## Shutting Down

`EventStorage`, `EventStore` and `EventStream` all implement `AutoCloseable`. A store that lives as long as the process needs no explicit close — but one created per tenant, per test or per hot reload does, because the PostgreSQL backend runs monitor threads that keep the whole storage reachable.

```java
try ( EventStore eventstore = PostgresEventStorage.newBuilder().buildStore() ) {
    // ... use the store ...
}   // stops the monitor threads and closes the pools the builder created
```

`buildStore()` returns a store that owns the storage it created, so closing it closes both. When you build a storage yourself and hand it to several stores, closing a store does **not** close the storage — see [Lifecycle and Shutdown](/posts/eventstore-lifecycle/).

## Testing

The `sliceworkz-eventstore-testing` module provides a `given / when / then` fixture over a fresh in-memory store:

```java
@Test
void studentCannotSubscribeTwice() {
    EventStoreFixture.inMemory(EventStreamId.forContext("learning"), LearningEvent.class)
        .given(
            event(new CourseDefined("abc001", "Java basics", 12)).tagged("course", "abc001"),
            event(new StudentSubscribed("123", "abc001")).tagged("course", "abc001"))
        .when(stream -> new Registrations(stream).subscribe("123", "abc001"))
        .expectResult(false)
        .expectNoEventsAppended();
}
```

Or drive an in-memory store directly, if you prefer to write the assertions yourself:

```java
@Test
void testCustomerNameChange() {
    EventStore eventstore = InMemoryEventStorage.newBuilder().buildStore();

    EventStreamId streamId = EventStreamId.forContext("customer").withPurpose("123");
    EventStream<CustomerEvent> stream = eventstore.getEventStream(streamId, CustomerEvent.class);

    stream.append(Event.of(new CustomerRegistered("John"), Tags.none()));
    stream.append(Event.of(new CustomerNameChanged("Jane"), Tags.none()));

    List<Event<CustomerEvent>> events = stream.query(EventQuery.matchAll());

    assertEquals(2, events.size());
    assertEquals("Jane", ((CustomerNameChanged) events.get(1).data()).name());
}
```

See [Testing Your Application](/posts/eventstore-testing/) for the full fixture API, including provoking DCB conflicts deterministically.

## Next Steps

- Read the [DCB Specification](https://dcb.events/specification/) to understand the theoretical foundation
- Explore the `sliceworkz-eventstore-examples` module for more complex scenarios
- Learn about [projections](/posts/eventstore-projecting-events/) and [point-in-time queries](/posts/eventstore-querying-events/#querying-until-a-certain-moment-in-time)
- Consider implementing [event upcasting](/posts/eventstore-defining-events/#approach-2-upcasting) for schema evolution
- Choose your [stream layout](/posts/eventstore-stream-design-and-performance/) with the measured trade-offs in hand
- Report what the store does to your metrics or tracing library through an [observer](/posts/eventstore-observability-micrometer-prometheus-grafana/)
- Understand [which exceptions to retry](/posts/eventstore-error-handling/) before writing your first retry loop
- Plan [store lifecycle and shutdown](/posts/eventstore-lifecycle/) before going to production
- Holding personal data in your events? See [Erasing Personal Data](/posts/eventstore-erasing-personal-data/) before your first append, since it decides how the events are declared
- Moving an existing store to another backend? See [Importing Events Between Stores](/posts/eventstore-importing-events/)

## License

EventStore is licensed under <a href="https://github.com/sliceworkz/eventstore/blob/develop/LICENSE">LGPL v3.0</a>.

## Support

For issues and questions:
- GitHub Issues: [Report an issue](https://github.com/sliceworkz/eventstore/issues)
