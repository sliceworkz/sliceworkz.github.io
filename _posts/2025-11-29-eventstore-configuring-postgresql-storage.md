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
    </dependency>

    <!-- HikariCP connection pooling (recommended) -->
    <dependency>
        <groupId>com.zaxxer</groupId>
        <artifactId>HikariCP</artifactId>
    </dependency>
</dependencies>
```

The PostgreSQL driver is marked as `provided` scope in the library, allowing you to choose your preferred version. HikariCP is used for high-performance connection pooling.

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
| `INITIALIZE` | the above, plus ownership of the store's tables and functions, since it drops them |

Every mode needs these grants at runtime. They come for free when the role created the tables itself; when a DBA created them, they have to be granted:

```sql
GRANT SELECT, INSERT                 ON <prefix>events    TO <role>;
GRANT SELECT, INSERT, UPDATE, DELETE ON <prefix>bookmarks TO <role>;
GRANT USAGE                          ON SEQUENCE <prefix>events_event_position_seq TO <role>;
```

Events are never updated or deleted — the store only ever appends to that table.

## Using Your Own DataSource

If your application already manages database connections, pass your existing `DataSource` to the builder:

```java
DataSource existingDataSource = // ... from your application context

EventStorage storage = PostgresEventStorage.newBuilder()
    .dataSource(existingDataSource)
    .build();

EventStore eventStore = EventStoreFactory.get().eventStore(storage);
```

> `build()` returns an `EventStorage`; `buildStore()` returns a ready-to-use `EventStore` and, because it created the storage itself, is the only handle on it — so closing that store closes the storage too. See [Lifecycle and Shutdown](/posts/eventstore-lifecycle/).
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

The configurations that can actually reach this are `DatabaseInitMode.NONE`, where nothing touches the database before the monitors do, and the realistic one: a **reachable main DataSource with an unreachable monitoring one**. The two are configured separately precisely because LISTEN/NOTIFY does not survive a transaction pooler, so "pooled works, direct is firewalled" is an ordinary misconfiguration.

For monitoring this in production, see the `sliceworkz.eventstore.notifications.up` gauge in [Eventstore Observability](/posts/eventstore-observability-micrometer-prometheus-grafana/).

## Configuring an EventStore-managed DataSource (db.properties)

The EventStore can automatically create and configure DataSources from a `db.properties` file. The library searches for this file in the following locations (in order):

1. System property: `-Deventstore.db.config=/path/to/db.properties`
2. Environment variable: `EVENTSTORE_DB_CONFIG=/path/to/db.properties`
3. Current working directory and up to 2 parent directories: `./db.properties`, `../db.properties`, `../../db.properties`

### Configuring the pooled and non-pooled connections

Define separate datasources:

```properties
# db.properties - Advanced configuration

# Pooled connections for regular operations
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

# Non-pooled connections for LISTEN/NOTIFY
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

Be sure the size your pooled datasource connections according to your application needs.
The non-pooled datasource used for monitoring appends with the NOTIFY/LISTEN mechanism only needs 2 connections.

leakDetectionThreshold on the pooled (application) connections should be set quite low, for the monitoring connections this should be at least 30 seconds, as the monitoring connection only refreshes after a longer LISTEN for updates.

## Preparing the Database Schema Manually via DDL

The recommended approach is to create the database schema manually using DDL scripts before deploying your application. This allows you to:

- Use database users with limited DML-only privileges for the application
- Separate schema management from application deployment
- Apply schema changes through controlled migration processes when a newer EventStore version ever requires schema updates

### Using the Initialization Script

The library includes a `quickstart.ddl.sql` script (unprefixed, ready to run) alongside the prefixed `ensure-schema.sql`. Both create the same objects:

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
    event_erasable_data JSONB,

    -- Tags as string array
    event_tags TEXT[] DEFAULT '{}'

) WITH (FILLFACTOR = 100);
```

The `stream_purpose` default matches `EventStreamId.DEFAULT_PURPOSE`, a public constant — so an interop layer doing raw SQL inserts can bind the same value the library does rather than copy the literal out of a script.

Five indexes are created on the events table:

| Index | Purpose |
|---|---|
| `idx_events_position_brin` | compact BRIN index over `event_position` |
| `idx_events_stream_type_position` | B-tree: stream + event type, ordered by `(event_tx, event_position)` |
| `idx_events_stream_position` | B-tree: ordered stream replay |
| `idx_events_tags` | GIN over the tag array, for tag-only lookups |
| `idx_events_stream_tags` | GIN over stream columns **and** tags — the DCB read path. Requires `btree_gin` |

The B-tree indexes are retained alongside the GIN ones because GIN cannot serve `ORDER BY`, and ordered stream replay needs it.

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
    event_erasable_data JSONB,
    event_tags TEXT[] DEFAULT '{}'
) WITH (FILLFACTOR = 100);

-- indexes, the btree_gin guard, functions and triggers follow
```

