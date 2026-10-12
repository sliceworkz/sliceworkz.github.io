---
layout: post
toc: true
title: Durable SQL Read Models
description: Writing eventually consistent read models into SQL tables with SqlReadModelProjector and SqlReadModelQuery, with the projector's position committed in the same transaction as its rows.
date: 2026-10-12 07:00:00
categories: [Eventmodeling Documentation,Eventmodeling Guides]
tags: [eventmodeling, read models, sql, projections, leader election, idempotency]
---

An in-memory eventually consistent read model is rebuilt from the start of the stream every time the
process starts, holds its whole state on the heap, and is projected separately by every instance. When
one of those becomes a problem, move the state into a database. [Choosing a Read Model](/posts/eventmodeling-read-models/)
calls this step 4 of the ladder. This post covers the two classes that implement it, both in the api
module's `org.sliceworkz.eventmodeling.readmodels.sql` package:

- **`SqlReadModelProjector<E>`**, the write side. It is an eventually consistent read model that writes
  rows per batch, in one transaction per batch.
- **`SqlReadModelQuery`**, the read side. It reads the same tables to answer queries.

The two share a **table prefix**, so several read models, or several bounded contexts, can live in one
database without their table names colliding.

## The Projector

A projector says which tables it needs, which events it wants, and what each event does to the rows. This
one keeps an account register: one row per account with its balance, plus a log of movements.

```java
public class AccountRegisterProjector extends SqlReadModelProjector<BankingDomainEvent> {

    public static final String TABLE_PREFIX = "rm_register";

    public AccountRegisterProjector ( DataSource dataSource ) {
        super(dataSource, TABLE_PREFIX);
    }

    @Override
    protected String[] createTables ( ) {
        return new String[] {
            """
            CREATE TABLE IF NOT EXISTS %s (
                account_id   VARCHAR(64) NOT NULL PRIMARY KEY,
                customer_id  VARCHAR(64) NOT NULL,
                balance      DECIMAL(19,2) NOT NULL,
                movements    INT NOT NULL,
                %s
            )""".formatted(table("accounts"), EVENT_REF_COLUMNS),
            """
            CREATE TABLE IF NOT EXISTS %s (
                account_id   VARCHAR(64) NOT NULL,
                amount       DECIMAL(19,2) NOT NULL,
                description  VARCHAR(255),
                %s
            )""".formatted(table("movements"), EVENT_REF_COLUMNS)
        };
    }

    @Override
    public EventQuery eventQuery ( ) {
        return EventQuery.matchAll();
    }

    @Override
    protected void project ( Event<BankingDomainEvent> event ) {
        switch ( event.data() ) {
            case AccountOpened opened -> insertIfAbsent(table("accounts"),
                    "account_id = ?", new Object[] { opened.accountId().value() },
                    "account_id, customer_id, balance, movements",
                    opened.accountId().value(), opened.customerId().value(), 0, 0);
            case MoneyDeposited deposited -> {
                updateOnce(table("accounts"),
                        "balance = balance + ?, movements = movements + 1", new Object[] { deposited.amount() },
                        "account_id = ?", deposited.accountId().value());
                insertOnce(table("movements"), "account_id, amount, description",
                        deposited.accountId().value(), deposited.amount(), deposited.description());
            }
            case MoneyWithdrawn withdrawn -> {
                updateOnce(table("accounts"),
                        "balance = balance - ?, movements = movements + 1", new Object[] { withdrawn.amount() },
                        "account_id = ?", withdrawn.accountId().value());
                insertOnce(table("movements"), "account_id, amount, description",
                        withdrawn.accountId().value(), withdrawn.amount().negate(), withdrawn.description());
            }
            case MonthOpened opened -> updateOnce(table("accounts"),
                    "balance = ?", new Object[] { opened.carryForwardBalance() },
                    "account_id = ?", opened.accountId().value());
            case MonthClosed closed -> { }
        }
    }
}
```

Here is what you write and what you get from the base class:

