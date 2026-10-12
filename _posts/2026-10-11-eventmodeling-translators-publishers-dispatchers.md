---
layout: post
toc: true
title: Translators, Publishers and Dispatchers
description: How events cross the edge of a bounded context — translators bring external events in, publishers map recorded facts into the published language, dispatchers send them out.
date: 2026-10-11 09:00:00
categories: [Eventmodeling Documentation,Eventmodeling Guides]
tags: [eventmodeling, translator, publisher, dispatcher, outbox, integration events, idempotency]
---

A bounded context runs on three streams, one per kind of event it declares — `BoundedContext<D, I, O>`:

| stream | holds | written by | read by |
|---|---|---|---|
| **inbound** (`I`) | events other systems told us about | `incoming(...)` | translators |
| **domain** (`D`) | the facts this context decided | commands, automations, translators | everything inside |
| **outbound** (`O`) | the published language: what we tell the world | publishers, and only publishers | dispatchers |

Three components move events across the edges:

```
                    ┌──────────────────── bounded context ────────────────────┐
external event ─► incoming ─► inbound ─► Translator ─► command ─► domain stream ─► Publisher ─► outbound ─► Dispatcher ─► message bus, webhook, mail…
                    └──────────────────────────────────────────────────────────┘
```

All three are projector-driven processors in a slice's **automation aspect**: they run on the single elected leader of the deployment, are bookmarked on the stream they read, retry a failing event with backoff, and are delivered **at least once**. This guide uses the banking domain of the [quickstart](/posts/eventmodeling-quickstart/), extended with what crosses the edge:

```java
sealed interface BankingInboundEvent {
    record SalaryPaid ( String accountId, long amountInCents, String payrollReference ) implements BankingInboundEvent { }
}

sealed interface BankingOutboundEvent {
    record AccountOpenedNotice ( AccountId accountId, CustomerId customerId ) implements BankingOutboundEvent { }
    record BalanceChanged ( AccountId accountId, long balanceInCents ) implements BankingOutboundEvent { }
}

// and on the domain side
record MoneyDeposited ( AccountId accountId, long amountInCents ) implements BankingDomainEvent { }
record DepositRefused ( AccountId accountId, long amountInCents, String reason ) implements BankingDomainEvent { }
```

Inbound and outbound events are **integration events**: a contract with another system, separate from the domain events you are free to evolve. A translator is where the external contract stops; a publisher is where yours starts.

## Translators: bringing events in

### The translator

A `Translator<I, D>` selects inbound events with `eventQuery()` and handles each in `translate(Event<I>, TranslatorContext)`. Its context is the same as an automation's: execute commands, or append a domain event no command decided on.

A payroll provider tells the bank a salary was paid. That becomes a deposit — through the command, so the bank's own rules decide it:

```java
/** A salary paid by the payroll provider becomes a deposit -- or, when the bank cannot accept it, a recorded refusal. */
public class SalaryPaidTranslator implements Translator<BankingInboundEvent, BankingDomainEvent> {

    @Override
    public EventQuery eventQuery ( ) {
        return EventQuery.forTypes(SalaryPaid.class);
    }

    @Override
    public void translate ( Event<BankingInboundEvent> event, TranslatorContext<BankingInboundEvent, BankingDomainEvent> context ) {
        switch ( event.data() ) {
            case SalaryPaid salary -> {
                AccountId accountId = ACCOUNT.id(salary.accountId());
                String key = "salary:" + event.reference().id().value();     // derived from the inbound event
                try {
                    context.executeWithRetry(new DepositMoneyCommand(accountId, salary.amountInCents()), key,
                                             RetryPolicy.DEFAULT);
                } catch ( BusinessException refused ) {
                    // decided about in-band: recorded as a fact, never thrown at the projector
                    context.event(new DepositRefused(accountId, salary.amountInCents(), refused.getMessage()),
                                  ACCOUNT.tags(accountId), key + "/refused");
                }
            }
        }
    }
}
```

`translate` is handed the inbound `Event`, not just its payload — the one-`when` rule every component follows — so the tags, the timestamp and the reference are at hand. The reference's id is what keys the translation: a redelivered inbound event raises nothing twice.

### A translator must not throw