**`drop-schema.sql`** — teardown for an existing eventstore schema. It drops the functions as well as the tables, because the triggers go with the tables via `CASCADE` but the functions do not:

```sql
DROP TABLE    IF EXISTS PREFIX_bookmarks CASCADE;
DROP TABLE    IF EXISTS PREFIX_events    CASCADE;
DROP FUNCTION IF EXISTS PREFIX_notify_event_appended()  CASCADE;
DROP FUNCTION IF EXISTS PREFIX_notify_bookmark_placed() CASCADE;
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

**Note**: Prefixes must be alphanumeric with underscores, end with an underscore, and be 32 characters or less. The 32-character cap is load-bearing: it keeps the longest generated index name inside PostgreSQL's 63-byte identifier limit, and a truncated index name would silently break both schema validation and idempotency detection.

**Security Best Practice**: Create the schema with a privileged database user (e.g., `eventstore_admin` with DDL rights), then run your application with a limited user (e.g., `eventstore_app` with only DML rights). This prevents applications from accidentally modifying the schema.

## Database Initialization Modes

The `DatabaseInitMode` enum controls how the database schema is handled at startup. Set it via `.databaseInitMode(DatabaseInitMode.xxx)` or use one of the convenience methods on the builder.

### ENSURE (default)

Brings the schema up to what this release expects, then validates it. Safe to run repeatedly, and safe to run from several instances at once:

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

**All the schema scripts run as a single transaction under a per-prefix advisory lock.** `CREATE TABLE / INDEX / EXTENSION IF NOT EXISTS` is not atomic against a concurrent creator, so without that lock several instances starting together against an empty database race on the system catalogs and most of them fail to start. One transaction across all scripts also makes `INITIALIZE`'s drop-then-create indivisible, so a second instance cannot drop what the first has just recreated. The lock is keyed on a hash of the table prefix, so two prefixed stores never block each other.

### VALIDATE

Verifies the schema is present, without creating or modifying anything. A read-only check:

```java
EventStore eventStore = PostgresEventStorage.newBuilder()
    .validateDatabase()
    .buildStore();
```

Validation checks that:

- the required tables exist (`PREFIX_events`, `PREFIX_bookmarks`)
- every expected column is present, with the right type and nullability
- the expected indexes exist by name — including `idx_events_stream_tags` and `idx_events_stream_idempotency`
- the notification functions exist
- each trigger exists **with the expected orientation** (row-level vs statement-level), not merely by name

If validation fails, an `EventStorageException` names what is missing or misconfigured.

> **What validation does not check.** It verifies that named objects *exist*; it does not check an index's method, columns or uniqueness, a column's default, or a function's body. So an index rebuilt as the wrong kind, or the idempotency index recreated without `UNIQUE`, passes validation. Where a DBA applies the DDL, apply the shipped script rather than hand-written equivalents.
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
- The database user lacks permissions to query `information_schema`
- Minimizing startup time is critical
- You have full confidence in the schema being correct

### INITIALIZE

Drops all event store objects — tables **and** functions — recreates them from scratch, then validates:

```java
EventStore eventStore = PostgresEventStorage.newBuilder()
    .initializeDatabase()
    .buildStore();
```

> **This mode is destructive** — all existing event data will be lost. Use only for test environments, fresh deployments, or when a clean slate is explicitly needed.
{: .prompt-danger }

### Schema Changes That No Mode Applies

`ENSURE` only ever *creates* tables, columns and indexes. Anything needing `ALTER TABLE` — changing a column default, or dropping a constraint — is outside what any mode does, and has to be applied by hand.

Two are worth checking against a schema that was not created by the shipped script:

```sql
-- stream_purpose must default to 'default', matching EventStreamId.DEFAULT_PURPOSE.
-- only affects raw SQL inserts: events written through the library always bind it explicitly
ALTER TABLE <prefix>events ALTER COLUMN stream_purpose SET DEFAULT 'default';

-- idempotency uniqueness must be the per-stream partial index, not a table-wide
-- UNIQUE on the column: a table-wide constraint rejects a key reused on a different stream
ALTER TABLE <prefix>events DROP CONSTRAINT IF EXISTS <prefix>events_idempotency_key_key;
CREATE UNIQUE INDEX IF NOT EXISTS <prefix>idx_events_stream_idempotency
    ON <prefix>events (stream_context, stream_purpose, idempotency_key)
    WHERE idempotency_key IS NOT NULL;
```

Neither needs a data migration.

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
    .initializeDatabase()       // Fresh schema every test run
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
    stream.query(EventQuery.matchAll()).toList();
} catch (EventStorageException e) {
    // Handle: "Query result exceeded absolute limit of 10000"
}
```

**Important**: This is an emergency brake, not a substitute for proper query design. Applications should:
- Use batched queries with `Limit` for large result sets
- Design queries to stay well below the hard limit
- Process results incrementally rather than loading everything into memory

Example of proper batched querying:

```java
EventReference lastRef = null;
while (true) {
    List<Event<CustomerEvent>> batch = stream.query(
        EventQuery.matchAll(),
        lastRef,
        Limit.to(500)  // Well below hard limit
    ).toList();

    if (batch.isEmpty()) break;

    // Process batch
    batch.forEach(this::processEvent);

    lastRef = batch.getLast().reference();
}
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

The DCB consistency check is an `INSERT ... WHERE NOT EXISTS (...)`. Under PostgreSQL's default READ COMMITTED isolation, each statement fixes its snapshot when it starts — so two concurrent appends at the same consistency boundary would both find it empty, both insert and both commit. That is a silent DCB violation: the store reports success to both callers and the invariant is gone. The conflicting row is a *phantom* at the moment of the check, so no row lock can cover it.

**Conditional appends therefore take a transaction-scoped advisory lock**, keyed on a hash of the table prefix plus `(stream_context, stream_purpose)`, as its own statement before the INSERT runs.

Things worth knowing about it:

- **Only conditional appends take it.** `AppendCriteria.none()` reads nothing and so cannot observe a stale boundary, which keeps bulk ingestion fully parallel.
- **The key is the stream, not the filter.** Hashing the filter would be finer grained and unsound: two overlapping-but-unequal filters hash differently and would not exclude each other. An append not confined to one fully specified stream falls back to a storage-wide key.
- **Cost** is around 5% against 8 concurrent writers spread over 1000 streams. Conditional appends to *one* stream serialize for the duration of a single INSERT — so a hot stream is the ceiling to watch, remembering that a stream is usually a bounded context rather than one aggregate.
- **Key collisions are harmless.** They only make two unrelated streams take turns; they can never let a real conflict through.
- **No DDL is involved**, so nothing has to be migrated to get this.

`SERIALIZABLE` would also be correct, but it is a poor fit here: a DCB boundary check always scans the tail of the log, which is exactly where every writer writes, so predicate locks collide constantly. Measured on the same workload it produced 86% serialization failures and a third of the throughput.

## What Can Stall Reads: the Visibility Barrier

The read path withholds events whose transaction is still in flight, using `event_tx < pg_snapshot_xmin(pg_current_snapshot())`. That is what keeps a reader from taking a reference past an event that has not committed yet.

`pg_snapshot_xmin` is the oldest transaction id still running — **a property of the whole PostgreSQL cluster, not of this store**. So every event appended since the oldest open transaction took its id is invisible to this store until that transaction ends.

> **Nothing fails and nothing is logged.** Reads just stop advancing: projections go quiet, bookmarks stop moving, `SELECT count(*)` in psql shows the events are there, and when the blocker finally ends everything appears at once.
{: .prompt-warning }

**Only transactions that have written count**, which is what makes this narrow rather than severe. PostgreSQL assigns a transaction id lazily, at the first write, and only assigned ids enter a snapshot's xmin:

- **Harmless at any duration**: `pg_dump`, reporting queries, analytics reads, a replica feed, and an `idle in transaction` connection that only ever read — at any isolation level, including SERIALIZABLE.
- **Not harmless**: a batch job, an ETL run, a migration, or an `idle in transaction` connection that wrote before going idle. `SELECT ... FOR UPDATE` and an explicit `pg_current_xact_id()` also assign an id without writing a row.

The blocker does not have to touch the events table, or even this database — transaction ids are cluster-wide, so a writer in a *different database of the same cluster* stalls this store just as effectively. The operational rule is "do not share a cluster with long-running write transactions", not "do not share a table".

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

The library does not meter this, and nothing in the `sliceworkz.eventstore.*` meters reveals it — they count and time the calls the store makes, all of which keep succeeding throughout a stall. Detection is external, on the database.

**Watch `pg_snapshot_xmin` standing still *while* something holds a transaction id**, never either alone. xmin also stops moving on a completely idle database, so "xmin has not advanced" fires on every quiet store; "a transaction holds an xid" fires on every append in flight. It is the combination that means events are being withheld.

Do **not** measure the effect by counting withheld events. `count(*) ... WHERE event_tx >= pg_snapshot_xmin(...)` has no index to use, so it is a sequential scan of the whole events table every time it is sampled.

### The Store Is a Mild Instance of Its Own Hazard

An append in flight holds a transaction id, so a second append that starts later and commits first cannot read its own event back until the first one finishes. That window is one INSERT long and self-clearing — the same mechanism, with a blocker lasting minutes rather than milliseconds, is the hazard above.

## A Note on `db.properties` and Secrets

A `db.properties` **value** never reaches an error message or a log line — only the key does. `db.<name>.password` goes through the same setter path as every other non-`datasource.` property, so a value interpolated into a message would be a database password in every log, stack trace and error reporter downstream.

A configuration failure therefore names the property and the type the setter expected, and stops there:

```
Error setting property 'maximumPoolSize' (expected int)
```

The detail is left to the cause, which for the realistic cases concerns a property that is never a secret. A stray line with an empty property name (`db.pooled.=x`) is rejected with that explanation rather than failing obscurely.