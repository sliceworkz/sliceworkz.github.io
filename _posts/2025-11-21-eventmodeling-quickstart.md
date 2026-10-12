---
layout: post
toc: true
title: EventModeling Quickstart Guide
description: Getting started with Sliceworkz EventModeling
icon: fas fa-rocket
categories: [Eventmodeling Documentation]
tags: [quickstart,eventmodeling,maven]
pin: true
order: 3
---


This guide gets you started with Sliceworkz EventModeling, an opinionated Java framework for building applications designed with <a href="https://eventmodeling.org">Event Modeling</a>.

Every box on an event model has a home in the framework: a **command** raises **domain events**, a **read model** answers questions out of them, an **automation** or a **policy** reacts to them, a **translator** takes in what other systems tell you, and a **publisher** and **dispatcher** tell other systems what happened. Each vertical slice of the model becomes a **feature slice** in the code. Underneath, everything is stored in the <a href="/eventstore/">Sliceworkz Eventstore</a>, and every command decides within a **Dynamic Consistency Boundary** (DCB) rather than an aggregate.

# Quickstart Guide

We build a small banking context: open an account, deposit money into it, and read it back two ways — one read model projected on demand, one projected in the background. The guide uses version **0.7.1**.


## Prerequisites

- Java 21 or later
- Maven

## Step 1: Add the Maven Dependencies

Sliceworkz EventModeling is available in <a href="https://mvnrepository.com/artifact/org.sliceworkz">Maven Central</a>. Its modules — and the Eventstore modules it builds on — are versioned together in a Bill-of-Materials pom. Import it:

```xml
<properties>
    <sliceworkz.eventmodeling.version>0.7.1</sliceworkz.eventmodeling.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.sliceworkz</groupId>
            <artifactId>sliceworkz-eventmodeling-bom</artifactId>
            <version>${sliceworkz.eventmodeling.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

Then add the framework implementation, an event storage, and the test support:

```xml
<dependencies>
    <!-- the framework (brings the API with it) -->
    <dependency>
        <groupId>org.sliceworkz</groupId>
        <artifactId>sliceworkz-eventmodeling-impl</artifactId>
    </dependency>

    <!-- in-memory event storage, for development and tests -->
    <dependency>
        <groupId>org.sliceworkz</groupId>
        <artifactId>sliceworkz-eventstore-infra-inmem</artifactId>
    </dependency>

    <!-- given/when/then test bases (brings JUnit 5) -->
    <dependency>
        <groupId>org.sliceworkz</groupId>
        <artifactId>sliceworkz-eventmodeling-testing</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

For production, use `sliceworkz-eventstore-infra-postgres` instead of the in-memory storage. For local development where events should survive a restart without a database, `sliceworkz-eventstore-infra-inmem-fs` keeps them in files.