What has arrived has arrived: the inbound event is a fact about the other system, and refusing it changes nothing about that. A translator runs behind a projector, so a translator that throws is **retried with backoff, and every inbound event behind the bad one waits**, reported as `TranslatorFailed` each round. One malformed message would stall the whole inbound edge.

So an inbound event that fails validation — an unknown account, an amount the bank will not take — is **decided about in-band**:

- ignore it, the `default -> { }` arm of a `switch` over several inbound types, or
- record the refusal as a domain event (`DepositRefused` above), which makes the dead-letter view an ordinary read model.

Catching `BusinessException` from the command is the clean way to do the second: the command keeps the rule, the translator records what came of it. Throw only for what retrying can fix — a dependency that is down. See [Where a Validation Goes](/posts/eventmodeling-where-validations-go/).

### `incoming(...)` or `translate(...)`

Application code (an HTTP endpoint receiving a webhook, a message-bus consumer) feeds inbound events in through the context's `TranslationCapability<BankingInboundEvent>` — the narrow type to hand such an adapter, since feeding the domain external events is a different job from executing commands on a user's behalf (see [Who May Do What](/posts/eventmodeling-who-may-do-what/)). There are two ways, reusing the same registered translators:

| | `incoming(event)` | `translate(event)` |
|---|---|---|
| runs | asynchronously, on the leader's translator processors | synchronously, on the calling thread |
| inbound event | appended to the inbound stream | stored nowhere |
| returns | nothing | the references of the domain events raised |
| a translator that throws | retried with backoff | the exception propagates to the caller |
| no translator matches | the event waits on the inbound stream | `NoTranslatorRegisteredException` |

```java
// a webhook: store the message, acknowledge, let the translator catch up
banking.incoming(new SalaryPaid(accountId.value(), 250_000, "PAY-2026-10"), "payroll:PAY-2026-10");

// a caller that needs to act on the outcome right away
List<EventReference> raised = banking.translate(new SalaryPaid(accountId.value(), 250_000, "PAY-2026-10"));
```

`incoming` is the default: the inbound event is durable before you acknowledge it, and the translator retries without the caller. Give it an **idempotency key from the sender's own message id** (`incoming(event, key)`), so a sender retrying its delivery stores the message once.

`translate` suits an integration that wants the domain events immediately. Two things differ. Because there is a caller to reject, a translator may fail loudly on this path. And the `Event` the translator is handed is built for the call: its reference carries a freshly minted id under a position that names no stored event — fit to key what the translation raises, so the events of *one* translation are consistent, and for nothing else. Calling `translate` twice with the same message is two translations; protect that with a key of your own on the commands the translator executes, derived from something in the payload such as `payrollReference`. `translate` runs the translators registered on the instance it is called on — so in a deployment split by aspect, call it on an instance that runs the aspect its translators are registered from, or it answers `NoTranslatorRegisteredException`.

### Registering a translator

```java
@FeatureSlice(chapter = "Money")
public class PayrollIntakeFeatureSlice implements Slice<Banking> {

    @Override
    public void configureAutomation ( BoundedContextBuilder<Banking> builder ) {
        builder.translator(new SalaryPaidTranslator());
    }
}
```

