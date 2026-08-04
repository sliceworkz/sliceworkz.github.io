---
layout: post
toc: true
title: Lifecycle and Shutdown
description: Starting, closing and releasing EventStorage, EventStore and EventStream
date: 2026-08-04 01:00:00
categories: [Eventstore Documentation,Eventstore deployment]
tags: [lifecycle,shutdown,autocloseable,close,startup]
---

This guide covers the lifecycle of the three objects your application holds — `EventStorage`, `EventStore` and `EventStream` — from a store that refuses to start because it cannot reach its database, to releasing the threads and connections a store owns when you are done with it.

## Three Objects, Three Scales

The library hands you three things with a lifecycle, and they are not equally expensive:

| Object | What it owns | When you close it |
|---|---|---|
| `EventStorage` | Background threads, JDBC connections, possibly the connection pools it created | When the storage is no longer used — typically at application shutdown |
| `EventStore` | Executors that dispatch notifications to subscribers | When the store is no longer used; does **not** close a storage you gave it |
| `EventStream` | Its subscriptions, and nothing else | When you are done subscribing; never needed for a query/append-only stream |

All three implement `AutoCloseable`, and `close()` is declared without a checked exception, so try-with-resources needs no `catch`.

## Starting a Store

`build()` connects, applies the configured `DatabaseInitMode`, and — on the PostgreSQL backend — starts the two LISTEN/NOTIFY monitor threads and waits for them to register their channels.

```java
EventStorage storage = PostgresEventStorage.newBuilder().build();
```

That wait is **bounded**, 10 seconds by default. The monitors have no failure mode of their own: on a `SQLException` they log, back off and retry for as long as the storage lives, so waiting on them without a deadline is waiting on something that may never happen.

### Expiry Is Fatal

When the deadline passes without the notification channels being established, `build()` closes the storage and throws an `EventStorageException`. There is deliberately no mode that starts anyway.

An event-sourced application that is never told when events are appended has read models that quietly stop advancing: nothing wakes a subscriber, so projections only move when something else happens to run them. It serves stale data with nothing in its own logs to say so — worse than not starting at all. Closing (rather than merely throwing) matters too: the monitor threads would otherwise keep retrying behind a storage the caller never received.

### Where Startup Legitimately Races the Database

Container orchestrators start applications and databases at the same time. Raise the deadline rather than lowering your expectations:

```java
EventStorage storage = PostgresEventStorage.newBuilder()
    .notificationStartupTimeout(Duration.ofSeconds(30))
    .build();
```

Within the deadline, the monitors' own retry loop does the waiting, so a store racing its database up succeeds rather than failing. The default is generous on purpose: a database that is up answers in milliseconds, and the cost of being too impatient with one that is merely slow — a cold pool, a simultaneous restart — is an application that refuses to boot.

> A **running** store still repairs itself. The same retry loop brings notifications back after an outage, with nothing to restart. The fail-fast is about not *starting* blind, not about tearing a live store down when its connection drops.
{: .prompt-info }

### Which Configurations Can Actually Hit This

With `ENSURE` or `VALIDATE`, the schema work runs before the monitors start and fails with a clear error, so an unreachable main `DataSource` never reaches the wait. The paths that can reach it are:

- `DatabaseInitMode.NONE`, where nothing touches the database before the monitors do
- a **reachable main DataSource with an unreachable monitoring one** — the realistic case, since the two are configured separately precisely because LISTEN/NOTIFY does not survive a transaction pooler. "Pooled works, direct is firewalled" is an ordinary misconfiguration

### Watching Notification Health

`sliceworkz.eventstore.notifications.up` is a gauge reading 1 or 0, tagged `storage` and `channel` (`event_appended` / `bookmark_placed`). It is registered by the storage constructor, so the series exists reading 0 from the moment the storage does — a gauge that only appears once notifications work is no use for alerting on notifications *not* working. It drops back to 0 when a running store loses its monitoring connection, which is the same silence as never having had one.