> **PostgreSQL version support.** EventModeling stores its events through Sliceworkz Eventstore and inherits its database support: the oldest supported PostgreSQL is **16**, and **18+** generates event ids with the native server-side `uuidv7()`. On 16 and 17 the ids are generated in Java, which needs the optional `com.github.f4b6a3:uuid-creator` dependency in your application; the right implementation is picked at startup. See the [PostgreSQL EventStorage guide](/posts/eventstore-configuring-postgresql-storage/#postgresql-version-support).
{: .prompt-info }

The framework logs through SLF4J; add the binding of your choice (Logback, for instance) to see what it reports.

## Step 2: Define the Domain

A bounded context has three kinds of events: **domain events** it records itself, **inbound events** other systems tell it, and **outbound events** it publishes. Next to them sit the **entities** — the kinds of thing events are about — each with an id type of its own.

Create `BankingDomain.java`:

```java
package com.example.banking;

import java.math.BigDecimal;
import java.time.LocalDate;

import org.sliceworkz.eventmodeling.domain.Entity;
import org.sliceworkz.eventmodeling.domain.EntityId;

public interface BankingDomain {

    // Entities: the kinds of thing events are about, each with its own id type
    record AccountId ( String value ) implements EntityId { }
    record CustomerId ( String value ) implements EntityId { }

    Entity<AccountId>  ACCOUNT  = Entity.of("account", AccountId::new);
    Entity<CustomerId> CUSTOMER = Entity.of("customer", CustomerId::new);

    // Domain events: the facts this context records, named in the past tense
    sealed interface BankingDomainEvent {
        record AccountOpened ( AccountId accountId, CustomerId customerId, LocalDate date ) implements BankingDomainEvent { }
        record MoneyDeposited ( AccountId accountId, BigDecimal amount ) implements BankingDomainEvent { }
    }

    // Inbound events: what other systems tell this context
    sealed interface BankingInboundEvent {
        record CustomerSuspended ( CustomerId customerId ) implements BankingInboundEvent { }
    }

    // Outbound events: what this context publishes to other systems
    sealed interface BankingOutboundEvent {
        record AccountAnnounced ( AccountId accountId ) implements BankingOutboundEvent { }
    }
}
```

**Key concepts:**
- A `sealed interface` per kind of event, with a `record` per event type: the compiler then checks every `switch` over them for exhaustiveness.
- Name domain events in the past tense (`AccountOpened`, not `OpenAccount`). The record's simple name is what the store keeps, so treat it as wire format — see [Defining Events](/posts/eventstore-defining-events/).
- An `Entity` names a tag key and makes the ids of its kind: `ACCOUNT.newId()` mints one, `ACCOUNT.tag(id)` tags an event with it. Because `AccountId` and `CustomerId` are different types, swapping them is a compile error rather than a query that silently finds nothing. See [Entities, Typed Ids and Tags](/posts/eventmodeling-entities-and-tags/).
- Records carried inside events keep a lenient constructor: they are rebuilt from stored history on every read, so a constructor that throws turns a tightened rule into an unreadable event. Validation happens before events exist — see [Where a Validation Goes](/posts/eventmodeling-where-validations-go/).

## Step 3: Declare the Bounded Context

The bounded context is the entry point to the application. Declare an interface for it, plus a narrower one for the code that only drives the domain:

```java
package com.example.banking;

import org.sliceworkz.eventmodeling.boundedcontext.ApplicationCapabilities;
import com.example.banking.BankingDomain.BankingDomainEvent;
import com.example.banking.BankingDomain.BankingOutboundEvent;

public interface BankingApi extends ApplicationCapabilities<BankingDomainEvent, BankingOutboundEvent> {
}
```

```java
package com.example.banking;

import org.sliceworkz.eventmodeling.boundedcontext.BoundedContext;
import com.example.banking.BankingDomain.BankingDomainEvent;
import com.example.banking.BankingDomain.BankingInboundEvent;
import com.example.banking.BankingDomain.BankingOutboundEvent;

public interface Banking
        extends BoundedContext<BankingDomainEvent, BankingInboundEvent, BankingOutboundEvent>, BankingApi {
}
```

**Key concepts:**
- `Banking` groups the three event types into one type, so feature slices and the builder name `Banking` instead of three type arguments.
- A built `Banking` can do everything: execute commands, erase personal data, stop automations, terminate itself. `BankingApi` carries only what application code needs — `execute`, `executeWithRetry`, `evaluate`, `read` and `aggregate` — and since `Banking` extends it, handing it out costs nothing but the reference type. See [Who May Do What](/posts/eventmodeling-who-may-do-what/).

## Step 4: A State Change — Opening an Account

Every slice of the event model becomes a feature slice: a package holding a class annotated `@FeatureSlice` and the components of the slice.

Create `features/openaccount/OpenAccountFeatureSlice.java`:

```java
package com.example.banking.features.openaccount;

import org.sliceworkz.eventmodeling.boundedcontext.BoundedContextBuilder;
import org.sliceworkz.eventmodeling.slices.FeatureSlice;
import org.sliceworkz.eventmodeling.slices.Slice;
import com.example.banking.Banking;

@FeatureSlice(chapter = "Account management")
public class OpenAccountFeatureSlice implements Slice<Banking> {

    @Override
    public void configureCommand ( BoundedContextBuilder<Banking> builder ) {
        builder.command(OpenAccountCommand.class);
    }
}
```

The caller of this command needs the id of the new account, so it is a `CommandWithResult`. Create `features/openaccount/OpenAccountCommand.java`:

```java
package com.example.banking.features.openaccount;

import java.time.LocalDate;

import org.sliceworkz.eventmodeling.commands.CommandContext;
import org.sliceworkz.eventmodeling.commands.CommandWithResult;
import org.sliceworkz.eventstore.events.Tags;
import com.example.banking.BankingDomain.AccountId;
import com.example.banking.BankingDomain.BankingDomainEvent;
import com.example.banking.BankingDomain.BankingDomainEvent.AccountOpened;
import com.example.banking.BankingDomain.CustomerId;

import static com.example.banking.BankingDomain.ACCOUNT;
import static com.example.banking.BankingDomain.CUSTOMER;

public class OpenAccountCommand implements CommandWithResult<BankingDomainEvent, AccountId> {

    private final CustomerId customerId;

    public OpenAccountCommand ( CustomerId customerId ) {
        if ( customerId == null ) {
            throw new IllegalArgumentException("customerId is required");
        }
        this.customerId = customerId;
    }

    @Override
    public AccountId execute ( CommandContext<BankingDomainEvent, BankingDomainEvent> context ) {

        // opening an account depends on no earlier fact, so there is nothing to decide on
        var result = context.noDecisionModels();

        AccountId accountId = ACCOUNT.newId();

        result.raiseEvent(
            new AccountOpened(accountId, customerId, LocalDate.now()),
            Tags.of(ACCOUNT.tag(accountId), CUSTOMER.tag(customerId)));

        return accountId;   // handed to the caller once the event is stored
    }
}
```

**Key concepts:**
- `@FeatureSlice` marks the class for discovery. It carries only what the code cannot say — the `chapter` of the event model the slice sits in. The slice's name comes from the class name, its bounded context from `Slice<Banking>`, and its type (state change, state read, automation, translation) is derived from what it registers.
- `configureCommand` is where a slice registers its command-side components. `builder.command(...)` is declarative: commands are instantiated by the caller, and registering one makes it part of the slice's inventory from startup.
- A command implements `Command<E>` (the caller gets the reference of the appended event) or `CommandWithResult<E, R>` (the caller also gets what `execute` returns, once the events are stored).
- `noDecisionModels()` is the explicit statement that this command decides on nothing; it hands back the result to raise events on.
- Every event is tagged with the entities it is about. Tags are how read models and decision models find events.

## Step 5: A State Change with a Decision — Depositing Money

A deposit is only meaningful on an account that exists, so this command decides on history. What a command decides on is a **decision model**: a small projection the command builds inline.

Create `features/deposit/AccountExistsDecisionModel.java`:

```java
package com.example.banking.features.deposit;

import org.sliceworkz.eventmodeling.commands.DecisionModel;
import org.sliceworkz.eventstore.events.Event;
import org.sliceworkz.eventstore.query.EventQuery;
import org.sliceworkz.eventstore.query.EventTypesFilter;
import com.example.banking.BankingDomain.AccountId;
import com.example.banking.BankingDomain.BankingDomainEvent;
import com.example.banking.BankingDomain.BankingDomainEvent.AccountOpened;

import static com.example.banking.BankingDomain.ACCOUNT;

public class AccountExistsDecisionModel implements DecisionModel<BankingDomainEvent> {

    private final AccountId accountId;
    private boolean exists;

    public AccountExistsDecisionModel ( AccountId accountId ) {
        this.accountId = accountId;
    }

    @Override
    public EventQuery eventQuery ( ) {
        return EventQuery.forEvents(EventTypesFilter.of(AccountOpened.class), ACCOUNT.tags(accountId));
    }

    @Override
    public void when ( Event<BankingDomainEvent> event ) {
        switch ( event.data() ) {
            case AccountOpened ignored -> exists = true;
            default -> { }
        }
    }

    public boolean exists ( ) {
        return exists;
    }
}
```

Create `features/deposit/DepositCommand.java`:

```java
package com.example.banking.features.deposit;

import java.math.BigDecimal;

import org.sliceworkz.eventmodeling.commands.BusinessException;
import org.sliceworkz.eventmodeling.commands.Command;
import org.sliceworkz.eventmodeling.commands.CommandContext;
import com.example.banking.BankingDomain.AccountId;
import com.example.banking.BankingDomain.BankingDomainEvent;
import com.example.banking.BankingDomain.BankingDomainEvent.MoneyDeposited;

import static com.example.banking.BankingDomain.ACCOUNT;

public class DepositCommand implements Command<BankingDomainEvent> {

    private final AccountId accountId;
    private final BigDecimal amount;

    public DepositCommand ( AccountId accountId, BigDecimal amount ) {
        // a malformed request is rejected before any history is read
        if ( accountId == null || amount == null || amount.signum() <= 0 ) {
            throw new IllegalArgumentException("a deposit needs an account and a positive amount");
        }
        this.accountId = accountId;
        this.amount = amount;
    }

    @Override
    public void execute ( CommandContext<BankingDomainEvent, BankingDomainEvent> context ) {

        // 1. select the decision models: projects them, and pins the consistency boundary
        var account = new AccountExistsDecisionModel(accountId);
        var result = context.decisionModels(account);

        // 2. check what history says
        BusinessException.when(!account.exists(), "Account " + accountId.value() + " does not exist");

        // 3. raise
        result.raiseEvent(new MoneyDeposited(accountId, amount), ACCOUNT.tags(accountId));
    }
}
```

Register it from `features/deposit/DepositFeatureSlice.java`, just like the first slice:

```java
@FeatureSlice(chapter = "Money movements")
public class DepositFeatureSlice implements Slice<Banking> {

    @Override
    public void configureCommand ( BoundedContextBuilder<Banking> builder ) {
        builder.command(DepositCommand.class);
    }
}
```

**Key concepts:**
- **Select, check, raise** is the shape of every command that decides.
- `decisionModels(...)` projects the models *and* makes their queries the command's consistency boundary. The append is checked against it: if an event matching any of those queries is appended between the read and the append, the command fails with an `OptimisticLockingException` instead of deciding on stale facts. Re-executing re-reads and re-decides; `executeWithRetry` does that for you with a bounded number of attempts.
- A rule history rejects is a `BusinessException`, not an `IllegalStateException`: the framework reports it as a rejection (`CommandRejected`), not as a failure, and the caller can tell the two apart.
- Commands never decide on a read model. A read model read from a command sits outside the consistency boundary — see [Commands and Decision Models](/posts/eventmodeling-commands-and-decision-models/).

## Step 6: A State Read — Account Details

A read model answers a question out of events. This one is **live**: projected when it is read, so it is always current.

Create `features/accountdetails/AccountDetailsReadModel.java`:

```java
package com.example.banking.features.accountdetails;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.Optional;

import org.sliceworkz.eventmodeling.readmodels.ReadModel;
import org.sliceworkz.eventstore.events.Event;
import org.sliceworkz.eventstore.query.EventQuery;
import com.example.banking.BankingDomain.AccountId;
import com.example.banking.BankingDomain.BankingDomainEvent;
import com.example.banking.BankingDomain.BankingDomainEvent.AccountOpened;
import com.example.banking.BankingDomain.BankingDomainEvent.MoneyDeposited;

import static com.example.banking.BankingDomain.ACCOUNT;

public class AccountDetailsReadModel implements ReadModel<BankingDomainEvent> {

    private final AccountId accountId;
    private AccountDetails details;

    public AccountDetailsReadModel ( AccountId accountId ) {
        this.accountId = accountId;
    }

    @Override
    public EventQuery eventQuery ( ) {
        // every event about this one account, whatever its type
        return EventQuery.forTags(ACCOUNT.tags(accountId));
    }

    @Override
    public void when ( Event<BankingDomainEvent> event ) {
        switch ( event.data() ) {
            case AccountOpened opened ->
                details = new AccountDetails(opened.accountId().value(), opened.customerId().value(), opened.date(), BigDecimal.ZERO);
            case MoneyDeposited deposited ->
                details = details.withBalance(details.balance().add(deposited.amount()));
        }
    }

    public Optional<AccountDetails> getAccountDetails ( ) {
        return Optional.ofNullable(details);
    }

    // a read model's answer is never stored, so it may validate as strictly as it likes
    public record AccountDetails ( String accountId, String customerId, LocalDate openDate, BigDecimal balance ) {
        public AccountDetails {
            if ( accountId == null || customerId == null || openDate == null || balance == null ) {
                throw new IllegalArgumentException("all fields are required");
            }
        }
        AccountDetails withBalance ( BigDecimal newBalance ) {
            return new AccountDetails(accountId, customerId, openDate, newBalance);
        }
    }
}
```

Register it from `features/accountdetails/AccountDetailsFeatureSlice.java`:

```java
@FeatureSlice(chapter = "Account management")
public class AccountDetailsFeatureSlice implements Slice<Banking> {

    @Override
    public void configureQuery ( BoundedContextBuilder<Banking> builder ) {
        builder.readmodel(AccountDetailsReadModel.class).live();
    }
}
```

**Key concepts:**
- A read model implements `ReadModel<E>`: `eventQuery()` says which events it is projected from, and `when(Event<E>)` handles each one. The domain event is `event.data()`; its tags, timestamp and reference sit beside it on the `Event`.
- A live read model is registered **by class**, and constructed per read with the arguments the read passes (`read(AccountDetailsReadModel.class, accountId)`).
- Every registration says how the read model is projected — `.live()` or `.eventuallyConsistent()`. `build()` rejects one that does not.
- A live read model is right whenever the events it replays are bounded by design, as here: one account's events. See [Choosing a Read Model](/posts/eventmodeling-read-models/) for what to do when they are not.

## Step 7: A State Read in the Background — Account Overview

A list of every account spans all accounts, so replaying it per read would get slower forever. It is projected once, in the background, and reads become free. That makes it **eventually consistent**: it lags the latest events by a moment.

Create `features/accountoverview/AccountOverviewReadModel.java`:

```java
package com.example.banking.features.accountoverview;

import java.time.LocalDate;
import java.util.Collections;
import java.util.LinkedHashSet;
import java.util.Set;

import org.sliceworkz.eventmodeling.readmodels.PublishingReadModel;
import org.sliceworkz.eventstore.events.Event;
import org.sliceworkz.eventstore.query.EventQuery;
import com.example.banking.BankingDomain.BankingDomainEvent;
import com.example.banking.BankingDomain.BankingDomainEvent.AccountOpened;

public class AccountOverviewReadModel
        extends PublishingReadModel<BankingDomainEvent, Set<AccountOverviewReadModel.AccountSummary>> {

    // registered as an instance: the framework projects this one in the background
    public static final AccountOverviewReadModel INSTANCE = new AccountOverviewReadModel();

    @Override
    public EventQuery eventQuery ( ) {
        return EventQuery.forTypes(AccountOpened.class);
    }

    @Override
    protected Set<AccountSummary> initialState ( ) {
        return Set.of();
    }

    @Override
    protected Set<AccountSummary> apply ( Set<AccountSummary> accounts, Event<BankingDomainEvent> event ) {
        return switch ( event.data() ) {
            case AccountOpened opened -> with(accounts,
                    new AccountSummary(opened.accountId().value(), opened.customerId().value(), opened.date()));
            default -> accounts;
        };
    }

    private static Set<AccountSummary> with ( Set<AccountSummary> accounts, AccountSummary added ) {
        Set<AccountSummary> next = new LinkedHashSet<>(accounts);
        next.add(added);
        return Collections.unmodifiableSet(next);
    }

    public record AccountSummary ( String accountId, String customerId, LocalDate openDate ) { }
}
```

Register the instance from `features/accountoverview/AccountOverviewFeatureSlice.java`:

```java
@FeatureSlice(chapter = "Account management")
public class AccountOverviewFeatureSlice implements Slice<Banking> {

    @Override
    public void configureQuery ( BoundedContextBuilder<Banking> builder ) {
        builder.readmodel(AccountOverviewReadModel.INSTANCE).eventuallyConsistent();
    }
}
```

**Key concepts:**
- An eventually consistent read model is registered **as an instance**: the framework projects that one object on a background thread, from a bookmark.
- `PublishingReadModel` is the base to extend. You write a state type, an initial value and a fold (`apply`); it publishes the state together with the position it reflects, once per batch, as one immutable value. A reader on another thread can never see a half-applied batch, and `published()` tells the caller exactly how current the answer is.
- Keep the state immutable: `apply` returns a new value rather than changing the old one.

## Step 8: Build, Start and Use the Bounded Context

Create `BankingApplication.java`:

```java
package com.example.banking;

import java.math.BigDecimal;
import java.util.Set;

import org.sliceworkz.eventmodeling.boundedcontext.BoundedContext;
import org.sliceworkz.eventmodeling.commands.CommandExecutionResult;
import org.sliceworkz.eventmodeling.events.InstanceFactory;
import org.sliceworkz.eventmodeling.readmodels.ReadModelResult;
import org.sliceworkz.eventstore.events.EventReference;
import org.sliceworkz.eventstore.infra.inmem.InMemoryEventStorage;
import org.sliceworkz.eventstore.spi.EventStorage;
import com.example.banking.BankingDomain.AccountId;
import com.example.banking.features.accountdetails.AccountDetailsReadModel;
import com.example.banking.features.accountoverview.AccountOverviewReadModel;
import com.example.banking.features.accountoverview.AccountOverviewReadModel.AccountSummary;
import com.example.banking.features.deposit.DepositCommand;
import com.example.banking.features.openaccount.OpenAccountCommand;

public class BankingApplication {

    public static void main ( String[] args ) {

        // 1. the event storage: in memory here, PostgreSQL in production
        EventStorage eventStorage = InMemoryEventStorage.newBuilder().build();

        // 2. build the bounded context; feature slices are found by scanning the package
        Banking banking = BoundedContext.newBuilder(Banking.class)
            .name("banking")
            .eventStorage(eventStorage)
            .instance(InstanceFactory.determine("banking-app"))
            .features()
                .rootPackage(BankingApplication.class.getPackage())
                .done()
            .build();

        // 3. start it: processors start, eventually consistent read models catch up
        banking.start();

        // 4. application code holds the narrow surface: execute commands, read read models
        BankingApi app = banking;

        CommandExecutionResult<AccountId> opened =
            app.execute(new OpenAccountCommand(BankingDomain.CUSTOMER.newId()));
        AccountId accountId = opened.response();
        EventReference openedAt = opened.eventReference().orElseThrow();
        System.out.println("Account opened: " + accountId.value());

        app.execute(new DepositCommand(accountId, new BigDecimal("100.00")));

        // 5. a live read model is projected when it is read: always current
        AccountDetailsReadModel details = app.read(AccountDetailsReadModel.class, accountId);
        System.out.println("Details: " + details.getAccountDetails().orElseThrow());

        // 6. an eventually consistent read model answers with the position it reflects
        ReadModelResult<Set<AccountSummary>> overview = AccountOverviewReadModel.INSTANCE.published();
        boolean includesOurAccount = overview.upTo() != null
            && !openedAt.happenedAfter(overview.upTo());
        System.out.println("Overview: " + overview.data() + " (includes our account: " + includesOurAccount + ")");

        // 7. shut down from the outside in: the context first, then the storage it was given
        banking.terminate();
        eventStorage.close();
    }
}
```

**Key concepts:**
- `BoundedContext.newBuilder(Banking.class)` resolves the three event types from the interface. `name` names the context (and the streams it keeps its events in), `instance` identifies this deployment, and `features().rootPackage(...)` scans for feature slices declared for `Banking`.
- `build()` assembles the context and checks every registration — a read model without a projection mode, a component of another context's event types, two components sharing a name. `start()` runs it.
- `read(AccountDetailsReadModel.class, accountId)` is typed by the class it asks for: no cast.
- `published()` hands back the state and the position it reflects as one observation, so a caller can check whether its own write is included instead of guessing.
- The context closes the event store it built; the storage came from you, so you close it, after the context.

## Step 9: Run the Application

```bash
mvn clean compile exec:java -Dexec.mainClass="com.example.banking.BankingApplication"
```

You should see output similar to:

```
Account opened: b8c6c4ec-ad2a-44ab-80fc-e23c07887391
Details: AccountDetails[accountId=b8c6c4ec-..., customerId=57db479c-..., openDate=2026-10-12, balance=100.00]
Overview: [AccountSummary[accountId=b8c6c4ec-..., customerId=57db479c-..., openDate=2026-10-12]] (includes our account: true)
```

The overview may say `includes our account: false` on another run: the background projector had not reached the new account yet. That is what eventually consistent means, and `published()` is what lets the caller see it.

## Step 10: Test a Command

The testing module's `CommandTest` runs a command against a fresh bounded context, given a history:

```java
package com.example.banking.features.deposit;

import java.math.BigDecimal;
import java.time.LocalDate;

import org.junit.jupiter.api.Test;
import org.sliceworkz.eventmodeling.testing.CommandTest;
import org.sliceworkz.eventstore.events.Tags;
import com.example.banking.BankingDomain.AccountId;
import com.example.banking.BankingDomain.BankingDomainEvent;
import com.example.banking.BankingDomain.BankingDomainEvent.AccountOpened;
import com.example.banking.BankingDomain.BankingDomainEvent.MoneyDeposited;
import com.example.banking.BankingDomain.BankingInboundEvent;
import com.example.banking.BankingDomain.BankingOutboundEvent;
import com.example.banking.BankingDomain.CustomerId;

import static com.example.banking.BankingDomain.ACCOUNT;
import static com.example.banking.BankingDomain.CUSTOMER;

class DepositCommandTest extends CommandTest<BankingDomainEvent, BankingInboundEvent, BankingOutboundEvent> {

    @Override public Class<BankingDomainEvent> domainEventType ( )     { return BankingDomainEvent.class; }
    @Override public Class<BankingInboundEvent> inboundEventType ( )   { return BankingInboundEvent.class; }
    @Override public Class<BankingOutboundEvent> outboundEventType ( ) { return BankingOutboundEvent.class; }

    AccountId account = ACCOUNT.id("acc-1");
    CustomerId customer = CUSTOMER.id("cust-1");

    @Test
    void depositOnAnOpenAccount ( ) {
        given()
            .event(new AccountOpened(account, customer, LocalDate.now()), Tags.of(ACCOUNT.tag(account), CUSTOMER.tag(customer)))
        .when(new DepositCommand(account, new BigDecimal("100")))
        .then()
            .event(new MoneyDeposited(account, new BigDecimal("100")), ACCOUNT.tags(account));
    }

    @Test
    void depositOnAnUnknownAccountIsRejected ( ) {
        given()
        .when(new DepositCommand(account, new BigDecimal("100")))
        .then()
            .businessError("Account acc-1 does not exist");
    }
}
```

- Seed history with the tags production would put on it: the decision model finds events by tag, so an untagged `AccountOpened` would not count as an open account.
- `.event(expected, tags)` checks the raised event and that it carries at least those tags.
- `.businessError(message)` asserts the command was rejected with a `BusinessException`, so a bug throwing some other exception with the same message does not pass.

Annotate a test `@ForEachBackend` instead of `@Test` to run it against every event storage you register, PostgreSQL included. See [Testing Feature Slices](/posts/eventmodeling-testing/).

## Project Structure

```
src/main/java/com/example/banking/
├── BankingDomain.java                    # entities and events
├── Banking.java                          # the bounded context
├── BankingApi.java                       # what application code may do with it
├── BankingApplication.java               # builds, starts and uses it
└── features/
    ├── openaccount/
    │   ├── OpenAccountFeatureSlice.java
    │   └── OpenAccountCommand.java
    ├── deposit/
    │   ├── DepositFeatureSlice.java
    │   ├── AccountExistsDecisionModel.java
    │   └── DepositCommand.java
    ├── accountdetails/
    │   ├── AccountDetailsFeatureSlice.java
    │   └── AccountDetailsReadModel.java
    └── accountoverview/
        ├── AccountOverviewFeatureSlice.java
        └── AccountOverviewReadModel.java
src/test/java/com/example/banking/features/deposit/
└── DepositCommandTest.java
```

## Next Steps

The guides take each part further:

1. [Bounded Contexts and Feature Slices](/posts/eventmodeling-bounded-contexts-and-feature-slices/) — slices, aspects, building and starting a context, and what `build()` checks
2. [Entities, Typed Ids and Tags](/posts/eventmodeling-entities-and-tags/) — how events are tagged and found
3. [Commands and Decision Models](/posts/eventmodeling-commands-and-decision-models/) — the consistency boundary, retries, idempotency keys and aggregates
4. [Where a Validation Goes](/posts/eventmodeling-where-validations-go/) — value objects, input checks, rules, uniqueness
5. [Business Rules and Enforcement Levels](/posts/eventmodeling-business-rules/) — rules people may override, and previewing a command before submitting it
6. [Choosing a Read Model](/posts/eventmodeling-read-models/) and [Durable SQL Read Models](/posts/eventmodeling-sql-read-models/)
7. [Automations and Policies](/posts/eventmodeling-automations-and-policies/) — reacting to events
8. [Translators, Publishers and Dispatchers](/posts/eventmodeling-translators-publishers-dispatchers/) — talking to other systems through events
9. [Ports, Adapters and Monitored Calls](/posts/eventmodeling-ports-and-adapters/) — talking to other systems through calls
10. [Who May Do What](/posts/eventmodeling-who-may-do-what/) — handing each caller only the capabilities it needs
11. [Testing Feature Slices](/posts/eventmodeling-testing/)
12. [Running in Production](/posts/eventmodeling-running-in-production/), [Observability and Tracing](/posts/eventmodeling-observability/) and [Personal Data and Erasure](/posts/eventmodeling-personal-data/)

The [examples module](https://github.com/sliceworkz/eventmodeling/tree/develop/sliceworkz-eventmodeling-examples) in the repository holds a larger banking context — closing the books, business rules, a policy — and a payments context with an automation whose external calls can fail.