`builder.translator(Class)` constructs a translator with a no-argument constructor itself; register an instance when it needs anything passed in. A translator's name is its class' simple name, and it keys its bookmark on the inbound stream — see [Names are bookmark keys](#names-are-bookmark-keys) below.

### The flow continues

Every event carries an `x-correlation-id` naming its flow. A translator continues the flow of what it translates: the domain events it raises carry the inbound event's correlation id (an inbound event without one starts a flow of its own), and on the `translate` path the caller's tracing. So "everything that came of this salary payment" is a tag query. See [Observability and Tracing](/posts/eventmodeling-observability/).

## Publishers: telling the world what was decided

### A command records, a publisher publishes

A command decides and records its conclusion as **domain events**. Telling other systems about that conclusion is a separate step: a `Publisher<D, O>` maps domain events into **outbound events** after they are stored.

```
command ─► domain event ─► Publisher ─► outbound event ─► Dispatcher ─► external system
```

Every hop is bookmarked and at least once, and every outbound event traces back to the domain event it was published for. A publisher whose domain event carries everything the integration event needs is a few lines:

```java
/** The domain event carries everything the integration event does, so this reads no read model. */
public class AccountOpenedNoticePublisher implements Publisher<BankingDomainEvent, BankingOutboundEvent> {

    @Override
    public EventQuery eventQuery ( ) {
        return EventQuery.forTypes(AccountOpened.class);
    }

    @Override
    public void publish ( Event<BankingDomainEvent> event, PublisherContext<BankingDomainEvent, BankingOutboundEvent> context ) {
        if ( event.data() instanceof AccountOpened opened ) {
            context.publish(new AccountOpenedNotice(opened.accountId(), opened.customerId()),
                            ACCOUNT.tags(opened.accountId()));
        }
    }
}
```

`publish` may call `context.publish(...)` zero, one or several times for one domain event; everything it publishes is appended to the outbound stream together, once the method returns.

### It decides nothing

The `PublisherContext` offers reads and `publish`, and nothing else: no commands, no domain events, no ports. Whatever needs deciding — *whether* to publish, *what* the message says beyond the facts — is a command's decision, recorded as a domain event first, which the publisher then maps. That is what makes a publisher's output a pure function of history, and therefore safe to redeliver.

A publication that depends on a decision taken later (waiting for other facts, a deadline, an external answer) is a to-do list and an automation executing a command, whose domain event a publisher then maps. See [Automations and Policies](/posts/eventmodeling-automations-and-policies/).

### Reading state: every read says *as of when*

An outbound event often carries more than the domain event: a balance, a status, a total. A publisher reads **live read models**, and every read is bounded, because whether the consumer receives the state *as it was when the fact was recorded* or *as it is now* is a decision every consumer lives with:

| read | sees | use it when |
|---|---|---|
| `readAsOfEvent(class, params...)` | up to and including the event being published | usually: a burst of changes publishes the sequence of states it went through |
| `readAsOf(reference, class, params...)` | up to and including an event you name | the decision was recorded by another event than the one being published |
| `readLatest(class, params...)` | up to the domain stream's head, pinned once per publication | the consumer wants the newest data and only convergence matters |

Each read model a publisher reads is declared in `reads()`:

```java
/** Every deposit is published as the balance it left the account with. */
public class BalanceChangedPublisher implements Publisher<BankingDomainEvent, BankingOutboundEvent> {

    @Override
    public EventQuery eventQuery ( ) {
        return EventQuery.forTypes(MoneyDeposited.class);
    }

    @Override
    public Set<Class<? extends ReadModel<? extends BankingDomainEvent>>> reads ( ) {
        return Set.of(AccountBalanceReadModel.class);
    }

    @Override
    public void publish ( Event<BankingDomainEvent> event, PublisherContext<BankingDomainEvent, BankingOutboundEvent> context ) {
        if ( event.data() instanceof MoneyDeposited deposited ) {
            var balance = context.readAsOfEvent(AccountBalanceReadModel.class, deposited.accountId());
            context.publish(new BalanceChanged(deposited.accountId(), balance.balanceInCents()),
                            ACCOUNT.tags(deposited.accountId()));
        }
    }
}
```

Two deposits of 10.00 and 5.00 publish `BalanceChanged(1000)` and `BalanceChanged(1500)`, in that order, however late the publisher gets round to them. With `readLatest` both would read 1500.

A bounded read cannot start from a seed or a snapshot that already reflects later events, so it replays from the beginning. Keep a read model a publisher reads *as of an event* to the size of a decision model — one entity's events, narrowly tagged, as `AccountBalanceReadModel` is. See [Choosing a Read Model](/posts/eventmodeling-read-models/).

### Publishing is effectively once per domain event

A crash between the outbound append and the bookmark, or two leaders overlapping during a failover, hands a domain event to the publisher again. The framework keys what it publishes so that **the first publication stands**:

- Each outbound event is keyed `<domain event id>/<n>`, in the order of the `publish` calls, and all of them are appended together.
- A redelivery that publishes the same again is a retry the store swallows.
- A redelivery that publishes something else — possible only through `readLatest`, since the state may have moved on — is recognised as published before, and stores nothing.

So outbound events need no key of yours. Every outbound event is also tagged with the framework's causation link:

```
account:..., x-published-from:<domain event id>, x-actor:BalanceChangedPublisher,
x-channel:publication, x-correlation-id:<the flow of the deposit>
```

`x-published-from` (`Publisher.TAG_PUBLISHED_FROM`) makes "why did we publish this" and "what did we publish for that" tag queries.

### Registering a publisher

A publisher belongs to the slice whose domain event it publishes and is part of that slice's **automation aspect**:

```java
@FeatureSlice(chapter = "Money")
public class BalanceNotificationFeatureSlice implements Slice<Banking> {

    @Override
    public void configureAutomation ( BoundedContextBuilder<Banking> builder ) {
        builder.readmodel(AccountBalanceReadModel.class).live();      // what the publisher reads, on this instance
        builder.publisher(new BalanceChangedPublisher());
        builder.dispatcher(new BalanceChangedDispatcher(builder.port(CustomerNotifications.class)));
    }
}
```

`build()` holds the registration to two rules, each naming every offender:

- **A publisher a slice registers from another hook than `configureAutomation` is refused.** It runs where automations run; registered from `configureQuery` it would publish from the instances serving queries, in a split deployment possibly from nowhere anyone looks.
- **Every class in `reads()` must be registered `.live()` on the same instance.** A read model registered only from `configureQuery` does not exist on an instance that runs automations alone. Register it beside the publisher; registering the same class `.live()` from `configureQuery` as well, for an endpoint, is fine — two identical live registrations are one.

A read of a class not declared in `reads()` is an `IllegalArgumentException` at the read.

> Where several slices publish the same integration event, one publisher in a slice of its own — holding the mapping and the dispatcher — is usually clearer than one per slice. A slice that only publishes is derived as an automation slice.
{: .prompt-tip }

## Dispatchers: the outbox

### Sending what was published

A `Dispatcher<O>` is a projection over the outbound stream: an `eventQuery()` and a `when(Event<O>)`. It is the **outbox pattern** without a table of its own — the outbound stream *is* the outbox, written in the same store as the domain, and the dispatcher's bookmark records how far it has been sent.

A dispatcher calls the outside world, so it takes a port:

```java
/** The port to whatever tells customers about their account: a push service, a mail server. */
public interface CustomerNotifications {

    /** Sends at most once per key; a repeat with the same key is acknowledged without sending again. */
    void balanceChanged ( AccountId accountId, long balanceInCents, String deduplicationKey );
}

/** Sends every BalanceChanged to the customer notification service. */
public class BalanceChangedDispatcher implements Dispatcher<BankingOutboundEvent> {

    private final CustomerNotifications notifications;

    public BalanceChangedDispatcher ( CustomerNotifications notifications ) {
        this.notifications = notifications;
    }

    @Override
    public EventQuery eventQuery ( ) {
        return EventQuery.forTypes(BalanceChanged.class);
    }

    @Override
    public void when ( Event<BankingOutboundEvent> event ) {
        if ( event.data() instanceof BalanceChanged changed ) {
            // the outbound event's id is stable across redeliveries: the receiver de-duplicates on it
            notifications.balanceChanged(changed.accountId(), changed.balanceInCents(), event.reference().id().value());
        }
    }
}
```

The application binds the adapter on the builder (`.adapter(pushService).forPort(CustomerNotifications.class)`); binding it `.monitored(...)` reports every call. See [Ports, Adapters and Monitored Calls](/posts/eventmodeling-ports-and-adapters/).

### At least once, to a system you do not control

The dispatcher is bookmarked after each batch it handles. A crash before the bookmark, a failure part-way through a batch (the batch is rolled back and handed over again from its start), a failover overlap, a bookmark lost or reset: each delivers events the external system has already received. Nothing the framework does can un-send a message, so **the receiver has to de-duplicate**, and the dispatcher has to give it something stable to de-duplicate on.

The outbound event's id is that thing: it is assigned once, when the publisher's append stored the event, and every redelivery carries the same one. Pass it along as the message id, the `Idempotency-Key` header, the deduplication id of the queue.

A dispatcher that throws is retried with backoff, holding everything behind it — which is what you want while the receiving end is down: nothing is lost, it all waits on the outbound stream. It is reported as `DispatcherFailed` each round, `DispatcherProcessed` when it moves again. An operator can also hold a dispatcher back deliberately during the receiver's maintenance window, `stopProcessor(ProcessorKind.DISPATCHER, "BalanceChangedDispatcher")`, and resume it with `restartProcessor`.

### Names are bookmark keys

A translator, a dispatcher, a publisher and a read model are each identified by a name — the class' simple name — which becomes the reader name of the **bookmark** recording how far that component has got. Two properties are load-bearing, and `build()` checks both:

- **Unique within its kind.** Two dispatchers sharing a name share one bookmark, so each advances it past events the other never sent. Nothing would throw; the events would simply never be published.
- **Stable across restarts.** An anonymous class has no simple name, and a lambda or generated class gets a new one per JVM run — a fresh bookmark on every start, so the whole outbound stream is re-sent at every boot.

For a dispatcher, getting the name wrong is the costliest mistake in the framework: duplicate publishing to an external system. Use a named class, and do not rename it casually — a new name is a new bookmark, and it starts from the beginning of the outbound stream.

## Why there is no command that raises outbound events

It looks simpler for a command to raise its domain event *and* the outbound event in one go. Two reasons it is not offered:

- **The two streams commit independently.** Domain and outbound are separate streams, and the event store appends to one stream per call, so a command raising into both could not be atomic: a crash between the two appends leaves a fact never published, or a publication of a fact never recorded.
- **A command raising outbound events only would have no consistency boundary.** Its decision models read domain events, and the optimistic-locking check travels with the append — to a stream where domain event types never occur. The check would match nothing and admit everything, silently.

A publisher needs none of this: it decides nothing, and its keys come from the domain event, so there is nothing to order and nothing to forget. The cost is one bookmarked hop between the fact and its publication.

## Operating and observing

All three run as processors and report the same lifecycle, named per kind:

| | started | did work | retrying | retired |
|---|---|---|---|---|
| translator | `TranslatorStarted` | — | `TranslatorFailed` | `TranslatorStopped` |
| publisher | `PublisherStarted` | `PublisherProcessed` | `PublisherFailed` | `PublisherStopped` |
| dispatcher | `DispatcherStarted` | `DispatcherProcessed` | `DispatcherFailed` | `DispatcherStopped` |

`…Failed` carries `consecutiveFailedRuns` and the event it failed on. `…Stopped` means retired on a permanent failure (an event that cannot be read, a closed storage) or stopped by an operator, and needs a human. `ProcessorAdminCapability.processors()` — part of `OperationsCapabilities` — lists them with their state, and `restartProcessor`/`stopProcessor` take a `ProcessorKind` (`TRANSLATOR`, `PUBLISHER`, `DISPATCHER`) and the name. See [Running in Production](/posts/eventmodeling-running-in-production/).

## Testing

Each has a test base in `sliceworkz-eventmodeling-testing` that drives it synchronously without registering it:

```java
@Test
void aBurstOfDepositsIsPublishedAsTheStatesItWentThrough ( ) {
    given(new MoneyDeposited(A1, 1000), ACCOUNT.tags(A1))
        .and(new MoneyDeposited(A1, 500), ACCOUNT.tags(A1))
    .whenPublished()
        .published(new BalanceChanged(A1, 1000), new BalanceChanged(A1, 1500))
    .and()
    .whenRepublished()                 // the same domain events handed over again
        .nothingPublished();
}
```

`PublisherTest` reads the declared live models bounded exactly as a deployment does; `DispatcherTest` has `whenDispatched()` and `whenRedeliveredFromTheStart()` (the lost bookmark); `TranslatorTest` drives the synchronous `translate` path. See [Testing Feature Slices](/posts/eventmodeling-testing/).

## Summary

- **Translators** turn inbound events into domain events, usually through a command. They must not throw on validation: decide in-band, record a refusal as a domain event. Key what they raise from the inbound event's id. `incoming` is durable and asynchronous; `translate` is synchronous and stores nothing.
- **Publishers** map recorded facts into outbound events and decide nothing. Every read says as of when; `readAsOfEvent` is the usual choice. Keys (`<event id>/<n>`) and `x-published-from` come for free, and the first publication stands. Register from `configureAutomation`, with their live read models beside them.
- **Dispatchers** send the outbound stream to the outside world, at least once. Hand the receiver the outbound event's id to de-duplicate on, and keep the dispatcher's name stable — it is its bookmark.
- The chain is command → domain event → publisher → outbound event → dispatcher; there is no shortcut that writes outbound events from a command.
