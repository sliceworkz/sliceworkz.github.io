---
layout: post
toc: true
title: Automations and Policies
description: Reacting to what happened — a to-do list worked off by an automation, or a policy that turns one event into one command — and what each costs when the work fails.
date: 2026-10-11 08:00:00
categories: [Eventmodeling Documentation,Eventmodeling Guides]
tags: [eventmodeling, automation, policy, todo list, idempotency, retries, dead letter]
---

Event Modeling has one pattern for "the system does something by itself": **events → to-do list → processor → command → events**. The framework offers it in two forms:

| | an **automation** | a **policy** |
|---|---|---|
| reacts to | the items of a to-do list, a read model of outstanding work | one domain event |
| shape | `TodoListReadModel` + `Automation<T,D,O>` | `Policy<D>` |
| does | anything a handler can do: execute commands, append events, call a port | maps the event onto one command, nothing else |
| failure | your `onFailure` decides, per item | the event stalls (or a rejection is skipped) |
| reach for it when | there is a deadline, an outside call, several facts to gather, an item to put aside | "whenever X happens, do Y" |

Both run in a slice's **automation aspect**: registered from `configureAutomation`, on the single elected leader of the deployment (see [Leader Election](/posts/eventstore-leader-election/) for the lease underneath), and bookmarked so a restart picks up where the last run stopped.

This guide extends the banking domain of the [quickstart](/posts/eventmodeling-quickstart/) with outgoing transfers and payment cards:

```java
public interface BankingDomain {
    record AccountId  ( String value ) implements EntityId { }
    record TransferId ( String value ) implements EntityId { }
    record CardId     ( String value ) implements EntityId { }

    Entity<AccountId>  ACCOUNT  = Entity.of("account", AccountId::new);
    Entity<TransferId> TRANSFER = Entity.of("transfer", TransferId::new);
    Entity<CardId>     CARD     = Entity.of("card", CardId::new);

    sealed interface BankingDomainEvent {
        // ...
        record AccountFrozen ( AccountId accountId, String reason ) implements BankingDomainEvent { }
        record AccountClosed ( AccountId accountId ) implements BankingDomainEvent { }
        record CardIssued ( AccountId accountId, CardId cardId ) implements BankingDomainEvent { }
        record CardBlocked ( AccountId accountId, CardId cardId, String reason ) implements BankingDomainEvent { }
        record TransferRequested ( TransferId transferId, AccountId accountId, String toIban, long amountInCents ) implements BankingDomainEvent { }
        record TransferExecuted ( TransferId transferId, String gatewayReference ) implements BankingDomainEvent { }
        record TransferAttemptFailed ( TransferId transferId, String reason, int attempt, Instant nextAttemptDueAt ) implements BankingDomainEvent { }
        record TransferAbandoned ( TransferId transferId, String reason, int attempts ) implements BankingDomainEvent { }
    }
}
```

