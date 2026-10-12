---
layout: post
toc: true
title: Observability and Tracing
description: What a bounded context reports about itself — lifecycle and outcome events for a monitoring stream, scoped observations for metrics and tracing, and correlation ids that follow one flow across every step.
date: 2026-10-12 14:00:00
categories: [Eventmodeling Documentation,Eventmodeling Deployment]
tags: [eventmodeling, observability, monitoring, tracing, correlation id, metrics, micrometer, dashboard]
---

A bounded context in production has to answer three questions from the outside: *what happened* (which commands ran, which were rejected, which read model stopped), *how long did it take*, and *what belongs together* (everything one request caused, across commands, automations and contexts). The framework answers each through its own seam, and names no metrics or tracing library in any of them:

| question | seam | what it is |
|---|---|---|
| what happened | `BoundedContextListener` | a stream of `BoundedContextEvent`s — facts about the context's lifecycle and the outcome of every piece of work, meant to be stored and read back |
| how long, and what nested in what | `BoundedContextObserver` | a scope per operation, opened before and closed after it on the caller's thread — the place to bind meters and spans |
| what belongs together | `Tracing` | actor, channel and a correlation id, written as `x-*` tags on every event the context appends |

All three are opt-in and cost nothing until configured. This guide uses the banking context of the [quickstart](/posts/eventmodeling-quickstart/); the eventstore underneath has an observer of its own, covered in [Observability with Micrometer](/posts/eventstore-observability-micrometer-prometheus-grafana/).

## Bounded Context Events

### Registering a listener

The kernel of a bounded context produces a `BoundedContextEvent` for everything it does, and hands it to the listener registered on the builder. Two implementations ship with the framework:

```java
// log every event at INFO, under the logger name BOUNDEDCONTEXT
.listener(new LoggingBoundedContextListener())

// append every event to a stream of your own -- a monitoring stream
.listener(new StreamAppendingBoundedContextListener(reports))
```

`BoundedContextListener` is a functional interface with one method, `on(EphemeralEvent<BoundedContextEvent>)`, and the builder takes one listener. To do two things, compose them:

```java
BoundedContextListener log = new LoggingBoundedContextListener();
BoundedContextListener monitor = new StreamAppendingBoundedContextListener(reports);

Banking banking = BoundedContext.newBuilder(Banking.class)
        .name("banking")
        .eventStorage(storage)
        .listener(event -> { log.on(event); monitor.on(event); })
        // ...
        .build();
```

With no listener registered, the kernel produces no events at all.

