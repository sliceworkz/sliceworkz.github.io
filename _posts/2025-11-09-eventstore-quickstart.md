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
    <sliceworkz.eventstore.version>0.2.1</sliceworkz.eventstore.version>
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
    <dependency>
        <groupId>org.sliceworkz</groupId>
        <artifactId>sliceworkz-eventstore-impl</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
...
```

For production use with PostgreSQL, you can replace the inmemory-storage with this one:

```xml
...
<dependency>
    <groupId>org.sliceworkz</groupId>
    <artifactId>sliceworkz-eventstore-infra-postgres</artifactId>
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

For production with PostgreSQL:

```java
import org.sliceworkz.eventstore.EventStore;
import org.sliceworkz.eventstore.infra.postgres.PostgresEventStorage;

EventStore eventstore = PostgresEventStorage.newBuilder()
    .name("mystore")
    .prefix("myapp_")
    .initializeDatabase()
    .buildStore();
```

> **Note:** PostgreSQL requires a `db.properties` file with connection settings.  Have a look at the example <a href="https://github.com/sliceworkz/eventstore/tree/develop/sliceworkz-eventstore-infra-postgres/src/main/quickstart">quickstart configuration</a> for a template.

> The `.initializeDatabase`{:.filepath} call drops and created the necessary tables and indexes.  The recommended way of working is to connect the DB with a user that only has DML rights, and create the database schema upfront with the DDL found in the <a href="https://github.com/sliceworkz/eventstore/tree/develop/sliceworkz-eventstore-infra-postgres/src/main/quickstart">quickstart configuration</a>
{: .prompt-warning }


### 3. Get an EventStream

An Eventstore gives access to EventStreams, which are identified by a 2-part (context and purpose).
It also takes the sealed interfaces class as a parameter to allow typed access to the events in the stream.

All events in the EventStore are of course organised sequentially, an EventStream is actually a subset of the Events in the overall Event history managed by the Eventstore.


This example allows to append and query CustomerEvents in an EventStream dedicated to that Customer:

```java
import org.sliceworkz.eventstore.stream.EventStream;
import org.sliceworkz.eventstore.stream.EventStreamId;

EventStreamId streamId = EventStreamId.forContext("customer").withPurpose("123");
EventStream<CustomerEvent> stream = eventstore.getEventStream(streamId, CustomerEvent.class);
```

> There are multiple other approaches to working with EventStreams, but this basic approach already allows you to implement a classical eventsourced Aggregate.
{: .prompt-info }

### 4. Append Events

You're now ready to append Events to your EventStream.
For now, we'll just add them after whatever Events already exists, without any conditions (AppendCriteria).

These are simple appends without optimistic locking:

```java
import org.sliceworkz.eventstore.events.Event;
import org.sliceworkz.eventstore.events.Tags;
import org.sliceworkz.eventstore.stream.AppendCriteria;

stream.append(
    AppendCriteria.none(),
    Event.of(new CustomerRegistered("John"), Tags.none())
);

stream.append(
    AppendCriteria.none(),
    Event.of(new CustomerNameChanged("Jane"), Tags.none())
);
```

### 5. Query Events

Of course, an EventStream can also be queried for the Events that are in it.

Querying all events in the stream:

```java
import org.sliceworkz.eventstore.query.EventQuery;
import java.util.stream.Stream;

Stream<Event<CustomerEvent>> allEvents = stream.query(EventQuery.matchAll());
allEvents.forEach(System.out::println);
```

Query with filters, in this example only returning Events of a certain type:

```java
import org.sliceworkz.eventstore.query.EventTypesFilter;

Stream<Event<CustomerEvent>> registrations = stream.query(
    EventQuery.forEvents(
        EventTypesFilter.of(CustomerRegistered.class),
        Tags.none()
    )
);
```

This ends our very basic setup of an eventsourced application with Sliceworkz EventStore.
Before diving into some more advanced scenarios, let's explain some of the required concepts.

## A Quick tour of the Core Concepts

### EventStore

The Eventstore is the main entry point for interacting with the event storage system (inmemory or postgres-database). 
It Provides access to EventStreams.

You would typically create a single EventStore object per application.

### EventStream

A type-safe stream of events identified by an `EventStreamId`. Supports both reading (via `query()`) and writing (via `append()`).

### Event

An immutable record containing the domain event data, tags, reference, timestamp, and stream information. 

### Tags

Key-value pairs attached to events that enable dynamic querying across different event types. This is the foundation of <a href="https://dcb.events">Dynamic Consistency Boundary</a> approach.

### EventQuery

Defines which events to retrieve from storage based on event types and tags. 
Supports filtering on specific Event types, Events with certain Tags, as well as point-in-time queries via an optional "until" reference.

### AppendCriteria

Controls optimistic locking when appending events. Contains an `EventQuery` and an optional reference to the last known event. If new matching events exist after the reference, the append fails with `OptimisticLockingException`.


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
stream.append(
    AppendCriteria.none(),
    Event.of(
        new CustomerRegistered("123", "John"),
        Tags.of("customer", "123")
    )
);

stream.append(
    AppendCriteria.none(),
    Event.of(
        new CustomerRegistered("456", "Alice"),
        Tags.of("customer", "456")
    )
);
```

### Query by Tag

Retrieve events for a specific customer:

```java
import java.util.List;

List<Event<CustomerEvent>> customer123Events = stream.query(
    EventQuery.forEvents(
        EventTypesFilter.any(),
        Tags.of("customer", "123")
    )
).toList();
```

### Conditional Append with Optimistic Locking

Ensure no new relevant events exist before appending:

```java
import org.sliceworkz.eventstore.events.EventReference;
import org.sliceworkz.eventstore.exceptions.OptimisticLockingException;
import java.util.Optional;

