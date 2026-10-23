---
layout: post
toc: true
title: Choosing a Read Model
description: The ladder of read models in Sliceworkz EventModeling, from a live model projected per read to background projections, seeded reads and snapshots, and when to climb each step.
date: 2026-10-11 06:00:00
categories: [Eventmodeling Documentation,Eventmodeling Guides]
tags: [eventmodeling, read models, projections, live model, eventual consistency, snapshots]
---

Every read model answers a question out of events. Read models differ in three things: *when* they are
projected, *where* their state lives, and *what* they start from. The registration API is built around
exactly those three choices.

This guide goes through them as a ladder. Each step buys one thing and costs one thing. **Do not take a
step until you have measured the problem it fixes.** Most read models should stop at step 1.

All samples use the banking domain from the [quickstart](/posts/eventmodeling-quickstart/): `AccountId`
and `CustomerId` typed ids, the `ACCOUNT` and `CUSTOMER` entities, and a sealed `BankingDomainEvent`
hierarchy with `AccountOpened`, `MoneyDeposited`, `MoneyWithdrawn`, `MonthClosed` and `MonthOpened`.

## Step 0: If a Command Decides on It, It Is Not a Read Model

When a command needs state to decide on, give it a `DecisionModel` and pass it to
`context.decisionModels(...)`. The events that model was projected from then become the command's
consistency boundary. If one of them is superseded before the command appends, the append conflicts.

`CommandContext.read(...)` is also available inside a command, but it sits **outside** that boundary and
is meant for auxiliary lookups. A command that decides on what `read(...)` returned can append against a
history the store does not agree with any more, and nothing will report it. No amount of read-model tuning
fixes that, so settle this question first. [Commands and Decision Models](/posts/eventmodeling-commands-and-decision-models/)
covers decision models in full.

The rule of thumb: **read models are for screens and integrations, decision models are for commands.**

## Step 1: Start With a Live Model

A live model is a class that says which events it wants and what each event does to its state. It is
projected **when you read it**: the framework builds an instance with the parameters of the read, replays
the matching events through it, and hands it back. There is no background thread, no staleness, nothing
to rebuild at start-up and nothing to migrate.

```java
public class AccountDetailsReadModel implements ReadModel<BankingDomainEvent> {

    public record AccountDetails ( AccountId accountId, CustomerId customerId, LocalDate openedOn ) { }

    private final AccountId accountId;
    private AccountDetails details;

    public AccountDetailsReadModel ( AccountId accountId ) {       // the read's parameters
        this.accountId = accountId;
    }

    @Override
    public EventQuery eventQuery ( ) {
        return EventQuery.forEvents(EventTypesFilter.of(AccountOpened.class), ACCOUNT.tags(accountId));
    }

    @Override
    public void when ( Event<BankingDomainEvent> event ) {
        switch ( event.data() ) {
            case AccountOpened opened -> details = new AccountDetails(opened.accountId(), opened.customerId(), opened.date());
            default -> { }
        }
    }

    public Optional<AccountDetails> details ( ) {
        return Optional.ofNullable(details);
    }
}
```

Its feature slice registers it, and says how it is projected:

```java
@FeatureSlice
public class AccountDetailsFeatureSlice implements Slice<Banking> {

    @Override
    public void configureQuery ( BoundedContextBuilder<Banking> builder ) {
        builder.readmodel(AccountDetailsReadModel.class).live();
    }
}
```

Application code reads it through its narrow capability type:

```java
BankingApi banking = bc;
AccountDetails details = banking.read(AccountDetailsReadModel.class, accountId).details().orElseThrow();
```

Keep these points in mind, because everything later builds on them:

- **The registration states the mode.** `build()` rejects a `readmodel(...)` registration that never
  called `.live()` or `.eventuallyConsistent()`, and names every read model that did not. The verb is
  technically redundant: a class can only be live, and calling `.eventuallyConsistent()` on a class
  registration throws `IllegalArgumentException`. It is required anyway, because how a read model is
  projected is a decision every caller of it lives with, so it should be visible where it is made.
- **One `when`, taking the `Event`.** Switch on `event.data()` for the domain event. The tags, the
  timestamp and the `reference()` are on the same argument when a projection needs them.