The event arrives as an `EphemeralEvent` that already carries the tags of the work it reports on — the instance, and the actor, channel and correlation id of the flow (see [Tracing](#tracing-one-flow-one-correlation-id) below) — so it can be appended to an `EventStream` as it is, or inspected through `data()`.

### What a listener costs, and what its failure costs

The listener is called **synchronously, on the thread doing the work**: a command's `CommandExecuted` is delivered on the thread that executed the command, before `execute` returns. Keep it fast, or buffer inside it. A monitoring store that is slow adds its latency to every command.

A listener's failure is **never the work's failure**. An exception escaping `on(...)` is contained by the kernel, logged at ERROR (the first time in full, repeats summarised at most once a minute) and reported to the observer's `listenerFailed`; the command, batch or projection carries on as if no listener were registered. This matters most where the event is emitted after the work is durable: a `CommandExecuted` whose delivery threw would otherwise report a command that succeeded as failed, and invite the caller to run it again. A listener that always throws does not stop the context coming up either.

The corollary: **nothing replays what a failing listener missed**. The event is dropped and the next one is delivered normally, so a monitoring stream is a best-effort record. A listener that must not lose events buffers and retries inside itself. `Error`s are not contained.

### The events

`BoundedContextEvent` is a sealed interface of records. Every one names its bounded context; the ones about work also carry the `FeatureSlice` it belongs to.

**Lifecycle**

| event | when |
|---|---|
| `BoundedContextStarting` | `start()` begins: the instance (logical, physical, process), the deployed and undeployed feature slices with their members, the aspects this instance runs, the declared rulebook, and every port binding with the slices that asked for it |
| `BoundedContextStarted` | the context is usable; `startupDurationMicros` is the time spent in `start()` |
| `BoundedContextStopping` / `BoundedContextStopped` | `stop()` or `terminate()`, before and after |

`BoundedContextStarting` is the inventory: it is the one place a dashboard learns what a deployment *carries* before any of it has done anything — a read model nothing has updated yet, a business rule never violated, a port never called.

**Commands, and the reads they make**

| event | when |
|---|---|
| `CommandExecuted` | a command ran and its events are durable; carries their references |
| `CommandRejected` | a `BusinessException` (a business rule saying no): carries the reason and, for a rejection on the rules, the rule judgements — no stack trace |
| `CommandFailedOnOptimisticLocking` | the consistency boundary moved: a DCB conflict, the routine outcome under contention |
| `CommandFailed` | anything else, with the whole `Failure` (type, message, stack trace) |
| `DecisionModelProjected`, `LiveModelProjected`, `AggregateLoaded` | a model was projected for a command or a read |

The split is the point: **alert on `CommandFailed`, chart `CommandRejected`**. A customer with insufficient funds is a business answer and belongs in a chart by command and reason; a `NullPointerException` belongs in an alert. One event for both would make the dashboard choose between alerting on every rejection and alerting on nothing (see [Where a Validation Goes](/posts/eventmodeling-where-validations-go/) for why rules throw `BusinessException`).

**Processors**

| kind | started | did work | failing, still retrying | stopped while the context runs |
|---|---|---|---|---|
| eventually consistent read model | `ReadModelProjectorStarted` | `EventuallyConsistentReadModelUpdated` | `ReadModelProjectorFailed` | `ReadModelProjectorStopped` |
| automation | `AutomationStarted` | `AutomationProcessed` | `AutomationFailed` | `AutomationStopped` |
| policy | `PolicyStarted` | `PolicyProcessed` | `PolicyFailed` | `PolicyStopped`, plus `PolicyEventSkipped` |
| translator | `TranslatorStarted` | — | `TranslatorFailed` | `TranslatorStopped` |
| publisher | `PublisherStarted` | `PublisherProcessed` | `PublisherFailed` | `PublisherStopped` |
| dispatcher | `DispatcherStarted` | `DispatcherProcessed` | `DispatcherFailed` | `DispatcherStopped` |

Three rules make these readable:

- **"Is it running?" is a fold of `...Started` and `...Stopped`, the later winning.** `...Started` is emitted for every processor when the context starts, so a read model whose stream holds nothing for it is announced from startup instead of never — otherwise an idle processor and a dead one would look the same for as long as nobody appends.
- **`...Stopped` is not emitted at shutdown.** `BoundedContextStopping` already says it for everything at once, which leaves `...Stopped` meaning the one state worth alerting on: a processor down while its context is up. It carries a reason — a permanent failure (with the failure and the event it died on) or `OPERATOR`.
- **`...Failed` is once per fruitless round**, paced by the backoff, with `consecutiveFailedRuns` (or `consecutiveFailedBatches` for an automation) so the alerting threshold is yours. Recovery has no event of its own: it is the next `...Updated`/`...Processed` (see [Running in Production](/posts/eventmodeling-running-in-production/#when-things-fail)).

`ReadModelProjectorStarted` also carries `readModelType` — whether the read model is `ephemeral`, `local` or `shared` — which is the only place that is stated before anything has been projected.

**Leadership, ports and operations**

| event | when |
|---|---|
| `LeadershipAcquired` / `LeadershipReleased` | per leader-only processor, with the fencing token or the reason it was released |
| `PortCalled` / `PortCallRejected` / `PortCallFailed` | a call through a monitored port returned, said no, or failed — with the caller (component and slice) and the duration |
| `PortCallsSummarized` | a summarized port's calls per interval: counts per outcome and a latency histogram |
| `InstructionHandled` / `InstanceStatusReported` | the answer to a management instruction (see [Running in Production](/posts/eventmodeling-running-in-production/#reaching-every-instance-the-management-stream)) |

Ports are covered in [Ports, Adapters and Monitored Calls](/posts/eventmodeling-ports-and-adapters/).

### Metrics and durations

Most work events carry a `Metrics` record:

| field | meaning |
|---|---|
| `durationMicros` | how long it took, **in microseconds**, on a monotonic clock |
| `queriesDone` | event-store queries made |
| `eventsStreamed` | events read |
| `eventsHandled` | events handed to the component |
| `until` | the reference the read was bounded at, where it was |

Every duration on every `BoundedContextEvent` — `Metrics.durationMicros`, `startupDurationMicros`, the port events' `durationMicros`, `totalMicros` and `maxMicros` — is a `long` in microseconds, measured the same way, so they compare directly: a port call nested in a command is never reported longer than the command, and a command finishing in 40 µs does not read as zero. One `CommandExecuted` from the scratch project behind these guides, as the logging listener prints it:

```text
CommandExecuted[boundedContext=banking, command=RegisterCustomer, raisedEvents=[1a197aa9-...:1:1],
  metrics=Metrics[durationMicros=76449, queriesDone=0, eventsStreamed=0, eventsHandled=0, until=null],
  slice=FeatureSlice[name=RegisterCustomer, type=STATE_CHANGE, context=Banking, chapter=Customers, members=[...]]]
Tags[x-instance-logical:banking-app, x-instance-physical:default, x-instance-process:vm-10682-...,
  x-actor:alice@bank, x-channel:api, x-command:RegisterCustomer, x-correlation-id:req-42]
```

The value records nested in these events ignore properties they do not know, so a reader on an older version keeps reading a monitoring stream written by a newer one.

### A monitoring stream

For anything beyond a log, append the events to a stream in a store of its own, shared by every instance and every bounded context you run:

```java
EventStore monitoring = EventStore.on(monitoringStorage).build();

EventStream<BoundedContextEvent> reports = monitoring.getEventStream(
        EventStreamId.forContext("monitoring").withPurpose("boundedcontext"), BoundedContextEvent.class);

Banking banking = BoundedContext.newBuilder(Banking.class)
        .name("banking")
        .eventStorage(storage)
        .instance(instance)
        .listener(new StreamAppendingBoundedContextListener(reports))
        // ...
        .build();
```

The same store is the natural home for the management stream an operator uses to stop and restart processors (see [Running in Production](/posts/eventmodeling-running-in-production/#reaching-every-instance-the-management-stream)). Because the events carry the instance, the slice and the correlation id, everything a monitoring tool needs is a query on that stream: the lifecycle events folded per processor and instance, `CommandRejected` grouped by reason, every event under one correlation id.

> In production the monitoring store must be **durable**, and its growth is yours to bound: the context appends an event per command, per projection round and per automation batch, forever. An in-memory monitoring store is for development only. Schedule a retention job, or roll to a fresh store periodically.
{: .prompt-warning }

### The Sliceworkz dashboard

The Sliceworkz dashboard is a separate project: a web dashboard embedded in a Javalin host, reading eventstore stores and eventmodeling monitoring streams. Handed the monitoring store, it adds its Applications screens — a live monitor, and screens per bounded context, command, read model, automation, translator, publisher, policy, dispatcher, port and instance, built from exactly the events above. Its `DashboardModule.monitoringListener(monitoringStore)` returns a `StreamAppendingBoundedContextListener` on the `monitoring/boundedcontext` stream it reads, so the listener above writes where it looks. Handed the same store as its management store, it adds stop, restart and skip buttons that append `ManagementInstruction`s. It carries no authentication of its own; protect its routes in the host application.

## The Observer: Scopes for Metrics and Tracing

A listener tells you what happened, after the fact. A metrics or tracing binding needs more: where an operation **starts**, where it **ends**, what it **answered**, and what ran **inside** it. That is `BoundedContextObserver`:

```java
public interface BoundedContextObserver {
    <O extends Outcome> Observation.Scope<O> start ( Observation<O> observation );
    default void listenerFailed ( String boundedContext, BoundedContextEvent event, Exception failure ) { }
}
```

The contract, for every operation:

1. `start(observation)` is called on the caller's thread, **before** the operation does anything;
2. the returned `Scope` is completed **exactly once**, with `completed(outcome)` or `failed(throwable)`;
3. `close()` is called in a `finally`, on the same thread.

Because all three happen on one thread, an observer can make a span current in `start` and end it in `close`, and everything the operation does in between — a live read model a command reads, the eventstore's own query, a port call — nests beneath it with nothing propagated. The duration is the observer's to measure, between `start` and `close`.

### What is observed

`Observation` is a sealed interface; each record names the operation and says which `Outcome` it completes with:

| observation | completes with |
|---|---|
| `CommandExecution` (with the caller's `Tracing`) | `Executed` (events raised per type, references appended), `Conflicted` or `Rejected` |
| `CommandEvaluation` — a preview, nothing appended | `Evaluated` |
| `ProvidedEvent`, `IncomingEvent` | `Provided`, `Done` |
| `Translation`, `TranslatorInvocation` | `Translated`, `Done` |
| `ReadModelBatch`, and a `ReadModelUpdate` per event inside it | `Projected` or `Cancelled`; `Done` |
| `LiveModelRead` | `LiveModelProjected` |
| `AggregateLoad`, `AggregateAppend` | `AggregateLoaded`; `Appended` or `Conflicted` |
| `SnapshotLoad`, `SnapshotSave` | `SnapshotFound` or `SnapshotMissed` (with the reason); `Done` |
| `AutomationRun` | `AutomationRan` (items streamed, handled, failed) |
| `PolicyReaction` | `Reacted`, `Ignored`, `AlreadyReacted`, `ReactionRejected` or `ReactionSkipped` |
| `Publication` | `Published` or `AlreadyPublished` |
| `Dispatch` | `Done` |
| `PortCall` | `PortReturned` or `PortRejected` |

**A completion is an answer, not only a success.** A DCB conflict and a business rejection complete — with `Conflicted` and `Rejected` — because they are answers the caller acts on. `failed` is kept for an operation that could not answer: a bug, a storage error, a poison event. It is the same split `CommandRejected` and `CommandFailed` make on the listener. Since `Observation` and `Outcome` are sealed, an observer switching with a `default` branch keeps compiling as operations are added.

Where an operation runs per stored event on a processor's thread (`ReadModelUpdate`, `Dispatch`, `Publication`, `PolicyReaction`), the observation carries the `Event`, and `Tracing.readFrom(event)` gives its flow. Where it runs for a caller, it carries the caller's `Tracing`.

### Binding Micrometer

The framework depends on no metrics library — an application binds the one it uses by writing an observer. A binding that times commands and read model batches with Micrometer:

```java
public class MicrometerObserver implements BoundedContextObserver {

    private static final Observation.Scope<Outcome> IGNORED = new Observation.Scope<>() {
        @Override public void completed ( Outcome outcome ) { }
        @Override public void failed ( Throwable failure ) { }
        @Override public void close ( ) { }
    };

    private final MeterRegistry registry;

    public MicrometerObserver ( MeterRegistry registry ) {
        this.registry = registry;
    }

    @Override
    @SuppressWarnings("unchecked")
    public <O extends Outcome> Observation.Scope<O> start ( Observation<O> observation ) {
        return switch ( observation ) {
            // command names come from the code, so they are a bounded tag
            case Observation.CommandExecution execution ->
                    (Observation.Scope<O>) timed("eventmodeling.command", "command", execution.command());
            case Observation.ReadModelBatch batch ->
                    (Observation.Scope<O>) timed("eventmodeling.readmodel.batch", "readmodel", batch.readModel());
            default -> (Observation.Scope<O>) IGNORED;
        };
    }

    private Observation.Scope<Outcome> timed ( String meter, String key, String name ) {
        Timer.Sample sample = Timer.start(registry);
        return new Observation.Scope<>() {
            private String outcome = "unknown";

            @Override
            public void completed ( Outcome answer ) {
                outcome = switch ( answer ) {
                    case Outcome.Executed executed   -> "executed";
                    case Outcome.Rejected rejected   -> "rejected";
                    case Outcome.Conflicted conflict -> "conflicted";
                    default                          -> "completed";
                };
            }

            @Override
            public void failed ( Throwable failure ) {
                outcome = "failed";
            }

            @Override
            public void close ( ) {
                sample.stop(registry.timer(meter, key, name, "outcome", outcome));
            }
        };
    }
}
```

Registered with `.observer(new MicrometerObserver(registry))`, two `OpenAccountCommand` executions produce:

```text
eventmodeling.command(TIMER)[command='OpenAccount', outcome='executed']; count=2.0, total_time=0.1688 seconds
```

A tracing binding has the same shape: start a span in `start`, make it current, record the outcome as an attribute, end it in `close`.

> **Cardinality is the observer's concern.** Names of commands, read models and event types are bounded by the code and make fine tags. `Tracing`s, aggregate identities and snapshot keys are data: an observer that turns them into metric tags must bound the values it admits, because a registry never evicts a meter. A tracer, on the other hand, wants them whole — which is why the framework hands both over unchanged and leaves the choice to the binding.
{: .prompt-warning }

### Observers never fail the work

The builder wraps the observer with `BoundedContextObserver.contained(...)`: whatever it throws — a `RuntimeException`, or a `LinkageError` from a binding whose library is missing at runtime — is logged (ERROR the first time, DEBUG after) and never reaches the operation. It still runs on the caller's thread, though, so **a slow observer is a slow context**. `BoundedContextObserver.NOOP` is the default and allocates nothing.

### Two layers, two observers

The bounded context reports what the framework adds; the eventstore it builds reports its own appends, queries, bookmarks, projector batches and erasures to an `EventStoreObserver`. By default that is the storage's own (what a storage builder's `.observer(...)` set); `.eventStoreObserver(...)` on the context builder takes precedence:

```java
Banking banking = BoundedContext.newBuilder(Banking.class)
        .name("banking")
        .eventStorage(storage)
        .observer(new TimingObserver())              // commands, read models, automations, ...
        .eventStoreObserver(storeObserver)           // appends, queries, projector batches, ...
        // ...
        .build();
```

Because both are called on the same thread, the eventstore's query observation nests inside the framework's `CommandExecution`. The eventstore's projector batches are named after the processor they belong to — the read model, translator or dispatcher — rather than an internal adapter class.

### Testing what is observed

The testing module publishes `RecordingBoundedContextObserver`: it records every scope with its parent, its outcome or failure and its thread, and `violations()` lists any breach of the scope contract. See [Testing Feature Slices](/posts/eventmodeling-testing/).

## Tracing: One Flow, One Correlation Id

### What every event carries

Every event a bounded context appends — domain, inbound, outbound and the monitoring events alike — carries the tracing of the work that produced it, as tags:

| tag | from |
|---|---|
| `x-instance-logical`, `x-instance-physical`, `x-instance-process` | the instance the context runs as |
| `x-actor` | who: a user, an agent's owner, an automation's name, `system` |
| `x-channel` | through what: `api`, `batch-import`, `automation`, ... |
| `x-command` | the command that raised it |
| `x-agent-id`, `x-agent-name` | an agent acting on someone's behalf |
| `x-correlation-id` | the flow it belongs to |
| `x-caused-by` | the event a policy reacted to |

`Tracing` is the record behind these tags, and `Tracing.readFrom(event)` reads it back off a stored event. Tag matching is containment, so the extra tags change no query and no consistency boundary.

### Handing a tracing to the work

Every capability that writes or reads has an overload taking a `Tracing`. Build it with a factory, which mints a fresh correlation id, and refine it with the copying methods:

```java
Tracing tracing = Tracing.actorAndChannel("bob@bank", "batch-import");
app.execute(new OpenAccountCommand(CUSTOMER.newId()), tracing);
```

`correlationId(id)`, `actor(...)`, `channel(...)` and `agent(id, name)` return a copy with one field changed. Only the factories (`init`, `actorAndChannel`, `automation`, `kernel`) mint a correlation id; copying a tracing or reading one back from an event reproduces the id exactly, so a flow is never silently split. An overload called without a tracing gets one minted for it, so every flow has an id from its first event.

### One request, one flow: `TracingScope`

An HTTP request typically checks access through a port, reads a read model and executes a command or two. Passing a tracing to each call is easy to forget, and every forgotten one starts a flow of its own. Instead, bind the request's tracing to the thread where the request enters, once:

```java
try ( TracingScope scope = TracingScope.bind(
        Tracing.actorAndChannel("alice@bank", "api").correlationId("req-42")) ) {
    app.execute(new RegisterCustomerCommand(alice, "Alice Martin", new Address("Main Street 1", "Ghent")));
    app.execute(new OpenAccountCommand(alice));
}
```

While it is bound, every capability called **without** a tracing on that thread uses it — `execute`, `read`, `evaluate`, `aggregate`, `event`, `incoming`, `translate` — and so does a call through a monitored port that has no component tracing of its own (a REST endpoint's membership check). An explicit tracing always wins. Take the correlation id from an incoming header where the caller sends one, and echo it in the response.

A scope belongs to its thread: `close()` restores what was bound before (scopes nest), is idempotent, and does nothing on another thread. Nothing is inherited by other threads — processor threads carry the tracing of the events they handle instead.

### Across processors

The flow continues by itself through most hops:

- **Translators** continue the flow of the inbound event they translate (a fresh id when it carries none).
- **Publishers** carry the source event's correlation id onto the outbound events, and tag them `x-published-from:<event id>`.
- **Policies** carry the triggering event's correlation id onto the command they issue, and tag what it raises `x-caused-by:<event id>`.
- **Monitoring events** carry the correlation id of the operation they report on, so a `CommandExecuted` is found under the same id as the domain events it raised.

**Automations are the one hop the framework cannot bridge alone**: a to-do item is your own type, projected out of history, and nothing links it to the event that put it on the list. `CorrelatedTodoItem` is that link. The to-do list reads the id off the triggering event in `when`, and the item carries it:

```java
public record CustomerToWelcome ( CustomerId customerId, String correlationId ) implements CorrelatedTodoItem { }

@Override
public synchronized void when ( Event<BankingDomainEvent> event ) {
    switch ( event.data() ) {
        case CustomerRegistered registered -> outstanding.put(registered.customerId().value(),
                new CustomerToWelcome(registered.customerId(), Tracing.readFrom(event).correlationId()));
        case CustomerWelcomed welcomed -> outstanding.remove(welcomed.customerId().value());
        default -> { }
    }
    lastEventReference = event.reference();
}
```

`handle` and `onFailure` are then given a context whose tracing carries the item's id, per item, so what the automation raises joins the flow without the automation doing anything:

```java
@Override
public Optional<EventReference> handle ( CustomerToWelcome item,
                                         AutomationContext<BankingDomainEvent, BankingOutboundEvent> context ) {
    // the context's tracing already carries item.correlationId(): what this raises joins the flow
    return context.execute(new WelcomeCustomerCommand(item.customerId()), "welcome/" + item.customerId().value());
}
```

An item that does not implement it keeps the batch's own tracing, whose correlation id names the automation run.

### Reading a whole flow

"Show me everything this request caused" is a tag query. Application code reaches its events through read models, but tooling — a support screen, a dashboard, a test — can ask the store directly, across every stream:

```java
EventStore store = EventStore.on(storage).build();
EventSource<String> everything = store.getRawEventStream(EventStreamId.anyContext());

List<Event<String>> flow = everything.query(EventQuery.forEvents(EventTypesFilter.any(),
        Tags.of(Tracing.TAG_CORRELATION_ID, "req-42")));
```

For the request above, that returned the two events the request appended and the one the automation raised for it:

```text
CustomerRegistered  x-actor:alice@bank                 x-channel:api         x-correlation-id:req-42
CustomerWelcomed    x-actor:WelcomeCustomerAutomation  x-channel:automation  x-correlation-id:req-42
AccountOpened       x-actor:alice@bank                 x-channel:api         x-correlation-id:req-42
```

Run the same query on the monitoring stream and you get the `CommandExecuted`s, the `AutomationProcessed` and the port calls of that flow beside them. The raw stream reads payloads as the JSON they are stored as, so nothing has to deserialize, and a protected value shows as its sealed envelope (see [Querying Events](/posts/eventstore-querying-events/)).

## Related

- [Running in Production](/posts/eventmodeling-running-in-production/) — failure handling, the operator surface and the management stream
- [Ports, Adapters and Monitored Calls](/posts/eventmodeling-ports-and-adapters/) — the port events and summarized ports
- [Automations and Policies](/posts/eventmodeling-automations-and-policies/) — `CorrelatedTodoItem` in context
- [Observability with Micrometer](/posts/eventstore-observability-micrometer-prometheus-grafana/) — the eventstore's own observer
- [Listeners](/posts/eventstore-listeners/) — the eventstore's append and bookmark listeners, which are a different thing
