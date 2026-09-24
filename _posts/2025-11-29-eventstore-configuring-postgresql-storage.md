---
layout: post
toc: true
title: PostgreSQL EventStorage
description: Configuring PostgreSQL EventStorage
date: 2025-11-29 05:00:00
categories: [Eventstore Documentation,Eventstore deployment]
tags: [postgres,eventstorage,database]
---

This guide covers how to configure and deploy the PostgreSQL-backed EventStore implementation in production environments.

## Configuring the Postgres EventStorage

To use PostgreSQL as your event storage backend, add the following dependencies to your Maven `pom.xml`:

```xml
<dependencies>
    <!-- PostgreSQL EventStorage implementation -->
    <dependency>
        <groupId>org.sliceworkz</groupId>
        <artifactId>sliceworkz-eventstore-infra-postgres</artifactId>
    </dependency>

    <!-- PostgreSQL JDBC driver -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <version>42.7.13</version>
    </dependency>
</dependencies>
```

The PostgreSQL driver is marked as `provided` scope in the library, allowing you to choose your preferred version — the BOM manages the eventstore modules, not third-party libraries, so give it a version. HikariCP, used for connection pooling, comes with the backend transitively.

`build()` returns a `PostgresEventStorage` — an `EventStorage` that also answers `isNotificationsAvailable()`, the one thing the PostgreSQL backend has to say that the storage contract does not cover. Keep that handle rather than widening it to `EventStorage` where a health check needs it. The implementation classes themselves are package-private: the builder is the only way to obtain a storage, which is what guarantees its version detection, its bounded startup and its pool ownership rules always apply. A builder can be reused: every `build()` resolves its own pools.

## PostgreSQL Version Support

**The oldest supported PostgreSQL is 16**, and **18+ is the version the library is built around**.

The floor is 16 rather than something older for two reasons. It is the oldest version with a support life worth committing to — 13 went end-of-life in November 2025, 14 follows in November 2026, 15 in November 2027 — and it is the oldest version the DCB consistency check actually works on: a conditional append uses a `VALUES` clause in a `FROM` position, and PostgreSQL only made the alias optional there in 16. On 15 and older, every conditional append fails.

An older server is **warned about, not rejected**. A hard failure would turn a library upgrade into an outage, so the store starts and logs a WARN naming the version:

```
PostgreSQL major version 15 is older than the oldest supported version 16 —
this configuration is untested and unsupported; plan an upgrade
```

The compliance suite runs against **16, 17 and 18**, so the floor is exercised on every build rather than merely claimed.

### UUIDv7 Generation

