---
layout: post
toc: true
title: Commands and Decision Models
description: How a command decides on history, raises domain events and gets its append checked against everything it read
date: 2026-10-11 03:00:00
categories: [Eventmodeling Documentation,Eventmodeling Guides]
tags: [eventmodeling, commands, decision models, dcb, optimistic locking, idempotency, aggregates]
---

A command is the write side of a *state change* slice: it reads the history it needs, decides, and raises
domain events. This guide covers how to write one, what the framework does around it — above all the
consistency boundary that makes its decision safe under concurrency — what it can throw and what to do
about each, and the second checked write path the framework offers, aggregates.

The samples use the banking domain of the [quickstart](/posts/eventmodeling-quickstart/), extended with
deposits, withdrawals and a monthly closing. The repository's own
[banking example](https://github.com/sliceworkz/eventmodeling/tree/develop/sliceworkz-eventmodeling-examples/src/main/java/org/sliceworkz/eventmodeling/examples/banking)
is the fuller version to copy from.

## The Shape of a Command

Every command has the same three steps: **select decision models, check, raise.**

```java
public record DepositCommand ( AccountId accountId, BigDecimal amount, String description, Overrides overrides )
        implements Command<BankingDomainEvent>, Overriding {

    public DepositCommand {
        if ( accountId == null || amount == null || amount.signum() <= 0 ) {
            throw new IllegalArgumentException("a deposit needs an account and a positive amount");
        }
        overrides = ( overrides == null ) ? Overrides.none() : overrides;
    }

    public DepositCommand ( AccountId accountId, BigDecimal amount, String description ) {
        this(accountId, amount, description, Overrides.none());
    }

    @Override
    public void execute ( CommandContext<BankingDomainEvent, BankingDomainEvent> context ) {
        var account = new AccountBalanceDecisionModel(accountId);
        var result = context.decisionModels(account);           // 1. select: projects AND pins the boundary

        BusinessException.when(!account.accountExists(), "Account does not exist");   // 2. check
        BusinessException.when(account.isClosed(), "Account is closed");

        BigDecimal balanceAfter = account.balance().add(amount);

        context.check(ORIGIN_OF_FUNDS_EXPLAINED, amount.compareTo(ORIGIN_OF_FUNDS_AMOUNT) > 0,
            "A deposit of " + amount + " is above " + ORIGIN_OF_FUNDS_AMOUNT + ": explain where the money comes from");

        context.check(BALANCE_WITHIN_GUARANTEE, balanceAfter.compareTo(GUARANTEED_BALANCE) > 0,
            "The balance would become " + balanceAfter + ", above the guaranteed " + GUARANTEED_BALANCE);

        result.raiseEvent(new MoneyDeposited(accountId, amount, description, context.ruleViolations()),   // 3. raise
            ACCOUNT.tags(accountId));
    }
}
```

- **The command is a record of its inputs.** The constructor rejects a malformed request before any
  history is read — no null account, no negative amount — with an `IllegalArgumentException`. Everything
  the command needs comes in through the constructor, including "now" when a rule depends on the day: a
  command that takes the date rather than reading a clock is a command a test can pin.
- **`context.decisionModels(...)` projects the models and hands back the `CommandResult`** the events are
  raised on. Raised events are appended together, in one append, after `execute` returns.
- **A rejection is a `BusinessException`.** `BusinessException.when(condition, message)` throws when the
  condition holds; `BusinessException.because(message)` throws unconditionally.
- **The `context.check(...)` lines are business rules with an enforcement level** — rules people may
  break under conditions, judged by the kernel after the command ran. They have their own guide,
  [Business Rules and Enforcement Levels](/posts/eventmodeling-business-rules/). A command without such
  rules simply leaves them out.

Register the command from the slice's `configureCommand`. That is declarative — commands are instantiated
by their caller — but it makes the slice a state change and lists the command from startup:

```java
@FeatureSlice(chapter = "Transactions")
public class DepositFeatureSlice implements Slice<Banking> {

    @Override
    public void configureCommand ( BoundedContextBuilder<Banking> builder ) {
        builder.command(DepositCommand.class);
        builder.businessRules(ORIGIN_OF_FUNDS_EXPLAINED, BALANCE_WITHIN_GUARANTEE);
    }
}
```

### Executing a command

Application code holds the narrow capability type, `BankingApi`, not the whole context (see
[Who May Do What](/posts/eventmodeling-who-may-do-what/)), and executes through it:

```java
Optional<EventReference> deposited = bank.execute(new DepositCommand(account, new BigDecimal("100"), "salary"));
```

`execute` answers the reference of the **last event the command appended**. It is empty when the command
raised nothing, and when an idempotent repeat was swallowed (see below) — in both cases nothing was
appended by this call.

When the caller needs to know something the command decided — a freshly minted id, typically — the
command implements `CommandWithResult<D, R>` instead, and `execute` returns a
`CommandExecutionResult<R>` carrying both the response and the event reference:

```java
public record OpenAccountCommand ( CustomerId customerId, LocalDate date ) implements CommandWithResult<BankingDomainEvent, AccountId> {

    @Override
    public AccountId execute ( CommandContext<BankingDomainEvent, BankingDomainEvent> context ) {
        var result = context.noDecisionModels();          // a fresh id: no history to decide on

        AccountId accountId = ACCOUNT.newId();
        result.raiseEvent(new AccountOpened(accountId, customerId, date),
                Tags.of(ACCOUNT.tag(accountId), CUSTOMER.tag(customerId)));

        return accountId;
    }
}

CommandExecutionResult<AccountId> opened = bank.execute(new OpenAccountCommand(customer, today));
AccountId account = opened.response();
```

This is how a caller learns what a command decided. Reading the event back out of the stream by the
returned reference is not: the context offers no stream access to application code, on purpose.

## The Command Context

`CommandContext<D, D>` is everything a command gets:

| Method | What it is for |
|---|---|
| `decisionModels(DecisionModel...)` | project the models the decision is made on, pin the consistency boundary, get the `CommandResult` |
| `noDecisionModels()` | the declaration that this command's append needs no guard; also gets the `CommandResult` |
| `actor()` | who the command is executed for — the actor of the `Tracing` it was executed with, empty without one |
| `read(ReadModelClass.class, params...)` | an auxiliary lookup of a live read model — **outside** the boundary |
| `check(rule, violated, message)` | check a business rule with an enforcement level |
| `ruleViolations()` | the violations the execution goes ahead with, to record in the payload |
| `justifies(...)` / `enforces(...)` | settle a post-justified override or a deferred enforcement |

And `CommandResult` is what events are raised on: `raiseEvent(event, tags)`, or
`raiseEvent(event, tags, idempotencyKey)` for an event keyed by hand, plus the idempotency key settings
described below. Tags come from the entities: `ACCOUNT.tags(accountId)`, or `Tags.of(ACCOUNT.tag(a),
CUSTOMER.tag(c))` for an event about several (see [Entities, Typed Ids and Tags](/posts/eventmodeling-entities-and-tags/)).

> **Decide on decision models, never on `context.read(...)`.** A read model read through the context is
> not part of the consistency boundary: one of the events it was projected from can be superseded between
> the read and the append, and the append still succeeds. Use it for lookups the raised events do not
> depend on. Anything a decision depends on belongs in a `DecisionModel`.
{: .prompt-warning }

A command that calls **neither** `decisionModels(...)` nor `noDecisionModels()` has no `CommandResult`
to raise events on, so it decided on nothing and raised nothing: its execution appends nothing and
succeeds. That is the ordinary shape of a command that reads, finds nothing to do and returns early.

## Decision Models

A `DecisionModel<D>` is a projection a command instantiates inline: `eventQuery()` scoped by tags to the
entity, `when(Event<D>)` folding the state, and accessors for the checks.

```java
public class AccountBalanceDecisionModel implements DecisionModel<BankingDomainEvent> {

    private final AccountId accountId;

    private boolean exists;
    private boolean closed;
    private BigDecimal balance = BigDecimal.ZERO;

    public AccountBalanceDecisionModel ( AccountId accountId ) {
        this.accountId = accountId;
    }

    @Override
    public EventQuery initQuery ( ) {
        // the savepoint: the newest of the opening and the month closings, nothing older
        return EventQuery.forEvents(
                EventTypesFilter.of(AccountOpened.class, MonthClosed.class),
                ACCOUNT.tags(accountId)
            ).backwards().limit(1);
    }

    @Override
    public EventQuery eventQuery ( ) {
        // the movements after it; the savepoint types stay out
        return EventQuery.forEvents(
                EventTypesFilter.of(MoneyDeposited.class, MoneyWithdrawn.class, AccountClosed.class),
                ACCOUNT.tags(accountId));
    }

    @Override
    public void when ( Event<BankingDomainEvent> event ) {
        switch ( event.data() ) {
            case AccountOpened opened -> exists = true;
            case MonthClosed month -> { exists = true; balance = month.closingBalance(); }
            case MoneyDeposited deposit -> balance = balance.add(deposit.amount());
            case MoneyWithdrawn withdrawal -> balance = balance.subtract(withdrawal.amount());
            case AccountClosed accountClosed -> closed = true;
            default -> { }
        }
    }

    public boolean accountExists ( ) { return exists; }
    public boolean isClosed ( ) { return closed; }
    public BigDecimal balance ( ) { return balance; }
}
```

The `initQuery()` is optional and implements the [savepoint pattern](/posts/eventstore-savepoint-pattern/):
it finds the newest `MonthClosed` (or the opening, for an account that has not closed a month yet), and
`eventQuery()` then replays only the movements after it. Without it the model replays the account's whole
history on every command. The two queries name **disjoint** event types, so a savepoint is never folded
twice. The repository's
[`ActivePeriodDecisionModel`](https://github.com/sliceworkz/eventmodeling/blob/develop/sliceworkz-eventmodeling-examples/src/main/java/org/sliceworkz/eventmodeling/examples/banking/features/currentperiod/ActivePeriodDecisionModel.java)
is the full version of this model.

A decision model is cheap to make and lives for one execution: construct it inside `execute`, from the
command's inputs. A command can use several — `context.decisionModels(account, exceptionsToday)` — and
they are read against one shared boundary.

## The Consistency Boundary

What makes a decision model more than a lookup is what `context.decisionModels(...)` does around it.
This is the [Dynamic Consistency Boundary](https://dcb.events/) of the
[EventStore](/posts/eventstore-appending-events/), applied for you:

1. **The head of the domain stream is taken first**, before any model is read.
2. **Every model read is bounded at that head**, so all of them see the same moment, however many reads
   the command makes.
3. **The raised events are appended with that head as the expected reference**, and with a lock filter
   that is the **union of every query a model was read with** — each `eventQuery()`, and the `initQuery()`
   of a model using a savepoint.
4. **If an event matching that filter was appended after the head, the append fails** with an
   `OptimisticLockingException`, and nothing is stored.

So the check at append time re-asks exactly the question the command decided on. A deposit landing on
the account between the read and the append conflicts; a deposit on another account does not.

Three details are worth knowing:

- **The savepoint query is locked on too.** "The newest closing is March" is a fact the command decided
  on as much as the movements after it. A `MonthClosed` landing after the read conflicts, whether or not
  the model found a savepoint. You do not widen `eventQuery()` with the savepoint types to get this — keep
  the two queries disjoint and the framework locks on both.
- **An empty boundary is still a boundary.** On an empty stream the head is absent, and the check reads
  "I decided on nothing matching" — any matching event appended meanwhile conflicts. This is what makes
  uniqueness checks work (see [Where a Validation Goes](/posts/eventmodeling-where-validations-go/)).
- **An `until` on a model's own query bounds its read only.** The framework strips it from the lock
  filter: a criteria filter carrying one would deem nothing after it relevant, and admit every append.

Pinning the head first also keeps the check cheap: on PostgreSQL the check only walks the events appended
*during* the command, not everything the stream received since the entity last changed (see
[Stream Design and Performance](/posts/eventstore-stream-design-and-performance/)).

## Three Ways a Command Fails

| Exception | Thrown | Means | Retry? |
|---|---|---|---|
| `IllegalArgumentException` | by the command's constructor, before any read | the request is malformed | never |
| `BusinessException` | by the command, after projecting | history says no | never — the answer *is* no |
| `OptimisticLockingException` | by the append | new relevant facts appeared since the decision | yes — re-execute, which re-projects and re-decides |

All three reach the caller unchanged. A business rule rejected by the kernel is a
`RuleViolationException`, which *is* a `BusinessException`, so everything said here holds for it too.

> **Reject with `BusinessException`, never with `IllegalStateException`.** The kernel tells a rejection
> from a bug by its type. An `IllegalStateException` thrown for a business rule is reported as a failure,
> with a stack trace, next to the real bugs a dashboard alerts on.
{: .prompt-tip }

Each outcome is also reported to the context's listeners as a `BoundedContextEvent`, so a monitoring
stream sees every execution without the caller having to log anything:

| Event | Outcome |
|---|---|
| `CommandExecuted` | appended (or nothing to append); carries the references of the raised events |
| `CommandRejected` | a `BusinessException`: carries the reason and, for a rejection on business rules, the rule judgements — no stack trace |
| `CommandFailedOnOptimisticLocking` | a DCB conflict; carries the expected last event |
| `CommandFailed` | anything else: type, message and stack trace |

Alert on `CommandFailed`; chart `CommandRejected` by command and reason. One subtlety: a rejection is
recognised by the exception **the command itself** threw. A `BusinessException` thrown from inside a
decision model's `when` arrives wrapped by the projector and is reported as `CommandFailed` — a rule judged
while history is read is a rule in the wrong place. Decision models fold; commands judge.

## Retrying a Conflict: `executeWithRetry`

An `OptimisticLockingException` is the routine DCB outcome under contention, and the answer is always the
same: execute again, which builds a fresh context, re-projects the models and re-decides on the new facts.
`executeWithRetry` is that loop, written once:

```java
Optional<EventReference> deposited =
    bank.executeWithRetry(new DepositCommand(account, new BigDecimal("100"), "salary"), alice, RetryPolicy.of(5));
```

It exists for every `execute` shape — with or without a tracing and an idempotency key, for `Command` and
`CommandWithResult` — taking a trailing `RetryPolicy`. `executeWithRetry(command)` uses
`RetryPolicy.DEFAULT`, three attempts.

- **Only `OptimisticLockingException` is retried.** A `BusinessException`, an `IllegalArgumentException`
  or a storage failure propagates from the attempt that raised it. A retry that ends in a
  `BusinessException` is the helper working: the re-decide on the new facts said no.
- **When the attempts run out, the last conflict is rethrown unchanged**, the earlier ones attached as
  suppressed exceptions, so existing `catch ( OptimisticLockingException e )` code keeps working.
- **There is no delay between attempts.** A conflict is cleared by re-reading, not by waiting. A boundary
  that keeps conflicting is a hot boundary — widen it or spread it over entities rather than backing off.
- **The same command instance is re-executed.** Keep commands immutable and build decision models inside
  `execute`, never in the constructor — a model filled in on the first attempt would carry stale state
  into the second.

Each attempt is an execution of its own and is reported as one: a retried command shows one
`CommandFailedOnOptimisticLocking` per conflicted attempt before its final outcome.

## Idempotency Keys

A caller that may send the same request twice — a client retrying a timed-out HTTP call, an automation
handed the same todo item again — gives the execution an idempotency key:

```java
Optional<EventReference> first  = bank.execute(new DepositCommand(account, new BigDecimal("50"), "gift"), "request-4711");
Optional<EventReference> repeat = bank.execute(new DepositCommand(account, new BigDecimal("50"), "gift"), "request-4711");
// first is present, repeat is empty: nothing was appended the second time
```

The store holds the key per event, scoped to the stream, and swallows a repeat silently. Derive the key
from **the request** — a client-generated request id, the id of the todo item being handled — and never
from the attempt: a key minted per attempt is a fresh key every time and de-duplicates nothing. The same
key is carried into every attempt of `executeWithRetry`, which is safe because a conflicted append stored
nothing.

A command raising **several** events turns one command-level key into one key per event: the events get
`<key>/1`, `<key>/2`, … in raise order. A repeat then finds every key stored and is swallowed whole. A
re-execution under the same key that raises a *different* set of events is refused with an
`IdempotencyKeyConflictException` and stores nothing — it cannot be a retry. The derivation is by
position, so a command whose number of events varies keys each event itself with
`raiseEvent(event, tags, key)`, from what that event is about.

The command decides how it treats a caller's key, through the `CommandResult`:

| Method | Effect |
|---|---|
| `idempotencyKey(key)` | use this key, whatever the caller passed |
| `fallbackIdempotencyKey(key)` | use the caller's key if there is one, this one otherwise |
| `exclusiveIdempotencyKey(key)` | use this key; a caller passing one is an error |
| `requireIdempotencyKey()` | the caller must pass a key |
| `forbidIdempotencyKey()` | the caller must not pass a key |

## Tracing: Who, Through What, in Which Flow

Every `execute` takes an optional `Tracing`: the actor, the channel and the correlation id of the flow the
command is part of. The kernel writes them as tags (`x-actor`, `x-channel`, `x-correlation-id`, …) on
every event the command appends, and the command can read the actor through `context.actor()` — which is
how a command decides who may override a rule.

```java
Tracing alice = Tracing.actorAndChannel("alice", "counter");
bank.execute(new WithdrawCommand(account, new BigDecimal("250"), "kitchen", today, Overrides.of(NO_OVERDRAFT.id())), alice);
```

Without a tracing, the command runs under the tracing the edge bound to the thread with `TracingScope`,
if any, and otherwise starts a fresh flow with no actor. An HTTP layer typically builds the tracing from
the authenticated user once per request. [Observability and Tracing](/posts/eventmodeling-observability/)
covers correlation across commands, automations and translators.

## Aggregates: the Second Checked Write Path

Decision models are the DCB way: each command picks the facts it needs, across whatever entities they
concern. The framework also supports the classic way, an **aggregate**: an object loaded for one
identity, holding its state, whose methods raise events. Both paths are checked; they draw the boundary
differently.

```java
public class AccountAggregate implements Aggregate<BankingDomainEvent> {

    private AggregateContext<BankingDomainEvent> context;
    private boolean opened;
    private BigDecimal balance = BigDecimal.ZERO;

    @Override
    public void setContext ( AggregateContext<BankingDomainEvent> context ) {
        this.context = context;
    }

    @Override
    public void when ( Event<BankingDomainEvent> event ) {
        switch ( event.data() ) {
            case AccountOpened opened -> this.opened = true;
            case MoneyDeposited deposit -> balance = balance.add(deposit.amount());
            case MoneyWithdrawn withdrawal -> balance = balance.subtract(withdrawal.amount());
            default -> { }
        }
    }

    public void withdraw ( BigDecimal amount, String description ) {
        BusinessException.when(!opened, "Account does not exist");
        BusinessException.when(balance.compareTo(amount) < 0, "Insufficient balance");
        // appended at once, tagged with the identity, guarded by the last event this instance saw
        context.raiseEvent(new MoneyWithdrawn(ACCOUNT.idIn(context.identity()).orElseThrow(), amount, description));
    }

    public BigDecimal balance ( ) {
        return balance;
    }
}
```

Register it from a slice with `builder.aggregate(AccountAggregate.class)`, and load it by its identity
through the `aggregate` capability, which `BankingApi` carries:

```java
AccountAggregate account = bank.aggregate(AccountAggregate.class, ACCOUNT.tags(accountId));
account.withdraw(new BigDecimal("10"), "coffee");
```

- **The identity is a set of tags.** Loading replays the events carrying those tags — by default of any
  type; override `eventQuery(Tags identity)` to narrow it — and every event the aggregate raises is tagged
  with the identity.
- **`raiseEvent` appends immediately**, checked against the last event the instance has seen. An instance
  that has fallen behind — loaded before another instance appended — gets an `OptimisticLockingException`.
  `context.updateFromStream()` catches it up; loading it again does the same.
- **Several events at once** go through `raiseEvents(List)`, or `eventAppender()` to stage events with
  `add(...)` and append them atomically with `append()`. `raiseEvent(event, key)` and
  `add(event, key)` take an idempotency key.
- **Snapshots** make loading a long-lived aggregate cheap: the aggregate implements
  `SnapshotCapable<S>` (`takeSnapshot`, `fromSnapshot`, `version`), you provide a `SnapshotStorage<S>`,
  and register it with
  `builder.aggregate(X.class).snapshots(storage).eventCountThreshold(100).readAndWrite()`. A snapshot
  whose version does not match is ignored and the aggregate is rebuilt from its events.

Which to reach for:

| | Decision models (DCB) | Aggregate |
|---|---|---|
| Boundary | exactly the facts this decision needs, across entities | everything tagged with one identity |
| Lifetime | built per execution | loaded per identity, may be held |
| Raising | events collected, appended once after `execute` | each `raiseEvent` appends at once |
| Contention | only on relevant facts | on any change to the identity |
| Business rules with enforcement levels, `evaluate(...)` previews | yes | no |

Decision models are the framework's primary path: a boundary drawn per decision conflicts only on facts
that matter to it, and it can span two entities — a transfer between two accounts — which an aggregate
cannot. An aggregate suits a team that already thinks in aggregates, or an entity whose rules genuinely
cover all of its history. The eventstore's [Aggregates and DCB](/posts/aggregates-and-dcb/) discusses the
trade-off at length.

## Testing a Command

The published `CommandTest` base runs a command against seeded history:

```java
public class WithdrawCommandTest extends CommandTest<BankingDomainEvent, BankingInboundEvent, BankingOutboundEvent> {

    @Test
    void withdrawsWhenTheBalanceAllows ( ) {
        given()
            .event(new AccountOpened(ACCOUNT_1, CUSTOMER.id("cust-1"), TODAY), ACCOUNT.tags(ACCOUNT_1))
            .event(new MoneyDeposited(ACCOUNT_1, new BigDecimal("100"), "salary"), ACCOUNT.tags(ACCOUNT_1))
            .when(new WithdrawCommand(ACCOUNT_1, new BigDecimal("40"), "groceries", TODAY, Overrides.none()))
            .then()
            .event(new MoneyWithdrawn(ACCOUNT_1, new BigDecimal("40"), "groceries"), ACCOUNT.tags(ACCOUNT_1));
    }

    @Test
    void rejectsAWithdrawalFromAnUnknownAccount ( ) {
        given()
            .when(new WithdrawCommand(ACCOUNT_1, new BigDecimal("10"), "nothing to take it from", TODAY, Overrides.none()))
            .then()
            .businessError("Account does not exist");
    }

    // domainEventType(), inboundEventType() and outboundEventType() return the three event classes
}
```

Seed events with the tags production gives them — a decision model selects by tag, so an untagged seed is
invisible to it. `businessError(message)` asserts the rejection is a `BusinessException`, so a rule thrown
as the wrong type fails the test. `AggregateTest` does the same for aggregates:
`given(ACCOUNT.tags(id)).events(...).when(account -> account.withdraw(...)).then()...`.
[Testing Feature Slices](/posts/eventmodeling-testing/) covers both, and running them against every
storage backend.

## Summary

| | Use |
|---|---|
| A state change | `Command<D>`: decision models, `BusinessException.when`, `raiseEvent` |
| A caller needing what was decided | `CommandWithResult<D, R>` → `CommandExecutionResult<R>` |
| History to decide on | `DecisionModel` built inside `execute`, savepoint `initQuery()` to bound the replay |
| No history needed | `noDecisionModels()` |
| A lookup the decision does not depend on | `context.read(...)` — outside the boundary |
| A conflict | `executeWithRetry(command[, tracing], policy)` |
| The same request twice | an idempotency key derived from the request |
| Who and in which flow | a `Tracing`; `context.actor()` in the command |
| An object per identity | `Aggregate`, loaded with `aggregate(X.class, tags)` |