| you write | the base class provides |
|---|---|
| `createTables()`: DDL, idempotent (`CREATE TABLE IF NOT EXISTS`) | `table("accounts")` → `rm_register_accounts` |
| `eventQuery()`: which events to project | `EVENT_REF_COLUMNS`: the four freshness columns to put in a table |
| `project(Event<E>)`: what an event does to the rows | `execute(sql, params...)` on the batch's own connection |
| | `updateOnce`, `insertIfAbsent`, `insertOnce`: idempotent writes |
| | `eventReference()`, `eventId()`, `eventPosition()`, `eventTx()`, `eventIndex()` of the event being projected |

`project(...)` replaces `when(...)`. The base class keeps `when` for itself so that it can record the
reference of the event being projected. Every write in `project` goes through the **batch's connection**:
`beforeBatch` opens a transaction, `afterBatch` commits it, `cancelBatch` rolls it back. You do not manage
connections.

### Registering It

A projector is an eventually consistent read model, so register **an instance** and say so. It belongs to
the slice's *projection* aspect, the part a deployment can run on a different process from the one
serving queries. The query side belongs to the *query* aspect. Both get their `DataSource` through a port,
so the application decides which database it is:

```java
@FeatureSlice
public class AccountRegisterFeatureSlice implements Slice<Banking> {

    private AccountRegisterQuery query;

    @Override
    public void configureProjection ( BoundedContextBuilder<Banking> builder ) {
        DataSource dataSource = builder.port(DataSource.class, "readmodel");
        AccountRegisterProjector projector = new AccountRegisterProjector(dataSource);
        projector.ensureTables();
        builder.readmodel(projector).eventuallyConsistent();
    }

    @Override
    public void configureQuery ( BoundedContextBuilder<Banking> builder ) {
        query = new AccountRegisterQuery(builder.port(DataSource.class, "readmodel"));
        builder.readmodel(SqlCurrentBalanceReadModel.class).live();
    }

    @Override
    public void startQuery ( Banking banking ) {
        // hand `query` to the endpoint that serves it
    }
}
```

The application binds the adapter once on the builder:

```java
Banking bc = BoundedContext.newBuilder(Banking.class)
        .name("banking")
        .eventStorage(storage)
        .instance(InstanceFactory.determine("banking"))
        .adapter(readModelDataSource).forPort(DataSource.class, "readmodel")
        .features()
            .rootPackage(Banking.class.getPackage())
            .done()
        .build();
```

`ensureTables()` runs your `createTables()` statements and creates the projector's bookmark table. Do not
declare that table in `createTables()`. The projector also creates it on first use, because the framework
may ask a projector where to resume before the slice has called `ensureTables()`.

## Progress Lives Next to the State

This is the whole design. A read model writes rows in its own database, while the framework bookmarks
its progress in the event store. Those are two stores with no transaction spanning them. If a process
crashes between committing the rows and placing the bookmark, the next run projects that batch again. For
a durable table, that means duplicated rows and double-counted totals, permanently and with nothing
raised.

`SqlReadModelProjector` therefore implements `SelfBookmarkingProjection`. Every batch writes the
reference of its last event into a `<prefix>_projection_bookmark` table (here
`rm_register_projection_bookmark`), **inside the transaction that writes the batch's rows**. The rows and
the position become durable together and cannot disagree. At start-up the projector resumes from that
table (`resumeFrom()`). The framework's bookmark in the event store is still written after each batch, as
the record operators and the dashboard see, but the projector never reads it back.

The row is keyed by `bookmarkReader()`, which defaults to `readmodelName()`, the class' simple name. Two
read models sharing a prefix therefore do not share a position.

### Rebuild by Dropping the Tables

An absent bookmark row means "replay from the beginning". There is deliberately no fallback to the event
store's bookmark, because that would hand a fresh database a position describing rows it does not have.
Emptying the tables, the bookmark table included, is therefore all a rebuild takes. For a `SHARED` read
model, without restarting anything:

1. Stop its projector on every instance. `ProcessorAdminCapability.stopProcessor(ProcessorKind.READ_MODEL, name)`
   does this locally, and the management stream does it remotely. See
   [Running in Production](/posts/eventmodeling-running-in-production/). A stopped leader hands its lease
   back.
