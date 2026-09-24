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
- **Archiving**: copying a closed period — one stream, or everything carrying a tag — to a separate store
- **Moving a store to another PostgreSQL cluster**: the supported alternative to a logical dump, which a younger cluster cannot read

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
- does **not decrypt**, so a sealed personal-data value moves as ciphertext, without keys and without the right to read it

Going through an `EventStream` instead would rewrite legacy events into current ones and lose the idempotency key, which the public `Event` record does not carry.

If you built your stores with `buildStore()`, keep a reference to the storage as well when you intend to import:

```java
EventStorage source = InMemoryFsEventStorage.newBuilder().directory("prototype-data").build();
EventStorage target = PostgresEventStorage.newBuilder().build();   // ENSURE creates the schema

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
| event payload | **Preserved** byte-for-byte, sealed values included |
| `position` and `tx` | **Reassigned by the target** |
| `index` | Always 0 at rest — it is a read-time upcasting artifact |

An import reproduces the source *order*, never its ordering numbers. That is modelled in the type: `EventToImport` is a `StoredEvent` minus `position`, `tx` and `index` — exactly the fields a caller controls — so nothing lets you hand-set a position that would be silently ignored.

> Imported events arrive at new (high) positions carrying old timestamps, so "a later position implies a later timestamp" no longer holds in a store that has absorbed an import.
{: .prompt-warning }

### Sealed Values Move as Ciphertext, and the Keys Do Not Move With Them

A [`Shreddable`](/posts/eventstore-erasing-personal-data/) value is stored as a sealed envelope inside the ordinary payload — opaque JSON like any other payload as far as an import is concerned. So the copy is verbatim, and needs no keys, no domain classes and no right to read the personal data.

The consequence is the obvious one: **a store imported into a deployment whose key store does not hold those keys cannot read any protected value.** Every such read throws a `ShreddingException` naming a key the store never held — an unknown key is deliberately *not* reported as erased, since it far more often means a miswired key store than a deliberate erasure. Migrate the keys alongside the events.

To accept the erasure deliberately — seeding an acceptance environment from production, say, with a history complete in every non-personal respect and permanently unreadable in every personal one — carry the key rows across *shredded*: material gone, reason stamped. The values then read as erased, and the audit says why. That is a stronger guarantee than any scrubbing script, since there is nothing left to scrub around.

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

## Selecting What to Copy

`.stream(...)` and `.matching(...)` narrow the run to part of the source, and **both are pushed into the storage query** the source is paged with:

```java
// one logical stream -- a concrete id, or a wildcard
ImportReport q1 = EventStoreImporter.from(live).to(cold)
    .stream(EventStreamId.forContext("ledger").withPurpose("2024Q1"))
    .run();

// every event carrying a tag, of the given types
ImportReport period = EventStoreImporter.from(live).to(cold)
    .matching(EventFilter.forTags(Tags.of("period", "2024Q1")))
    .run();
```

That is what makes the importer an archiving tool. On PostgreSQL a run over one closed period costs what that period's events cost, answered from the stream and tag indexes, not a walk over the table. The alternative — dropping unwanted events in `transform` — reads the whole source to discard most of it: fine against the in-memory store, a full pass over the table in production. The two compose, and the transformation only sees what the selection read.

- **Types are matched by stored name**, since nothing is upcast on this path: select a legacy type by its legacy name, with `EventTypesFilter.of(Set.of(EventType.named("CustomerRegistered")))`.
- **A filter carrying its own `until`** bounds the run there when it is earlier than the source head, and `ImportReport.sourceTo()` then names that boundary, so a later `.after(report.sourceTo())` continues correctly.
- **Selecting does not touch the source.** An archive is a copy. Removing the copied range from the live store is a separate, deliberate operator act — and the bookmarks foreign key makes it fail loudly for a reader still pointing into that range.

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
        ? EventToImport.from(stored).withType(EventType.named("CustomerEnrolled"))
        : EventToImport.from(stored)))
```

For a bare rename, [`@EventName`](/posts/eventstore-defining-events/#eventname-a-stored-name-that-is-not-the-class-name) on the renamed class is usually simpler: it needs no copy at all.

**Filtering a context out of a copy** — for a scrubbed acceptance environment (`.stream(...)` selects *in*, which is cheaper; this drops):

```java
.transform(stored -> stored.stream().context().equals("payments")
    ? Optional.empty()
    : Optional.of(EventToImport.from(stored)))
```

**Rewriting payloads** — `withPayload(...)` takes the raw JSON document, so a mechanical schema change can be applied without domain classes. The payload must stay a JSON document: every backend refuses one that is not, storing nothing of the batch.

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

**Check a target in raw mode.** If you probe the target for an event before importing, use a raw stream:

```java
// raw: no mappings, so nothing can fail to deserialize
EventSource<String> raw = targetStore.getRawEventStream(EventStreamId.anyContext());
boolean present = raw.getEventById(someId).isPresent();
```

A typed stream answers presence correctly too — a legacy event whose upcast yields zero current events is *present* with an empty list, not absent — but it needs the domain classes and throws on an event its mappings cannot read.

## Moving a Store to Another Cluster

A `pg_dump` restored into a fresh PostgreSQL cluster keeps the source's transaction ids, which the younger cluster reads as the future: the history is invisible, and new appends sort before it. The store [refuses to start](/posts/eventstore-configuring-postgresql-storage/#backup-and-restore) in that state. An import is the supported way to move: the target assigns both ordering columns, in source order, so the ordering is the new cluster's own.

What the importer does *not* carry, and the runbook therefore has to:

- **The `btree_gin` extension** on the target database. Let `ENSURE` create the target schema, or install the extension first.
- **Bookmarks.** Copy `<prefix>bookmarks` across *after* the events. A bookmark stores only the event id, the import preserves ids, and the target answers a bookmark's position from its own events — so the copied table is valid as it stands and the foreign key holds.
- **Shredding keys.** Copy `<prefix>shredding_keys` alongside, shredded rows included, so erased values still read as erased.
- **Leases** are deliberately not migrated; they expire.
- **Anything outside the store holding event references** — a read model's own cursor columns, say — holds the source's coordinates. Rebuild such read models on the target rather than resuming them.

Import in batches with `SKIP_EXISTING_ID` so a failed run resumes, and feed `ImportReport.sourceTo()` into a later run's `.after(...)` to pick up events appended to the source during the cutover.

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
    EventType.named("CustomerRegistered"),
    EventId.of(UUID.randomUUID().toString()),
    "{\"name\":\"John\"}",              // payload: a JSON document
    Tags.of("customer", "123"),
    Instant.parse("2024-01-15T10:30:00Z"),
    null                                 // idempotency key
);

target.importEvents(List.of(synthetic), ImportMode.FAIL_ON_EXISTING_ID);
```

> This bypasses `append()` and everything that path guarantees: no optimistic locking, no serialization from a typed domain event, no check that the payload matches the type name — only that it is a JSON document. It is a tool for fixtures and migrations, not a second write path for an application.
{: .prompt-danger }

For testing application code, prefer the [testing fixture](/posts/eventstore-testing/), which seeds history through the ordinary append path.
