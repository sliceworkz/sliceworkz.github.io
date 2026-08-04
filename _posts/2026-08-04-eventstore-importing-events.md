---
layout: post
toc: true
title: Importing Events Between Stores
description: Copying, migrating and cloning event stores with EventStoreImporter
date: 2026-08-04 02:00:00
categories: [Eventstore Documentation,Eventstore deployment]
tags: [import,migration,eventstorage,backup,cloning]
---

This guide covers `EventStoreImporter` — the supported way to move events from one `EventStorage` into another, preserving event identity, timestamps and idempotency keys.

## What Importing Is For

Appending is how an application writes new facts. Importing is for moving facts that already happened somewhere else:

- **Backend migration**: moving a store from file-persisted in-memory storage to PostgreSQL when a prototype goes to production
- **Environment seeding**: copying a production store into an acceptance environment
- **Store splitting or merging**: relocating one context's events into a store of their own, or consolidating several stores into one
- **Schema migration at rest**: rewriting event types, tags or payloads across a whole history instead of carrying upcasters forever
- **Archiving**: moving old events to a separate store, remapped onto an archive stream

```java
ImportReport report = EventStoreImporter.from(sourceStorage)
                                        .to(targetStorage)
                                        .run();

LOGGER.info("migrated: {}", report);
```

`EventStoreImporter` lives in `org.sliceworkz.eventstore.migration` in the API module — no extra dependency is needed.

## It Works on Storages, Not Stores

Both ends of an import are an `EventStorage`, not an `EventStore`. That is deliberate, and it is what makes an import faithful.

A storage deals in `StoredEvent`: an opaque JSON payload plus a type *name*. So an import:

- needs **no domain classes on the classpath** — a migration tool does not have to be built against the application
- does **no serialization round-trip**, so a payload cannot be subtly rewritten by a mapper on the way through
- does **not upcast**, so legacy events arrive in the target as legacy events, not as their current equivalents
- does **not re-split `@Erasable` fields** against annotations that may have changed since they were written

Going through an `EventStream` instead would rewrite legacy events into current ones and lose the idempotency key, which the public `Event` record does not carry.

If you built your stores with `buildStore()`, keep a reference to the storage as well when you intend to import:

```java
EventStorage source = InMemoryFsEventStorage.newBuilder().directory("prototype-data").build();
EventStorage target = PostgresEventStorage.newBuilder().ensureDatabase().build();

EventStoreImporter.from(source).to(target).run();

target.close();
source.close();
```

## What Survives an Import, and What Does Not

| Field | Outcome |
|---|---|
| `EventId` | **Preserved** — identity survives the copy, which is what makes a resumable import possible |
| timestamp | **Preserved** |
| idempotency key | **Preserved**, and still scoped per stream in the target |
| event type, tags | **Preserved** |
| immutable and erasable payloads | **Preserved** byte-for-byte |
| `position` and `tx` | **Reassigned by the target** |
| `index` | Always 0 at rest — it is a read-time upcasting artifact |

An import reproduces the source *order*, never its ordering numbers. That is modelled in the type: `EventToImport` is a `StoredEvent` minus `position`, `tx` and `index` — exactly the fields a caller controls — so nothing lets you hand-set a position that would be silently ignored.

> Imported events arrive at new (high) positions carrying old timestamps, so "a later position implies a later timestamp" no longer holds in a store that has absorbed an import.
{: .prompt-warning }

## Import Modes

`EventStorage.ImportMode` decides what happens when an event id is already present in the target:

**`FAIL_ON_EXISTING_ID`** (the default) — an already-present event id aborts the batch with an `EventImportConflictException`. The safer choice when the target is expected to be free of these events and an unexpected overlap should stop the operation rather than be absorbed.

**`SKIP_EXISTING_ID`** — an already-present event id is skipped and the rest of the batch is imported. This is the **resume mode**: matching is on the id alone, with no payload read back or compared.

```java
ImportReport report = EventStoreImporter.from(source).to(target)
    .mode(ImportMode.SKIP_EXISTING_ID)
    .run();
```

An idempotency key already in use by a *different* event on the same stream is fatal in **both** modes. Skipping is keyed on event identity, and a colliding idempotency key means two different events claim the same key on the same stream, which no mode is willing to absorb.

## Resuming and Catching Up

`ImportReport.sourceTo()` is the reference in the **source** the run read up to. Feed it into a later run's `.after(...)` to pick up only what has been appended since:

```java
ImportReport first = EventStoreImporter.from(source).to(target).run();

// ... the source keeps receiving events ...

ImportReport catchUp = EventStoreImporter.from(source).to(target)
    .mode(ImportMode.SKIP_EXISTING_ID)
    .after(first.sourceTo())
    .run();
```

The catch-up run costs O(new events) rather than re-reading the whole history.

Reads are always bounded at the **source head captured before the first write**. That is what makes `from(x).to(x)` — cloning inside one store — terminate instead of re-reading its own writes forever, and it is why events appended to the source *during* a run are excluded rather than partially included.

## Transforming Events on the Way Through

`.transform(...)` receives each `StoredEvent` and returns an `Optional<EventToImport>`. Returning an empty `Optional` drops the event; every field has a wither, so anything can be rewritten:

```java
EventStreamId archive = EventStreamId.forContext("customer-archive");

ImportReport report = EventStoreImporter.from(source).to(target)
    .transform(stored -> Optional.of(EventToImport.from(stored)
        .withStream(archive)
        .withTags(stored.tags().merge(Tags.of(Tag.of("archived"))))))
    .run();
```

Some things this makes possible:

**Renaming an event type across the whole history** — an alternative to keeping a legacy class and an upcaster forever:

```java
.transform(stored -> Optional.of(
    stored.type().name().equals("CustomerRegistered")
        ? EventToImport.from(stored).withType(EventType.ofType("CustomerEnrolled"))
        : EventToImport.from(stored)))
```

**Filtering a context out of a copy** — for a scrubbed acceptance environment:

```java
.transform(stored -> stored.stream().context().equals("payments")
    ? Optional.empty()
    : Optional.of(EventToImport.from(stored)))
```

**Rewriting payloads** — `withImmutableData(...)` and `withErasableData(...)` take the raw JSON, so a mechanical schema change can be applied without domain classes.

Events dropped by the transform are counted in `ImportReport.dropped()`.

> The transform can rewrite anything, including the event id — which makes `SKIP_EXISTING_ID` meaningless, since nothing stable is left to match on. Rewrite ids only in a single-pass copy you are prepared to redo from scratch.
{: .prompt-warning }

## Batching and Progress

Events are read, transformed and written in batches, one committed transaction per batch. The default batch size is 1000:

```java
ImportReport report = EventStoreImporter.from(source).to(target)
    .batchSize(5000)
    .onProgress(r -> LOGGER.info("import progress: {}", r))
    .run();
```

`onProgress` is called after every batch with a cumulative `ImportReport`, which makes it a natural place to drive a progress bar or a log line on a long migration.

## Reading the ImportReport

```java
public record ImportReport (
    long read,                          // stored events read from the source
    long dropped,                       // events the transform returned empty for
    long imported,                      // events actually written to the target
    long skipped,                       // events skipped as already present (SKIP_EXISTING_ID)
    EventReference sourceFrom,          // the .after(...) cursor this run started from, or null
    EventReference sourceTo,            // the source head this run was bounded at
    EventReference firstTargetReference, // first reference assigned in the target
    EventReference lastTargetReference,  // last reference assigned in the target
    Duration duration
) { }
```

`read` equals `dropped + imported + skipped` for a completed run. `sourceTo` is the value to keep for a later catch-up.

## Caveats That Matter Operationally

**Atomic per batch only.** A failure part-way leaves earlier batches committed. Re-run with `SKIP_EXISTING_ID` to continue. There is no dry-run mode.

**Nothing is verified.** Matching is on id; the faithfulness of a migration is the caller's problem. If a transform rewrites payloads, only your own checks will catch a mistake.

**One importer at a time per target.** The conflict check and the insert are not under a common lock, so two concurrent imports into the same store can interleave in ways neither mode is designed for.

**Listeners are notified** exactly as for appends, so merging into a live store wakes its projections. That is usually what you want — but it means a large import into a production store also drives that store's read models.

**Check a target in raw mode.** If you probe the target for an event before importing, open the stream with no event root classes:

```java
// raw: no mappings, so nothing can fail to deserialize
EventStream<Object> raw = targetStore.getEventStream(EventStreamId.anyContext());
boolean present = !raw.getEventById(someId).isEmpty();
```

With domain classes registered, `getEventById` upcasts — and a legacy event whose upcast yields zero current events comes back as an empty list even though it exists, which reads as a false negative.

## PostgreSQL Specifics

The target requires no DDL change for importing: `event_id` is already a plain `UUID NOT NULL UNIQUE`, and `event_timestamp` is nullable with a `CURRENT_TIMESTAMP` default, so both can be supplied explicitly.

- **Imported event ids must be UUIDs.** This is validated up front to give a clear error rather than an opaque cast failure.
- **Statements are chunked at 5000 rows** inside a batch's transaction (nine parameters per row against the 65535-parameter wire ceiling).
- **`timestamptz` keeps microseconds and rounds anything finer.** A nanosecond-precision timestamp, as an in-memory store produces, lands up to half a microsecond from where it started. This is the only lossy part of an in-memory → PostgreSQL → in-memory round trip.

## Writing Synthetic Events

`EventToImport`'s canonical constructor is public, so it can also write events with a chosen id and timestamp straight into a store — useful for fixtures and for reproducing a reported history:

```java
EventToImport synthetic = new EventToImport(
    EventStreamId.forContext("customer").withPurpose("123"),
    EventType.ofType("CustomerRegistered"),
    EventId.of(UUID.randomUUID().toString()),
    "{\"name\":\"John\"}",              // immutable payload
    null,                                // erasable payload
    Tags.of("customer", "123"),
    LocalDateTime.of(2024, 1, 15, 10, 30),
    null                                 // idempotency key
);

target.importEvents(List.of(synthetic), ImportMode.FAIL_ON_EXISTING_ID);
```

> This bypasses `append()` and everything that path guarantees: no optimistic locking, no serialization from a typed domain event, no check that the payload matches the type name. It is a tool for fixtures and migrations, not a second write path for an application.
{: .prompt-danger }

For testing application code, prefer the [testing fixture](/posts/eventstore-testing/), which seeds history through the ordinary append path.