2. Empty the read model: drop and recreate its tables (`ensureTables()` recreates them), or delete their
   rows. **Include its `_projection_bookmark` table.**
3. Start the projector again with `restartProcessor(...)`. The instance that wins the lease asks the
   projector where to resume, finds no bookmark row, and replays the stream.

Nothing else needs to be reset. A projector only creates the bookmark table on its own; your domain tables
are created by `ensureTables()`, which a slice calls while it is configured. Dropping the tables without
recreating them leaves `project()` failing, and retrying, until they exist again. For an `EPHEMERAL` or
`LOCAL` read model, which is not elected, the projector is asked where to resume when its context is
built, so empty its tables while the process is down.

> `tracksItsOwnBookmark()` returns `true`. Override it to return `false` only for a `DataSource` that may
> not be issued DDL. The read model then falls back to the event store's bookmark, which is written
> *after* the commit, so a crash in between re-projects a batch.
{: .prompt-warning }

## Write `project()` So a Repeat Changes Nothing

The own bookmark means the framework never replays a committed batch at a projector. A repeat can still
happen in a failover. A leader paused past its lease, by a long GC pause or a VM freeze, can commit one
batch that the newly elected leader then projects again (see "Leader Election and Fencing" below). Anything
that deliberately re-runs a projection replays too. So `project()` should be idempotent wherever it can
be cheaply.

A plain `UPDATE ... SET column = ?` and a `DELETE` are idempotent already. An increment, an append to a
running total and a plain `INSERT` are not. Three helpers make them idempotent:

| helper | for | how it recognises a repeat |
|---|---|---|
| `updateOnce(table, set, setParams, where, whereParams...)` | an increment, a running total, an aggregate row | the row's freshness columns already reflect this event or a later one |
| `insertIfAbsent(table, keyClause, keyParams, columns, values...)` | creating the row `updateOnce` guards | a row matching the key exists |
| `insertOnce(table, columns, values...)` | an append-style table: a log, a list, a dead-letter view | a row with this event's id exists |

`updateOnce` and `insertOnce` need the table to carry `EVENT_REF_COLUMNS` and write those columns for you.
Name only your domain columns in the clauses. On a row, the freshness columns mean "the newest event this
row reflects". The helpers therefore have no one-row-per-event assumption. The usual aggregate, one
customer row advanced by every order, is exactly the case `updateOnce` is built for, and so is one event
updating many rows.

Two traps the helpers are designed around:

- **`insertIfAbsent` deliberately leaves the freshness columns at zero.** Writing the current event's
  reference there looks natural, but the `updateOnce` that follows would then find the row already marked
  with that event and skip it. The first deposit of every new account would create the row and never be
  counted. Insert a neutral row and let `updateOnce` do all the arithmetic. If the inserted values already
  account for the event, use `insertOnce`, which records the reference.
- **`insertOnce` keys on the event id, not on the domain key.** It works for tables where two different
  events legitimately produce identical-looking rows. It does *not* cover one event inserting several rows
  into one table, because the second row looks like a repeat of the first. Give those rows a unique domain
  key and use `execute(...)` with a constraint instead.

### Why Not Compare Positions Yourself

The freshness check is the total `(tx, position, index)` order, the same order as
`EventReference.happenedAfter` and the DCB check. It is **never** `last_event_position` alone. On
PostgreSQL a position is a `bigserial` and a transaction id an `xid8`, and the two are assigned
independently. An event can therefore hold a *lower* position and a *higher* transaction than an event
that committed before it. A guard on position alone silently **discards** such events, which is worse
than the duplication it was meant to prevent, and it happens rarely enough to reach production. The
helpers spell the comparison out in portable SQL, which is why they exist. Use them rather than writing
`WHERE last_event_position < ?`.

## Where the State Lives: Derived From the DataSource

