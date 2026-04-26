---
layout: post
toc: true
title: EventModeling Quickstart Guide
description: Gettings started with the Sliceworkz EventModeling
icon: fas fa-rocket
categories: [Eventmodeling Documentation]
tags: [quickstart,eventmodeling,maven]
pin: true
order: 3
---


This guide will help you get started with the EventModeling library for Java. 

EventStore is an opinionated Java-base implementation framework for <a href="https://eventmodeling.org">Event Modeling</a>.
It builds upon the <a href="/eventstore/">EventStore</a>, and provides direct implementation mechanisms for Commands, Readmodels, Automations and Translations.

# Quickstart Guide

This guide walks you through building your first application with the Sliceworkz Event Modeling framework. We'll create a simple banking domain with account management functionality.


## Prerequisites

- Java 21 or later
- Maven 3.6 or later

## Step 1: Add Maven Dependencies

Sliceworkz Eventmodeling is available in <a href="https://mvnrepository.com/artifact/org.sliceworkz">maven central</a>.

All Eventmodeling modules are bundled in a Bill-Of-Material pom file.
Add the Eventmodeling BOM to your project pom.xml to manage dependency versions:

```xml
...
<properties>
    <sliceworkz.eventmodeling.version>0.5.0</sliceworkz.eventmodeling.version>
</properties>
...
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
...
```


Add the Event Modeling implementation and an event storage provider to your `pom.xml`:

```xml
<dependencies>
    <!-- Event Modeling Framework Implementation -->
    <dependency>
        <groupId>org.sliceworkz</groupId>
        <artifactId>sliceworkz-eventmodeling-impl</artifactId>
    </dependency>

    <!-- In-Memory Event Storage (for development/testing) -->
    <dependency>
        <groupId>org.sliceworkz</groupId>
        <artifactId>sliceworkz-eventstore-infra-inmem</artifactId>
    </dependency>
</dependencies>
```

**Note:** For production use, replace `sliceworkz-eventstore-infra-inmem` with `sliceworkz-eventstore-infra-postgres` for PostgreSQL-based event storage. For local development with file persistence (events survive restarts without requiring PostgreSQL), use `sliceworkz-eventstore-infra-inmem-fs` instead.