// 1. Query current state
List<Event<CustomerEvent>> events = stream.query(
    EventQuery.forEvents(
        EventTypesFilter.any(),
        Tags.of("customer", "123")
    )
).toList();

// 2. Make business decision based on events
// ... process events and decide to change name ...

// 3. Get reference to last known event
EventReference lastKnownEvent = events.getLast().reference();

// 4. Append with optimistic lock
try {
    stream.append(
        AppendCriteria.of(
            EventQuery.forEvents(
                EventTypesFilter.any(),
                Tags.of("customer", "123")
            ),
            Optional.of(lastKnownEvent)
        ),
        Event.of(
            new CustomerNameChanged("123", "Jane"),
            Tags.of("customer", "123")
        )
    );
} catch (OptimisticLockingException e) {
    // New events were appended since we queried
    // Retry: query again, make decision, append
}
```

### The DCB Pattern

This pattern implements the <a href="https://dcb.events/specification">Dynamic Consistency Boundary specification</a>:

1. **Query** relevant events with an `EventQuery`
2. **Note** the reference of the last relevant event
3. **Decide** based on the events retrieved
4. **Append** new events with `AppendCriteria` containing the same query and last reference
5. If new events matching the query exist after the reference, the append **fails**

This ensures your business decisions are based on complete information and prevents conflicts.
Any new Events that were appended to the Eventstore that wouldn't have influenced your decision are not blocking the append.

## Event Subscriptions

Subscriptions allow your application to be notified of new Events being appended (by another thread, or another process or server in case of postgres/database-storage)

This would be useful to update any read models, for example, and realizes eventual consistency.

An example of subscribing to newly appended events:

```java
import org.sliceworkz.eventstore.listener.EventStreamEventuallyConsistentAppendListener;
import org.sliceworkz.eventstore.events.EventReference;
import org.sliceworkz.eventstore.utils.Handle;
import org.sliceworkz.eventstore.query.Limit;

// Open read-only stream for all events
EventStream<Object> stream = eventstore.getEventStream(EventStreamId.anyContext());

// Get reference to last event as starting point
Handle<EventReference> lastSeen = Handle.of(
    stream.queryBackwards(EventQuery.matchAll(), Limit.to(1))
          .findFirst()
          .map(Event::reference)
          .orElse(null)
);

// Subscribe to new appends
stream.subscribe(new EventStreamEventuallyConsistentAppendListener() {
    @Override
    public void eventsAppended(EventReference atLeastUntil) {
        List<Event<Object>> newEvents = stream.query(
            EventQuery.matchAll(),
            lastSeen.get()
        ).toList();

        newEvents.forEach(System.out::println);

        if (!newEvents.isEmpty()) {
            lastSeen.set(newEvents.getLast().reference());
        }
    }
});
```

> **Note:** Notifications only tell you that Events were appended to **at least** a certain reference.  By the time you reach out to query new Events, it is perfectly possible that more have been appended.  Additionally, not all Event adds are notified individually per se.


## PostgreSQL Configuration

Create a `db.properties` file in your working directory:

```properties
db.pooled.url=jdbc:postgresql://<host>/<db>
db.pooled.username=<user>
db.pooled.password=<password>
db.pooled.leakDetectionThreshold=2000
db.pooled.maximumPoolSize=25
db.pooled.datasource.sslmode=require
db.pooled.datasource.channelBinding=require
db.pooled.datasource.cachePrepStmts=true
db.pooled.datasource.prepStmtCacheSize=250
db.pooled.datasource.prepStmtCacheSqlLimit=2048

db.nonpooled.url=jdbc:postgresql://<host>/<db>
db.nonpooled.username=<user>
db.nonpooled.password=<password>
db.nonpooled.leakDetectionThreshold=70000
db.nonpooled.maximumPoolSize=2
db.nonpooled.datasource.sslmode=require
db.nonpooled.datasource.channelBinding=require
db.nonpooled.datasource.cachePrepStmts=true
db.nonpooled.datasource.prepStmtCacheSize=250
db.nonpooled.datasource.prepStmtCacheSqlLimit=2048
```


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

EventStorage storage = PostgresEventStorage.newBuilder()
    .dataSource(dataSource)
    .prefix("myapp_")		// if you want your tables prefixed
    .initializeDatabase()	// drop/create the database - do this only in DEV
    .build();

EventStore eventstore = EventStoreFactory.get().eventStore(storage);
```

## Testing

For testing, use the in-memory storage:

```java
@Test
void testCustomerNameChange() {
    EventStore eventstore = InMemoryEventStorage.newBuilder().buildStore();

    EventStreamId streamId = EventStreamId.forContext("customer").withPurpose("123");
    EventStream<CustomerEvent> stream = eventstore.getEventStream(streamId, CustomerEvent.class);

    stream.append(AppendCriteria.none(), Event.of(new CustomerRegistered("John"), Tags.none()));
    stream.append(AppendCriteria.none(), Event.of(new CustomerNameChanged("Jane"), Tags.none()));

    List<Event<CustomerEvent>> events = stream.query(EventQuery.matchAll()).toList();

    assertEquals(2, events.size());
    assertEquals("Jane", ((CustomerNameChanged) events.get(1).data()).name());
}
```

## Next Steps

- Read the [DCB Specification](https://dcb.events/specification/) to understand the theoretical foundation
- Explore the `sliceworkz-eventstore-examples` module for more complex scenarios
- Review the API documentation for advanced features like projections and point-in-time queries
- Consider implementing event upcasting for schema evolution

## License

EventStore is licensed under <a href="https://github.com/sliceworkz/eventstore/blob/develop/LICENSE">LGPL v3.0</a>.

## Support

For issues and questions:
- GitHub Issues: [Report an issue](https://github.com/sliceworkz/eventstore/issues)
