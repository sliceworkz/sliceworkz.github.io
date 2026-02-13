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
    <sliceworkz.eventmodeling.version>0.2.0</sliceworkz.eventmodeling.version>
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

**Note:** For production use, replace `sliceworkz-eventstore-infra-inmem` with `sliceworkz-eventstore-infra-postgres` for PostgreSQL-based event storage.

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

Create `BankingBoundedContext.java`:

```java
package com.example.banking;

import org.sliceworkz.eventmodeling.boundedcontext.BoundedContext;
import com.example.banking.BankingDomain.BankingDomainEvent;
import com.example.banking.BankingDomain.BankingInboundEvent;
import com.example.banking.BankingDomain.BankingOutboundEvent;

public interface BankingBoundedContext
    extends BoundedContext<BankingDomainEvent, BankingInboundEvent, BankingOutboundEvent> {
    // This interface provides execute() and read() capabilities
}
```

## Step 4: Create a STATE_CHANGE Feature Slice (Command)

State change features handle commands that modify state. We'll implement a feature to open bank accounts.

### 4.1: Create the Feature Slice Configuration

Create `features/openaccount/OpenAccountFeatureSlice.java`:

```java
package com.example.banking.features.openaccount;

import org.sliceworkz.eventmodeling.boundedcontext.BoundedContextBuilder;
import org.sliceworkz.eventmodeling.slices.FeatureSlice;
import org.sliceworkz.eventmodeling.slices.FeatureSlice.Type;
import org.sliceworkz.eventmodeling.slices.FeatureSliceConfiguration;
import com.example.banking.BankingDomain.BankingDomainEvent;
import com.example.banking.BankingDomain.BankingInboundEvent;
import com.example.banking.BankingDomain.BankingOutboundEvent;

@FeatureSlice(
    type = Type.STATE_CHANGE,
    context = "banking",
    chapter = "Account management",
    tags = {"online"}
)
public class OpenAccountFeatureSlice
    implements FeatureSliceConfiguration<BankingDomainEvent, BankingInboundEvent, BankingOutboundEvent> {

    @Override
    public void configureCommand(
        BoundedContextBuilder<BankingDomainEvent, BankingInboundEvent, BankingOutboundEvent> builder) {
        // Commands are automatically discovered, no explicit registration needed
    }

    @Override
    public void configureQuery(
        BoundedContextBuilder<BankingDomainEvent, BankingInboundEvent, BankingOutboundEvent> builder) {
    }

    @Override
    public void configureAutomation(
        BoundedContextBuilder<BankingDomainEvent, BankingInboundEvent, BankingOutboundEvent> builder) {
    }
}
```

**Key concepts:**
- `@FeatureSlice` annotation marks this class for automatic discovery
- `type = Type.STATE_CHANGE` indicates this feature handles commands
- `context`, `chapter`, and `tags` provide organizational metadata
- Each feature slice has three configuration methods for commands, queries, and automations

### 4.2: Create the Command Implementation

Create `features/openaccount/OpenAccountCommand.java`:

```java
package com.example.banking.features.openaccount;

import java.time.LocalDate;
import org.sliceworkz.eventmodeling.commands.Command;
import org.sliceworkz.eventmodeling.commands.CommandContext;
import org.sliceworkz.eventmodeling.commands.CommandResult;
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
    public CommandResult<BankingDomainEvent, BankingDomainEvent> execute(
        CommandContext<BankingDomainEvent, BankingDomainEvent> context) {

        var result = context.noDecisionModels();

        // Generate a unique account ID
        DomainConceptId accountId = DomainConceptId.create();

        // Raise the AccountOpened event with appropriate tags
        return result.raiseEvent(
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
- `execute()` method receives a `CommandContext` and returns a `CommandResult`
- `raiseEvent()` returns the `CommandResult`, allowing fluent return statements
- Tag events with domain concepts for efficient querying

## Step 5: Create a STATE_READ Feature Slice (Read Model)

Read models project events into queryable views. We'll create a read model to view account details.

### 5.1: Create the Feature Slice Configuration

Create `features/accountdetails/AccountDetailsFeatureSlice.java`:

```java
package com.example.banking.features.accountdetails;

import org.sliceworkz.eventmodeling.boundedcontext.BoundedContextBuilder;
import org.sliceworkz.eventmodeling.slices.FeatureSlice;
import org.sliceworkz.eventmodeling.slices.FeatureSlice.Type;
import org.sliceworkz.eventmodeling.slices.FeatureSliceConfiguration;
import com.example.banking.BankingDomain.BankingDomainEvent;
import com.example.banking.BankingDomain.BankingInboundEvent;
import com.example.banking.BankingDomain.BankingOutboundEvent;