`storage()` decides who projects and what happens at start-up (see
[the storage table](/posts/eventmodeling-read-models/#where-the-state-lives-and-what-start-waits-for)).
`SqlReadModelProjector` derives it from the `DataSource`'s JDBC URL:

| DataSource | `storage()` | meaning |
|---|---|---|
| in-memory H2 (`jdbc:h2:mem:...`) | `EPHEMERAL` | dies with the process: every instance projects its own copy, from the beginning, and `start()` waits for it |
| anything else, or a URL that cannot be determined | `SHARED` | one database for the deployment, projected by a single elected leader |

`SHARED` is the fallback on purpose, because the two possible mistakes do not cost the same. A shared read
model mistaken for an ephemeral one would have its bookmark dropped on every start and replay its whole
history into a durable database. A follower's copy left unfilled is the cheaper error. If your database is
durable but private to one instance (an embedded file database, say), override `storage()` to return
`ReadModelStorage.LOCAL`. Every instance then projects its own copy and resumes from its own bookmark.

## Leader Election and Fencing for `SHARED`

A `SHARED` read model is projected by **one instance of the deployment**, elected by holding a lease in
the event storage. Every other instance stands by and answers queries from the same tables. The election
is configured on the bounded context builder with `.leadershipPriority(long)` and
`.leadershipIntervals(heartbeat, ttl)` (defaults 5s and 20s). It is never on the processing path: a
leader projects continuously, and election adds no lag. The eventstore's
[Leader Election](/posts/eventstore-leader-election/) post covers the lease semantics.

What election does *not* promise is exactly-once delivery through a failover. A leader paused past its
ttl can wake up and commit a batch while the new leader has already taken over. The **fencing token**
closes half of that window:

- Each acquisition of the lease carries a token that only ever grows. On promotion, the framework hands it
  to the projector (`fencedBy(long)`), and the projector stores it in its bookmark row.
- `resumeFrom()` raises the stored token **before** reading the position it resumes from. From the moment
  the new leader knows where to resume, the old leader cannot move that position any more.
- Every batch's bookmark write compares tokens inside the batch transaction. A superseded leader finds a
  newer token stored, and its whole batch fails with a **`StaleLeadershipException`**. Its rows roll back
  together with the refused bookmark, its projector retires, and its instance hands the lease back.

The other half of the window stays at-least-once: a zombie batch that commits *before* the new leader is
promoted cannot be rejected by any check on the bookmark. That is why the idempotency helpers are not
optional. A token of 0 (an `EPHEMERAL` read model, or a storage without lease support) leaves writes
unfenced.

## The Query Side

`SqlReadModelQuery` shares the projector's table prefix and adds your query methods on top of a few
protected helpers:

```java
public class AccountRegisterQuery extends SqlReadModelQuery {

    public record AccountRow ( AccountId accountId, CustomerId customerId, BigDecimal balance, int movements ) { }

    public AccountRegisterQuery ( DataSource dataSource ) {
        super(dataSource, AccountRegisterProjector.TABLE_PREFIX);           // the same prefix as the projector
    }

    public List<AccountRow> accountsOf ( CustomerId customerId ) {
        return queryList("SELECT * FROM %s WHERE customer_id = ? ORDER BY account_id".formatted(table("accounts")),
                this::row, customerId.value());
    }

    public Optional<ReadModelResult<AccountRow>> account ( AccountId accountId ) {
        return querySingleWithRef("SELECT * FROM %s WHERE account_id = ?".formatted(table("accounts")),
                this::row, accountId.value());
    }

    private AccountRow row ( ResultSet rs ) throws SQLException {
        return new AccountRow(ACCOUNT.id(rs.getString("account_id")), CUSTOMER.id(rs.getString("customer_id")),
                rs.getBigDecimal("balance"), rs.getInt("movements"));
    }
}
```

| helper | returns |
|---|---|
| `queryList(sql, mapper, params...)` | `List<R>` |
| `querySingle(sql, mapper, params...)` | `Optional<R>` |
| `queryListWithRef(sql, mapper, params...)` | `ReadModelResult<List<R>>`, whose `upTo()` is the newest event any returned row reflects |
| `querySingleWithRef(sql, mapper, params...)` | `Optional<ReadModelResult<R>>`, whose `upTo()` is that row's event |

The `...WithRef` variants need the four freshness columns in the select list (`SELECT *` includes them).
Use the reference to check that a caller's own write is in the row it reads:

```java
EventReference write = banking.execute(new DepositCommand(accountId, amount, "salary")).orElseThrow();

boolean included = query.account(accountId)
        .map(row -> !write.happenedAfter(row.upTo()))
        .orElse(false);
```

> The reference from `...WithRef` is a **lower bound** on how far the read model has been projected,
> not the position itself. Later events may have changed other rows, or created rows this query did not
> select. It is good enough to answer "is my write in this row". It is not a base to seed from.
{: .prompt-info }

## Seeding a Current Read From the Rows: `loadBaseAt`

A [seeded read](/posts/eventmodeling-read-models/#step-5-the-seeded-read-current-without-replaying)
starts from a maintained base and projects only the events that have not reached it yet. For a SQL base,
the state and its position are two reads: the rows, and the bookmark row. Whatever the projector commits
between those two reads lands in one and not the other. Read the bookmark first, and you re-apply events
the base already contains. Read it afterwards, and you skip events that are in neither. Both are silent.

`loadBaseAt(reader, loader)` takes them as one observation. It reads the bookmark, runs your loader, reads
the bookmark again, and retries while the two differ. This is sound because the projector commits its rows
and its bookmark in one transaction, so an unchanged bookmark proves no batch landed in between. It needs
no particular isolation level and takes no lock.

```java
/** The projector's bookmarkReader(): its readmodelName(), the class' simple name. */
private static final String READER = AccountRegisterProjector.class.getSimpleName();

/** One account's balance and the position it reflects, as one observation: a base to seed from. */
public ReadModelResult<BigDecimal> balanceBase ( AccountId accountId ) {
    BigDecimal[] balance = { BigDecimal.ZERO };
    Optional<EventReference> at = loadBaseAt(READER, conn -> {
        balance[0] = BigDecimal.ZERO;                                   // replace on every attempt, never add
        try ( var stmt = conn.prepareStatement("SELECT balance FROM %s WHERE account_id = ?".formatted(table("accounts"))) ) {
            stmt.setString(1, accountId.value());
            try ( var rs = stmt.executeQuery() ) {
                if ( rs.next() ) {
                    balance[0] = rs.getBigDecimal("balance");
                }
            }
        }
    });
    return new ReadModelResult<>(balance[0], at.orElse(null));
}
```

Two rules apply to the loader. It may run more than once, so it must *replace* what it loaded rather than
add to it. And an account without a row is a zero balance *at the returned position*, not an absent base.
`loadBaseAt` returns empty only when the read model has projected nothing at all, and then replaying the
whole stream is the correct answer. A missing bookmark *table*, or an unreachable database, throws rather
than being reported as "nothing projected".

The seeded read itself is an ordinary live model:

```java
public class SqlCurrentBalanceReadModel implements SeededReadModel<BankingDomainEvent> {

    private final AccountRegisterQuery register;
    private final AccountId accountId;
    private BigDecimal balance = BigDecimal.ZERO;

    public SqlCurrentBalanceReadModel ( AccountRegisterQuery register, AccountId accountId ) {
        this.register = register;
        this.accountId = accountId;
    }

    @Override
    public Optional<EventReference> seed ( ) {
        ReadModelResult<BigDecimal> base = register.balanceBase(accountId);
        balance = base.data();
        return Optional.ofNullable(base.upTo());
    }

    @Override
    public EventQuery eventQuery ( ) {
        return EventQuery.forEvents(EventTypesFilter.any(), ACCOUNT.tags(accountId));
    }

    @Override
    public void when ( Event<BankingDomainEvent> event ) {
        // the projector's SQL does the same arithmetic; a test pins that the two agree
        balance = switch ( event.data() ) {
            case AccountOpened ignored -> BigDecimal.ZERO;
            case MoneyDeposited d -> balance.add(d.amount());
            case MoneyWithdrawn w -> balance.subtract(w.amount());
            case MonthOpened o -> o.carryForwardBalance();
            case MonthClosed ignored -> balance;
        };
    }

    public BigDecimal balance ( ) {
        return balance;
    }
}
```

```java
BigDecimal current = banking.read(SqlCurrentBalanceReadModel.class, query, accountId).balance();
```

The base is computed by SQL arithmetic and the delta by Java, so the "write the fold once" rule cannot be
met literally here. Where the throughput of the projector rules out reading a row, folding it in Java and
writing it back, keep the two separate and **pin their equivalence with a test**. Otherwise the answer
starts depending on how far the projector got.

## Testing With `SqlReadModelTest`

`SqlReadModelTest` (in `sliceworkz-eventmodeling-testing`) drives a projector through the same batch
lifecycle the framework uses, without a bounded context. Put the scenarios in an abstract inner class, and
add one `@Nested` class per database:

```java
class AccountRegisterTest extends SqlReadModelTest<BankingDomainEvent> {

    abstract static class Tests extends AbstractSqlReadModelTests<BankingDomainEvent> {

        final AccountId account = ACCOUNT.newId();
        final CustomerId customer = CUSTOMER.newId();

        @Override protected SqlReadModelProjector<BankingDomainEvent> createProjector ( DataSource ds ) { return new AccountRegisterProjector(ds); }
        @Override protected SqlReadModelQuery createQuery ( DataSource ds ) { return new AccountRegisterQuery(ds); }
        @Override protected String[] tablesToDrop ( ) {
            return new String[] { "rm_register_accounts", "rm_register_movements", "rm_register_projection_bookmark" };
        }

        @Test
        void depositsAddUp ( ) {
            Optional<EventReference> last = projectEvents(
                    new AccountOpened(account, customer, LocalDate.now()),
                    new MoneyDeposited(account, new BigDecimal("25.00"), "salary"),
                    new MoneyDeposited(account, new BigDecimal("5.00"), "gift"));

            AccountRegisterQuery query = query();
            assertEquals(0, new BigDecimal("30").compareTo(query.account(account).orElseThrow().data().balance()));
            assertEquals(last, projectedUpTo());                                // the position committed with the rows
            assertEquals(last.orElseThrow(), query.balanceBase(account).upTo());  // and the base it seeds from
        }
    }

    @Nested class OnH2 extends Tests {
        @Override protected DataSource createDataSource ( ) { return h2DataSource(); }
    }

    @Nested class OnPostgres extends Tests {
        @Override protected DataSource createDataSource ( ) { return postgresDataSource(); }
    }
}
```

- `projectEvents(...)` projects the events as one batch, with ascending positions and transaction ids,
  and returns the reference of the last one.
- `projectedUpTo()` is where the projector would resume, read from committed state.
- `restartedProjector()` builds a second projector over the same database, the way a restarted instance
  would. Use it to replay events the read model has already seen and assert that nothing changes.
- `h2DataSource()` is a fresh in-memory H2 in PostgreSQL mode. `postgresDataSource()` (or
  `postgresDataSource(image)`) is a shared PostgreSQL Testcontainer, kept for the life of the JVM. List
  the bookmark table in `tablesToDrop()` too, or the next test resumes from the previous test's position.
- The testing module brings H2, the PostgreSQL driver and Testcontainers with it. Running the PostgreSQL
  variant needs a Docker daemon.

Run the read model's SQL against PostgreSQL as well as H2. H2 in PostgreSQL mode is close, but it is not
the database you deploy on.

## Writing a Projector for Another Store

Nothing else ships, but the contract is small. Implement `SelfBookmarkingProjection` (`resumeFrom()`, and
optionally `fencedBy(long)`) together with the eventstore's `BatchAwareProjection` against your store.
What decides whether it can work is atomicity: if your store cannot commit the state and the position
together, you get at-least-once delivery, and everything above about idempotency applies twice over.
`SqlReadModelProjector` is the worked example to copy. The eventstore's
[Projecting Events](/posts/eventstore-projecting-events/#batch-aware-projections) post explains the batch
lifecycle it builds on.