> The automation half of this guide mirrors the payments example in the repository: [`ExecutePaymentAutomation`](https://github.com/sliceworkz/eventmodeling/blob/develop/sliceworkz-eventmodeling-examples/src/main/java/org/sliceworkz/eventmodeling/examples/payments/features/executepayment/ExecutePaymentAutomation.java) and [`PaymentsToExecuteTodoList`](https://github.com/sliceworkz/eventmodeling/blob/develop/sliceworkz-eventmodeling-examples/src/main/java/org/sliceworkz/eventmodeling/examples/payments/features/executepayment/PaymentsToExecuteTodoList.java), and the policy half [`BlockCardsOfAccountPolicy`](https://github.com/sliceworkz/eventmodeling/blob/develop/sliceworkz-eventmodeling-examples/src/main/java/org/sliceworkz/eventmodeling/examples/banking/features/blockcardsofaccount/BlockCardsOfAccountPolicy.java). Read them next to this page.
{: .prompt-info }

## Automations

### The to-do list is the queue

There is no queue table, no in-memory work list, no "processing" flag. The outstanding work **is** a read model projected from the event history: an item is on the list because an event put it there, and it leaves because another event took it off. Everything else follows from that one rule:

- **Delivery is at least once.** An item that was not handled is still on the list on the next round, and after every restart. An item is only "done" once the events its handling raised have been projected and drop it — so a crash between the append and the bookmark hands the same item over again.
- **Nothing kept outside events survives.** A retry counter in a field, a "gave up on this one" set in memory: the next restart rebuilds the list from events and the item comes straight back.
- **The to-do list owns the order.** Items are handled in the order `streamItems` returns them. The framework never re-orders and never checks.

A to-do list implements `TodoListReadModel<D, T>`: a read model with `streamItems(Limit)` and `lastEventReference()`. Because the automation reads it on its thread while the list's own projector updates it on another, build it on [`PublishingReadModel`](/posts/eventmodeling-read-models/) — the automation then always reads one consistent, immutable state:

```java
public class TransfersToExecuteTodoList
        extends PublishingReadModel<BankingDomainEvent, Map<TransferId, TransfersToExecuteTodoList.TransferToExecute>>
        implements TodoListReadModel<BankingDomainEvent, TransfersToExecuteTodoList.TransferToExecute> {

    /**
     * @param attempts how many attempts failed in a way worth recording -- projected, so it survives a restart
     * @param dueAt    when it may be attempted again, null when it is due now
     */
    public record TransferToExecute ( TransferId transferId, String toIban, long amountInCents,
                                      int attempts, Instant dueAt, String correlationId ) implements CorrelatedTodoItem {

        boolean isDue ( Instant now ) {
            return dueAt == null || !dueAt.isAfter(now);
        }
    }

    @Override
    public EventQuery eventQuery ( ) {
        return EventQuery.forTypes(TransferRequested.class, TransferExecuted.class,
                                   TransferAttemptFailed.class, TransferAbandoned.class);
    }

    @Override
    protected Map<TransferId, TransferToExecute> initialState ( ) {
        return Map.of();
    }

    @Override
    protected Map<TransferId, TransferToExecute> apply ( Map<TransferId, TransferToExecute> state, Event<BankingDomainEvent> event ) {
        var next = new LinkedHashMap<>(state);          // requested order is the handling order
        switch ( event.data() ) {
            case TransferRequested requested -> next.put(requested.transferId(), new TransferToExecute(
                    requested.transferId(), requested.toIban(), requested.amountInCents(), 0, null,
                    Tracing.readFrom(event).correlationId()));
            case TransferExecuted executed -> next.remove(executed.transferId());
            case TransferAbandoned abandoned -> next.remove(abandoned.transferId());
            case TransferAttemptFailed failed -> next.computeIfPresent(failed.transferId(), ( id, item ) ->
                    new TransferToExecute(id, item.toIban(), item.amountInCents(),
                            failed.attempt(), failed.nextAttemptDueAt(), item.correlationId()));
            default -> { }
        }
        return next;
    }

    @Override
    public Stream<TransferToExecute> streamItems ( Limit limit ) {
        Instant now = Instant.now();
        return state().values().stream()          // one published, immutable state: no half-applied batch
                .filter(item -> item.isDue(now))   // a deferred item is withheld until it is due
                .limit(limit.value());
    }

    @Override
    public Optional<EventReference> lastEventReference ( ) {
        return Optional.ofNullable(upTo());
    }
}
```

Three things in there are worth copying:

1. **Removal is projected, never called.** `TransferExecuted` and `TransferAbandoned` take an item off. Removing it from a map by hand would bring it back at the next restart.
2. **The retry count lives on the item, projected from `TransferAttemptFailed`.** The automation reads it from there; nothing counts in memory.
3. **`streamItems` withholds what is not due.** That is the whole of "retry later with a delay": the item stays on the list, it is just not offered until its time comes.

### The automation

`Automation<T, D, O>` takes the item type first, then the domain events, then the outbound events. Note the to-do list is declared the other way round, `TodoListReadModel<D, T>`. The one method that does work is `handle`:

```java
public class ExecuteTransferAutomation implements Automation<TransferToExecute, BankingDomainEvent, BankingOutboundEvent> {

    static final int MAX_ATTEMPTS = 3;

    private final TransfersToExecuteTodoList todoList;
    private final TransferGateway gateway;

    public ExecuteTransferAutomation ( TransfersToExecuteTodoList todoList, TransferGateway gateway ) {
        this.todoList = todoList;
        this.gateway = gateway;
    }

    @Override
    public TodoListReadModel<BankingDomainEvent, TransferToExecute> getTodoList ( ) {
        return todoList;
    }

    @Override
    public Optional<EventReference> handle ( TransferToExecute transfer, AutomationContext<BankingDomainEvent, BankingOutboundEvent> context ) {
        // derived from the item, never from the attempt: the same transfer always gets the same key
        String key = "transfer-executed:" + transfer.transferId().value();
        String gatewayReference = gateway.execute(transfer.toIban(), transfer.amountInCents(), key);
        return context.event(new TransferExecuted(transfer.transferId(), gatewayReference),
                             TRANSFER.tags(transfer.transferId()), key);
    }

    // onFailure, delayBeforeNextBatch: below
}
```

`handle` returns the reference of the last event it produced, or `Optional.empty()` when it produced none. The `AutomationContext` offers two things: executing commands (`execute`, `executeWithRetry`) and appending a domain event no command decided on (`event`). Nothing else — an automation writes to the domain stream only, and what it decides reaches the outside world through a [publisher](/posts/eventmodeling-translators-publishers-dispatchers/).

**The idempotency key is derived from the item.** Since delivery is at least once, the same transfer will occasionally be handled twice. A key built from the transfer id makes the second append a no-op: the storage swallows a key it already holds on the stream and `event(...)` answers `Optional.empty()`. A key built from anything that changes per attempt — a random UUID, a timestamp, the attempt number — de-duplicates nothing. The gateway is handed the same key for the same reason: appending our event once is no use if the money moved twice.

> A command a handler executes gets its key the same way: `context.execute(command, "close-month:" + item.accountId().value())`. A command raising several events turns one key into `<key>/1`, `<key>/2`, … so a retry is swallowed whole. See [Commands and Decision Models](/posts/eventmodeling-commands-and-decision-models/).
{: .prompt-tip }

### Registering it: the slice owns its wiring

```java
@FeatureSlice(chapter = "Transfers")
public class ExecuteTransferFeatureSlice implements Slice<Banking> {

    @Override
    public void configureAutomation ( BoundedContextBuilder<Banking> builder ) {
        TransferGateway gateway = builder.port(TransferGateway.class);
        var todoList = new TransfersToExecuteTodoList();
        builder.readmodel(todoList).eventuallyConsistent();
        builder.automation(new ExecuteTransferAutomation(todoList, gateway));
    }
}
```

The gateway is a **port**: the slice asks for it by its interface, the application binds an adapter on the builder (`.adapter(gateway).forPort(TransferGateway.class)`), and a deployed slice whose port has no adapter fails at `build()` naming the port. See [Ports, Adapters and Monitored Calls](/posts/eventmodeling-ports-and-adapters/).

**The to-do list must be registered `.eventuallyConsistent()` on the same instance.** The automation waits for the to-do list's projector to have caught up before it starts a batch; registered alone, it would wait for a bookmark nobody ever writes. `build()` catches this for an `EPHEMERAL` or `LOCAL` to-do list:

```
automation registered without a projector for its todo list on this instance:
  BlockCardsOfFrozenAccountsAutomation (its todo list 'FrozenAccountsTodoList' is ephemeral and not
  registered: add builder.readmodel(todoList).eventuallyConsistent())
```

A `SHARED` to-do list (a durable [SQL read model](/posts/eventmodeling-sql-read-models/)) is exempt: its projector holds a lease of its own and may legitimately run on another instance.

### When `handle` throws: `onFailure`

Every failure is contained per item, and what it costs is the automation's decision. `onFailure(item, cause, context)` returns an `AutomationFailureAction`. **All three retry the item** — the framework has nothing to drop it from. They differ only in what happens to the rest of the work while it waits:

| action | this item | the items behind it |
|---|---|---|
| `RETRY_ITEM` (default) | on the next batch | wait for it |
| `CONTINUE_AND_RETRY_ITEM_LATER` | on a later batch | handled now, ahead of it |
| `STOP_AUTOMATION` | after a restart | wait for a human |

The default never lets work overtake a failure, because the framework cannot know whether the items behind the failing one depend on it. Of the two ways to be wrong, holding up independent items is a visible stall that clears with the cause; overtaking dependent ones produces wrong results and reports nothing. `CONTINUE_AND_RETRY_ITEM_LATER` is the one action that gives up the order, and you choose it when the items are genuinely independent — which transfers are.

The distinction that decides the action is not "transient or permanent" but **whose problem is it**:

```java
@Override
public AutomationFailureAction onFailure ( TransferToExecute transfer, Throwable cause, AutomationContext<BankingDomainEvent, BankingOutboundEvent> context ) {
    return switch ( cause ) {
        // the gateway, not the transfer: nothing to record, nothing to gain by moving on
        case GatewayUnavailableException down -> AutomationFailureAction.RETRY_ITEM;

        // this transfer, for good: the dead letter
        case TransferRejectedException rejected -> abandon(transfer, rejected.getMessage(), context);

        // this transfer, for now: defer it, unless it has used up its attempts
        case TransferDeclinedException declined -> {
            int attempt = transfer.attempts() + 1;
            if ( attempt >= MAX_ATTEMPTS ) {
                yield abandon(transfer, "declined %d times: %s".formatted(attempt, declined.getMessage()), context);
            }
            context.event(new TransferAttemptFailed(transfer.transferId(), declined.getMessage(), attempt,
                                                    Instant.now().plus(Duration.ofSeconds(5L << (attempt - 1)))),
                          TRANSFER.tags(transfer.transferId()),
                          "transfer-attempt-failed:%s:%d".formatted(transfer.transferId().value(), attempt));
            yield AutomationFailureAction.CONTINUE_AND_RETRY_ITEM_LATER;
        }

        // nobody planned for this: a human decides whether carrying on is safe
        default -> AutomationFailureAction.STOP_AUTOMATION;
    };
}

private AutomationFailureAction abandon ( TransferToExecute transfer, String reason, AutomationContext<BankingDomainEvent, BankingOutboundEvent> context ) {
    context.event(new TransferAbandoned(transfer.transferId(), reason, transfer.attempts() + 1),
                  TRANSFER.tags(transfer.transferId()),
                  "transfer-abandoned:" + transfer.transferId().value());
    return AutomationFailureAction.CONTINUE_AND_RETRY_ITEM_LATER;   // the item leaves once this is projected
}
```

| failure | event recorded | action | why |
|---|---|---|---|
| gateway unreachable | none | `RETRY_ITEM` | not a fact about this transfer; every transfer behind it would fail the same way |
| declined, may work later | `TransferAttemptFailed` | `CONTINUE_AND_RETRY_ITEM_LATER` | a fact about this one transfer; the event defers it, the others proceed |
| rejected, or out of attempts | `TransferAbandoned` | `CONTINUE_AND_RETRY_ITEM_LATER` | the dead letter: the projection drops the item for good |
| anything unexpected | none | `STOP_AUTOMATION` | a bug or a misconfiguration; nobody knows whether continuing is safe |

### Dead letters and delayed retries are events

There is no dead-letter queue and no retry scheduler in the framework, and none is needed:

- **A dead letter is an event the to-do list projects as "gone"** (`TransferAbandoned`). A read model over the same events is the dead-letter view an operator looks at.
- **A delayed retry is an event the to-do list projects as "not yet due"** (`TransferAttemptFailed.nextAttemptDueAt`), honoured by `streamItems`.

Both survive a restart because they are events, and both actually unblock the queue because only the to-do list decides what the next batch is offered. Anything held beside the to-do list would be lost at restart *and* unable to change what `streamItems` returns.

The events recorded from `onFailure` carry keys derived from the item and the attempt number (`transfer-attempt-failed:<id>:<n>`), so a redelivered failure does not record the same attempt twice.

> **There is no inline retry.** An item is never handed to `handle` twice within one batch. A failure worth retrying in milliseconds is about the call the handler made, so retry it inside `handle`, where the code knows what it attempted. The framework's answer is to back the whole batch off.
{: .prompt-info }

### Executing commands from `handle`

Most automations do not call the outside world; they execute a command per item. A command can conflict with another writer (`OptimisticLockingException`, the routine DCB outcome), and the bounded retry for that is `executeWithRetry`:

```java
@Override
public Optional<EventReference> handle ( AccountToLock item, AutomationContext<BankingDomainEvent, BankingOutboundEvent> context ) {
    return context.executeWithRetry(
            new BlockCardsOfAccountCommand(item.accountId(), "account frozen: " + item.reason()),
            "block-cards:" + item.frozenEventId(),          // derived from the item: the same on every redelivery
            RetryPolicy.of(5));
}
```

Each attempt re-executes the command, which re-reads its decision models and decides again against the new facts; only `OptimisticLockingException` is retried. A `BusinessException` from the re-decide propagates — and lands in your `onFailure`. A conflict cleared inside `handle` never reaches `onFailure` at all.

### Pacing: `batchSize`, `idlePollInterval`, `delayBeforeNextBatch`

An automation handles items in **batches**. Within a batch the to-do list does not move: events raised for one item reach the list only after the batch, once its projector has caught up. Between batches the automation waits until the to-do list has been projected past what it produced.

| method | default | what it decides |
|---|---|---|
| `batchSize()` | `50` | how stale the to-do list may be while a batch runs. `1` gives every item a list that reflects what the previous item raised — the choice when one item can cancel or supersede the ones behind it, at the cost of a projection round trip per item |
| `idlePollInterval()` | 10 s | how often an automation with nothing to do looks again. New work wakes it at once anyway; this paces only what the to-do list cannot see coming (a due time passing, a port asked in `streamItems`). An automation whose list reads the passing of the hour should choose an hour |
| `delayBeforeNextBatch(failedBatches, lastFailure)` | doubles from 10 s to 5 min | how long a batch that failed and handled nothing holds the automation back |

The third one matters most when a dependency is down. A failing batch is held for its delay **even if the to-do list changes underneath it**: new work arriving says nothing about whether the gateway is back, and releasing on it would retry at the pace of the traffic. Transfers are worth chasing harder than the default:

```java
@Override
public Duration delayBeforeNextBatch ( int consecutiveFailedBatches, Throwable lastFailure ) {
    if ( consecutiveFailedBatches <= 0 ) {
        return idlePollInterval();
    }
    Duration backoff = Duration.ofSeconds(2).multipliedBy(1L << Math.min(consecutiveFailedBatches - 1, 5));
    return backoff.compareTo(Duration.ofMinutes(1)) > 0 ? Duration.ofMinutes(1) : backoff;
}
```

Keep the two delays apart in your head: `TransferAttemptFailed.nextAttemptDueAt` defers **one item** and is durable because it is an event; `delayBeforeNextBatch` paces **the whole automation** and is deliberately not durable — a restart is a fine reason to try again at once.

### Correlation: `CorrelatedTodoItem`

Every event carries an `x-correlation-id` naming the flow it belongs to — command, domain event, automation, outbound event. An automation is the one hop the framework cannot bridge by itself: a to-do item is your own type and nothing links it back to the event that put it on the list.

`CorrelatedTodoItem` is that link. The to-do list captures the id where it is visible (`Tracing.readFrom(event).correlationId()` in `apply`, above) and the item carries it. When the item implements the interface, `handle` and `onFailure` get a context whose tracing carries that id, per item, so everything the automation raises for it lands in the same flow as the request that started it. An item that does not implement it gets the batch's tracing, whose id names the automation run. See [Observability and Tracing](/posts/eventmodeling-observability/).

### Parallelism: partition across automations

One automation is **sequential by construction**: one thread, on one leader, the next item only after the current one. That is what makes the order of `streamItems` mean anything, so there is no concurrency setting.

Where one queue is not fast enough, partition the work across several automation classes, each with its own to-do list over a disjoint, **stable** share of the items. One to-do list class can serve all of them under different `readmodelName()`s. Each automation runs on its own thread under its own lease, so the partitions proceed in parallel — possibly on different instances. Derive the partition from something like the account id, never from arrival order, because ordering holds within one automation and nowhere else.

### Watching and operating automations

An automation reports its lifecycle as bounded-context events:

| event | means |
|---|---|
| `AutomationStarted` | running — with reason `BOUNDED_CONTEXT_START` or `RESTART`; emitted even for an empty to-do list, so a deployed automation is visible from startup |
| `AutomationProcessed` | a batch ran, with its metrics (events streamed and handled, duration) |
| `AutomationFailed` | a batch failed and handled nothing; carries `consecutiveFailedBatches` so you pick your own alerting threshold |
| `AutomationStopped` | down while its context is up: a `STOP_AUTOMATION`, or an operator stop |

Fold `AutomationStarted`/`AutomationStopped` (the later wins) to answer "is it running". There is no "recovered" event: recovery is the next `AutomationProcessed` with a non-zero `eventsHandled`.

An operator holds `OperationsCapabilities` (see [Who May Do What](/posts/eventmodeling-who-may-do-what/)): `automations()` returns an `AutomationStatus` per automation — `running`, `leader`, `itemsFailed`, `consecutiveFailedBatches`, `lastFailure`, `stoppedBy`. A climbing `consecutiveFailedBatches` with nothing handled is a stall. `stopAutomation(name)` parks one (and hands its lease to another instance); `restartAutomation(name)` puts it back. Restarting without fixing the cause hands the same item over and stops again.

## Policies

### "Whenever X, then Y"

A large share of real to-do lists hold exactly one item per occurrence of one event, until the command's own event takes it off. Take "whenever an account is frozen, block its cards". As a to-do list it needs a list of frozen accounts, removed when... a `CardBlocked` arrives? An account without cards never produces one and stays on the list forever. A whole read model exists only to turn "an event happened" into "an item is outstanding", and it is easy to get wrong.

A **policy** reacts to the event itself:

```java
/** Whenever an account is frozen or closed, block its cards. */
public class BlockCardsOfAccountPolicy implements Policy<BankingDomainEvent> {

    @Override
    public EventQuery eventQuery ( ) {
        return EventQuery.forTypes(AccountFrozen.class, AccountClosed.class);
    }

    @Override
    public Optional<Command<BankingDomainEvent>> react ( Event<BankingDomainEvent> event ) {
        return switch ( event.data() ) {
            case AccountFrozen frozen -> Optional.of(new BlockCardsOfAccountCommand(frozen.accountId(), "account frozen: " + frozen.reason()));
            case AccountClosed closed -> Optional.of(new BlockCardsOfAccountCommand(closed.accountId(), "account closed"));
            default -> Optional.empty();
        };
    }
}
```

- **`eventQuery()`** names one event type, or several variants of one fact that lead to the same reaction.
- **`react(event)`** answers the command to issue, or `Optional.empty()` to let the event pass.
- **`policyName()`** defaults to the class' simple name and keys the policy's bookmark, lease and idempotency keys. Override it to keep the old name when you rename the class — a new name is a new policy, which either replays the history or skips what happened in between.

### It maps, and nothing more

`react` is handed no context: a policy has no ports, reads no read models, appends nothing and decides nothing. **The command decides**, on its own decision models, inside its own consistency boundary, exactly as it would for a user:

```java
/** Blocks every card of the account not blocked yet; with nothing left to block it raises nothing. */
public record BlockCardsOfAccountCommand ( AccountId accountId, String reason ) implements Command<BankingDomainEvent> {

    @Override
    public void execute ( CommandContext<BankingDomainEvent, BankingDomainEvent> context ) {
        var cards = new UsableCardsDecisionModel(accountId);
        var result = context.decisionModels(cards);
        for ( CardId cardId : cards.usable() ) {
            result.raiseEvent(new CardBlocked(accountId, cardId, reason),
                              Tags.of(ACCOUNT.tag(accountId), CARD.tag(cardId)));
        }
    }
    // UsableCardsDecisionModel: CardIssued adds, CardBlocked removes, tagged with the account
}
```

The command is **idempotent by its own rule**: run again for the same fact, it finds nothing left to block and raises nothing. That is what keeps the whole reaction safe to repeat, and it is the property to design every policy-issued command for. Its decision model is also its boundary: a card issued while the cards are being blocked conflicts the append, and the retry blocks it too.

### Registration: two choices, both mandatory

```java
@FeatureSlice(chapter = "Cards")
public class BlockCardsOfAccountFeatureSlice implements Slice<Banking> {

    @Override
    public void configureAutomation ( BoundedContextBuilder<Banking> builder ) {
        builder
            .command(BlockCardsOfAccountCommand.class)
            .policy(new BlockCardsOfAccountPolicy())
                .fromNowOn()
                .stallOnRejection();
    }
}
```

Neither choice has a default, because only the domain knows the answer; `build()` names a policy that left one unsaid:

```
policy registered without saying where it starts and what a rejection does:
  BlockCardsOfAccountPolicy (say .stallOnRejection() or .skipRejections())
```

**Where it starts, the first time it leads:**

| | reacts to |
|---|---|
| `fromNowOn()` | what happens after it first runs; the history before it was deployed is not revisited |
| `fromTheBeginning()` | every matching event ever recorded, once — for a policy deployed together with its events, or one whose command is idempotent over the whole history |

A policy has no to-do list remembering what is done, so deployed over an existing history without this choice it would issue its command for every matching event ever recorded. Once it has a bookmark it resumes from there, whatever this says.

**What a business rejection of its command does:**

| | on a `BusinessException` from the command |
|---|---|
| `stallOnRejection()` | the policy retries that event with backoff and everything behind it waits, reported as `PolicyFailed` each round — a rejection means something is wrong |
| `skipRejections()` | the rejection is the command's answer: reported as `PolicyEventSkipped` (and the command's own `CommandRejected`), and the policy moves on |