@FeatureSlice(
    type = Type.STATE_READ,
    context = "banking",
    tags = {"online"}
)
public class AccountDetailsFeatureSlice
    implements FeatureSliceConfiguration<BankingDomainEvent, BankingInboundEvent, BankingOutboundEvent> {

    @Override
    public void configureCommand(
        BoundedContextBuilder<BankingDomainEvent, BankingInboundEvent, BankingOutboundEvent> builder) {
    }

    @Override
    public void configureQuery(
        BoundedContextBuilder<BankingDomainEvent, BankingInboundEvent, BankingOutboundEvent> builder) {
        // Register the read model
        builder.readmodel(AccountDetailsReadModel.class);
    }

    @Override
    public void configureAutomation(
        BoundedContextBuilder<BankingDomainEvent, BankingInboundEvent, BankingOutboundEvent> builder) {
    }
}
```

### 5.2: Create the Read Model Implementation

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

## Step 6: Build and Initialize the Bounded Context

Now tie everything together by creating and configuring the bounded context:

Create `BankingApplication.java`:

```java
package com.example.banking;

import java.util.Optional;
import org.sliceworkz.eventmodeling.boundedcontext.BoundedContext;
import org.sliceworkz.eventmodeling.domain.DomainConceptId;
import org.sliceworkz.eventmodeling.events.Instance;
import org.sliceworkz.eventmodeling.events.InstanceFactory;
import org.sliceworkz.eventstore.EventStore;
import org.sliceworkz.eventstore.EventStoreFactory;
import org.sliceworkz.eventstore.events.Event;
import org.sliceworkz.eventstore.events.EventReference;
import org.sliceworkz.eventstore.infra.inmem.InMemoryEventStorage;
import org.sliceworkz.eventstore.spi.EventStorage;
import com.example.banking.BankingDomain.BankingDomainEvent;
import com.example.banking.BankingDomain.BankingDomainEvent.AccountOpened;
import com.example.banking.BankingDomain.BankingInboundEvent;
import com.example.banking.BankingDomain.BankingOutboundEvent;
import com.example.banking.features.accountdetails.AccountDetailsReadModel;
import com.example.banking.features.openaccount.OpenAccountCommand;

public class BankingApplication {

    public static void main(String[] args) {

        // 1. Create event storage (in-memory for this example)
        EventStorage eventStorage = InMemoryEventStorage.newBuilder().build();
        EventStore eventStore = EventStoreFactory.get().eventStore(eventStorage);

        // 2. Create an instance identifier (for multi-tenancy/deployment)
        Instance instance = InstanceFactory.determine("banking-app");

        // 3. Build the bounded context
        BankingBoundedContext bc = BoundedContext.newBuilder(
                BankingDomainEvent.class,
                BankingInboundEvent.class,
                BankingOutboundEvent.class
            )
            .name("banking")
            .eventStorage(eventStorage)
            .instance(instance)
            .features()
                .rootPackage(BankingApplication.class.getPackage())  // Scans for @FeatureSlice
                .done()
            .build(BankingBoundedContext.class);

        // 4. Execute a command
        DomainConceptId customerId = DomainConceptId.create();
        Optional<EventReference> eventRef = bc.execute(new OpenAccountCommand(customerId));

        if (eventRef.isPresent()) {
            System.out.println("Account opened successfully!");

            // 5. Retrieve the event that was raised
            var eventStream = eventStore.getEventStream(
                org.sliceworkz.eventstore.stream.EventStreamId
                    .forContext("banking")
                    .withPurpose("domain"),
                BankingDomainEvent.class
            );

            AccountOpened accountOpened = eventStream
                .getEventById(eventRef.get().id())
                .map(Event::data)
                .map(e -> (AccountOpened) e)
                .get();

            // 6. Query the read model
            AccountDetailsReadModel readModel = bc.read(
                AccountDetailsReadModel.class,
                accountOpened.accountId()
            );

            readModel.getAccountDetails().ifPresent(details -> {
                System.out.println("Account Details:");
                System.out.println("  Account ID: " + details.accountId());
                System.out.println("  Customer ID: " + details.customerId());
                System.out.println("  Open Date: " + details.openDate());
            });
        }
    }
}
```

## Step 7: Run Your Application

Build and run your application:

```bash
mvn clean compile exec:java -Dexec.mainClass="com.example.banking.BankingApplication"
```

You should see output similar to:

```
Account opened successfully!
Account Details:
  Account ID: <generated-uuid>
  Customer ID: <generated-uuid>
  Open Date: 2025-11-21
```

## Project Structure

Your final project structure should look like this:

```
src/main/java/com/example/banking/
├── BankingDomain.java                           # Event definitions
├── BankingBoundedContext.java                   # Bounded context interface
├── BankingApplication.java                      # Main application
└── features/
    ├── openaccount/
    │   ├── OpenAccountFeatureSlice.java         # Feature configuration
    │   └── OpenAccountCommand.java              # Command implementation
    └── accountdetails/
        ├── AccountDetailsFeatureSlice.java      # Feature configuration
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
7. **Switch to PostgreSQL** for production-ready event storage
8. **Add observability** with Micrometer metrics for monitoring commands, events, and read models
9. **Add tests** using the `sliceworkz-eventmodeling-testing` module

For more examples, see the `sliceworkz-eventmodeling-examples` module in the repository.
