---
layout: post
toc: true
title: Running in Production
description: PostgreSQL storage, several instances of one deployment, leader election, splitting front and batch instances, failure handling and the operator's controls.
date: 2026-10-11 13:00:00
categories: [Eventmodeling Documentation,Eventmodeling Deployment]
tags: [eventmodeling, production, postgresql, leader election, high availability, operations, management]
---

The [quickstart](/posts/eventmodeling-quickstart/) runs a bounded context in one process on in-memory storage. This guide is about the other end: a bounded context on PostgreSQL, deployed as several instances, split into instances that serve requests and instances that do the background work, and kept running by an operator who can see what each processor is doing and stop or restart it without restarting anything else.

Nothing in your feature slices changes for any of this. Commands, read models, automations and policies are written once; where they run is decided by the builder that assembles the context.

## Storage: PostgreSQL

A bounded context keeps its events in an `EventStorage` you build and hand it. In production that is the PostgreSQL backend:

```java
HikariDataSource dataSource = new HikariDataSource();
dataSource.setJdbcUrl(System.getenv("DB_URL"));
dataSource.setUsername(System.getenv("DB_USERNAME"));
dataSource.setPassword(System.getenv("DB_PASSWORD"));

EventStorage storage = PostgresEventStorage.newBuilder()
        .name("banking")
        .dataSource(dataSource)
        .prefix("banking_")
        .build();

Banking banking = BoundedContext.newBuilder(Banking.class)
        .name("banking")
        .eventStorage(storage)
        .instance(InstanceFactory.determine("banking", System.getenv("HOSTNAME")))
        .shredding(PostgresShreddingKeyStore.on(dataSource, "banking_"))
        .features()
            .rootPackage(Banking.class.getPackage())
            .done()
        .build();

banking.start();
```

Everything about the storage itself — the schema modes, `db.properties`, table prefixes, the separate monitoring `DataSource` that LISTEN/NOTIFY needs when the main one goes through a transaction pooler, backups — is the eventstore's and documented there: see [Configuring PostgreSQL Storage](/posts/eventstore-configuring-postgresql-storage/) and [Lifecycle](/posts/eventstore-lifecycle/). The `shredding(...)` line is only needed when your events carry personal data; see [Personal Data and Erasure](/posts/eventmodeling-personal-data/).

Add the backend next to the framework; the eventmodeling BOM manages its version:

```xml
<dependency>
    <groupId>org.sliceworkz</groupId>
    <artifactId>sliceworkz-eventstore-infra-postgres</artifactId>
</dependency>
```

### Who closes what

The bounded context builds an `EventStore` of its own over the storage, and that store is the only thing it closes. The storage stays yours: one storage can back several bounded contexts, and it usually outlives them.

```java
banking.terminate();   // closes the EventStore the context built; storage untouched
storage.close();       // ours, closed once no context runs on it
dataSource.close();    // supplied by us, so closed after the storage
```

Close in that order. A storage closed before its contexts are terminated makes every operation of those contexts fail with `EventStorageClosedException`, and a `DataSource` closed before the storage leaves the storage's notification monitors retrying against a dead pool, which they cannot tell apart from a database outage.

A bounded context also registers a **JVM shutdown hook** that terminates it, so a process that simply exits leaves no processor thread behind. Calling `terminate()` yourself deregisters that hook, which is what an application that outlives its context (a test, a hot reload, a per-tenant context) must do so the context can be collected. The storage gets no such hook: close it in your own shutdown path.

### Naming an instance

`InstanceFactory.determine(logical, physical)` names the instance a context runs as:

| part | what it is | example |
|---|---|---|
| `logical` | the deployment — the same on every instance of it | `"banking"` |
| `physical` | this instance, **stable across restarts** | the pod or host name; `"default"` when you pass none |
| `process` | this JVM run, minted for you | host, pid, start time and a random suffix |