> **PostgreSQL version support.** From `0.5.0` onwards, EventModeling depends on Eventstore `0.8.0`, which targets **PostgreSQL 18+** by default and uses native server-side `uuidv7()` for event ids. PostgreSQL 13–17 are still supported through a legacy code path that requires the optional `com.github.f4b6a3:uuid-creator` dependency to be added explicitly to your application; the right implementation is picked automatically at startup. See the [PostgreSQL EventStorage guide](/posts/eventstore-configuring-postgresql-storage/#postgresql-version-support-and-uuidv7-generation) for the dependency snippet and runtime behaviour.
{: .prompt-info }

## Step 2: Define Your Domain Events

Create a domain interface that defines your three event types: domain events (internal), inbound events (received from external systems), and outbound events (published to external systems).

Create `BankingDomain.java`:

```java
package com.example.banking;

import java.time.LocalDate;
import org.sliceworkz.eventmodeling.domain.DomainConcept;
import org.sliceworkz.eventmodeling.domain.DomainConceptId;

public interface BankingDomain {

    // Define domain concepts for tagging events
    DomainConcept CONCEPT_ACCOUNT = DomainConcept.of("account");
    DomainConcept CONCEPT_CUSTOMER = DomainConcept.of("customer");

    // Domain Events: Internal state changes (use past-tense names)
    sealed interface BankingDomainEvent {
        record AccountOpened(
            DomainConceptId accountId,
            DomainConceptId customerId,
            LocalDate date
        ) implements BankingDomainEvent { }
    }

    // Inbound Events: Events received from external systems
    sealed interface BankingInboundEvent {
        record CustomerSuspended(
            DomainConceptId customerId
        ) implements BankingInboundEvent { }
    }

    // Outbound Events: Events published to external systems
    sealed interface BankingOutboundEvent {
        record AccountAnnounced(
            DomainConceptId id
        ) implements BankingOutboundEvent { }
    }
}
```

**Key concepts:**
- Use `sealed interface` to create type-safe event hierarchies
- Use `record` for immutable event data
- Name domain events in past-tense (e.g., `AccountOpened`, not `OpenAccount`)
- Domain concepts are used to tag and query events

## Step 3: Create a Bounded Context Interface

The bounded context is the main entry point to your application. Create an interface that extends `BoundedContext`:

Create `Banking.java`:

```java
package com.example.banking;

import org.sliceworkz.eventmodeling.boundedcontext.BoundedContext;
import com.example.banking.BankingDomain.BankingDomainEvent;
import com.example.banking.BankingDomain.BankingInboundEvent;
import com.example.banking.BankingDomain.BankingOutboundEvent;

public interface Banking
    extends BoundedContext<BankingDomainEvent, BankingInboundEvent, BankingOutboundEvent> {
    // This interface provides execute() and read() capabilities
}
```

**Key concepts:**
- `BoundedContext` itself extends `EventTypes<D,I,O>`, so your context interface acts as a single type parameter that groups all three event types
- This allows the builder and feature slice APIs to use `Banking` as a single type parameter instead of repeating three event types everywhere

## Step 4: Create a STATE_CHANGE Feature Slice (Command)

State change features handle commands that modify state. We'll implement a feature to open bank accounts.

### 4.1: Create the Feature Slice Configuration

Create `features/openaccount/OpenAccountFeatureSlice.java`:

```java
package com.example.banking.features.openaccount;

import org.sliceworkz.eventmodeling.slices.FeatureSlice;
import org.sliceworkz.eventmodeling.slices.FeatureSlice.Type;
import org.sliceworkz.eventmodeling.slices.Slice;
import com.example.banking.Banking;

@FeatureSlice(
    type = Type.STATE_CHANGE,
    context = "banking",
    chapter = "Account management",
    tags = {"online"}
)
public class OpenAccountFeatureSlice implements Slice<Banking> {
}
```

**Key concepts:**
- `@FeatureSlice` annotation marks this class for automatic discovery
- `type = Type.STATE_CHANGE` indicates this feature handles commands
- `context`, `chapter`, and `tags` provide organizational metadata
- Feature slices implement `Slice<C>` with your bounded context type as the type parameter
- `Slice` provides default no-op implementations for all lifecycle methods, so you only override what you need
- For `STATE_CHANGE` slices, commands don't need explicit registration

### 4.2: Create the Command Implementation

Create `features/openaccount/OpenAccountCommand.java`:

```java
package com.example.banking.features.openaccount;

import java.time.LocalDate;
import org.sliceworkz.eventmodeling.commands.Command;
import org.sliceworkz.eventmodeling.commands.CommandContext;
import org.sliceworkz.eventmodeling.domain.DomainConceptId;
import org.sliceworkz.eventmodeling.domain.DomainConceptTag;
import org.sliceworkz.eventstore.events.Tags;
import com.example.banking.BankingDomain;
import com.example.banking.BankingDomain.BankingDomainEvent;
import com.example.banking.BankingDomain.BankingDomainEvent.AccountOpened;

public class OpenAccountCommand implements Command<BankingDomainEvent> {

    private final DomainConceptId customerId;

    public OpenAccountCommand(DomainConceptId customerId) {
        this.customerId = customerId;
    }

    @Override
    public void execute(
        CommandContext<BankingDomainEvent, BankingDomainEvent> context) {

        var result = context.noDecisionModels();

        // Generate a unique account ID
        DomainConceptId accountId = DomainConceptId.create();

        // Raise the AccountOpened event with appropriate tags
        result.raiseEvent(
            new AccountOpened(accountId, customerId, LocalDate.now()),
            Tags.of(
                DomainConceptTag.of(BankingDomain.CONCEPT_ACCOUNT, accountId),
                DomainConceptTag.of(BankingDomain.CONCEPT_CUSTOMER, customerId)
            )
        );
    }
}
```

**Key concepts:**
- Commands implement `Command<EVENT_TYPE>`
- `execute()` method receives a `CommandContext` and returns `void`
- `raiseEvent()` registers the event to be persisted after the command completes
- Tag events with domain concepts for efficient querying
- If you need to return a value to the caller (e.g., a generated ID), use `CommandWithResult<EVENT_TYPE, RESPONSE_TYPE>` instead — see Step 5 below

## Step 5: Create a STATE_CHANGE with CommandWithResult

Sometimes a command needs to return a value to the caller — for example, a generated ID. `CommandWithResult` allows this while still guaranteeing the response is only delivered after events are persisted.

### 5.1: Create the Feature Slice Configuration

Create `features/openaccountwithresult/OpenAccountWithResultFeatureSlice.java`:

```java
package com.example.banking.features.openaccountwithresult;

import org.sliceworkz.eventmodeling.slices.FeatureSlice;
import org.sliceworkz.eventmodeling.slices.FeatureSlice.Type;
import org.sliceworkz.eventmodeling.slices.Slice;
import com.example.banking.Banking;

@FeatureSlice(
    type = Type.STATE_CHANGE,
    context = "banking",
    chapter = "Account management",
    tags = {"online"}
)
public class OpenAccountWithResultFeatureSlice implements Slice<Banking> {
}
```

### 5.2: Create the CommandWithResult Implementation

Create `features/openaccountwithresult/OpenAccountWithResultCommand.java`:

```java
package com.example.banking.features.openaccountwithresult;

import java.time.LocalDate;
import org.sliceworkz.eventmodeling.commands.CommandContext;
import org.sliceworkz.eventmodeling.commands.CommandWithResult;
import org.sliceworkz.eventmodeling.domain.DomainConceptId;
import org.sliceworkz.eventmodeling.domain.DomainConceptTag;
import org.sliceworkz.eventstore.events.Tags;
import com.example.banking.BankingDomain;
import com.example.banking.BankingDomain.BankingDomainEvent;
import com.example.banking.BankingDomain.BankingDomainEvent.AccountOpened;

public class OpenAccountWithResultCommand
    implements CommandWithResult<BankingDomainEvent, DomainConceptId> {

    private final DomainConceptId customerId;

    public OpenAccountWithResultCommand(DomainConceptId customerId) {
        this.customerId = customerId;
    }

    @Override
    public DomainConceptId execute(
        CommandContext<BankingDomainEvent, BankingDomainEvent> context) {

        var result = context.noDecisionModels();

        DomainConceptId accountId = DomainConceptId.create();

        result.raiseEvent(
            new AccountOpened(accountId, customerId, LocalDate.now()),
            Tags.of(
                DomainConceptTag.of(BankingDomain.CONCEPT_ACCOUNT, accountId),
                DomainConceptTag.of(BankingDomain.CONCEPT_CUSTOMER, customerId)
            )
        );

        return accountId;  // Returned to caller after events are persisted
    }
}
```

**Key concepts:**
- `CommandWithResult<EVENT_TYPE, RESPONSE_TYPE>` allows returning a value from command execution
- The `execute()` method returns the response type directly
- The response is only delivered to the caller after events have been successfully persisted
- The caller receives a `CommandExecutionResult<RESPONSE_TYPE>` containing both the event reference and the response value

## Step 6: Create a STATE_READ Feature Slice (Read Model)

Read models project events into queryable views. We'll create a read model to view account details.

### 6.1: Create the Feature Slice Configuration

Create `features/accountdetails/AccountDetailsFeatureSlice.java`:

```java
package com.example.banking.features.accountdetails;

import org.sliceworkz.eventmodeling.boundedcontext.BoundedContextBuilder;
import org.sliceworkz.eventmodeling.slices.FeatureSlice;
import org.sliceworkz.eventmodeling.slices.FeatureSlice.Type;
import org.sliceworkz.eventmodeling.slices.Slice;
import com.example.banking.Banking;

@FeatureSlice(
    type = Type.STATE_READ,
    context = "banking",
    tags = {"online"}
)
public class AccountDetailsFeatureSlice implements Slice<Banking> {

    @Override
    public void configureQuery(BoundedContextBuilder<Banking> builder) {
        // Register the read model
        builder.readmodel(AccountDetailsReadModel.class);
    }
}
```

### 6.2: Create the Read Model Implementation

Create `features/accountdetails/AccountDetailsReadModel.java`:

```java
package com.example.banking.features.accountdetails;

import java.time.LocalDate;
import java.util.Optional;
import org.sliceworkz.eventmodeling.domain.DomainConceptId;
import org.sliceworkz.eventmodeling.domain.DomainConceptTags;
import org.sliceworkz.eventmodeling.readmodels.ReadModel;
import org.sliceworkz.eventstore.query.EventQuery;
import org.sliceworkz.eventstore.query.EventTypesFilter;
import com.example.banking.BankingDomain;
import com.example.banking.BankingDomain.BankingDomainEvent;
import com.example.banking.BankingDomain.BankingDomainEvent.AccountOpened;

public class AccountDetailsReadModel implements ReadModel<BankingDomainEvent> {

    private final DomainConceptId accountId;
    private AccountDetails account;

    public AccountDetailsReadModel(DomainConceptId accountId) {
        this.accountId = accountId;
    }

    public Optional<AccountDetails> getAccountDetails() {
        return Optional.ofNullable(account);
    }

    @Override
    public EventQuery eventQuery() {
        // Query only events tagged with this specific account
        return EventQuery.forEvents(
            EventTypesFilter.any(),
            DomainConceptTags.of(BankingDomain.CONCEPT_ACCOUNT, accountId)
        );
    }

    @Override
    public void when(BankingDomainEvent event) {
        // Handle events to build up the read model state
        switch (event) {
            case AccountOpened ao:
                this.account = AccountDetails.of(
                    ao.accountId().value(),
                    ao.customerId().value(),
                    ao.date()
                );
                break;
        }
    }

    public record AccountDetails(String accountId, String customerId, LocalDate openDate) {

        public AccountDetails {
            if (accountId == null) throw new IllegalArgumentException("accountId is required");
            if (customerId == null) throw new IllegalArgumentException("customerId is required");
            if (openDate == null) throw new IllegalArgumentException("openDate is required");
        }

        public static AccountDetails of(String accountId, String customerId, LocalDate openDate) {
            return new AccountDetails(accountId, customerId, openDate);
        }
    }
}
```

**Key concepts:**
- Read models implement `ReadModel<EVENT_TYPE>`
- `eventQuery()` defines which events to replay into this model
- `when()` method handles each event to build up the model's state
- Read models are instantiated per query (pass parameters via constructor)

## Step 7: Build and Initialize the Bounded Context

Now tie everything together by creating and configuring the bounded context:

Create `BankingApplication.java`:

```java
package com.example.banking;

import org.sliceworkz.eventmodeling.boundedcontext.BoundedContext;
import org.sliceworkz.eventmodeling.commands.CommandExecutionResult;
import org.sliceworkz.eventmodeling.domain.DomainConceptId;
import org.sliceworkz.eventmodeling.events.Instance;
import org.sliceworkz.eventmodeling.events.InstanceFactory;
import org.sliceworkz.eventstore.infra.inmem.InMemoryEventStorage;
import org.sliceworkz.eventstore.spi.EventStorage;
import com.example.banking.features.accountdetails.AccountDetailsReadModel;
import com.example.banking.features.openaccountwithresult.OpenAccountWithResultCommand;

public class BankingApplication {

    public static void main(String[] args) {

        // 1. Create event storage (in-memory for this example)
        EventStorage eventStorage = InMemoryEventStorage.newBuilder().build();

        // 2. Create an instance identifier (for multi-tenancy/deployment)
        Instance instance = InstanceFactory.determine("banking-app");

        // 3. Build the bounded context
        Banking bc = BoundedContext.newBuilder(Banking.class)
            .name("banking")
            .eventStorage(eventStorage)
            .instance(instance)
            .features()
                .rootPackage(BankingApplication.class.getPackage())  // Scans for @FeatureSlice
                .done()
            .build();

        // 4. Start the bounded context (triggers feature slice lifecycle)
        bc.start();

        // 5. Execute a command that returns a result
        DomainConceptId customerId = DomainConceptId.create();
        CommandExecutionResult<DomainConceptId> result =
            bc.execute(new OpenAccountWithResultCommand(customerId));

        DomainConceptId accountId = result.response();
        System.out.println("Account opened with ID: " + accountId);

        // 6. Query the read model
        AccountDetailsReadModel readModel = bc.read(
            AccountDetailsReadModel.class,
            accountId
        );

        readModel.getAccountDetails().ifPresent(details -> {
            System.out.println("Account Details:");
            System.out.println("  Account ID: " + details.accountId());
            System.out.println("  Customer ID: " + details.customerId());
            System.out.println("  Open Date: " + details.openDate());
        });
    }
}
```

## Step 8: Run Your Application

Build and run your application:

```bash
mvn clean compile exec:java -Dexec.mainClass="com.example.banking.BankingApplication"
```

You should see output similar to:

```
Account opened with ID: <generated-uuid>
Account Details:
  Account ID: <generated-uuid>
  Customer ID: <generated-uuid>
  Open Date: 2026-03-22
```

## Project Structure

Your final project structure should look like this:

```
src/main/java/com/example/banking/
├── BankingDomain.java                           # Event definitions
├── Banking.java                                 # Bounded context interface
├── BankingApplication.java                      # Main application
└── features/
    ├── openaccount/
    │   ├── OpenAccountFeatureSlice.java         # Feature slice (Slice<Banking>)
    │   └── OpenAccountCommand.java              # Command implementation
    ├── openaccountwithresult/
    │   ├── OpenAccountWithResultFeatureSlice.java  # Feature slice (Slice<Banking>)
    │   └── OpenAccountWithResultCommand.java    # CommandWithResult implementation
    └── accountdetails/
        ├── AccountDetailsFeatureSlice.java      # Feature slice (Slice<Banking>)
        └── AccountDetailsReadModel.java         # Read model implementation
```

## Next Steps

Now that you have a working Event Modeling application, you can:

1. **Add more events** to `BankingDomainEvent` (e.g., `MoneyDeposited`, `MoneyWithdrawn`)
2. **Implement decision models** to enforce business rules in commands
3. **Use aggregates** for traditional aggregate-style command handling with snapshot support
4. **Create automations** to implement automated activities
5. **Add translators** to handle inbound events from external systems
6. **Implement dispatchers** for the outbox pattern to publish outbound events
7. **Use adapter/port bindings** to inject infrastructure dependencies into feature slices
8. **Add feature slice lifecycle hooks** (`startCommand`, `startQuery`, `startAutomation`, `startProjection`) for post-build initialization
9. **Use SQL-backed read models** with `SqlReadModelProjector` and `SqlReadModelQuery` for persistent, database-backed read models (supports H2 for development and PostgreSQL for production)
10. **Switch to PostgreSQL** for production-ready event storage
11. **Add observability** with Micrometer metrics for monitoring commands, events, and read models
12. **Add tests** using the `sliceworkz-eventmodeling-testing` module, including `satisfies()` for custom assertions on live model results and `SqlReadModelTest` for testing SQL read models against both H2 and PostgreSQL

For more examples, see the `sliceworkz-eventmodeling-examples` module in the repository.
