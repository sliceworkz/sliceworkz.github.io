---
layout: post
toc: true
title: Who May Do What
description: Handing each caller the narrow capability type its job needs instead of the whole bounded context
date: 2026-10-12 11:00:00
categories: [Eventmodeling Documentation,Eventmodeling Guides]
tags: [eventmodeling, capabilities, bounded context, api design, architecture]
---

A built bounded context can do everything: execute commands, read read models, append a domain event no command raised, translate inbound events, stop an automation, restart a projector, erase a person's data, hand out a port, and terminate itself. The code that built it needs all of that.

Almost nothing else does. A web controller handed `Banking` can call `erase()`, `stopAutomation()` and `event()`, and the compiler sees nothing wrong — nor is any of them a mistake anybody makes on purpose. This guide shows how to hand each caller a reference whose type says what it may do, which turns those mistakes into compile errors at the cost of a type name.

## The Rule: a Reference Says Who Holds It

The framework files every capability of a bounded context under an **audience** — one interface per kind of caller. `BoundedContext` extends `AllCapabilities`, and `AllCapabilities` is nothing but the composition of those audiences:

| Audience | Interface | What it carries |
|---|---|---|
| **Application** — a controller, a scheduled job, any adapter driving the domain | `ApplicationCapabilities<D,O>` | `execute`, `executeWithRetry`, `evaluate`, `read`, `aggregate` |
| **Inbound edge** — a webhook, a message consumer feeding the domain | `TranslationCapability<I>` | `incoming`, `translate` |
| **Operator** — an admin endpoint, a dashboard | `OperationsCapabilities` | `automations`, `restartAutomation`, `stopAutomation`, `processors`, `restartProcessor`, `stopProcessor`, `skipStalledEvent` |
| **Erasure requests** | `PrivacyCapability` | `erase`, `eraseCategory` |
| **The escape hatch** | `ProvidedEventCapability<D>` | `event` |
| **Owner** — whatever built the context | your context interface (`Banking`) | all of the above, plus `start`/`stop`/`terminate`, `port`, `getDeployedFeatureSlices`/`getUndeployedFeatureSlices` |

`OperationsCapabilities` is itself the composition of `AutomationAdminCapability` (the three automation methods) and `ProcessorAdminCapability` (the four processor methods), for code that needs only one of the two.

The owner's own capabilities — `LifecycleCapability`, `PortsCapability`, `FeatureSliceCapabilities` — are not grouped into an audience, because only the code that called `build()` has a use for them.

> This is a boundary of discipline, not of security. The object behind a narrowed reference is still the whole context, and a cast reaches it. What narrowing buys is that reaching past the boundary has to be written down, so it shows up in review instead of in an incident.
{: .prompt-info }

## Narrowing Costs a Reference Type

A built context already *is* each of these interfaces, so there is nothing to wrap and nothing to convert. Assigning it to the narrower type is the whole technique:

```java
Banking banking = BoundedContext.newBuilder(Banking.class) /* ... */ .build();
banking.start();                                        // the owner, holding everything

ApplicationCapabilities<BankingDomainEvent, BankingOutboundEvent> app = banking;
app.execute(new OpenAccountCommand(customerId));        // compiles
app.read(AccountDetailsReadModel.class, accountId);     // compiles
app.terminate();                                        // does not compile
app.erase("customer", id, reason);                      // does not compile
app.event(new AccountOpened(...));                      // does not compile
app.stopAutomation("MonthEndClosingAutomation");        // does not compile
```

Build the context in one place — your `main`, your dependency-injection configuration — and hand out the narrow types from there. A caller then gets the reach its job needs, and cannot be handed another's by a wiring change nobody looked at twice.

## An Alias Interface: `BankingApi`

`ApplicationCapabilities<BankingDomainEvent, BankingOutboundEvent>` at every call site is the same noise `BoundedContext<BankingDomainEvent, BankingInboundEvent, BankingOutboundEvent>` would be, which is why the context interface `Banking` exists at all. Give the application surface the same treatment:

```java
public interface BankingApi extends ApplicationCapabilities<BankingDomainEvent, BankingOutboundEvent> { }

public interface Banking extends BoundedContext<BankingDomainEvent, BankingInboundEvent, BankingOutboundEvent>,
                                 BankingApi { }
```

With this declaration `Banking` reaches `ApplicationCapabilities` twice — through `BoundedContext` and through `BankingApi` — with the same type arguments both ways. That is legal Java, and it is what keeps the alias honest: an alias declared with the wrong event types does not compile, rather than becoming a second surface. A built `Banking` is a `BankingApi` with no cast:

```java
/** A web controller: decides through commands, reads through read models, and nothing else. */
public class AccountsController {

    private final BankingApi app;

    public AccountsController ( BankingApi app ) {
        this.app = app;
    }

    public AccountId openAccount ( String customerId ) {
        return app.execute(new OpenAccountCommand(BankingDomain.CUSTOMER.id(customerId))).response();
    }

    public Optional<AccountDetails> account ( String accountId ) {
        return app.read(AccountDetailsReadModel.class, BankingDomain.ACCOUNT.id(accountId)).getAccountDetails();
    }
}
```

The alias is optional — `Banking` is an `ApplicationCapabilities<BankingDomainEvent, BankingOutboundEvent>` whether you declare `BankingApi` or not — but it is what makes the narrow type the convenient one to write.

The same goes for the endpoints a feature slice wires in its `start...` hooks: the hook is handed a `Banking`, and the endpoint's constructor takes a `BankingApi`. See [Bounded Contexts and Feature Slices](/posts/eventmodeling-bounded-contexts-and-feature-slices/#two-hooks-per-aspect-configure-and-start).

## One Place Holds Everything

Wiring a context for all its audiences looks like this:

```java
/** The one place that holds the whole context, and hands out narrow references from it. */
public class BankingWiring {

    final Banking banking;
    final AccountsController accounts;
    final OperationsEndpoint operations;
    final ErasureRequests erasures;
    final CustomerFeed customerFeed;
    final LegacyAccountImport legacyImport;

    public BankingWiring ( EventStorage storage, HttpRoutes routes ) {
        banking = BankingDeployments.allInOne(storage, routes);

        accounts     = new AccountsController(banking);     // a Banking is a BankingApi
        operations   = new OperationsEndpoint(banking);     // ... and an OperationsCapabilities
        erasures     = new ErasureRequests(banking);        // ... and a PrivacyCapability
        customerFeed = new CustomerFeed(banking);           // ... and a TranslationCapability
        legacyImport = new LegacyAccountImport(banking);    // ... and a ProvidedEventCapability
    }

    public void start ( ) {
        banking.start();
    }

    public void stop ( ) {
        banking.terminate();
    }
}
```

Each class downstream names exactly the capability it uses in its constructor, so its signature documents its reach and a reviewer sees immediately when that reach grows.

## Why Each Line of the Table Falls Where It Does

### `event()` Is Not an Application Capability

`event(...)` appends a domain event that no command raised. No [decision model](/posts/eventmodeling-commands-and-decision-models/) is read, no consistency boundary is pinned, and none is checked on append, so no `OptimisticLockingException` is possible. That is right for facts already settled before they reach the context:

- a CRUD front-end recording what it has stored elsewhere,
- an import or a migration of history from another system,
- an automation or a translator, which reach `event()` through their own context.

On the application surface it would be the one capability whose misuse is silent: the append succeeds, and the invariant a command would have checked is never checked. So an application opts into it by naming `ProvidedEventCapability`, and has then said so:

```java
/** A one-off migration: records accounts the old system already opened. */
public class LegacyAccountImport {

    private final ProvidedEventCapability<BankingDomainEvent> provided;

    public LegacyAccountImport ( ProvidedEventCapability<BankingDomainEvent> provided ) {
        this.provided = provided;
    }

    public void importAccount ( String legacyAccountNumber, String customerNumber, LocalDate openedOn ) {
        AccountId accountId = ACCOUNT.id(legacyAccountNumber);
        CustomerId customerId = CUSTOMER.id(customerNumber);
        provided.event(
            new AccountOpened(accountId, customerId, openedOn),
            Tags.of(ACCOUNT.tag(accountId), CUSTOMER.tag(customerId)),
            "legacy-import/" + legacyAccountNumber);   // derived from the fact, so a re-run stores nothing twice
    }
}
```