All three are written on every event the context appends (as `x-instance-*` tags, see [Observability and Tracing](/posts/eventmodeling-observability/)) and they address an instance from the management stream (below). The physical name matters for one more thing: a `LOCAL` read model's bookmark is keyed by `logical#physical`, so two instances sharing a physical name would share one bookmark for what are two private copies. Give every instance its own, and keep it across restarts so a `LOCAL` read model resumes rather than replays.

## Several Instances of One Deployment

Run the same deployment twice and every instance builds the same bounded context with the same processors. Some of them must run on every instance; some must run on exactly one. The framework decides which from what each processor is, never from configuration:

| processor | runs on | why |
|---|---|---|
| `EPHEMERAL` read model (in memory, the default) | **every** instance | each instance holds its own copy and must project it |
| `LOCAL` read model (durable, private to one instance) | **every** instance | same, resuming from its own bookmark |
| `SHARED` read model (durable, shared by the deployment) | **one** elected leader | one copy, written once |
| automation | **one** elected leader | its work must happen once |
| policy | **one** elected leader | its command must be issued once |
| translator | **one** elected leader | its domain events must be raised once |
| publisher | **one** elected leader | its outbound events must be appended once |
| dispatcher | **one** elected leader | sending to an external system twice is the worst outcome there is |

Commands and live read models are not processors at all: they run on whichever instance a request arrives at, and need no coordination beyond the consistency boundary every command already checks (see [Commands and Decision Models](/posts/eventmodeling-commands-and-decision-models/)).

The storage class is a property of the read model — `ReadModel.storage()`, `EPHEMERAL` unless it says otherwise — because it follows from where the read model writes. That is also why "in-memory state projected on one leader only" cannot be configured: every other instance would answer from an empty model. See [Choosing a Read Model](/posts/eventmodeling-read-models/) and [Durable SQL Read Models](/posts/eventmodeling-sql-read-models/).

> `start()` does not return until every `EPHEMERAL` read model has been projected completely, so they are usable right after it. `LOCAL` and `SHARED` read models resume from their bookmarks and catch up in the background. The wait has a safety timeout of 5 minutes, after which startup continues with a warning; change it with `-Dsliceworkz.eventmodeling.readmodel.ephemeral.projection.timeout.ms=...`.
{: .prompt-info }

## Leader Election

Every leader-only processor holds a **lease** in the event storage, named after the processor — the same name that keys its bookmark, so each lease guards exactly the writer of that bookmark. One processor, one lease: an instance contends only for the processors it deploys, so a deployment that puts one automation on one set of instances and another automation elsewhere has each set win the lease nobody else wants. The lease mechanics themselves — expiry on the storage's clock, fencing tokens, step-down — are the eventstore's, described in [Leader Election with Leases](/posts/eventstore-leader-election/); here is what the framework does with them.

One elector per bounded context renews every lease the instance holds on a heartbeat, on a virtual thread of its own. **Election is never on the processing path**: a processor reads whether it is the leader at the top of every loop, and no batch, projection or query ever waits for a lease call. A leader projects continuously across renewals, so election adds nothing to a read model's lag. On PostgreSQL the lease tables sit outside the event log, so election traffic takes no lock an append or a query takes.

`start()` runs one election round before any processor's first pass, and `stop()` and `terminate()` release the leases held, so a standby takes over within a heartbeat rather than waiting out the time-to-live.

### Priority and pacing

Two builder settings tune it:

```java
Banking banking = BoundedContext.newBuilder(Banking.class)
        .name("banking")
        .eventStorage(storage)
        .instance(instance)
        .leadershipPriority(10)
        .leadershipIntervals(Duration.ofSeconds(5), Duration.ofSeconds(20))
        // ...
        .build();
```