Event ids are stored as time-ordered `UUIDv7` values (per [RFC 9562](https://www.rfc-editor.org/rfc/rfc9562#section-5.7)), which improves B-tree index locality for append-heavy workloads. How they are generated depends on the server version:

| PostgreSQL version | UUIDv7 generation            | Implementation                   | Extra runtime dependency                                    |
|--------------------|------------------------------|----------------------------------|-------------------------------------------------------------|
| **18+**            | Server-side `uuidv7()`       | `PostgresEventStorageImpl`       | none                                                         |
| 16–17              | Java-side via `uuid-creator` | `PostgresLegacyEventStorageImpl` | `com.github.f4b6a3:uuid-creator` (must be added explicitly)  |

The right implementation is **picked automatically at `build()` time**: the builder borrows a connection, reads the server's major version and selects the matching implementation. The same library binary therefore works against 16, 17 and 18 — no code or property changes when you upgrade your database.

The chosen path is logged at startup; grep for `uuidv7` to see which one was selected:

```
PostgreSQL major version 18 detected — using native server-side uuidv7() via PostgresEventStorageImpl
```

```
PostgreSQL major version 17 detected — using Java-side uuidv7 generation via PostgresLegacyEventStorageImpl
```

Version detection failing does **not** fail the build: it logs a WARN and falls back to the legacy implementation. It is the schema work, not the version probe, that provides fail-fast behaviour on an unreachable database.

### Optional `uuid-creator` Dependency (for PostgreSQL 16–17)

Because Java-side UUIDv7 generation is only needed below 18, `com.github.f4b6a3:uuid-creator` is declared as **optional** in `sliceworkz-eventstore-infra-postgres`. Applications targeting 18+ only can ignore it and keep a smaller dependency tree.

Applications that may connect to 16 or 17 must declare it explicitly, with a version — the eventstore BOM manages the eventstore modules, not third-party libraries:

```xml
<dependency>
    <groupId>com.github.f4b6a3</groupId>
    <artifactId>uuid-creator</artifactId>
    <version>6.1.1</version>
</dependency>
```

If the legacy path is selected at runtime but `uuid-creator` is missing from the classpath, `build()` fails fast with an `EventStorageException` that names the remedy:

> `… — Java-side uuidv7 generation is required, but the optional 'com.github.f4b6a3:uuid-creator' dependency is missing from the classpath. Either add it to your application's build (see PostgresEventStorage Javadoc for the dependency snippet), or upgrade the PostgreSQL server to version 18+ for native uuidv7() support.`

> **Future-proofing.** Once your deployments have all moved to 18+, drop the legacy dependency from your build. The library continues to function unchanged — `PostgresEventStorageImpl` becomes the sole implementation.
{: .prompt-tip }

## The `btree_gin` Extension

The schema needs the `btree_gin` extension. It backs `idx_events_stream_tags`, the combined stream + tags GIN index that serves DCB reads scoping by stream **and** filtering by tags in one index, and schema validation requires that index to exist.

`btree_gin` is a standard contrib extension, available on the major managed PostgreSQL offerings, and it is *trusted* — so installing it involves no superuser. It does, however, need `CREATE` on the **database**, which is a different privilege from `CREATE` on the schema.

> This is the one place that difference shows, and it bites the ordinary locked-down deployment: a role granted `CREATE` on its schema and nothing on the database creates every table, index, function and trigger here and then cannot create the extension. The schema scripts run as one transaction, so that is not a missing index — the whole schema rolls back and the store does not start.
{: .prompt-warning }

Two ways to run `ENSURE` under such a role, both fine:

```sql
-- recommended: a DBA installs it once, and the application role never needs the privilege
CREATE EXTENSION btree_gin;

-- or: grant it, for the first start at least
GRANT CREATE ON DATABASE <database> TO <role>;
```

The first is the recommended split, and it costs the application role nothing afterwards: the schema script pre-checks `pg_extension` and skips the statement entirely when the extension is present, so an unprivileged role starts against it indefinitely — not even a `NOTICE` in the log. A store that has neither option fails to start with an error naming both remedies rather than a bare `permission denied to create extension`.

**Where the extension lives does not matter.** `CREATE EXTENSION btree_gin SCHEMA extensions`, the convention on several managed offerings, serves the index with no `search_path` change and no `USAGE` grant on that schema — resolving the default GIN operator class is not `search_path`-filtered.

Check a role before deploying it:

```sql
SELECT has_database_privilege('<role>', current_database(), 'CREATE') AS can_create_extension,
       has_schema_privilege('<role>', current_schema(), 'CREATE')     AS can_create_tables,
       EXISTS (SELECT 1 FROM pg_extension WHERE extname = 'btree_gin') AS extension_installed;
```

`can_create_extension` only has to be true when `extension_installed` is false.

## Database Privileges

What the application role needs depends on the `DatabaseInitMode` it starts with:

| Mode | What the role must be allowed to do |
|---|---|
| `NONE`, `VALIDATE` | `CONNECT`, `USAGE` on the schema, and no DDL at all |
| `ENSURE` (default) | the above, plus `CREATE` on the **schema** — and, *only if `btree_gin` is not installed yet*, `CREATE` on the **database** |
| `RECREATE` | the above, plus ownership of the store's tables and functions, since it drops them |

Every mode needs these grants at runtime. They come for free when the role created the tables itself; when a DBA created them, they have to be granted:

```sql
GRANT SELECT, INSERT                 ON <prefix>events           TO <role>;
GRANT SELECT, INSERT, UPDATE, DELETE ON <prefix>bookmarks        TO <role>;
GRANT SELECT, INSERT, UPDATE         ON <prefix>leases           TO <role>;
GRANT SELECT, INSERT, UPDATE, DELETE ON <prefix>lease_contenders TO <role>;
GRANT SELECT, INSERT, UPDATE         ON <prefix>shredding_keys   TO <role>;
GRANT USAGE                          ON SEQUENCE <prefix>events_event_position_seq TO <role>;
```

Events are never updated or deleted — the store only ever appends to that table. Lease rows are never deleted either — a release backdates the heartbeat so the fencing token survives — so the leases table needs no `DELETE`; contender rows *are* pruned, so that table does. See [Leader Election with Leases](/posts/eventstore-leader-election/).

**The shredding key table needs no `DELETE` either, and deliberately so.** Erasing a data subject *updates* the row: it nulls `key_material` and stamps `shredded_at` and `shredded_reason`. Keeping the row is what leaves the erasure an audit trail — the events themselves record nothing about it — and what lets a key id keep resolving to "erased" rather than to "unknown". An unknown key id fails the read, since it means the events were sealed against another store. Granting `DELETE` here would let an erasure be made untraceable, and turn the deleted subject's events unreadable. See [Erasing Personal Data](/posts/eventstore-erasing-personal-data/).

### A Role That Must Not Read Personal Data

A role that reads the events but never the personal data in them — a reporting service — is granted every column of the key table *except* `key_material`:

```sql
GRANT SELECT (key_id, subject_type, subject_id, subject_category, created_at, shredded_at, shredded_reason)
    ON <prefix>shredding_keys TO <reporting_role>;
```

Resolving a key under that role fails with `insufficient_privilege` (SQLSTATE `42501`), which the key store reports as a *denial* rather than an outage: every protected value reads as `Shreddable.Withheld`, projections advance, and the [audit](/posts/eventstore-erasing-personal-data/#auditing-what-is-protected-and-what-was-erased) still works — it never references `key_material`, since PostgreSQL checks `SELECT` privilege on every column a statement mentions, `WHERE` clauses included. Because `information_schema` shows a role only the columns it may read, schema validation would report `key_material` as missing: such a role starts its store with `DatabaseInitMode.NONE`.

Row-level security on the key table does **not** produce a denial: a hidden row is indistinguishable from an absent one, which is a key the store never held, and that fails the read. See [Readers That May Not See Personal Data](/posts/eventstore-erasing-personal-data/#readers-that-may-not-see-personal-data).

## Using Your Own DataSource

If your application already manages database connections, pass your existing `DataSource` to the builder:

```java
DataSource existingDataSource = // ... from your application context

PostgresEventStorage storage = PostgresEventStorage.newBuilder()
    .dataSource(existingDataSource)
    .build();

EventStore eventStore = EventStore.on(storage).build();
```

> `build()` returns a `PostgresEventStorage`; `buildStore()` returns a ready-to-use `EventStore` and, because it created the storage itself, is the only handle on it — so closing that store closes the storage too. See [Lifecycle and Shutdown](/posts/eventstore-lifecycle/).
{: .prompt-info }

A `DataSource` you pass in this way is **never closed** by the storage. One the builder creates itself from `db.properties` is. If you supply the pool, close the storage before closing the pool.

This approach is useful when:
- Using application server connection pools (e.g., Tomcat, WildFly)
- Integrating with Spring's DataSource management
- Sharing a connection pool across multiple components
- Using custom DataSource implementations

When providing a custom DataSource, ensure it's properly configured with:
- Sufficient connection pool size for your workload
- Appropriate connection timeout settings
- PostgreSQL-specific optimizations if using HikariCP

## Regular and Monitoring Connections

The EventStore uses two types of database connections:

**Regular DataSource**: Used for standard operations (queries, appends, bookmark management). Should use connection pooling for performance.

**Monitoring DataSource**: Used exclusively for PostgreSQL's LISTEN/NOTIFY mechanism to receive real-time event notifications.

```java
DataSource pooledDataSource = // ... HikariCP pooled connections
DataSource monitoringDataSource = // ... separate non-pooled connection

EventStore eventStore = PostgresEventStorage.newBuilder()
    .dataSource(pooledDataSource)
    .monitoringDataSource(monitoringDataSource)
    .buildStore();
```

### Why Separate Datasources?

PostgreSQL's LISTEN/NOTIFY requires dedicated, long-lived connections that cannot be pooled. Using a separate monitoring DataSource:

- **Optimization**: Configure a short leakDetection time for regular connections, and a longer one for monitoring event appends via LISTEN/NOTIFY
- **Prevents blocking**: Long-running LISTEN connections don't consume regular connection pool resources
- **PgBouncer compatibility**: Connection poolers like PgBouncer don't support LISTEN/NOTIFY in transaction pooling mode. A separate direct connection bypasses this limitation
- **Performance**: Isolates notification traffic from query traffic

If you don't provide a separate monitoring DataSource, the regular DataSource is used for both purposes.

The monitoring connections include built-in resilience: if a LISTEN connection drops, it automatically reconnects with exponential backoff (1 second up to 30 seconds) to avoid flooding logs or exhausting the connection pool during database outages.

A connection can also die **silently**: a NAT or firewall that dropped its state, a partition, a crashed host. The monitors wait for notifications with a bare socket read and send nothing meanwhile, so such a connection would otherwise look exactly like a quiet channel, reported up forever. Every monitoring connection therefore runs under a 5-second network timeout, and a monitor that has heard nothing for `notificationProbeInterval` (30 seconds by default) sends one round trip and replaces a connection that does not answer. A busy channel is never probed. See [Timeouts](#timeouts-a-stalled-lock-holder-and-a-socket-that-dies-silently).

Nothing that arrives on a channel can take a monitor down, either. A `NOTIFY` channel is a database-wide name that any session can publish on; a payload that does not parse is logged at ERROR and dropped, and a listener that throws is contained. The monitor reads on.

When connecting to a database through pbBouncer for the monitoring, you will find the realtime notification mechanism not to react to appends immediately, but only after 30 seconds or so.  While this functionally works, your expectations towards eventual consistency keeping up are without a doubt higher than that.

### Startup Waits for the Notification Channels

`build()` finishes by starting the two LISTEN/NOTIFY monitors and waiting for them to register their channels — a wait bounded at **10 seconds** by default. When the deadline passes, the storage is closed and an `EventStorageException` is thrown.

Failing is deliberate, and there is no mode that starts anyway. An application that is never told when events are appended has read models that quietly stop advancing: nothing wakes a subscriber, so it serves stale data with nothing in its own logs to say so.

Where startup legitimately races the database coming up — container orchestration, a simultaneous restart — raise the deadline rather than removing it:

```java
EventStore eventStore = PostgresEventStorage.newBuilder()
    .notificationStartupTimeout(Duration.ofSeconds(30))
    .buildStore();
```

Within the deadline, the monitors' own retry loop does the waiting, so a store racing its database up succeeds. Note that a *running* store repairs itself the same way after an outage — the fail-fast is about not starting blind, not about tearing a live store down when its connection drops.

With `ENSURE` or `VALIDATE` the schema work runs first, and under `NONE` the [restored-history check](#backup-and-restore) does — so a dead *main* DataSource fails there, with a clear error, and never reaches the wait. The configuration that realistically does is a **reachable main DataSource with an unreachable monitoring one**. The two are configured separately precisely because LISTEN/NOTIFY does not survive a transaction pooler, so "pooled works, direct is firewalled" is an ordinary misconfiguration.

For monitoring this in production, an [observer](/posts/eventstore-observability-micrometer-prometheus-grafana/#notification-channel-health) is told whenever a channel goes up or down, and `PostgresEventStorage.isNotificationsAvailable()` answers the same state for a health endpoint.

## Configuring an EventStore-managed DataSource (db.properties)

Without a `DataSource`, the builder describes its pools from a `db.properties` file (template: [`src/main/quickstart/db.properties`](https://github.com/sliceworkz/eventstore/tree/develop/sliceworkz-eventstore-infra-postgres/src/main/quickstart)). It takes the first of these that is present:

1. a `DataSource` passed to `.dataSource(...)` (and `.monitoringDataSource(...)`)
2. `Properties` or a file passed to `.configuration(...)`
3. the file named by the system property `eventstore.db.config` (`-Deventstore.db.config=/path/to/db.properties`)
4. the file named by the environment variable `EVENTSTORE_DB_CONFIG`
5. `./db.properties` in the working directory of the process
6. `db.properties` at the root of the classpath (`src/main/resources` in a Maven project)

The lookup **never walks into parent directories**. When nothing is found, `build()` throws an `EventStorageException` naming every location it tried. Several stores in one process are configured by giving each builder its own `.configuration(...)`, or by sharing one pool between them through `.dataSource(...)` when they live in one database under different prefixes.

```java
EventStore store = PostgresEventStorage.newBuilder()
    .configuration(Path.of("/etc/myapp/eventstore.properties"))
    .buildStore();
```

### Configuring the pooled and non-pooled connections

Define separate datasources. Keys inside a section are HikariCP properties; keys under `datasource.` are handed to the PostgreSQL JDBC driver as they are:

```properties
# db.properties

# Pooled connections for regular operations
db.pooled.url=jdbc:postgresql://<host>/<db>
db.pooled.username=<user>
db.pooled.password=<password>
db.pooled.leakDetectionThreshold=2000
db.pooled.maximumPoolSize=25
db.pooled.datasource.sslmode=require
db.pooled.datasource.channelBinding=require

# Non-pooled connections for LISTEN/NOTIFY
db.nonpooled.url=jdbc:postgresql://<host>/<db>
db.nonpooled.username=<user>
db.nonpooled.password=<password>
db.nonpooled.leakDetectionThreshold=70000
db.nonpooled.maximumPoolSize=2
db.nonpooled.datasource.sslmode=require
db.nonpooled.datasource.channelBinding=require
```

Be sure to size your pooled datasource connections according to your application needs.
The non-pooled datasource used for monitoring appends with the NOTIFY/LISTEN mechanism only needs 2 connections.

leakDetectionThreshold on the pooled (application) connections should be set quite low, for the monitoring connections this should be at least 30 seconds, as the monitoring connection only refreshes after a longer LISTEN for updates.

The driver's prepared statements need no configuration: they are cached client-side by default and server-prepared from the fifth execution on. The knobs, should you want them, are `preparedStatementCacheQueries`, `preparedStatementCacheSizeMiB` and `prepareThreshold` — not the `cachePrepStmts` family, which the PostgreSQL driver silently ignores.

## Timeouts: a Stalled Lock Holder, and a Socket That Dies Silently

Two waits in this backend have no natural end, and the builder bounds both:

```java
PostgresEventStorage storage = PostgresEventStorage.newBuilder()
    .lockTimeout(Duration.ofSeconds(10))                // default; Duration.ZERO waits without bound
    .notificationProbeInterval(Duration.ofSeconds(30))  // default
    .build();
```

**`lockTimeout`** bounds how long a conditional append waits for its stream's [advisory lock](#how-concurrent-appends-stay-safe) (and a lease request for its lease's lock). A healthy holder releases it within one INSERT, so ordinary contention never hits the bound; a *stalled* holder does — a paused process, a session the server still believes in after its client has gone. Without the bound, every conditional append to that stream parks behind the holder inside a checked-out pool connection until the pool is empty, and from then on every operation of the store fails on the pool's connection timeout, reads included. With it, the parked appends fail one at a time with an `EventStorageException` naming the stream and the bound (the cause carries SQLSTATE `55P03`), nothing is written, and the store stays up for everything else. The bound is sent as `SET LOCAL lock_timeout`, so it lives and dies with the append's transaction. Find the holder with:

```sql
SELECT l.objid, a.pid, a.state, a.xact_start, a.application_name, a.client_addr, a.query
FROM pg_locks l JOIN pg_stat_activity a ON a.pid = l.pid
WHERE l.locktype = 'advisory' AND l.granted;
```

**`notificationProbeInterval`** bounds how long a LISTEN/NOTIFY monitoring connection may stay silent before its monitor checks it is still there — see [Regular and Monitoring Connections](#regular-and-monitoring-connections).

Both are library-level bounds, independent of the driver's. Two driver settings complement them and are yours to set under `datasource.`: `tcpKeepAlive=true` detects the same dead socket, but only after the operating system's keepalive time (two hours by default on Linux); `socketTimeout=<seconds>` bounds *every* read on that pool, so set it only above the longest statement the store legitimately runs — a large import batch, a `CREATE INDEX` under `ENSURE` — or it becomes the failure it was meant to catch.

The defaults are public constants on the builder: `DEFAULT_LOCK_TIMEOUT`, `DEFAULT_NOTIFICATION_PROBE_INTERVAL` and `DEFAULT_NOTIFICATION_STARTUP_TIMEOUT`.

## Connection Pool Metrics

`poolMetrics(MetricsTrackerFactory)` hands HikariCP's own metrics seam to the pools this storage uses — the main and the monitoring pool, whether the builder created them or you supplied them (a pool that already has a tracker keeps it). HikariCP ships a tracker factory for Micrometer and one for the Prometheus client, so pool metrics land wherever you measure, while this library names no metrics library at all:

```java
PostgresEventStorage.newBuilder()
    .poolMetrics(new MicrometerMetricsTrackerFactory(meterRegistry))
    .build();
```

Not set by default: the pools then report nothing. What the *store* reports goes through its observer — see [Eventstore Observability](/posts/eventstore-observability-micrometer-prometheus-grafana/).

## Preparing the Database Schema Manually via DDL

The recommended approach is to create the database schema manually using DDL scripts before deploying your application. This allows you to:

- Use database users with limited DML-only privileges for the application
- Separate schema management from application deployment
- Apply schema changes through controlled migration processes when a newer EventStore version ever requires schema updates

### Using the Initialization Script

The library includes a `quickstart.ddl.sql` script (unprefixed, ready to run) alongside the prefixed `ensure-schema.sql`. Both create the same objects — the events table, the bookmarks table, the two lease tables, the shredding key table, their indexes, and the notification functions and triggers.

```sql
CREATE TABLE IF NOT EXISTS events (
    -- Primary key and positioning
    event_position BIGSERIAL PRIMARY KEY,

    -- XID8 transaction id
    event_tx xid8 DEFAULT pg_current_xact_id()::xid8 NOT NULL,

    -- Event identification
    event_id UUID NOT NULL UNIQUE,

    -- Idempotency key: uniqueness is scoped per stream by the partial
    -- unique index below, not globally
    idempotency_key TEXT,

    -- Stream identification
    stream_context TEXT NOT NULL,
    stream_purpose TEXT NOT NULL DEFAULT 'default',

    -- Event metadata
    event_type TEXT NOT NULL,

    -- Transaction information
    event_timestamp TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,

    -- Event payload
    event_data JSONB NOT NULL,

    -- Tags as string array
    event_tags TEXT[] DEFAULT '{}'

) WITH (FILLFACTOR = 100);
```

The `stream_purpose` default matches `EventStreamId.DEFAULT_PURPOSE`, a public constant — so an interop layer doing raw SQL inserts can bind the same value the library does rather than copy the literal out of a script.

These indexes are created on the events table:

| Index | Purpose |
|---|---|
| `idx_events_global_order` | B-tree on `(event_tx, event_position)` — the global order, for reads that bind **no** stream column: a wildcard stream, `head()` of the whole store, an unscoped import, the startup check. Partial on `event_position > 0` |
| `idx_events_context_order` | B-tree on `(stream_context, event_tx, event_position)` — for reads that bind the context and leave the purpose open: a whole-context replay over a context split into a stream per entity. Partial on `event_tx > '0'::xid8` |
| `idx_events_stream_type_position` | B-tree: stream + event type, ordered by `(event_tx, event_position)` |
| `idx_events_stream_position` | B-tree: ordered stream replay, and `head()` of a stream |
| `idx_events_tags` | GIN over the tag array, for tag-only lookups |
| `idx_events_stream_tags` | GIN over stream columns **and** tags — the DCB read path. Requires `btree_gin` |

The B-tree indexes are retained alongside the GIN ones because GIN cannot serve `ORDER BY`, and ordered stream replay needs it.

**The two order indexes are partial on a tautology, and that is the point.** Both predicates hold for every row — `event_position` is a `bigserial` starting at 1, and `event_tx` never holds transaction id 0 — so each index covers the whole table. What the predicate decides is who may *enter* it: PostgreSQL admits a partial index only to a statement whose own predicates imply the index's, and it cannot prove a tautology on a column with no `CHECK` constraint by itself. The store spells the predicate out for exactly the reads whose scope binds no more than that index leads with, and for no others.

Without that, a read of one stream could be served by a wider order index — walking the global order and filtering on the stream columns — whenever the stream is a large share of the table. For a stream that has been quiet while others wrote, that walk covers everything written since: measured on a 500.000-event table with a quiet stream 300.000 events back, `head()` of that stream took 24 ms against 0.08 ms off its own index, and its consistency check 56 ms against 0.02 ms. Correct, unlogged, and growing with the table. A partial index is closed to a statement that does not imply its predicate whatever the estimate, so this holds for a cached generic plan too.

Idempotency uniqueness is a **partial** unique index, so events without a key are not indexed at all:

```sql
CREATE UNIQUE INDEX IF NOT EXISTS idx_events_stream_idempotency ON events (
    stream_context,
    stream_purpose,
    idempotency_key
) WHERE idempotency_key IS NOT NULL;
```

Scoping it to `(stream_context, stream_purpose)` is what makes the same key usable on two unrelated streams without colliding. It is also named deliberately: the store recognises a duplicate append by the **index name PostgreSQL reports in the error**, never by matching message text, so renaming it breaks idempotency detection.

Beyond the tables, the script creates two `plpgsql` functions and their triggers, which drive LISTEN/NOTIFY. Extract the script from the JAR or copy it from the source repository, then execute it with your preferred database client or migration tool.

> Identifier length is a real constraint here. PostgreSQL truncates identifiers at 63 bytes, and the 32-character prefix cap keeps the longest generated index name at 61. Do not lengthen the generated names.
{: .prompt-warning }

### The Bookmarks Table and Its Foreign Key

```sql
CREATE TABLE IF NOT EXISTS bookmarks (
    reader TEXT PRIMARY KEY,
    event_id UUID NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_tags TEXT[] DEFAULT '{}',
    CONSTRAINT fk_bookmarks_event_id
        FOREIGN KEY (event_id)
        REFERENCES events(event_id)
);

CREATE INDEX IF NOT EXISTS idx_bookmarks_event_id ON bookmarks(event_id);
```

**A bookmark stores the event id and nothing else about the event.** `getBookmark` and `getBookmarks` join the events row on its unique `event_id` index — one probe — to answer the bookmark's transaction and position, and the bookmark trigger does the same for its notification payload. So a bookmark always reads back with the store's own coordinates for the event it names, and a bookmarks table copied between stores by id is valid as it stands.

The foreign key deliberately does **not** cascade: an event deletion would otherwise silently remove the bookmarks of the readers still pointing into the deleted range — the lagging ones — and an absent bookmark means "replay from the beginning". With the default `NO ACTION`, deleting events out from under an outstanding bookmark fails loudly.

The foreign key is what makes `placeBookmark` reject a reference this store never stored — the realistic mistake being a reference carried over from a *different* store or prefix. The store recognises the violation by the **constraint name** the server reports, exactly as it does for the idempotency index, so renaming it turns a clear `EventStorageException` back into an opaque SQL failure. The in-memory backends enforce the same rule against their own log, so the contract is identical on every backend — see [Bookmarking](/posts/eventstore-bookmarking/#the-reference-must-name-a-stored-event).

### The Lease Tables

Leader election is backed by two tables that sit deliberately **outside** the event log:

```sql
CREATE TABLE IF NOT EXISTS leases (
    lease_name TEXT PRIMARY KEY,
    lease_owner TEXT NOT NULL,
    priority BIGINT NOT NULL,
    fencing_token BIGINT NOT NULL,
    ttl_millis BIGINT NOT NULL,
    acquired_at TIMESTAMP WITH TIME ZONE NOT NULL,
    heartbeat_at TIMESTAMP WITH TIME ZONE NOT NULL
);

CREATE TABLE IF NOT EXISTS lease_contenders (
    lease_name TEXT NOT NULL,
    contender TEXT NOT NULL,
    priority BIGINT NOT NULL,
    ttl_millis BIGINT NOT NULL,
    heartbeat_at TIMESTAMP WITH TIME ZONE NOT NULL,
    PRIMARY KEY (lease_name, contender)
);
```

Election traffic never touches the events table, takes no lock that any query or append takes, and neither pins nor waits on the `pg_snapshot_xmin` barrier — which is also why a lease is not modelled as events. Writes are serialized per lease by a transaction-scoped advisory lock, and every timestamp in these tables is written and compared with the **database server's clock**. See [Leader Election with Leases](/posts/eventstore-leader-election/).

### The Shredding Keys Table

Personal data in event payloads is protected by [crypto-shredding](/posts/eventstore-erasing-personal-data/): values are encrypted under a key held per data subject, and an erasure destroys the key instead of touching an event. `PostgresEventStorage.newBuilder().shredding()` keeps those keys in this store's own database, on the same `DataSource` as the events.

Sharing the `DataSource` buys the schema machinery, one set of credentials, and a backup carrying both. It does **not** put a key and the event sealed under it in one transaction: a key is minted on a connection of its own and committed *before* the append it seals for. A rolled-back append therefore leaves a key row with no event under it, which the subject's next append seals under — and an event whose key was never persisted cannot happen.

```sql
CREATE TABLE IF NOT EXISTS shredding_keys (
    key_id TEXT PRIMARY KEY,
    subject_type TEXT NOT NULL,
    subject_id TEXT NOT NULL,
    subject_category TEXT NOT NULL,
    key_material BYTEA,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    shredded_at TIMESTAMP WITH TIME ZONE,
    shredded_reason TEXT
);

-- Serves both hot paths: resolving the active key for a subject on append, and finding every key a
-- subject holds when erasing. Partial on the un-shredded rows, because that is what both look for and
-- because the shredded rows accumulate for good.
CREATE UNIQUE INDEX IF NOT EXISTS idx_shredding_keys_active
    ON shredding_keys (subject_type, subject_id, subject_category)
    WHERE key_material IS NOT NULL;

-- Erasure looks up every key ever minted for a subject, shredded ones included: a subject appended for
-- after an earlier erasure holds a second key, and missing it would leave that data readable.
CREATE INDEX IF NOT EXISTS idx_shredding_keys_subject
    ON shredding_keys (subject_type, subject_id, subject_category);
```

`key_material` is nullable and that is the whole mechanism: an erasure nulls it, stamps `shredded_at` and `shredded_reason`, and keeps the row. The unique index is partial on `key_material IS NOT NULL`, so a subject holds exactly one *live* key per category while every key it ever held stays on record.

There is deliberately **no foreign key** between this table and the events table, in either direction. Events name their keys through ordinary `dek:` tags; a constraint would either block pruning events or cascade keys away with them — and a cascade here would erase data nobody asked to erase.

### Append Notifications

The trigger on the events table is `AFTER INSERT ... REFERENCING NEW TABLE AS inserted FOR EACH STATEMENT`, and the function emits **one `pg_notify` per distinct stream touched by the statement**, not one per row. A 1000-event append therefore queues one notification per stream rather than 1000.

The reference each notification carries is the maximum over the total `(event_tx, event_position)` order — not the maximum position. The two genuinely disagree, and a notification naming a reference a reader has already passed is dropped, so this distinction is what keeps subscriptions alive rather than silently stranded.

The bookmark trigger is `FOR EACH ROW`, because placing a bookmark is a single-row upsert: per-row and per-statement are the same count there.

### Co-locating Multiple EventStore Instances

In some scenarios, you want to create multiple eventstores, without configuring multiple databases:
- separating environments like dev(development), tst(testing) and acc(acceptance)
- separating tenants like customer1, customer2, etc...
- separating application components or bounded contexts (sales, orders, invoicing, ...)

For these scenario's, EventStore supports the usage of prefixes, in which all required database objects are uniquely identified by prefixing them.

The library includes two DDL scripts with "PREFIX_" placeholders for multi-tenant deployments:

**`ensure-schema.sql`** — the creation script, written to be safe to run repeatedly. Tables and indexes use `IF NOT EXISTS`, the two notification functions are `CREATE OR REPLACE`d, and each trigger is compared against the shape this release expects and recreated only when it differs:

```sql
CREATE TABLE IF NOT EXISTS PREFIX_events (
    event_position BIGSERIAL PRIMARY KEY,
    event_tx xid8 DEFAULT pg_current_xact_id()::xid8 NOT NULL,
    event_id UUID NOT NULL UNIQUE,
    idempotency_key TEXT,
    stream_context TEXT NOT NULL,
    stream_purpose TEXT NOT NULL DEFAULT 'default',
    event_type TEXT NOT NULL,
    event_timestamp TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    event_data JSONB NOT NULL,
    event_tags TEXT[] DEFAULT '{}'
) WITH (FILLFACTOR = 100);

-- indexes, the btree_gin guard, functions and triggers follow,
-- then the bookmarks, lease and shredding key tables
```

**`drop-schema.sql`** — teardown for an existing eventstore schema. It drops the functions as well as the tables, because the triggers go with the tables via `CASCADE` but the functions do not:

```sql
DROP TABLE    IF EXISTS PREFIX_bookmarks CASCADE;
DROP TABLE    IF EXISTS PREFIX_events    CASCADE;
DROP FUNCTION IF EXISTS PREFIX_notify_event_appended()  CASCADE;
DROP FUNCTION IF EXISTS PREFIX_notify_bookmark_placed() CASCADE;
DROP TABLE    IF EXISTS PREFIX_leases           CASCADE;
DROP TABLE    IF EXISTS PREFIX_lease_contenders CASCADE;
DROP TABLE    IF EXISTS PREFIX_shredding_keys   CASCADE;
```

By replacing all occurrences of "PREFIX_" in these files by e.g. "myapp_", you create a private eventstorage to be used by a specific application.


### Connecting to an EventStore database with prefixes

Configure each instance with its prefix:

```java
EventStore tenant1Store = PostgresEventStorage.newBuilder()
    .prefix("tenant1_")
    .buildStore();

EventStore tenant2Store = PostgresEventStorage.newBuilder()
    .prefix("tenant2_")
    .buildStore();
```

**Note**: Prefixes must be ASCII letters, digits and underscores, must not start with a digit, must end with an underscore, and be 32 characters or less. The 32-character cap is load-bearing: it keeps the longest generated index name inside PostgreSQL's 63-byte identifier limit, and a truncated index name would silently break both schema validation and idempotency detection.

**A prefix is folded to lowercase**, because that is the name PostgreSQL gives every object it is used on unquoted: a store configured with `Tenant1_` *is* the store whose tables are `tenant1_*`, and the store compares names, channels and lock keys in that folded form too. A leading digit is rejected, since `1tenant_events` is not an identifier PostgreSQL parses unquoted at all.

**Security Best Practice**: Create the schema with a privileged database user (e.g., `eventstore_admin` with DDL rights), then run your application with a limited user (e.g., `eventstore_app` with only DML rights). This prevents applications from accidentally modifying the schema.

## Database Initialization Modes

The `DatabaseInitMode` enum controls how the database schema is handled at startup. Set it via `.databaseInitMode(DatabaseInitMode.xxx)` or use one of the convenience methods on the builder.

### ENSURE (default)

Brings the schema up to what this release expects, then validates it. Safe to run repeatedly, and safe to run from several instances at once. `ENSURE` is also the mode that *initializes* a new database — there is no separate "initialize" mode:

```java
// default behaviour — no need to specify
EventStore eventStore = PostgresEventStorage.newBuilder()
    .buildStore();

// or explicitly
EventStore eventStore = PostgresEventStorage.newBuilder()
    .ensureDatabase()
    .buildStore();
```

What "brings up to date" means differs per kind of object, and the difference matters:

| Object | What ENSURE does |
|---|---|
| Tables, columns | **Created if absent**, never altered |
| Indexes | **Created if absent**, never rebuilt |
| The `btree_gin` extension | Created if absent, skipped entirely when already present |
| Functions | **`CREATE OR REPLACE`d every time** — the body always matches this release |
| Triggers | Compared against the expected shape (timing, orientation, transition table, target function) and **recreated only when it differs** |

Comparing triggers rather than unconditionally replacing them is what keeps the ordinary startup cheap: when the trigger is already correct, this is a catalog read that takes no lock on the events table at all. `CREATE OR REPLACE TRIGGER` would be simpler, but it rewrites unconditionally and takes `ACCESS EXCLUSIVE` on every start of every instance.

Recreating a trigger whose shape has drifted *does* take a brief `ACCESS EXCLUSIVE` lock — a one-off, since the comparison finds it correct on every later start.

> ENSURE needs `CREATE` on the schema, and — only when `btree_gin` is not installed yet — `CREATE` on the database. See [Database Privileges](#database-privileges) above.
{: .prompt-info }

**All the schema scripts run as a single transaction under a per-prefix advisory lock.** `CREATE TABLE / INDEX / EXTENSION IF NOT EXISTS` is not atomic against a concurrent creator, so without that lock several instances starting together against an empty database race on the system catalogs and most of them fail to start. One transaction across all scripts also makes `RECREATE`'s drop-then-create indivisible, so a second instance cannot drop what the first has just recreated. The lock is keyed on a hash of the table prefix, so two prefixed stores never block each other.

### VALIDATE

Verifies the schema is present, without creating or modifying anything. A read-only check:

```java
EventStore eventStore = PostgresEventStorage.newBuilder()
    .validateDatabase()
    .buildStore();
```

Validation checks that:

- the required tables exist (`PREFIX_events`, `PREFIX_bookmarks`, `PREFIX_leases`, `PREFIX_lease_contenders`, `PREFIX_shredding_keys`)
- every expected column is present, with the right type and nullability
- the expected indexes exist by name — including `idx_events_stream_tags` and `idx_events_stream_idempotency` — and the two order indexes carry their admission predicates
- the bookmarks foreign key exists by name (`fk_bookmarks_event_id`)
- the notification functions exist
- each trigger exists **with the expected orientation** (row-level vs statement-level), not merely by name

If validation fails, an `EventStorageException` names what is missing or misconfigured.

> **What validation does not check.** It verifies that named objects *exist*; apart from the order indexes' predicates, it does not check an index's method, columns or uniqueness, a column's default, a foreign key's delete rule, or a function's body. So an index rebuilt as the wrong kind, or the idempotency index recreated without `UNIQUE`, passes validation. Where a DBA applies the DDL, apply the shipped script rather than hand-written equivalents.
{: .prompt-warning }

Checking the trigger's orientation is worth the extra query, because the failure it prevents is not loud: a statement-level trigger bound to a row-level function body does not raise in PostgreSQL. It emits a notification with every field null, which becomes a wildcard stream with a zero reference that every concrete subscriber rejects — live updates stop with nothing thrown and nothing logged.

### NONE

Skips all schema handling. Trusts that the schema exists and is correct, and minimizes startup time:

```java
EventStore eventStore = PostgresEventStorage.newBuilder()
    .databaseInitMode(DatabaseInitMode.NONE)
    .buildStore();
```

Use this when:
- The database user lacks permissions to query `information_schema` — or is a [reporting role](#a-role-that-must-not-read-personal-data) that may not see every column
- Minimizing startup time is critical
- You have full confidence in the schema being correct

`NONE` is the recommended mode for production with a DBA-managed schema. Even under `NONE`, `build()` runs the one-probe [restored-history check](#backup-and-restore).

### RECREATE

Drops all event store objects — tables **and** functions — recreates them from scratch, then validates. It is the one destructive mode, and named for it:

```java
EventStore eventStore = PostgresEventStorage.newBuilder()
    .recreateDatabase()
    .buildStore();
```

> **This mode is destructive** — all existing event data will be lost. Use only for test environments, fresh deployments, or when a clean slate is explicitly needed.
{: .prompt-danger }

### Schema Changes That No Mode Applies

`ENSURE` only ever *creates* tables, columns and indexes. Anything needing `ALTER TABLE` — changing a column default, altering a constraint, rebuilding an index differently — is outside what any mode does, and has to be applied by hand. Combined with what [validation does not check](#validate), that is the argument for applying the shipped `quickstart.ddl.sql` / `ensure-schema.sql` rather than hand-written equivalents: a schema that differs from it in one of those ways starts cleanly and stays wrong.

> **`VALIDATE` and `NONE` change nothing at all**, including the function bodies — and validation cannot detect a stale body, since it does not compare function source. Where a deployment is pinned to either mode, apply the shipped `quickstart.ddl.sql` / `ensure-schema.sql` as part of the release rather than expecting the application to bring the schema forward.
{: .prompt-warning }

### Recommended Configurations

**Production with external schema management** (DML-only database user):

```java
EventStore eventStore = PostgresEventStorage.newBuilder()
    .validateDatabase()         // Verify schema exists at startup
    .buildStore();
```

**Production with auto-managed schema** (DDL-capable database user):

```java
EventStore eventStore = PostgresEventStorage.newBuilder()
    .buildStore();              // Default ENSURE mode
```

**Test environments:**

```java
EventStore eventStore = PostgresEventStorage.newBuilder()
    .recreateDatabase()         // Fresh schema every test run
    .buildStore();
```

## Configuring a Hard Limit on Query Result Size

To prevent memory exhaustion from poorly designed queries, configure an absolute limit on result set size:

```java
EventStore eventStore = PostgresEventStorage.newBuilder()
    .resultLimit(10000)  // Maximum 10,000 events per query
    .buildStore();
```

When a query would exceed this limit, an `EventStorageException` is thrown:

```java
try {
    // Query that exceeds the limit
    stream.query(EventQuery.matchAll());
} catch (EventStorageException e) {
    // Handle: "Query result exceeded absolute limit of 10000"
}
```

**Important**: This is an emergency brake, not a substitute for proper query design. Applications should:
- Page through large result sets with `EventQuery.limit(n)`
- Design queries to stay well below the hard limit
- Process results incrementally rather than loading everything into memory

Example of proper batched querying:

```java
EventQuery query = EventQuery.matchAll().limit(500);   // well below the hard limit
EventReference cursor = null;
EventPage<CustomerEvent> page;
do {
    page = stream.page(query, cursor);
    page.events().forEach(this::processEvent);
    cursor = page.lastStoredEventReference().orElse(null);
} while (page.storedEventCount() == 500);
```

The hard limit applies globally to all queries on that EventStore instance. Individual queries can use lower limits, but cannot exceed the configured maximum.

## Configuring Multiple EventStore Instances on the Same Database

Multiple EventStore instances can safely connect to the same PostgreSQL database schema simultaneously. 

### Multiple Processes/Machines

In a multi-machine setup, each process connects to the same database:

```java
// Application Server 1
EventStore storeOnServer1 = PostgresEventStorage.newBuilder()
    .name("api-server-1")
    .buildStore();

// Application Server 2
EventStore storeOnServer2 = PostgresEventStorage.newBuilder()
    .name("api-server-2")
    .buildStore();

// Batch Worker
EventStore batchStore = PostgresEventStorage.newBuilder()
    .name("batch-worker")
    .buildStore();
```

You can use different names or the same for the storage one on all machines, this only differs in logging output but has no runtime impact.


All instances share the same event data and can:
- Append events concurrently
- Query the same event streams
- Subscribe to event notifications
- Use optimistic locking (DCB) for safe concurrent writes

### Benefits of Multiple Instances

**Load Balancing**: Distribute query load across multiple application servers

**Failover**: If one instance fails, others continue operating

**Workload Separation**:
- Online instances serve REST APIs and user interactions
- Batch instances run asynchronous projections and integrations
- Dedicated instances for reporting or analytics

**Horizontal Scaling**: Add more instances as load increases

**Example Architecture**:
```
┌─────────────────┐     ┌─────────────────┐
│  API Server 1   │     │  API Server 2   │
│  (Application)  │     │  (Application)  │
└────────┬────────┘     └────────┬────────┘
         │                       │
         └───────────┬───────────┘
                     │
         ┌───────────┴───────────┐
         │  PostgreSQL Database  │
         │   (Shared Database)   │
         └───────────┬───────────┘
                     │
         ┌───────────┴───────────┐
         │   Batch Processor     │
         │     (Application)     │
         └───────────────────────┘
```

PostgreSQL's MVCC (Multi-Version Concurrency Control) ensures safe concurrent access. The DCB optimistic locking mechanism prevents conflicting writes at the application level.

## How Concurrent Appends Stay Safe

The DCB consistency check asks whether an event matching the criteria exists after the expected reference. Under PostgreSQL's default READ COMMITTED isolation, each statement fixes its snapshot when it starts — so two concurrent appends at the same consistency boundary would both find it empty, both insert and both commit. That is a silent DCB violation: the store reports success to both callers and the invariant is gone. The conflicting row is a *phantom* at the moment of the check, so no row lock can cover it.

**Conditional appends therefore take a transaction-scoped advisory lock**, keyed on a hash of the table prefix plus `(stream_context, stream_purpose)`, as its own statement before the check and the insert run.

Things worth knowing about it:

- **Only conditional appends take it.** An append without criteria reads nothing and so cannot observe a stale boundary, which keeps bulk ingestion fully parallel.
- **The key is the stream, not the filter.** Hashing the filter would be finer grained and unsound: two overlapping-but-unequal filters hash differently and would not exclude each other.
- **Cost.** Conditional appends to *one* stream serialize for the duration of a single INSERT, so a hot stream is a throughput ceiling: measured, writers sharing one stream's lock stay flat at ~1.4 appends/ms from one writer to sixteen, where writers spread over entities scale from 5.5 to 24. Keeping each decision's hold on the lock short — take `head()` before reading, and present it as the reference — is what keeps a single context stream scaling; see [Stream Design and Performance](/posts/eventstore-stream-design-and-performance/).
- **The wait is bounded** by `lockTimeout` (10 seconds by default), so a stalled holder fails the appends queued behind it instead of draining the pool — see [Timeouts](#timeouts-a-stalled-lock-holder-and-a-socket-that-dies-silently).
- **Key collisions are harmless.** They only make two unrelated streams take turns; they can never let a real conflict through.

`SERIALIZABLE` would also be correct, but it is a poor fit here: a DCB boundary check always scans the tail of the log, which is exactly where every writer writes, so predicate locks collide constantly. Measured on the same workload it produced 86% serialization failures and a third of the throughput.

### The Check's Shape Follows the Criteria

The SQL the check runs is derived from the criteria, not configured:

- **With an expected reference** — the ordinary case — the check is an ordered probe (`ORDER BY event_tx, event_position LIMIT 1`) that walks the stream's position index forward *from the reference* and stops at the first match. Its cached plan is that walk, so it is stable, and OR-ing several facts together costs little (2.6× at ten OR-ed facts). Its one cost is a **stale cursor**: the walk is linear in the stream events appended since the reference, about 0.2 µs each. Taking the reference from `head()` before the decision's read keeps that walk to the few events appended during the decision — see [Optimistic Locking](/posts/eventstore-appending-events/#why-the-head-and-not-the-last-relevant-event).
- **Without a reference** — "I decided on an empty boundary", the uniqueness pattern — the check is a `NOT EXISTS`, planned from its bound values rather than from a cached generic plan, so the tag index answers it (~2.3 ms at ten million events).

One uniform `NOT EXISTS` for every criteria, left to the plan cache, was measured and rejected: a `NOT EXISTS` is priced by how soon a row turns up, while a DCB check expects none, so the cache settled on plans built for the wrong question — with a cliff at two OR-ed facts and a whole-table scan on the empty boundary.

## What Can Stall Reads: the Visibility Barrier

The read path withholds events whose transaction is still in flight, using `event_tx < pg_snapshot_xmin(pg_current_snapshot())`. That is what keeps a reader from taking a reference past an event that has not committed yet.

`pg_snapshot_xmin` is the oldest transaction id still running — **a property of the whole PostgreSQL cluster, not of this store**. So every event appended since the oldest open transaction took its id is invisible to this store until that transaction ends.

> **Nothing fails and nothing is logged.** Reads just stop advancing: projections go quiet, bookmarks stop moving, `SELECT count(*)` in psql shows the events are there, and when the blocker finally ends everything appears at once.
{: .prompt-warning }

**Only transactions that have written count**, which is what makes this narrow rather than severe. PostgreSQL assigns a transaction id lazily, at the first write, and only assigned ids enter a snapshot's xmin:

- **Harmless at any duration**: `pg_dump`, reporting queries, analytics reads, a replica feed, and an `idle in transaction` connection that only ever read — at any isolation level, including SERIALIZABLE.
- **Not harmless**: a batch job, an ETL run, a migration, or an `idle in transaction` connection that wrote before going idle. `SELECT ... FOR UPDATE` and an explicit `pg_current_xact_id()` also assign an id without writing a row.

The blocker does not have to touch the events table, or even this database — transaction ids are cluster-wide, so a writer in a *different database of the same cluster* stalls this store just as effectively. The operational rule is "do not share a cluster with long-running write transactions", not "do not share a table".

**Append notifications wait for the barrier.** A notification announcing events a query cannot yet see would wake a subscriber that reads nothing and then goes back to sleep — until the next append to its stream, which may be a long time coming. So the append monitor holds a notification back until the event it names is below the barrier: a subscribed projection catches up by itself the moment the blocker ends. A notification withheld for more than 10 seconds is logged at WARN — the one log line the library writes about a stall.

**Read-your-own-writes does not hold while a blocker is open.** A caller can append successfully and not read the event back, because the append-side check deliberately carries no `xmin` filter and sees committed events the reader cannot. Under DCB that surfaces as an optimistic-locking conflict a retry loop **cannot clear**: the decider re-reads its boundary, gets the same stale reference, appends, and conflicts again for as long as the stall lasts.

### Diagnosing It

`backend_xid IS NOT NULL` is the whole predicate. Filtering on `state <> 'idle'` or on the age of `xact_start` reports harmless read-only sessions as suspects:

```sql
SELECT pg_snapshot_xmin(pg_current_snapshot());          -- the barrier

SELECT pid, datname, usename, application_name, state,
       now() - xact_start AS held_for, backend_xid, query
FROM   pg_stat_activity
WHERE  backend_xid IS NOT NULL                            -- only these can stall the store
ORDER  BY xact_start;                                     -- the oldest is the culprit
```

Deliberately unfiltered by `datname`: the culprit may be in another database of the cluster.

> Run this as a superuser or as a member of `pg_read_all_stats`. For anyone else, `pg_stat_activity` blanks `xact_start`, `query` and `state` for other roles' sessions — and blanks them to NULL rather than refusing, so the natural "age of the oldest blocking transaction" check reports a confident all-clear right through a stall another role is causing. `backend_xid` is *not* blanked, so a count of blocking transactions does survive on ordinary privileges; the age does not.
{: .prompt-warning }

### Alerting on It

Nothing the store [reports to an observer](/posts/eventstore-observability-micrometer-prometheus-grafana/) reveals this — every call the store makes keeps succeeding throughout a stall — and the WARN about a withheld notification only appears once one has been held for 10 seconds. Detection is external, on the database.

**Watch `pg_snapshot_xmin` standing still *while* something holds a transaction id**, never either alone. xmin also stops moving on a completely idle database, so "xmin has not advanced" fires on every quiet store; "a transaction holds an xid" fires on every append in flight. It is the combination that means events are being withheld.

Do **not** measure the effect by counting withheld events. `count(*) ... WHERE event_tx >= pg_snapshot_xmin(...)` has no index to use, so it is a sequential scan of the whole events table every time it is sampled.

### The Store Is a Mild Instance of Its Own Hazard

An append in flight holds a transaction id, so a second append that starts later and commits first cannot read its own event back until the first one finishes. That window is one INSERT long and self-clearing — the same mechanism, with a blocker lasting minutes rather than milliseconds, is the hazard above.

## Backup and Restore

**Back the cluster up physically, never with `pg_dump`.** `pg_basebackup` with WAL archiving, pgBackRest, Barman, a filesystem or managed-service snapshot, or a promoted replica all restore the transaction counter intact, so the next transaction id the restored cluster assigns is above every stored `event_tx`. A restore of that kind needs nothing from this library.

**What a logical dump restored into a fresh cluster does.** `pg_dump` copies `event_tx` as plain data, so the restored history keeps the *source* cluster's transaction ids, while a fresh cluster hands out ids from a few hundred. Two things then fail, silently:

- **The history reads as absent.** Every read sits behind the visibility barrier, and restored events carry ids above the new cluster's counter. `SELECT count(*)` in psql shows every row; the store sees none.
- **New events sort before all of history.** An append gets a low id and orders before every restored event, so a projector bookmarked at the old head never advances, and a lock check whose reference is in history sees nothing after it and admits every conflict.

**The store refuses to start in that state.** `build()` checks, under every `DatabaseInitMode`, that the newest stored event does not carry a transaction id at or above the next id the cluster will assign — one probe off `idx_events_global_order`, well under a millisecond whatever the store holds. No append can produce such a row, so a hit is unambiguous and fatal: the storage is closed and `build()` throws an `EventStorageException` naming the highest stored id, the cluster's next id, how many events and streams are affected, and the remedies:

1. **Restore physically instead**, if a physical backup exists.
2. **Move the counter, if nothing has been appended yet**, with `pg_resetwal -e <epoch> -x <xid>` on the stopped cluster — the postgres module README works the arithmetic through from the id the error reports. Managed services do not expose `pg_resetwal`.
3. **Copy the events into a fresh store with [`EventStoreImporter`](/posts/eventstore-importing-events/#moving-a-store-to-another-cluster)**, which lets the target assign both ordering columns. This is the supported way to move a store between clusters: a major-version upgrade by dump, a cloud migration, a change of hosting.

## A Note on `db.properties` and Secrets

A `db.properties` **value** never reaches an error message or a log line — only the key does. `db.<name>.password` goes through the same setter path as every other non-`datasource.` property, so a value interpolated into a message would be a database password in every log, stack trace and error reporter downstream.

A configuration failure therefore names the property and the type the setter expected, and stops there:

```
Error setting property 'maximumPoolSize' (expected int)
```

The detail is left to the cause, which for the realistic cases concerns a property that is never a secret. A stray line with an empty property name (`db.pooled.=x`) is rejected with that explanation rather than failing obscurely.