- **The read is typed by the class you ask for.** `read(AccountDetailsReadModel.class, id)` *is* an
  `AccountDetailsReadModel`. You can chain on it or assign it to `var` without a cast. Assigning it to an
  unrelated type is a compile error, not a `ClassCastException` at runtime.
- **Constructor parameters are the read's parameters, matched by type.** The framework looks at the
  constructors it can reach, keeps the ones every argument fits, and picks the most specific of those,
  using Java's own overload rules. A read model can therefore offer several constructors:

```java
public RecentTransactionsReadModel ( AccountId accountId ) {               // read(X.class, accountId)
    this(accountId, 10);
}

public RecentTransactionsReadModel ( AccountId accountId, int count ) {    // read(X.class, accountId, 25)
    this.accountId = accountId;
    this.count = count;
}
```

When two constructors fit equally well, the read fails and names the read model, the argument types
passed and the constructors it compared. It does not pick one by declaration order. A live read model
that can never be constructed is rejected at `build()`: an interface, an abstract class, or a class with
no constructor the framework can reach.

**The one rule of live models: the replay must be bounded by design.** A live model scoped by tag to one
entity with a short life is ideal. A live model that replays an ever-growing set of events is correct in
development, gets slower in production forever, and nothing reports it. Here is a simple test: if you
cannot state the bound in one sentence ("one account's opening event"), it is not bounded. Go to step 2
before you consider anything heavier.

*Costs:* one query pass per read, proportional to the number of matching events.

## Step 2: Before Escalating, Bound the Replay

When a live model gets slow, the reflex is to cache it. That is usually the expensive answer to a
problem the *domain* can solve more cheaply. There are two techniques, and both keep you at step 1's cost
model.