Every *other* failure always stalls the policy, and a permanent one (an unreadable event, a closed storage) retires it for an operator to restart. The card-blocking command rejects nothing by design, so a rejection would be a bug, and skipping it would leave a frozen account's cards usable — hence `stallOnRejection()`.

> There is deliberately no `onFailure` with a "retry later" action, as an automation has. Once the bookmark passes an event, nothing remembers it: a policy has nowhere to put an event aside. Work that must be put aside while the rest proceeds belongs on a to-do list.
{: .prompt-warning }

### Getting a stalled policy moving

An operator who decides a stalled event should not be reacted to after all skips it — on the instance the policy leads on:

```java
OperationsCapabilities ops = banking;
EventId stalledOn = policyFailed.failedAt().id();          // from the PolicyFailed event
ops.skipStalledEvent(ProcessorKind.POLICY, "BlockCardsOfAccountPolicy", stalledOn);
```

Only the event the policy is actually stalled on can be skipped, the skip takes effect at once rather than after the backoff, and it is reported as `PolicyEventSkipped` with reason `OPERATOR`. From another process, the same is the `ManagementInstruction.SkipStalledEvent` instruction — see [Running in Production](/posts/eventmodeling-running-in-production/).

### Redelivery: the reaction stands

A policy runs one domain event at a time, bookmarked after each. A crash between the command's append and the bookmark, or two leaders overlapping during a failover, hands the same event over again. The framework makes the reaction effectively once per event:

- The command runs with the policy as actor, `policy` as channel, the triggering event's correlation id, and every event it raises is tagged **`x-caused-by:<triggering event id>`**. Before reacting, the policy looks for an event it raised for this cause; finding one, it does not react again.
- The command is executed under the key `policy:<policy name>@<event id>`, which covers two leaders reacting at once.

What neither can cover is a reaction that raised *nothing* the first time — which is why the command must be idempotent by its own rules as well. A command that creates an entity derives the new entity's id from the triggering event rather than minting a fresh one.

A `CardBlocked` raised by the policy reads back with these tags (among the instance tags):

```
account:..., card:..., x-actor:BlockCardsOfAccountPolicy, x-channel:policy,
x-command:BlockCardsOfAccount, x-caused-by:<id of AccountFrozen>, x-correlation-id:<the freeze's flow>
```

The lifecycle events mirror the other processors: `PolicyStarted` (carrying both registration choices), `PolicyProcessed`, `PolicyFailed` (with `failedAt`, the stalled event), `PolicyStopped` and `PolicyEventSkipped`.

## Choosing between them

Take a **policy** when one fact simply leads to one action, and the command can find out everything else itself. Take an **automation with a to-do list** when the work needs something a policy deliberately lacks:

| the work needs | why a policy cannot |
|---|---|
| a port: a payment gateway, a mail server | `react` gets no context and calls nothing |
| a deadline or the passing of time | a policy reacts to events, not to their absence |
| items gathered from several facts, or several changes conflated into one action | a policy sees one event at a time |
| an item put aside and retried later while the rest proceeds | once the bookmark passes an event, nothing remembers it |