For a health endpoint, `PostgresEventStorageImpl.isNotificationsAvailable()` exposes the same state, at the cost of a downcast from `EventStorage`.

## Closing a Store

A store that lives as long as the process needs no explicit close. One created per tenant, per test or per hot reload does: the PostgreSQL backend runs two monitor threads, each holding a JDBC connection, and those threads keep the whole storage reachable — so a dropped-but-unclosed storage is never reclaimed by garbage collection.

```java
try ( EventStore eventStore = PostgresEventStorage.newBuilder().buildStore() ) {
    EventStream<CustomerEvent> stream = eventStore.getEventStream(streamId, CustomerEvent.class);
    // ... use the store ...
}   // stops the monitors and closes the pools the builder created
```

### The Contract Every Backend Implements

**Idempotent** — later calls do nothing and never throw.

**Blocks, bounded** — when `close()` returns, the background threads have finished and released their connections. On PostgreSQL the monitors poll in 100 ms slices, so they notice the stop within that, `UNLISTEN`, and hand their connections back to the pool healthy: closing takes about 100 ms and logs nothing. A monitor that fails to stop on its own within 2 seconds is interrupted instead (hard bound 5 seconds); that path breaks its connection under the driver, so the pool logs "connection marked as broken" — an interrupted shutdown, not a normal one.

**Ownership** — a `DataSource` you passed to `.dataSource(...)` is never closed; one the builder created from `db.properties` is.

> If you supply the pool, close the storage **before** closing the pool. The other order leaves the monitors retrying against a dead pool, which they cannot distinguish from a database outage.
{: .prompt-warning }

**Terminal** — there is no reopening. `start()` on a closed storage throws.

**Operations throw afterwards** — every read and write throws `EventStorageClosedException` (a subclass of `EventStorageException`); `name()` keeps working, so logging does not break. A closed storage does not keep serving reads while its notifications are dead, which would strand projections silently.

### Storage Ownership: Who Closes What

Closing an `EventStore` does **not** close a storage you handed it. A storage is the expensive object, it can back several stores, and it usually outlives them — so closing it is the caller's job, after closing the stores built on it.

```java
EventStorage storage = PostgresEventStorage.newBuilder().build();

EventStore writeStore = EventStoreFactory.get().eventStore(storage);
EventStore readStore  = EventStoreFactory.get().eventStore(storage);

// ... application runs ...

writeStore.close();     // readStore keeps working
readStore.close();
storage.close();        // now the monitors stop and the pools are released
```

The one case that owns its storage is composed rather than special-cased. `buildStore()` creates the storage, hands back nothing else, and therefore returns a store that closes both:

```java
// this store owns its storage: closing it closes everything
EventStore eventStore = PostgresEventStorage.newBuilder().buildStore();
```

You can build the same pairing yourself with `EventStore.owning(...)` when you create a storage and a store together:

```java
EventStorage storage = PostgresEventStorage.newBuilder().build();
EventStore eventStore = EventStore.owning(
    EventStoreFactory.get().eventStore(storage, registry),
    storage
);

eventStore.close();     // closes the store, then the storage
```

A closed `EventStore`'s streams throw too, for the same reason a closed storage's operations do: its notifications have stopped, so letting it keep reading would strand its subscribers silently.

> Spring infers `close` as the destroy method for a `@Bean`, and CDI has `@Disposes` — so declaring an `EventStore` or `EventStorage` bean gives you shutdown for free in both.
{: .prompt-tip }

## Closing a Stream

`EventStream` is `AutoCloseable` too, but at a much smaller scale: the only thing a stream owns is its subscriptions.

### A Stream You Only Query and Append Through Owns Nothing

`getEventStream()` registers nothing with the storage. Registration happens on the **first** `subscribe(...)`, because a stream with no subscribers has nothing to do with a notification anyway. Most streams are in this category, are handed out per operation, and need no lifecycle handling at all:

```java
// no cleanup needed: nothing was registered
EventStream<CustomerEvent> stream = eventStore.getEventStream(streamId, CustomerEvent.class);
stream.append(AppendCriteria.none(), Event.of(new CustomerRegistered("John"), Tags.none()));
```

### A Stream You Subscribe To Is Held by the Storage

Once subscribed, the storage holds the stream **strongly** until it is closed. That is what makes live updates survive the caller dropping the variable:

```java
// this keeps working — the storage holds the stream, so the subscription cannot be collected
eventStore.getEventStream(streamId, CustomerEvent.class)
          .subscribe(reference -> { updateReadModel(); return reference; });
```

The cost of that guarantee is that nothing releases it on your behalf. A subscribed stream that is never closed is retained for the lifetime of the storage — deliberately a leak you can find, rather than a subscription that dies at an unpredictable garbage collection with no error and no log.

**So close what you subscribe to**, or close the store, which closes them all:

```java
try ( EventStream<CustomerEvent> stream = eventStore.getEventStream(streamId, CustomerEvent.class) ) {
    Projector.from(stream).towards(projection).subscribe().build();
    // ... application runs ...
}   // subscriptions ended, registration released
```

### Closing a Stream Is Not Terminal

Unlike closing a store or a storage, closing a stream ends its subscriptions and clears its listeners, but leaves the handle usable for `query()`, `append()` and bookmark operations. Subscribing again re-registers it. A stream is a cheap per-operation handle, not a connection — there is nothing to protect by poisoning it. It is idempotent, and closing a never-subscribed stream is a no-op.

### Streams Are Cheap Because the Expensive Part Is Shared

`getEventStream()` allocates a stream object and resolves about ten Micrometer meters — roughly **2 µs and 1 KB**. The payload serializer/deserializer is *not* rebuilt per call: the store caches one per distinct set of event root classes and hands the same instance to every stream opened with that mapping.

That matters more than the construction cost suggests. Jackson caches its per-type serializers inside the mapper, so a serde per call would give every stream a cold type cache and re-run bean introspection on the first serialization of each record type. Measured on a 24-record sealed hierarchy, that difference was **~175 µs / 139 KB against ~36 µs / 69 KB** per query. With the serde shared, obtaining a fresh stream per operation costs the same as keeping one.

Only the serde is shared — never the stream itself. A stream is stateful (subscriber lists, a subscribed flag), so sharing it would make one caller's `close()` end another caller's subscriptions.

## Putting It Together

A typical application wiring, with everything released in the right order:

```java
public class Application implements AutoCloseable {

    private final EventStore eventStore;
    private final EventStream<CustomerEvent> subscribedStream;

    public Application ( ) {
        // buildStore() owns its storage, so closing this store closes everything
        this.eventStore = PostgresEventStorage.newBuilder()
            .validateDatabase()
            .notificationStartupTimeout(Duration.ofSeconds(30))
            .buildStore();

        this.subscribedStream = eventStore.getEventStream(
            EventStreamId.forContext("customer"), CustomerEvent.class);

        Projector.from(subscribedStream)
            .towards(new CustomerSummary())
            .subscribe()
            .bookmarkProgress()
                .withReader("customer-summary")
                .done()
            .build();
    }

    /** Per-operation streams need no lifecycle handling at all. */
    public void register ( String id, String name ) {
        eventStore.getEventStream(EventStreamId.forContext("customer"), CustomerEvent.class)
                  .append(AppendCriteria.none(),
                          Event.of(new CustomerRegistered(id, name), Tags.of("customer", id)));
    }

    @Override
    public void close ( ) {
        subscribedStream.close();   // end the subscription
        eventStore.close();         // store, then the storage it owns
    }
}
```

If the store outlives nothing — that is, if it lives exactly as long as the JVM — you can skip all of it. The lifecycle exists for the cases where it does not.