| setting | default | meaning |
|---|---|---|
| `leadershipPriority(long)` | `0` | when a live contender with a **strictly higher** priority appears, the leader finishes its batch and hands the lease over — the way a preferred instance (the one in the primary zone, the bigger machine) regains leadership when it comes back. Equal priorities never preempt, which keeps a symmetric deployment stable |
| `leadershipIntervals(heartbeat, ttl)` | 5 s, 20 s | the leader renews every `heartbeat`; a lease not renewed within `ttl` expires and is taken over. `ttl` must be at least twice the heartbeat, so a single failed renewal does not cost leadership |

`ttl` is what a crash costs: an instance that dies without releasing its leases holds the work off for up to that long. Nothing is lost in that pause — the work is in the event store and the to-do lists — it just waits. A graceful stop or a step-down hands over within a heartbeat or two.

### Demote before takeover

The rule that keeps two leaders from running at once: a challenger may take a lease only once it has expired on the storage's clock, at `ttl`; a leader that cannot **confirm** a renewal demotes itself at `ttl - heartbeat` after the last confirmed one, judged on its own clock. Demoting is local — a flag on the processor — so it never waits for the storage. Every lease call is bounded, a whole round finishes within a heartbeat whatever the storage does, and a request that stalls (an unreachable database, a socket that died without closing) stays in flight once rather than being issued again. A leader that loses its database therefore steps down on time instead of holding on past the moment someone else may take over.

### A processor that stops itself, or keeps failing

A leader that does no work must not hold the lease, or the work runs nowhere in the deployment. So:

- a processor that **stops itself** — a projector retired by a permanent failure, an automation whose `onFailure` answered `STOP_AUTOMATION`, a processor an operator stopped — hands its lease back and is not contended for on this instance until it is started again;
- a projector-driven leader (a `SHARED` read model, translator, publisher, dispatcher or policy) that **keeps failing** — three consecutive failed runs — yields its lease and sits out the election for one `ttl`. Its trouble (the read model's database unreachable from this host) says nothing about the standby's, so the standby gets a chance; with nobody to yield to, the same instance takes the lease back and keeps retrying.

Both show up as `LeadershipReleased` with a reason (`PROCESSOR_STOPPED`, `PROCESSOR_FAILING`, and `STEPPED_DOWN`, `LOST`, `RENEWAL_FAILED`, `STOPPED` for the other paths). A promoted projector rebuilds its position from the durable bookmark rather than from the cursor it held while it was a standby, so it resumes where the previous leader got to.

### What it does not promise

Leader election gives **one processor at a time, not exactly-once through a failover**. A leader paused past its time-to-live — a long GC, a frozen VM — can finish a batch it had already started while the new leader begins. No lease can prevent that window; what contains it is what the framework already asks for:

- **idempotency keys** on what automations and policies raise, derived from the to-do item or the triggering event, never from the attempt — a repeat is swallowed by the store (see [Automations and Policies](/posts/eventmodeling-automations-and-policies/));
- **publishers** key their outbound events by the domain event they publish, so the first publication stands (see [Translators, Publishers and Dispatchers](/posts/eventmodeling-translators-publishers-dispatchers/));
- **DCB conflicts**: a command executed twice on stale facts conflicts rather than deciding twice;
- for SQL read models, `updateOnce`/`insertOnce` and the **fencing token**: a `SHARED` SQL read model stores the lease's token in its bookmark row, raises it before reading where to resume, and compares it inside every batch transaction, so a superseded leader's commit is refused whole. What remains open is a zombie batch committed before the new leader is promoted. See [Durable SQL Read Models](/posts/eventmodeling-sql-read-models/).

A dispatcher is the one processor whose repeat no store can swallow — the external system received it. Make the receiving side idempotent on the outbound event's id where it can be.

### A storage without leases

The in-memory and PostgreSQL storages implement leases (in memory, a single process simply wins everything). A third-party `EventStorage` that does not support them makes the context promote every leader-only processor on this instance, with one WARN saying that a second instance would duplicate the work. One instance on such a storage is fine; two are not.

## Splitting Front and Batch Instances

A slice has four **aspects**, and each has its own `configure...` method on `Slice`:

| aspect | configured in | what it registers |
|---|---|---|
| `COMMAND` | `configureCommand` | the commands it declares |
| `QUERY` | `configureQuery` | the read models requests read |
| `AUTOMATION` | `configureAutomation` | automations and their to-do lists, policies, publishers, dispatchers |
| `PROJECTION` | `configureProjection` | the eventually consistent read models kept up to date in the background |

The `features()` sub-builder says which aspects an instance runs. An aspect that is disabled is not configured at all — its `configure...` method and its `start...` method are not called — so whatever it would register does not exist on that instance. All four are enabled by default. Reading the role from the environment gives one artifact with two deployments:

```java
String role = System.getenv().getOrDefault("BANKING_ROLE", "all");   // front | batch | all
boolean front = !role.equals("batch");
boolean batch = !role.equals("front");

Banking banking = BoundedContext.newBuilder(Banking.class)
        .name("banking")
        .eventStorage(storage)
        .instance(InstanceFactory.determine("banking", System.getenv("HOSTNAME")))
        .shredding(PostgresShreddingKeyStore.on(dataSource, "banking_"))
        .features()
            .rootPackage(Banking.class.getPackage())
            .enableCommands(front)
            .enableQueries(front)
            .enableAutomations(batch)
            .enableProjections(batch)
            .done()
        .build();
```

There are also argument-less `enableX()` and `disableX()` methods for each aspect. Front instances scale with traffic and carry no leases; batch instances run the processors and elect among themselves. Where each kind of registration belongs follows from that:

- An **`EPHEMERAL` read model a request reads** belongs in `configureQuery`: every instance that serves the read needs its own copy, so it must be projected where queries run.
- A **`SHARED` SQL read model** belongs in `configureProjection`: one batch instance projects it, and the front instances read the shared database through their own query code.
- An **automation registers its to-do list next to it**, in `configureAutomation`. `build()` refuses an automation whose `EPHEMERAL` or `LOCAL` to-do list is not projected on the same instance, naming it — otherwise it would wait for a bookmark nobody writes.
- **Publishers and policies must be registered from `configureAutomation`**; `build()` refuses one registered from another aspect. A publisher registers the live read models it reads beside it, since a front-only `configureQuery` registration does not exist on a batch instance.
- A component registered **directly on the builder** rather than from a slice belongs to no aspect and runs wherever that builder runs.

Every instance announces which slices and which aspects it deployed in its `BoundedContextStarting` event (see [Observability and Tracing](/posts/eventmodeling-observability/)), so a dashboard can show what runs where.

A slice-level `filter(...)` on the same sub-builder narrows which slices an instance deploys at all — by name, chapter, class or package — for a deployment that runs part of a context on dedicated hardware.

## When Things Fail

The framework sorts failures into ones retrying can fix and ones it cannot, and reacts to each the same way on every instance.

### Projectors retry with backoff

Read model projectors, translators, publishers, dispatchers and policies all run on the eventstore's `Projector`. A failure inside a batch rolls the cursor back to the start of that batch and places no bookmark, so a retry re-offers exactly those events — retrying never skips anything. The framework therefore retries by default:

| | default | override |
|---|---|---|
| first retry after | 10 s | `-Dsliceworkz.eventmodeling.projector.retry.initial.ms=...` |
| doubling up to | 5 min | `-Dsliceworkz.eventmodeling.projector.retry.max.ms=...` |

The wait is not cut short by new events arriving, so a failing read model retries at its backoff pace rather than at the pace of the traffic feeding it. Each fruitless round is reported as a `...Failed` event (`ReadModelProjectorFailed`, `DispatcherFailed`, ...) carrying `consecutiveFailedRuns`, so you choose your own alerting threshold. There is no "recovered" event: recovery is the next ordinary update.

A small, closed list of causes cannot be fixed by retrying and **retires** the processor instead (`...Stopped`, reason with the failure):

| cause | why retrying cannot help |
|---|---|
| `EventDeserializationException` | a poison event: the payload cannot be read with these mappings (see [Error Handling](/posts/eventstore-error-handling/)) |
| `EventSerializationException` | a payload cannot be written |
| `EventStorageClosedException` | the storage is gone |
| `StaleLeadershipException` | a newer leader's fencing token is stored, and it only grows |
| `IdempotencyKeyConflictException` | a batch mixing stored and new keys is refused identically on every attempt |

Everything else — your projection's own exceptions included — is retried. Between a dead database and a bug the framework cannot tell, and of the two ways to be wrong, retrying a bug is a visible stall where stopping on an outage would be a read model that never comes back. A retired processor is restarted from the operator surface below once the cause is fixed.

### Automations back off

An automation contains failures per to-do item, and what a failing item costs is its own decision through `onFailure` (see [Automations and Policies](/posts/eventmodeling-automations-and-policies/)). When batches keep failing without handling anything, the default `delayBeforeNextBatch` waits 10 s and doubles up to 5 minutes, so a dependency that is down for an hour costs a handful of attempts. `AutomationFailed` is reported once per such batch, with `consecutiveFailedBatches`.

### A policy stalls

A policy has no to-do list to put an event aside in, so a failing reaction **stalls** it: the event is retried with backoff while every event behind it waits, reported as `PolicyFailed` naming the event. The operator's way out is `skipStalledEvent`, below.

## Operating a Running Context

### The operator's surface: `OperationsCapabilities`

An operations tool — an admin endpoint, a script, a dashboard — holds `OperationsCapabilities` and nothing else. A built context already is one, so narrowing costs a reference type (see [Who May Do What](/posts/eventmodeling-who-may-do-what/)):

```java
public class AdminEndpoint {

    private final OperationsCapabilities ops;

    public AdminEndpoint ( OperationsCapabilities ops ) {
        this.ops = ops;
    }

    public void report ( ) {
        for ( AutomationStatus automation : ops.automations() ) {
            System.out.printf("%s running=%s leader=%s failedBatches=%d stoppedBy=%s%n",
                    automation.automation(), automation.running(), automation.leader(),
                    automation.consecutiveFailedBatches(), automation.stoppedBy());
        }
        for ( ProcessorStatus processor : ops.processors() ) {
            System.out.printf("%s %s [%s] running=%s leader=%s failedRuns=%d%n",
                    processor.kind(), processor.name(), processor.storage(), processor.running(),
                    processor.leader(), processor.consecutiveFailedRuns());
        }
    }
}

AdminEndpoint admin = new AdminEndpoint(banking);
```

It has no `read` and no `execute`: an operator stops and starts things, and decides nothing in the domain.

| method | does |
|---|---|
| `automations()` | an `AutomationStatus` per automation: running, leader, items failed, consecutive failed batches, `lastFailure`, `stoppedBy` |
| `stopAutomation(id)` / `restartAutomation(id)` | park an automation after the item in hand / put it back |
| `processors()` | a `ProcessorStatus` per read model projector, translator, dispatcher, publisher and policy: kind, name, storage, running, leader, consecutive failed runs, `lastFailure`, `stoppedBy` |
| `stopProcessor(kind, name)` / `restartProcessor(kind, name)` | the same for a processor (`ProcessorKind.READ_MODEL`, `TRANSLATOR`, `DISPATCHER`, `PUBLISHER`, `POLICY`) |
| `skipStalledEvent(POLICY, name, eventId)` | move a stalled policy past the event it is stuck on, without reacting to it |

`running` says the processor would work if elected; `leader` says it actually is the one working on this instance. `lastFailure` and `stoppedBy` are kept apart because a healthy processor has usually survived failures, and the one an operator wants is the one that stopped it.

The calls are safe to repeat: stopping what is stopped or restarting what runs answers `false`, and a name that is not registered is an `IllegalArgumentException` naming the ones that are. What they are for:

```java
// rebuilding a read model: stop its projector everywhere, drop its tables, start it again
ops.stopProcessor(ProcessorKind.READ_MODEL, "AccountBalancesProjector");
// ... drop its tables ...
ops.restartProcessor(ProcessorKind.READ_MODEL, "AccountBalancesProjector");

// the payment gateway is down for maintenance
ops.stopAutomation("ExecutePaymentAutomation");
// ...
ops.restartAutomation("ExecutePaymentAutomation");

// a policy is stalled on an event nobody should react to after all (PolicyFailed names it)
ops.skipStalledEvent(ProcessorKind.POLICY, "BlockCardsOfAccountPolicy", stalledOn);
```

Stopping a **leader-only** processor or automation hands its lease back, so another instance takes the work over; stopping it for the whole deployment means stopping it on every instance. A batch in progress finishes (or rolls back whole) before the processor parks, and its position is untouched, so a restart resumes where it durably left off. Restarting a processor retired by a permanent failure without fixing the cause replays the same batch and stops again — read `stoppedBy` first.

> Every method addresses **the instance it is called on**. Each instance runs its own processors, and a standby's automation is parked whatever you call on it. Reaching the other instances is the management stream's job.
{: .prompt-info }

### Reaching every instance: the management stream

An operator outside the process reaches the instances through the event store, which every instance can already reach. Each context subscribes to a stream of `ManagementInstruction`s with `.management(stream)`; an operator appends an instruction; every instance reads it; the ones it targets apply it through their own `OperationsCapabilities` and answer.

```java
EventStore monitoring = EventStore.on(monitoringStorage).build();

EventStream<BoundedContextEvent> reports = monitoring.getEventStream(
        EventStreamId.forContext("monitoring").withPurpose("boundedcontext"), BoundedContextEvent.class);
EventStream<ManagementInstruction> instructions = monitoring.getEventStream(
        ManagementInstruction.STREAM, ManagementInstruction.class);

Banking banking = BoundedContext.newBuilder(Banking.class)
        .name("banking")
        .eventStorage(storage)
        .instance(instance)
        .listener(new StreamAppendingBoundedContextListener(reports))
        .management(instructions)
        // ...
        .build();
```

Hand each bounded context a stream of its own (open it per context, as above): `terminate()` closes the stream it was given. The instructions are records of the sealed `ManagementInstruction`, each with a `Target`:

| instruction | applies |
|---|---|
| `StopAutomation` / `StartAutomation` | `stopAutomation` / `restartAutomation` |
| `StopProcessor` / `StartProcessor` | `stopProcessor` / `restartProcessor` |
| `SkipStalledEvent` | `skipStalledEvent` |
| `StopBoundedContext` / `StartBoundedContext` | `stop()` / `start()` of the context itself |
| `ReportStatus` | answers with an `InstanceStatusReported` carrying the same status lists |

| `Target` | reaches |
|---|---|
| `Target.any()` | every instance of every context |
| `Target.boundedContext("banking")` | every instance of one context |
| `Target.instance("banking", logical, physical)` | one deployed instance |
| `Target.process("banking", logical, physical, process)` | one JVM run |

An operator — a script, or the Sliceworkz dashboard — appends:

```java
instructions.append(Event.of(new StopProcessor(
        Target.boundedContext("banking"), ProcessorKind.DISPATCHER, "AccountAnnouncedDispatcher"), Tags.none()));
instructions.append(Event.of(new StartProcessor(
        Target.instance("banking", "banking", "pod-2"), ProcessorKind.DISPATCHER, "AccountAnnouncedDispatcher"), Tags.none()));
instructions.append(Event.of(new ReportStatus(Target.any()), Tags.none()));
```

Each targeted instance answers through its listener with an `InstructionHandled` — so the answer lands on the monitoring stream next to everything else the instance reports — with one of four outcomes:

| outcome | meaning |
|---|---|
| `APPLIED` | done |
| `NO_CHANGE` | already stopped, already running |
| `REJECTED` | this instance has no such automation or processor; the detail names what it has |
| `FAILED` | applying it threw; contained, so one bad instruction never ends the subscription |

Run against the in-memory storage, stopping the welcome automation of the scratch project answered:

```text
InstructionHandled[boundedContext=banking, instruction=StopAutomation, subject=WelcomeCustomerAutomation, outcome=APPLIED, detail=null]
```

Three properties to know:

- **Read from the head, never replayed.** A context reads the stream from its head at `start()`, without a bookmark: an instruction is something an operator did, not a standing rule, and an instance coming up must not replay last week's stops. An instruction issued while an instance was down is not seen by it — visible as the answer that never comes. Press it again.
- **Applied one at a time, in order,** on a thread of the context's own, so a slow instruction (starting a context waits for its ephemeral read models) never blocks the store's notifications. The stream keeps being read while the context is stopped — that is what lets `StartBoundedContext` bring it back — and is released by `terminate()`.
- **Whoever can append to the stream controls the processors.** Nothing authenticates an instruction beyond the store having accepted it. Put the stream on a store with matching protection — usually the monitoring store — and protect whatever appends to it as you would a shell on the instance. Every instruction carries the tracing tags of whoever appended it, so the stream is also the record of who did what.

> The Sliceworkz dashboard mounts its operator buttons (stop, restart, skip a stalled event, per instance or per context) only when it is handed the management store, and carries no authentication of its own: protect its routes in the host application before you do.
{: .prompt-warning }

## Threads, and What an Idle Processor Costs

Every processor runs on a **virtual thread**, created by `start()` rather than by `build()`, so a context that was built and never started holds no thread. An idle processor parks on a `java.util.concurrent` lock condition rather than in a monitor wait, which unmounts the virtual thread: on Java 21 a parked processor holds no platform thread, so a deployment with hundreds of read models and automations across its contexts costs a handful of platform threads, not one each.

Two consequences for sizing: count processors freely, and look for the load in what they do — the queries a projector runs, the calls an automation makes — rather than in how many there are. A processor that is caught up costs nothing until an append wakes it.

## A Checklist

- Build the storage, hand it to the context, and close it yourself after `terminate()` — then whatever you gave the storage.
- Name every instance with a stable, distinct physical name.
- Put `EPHEMERAL` read models a request reads in `configureQuery`; `SHARED` ones in `configureProjection`; automations, policies, publishers and dispatchers in `configureAutomation`.
- Derive idempotency keys from the item or the triggering event; make external receivers idempotent where they can be.
- Register a listener writing to a durable monitoring store, and alert on `...Failed` and `...Stopped` events (see [Observability and Tracing](/posts/eventmodeling-observability/)).
- Give operators `OperationsCapabilities`, and the management stream for reaching other instances — on a store only they can append to.

## Related

- [Bounded Contexts and Feature Slices](/posts/eventmodeling-bounded-contexts-and-feature-slices/) — the builder, slices and aspects
- [Durable SQL Read Models](/posts/eventmodeling-sql-read-models/) — `SHARED` read models, their own position and the fencing token
- [Automations and Policies](/posts/eventmodeling-automations-and-policies/) — `onFailure`, pacing and idempotency keys
- [Observability and Tracing](/posts/eventmodeling-observability/) — what every instance reports, and how to read it
- [Leader Election with Leases](/posts/eventstore-leader-election/) — the lease primitive underneath
- [Configuring PostgreSQL Storage](/posts/eventstore-configuring-postgresql-storage/) and [Lifecycle](/posts/eventstore-lifecycle/) — the storage itself
- The banking and payments examples: [sliceworkz-eventmodeling-examples](https://github.com/sliceworkz/eventmodeling/tree/develop/sliceworkz-eventmodeling-examples)