The transfer example needs a port and retries per item: a to-do list. The card example needs neither: a policy.

## Testing

Both have a synchronous, deterministic test base in `sliceworkz-eventmodeling-testing` — no polling, no sleeps. `AutomationTest` runs the production batch loop; its `whenItemsAreRedelivered()` replays the crash between the append and the bookmark, so an item-derived key is a one-line assertion:

```java
@Test
void aRedeliveredTransferIsRecordedOnce ( ) {
    given(new TransferRequested(T1, A1, "BE12", 100))
        .whenBatchRuns()
            .events(new TransferExecuted(T1, "GW-1"))
        .and()
        .whenItemsAreRedelivered()       // the crash between the append and the bookmark
            .itemsHandled(1)
            .noEvents();                  // the item-derived key made the repeat a no-op
}
```

`PolicyTest` does the same for a policy, `whenRedelivered()` being the case where a correct policy raises nothing:

```java
@Test
void aFrozenAccountsCardsAreBlockedOnce ( ) {
    given(new CardIssued(A1, C1), ACCOUNT.tags(A1))
        .and(new AccountFrozen(A1, "fraud"), ACCOUNT.tags(A1))
    .whenReacting()
        .events(new CardBlocked(A1, C1, "account frozen: fraud"))
    .and()
    .whenRedelivered()
        .noEvents();
}
```

See [Testing Feature Slices](/posts/eventmodeling-testing/) for both bases, and [`ExecutePaymentAutomationTest`](https://github.com/sliceworkz/eventmodeling/blob/develop/sliceworkz-eventmodeling-examples/src/test/java/org/sliceworkz/eventmodeling/examples/payments/features/executepayment/ExecutePaymentAutomationTest.java) for every `onFailure` branch covered, deferred retries included — tested by seeding an attempt whose due time has already passed rather than waiting one out.

## Summary

- The to-do list is the queue, projected from events. Removal, retry counts, dead letters and delays are all events it projects.
- Delivery is at least once: give every effect an idempotency key derived from the item.
- `onFailure` decides what a failure costs the rest of the work; the default holds the order.
- `delayBeforeNextBatch` backs off a failing dependency; `idlePollInterval` paces an idle automation; `batchSize` bounds how stale the list may be.
- Register the to-do list `.eventuallyConsistent()` beside the automation; `build()` checks.
- A policy maps one event to one command and nothing else; say where it starts and what a rejection does; make its command idempotent.

Next: [Translators, Publishers and Dispatchers](/posts/eventmodeling-translators-publishers-dispatchers/) — how events enter and leave a bounded context.
