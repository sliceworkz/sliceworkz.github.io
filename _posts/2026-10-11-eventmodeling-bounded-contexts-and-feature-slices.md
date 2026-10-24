---
layout: post
toc: true
title: Bounded Contexts and Feature Slices
description: How a bounded context is declared, built from feature slices, deployed in parts, started and shut down
date: 2026-10-11 01:00:00
categories: [Eventmodeling Documentation,Eventmodeling Guides]
tags: [eventmodeling, bounded context, feature slice, builder, deployment, lifecycle]
---

An application on the Sliceworkz EventModeling framework is a set of **bounded contexts**, and each bounded context is a set of **feature slices**: one vertical slice per command, read model, automation or translation of your event model. This guide covers how you declare a context, how slices are discovered and what they register, how one context is split over several processes, what `build()` refuses and why, and how a context is shut down.

The examples use the banking domain from the [quickstart](/posts/eventmodeling-quickstart/). The reference code in the repository is the [banking example](https://github.com/sliceworkz/eventmodeling/tree/develop/sliceworkz-eventmodeling-examples/src/main/java/org/sliceworkz/eventmodeling/examples/banking).

## The Bounded Context Interface

A bounded context is an interface you declare, extending `BoundedContext<D,I,O>` with your three event roots:

```java
public interface BankingDomain {

    record AccountId ( String value ) implements EntityId { }
    record CustomerId ( String value ) implements EntityId { }

    Entity<AccountId>  ACCOUNT  = Entity.of("account", AccountId::new);
    Entity<CustomerId> CUSTOMER = Entity.of("customer", CustomerId::new);

    // the facts this context records
    sealed interface BankingDomainEvent {
        record AccountOpened ( AccountId accountId, CustomerId customerId, LocalDate date ) implements BankingDomainEvent { }
        record MoneyDeposited ( AccountId accountId, BigDecimal amount ) implements BankingDomainEvent { }
    }

    // what other systems tell this context
    sealed interface BankingInboundEvent {
        record CustomerSuspended ( CustomerId customerId ) implements BankingInboundEvent { }
    }

    // what this context publishes to other systems
    sealed interface BankingOutboundEvent {
        record AccountAnnounced ( AccountId accountId ) implements BankingOutboundEvent { }
    }
}

public interface BankingApi extends ApplicationCapabilities<BankingDomainEvent, BankingOutboundEvent> { }

public interface Banking extends BoundedContext<BankingDomainEvent, BankingInboundEvent, BankingOutboundEvent>, BankingApi { }
```

The three type parameters are the three kinds of event a context deals with:

| Type | Role | Written by |
|---|---|---|
| **Domain events** (`D`) | The facts this context records about its own domain | Commands, aggregates, automations, policies, translators |
| **Inbound events** (`I`) | Integration events received from other systems | The inbound edge (`incoming(...)`), turned into domain events by translators |
| **Outbound events** (`O`) | Integration events this context publishes | Publishers only, sent on by dispatchers |

Each root is a sealed interface of records — see [Defining Events](/posts/eventstore-defining-events/) for why, and for how a record's simple name becomes its stored type name. The `Entity` declarations are covered in [Entities, Typed Ids and Tags](/posts/eventmodeling-entities-and-tags/), and `BankingApi` — the narrow surface application code holds instead of `Banking` — in [Who May Do What](/posts/eventmodeling-who-may-do-what/).

The interface `Banking` exists so that nobody has to spell out `BoundedContext<BankingDomainEvent, BankingInboundEvent, BankingOutboundEvent>` again. It is also where the framework reads your event types from: `BoundedContext.newBuilder(Banking.class)` resolves the three roots off the interface's type arguments, and `build()` hands you a proxy that implements exactly that interface.

## Building a Context

```java
EventStorage storage = InMemoryEventStorage.newBuilder().build();

Banking banking = BoundedContext.newBuilder(Banking.class)
    .name("banking")
    .eventStorage(storage)
    .instance(InstanceFactory.determine("banking-app"))
    .listener(new LoggingBoundedContextListener())
    .adapter(routes).forPort(HttpRoutes.class)
    .features()
        .rootPackage(Banking.class.getPackage())
        .done()
    .build();

banking.start();
```

| Setting | Required | What it does |
|---|---|---|
| `name(...)` | yes | The context's name. It names the streams the context writes to (below), so treat it as stored data |
| `eventStorage(...)` | yes | The [`EventStorage`](/posts/eventstore-quickstart/) to run on. The context builds its own `EventStore` over it |
| `instance(...)` | yes | Which deployment this process is: `InstanceFactory.determine(logical)` or `determine(logical, physical)` |
| `listener(...)` | no | Receives every `BoundedContextEvent` the kernel produces. `LoggingBoundedContextListener` logs them; `StreamAppendingBoundedContextListener` appends them to a stream for a dashboard. See [Observability and Tracing](/posts/eventmodeling-observability/) |
| `adapter(x).forPort(T.class)` | no | Binds an adapter to a port a slice asks for. See [Ports, Adapters and Monitored Calls](/posts/eventmodeling-ports-and-adapters/) |
| `features()` | no | The feature-slice sub-builder: where to scan, which slices, which aspects. Closed with `done()` |

`build()` throws an `IllegalArgumentException` when `name` or `instance` is missing. The builder carries more settings — `observer(...)`, `shredding(...)`, `businessRules(...)`, `leadershipPriority(...)`, `management(...)` — which the guides on [observability](/posts/eventmodeling-observability/), [personal data](/posts/eventmodeling-personal-data/), [business rules](/posts/eventmodeling-business-rules/) and [running in production](/posts/eventmodeling-running-in-production/) cover.

### The Instance

An `Instance` is three strings, and they say where this context runs:

- **logical** — the deployment, the name you pass (`"banking-app"`)
- **physical** — which copy of it (`"front"`, `"batch"`, a pod name), `"default"` when you pass none
- **process** — minted once per JVM run from the host name, the process id, the start time and a random part

Every event the context appends carries them as `x-instance-*` tags, and processors that elect a leader use them to tell one process from another. Pass a logical name that is stable across restarts and a physical one that distinguishes the parts of a split deployment.

### Registering Directly on the Builder

Every component a slice registers can also be registered straight on the builder — `builder.readmodel(...)`, `.automation(...)`, `.translator(...)` and so on. That is useful for a small test or a context with a handful of parts. For an application, register them from feature slices: a slice keeps a component next to the code that wires it, and it is what lets a deployment run only some of its parts.

## Feature Slices

A feature slice is a class annotated `@FeatureSlice` that implements `Slice<C>`, where `C` is the bounded context it belongs to:

```java
@FeatureSlice(chapter = "Account management")
public class AccountDetailsFeatureSlice implements Slice<Banking> {

    @Override
    public void configureQuery ( BoundedContextBuilder<Banking> builder ) {
        // register: projected per read, so always current
        builder.readmodel(AccountDetailsReadModel.class).live();
    }

    @Override
    public void startQuery ( Banking banking ) {
        // wire: the endpoint holds the narrow BankingApi, not the whole context
        new AccountDetailsEndpoint(banking, banking.port(HttpRoutes.class));
    }
}
```

The builder's `features().rootPackage(...)` scans that package and its subpackages for `@FeatureSlice` classes, instantiates each through its no-argument constructor, and calls its hooks.

### The Annotation Carries What the Code Cannot Say

`@FeatureSlice` has one attribute, `chapter`: the column of the event model the slice is grouped under, both in the model and on a dashboard. Everything else about a slice is read off the code:

- **Its name** is the class's simple name without the `FeatureSlice` suffix (`AccountDetails`), from `Slice.name()`.
- **Its bounded context** is the `C` of `Slice<C>`.
- **Its type** — state change, state view, automation, translation — is derived from what it registers (below).

A value repeated in an annotation is one nobody checks against the code, and it agrees with the code only until the first refactoring. So the annotation holds the chapter, and nothing that could drift.

### Slices Are Scanned for the Context They Declare

Several bounded contexts may scan one package — the repository's banking examples do exactly that. A context deploys a slice only when the context type it declared can accept this one: `Slice<Banking>` is deployed by `Banking`, a slice declared over a sibling context is skipped, and one declared over a supertype (`Slice<BoundedContext<?,?,?>>`) is deployed by every context. A skipped slice is not instantiated at all, and it does not appear in either inventory of the context: it is none of its business.

> A slice declared over the wrong context is not an error, it is simply not deployed. If a slice does not turn up, check the type argument of `Slice<C>` first; the skip is logged at DEBUG.
{: .prompt-tip }

### Two Hooks per Aspect: Configure and Start

A slice has four **aspects**, and two hooks per aspect:

| Aspect | Configure hook (during `build()`) | Start hook (during `start()`) | What belongs there |
|---|---|---|---|
| `COMMAND` | `configureCommand(builder)` | `startCommand(context)` | Commands (`builder.command(...)`), aggregates, and the endpoints that execute them |
| `QUERY` | `configureQuery(builder)` | `startQuery(context)` | Live read models, in-memory eventually consistent read models, query objects, and the endpoints that read them |
| `AUTOMATION` | `configureAutomation(builder)` | `startAutomation(context)` | Automations and their todo lists, policies, publishers, translators and dispatchers |
| `PROJECTION` | `configureProjection(builder)` | `startProjection(context)` | The projectors of durable, shared read models (an SQL read model's projector) |

The two kinds of hook do different work:

- **`configure...`** registers components on the builder. It runs inside `build()`, before the context exists. A slice may take a port here (`builder.port(DataSource.class)`) to construct a component that needs one.
- **`start...`** wires whatever runs beside the components — an HTTP endpoint, a scheduler — once the context has been built and is starting. It is handed the context as your interface type (`Banking`), so an endpoint can hold the narrow `BankingApi`. A port looked up through that context is reported as called by this slice, even from a request thread.

All eight hooks have empty default implementations; a slice overrides the ones it uses.

The four aspects exist so that one slice can run in several places. The read model that answers a query, the projector keeping a shared SQL read model up to date, and the automation driving work forward need not live in the same process. The Projects context in the Sliceworkz Modeler registers its SQL projector from `configureProjection` and the query object reading the same tables from `configureQuery`:

```java
@Override
public void configureProjection ( BoundedContextBuilder<Projects> builder ) {
    DataSource dataSource = builder.port(DataSource.class, "readmodel");
    ProjectListProjector projector = new ProjectListProjector(dataSource);
    projector.ensureTables();
    builder.readmodel(projector).eventuallyConsistent();
}

@Override
public void configureQuery ( BoundedContextBuilder<Projects> builder ) {
    DataSource dataSource = builder.port(DataSource.class, "readmodel");
    query = new ProjectListQuery(dataSource);
}
```

An eventually consistent read model held **in memory** — a `PublishingReadModel` like the quickstart's account overview — is different: its state lives in the process that projects it, so it has to be registered where it is read, from `configureQuery`. See [Choosing a Read Model](/posts/eventmodeling-read-models/) and [Durable SQL Read Models](/posts/eventmodeling-sql-read-models/).

### Where Components Must Be Registered

Most of the table above is convention, but `build()` enforces two parts of it:

- **A publisher and a policy belong to the automation aspect.** Registered by a slice from any other hook, `build()` fails naming the slice and the hook. A publisher or policy runs where automations run; registered from `configureQuery` it would quietly run on the instances serving queries — and in a split deployment with no automation instance it would run nowhere anyone looks.
- **An automation needs its todo list projected on the same instance.** The automation waits for the projector of its todo list, so `build()` rejects an automation whose `EPHEMERAL` or `LOCAL` todo list is not also registered `.eventuallyConsistent()` on this instance. Register the two side by side in `configureAutomation`, as the payments example does:

```java
@Override
public void configureAutomation ( BoundedContextBuilder<Payments> builder ) {
    PaymentGateway gateway = builder.port(PaymentGateway.class);
    var todoList = new PaymentsToExecuteTodoList();
    builder.readmodel(todoList).eventuallyConsistent();
    builder.automation(new ExecutePaymentAutomation(todoList, gateway));
}
```

A component registered straight on the builder, outside any slice, runs wherever that builder runs.

### Declaring Commands Is Declarative

A command is not wired into the context: the caller constructs it and executes it. So `builder.command(OpenAccountCommand.class)` changes nothing about how commands run. What it does is put the command in the slice's inventory, announced when the context starts — which is how a dashboard shows a command before its first execution, and how the slice's type comes out right.

```java
@FeatureSlice(chapter = "Account management")
public class OpenAccountFeatureSlice implements Slice<Banking> {

    @Override
    public void configureCommand ( BoundedContextBuilder<Banking> builder ) {
        builder.command(OpenAccountCommand.class);
    }
}
```

### A Slice's Type Is Derived, Never Declared

The framework works out which Event Modeling pattern a slice is (`SliceType`) from the components it registered:

| Registered | Type |
|---|---|
| a command or an aggregate | `STATE_CHANGE` |
| read models only | `STATE_VIEW` |
| an automation and its todo list, a policy, a command beside a read model, or only a publisher | `AUTOMATION` |
| a translator | `TRANSLATION` |
| parts of more than one pattern | `UNCLEAR` |
| nothing | `UNDEFINED` |

The rule is the one the Sliceworkz Modeler applies to a modeled slice, so a slice in the code and the same slice in the model come out as the same type. Three consequences:

- **Only what is registered counts.** A slice that executes a command from an endpoint in `startCommand` but never registers it is `UNDEFINED`. Declare the command with `builder.command(...)`.
- **A command beside a read model is an automation**, exactly as in the model: the read model is read as the todo list the command is issued from. A command and the live read model its screen shows are two slices in Event Modeling — split them, and both types come out right.
- **Only what is deployed is registered.** A slice is configured only for the aspects its instance runs, so the type reported by an instance is the type of what that instance deployed of the slice.

The type is reported on `BoundedContextStarting` and on every kernel event attributed to the slice, together with its name, chapter and members.

## Feature Package Layout

One package per slice, named for the slice, under a `features` package beside the context interface:

```
com/example/banking/
  Banking.java                        bounded context interface
  BankingApi.java                     the application's narrow surface
  BankingDomain.java                  entities, ids and the three event hierarchies
  features/
    openaccount/
      OpenAccountFeatureSlice.java    @FeatureSlice, registers the command
      OpenAccountCommand.java
    deposit/
      DepositFeatureSlice.java
      DepositCommand.java
      AccountExistsDecisionModel.java
    accountdetails/
      AccountDetailsFeatureSlice.java
      AccountDetailsReadModel.java    live read model
      AccountDetailsEndpoint.java     wired from startQuery
    accountoverview/
      AccountOverviewFeatureSlice.java
      AccountOverviewReadModel.java   eventually consistent PublishingReadModel
```

Keep everything a slice needs inside its package. A decision model that several commands share can live in a package of its own, the way the banking example's `accountstanding` package holds `AccountStandingDecisionModel`. Slices do not import from each other's packages: that is what keeps a slice deletable, movable and deployable on its own.

## Deploying a Context in Parts

The `features()` sub-builder decides **which slices** this process deploys and **which of their aspects**. All four aspects are enabled by default, so a single process runs everything. Split them over processes with `disableX()` (or `enableX(boolean)`, for a value read from configuration):

```java
// the front process: accepts commands and answers queries, runs no background processors
Banking front = BoundedContext.newBuilder(Banking.class)
    .name("banking")
    .eventStorage(storage)
    .instance(InstanceFactory.determine("banking-app", "front"))
    .adapter(routes).forPort(HttpRoutes.class)
    .features()
        .rootPackage(Banking.class.getPackage())
        .disableAutomations()
        .disableProjections()
        .done()
    .build();

// the batch process: automations and durable projections, nothing from outside
Banking batch = BoundedContext.newBuilder(Banking.class)
    .name("banking")
    .eventStorage(storage)
    .instance(InstanceFactory.determine("banking-app", "batch"))
    .features()
        .rootPackage(Banking.class.getPackage())
        .disableCommands()
        .disableQueries()
        .done()
    .build();
```

Both processes run the same code against the same storage under the same context name; each calls only the hooks of the aspects it runs. A disabled aspect's `configure...` and `start...` hooks are never called, so its components are never registered and its endpoints never wired.

`filter(...)` selects slices rather than aspects. It takes a predicate over the slice instance, so you can select on `name()`, `chapter()`, the class or its package:

```java
.features()
    .rootPackage(Banking.class.getPackage())
    .filter(slice -> !slice.chapter().equals("Money movements"))
    .done()
```

A slice that the filter leaves out is still instantiated and reported as *undeployed* on `BoundedContextStarting`, which is how a dashboard shows that it exists and is switched off here. `getDeployedFeatureSlices()` and `getUndeployedFeatureSlices()` on the built context return the two lists.

> Where a processor that must run once per deployment — an automation, a policy, a shared SQL projector — is deployed on several instances, the instances elect one leader per processor. See [Running in Production](/posts/eventmodeling-running-in-production/).
{: .prompt-info }

## What `build()` Refuses

`build()` constructs everything and starts nothing. Once the feature slices have registered their components, and before it constructs a single processor, it checks the registrations — all of them, the slices' included — and fails with an `IllegalArgumentException` that names **every** offender at once. The component names (the last two rows) are checked as each kind of processor is constructed. Each check catches a mistake that would otherwise fail silently: no exception, no log line, just a read model that stays empty or a processor that replays history on every start.

| Mistake | Message (abridged) | What would happen otherwise |
|---|---|---|
| A read model registered without `.live()` or `.eventuallyConsistent()` | `readmodel registered without saying how it is projected: Overview` | Nobody decided whether the read model is current or lags; every caller lives with that difference |
| A component declared over another context's event types | `component registered over another bounded context's event types: readmodel PaymentsReadModel (its domain event type is ...PaymentEvent, where this context's is ...BankingDomainEvent)` | Its query names types that never occur on this stream, so it stays empty for good |
| A live read model class that is abstract or has no reachable constructor | names the class | Every read of it fails |
| An automation whose todo list is not projected on this instance | `automation registered without a projector for its todo list on this instance: ...` | The automation waits forever on a bookmark nobody writes |
| A publisher or a policy registered from a hook other than `configureAutomation` | `... registered outside the automation aspect: ...` | It runs on the wrong instances |
| A policy without its two choices (`fromNowOn()`/`fromTheBeginning()`, `stallOnRejection()`/`skipRejections()`) | `policy registered without saying where it starts and what a rejection does: ...` | A default would decide something about your domain silently |
| A read model, automation, policy, translator, publisher or dispatcher that is an anonymous class or lambda | `readmodel ...$1 must be a named class: its name identifies it and keys the bookmark recording its progress...` | A fresh bookmark on every start: the whole stream handled again at every boot |
| Two components of one kind with the same name | `duplicate readmodel name 'Overview' - bookmarks would collide` | Both advance one bookmark past events the other never saw |

Registering a read model class and then calling `.eventuallyConsistent()` on it, or an instance with `.live()`, fails at the call itself: a class can only be projected live, per read, and an instance only in the background.

### Component Names Are Bookmark Keys

The last two rows deserve a word. A read model, automation, policy, translator, publisher or dispatcher is identified by a name — the class's simple name, or whatever `readmodelName()` (read models) or `policyName()` (policies) returns — and that name keys the [bookmark](/posts/eventstore-bookmarking/) recording how far the component has processed the stream. Two properties follow, and `build()` enforces both:

- **Unique within its kind.** Two components sharing a name share one bookmark.
- **Stable across restarts.** An anonymous class has no name of its own; a lambda or a generated proxy gets one that changes per JVM run. Either way the bookmark is new at every start, and the whole history is handled again. For a dispatcher, that is publishing everything to an external system again on every restart.

So components are named classes. An anonymous read model that returns a fixed `readmodelName()` is accepted: it named itself. And because the name is stored, renaming a class renames its bookmark — rename a durable read model's class and it rebuilds from the start, unless it overrides `readmodelName()` to keep the old one.

### A Failed Build Leaves Nothing Running

When `build()` throws, you get no context back, so you could not release what it had constructed. It releases it itself: the parts it had already built are terminated and the `EventStore` it opened is closed. The storage you passed in stays open, as it always does.

## `build()`, Then `start()`

`build()` assembles. It registers the components, opens the context's streams and constructs the processors in a stopped state; no processor thread exists yet. `start()` runs:

1. It announces the context with a `BoundedContextStarting` event, carrying the deployed and undeployed slices, the aspects this instance runs, the rulebook and the port bindings.
2. It calls each deployed slice's `start...` hooks for the aspects this instance runs.
3. It runs one leader election round, then starts the processors.
4. It **waits until every `EPHEMERAL` read model has been projected completely**, so an in-memory read model is usable as soon as `start()` returns. Durable read models catch up in the background from their bookmarks. The wait has a safety timeout of five minutes (`-Dsliceworkz.eventmodeling.readmodel.ephemeral.projection.timeout.ms=...`), after which startup continues with a warning.
5. It reports `BoundedContextStarted`, with the startup duration.

A second `start()` on a started context is ignored with a warning.

## Stopping and Terminating

| Call | What it does | Can it start again? |
|---|---|---|
| `stop()` | Stops the processors and hands back the leases this instance holds, so another instance takes over | Yes, `start()` again |
| `terminate()` | Stops everything, drains the processor threads, releases the leases and **closes the `EventStore` the context built** | No: `start()` on a terminated context throws `IllegalStateException` |

Ownership decides who closes what:

- **The context closes its own store.** It built the `EventStore` over the storage you gave it, and nothing else holds that store, so `terminate()` closes it.
- **The storage is yours.** The context never closes it: it came from outside, one storage can back several contexts, and it usually outlives them. Close it yourself, after terminating every context on it.
- **What you supplied to the storage — a `DataSource`, say — is closed after the storage.**

```java
banking.terminate();   // closes the store the context built; the storage is untouched
storage.close();       // ours to close, once no context runs on it
pool.close();          // supplied by us to the storage, so closed after it
```

The [eventstore lifecycle guide](/posts/eventstore-lifecycle/) explains what closing a storage releases.

### The Shutdown Hook

Every context registers a JVM shutdown hook that terminates it, so a process that simply exits leaves nothing stranded. `terminate()` deregisters that hook, which matters for anything that builds and terminates contexts within one JVM — a test suite above all: an unterminated context stays reachable through its hook, with everything it holds, until the JVM exits. Terminate what you build.

## Where a Context Keeps Its Events

A context writes to three streams, named after it:

| Stream | Id for the context `banking` | Holds |
|---|---|---|
| `BoundedContextStreams.domain("banking")` | `banking#domain` | Domain events |
| `BoundedContextStreams.inbound("banking")` | `banking#inbound` | Inbound events received through `incoming(...)` |
| `BoundedContextStreams.outbound("banking")` | `banking#outbound` | Outbound events written by publishers |

That layout is wire format: a dashboard, a test asserting on what a command appended, or a migration script finds the events there, usually from another process that holds no built context — which is why it is a public function of the context's name. It also means **the context name is stored data**: renaming a context sends every new event to new streams and leaves the old ones behind.

`BoundedContextStreams` is for tooling and tests. **Application code does not open these streams.** There is deliberately no capability on the context that hands one out. Everything an application does with its events is served by a component that carries a bookmark, a lease, retry and lifecycle events:

| You want to... | Use |
|---|---|
| decide | a command, or an aggregate |
| read | a read model |
| react to what happened | an automation and its todo list, or a policy |
| take in what other systems say | a translator |
| tell other systems | a publisher and a dispatcher |
| know what a command decided, such as a minted id | a `CommandWithResult` |

A hand-rolled subscription on a raw stream gives all of that up: it misses whatever is appended while the process is down, runs on every instance, and is not retried when it throws. [Who May Do What](/posts/eventmodeling-who-may-do-what/) has the full picture of who holds which part of a context.

## Summary

- Declare a context as an interface extending `BoundedContext<D,I,O>` over three sealed event roots, and build it with `BoundedContext.newBuilder(...)`: `name`, `eventStorage` and `instance` are required.
- A feature slice is `@FeatureSlice(chapter = ...)` on a class implementing `Slice<C>`. Its name, context and type are read off the code.
- `configure...` hooks register components during `build()`; `start...` hooks wire endpoints during `start()`. One pair per aspect: command, query, automation, projection.
- Publishers and policies go in `configureAutomation`, and an automation's todo list is registered next to it.
- `features()` selects slices with `filter(...)` and aspects with `disableX()`, which is how one context is split over front and batch processes.
- `build()` rejects, naming every offender, what would otherwise fail silently: a read model without a mode, a component of another context, an unprojected todo list, an anonymous or duplicate component name.
- `terminate()` closes the store the context built; the storage is yours to close afterwards.
- `BoundedContextStreams` says where a context's events live, for tooling. Application code reaches its events only through commands, read models and processors.

Next: [Entities, Typed Ids and Tags](/posts/eventmodeling-entities-and-tags/).
