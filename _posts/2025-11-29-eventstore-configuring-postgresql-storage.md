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

## Using Your Own DataSource

If your application already manages database connections, pass your existing `DataSource` to the builder:

```java
DataSource existingDataSource = // ... from your application context

EventStore eventStore = PostgresEventStorage.newBuilder()
    .dataSource(existingDataSource)
    .build();
```

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
    .build();
```

### Why Separate Datasources?

PostgreSQL's LISTEN/NOTIFY requires dedicated, long-lived connections that cannot be pooled. Using a separate monitoring DataSource:

- **Optimization**: Configure a short leakDetection time for regular connections, and a longer one for monitoring event appends via LISTEN/NOTIFY
- **Prevents blocking**: Long-running LISTEN connections don't consume regular connection pool resources
- **PgBouncer compatibility**: Connection poolers like PgBouncer don't support LISTEN/NOTIFY in transaction pooling mode. A separate direct connection bypasses this limitation
- **Performance**: Isolates notification traffic from query traffic

If you don't provide a separate monitoring DataSource, the regular DataSource is used for both purposes.

When connecting to a database through pbBouncer for the monitoring, you will find the realtime notification mechanism not to react to appends immediately, but only after 30 seconds or so.  While this functionally works, your expectations towards eventual consistency keeping up are without a doubt higher than that.

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

The library includes an `quickstart.ddl.sql` script (available in the JAR or source repository):

```sql
CREATE TABLE events (
    -- Primary key and positioning
    event_position BIGSERIAL PRIMARY KEY,

    -- XID8 transaction id
    event_tx xid8 DEFAULT pg_current_xact_id()::xid8 NOT NULL,

    -- Event identification
    event_id UUID NOT NULL UNIQUE,

    -- Stream identification
    stream_context TEXT NOT NULL,
    stream_purpose TEXT NOT NULL DEFAULT '',

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

-- Additional indexes, functions, and triggers...
```

Extract the script from the JAR or copy from the source repository, then execute it using your preferred database client or migration tool.

### Co-locating Multiple EventStore Instances

In some scenarios, you want to create multiple eventstores, without configuring multiple databases:
- separating environments like dev(development), tst(testing) and acc(acceptance)
- separating tenants like customer1, customer2, etc...
- separating application components or bounded contexts (sales, orders, invoicing, ...)

For these scenario's, EventStore supports the usage of prefixes, in which all required database objects are uniquely identified by prefixing them.

You can find a DDL script named `initialisation.sql`, which has exactly the same content as `quickstart.ddl.sql`, but with a "PREFIX_" before each object that you can replace by any tenant name you like, ending with "_":

```sql
CREATE TABLE PREFIX_events (
    -- Primary key and positioning
    event_position BIGSERIAL PRIMARY KEY,

    -- XID8 transaction id
    event_tx xid8 DEFAULT pg_current_xact_id()::xid8 NOT NULL,

    -- Event identification
    event_id UUID NOT NULL UNIQUE,

    -- Stream identification
    stream_context TEXT NOT NULL,
    stream_purpose TEXT NOT NULL DEFAULT '',

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

-- Additional indexes, functions, and triggers...
```

By replacing all occurences of "PREFIX_" in this file by e.g. "myapp_", you create a private eventstorage to be used by a specific application.


### Connecting to an EventStore database with prefixes

Configure each instance with its prefix:

```java
EventStore tenant1Store = PostgresEventStorage.newBuilder()
    .prefix("tenant1_")
    .build();

EventStore tenant2Store = PostgresEventStorage.newBuilder()
    .prefix("tenant2_")
    .build();
```

**Note**: Prefixes must be alphanumeric with underscores, end with an underscore, and be 32 characters or less.

**Security Best Practice**: Create the schema with a privileged database user (e.g., `eventstore_admin` with DDL rights), then run your application with a limited user (e.g., `eventstore_app` with only DML rights). This prevents applications from accidentally modifying the schema.

## Auto-creating the Database Schema on Startup

For development and testing environments, you can enable automatic schema creation:

```java
EventStore eventStore = PostgresEventStorage.newBuilder()
    .initializeDatabase()  // Enable auto-creation
    .build();
```

This executes the initialization DDL automatically when the EventStore is built. The operation is idempotent—safe to run multiple times without affecting existing data.

**Warning**: Auto-creation requires the database user to have DDL privileges (CREATE TABLE, CREATE FUNCTION, CREATE TRIGGER, CREATE INDEX). This is generally not recommended for production environments where you should:
- Limit application database users to DML operations only (SELECT, INSERT, UPDATE)
- Manage schema changes through controlled migration processes
- Separate schema management from application deployment

Disable auto-creation for production (it's disabled by default):

```java
EventStore eventStore = PostgresEventStorage.newBuilder()
    .initializeDatabase(false)  // Explicitly disable (default)
    .build();
```

## Checking the Database Schema on Startup

By default, the EventStore validates the database schema on startup to ensure all required objects exist and are properly configured:

```java
EventStore eventStore = PostgresEventStorage.newBuilder()
    .checkDatabase(true)  // Default behavior
    .build();
```

The validation checks:
- Required tables exist (`PREFIX_events`, `PREFIX_bookmarks`)
- All columns are present with correct types
- Indexes are properly configured
- PostgreSQL functions and triggers exist

If validation fails, an `EventStorageException` is thrown with details about missing or misconfigured objects. This provides early detection of schema issues before runtime failures occur.

**Disabling Schema Checks:**

```java
EventStore eventStore = PostgresEventStorage.newBuilder()
    .checkDatabase(false)  // Disable validation
    .build();
```

Disable checks only when:
- The database user lacks permissions to query `information_schema`
- You want to defer validation to a later time
- Minimizing startup time is critical

**Recommended Production Configuration:**

```java
EventStore eventStore = PostgresEventStorage.newBuilder()
    .initializeDatabase(false)  // No auto-creation in production
    .checkDatabase(true)         // But validate schema exists
    .build();
```

This ensures the schema is present without requiring DDL privileges.

## Configuring a Hard Limit on Query Result Size

To prevent memory exhaustion from poorly designed queries, configure an absolute limit on result set size:

```java
EventStore eventStore = PostgresEventStorage.newBuilder()
    .resultLimit(10000)  // Maximum 10,000 events per query
    .build();
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
    .build();

// Application Server 2
EventStore storeOnServer2 = PostgresEventStorage.newBuilder()
    .name("api-server-2")
    .build();

// Batch Worker
EventStore batchStore = PostgresEventStorage.newBuilder()
    .name("batch-worker")
    .build();
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