**Narrow `eventQuery()` with tags.** Tags are the query dimension of the event store. A model that filters
on an entity's tag replays that entity's events, not the whole stream. Design the tags so the query stays
small. The banking example's
[`MonthStatementReadModel`](https://github.com/sliceworkz/eventmodeling/blob/develop/sliceworkz-eventmodeling-examples/src/main/java/org/sliceworkz/eventmodeling/examples/banking/features/monthstatement/MonthStatementReadModel.java)
filters on account *and* month, so one statement replays one period.

**Introduce a savepoint event, and return it from `initQuery()`.** A savepoint is an ordinary domain event
that summarises everything before it. In "closing the books", `MonthClosed` summarises a period and
`MonthOpened` carries the closing balance forward. A live model finds the newest savepoint with a
backwards `limit(1)` query and replays only what came after it:

```java
public class CurrentPeriodReadModel implements ReadModel<BankingDomainEvent> {

    private final AccountId accountId;
    private YearMonth month;
    private BigDecimal balance = BigDecimal.ZERO;
    private int transactions;

    public CurrentPeriodReadModel ( AccountId accountId ) {
        this.accountId = accountId;
    }

    @Override
    public EventQuery initQuery ( ) {                      // the newest savepoint, and nothing before it
        return EventQuery.forEvents(EventTypesFilter.of(AccountOpened.class, MonthOpened.class), ACCOUNT.tags(accountId))
                .backwards().limit(1);
    }

    @Override
    public EventQuery eventQuery ( ) {                     // only what happened after it
        return EventQuery.forEvents(EventTypesFilter.of(MoneyDeposited.class, MoneyWithdrawn.class, MonthClosed.class),
                ACCOUNT.tags(accountId));
    }

    @Override
    public void when ( Event<BankingDomainEvent> event ) {
        switch ( event.data() ) {
            case AccountOpened opened -> month = YearMonth.from(opened.date());
            case MonthOpened opened   -> { month = opened.month(); balance = opened.carryForwardBalance(); }
            case MoneyDeposited d     -> { balance = balance.add(d.amount()); transactions++; }
            case MoneyWithdrawn w     -> { balance = balance.subtract(w.amount()); transactions++; }
            case MonthClosed closed   -> { }
        }
    }
}
```

The projector runs `initQuery()` first and feeds its result through `when(...)`. It then starts
`eventQuery()` after that event. When there is no savepoint yet, the init query finds nothing and the
model replays from the beginning.

- **Give the two queries disjoint event types.** The savepoint types belong to `initQuery()` only.
  Otherwise savepoints are processed twice, and you cannot recover from a buggy savepoint by
  ignoring it.
- **The savepoint is written by the domain.** A `CloseMonthCommand` decides the books are closed. That is
  a business fact with its own meaning, and the faster read model follows from it.
- **`initQuery()` applies to live models only.** A bookmarked background projection must see every event,
  so eventually consistent read models ignore it, and `build()` logs a warning. In a `DecisionModel` it is
  part of the consistency boundary: the command locks on both queries.

The eventstore's [savepoint pattern](/posts/eventstore-savepoint-pattern/) post explains the mechanism in
more depth. The banking example's
[`BankingClosingTheBooksExample`](https://github.com/sliceworkz/eventmodeling/blob/develop/sliceworkz-eventmodeling-examples/src/main/java/org/sliceworkz/eventmodeling/examples/banking/BankingClosingTheBooksExample.java)
runs it end to end.

**When to leave step 2:** when the replay genuinely cannot be bounded. That means the read spans *many*
entities (a list, an overview, a total), or no savepoint makes sense in the domain.

## Step 3: Eventually Consistent, in Memory

An eventually consistent read model is projected **once, in the background**, on a processor of its own.
Reads cost nothing. The price is that the answer lags whatever the projector has not reached yet. You
register an **instance**, not a class: one long-lived object, fed continuously.

Do not hand-roll the obvious version, a class that mutates a `Map` field from `when()` and exposes a
getter. The projector thread mutates the map while readers read it, so a reader can see a half-applied
batch, or even get a `ConcurrentModificationException`. The answer also carries no position, so nothing
can tell how stale it is. Extend `PublishingReadModel` instead. You write three things: a state type with
an initial value, a pure fold, and a query.

```java
public class AccountBalancesReadModel extends PublishingReadModel<BankingDomainEvent, Map<AccountId,BigDecimal>> {

    @Override
    protected Map<AccountId,BigDecimal> initialState ( ) {
        return Map.of();
    }

    @Override
    protected Map<AccountId,BigDecimal> apply ( Map<AccountId,BigDecimal> state, Event<BankingDomainEvent> event ) {
        AccountId account = BalanceFold.accountOf(event.data());
        Map<AccountId,BigDecimal> next = new HashMap<>(state);
        next.put(account, BalanceFold.apply(state.getOrDefault(account, BigDecimal.ZERO), event.data()));
        return Map.copyOf(next);                           // never mutated after this
    }

    @Override
    public EventQuery eventQuery ( ) {
        return EventQuery.matchAll();
    }

    public BigDecimal balanceOf ( AccountId accountId ) {
        return state().getOrDefault(accountId, BigDecimal.ZERO);
    }
}
```

The base class folds each batch to the side. Once per batch, it publishes **the immutable state together
with the position it reflects**, in one volatile write. A cancelled batch publishes nothing. The contract
on your side: **never mutate a state after `apply` returns it.** The cost of that rule is one copy per
batch (here, one per event, which is fine for a handful of accounts; for a large map, use a persistent
collection).

The slice keeps the instance it registers, and hands it to whatever serves the read:

```java
@FeatureSlice
public class CurrentBalanceFeatureSlice implements Slice<Banking> {

    private final AccountBalancesReadModel balances = new AccountBalancesReadModel();
    private CurrentBalanceEndpoint endpoint;

    @Override
    public void configureQuery ( BoundedContextBuilder<Banking> builder ) {
        builder.readmodel(balances).eventuallyConsistent();              // the base, projected in the background
        builder.readmodel(CurrentBalanceReadModel.class).live();         // a seeded read on top of it (step 5)
    }

    @Override
    public void startQuery ( Banking banking ) {
        endpoint = new CurrentBalanceEndpoint(banking, balances);
    }
}
```

`read(...)` builds live models. An eventually consistent read model is read through the instance itself,
which is why the slice passes it on.

### Reading the State With Its Position

`published()` returns a `ReadModelResult<STATE>`. It is a record holding `data()`, the state, and
`upTo()`, the reference of the last event folded into it. You get both from **one observation**.
`state()` and `upTo()` are shortcuts for the two halves. If you need both, call `published()` once instead
of calling the shortcuts one after the other.

That position answers a question a stale answer cannot: *does this already include my own write?* A
command execution returns the reference of the event it appended:

```java
EventReference write = banking.execute(new DepositCommand(accountId, amount, "salary")).orElseThrow();

ReadModelResult<Map<AccountId,BigDecimal>> published = balances.published();
boolean includesMyWrite = published.includes(write);
```

`includes(write)` is `upTo != null && !write.storedEventHappenedAfter(upTo)`: the write is in the answer
once the stored event it was appended as is at or before `upTo`. Use it rather than comparing references
yourself, and never compare with `position()` alone: a reference is ordered by `(tx, position, index)`, and
a position and a transaction id are assigned independently, so the two orders can disagree. Note that
`write.happenedBefore(upTo)` is strict, and so is `false` exactly when your write is the newest event
folded — which is why `includes` asks "not after".

> `upTo()` is `null` until the first batch is published. Treat that as "includes nothing yet".
{: .prompt-info }

### Where the State Lives, and What `start()` Waits For

`ReadModel.storage()` returns a `ReadModelStorage`. It decides who projects the read model and what
happens to its bookmark when the process starts:

| | state | projected by | at start |
|---|---|---|---|
| `EPHEMERAL` (default) | in memory, gone with the process | every instance, its own copy | bookmark dropped, full replay |
| `LOCAL` | durable, private to one instance | every instance, its own copy | resumes from its own bookmark |
| `SHARED` | one store the whole deployment uses | **a single elected leader** | resumes from the shared bookmark |

`PublishingReadModel` is always `EPHEMERAL`. That has a consequence: it is rebuilt from the start of the
stream at every process start, and `bc.start()` **waits** until every `EPHEMERAL` read model has finished
that initial projection. Those read models are therefore usable as soon as `start()` returns. The rebuild
is start-up latency, not background noise. The wait has a safety timeout of 5 minutes, which you can
change with `-Dsliceworkz.eventmodeling.readmodel.ephemeral.projection.timeout.ms=...`. When the timeout
expires, start-up continues with a warning and the read model keeps catching up in the background.
`LOCAL` and `SHARED` read models resume from their bookmark and never hold up `start()`.

Other points to know:

- **The read model's name is its bookmark key.** It defaults to the class' simple name. Override
  `readmodelName()` to run one class as several differently named instances. Names must be unique within
  a context and stable across restarts, and `build()` checks both. Two read models sharing a name would
  share a bookmark and silently skip each other's events.
- **Staleness is bounded and observable.** The answer is at most a poll interval behind, and every answer
  can carry its `upTo`.

*Costs:* the answer lags by up to a poll interval, and the whole stream is replayed at every process start.

## What a Projector Does When Projection Fails

A background projection can fail. The read model's own code may throw, or a database it writes to may be
down. The processor handles the two kinds of failure differently.

- **Most failures are retried with backoff.** The projector rolls its cursor back to the start of the
  failed batch, and no bookmark is placed for that batch. The next attempt offers exactly the same events
  again, so a retry never skips an event. The delay starts at 10 seconds and doubles up to 5 minutes. You
  can tune it with `-Dsliceworkz.eventmodeling.projector.retry.initial.ms` and
  `-Dsliceworkz.eventmodeling.projector.retry.max.ms`. A transient outage therefore heals itself, with
  nothing to restart.
- **A failure that retrying cannot help retires the processor.** That covers a poison event (one that
  cannot be deserialized), a closed storage, and a superseded leader. The processor stops and stays
  stopped until someone acts.

Each state is reported as a bounded-context event, so you can tell "nothing to do" from "quietly going
stale":

| event | meaning |
|---|---|
| `ReadModelProjectorStarted` | the projector runs; emitted for every eventually consistent read model at start |
| `EventuallyConsistentReadModelUpdated` | a catch-up that handled something |
| `ReadModelProjectorFailed` | a retry round got nowhere; carries `consecutiveFailedRuns` so you choose the alerting threshold |
| `ReadModelProjectorStopped` | retired on a permanent failure (or stopped by an operator); carries the cause and the event it failed on |

Combine `Started` and `Stopped`, letting the later one win, to answer "is it still projecting?". There is
no "recovered" event. Recovery shows up as the next `EventuallyConsistentReadModelUpdated`. Neither event
is emitted at a normal shutdown, so `Stopped` always signals the one state worth alerting on.

An operator puts a retired projector back without restarting the context, through
`ProcessorAdminCapability` (part of `OperationsCapabilities`):

```java
OperationsCapabilities operations = bc;
operations.processors();                                                       // status per processor
operations.restartProcessor(ProcessorKind.READ_MODEL, "AccountBalancesReadModel");
```

Restarting without fixing a permanent cause replays the same batch and stops again.
[Running in Production](/posts/eventmodeling-running-in-production/) covers the operator side, including
stopping a projector to rebuild a read model. [Observability and Tracing](/posts/eventmodeling-observability/)
covers the events.

## Step 4: Eventually Consistent, Durable

Move the state into a database when one of these is true:

- the start-up rebuild is too slow,
- the state outgrows the heap,
- the deployment should project once instead of once per instance (`SHARED`, with an elected leader).

The mechanism is a pair of classes, `SqlReadModelProjector` and `SqlReadModelQuery`. The projector writes
rows and keeps its own position in the same transaction. [Durable SQL Read Models](/posts/eventmodeling-sql-read-models/)
covers them in detail.

*Costs:* tables and migrations, a `project()` that must be idempotent, and leader election for `SHARED`.

## Step 5: The Seeded Read, Current Without Replaying

Steps 3 and 4 buy cheap reads and pay for them in staleness. Step 1 is current but pays on every read. A
seeded read combines them, for the reads where the lag is unacceptable: money, availability, anything a
user acts on immediately. It loads a **base** from an eventually consistent model and projects **only the
events that have not reached it yet**, which is usually none.

A seeded read is a live model with one extra method. It is a *per-read* decision: the dashboard keeps
reading the eventually consistent model, and the one screen that has to be current adds a seeded read
beside it.

```java
public class CurrentBalanceReadModel implements SeededReadModel<BankingDomainEvent> {

    private final AccountBalancesReadModel balances;
    private final AccountId accountId;

    private BigDecimal balance = BigDecimal.ZERO;
    private EventReference upTo;

    public CurrentBalanceReadModel ( AccountBalancesReadModel balances, AccountId accountId ) {
        this.balances = balances;
        this.accountId = accountId;
    }

    @Override
    public Optional<EventReference> seed ( ) {
        ReadModelResult<Map<AccountId,BigDecimal>> base = balances.published();   // rule 1: one observation
        balance = base.data().getOrDefault(accountId, BigDecimal.ZERO);           // rule 2: unknown = zero AT that position
        upTo = base.upTo();
        return Optional.ofNullable(upTo);                                         // empty only when nothing was projected
    }

    @Override
    public EventQuery eventQuery ( ) {                       // this account only, so the delta stays tiny
        return EventQuery.forEvents(EventTypesFilter.any(), ACCOUNT.tags(accountId));
    }

    @Override
    public void when ( Event<BankingDomainEvent> event ) {
        balance = BalanceFold.apply(balance, event.data());  // rule 3: the same fold as the base
        upTo = event.reference();
    }

    public BigDecimal balance ( ) { return balance; }
    public EventReference upTo ( ) { return upTo; }
}
```

Register it as a live model. The read passes the base instance as a parameter:

```java
builder.readmodel(CurrentBalanceReadModel.class).live();

BigDecimal current = banking.read(CurrentBalanceReadModel.class, balances, accountId).balance();
```

The framework calls `seed()` once, right after construction, and starts the projection after the position
it returns. The delta is paged and ordered exactly like a full projection. Three rules make a seeded read
correct, and breaking any of them fails *silently*:

1. **Take the state and the position from one observation.** If you read them separately, whatever the
   projector committed in between ends up in only one of them. Read the position first, and you re-apply
   events the base already holds. Read it afterwards, and you skip events that are in neither the base nor
   the delta. Use `PublishingReadModel.published()`, or `SqlReadModelQuery.loadBaseAt(...)` for a SQL base.
2. **Empty means "replay everything".** An entity the base has never seen is *default state at the base's
   position*, and you return that position, not `Optional.empty()`. Returning empty still produces a
   correct answer, so no test fails. It just replays the whole history on every read, and you only notice
   once the stream is long. The `LiveModelProjected` event carries `seededAt`, so a seed that quietly
   returns empty shows up in monitoring.
3. **Write the fold once.** The base and the delta are projected by different code paths. If they
   disagree, the answer depends on how far the projector happened to get, which is exactly what a seeded
   read exists to prevent. Extract the rule as a pure function of `(state, event)` and call it from both
   sides:

```java
/** What one event does to one account's balance: the whole rule, written once. */
final class BalanceFold {

    static BigDecimal apply ( BigDecimal balance, BankingDomainEvent event ) {
        return switch ( event ) {
            case AccountOpened ignored    -> BigDecimal.ZERO;
            case MoneyDeposited deposited -> balance.add(deposited.amount());
            case MoneyWithdrawn withdrawn -> balance.subtract(withdrawn.amount());
            case MonthOpened opened       -> opened.carryForwardBalance();
            case MonthClosed ignored      -> balance;
        };
    }
}
```

`build()` rejects two registrations that would otherwise fail silently. A `SeededReadModel` registered as
an eventually consistent instance would resume from its bookmark, so `seed()` would never be called. A
`SeededReadModel` registered with `.snapshots(...)` would have two competing answers to "where does this
projection start".

The banking example's [`currentbalance`](https://github.com/sliceworkz/eventmodeling/tree/develop/sliceworkz-eventmodeling-examples/src/main/java/org/sliceworkz/eventmodeling/examples/banking/features/currentbalance)
slice is the worked version, with
[`BalanceFold`](https://github.com/sliceworkz/eventmodeling/blob/develop/sliceworkz-eventmodeling-examples/src/main/java/org/sliceworkz/eventmodeling/examples/banking/features/currentbalance/BalanceFold.java)
shared by both sides.

*Costs:* a base load plus the delta, per read.

## Step 6: Snapshots, Only With No Base to Seed From

One gap remains. It is a **per-entity** live model with a long history, where no savepoint makes sense
in the domain and no maintained projection exists to seed from. For that case, and only that case, the
framework can snapshot the live model itself. The read model implements `SnapshotCapable<S>`:

```java
public class AccountHistoryReadModel implements ReadModel<BankingDomainEvent>, SnapshotCapable<AccountHistoryReadModel.History> {

    public record History ( BigDecimal deposited, BigDecimal withdrawn, int movements ) { }

    private History history = new History(BigDecimal.ZERO, BigDecimal.ZERO, 0);

    // constructor, eventQuery() and when(...) as for any live model

    @Override public History takeSnapshot ( )                 { return history; }
    @Override public void fromSnapshot ( History snapshot )   { history = snapshot; }
    @Override public String version ( )                       { return "1"; }   // bump when History changes shape
}
```

Register it with a `SnapshotStorage`. Calling `snapshots(...)` declares the read model live, so `.live()`
is not called as well:

```java
builder.readmodel(AccountHistoryReadModel.class)
       .snapshots(snapshotStorage)
       .eventCountThreshold(1000)        // save a snapshot after a read that streamed this many events
       .readAndWrite();                  // or readOnly() / writeOnly()
```

A read loads the snapshot, projects only the events after its reference, and saves a fresh snapshot when
the read streamed more events than the threshold. The snapshot key is derived from the read model's name
and the read's parameters. A snapshot is loaded only when its `version()` matches exactly. Others are
ignored and get overwritten.

You implement the `SnapshotStorage<S>` yourself: `load(key, version)` returns an optional `SnapshotRecord`
holding the snapshot and its event reference, and `save(key, version, snapshot, reference)` stores one.
Optionally, `classifyMiss(key, version)` reports why a load missed (`ABSENT` or `VERSION_MISMATCH`), so
monitoring can tell a cold cache from a version bump.

Snapshots are the last resort for a reason. They have more moving parts than a seeded read for the same
effect, and the base is only as fresh as whoever last read past the threshold. If a projection exists
that could seed the read, seed it (step 5). If a savepoint could bound the replay, bound it (step 2).

## Shapes to Avoid

**A read model that mutates its own fields and exposes a getter.** A reader can see a half-applied batch,
iterating a collection that is being mutated can throw, and the answer carries no position. Extend
`PublishingReadModel`. For a model so large that a copy per batch is too much, guard a mutable model with a
lock and copy out only the answer and its position.

**A live model whose replay is unbounded.** It is correct in development, gets linearly slower forever,
and nothing reports it. See step 2.

**Snapshots as a performance patch where a projection exists.** Use a seeded read instead.

**Catching a read model up by hand in application code.** The fold is the easy part. What goes wrong is
taking the state and the position as two observations. Use `SeededReadModel`.

**Comparing positions yourself.** Freshness follows the total `(tx, position, index)` order
(`EventReference.happenedAfter`). A comparison on position alone silently *discards* events. Use the SQL
helpers and `loadBaseAt` rather than writing the comparison yourself.

**Deciding on a read model in a command.** Use a `DecisionModel` (step 0).

## Quick Reference

| | registration | projected | state lives in | starts from | answer is |
|---|---|---|---|---|---|
| Live model | `readmodel(X.class).live()` | per read | nothing, built per read | the stream | current |
| Live, bounded | same | per read | nothing | a tag scope or the newest savepoint | current |
| EC in memory (`PublishingReadModel`) | `readmodel(instance).eventuallyConsistent()` | background | heap, per instance | the stream, at every start | ≤ a poll interval behind |
| EC durable (`SqlReadModelProjector`) | `readmodel(instance).eventuallyConsistent()` | background | a database | its own bookmark | ≤ a poll interval behind |
| Seeded (`SeededReadModel`) | `readmodel(X.class).live()` | per read, delta only | its base's | the base's position | current |
| Live + snapshot (`SnapshotCapable`) | `readmodel(X.class).snapshots(store)...` | per read, delta only | the snapshot store | a stored snapshot | current |

When to take the next step:

| you observe | take |
|---|---|
| a command needs state to decide on | a decision model, always |
| a live read is measurably slow | bound the replay **first** |
| the read spans many entities, or no bound exists | eventually consistent, in memory |
| the start-up rebuild is too slow, state outgrows the heap, or it should project once per deployment | durable SQL |
| the eventually consistent answer is not current enough *for one read* | a seeded read for that read |
| per entity, long history, no savepoint, no base | snapshots |

## "Current" Means Current as Far as the Store Will Show You

One caveat applies to every row of the table above. An event whose transaction is still in flight is
withheld from every reader alike. On PostgreSQL this is the `pg_snapshot_xmin` visibility barrier: a
long-running writing transaction anywhere in the cluster holds back what *every* reader can see. A live
model or a seeded read is therefore exactly as fresh as the store allows, and no fresher. A stalled
barrier makes background projections go quiet without failing. See
[What Can Stall Reads: the Visibility Barrier](/posts/eventstore-configuring-postgresql-storage/#what-can-stall-reads-the-visibility-barrier).

What a seeded or published read *can* offer is proof. Expose the reference it reached, and a caller that
just executed a command checks that its own write is included, with `!write.happenedAfter(upTo)`, instead
of hoping.

## Testing Read Models

- **`LiveModelTest`** (in `sliceworkz-eventmodeling-testing`) seeds events, reads a live model with the
  parameters you give it, and asserts on the result. It covers steps 1, 2, 5 and 6.
- **`SqlReadModelTest`** drives a SQL projector through its batch lifecycle against H2 and PostgreSQL. See
  [Durable SQL Read Models](/posts/eventmodeling-sql-read-models/).
- Annotate a scenario `@ForEachBackend` to run it against every registered event storage. A plain `@Test`
  runs once, in memory.

[Testing Feature Slices](/posts/eventmodeling-testing/) covers all the test bases.
