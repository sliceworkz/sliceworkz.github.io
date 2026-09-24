---
layout: post
toc: true
title: Upgrading to 0.11
description: What changed between eventstore 0.10 and 0.11.1, what to rename, and what to migrate in a PostgreSQL database
date: 2026-09-24 01:00:00
categories: [Eventstore Documentation,Eventstore deployment]
tags: [upgrade,migration,release notes,postgres,api changes]
---

This guide takes an application from eventstore **0.10.x** to **0.11.1**. The 0.11 line reworked a good part of the public API — mostly to make one way of doing each thing, and to make the compiler catch what used to fail at runtime — and it changes the PostgreSQL schema in one place that needs a hand-applied migration.

Work through it in this order: [bump the version](#1-bump-the-version), [fix what no longer compiles](#2-fix-what-no-longer-compiles), [check the behaviour changes](#3-behaviour-changes-that-compile-silently) that compile silently, then [migrate the database](#4-postgresql-schema) before the first start.

## 1. Bump the Version

```xml
<properties>
    <sliceworkz.eventstore.version>0.11.1</sliceworkz.eventstore.version>
</properties>
```

Two dependency changes may show up in your build:

- **Micrometer is gone** from every eventstore artifact. If your application relied on getting it transitively, declare it yourself. The store now reports through its own [`EventStoreObserver`](/posts/eventstore-observability-micrometer-prometheus-grafana/) interface.
- **The api jar carries SLF4J and nothing else of note**; Jackson 3 (`tools.jackson.*`) comes with the implementation and the backends. The storage backends pull in `sliceworkz-eventstore-impl` at runtime, so an explicit runtime dependency on it can go.

Every jar now declares an `Automatic-Module-Name`, so module-path users can write `requires org.sliceworkz.eventstore;` and `requires org.sliceworkz.eventstore.infra.postgres;`.

## 2. Fix What No Longer Compiles

### Renames

| 0.10 | 0.11 |
|---|---|
| `EventStoreFactory.get().eventStore(storage)` | `EventStore.on(storage).build()` |
| `EventStoreFactory.get().eventStore(storage, registry, meterOptions, codec)` | `EventStore.on(storage).observer(observer).shredding(codec).build()` |
| `Projector.from(s).towards(p)` | `Projector.from(s).into(p)` |
| `.bookmarkProgress().withReader(r).withTags(t).readBeforeEachExecution().done()` | `.bookmarkAs(r, t)` — reading before every run is the default |
| `.readAtCreationOnly()` / `.readBeforeFirstExecution()` | `.readBookmarkOnce()` |
| `.readOnManualTriggerOnly()` | `.readBookmarkOnRequest()` |
| `EventStreamEventuallyConsistentAppendListener` | `AppendListener` |
| `EventStreamEventuallyConsistentBookmarkListener` | `BookmarkListener` |
| `Upcast<L, T>` | `Upcaster<L, T>` |
| `@LegacyEvent(upcast = X.class)` | `@LegacyEvent(upcaster = X.class)` |
| `EventType.ofType("Name")` | `EventType.named("Name")` |
| `query.combineWith(other)` | `query.or(other)` |
| `query.matches(event)`, `query.isMatchAll()`, … | `query.filter().matches(event)`, `query.filter().isMatchAll()`, … |
| `EventStreamId.canRead(other)` | `EventStreamId.covers(other)` |
| `DatabaseInitMode.INITIALIZE` / `.initializeDatabase()` | `DatabaseInitMode.RECREATE` / `.recreateDatabase()` |
| `eventStore.erase(subject, reason)` | `eventStore.eraseCategory(subject, reason)` for one category — or `erase(type, id, reason)` for the whole person, which is what an art.17 request wants |
| `ShreddingKeyStore.resolve(keyId)` → `Optional<SecretKey>` | `resolveKey(keyId)` → `KeyResolution` (`Resolved` / `Erased` / `Withheld`) |
| `ShreddingCodec.unseal(sealed)` → `Optional<String>` | `open(sealed)` → `Unsealed` (`Plaintext` / `Erased` / `Withheld`) |
| `EventToImport.withImmutableData(json)`, `StoredEvent.immutableData()` | `withPayload(json)`, `payload()` |
| fixture `ProjectionRun.upTo(ref)` / `.expectEventsProcessed(n)` | `.until(ref)` / `.expectEventsHandled(n)` |

### Types That Changed

| API | 0.10 | 0.11 |
|---|---|---|
| `EventSource.query(...)` | `Stream<Event<E>>` | `List<Event<E>>`, read in full — drop the `.toList()` |
| `EventSource.getEventById(id)` | `List<Event<E>>` | `Optional<List<Event<E>>>` — `isPresent()` is the presence check |
| `EventSource.subscribe(listener)` | `void` | `Subscription` — the handle that ends that one listener |
| `Event.timestamp()`, `StoredEvent`, `EventToImport` | `LocalDateTime` (UTC by convention) | `Instant` |
| `EventSink.append(criteria, events)` | `List<EphemeralEvent<? extends E>>` | `List<? extends EphemeralEvent<? extends E>>` — a mapped list fits with no type witness; *implementors* of `EventSink` widen their override |
| `PostgresEventStorage.Builder.build()` | `EventStorage` | `PostgresEventStorage`, with `isNotificationsAvailable()` |
| `SubjectErasureReport` from `erase(type, id, reason)` | — | one `ErasureReport` per category erased |

`timestamp()` as an `Instant` is the change most likely to reach beyond the store: any code that treated the `LocalDateTime` as UTC can drop that convention, and rendering it for a person means choosing a zone — `event.timestamp().atZone(zone)`.

### Removed, With What to Use Instead

| Removed | Instead |
|---|---|
| `getEventStream(id)` with no root classes (an untyped `EventStream<Object>`) | `getRawEventStream(id)` — an `EventSource<String>` whose data is the stored JSON document — or `getEventStream(id, Set.of(...roots))` for a stream typed wider than one hierarchy |
| `query(q, cursor, Limit)` | `query(q.limit(n), cursor)`, or `page(q.limit(n), cursor)` for an [`EventPage`](/posts/eventstore-querying-events/#paging-through-a-stream) |
| `EventWithMetaDataHandler`, `ProjectionWithoutMetaData`, `when(E data)` | `EventHandler` / `Projection` with the one method `when(Event<E> event)`; switch on `event.data()` |
| `Event.cast()` | open the stream typed at the narrower root — a handle is cheap, and several may share one stream id |
| `EventQuery.merge(...)`, `MergedEventQueries` | `or(...)` to combine queries you chose; run disjoint queries separately |
| `MeterOptions`, `.meterRegistry(...)`, `.meterOptions(...)` on the builders | `.observer(EventStoreObserver)`; `.poolMetrics(MetricsTrackerFactory)` for HikariCP on PostgreSQL |
| `append(criteria, events, targetStream)` through a wildcard stream | open the target stream by its own id and append there |
| `PostgresEventStorageImpl` & co. as public types | the builder is the only way to a storage; `PostgresEventStorage` is the public handle |

### New Things Worth Adopting While You Are There

- **`stream.append(event)`** without criteria, for an unconditional append.
- **`stream.head()`** to pin a consistency boundary *before* reading — see [Optimistic Locking](/posts/eventstore-appending-events/#optimistic-locking).
- **`EventQuery.forTypes(Root.class).tagged("key", value)`** and **`EventQuery.forTags(tags)`**; a sealed root names its whole hierarchy.
- **`@EventName`** for a class whose stored name cannot be its own name.
- **Upcast chains**, and `targetTypes()` derived from the declaration for a one-to-one upcaster.
- **Reader entitlement** — `ShreddingCodec.withholdingAll()`, `restrictedTo(categories)` and `Shreddable.Withheld`; add the `Withheld` case to every exhaustive `switch` over a `Shreddable`, which the compiler will point out.
- **`Projector.Builder.named(...)`**, for the name an observer sees.

## 3. Behaviour Changes That Compile Silently

These need a look even when everything builds:

- **An `AppendCriteria` filter must carry no `until`.** If you built criteria from "the query I read with" and that query was bounded, optimistic locking was off. Hand the criteria the unbounded query, with the boundary as its reference.
- **A filter naming a legacy type is refused** on a stream that upcasts it (`IllegalArgumentException`). Name the current type.
- **A wildcard stream refuses `append`** with `IllegalArgumentException`.
- **A sealed interface in `EventTypesFilter.of(...)` now stands for every type under it.** A filter that named an interface used to match nothing; it now matches the hierarchy. A non-sealed interface is refused.
- **Idempotency keys work per batch.** A batch whose keys are all stored is swallowed as a retry; a batch mixing stored and new keys throws `IdempotencyKeyConflictException`; a batch repeating a key throws `IllegalArgumentException`. See [Idempotency](/posts/eventstore-appending-events/#a-batch-is-the-unit-of-de-duplication).
- **`Tags.of(Tag...)` drops a repeated tag** rather than failing, and `tags.tag(key)` throws `IllegalStateException` when several tags carry the key — use `tags(key)`.
- **A bookmarked projector reads its bookmark before every run** by default.
- **A projector reads each batch whole**: a poison event means none of its batch reaches the projection. `ProjectorException.getEventReference()` is the last event handed to `when`, possibly from an earlier batch, or null.
- **A savepoint handler that throws** fails the run as a `ProjectorException`, and the next run re-runs `initQuery()`.
- **A shredding key id the store has never held now throws** `ShreddingException` instead of reading as erased. A store whose key store is miswired — wrong directory, prefix or database, events imported without keys — fails loudly now.
- **Payloads must be JSON documents** on the raw SPI path (imports, fixtures writing `EventToImport` directly), on every backend.
- **`erase` means the whole person.** Code that called `erase(subject, reason)` for one category must now call `eraseCategory`.
- **The AES-GCM codec refuses** a key that is not 256-bit AES material, and a `|` in a subject's type, id or category.
- **The PostgreSQL shredding key cache is bounded** at 10.000 entries, least recently used evicted first.
- **Lease fencing tokens** increase when an owner re-acquires its *own* expired or released lease.
- **A PostgreSQL table prefix is folded to lowercase**, and one starting with a digit is refused. A store configured with `Tenant1_` already lived in `tenant1_*` tables, and now also gets its notifications.
- **`db.properties` is found at fixed locations only** — system property, environment variable, `./db.properties`, the classpath root — and never in parent directories. See [where it is found](/posts/eventstore-configuring-postgresql-storage/#configuring-an-eventstore-managed-datasource-dbproperties).
- **PostgreSQL conditional appends wait at most `lockTimeout`** (10 seconds) for their stream's lock, then fail with an `EventStorageException`.
- **A PostgreSQL store restored with `pg_dump` into a younger cluster refuses to start.** See [Backup and Restore](/posts/eventstore-configuring-postgresql-storage/#backup-and-restore).

## 4. PostgreSQL Schema

`ENSURE`, the default mode, brings an existing 0.10 database forward by itself on the next start — with **one exception that applies to every mode**.

### The Bookmarks Table

A bookmark now stores only the event id; its position and transaction are answered from the event. The bookmarks table must drop its two ordering columns — they are `NOT NULL` and nothing writes them any more, so the first `placeBookmark` would fail on them. `ENSURE` never drops a column, so apply this by hand, **before** starting 0.11.1, under every init mode:

```sql
ALTER TABLE <prefix>bookmarks DROP COLUMN event_position, DROP COLUMN event_tx;
```

No data migration is needed: the columns were a copy of what the events row says. Schema validation reports an un-migrated table with this statement under `ENSURE` and `VALIDATE`; under `NONE`, the first `placeBookmark` names it.

### What ENSURE Does for You

On the first start of 0.11.1 against a 0.10 database, `ENSURE`:

- **creates two order indexes**, `idx_events_global_order` and `idx_events_context_order` (see [the index table](/posts/eventstore-configuring-postgresql-storage/#using-the-initialization-script)). A plain `CREATE INDEX` blocks appends while it builds — on a large table, prefer creating them by hand first, as below;
- **replaces the `notify_bookmark_placed` function**, which now looks the bookmarked event up for its notification payload;
- **drops** `idx_events_tx_position` and `idx_events_context_tx_position`, in case a 0.11.0 start created them.

### What a VALIDATE or NONE Deployment Applies by Hand

`VALIDATE` and `NONE` change nothing, so a DBA applies the same, with the store's prefix. The indexes can be built without blocking appends — each statement on its own, outside a transaction block, which `CONCURRENTLY` requires:

```sql
CREATE INDEX CONCURRENTLY IF NOT EXISTS <prefix>idx_events_global_order
    ON <prefix>events (event_tx, event_position) WHERE event_position > 0;
CREATE INDEX CONCURRENTLY IF NOT EXISTS <prefix>idx_events_context_order
    ON <prefix>events (stream_context, event_tx, event_position) WHERE event_tx > '0'::xid8;
DROP INDEX CONCURRENTLY IF EXISTS <prefix>idx_events_tx_position;
DROP INDEX CONCURRENTLY IF EXISTS <prefix>idx_events_context_tx_position;
```

Keep the predicates exactly as written: they are what keeps a single-stream read from wandering into these indexes, and validation checks them. Then apply the current `notify_bookmark_placed` function body from the shipped `ensure-schema.sql` (or `quickstart.ddl.sql`).

### Optional Clean-up

The `event_erasable_data` column is no longer part of the schema. Nothing reads or writes it and validation no longer requires it; drop it at your convenience:

```sql
ALTER TABLE <prefix>events DROP COLUMN event_erasable_data;
```

A reporting role that must never read personal data can now be granted every column of `<prefix>shredding_keys` except `key_material` — see [A Role That Must Not Read Personal Data](/posts/eventstore-configuring-postgresql-storage/#a-role-that-must-not-read-personal-data).

## 5. Observability

The Micrometer meters — `sliceworkz.eventstore.append`, `.query.duration`, `.notifications.up` and the rest — are gone, along with `MeterOptions`. Dashboards and alerts built on them need an observer that produces what they read. The [Observability](/posts/eventstore-observability-micrometer-prometheus-grafana/) page walks through writing one for Micrometer, including the `notifications.up` gauge, and the PromQL to go with it.

The one signal to restore first is notification health: alert on it through `notificationChannelChanged`, or poll `PostgresEventStorage.isNotificationsAvailable()` from a health endpoint.