Give what you append through `event()` an idempotency key derived from the fact — here the legacy account number — never from the attempt. A re-run of the import then appends nothing a second time; a key minted per attempt would de-duplicate nothing. See [Appending Events](/posts/eventstore-appending-events/#idempotency) for how the store handles the key.

### Lifecycle Belongs to the Owner

`start()`, `stop()` and `terminate()` are the other end of `build()`. `terminate()` closes the `EventStore` the context built and drains its processor threads; whoever built the context is the only caller with a reason to call it. An operator who needs something to stop stops a *named* automation or processor instead, which is what `OperationsCapabilities` carries.

### An Operator Decides Nothing

`OperationsCapabilities` has no `read` and no `execute`. An operations tool that can also execute commands is one that can make business decisions under an operator's credentials:

```java
/** An admin endpoint: sees and restarts processors, decides nothing. */
public class OperationsEndpoint {

    private final OperationsCapabilities ops;

    public OperationsEndpoint ( OperationsCapabilities ops ) {
        this.ops = ops;
    }

    public List<ProcessorStatus> processors ( ) {
        return ops.processors();
    }

    public List<AutomationStatus> automations ( ) {
        return ops.automations();
    }

    public boolean restartReadModelProjector ( String readModel ) {
        return ops.restartProcessor(ProcessorKind.READ_MODEL, readModel);   // false when already running
    }
}
```

Both halves of it address **the instance they are called on**. Every instance runs its own processors, so stopping an automation on one instance stops it there. Reaching the other instances of a deployment is the management stream's job, which ends up calling exactly these methods on the instances an instruction names — see [Running in Production](/posts/eventmodeling-running-in-production/) and [Automations and Policies](/posts/eventmodeling-automations-and-policies/) for what stopping and restarting does to each kind of processor.

### An Erasure Is Its Own Audience

`erase(...)` answers a request from a person, under an authority neither a controller nor an operations dashboard carries, and it is irreversible: it destroys keys, and every value sealed under them becomes unreadable in the event table, the write-ahead log, the replicas and every backup. It stays a leaf of its own, so that holding it is a decision:

```java
/** Handles a person's request to be forgotten, and nothing else. */
public class ErasureRequests {

    private final PrivacyCapability privacy;

    public ErasureRequests ( PrivacyCapability privacy ) {
        this.privacy = privacy;
    }

    public SubjectErasureReport forget ( String customerId, String requestNumber ) {
        return privacy.erase("customer", customerId, ErasureReason.of("GDPR art.17 request " + requestNumber));
    }
}
```

`erase(type, id, reason)` erases the person, in every category; `eraseCategory(subject, reason)` erases one category only. [Personal Data and Erasure](/posts/eventmodeling-personal-data/) covers the difference and what an erasure does not reach.

### Aggregates Ride With Commands

`aggregate(...)` is a second write path. It sits on the application surface beside `execute` because it, too, writes **through a boundary**: an aggregate raises events within its identity and conflicts when that identity moved underneath it, which is exactly what `event()` does not do. A team uses one of the two write styles and simply never registers the other, so carrying both costs it nothing. See [Aggregates and DCB](/posts/aggregates-and-dcb/).

### The Inbound Edge Is Separate From the Application

`incoming(...)` and `translate(...)` feed the domain with events from outside, which is a different job from deciding on behalf of a user:

```java
/** Consumes the CRM's messages and hands them to the context. */
public class CustomerFeed {

    private final TranslationCapability<BankingInboundEvent> inbound;

    public CustomerFeed ( TranslationCapability<BankingInboundEvent> inbound ) {
        this.inbound = inbound;
    }

    public void onCustomerSuspended ( String messageId, String customerNumber ) {
        inbound.incoming(new CustomerSuspended(CUSTOMER.id(customerNumber)), "crm/" + messageId);
    }
}
```

The key is the CRM's message id, so a message the broker redelivers is stored once. An adapter that genuinely both receives and decides holds both types; most hold one. [Translators, Publishers and Dispatchers](/posts/eventmodeling-translators-publishers-dispatchers/) covers the two paths.

### The Groupings Are by Audience, Never by Style

The audiences are *who* holds the reference, not *how* a team writes its domain. An alternative grouping — a `CQRSCapabilities` and a `DCBCapabilities` for the two ways to drive the framework — loses because a style is not a caller. Both would carry `read`, both would need the lifecycle to be the one type an application holds, and the nearest thing to an application surface would end up handing out `terminate()`. Which write path a team uses is a property of what it registers, not of who holds the reference.

## No Audience Opens the Event Log

There is no capability, on any audience, that hands out the context's event streams. An application reaches its events only through what the framework puts in front of them:

| You want to... | Use |
|---|---|
| decide | a command, or an aggregate |
| read | a read model |
| react to what happened | an automation and its todo list, or a policy |
| take in what other systems say | a translator |
| tell other systems | a publisher and a dispatcher |
| know what a command decided, such as an id it minted | a `CommandWithResult` |

Each of those carries a bookmark, a lease, retry, and lifecycle events that say it is working. A raw read or subscription gives all of that up: it misses whatever is appended while the process is down, runs on every instance, and is not retried when it throws. And a caller that needs something a command decided gets it from the command's result, rather than by reading the event back out of the stream:

```java
AccountId accountId = app.execute(new OpenAccountCommand(customerId)).response();
```

Tooling and tests that do read a context's streams find them through `BoundedContextStreams`, which states where a context keeps its events from the context's name alone — see [Where a Context Keeps Its Events](/posts/eventmodeling-bounded-contexts-and-feature-slices/#where-a-context-keeps-its-events).

## Processors Get Their Own Narrow Contexts

The same principle applies inside the framework. A component is never handed the bounded context; it gets a context of its own, carrying what its role needs:

| Component | Is handed | Which can |
|---|---|---|
| Command | `CommandContext` | read its decision models, raise events, check business rules |
| Automation | `AutomationContext` | execute commands, append provided events |
| Translator | `TranslatorContext` | execute commands, append provided events |
| Publisher | `PublisherContext` | read live read models at a bound, publish outbound events |

An automation cannot erase a person, and a publisher cannot execute a command. The guides on [commands](/posts/eventmodeling-commands-and-decision-models/), [automations](/posts/eventmodeling-automations-and-policies/) and [publishers](/posts/eventmodeling-translators-publishers-dispatchers/) cover each.

## Reading Across Contexts: `UnboundedReadModelCapability`

One capability sits outside the table on purpose. `read(...)` projects a read model over the context's own domain stream. `UnboundedReadModelCapability<D>` offers `readUnbounded(...)`, which projects over **every stream in the store**, regardless of context boundaries. It is meant for internal dashboards and monitoring, not for application code, and it is not part of `AllCapabilities` — an ordinary bounded context does not carry it.

You opt in by naming it on a context interface of your own. `build()` returns a proxy over exactly the interface it was asked for, so what that interface names is what you get:

```java
public interface BankingDashboard
        extends BoundedContext<BankingDomainEvent, BankingInboundEvent, BankingOutboundEvent>,
                UnboundedReadModelCapability<BankingDomainEvent> {
}

BankingDashboard dashboard = BoundedContext.newBuilder(BankingDashboard.class)
    .name("banking")
    .eventStorage(storage)
    .instance(InstanceFactory.determine("banking-dashboard"))
    .readmodel(AccountDetailsReadModel.class).live()
    .build();

dashboard.start();
AccountDetailsReadModel details = dashboard.readUnbounded(AccountDetailsReadModel.class, accountId);
```

As with `read`, the read model must be registered `.live()` on that context. This is the same mechanism as narrowing, used in the other direction: the interface you build decides the surface you hold.

## Adding a Capability of Your Own

When you extend a context interface, decide whose a method is, and put it on the narrowest interface that serves that caller. A capability that widens only the wide surface breaks no test: the extra method is simply never called through a narrow type, and nobody notices it is reachable from everywhere the wide type goes. The framework holds its own surface to this — every method reachable on a bounded context is filed under some narrower audience, and that is checked by its build.

## Summary

- `BoundedContext` is the composition of audiences: application, inbound edge, operator, erasure, escape hatch, owner.
- Narrowing is an assignment: a built context already is each audience. Build the context in one place and hand out narrow types from there.
- Declare `BankingApi extends ApplicationCapabilities<D,O>` and let `Banking` extend it; controllers and endpoints take `BankingApi`.
- `event()` appends without a boundary check, so it is off the application surface; opt in with `ProvidedEventCapability` and an idempotency key derived from the fact.
- Lifecycle is the owner's; operators stop and restart named processors on the instance they call, and decide nothing; erasure is its own audience.
- No audience opens the event log; `UnboundedReadModelCapability` is named explicitly by a dashboard interface that needs it.

The repository's [`WHO-MAY-DO-WHAT.md`](https://github.com/sliceworkz/eventmodeling/blob/develop/WHO-MAY-DO-WHAT.md) is the reference this guide follows, and the banking example's [`BankingApi`](https://github.com/sliceworkz/eventmodeling/blob/develop/sliceworkz-eventmodeling-examples/src/main/java/org/sliceworkz/eventmodeling/examples/banking/BankingApi.java) the shape to copy.

Next: [Testing Feature Slices](/posts/eventmodeling-testing/).
